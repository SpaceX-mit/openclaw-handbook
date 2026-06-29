# OpenClaw 系统安全功能分析

> 源码级安全分析。证据以仓库根相对 `file:line` 标注（来自 SECURITY.md + 三路并行源码调查）。
> 覆盖：威胁模型 → 认证/授权/Scope → 设备配对/身份 → 沙箱 → 工具策略/exec → 密钥管理 → SSRF/网络 → 输入信任边界 → 插件安全 → 静态分析 → 已知短板。

---

## 0. 威胁模型（决定一切设计取舍）

OpenClaw 的安全姿态由其定位决定 —— **「本地优先、面向受信任操作者的 agent 基础设施」**，**不是**多租户共享网关上对抗性用户间的隔离边界（`SECURITY.md`）。这条定位是理解所有安全设计的钥匙。

**明确不视为漏洞**（`SECURITY.md` "What Usually Is Not a Security Bug"）：
- 无边界绕过的纯 prompt injection。
- 受信任操作者使用本地 shell/浏览器/脚本等**有意功能**。
- 在运行 OpenClaw 前篡改进程/子进程环境变量。
- 操作者主动安装/启用恶意插件后的行为。
- 多个对抗性用户共享一个网关却期望按用户隔离。

**真正的安全边界**（受保护对象）：auth / scope 授权 / approval 审批 / sandbox 沙箱 / tool 工具边界。报告必须证明**跨越**这些边界才算漏洞。安全工作由 NVIDIA、Tencent 等组织的工程师/研究员共同参与。

---

## 1. 认证（Authentication）

网关把所有连接认证归一为单一 `GatewayAuthResult`，`method` 是闭合联合：`none | token | password | tailscale | device-token | bootstrap-token | trusted-proxy`（`src/gateway/auth.ts:39`）。

| 认证方式 | 机制 | 证据 |
|---|---|---|
| **共享 token / password** | 常量时间比较 `safeEqualSecret`；**缺凭证不消耗限流额度**，仅 mismatch 才 `recordFailure`（避免锁死未带凭证的客户端） | `auth.ts:405,427,399-409` |
| **device-token** | 每设备 bearer，校验配对设备库 | `infra/device-pairing.ts:1052` |
| **bootstrap-token** | 首次配对用，**10 分钟 TTL**，绑定到首个赎回它的设备身份 | `device-bootstrap.ts:23,433,484` |
| **approval-runtime-token** | loopback 操作者审批客户端用，`HMAC-SHA256(socketToken, "openclaw:gateway-approval-runtime-token:v1")`，`timingSafeEqual` + 长度守卫 | `operator-approval-runtime-token.ts:21` |
| **tailscale 头认证** | 仅 `ws-control-ui` 面；要求 loopback + 完整 `x-forwarded-*` + `tailscale whois` 校验 | `auth.ts:359,223` |
| **trusted-proxy** | 校验远端为配置的可信代理，强制必需头 + 用户头 + 可选 allowUsers | `auth.ts:305` |

**默认 token 模式**：即使未设 token，模式仍默认 token，启动断言 `assertGatewayAuthConfigured` 产出清晰「缺 token」错误而非静默关闭认证（`auth.ts:257`）。trusted-proxy 与 token 互斥。

---

## 2. 授权与 Scope 模型（最小权限）

**Scope 词表**（闭合 6 个，`src/gateway/operator-scopes.ts:3`）：`operator.admin`（超级用户，绕过所有逐方法检查）、`operator.read`、`operator.write`、`operator.approvals`、`operator.pairing`、`operator.talk.secrets`。

**方法门控**：每个网关方法经静态 descriptor 映射到所需 scope（`src/gateway/methods/core-descriptors.ts`），解析优先级 core descriptor → 保留命名空间 → 插件 descriptor（`method-scopes.ts:43`）。强制点 `authorizeGatewayMethod`（`server-methods.ts:230`）。

**最小权限特征**：
- **默认拒绝**：未分类方法默认要求 `ADMIN_SCOPE`（`method-scopes.ts:177`），最小权限解析对未分类方法返回 `[]`（无可满足 scope）。
- **读写层级**：write 满足 read 要求，反之不行。
- **`controlPlaneWrite` 标记**：高影响 admin 方法（`config.apply`/`config.patch`/`update.run`/`gateway.restart.request`）额外分类。

**🔑 反提权核心控制：scope 绑定身份**（`message-handler.ts:846-849`）—— **自声明的 `connectParams.scopes` 不被信任**。无设备身份的客户端，认证后 scope 被 `clearUnboundScopes()` 清空（默认拒绝）；有设备身份时，有效 scope 来自设备 `approvedScopes` 基线，且**设备签名覆盖 role+scopes**（见 §3），故客户端无法授予自己 owner 未批准的 scope。有专门的回归 POC 测试 `server.silent-scope-upgrade-reconnect.poc.test.ts`。

---

## 3. 设备配对与身份（Ed25519）

**身份**（`src/infra/device-identity.ts`）：`crypto.generateKeyPairSync("ed25519")`，`deviceId` = 公钥 SHA-256 指纹（`:77,100`）。身份与密钥材料绑定 —— 握手重新派生 deviceId 并拒绝不匹配（`message-handler.ts:1136`）。

**签名校验**（`verifyDeviceSignature`，`device-identity.ts:329`）握手强制：
- `deviceId === fingerprint(publicKey)`。
- 时效：`|now - signedAt| ≤ DEVICE_SIGNATURE_SKEW_MS`（重放窗口）。
- nonce 必须等于服务端签发的 `connectNonce`（防重放）。
- 签名覆盖 `deviceId, clientId, clientMode, role, scopes, signedAtMs, token, nonce`（v3 加 platform/deviceFamily）。**role 与 scopes 在签名载荷内 → 无法越权声明。**

**配对信任模型**（`src/infra/device-pairing.ts`）：
- 待批准请求等 owner 批准（5 分钟 TTL）；重连保留原 `ts` 防攻击者抢 `--latest` 批准竞态（`:373`）。
- **批准提权守卫**：授予 operator role 要求**批准者自身持有被授予的 scope**（`caller-missing-scope`，`:723`）。
- **有效 role 失败关闭**：无 token 的遗留记录授予零权限（`:269`）。
- token 校验（`:1052`）：配对+role存在+未撤销+常量时间匹配+签发代际新鲜+scope⊆批准基线（双重检查）。

**Node 自动批准**（`node-pairing-auto-approve.ts`）窄化：仅首次 node-role 配对、空 scope、非 loopback、匹配 `autoApproveCidrs` 白名单；升级/浏览器路径拒绝。system-run 命令的 node 批准需 `operator.admin`。

---

## 4. 沙箱（Sandbox）

**关键认知：沙箱 mode ≠ 权限等级。** mode 控制「哪些会话被沙箱化」，权限由独立轴控制。

**mode**（`src/agents/sandbox/types.ts:77`）：`off`（默认）/ `non-main`（除主会话外都沙箱）/ `all`。门控 `shouldSandboxSession`（`runtime-status.ts:22`）。

**权限轴（独立）**：
- `workspaceAccess: none(默认)|ro|rw`（`types.ts:36`）—— 最接近「只读 vs 可写」。
- exec `security: deny|allowlist|full`（见 §5）。
- **无单一「danger-full-access」开关**；`dangerously*` Docker 标记是显式逃生口。

**后端**（`backend.ts:36`）：`docker`（默认）+ `ssh`，插件可注册自定义。

**Docker 隔离硬化（默认开）**（`docker.ts:393-519`）：
- `--security-opt no-new-privileges` **始终**加。
- `--cap-drop ALL`（默认）。
- `--read-only` 根文件系统（默认）。
- `--network none`（默认）。
- 可选 `--user`/`--pids-limit`/`--memory`/`--cpus`/seccomp/apparmor。
- **bind mount 限制**：阻止挂载保留容器路径 + 主机路径**硬黑名单**（`/etc`/`/proc`/`/sys`/`/dev`/`/root`/Docker socket/`.ssh`/`.aws`/`.docker`/`.gnupg` 等，`validate-sandbox-security.ts:23`）。

**路径限制（分层）**（`sandbox-paths.ts`）：词法逃逸检查（拒 `..`/绝对/盘符）→ 符号链接/硬链接别名逃逸断言 `assertNoPathAliasEscape` → 远程/容器 FS 规范路径再校验。媒体入口白名单仅限 tmp/托管媒体/沙箱根。

---

## 5. 工具策略与 exec 安全

### 5.1 工具策略管线（严格相减，~11 层）
`applyToolPolicyPipeline`（`tool-policy-pipeline.ts:127`）按序应用，**每层只能移除工具（交集语义），无层可重新授予**：
1. profile → 2. provider profile → 3. global allow → 4. global provider → 5. agent → 6. agent provider → 7. group → 8. sender → 9. **sandbox**（仅沙箱时）→ 10. **subagent**（仅子代理）→ 11. **inherited**（父子代理继承）。

**匹配规则**（`tool-policy-match.ts:9`）：**deny 永远胜**；**空 allow = 允许全部未被 deny 的**（注意这是有意保留的 footgun 语义）。管线注释锚定已修复漏洞 **GHSA-mhm4-93fw-4qr2**（`tool-dispatch.ts:54`）。

### 5.2 exec/bash 安全
- **三级 security**：`deny|allowlist|full`，ask 级 `off|on-miss|always`（`exec-approvals.ts:38`）。
- **host-target 门控**：`host=auto` 且沙箱可用时，agent **无法**改 `gateway`/`node` 逃出沙箱（`bash-tools.exec-runtime.ts:215`）。
- **审批仅能收紧不能放松**：approval 文件只能 `minSecurity`/`maxAsk`，永不放松配置策略（`exec-defaults.ts:199`）。
- **node-host system.run**（`node-host/exec-policy.ts:55`）：deny→硬阻；ask=always 或 allowlist miss→需审批；Windows `cmd.exe /c` 包装强制审批（语义不同）。
- **allowlist 分析 + argv 重写**：批准后可把 shell 命令重写为更安全的 argv，但**仅当重建命令仍满足策略**（`invoke-system-run-allowlist.ts:137`）。

### 5.3 exec 审批（human-in-the-loop）
agent 请求审批 → 经网关分发 → agent 阻塞在 `exec.approval.waitDecision`（超时）。审批方法需 `operator.approvals` scope；改审批**策略**需 `operator.admin`（查看/解决与改策略分离）。支持 allow-always 持久决策、safe-bin 分类。审批 socket = Unix socket + bearer token（`exec-approvals.ts:262`）。

---

## 6. 密钥管理（Secrets）

**Secret 引用模型**（`src/config/types.secrets.ts:21`）：`SecretInput = string | SecretRef`，`SecretRef = {source: "env"|"file"|"exec", provider, id}`。**无 plaintext/ref 布尔开关 —— 值的形状即模式**。字面量字符串=明文（审计标 `PLAINTEXT_FOUND`），SecretRef=引用。严格读点遇未解析 ref 抛 `UnresolvedSecretInputError`。

**存储**：渠道/provider 凭证 `~/.openclaw/credentials/`；模型 auth profile `~/.openclaw/agents/<id>/agent/auth-profiles.json`（SQLite）；设备私钥 `<stateDir>/identity/device.json`。

**文件保护**：私有文件存储 `private:true`（owner-only）；secrets 写用 `0o700` 目录 + `0o600` 文件；exec-approvals 用 `O_RDWR|O_NOFOLLOW`（防符号链接攻击）+ `0o600`。

**运行时解析**（`src/secrets/resolve.ts`）每 ref 先语法校验再解析：
- **env**：可选 allowlist；缺失抛错。
- **file**：路径必须绝对、非符号链接、当前用户拥有、非组/世界可写（`assertSecurePath`），1MB 上限。
- **exec**：同安全路径检查；`shell:false`（无注入）；空/allowlist env；1MB stdout 上限 + 双超时；**需 `--allow-exec` 显式同意**（`exec-resolution-policy.ts`）。

**header 凭证策略**（`model-provider-header-policy.ts`）：保守分类哪些 provider 请求头携凭证（`authorization`/`x-api-key`/含 `token|secret|password` 等），「宁可误报也不漏 key」。

---

## 7. 密钥审计与日志脱敏（「日志绝不打印密钥」的强制机制）

**审计**（`openclaw secrets audit`，`src/secrets/audit.ts`）：稳定码 `PLAINTEXT_FOUND|REF_UNRESOLVED|REF_SHADOWED|LEGACY_RESIDUE`；扫描 config/auth 库/`models.json`/遗留 `auth.json`/`.env`；**findings 只含路径/provider 元数据，绝不含解析后的密钥值**。

**日志脱敏**（`src/logging/redact.ts`）—— 这是「日志不打印密钥」的执行点：
- 默认 `"tools"` 模式（开）。默认模式集覆盖 env/CLI/JSON/header 赋值、URL query、URL userinfo、连接串密码、PEM 私钥块、**~80 个厂商 token 前缀**（`sk-`/`ghp_`/`xox[baprs]-`/`AKIA`/JWT 等）。
- **抗混淆**：处理百分号编码、隐形字符夹杂的 key、Hangul filler；base64 token 用非 base64 左边界避免污染 data-URL。
- **递归结构脱敏**：按敏感 key 名遍历对象。
- **`redactToolPayloadText` 强制 tools 模式**（无视日志配置），保证工具 UI 面始终脱敏（`:1012`）。
- prefilter 注释警告：缺触发器会**静默泄漏**一种形态 —— 每族都留测试 fixture。

**URL 凭证脱敏**（`packages/net-policy/src/redact-sensitive-url.ts`）：userinfo → `***`，~28 个敏感 query 参数名掩码。

**工具结果细节脱敏（绝不进 LLM）**（`session-tool-result-guard.ts`）：持久化/发模型前，限制超大结果 + `redactToolPayloadTextWithConfig` + 敏感字段脱敏。

---

## 8. SSRF / 网络策略

**IP 分类**（`packages/net-policy/src/ip.ts`，用 `ipaddr.js`）：
- **默认阻止所有 RFC 1918 私有段** + unspecified/broadcast/multicast/linkLocal/loopback/CGNAT/reserved + RFC2544 benchmark。
- IPv6：loopback/linkLocal/uniqueLocal/multicast/reserved 等。
- **云元数据地址显式阻止**：`100.100.100.200`、`fd00:ec2::254`、`metadata.google.internal`。
- **抗解析器绕过**：拒绝 legacy/八进制/十六进制 IPv4 简写；解码 IPv6 过渡前缀（6to4/Teredo/NAT64/ISATAP/IPv4-mapped）内嵌的 IPv4 再检查 —— 修复 GHSA `ipv6-transition-special-use-ssrf-guard-bypass`。

**SSRF 守卫**（`src/infra/net/ssrf.ts`）：
- **两阶段检查 + DNS pinning**：DNS 前检查字面量 → DNS 应答再检查（公开主机名不能解析到私有目标）→ pinned lookup 防 DNS-rebinding TOCTOU。
- **失败关闭**：畸形 IPv6 字面量视为私有/阻止。
- 阻止主机名：`localhost`/`*.localhost`/`*.local`/`*.internal`。
- **逃生口全部显式命名 dangerous**：`dangerouslyAllowPrivateNetwork` 等；即使允许私网，元数据/link-local 的 DNS-rebind 目标仍阻止。
- `allowedOrigins` 仅提升当前请求主机名，**重定向逐 URL 重新评估**（防重定向提权）。
- 明文 HTTP 限私有/内部目标，除非显式 opt-in。

---

## 9. 输入信任边界

**入站信封脱敏**（`src/auto-reply/envelope.ts:60`）：`sanitizeEnvelopeHeaderPart` 剥 CR/LF、折叠空白、中和 `[`/`]`→`(`/`)`，使攻击者控制的 sender/host/IP 元数据无法突破 `[channel from host ip ts]` 前缀伪造信封结构。

**prompt-injection 守卫**：
- `sanitize-for-prompt.ts`：剥 Unicode Cc/Cf 控制+格式字符（威胁模型 OC-19：目录名注入指令）；`wrapUntrustedPromptDataBlock` 用 `<untrusted-text>` 标签 + HTML 转义包裹。
- `src/security/external-content.ts`（**最强边界**）：`wrapExternalContent` 用**随机 8 字节 id 边界标记**（`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>`）包裹 email/webhook/API/browser/web 内容，前置详细 SECURITY NOTICE；**净化伪造标记**（跨空白/下划线/同形字/零宽/全角变体折叠）；**剥 ~25 个 LLM 特殊 token 字面量**（ChatML `<|im_start|>`/Llama `<|eot_id|>`/Mistral `[INST]` 等）防伪造聊天模板控制 token。

---

## 10. 插件安全

**信任分级与溯源**（`loader-provenance.ts`）：重复候选信任排名 `config(0) > 开发根 bundled(1) > 显式全局安装(2) > bundled(3) > workspace(4) > 其他(5)`；无安装记录/load-path 溯源的插件标「视为未跟踪本地代码」+ 警告，建议 `plugins.allow` 钉住。

**安装策略与扫描（全程失败关闭）**（`src/security/install-policy.ts` + `install-security-scan.runtime.ts`）：
- 操作者可配 `security.installPolicy.exec` 外部策略命令（`allow|block`），**缺配置/空输出/超时/非零退出/block 全部阻止安装**；策略命令本身过安全路径检查 + `shell:false`。
- `before_install` 插件钩子可额外阻止（失败关闭）。
- **依赖黑名单扫描**：遍历清单/`node_modules`（限深度），解析校验符号链接目标在安装根内，阻止黑名单依赖。
- **源权威分类**：ClawHub=`openclaw`/不可变；git/npm=`third-party`；本地=`user`。

**ClawHub 溯源**（`clawhub.ts`）：SHA-256 归档完整性校验；回退归档校验有每文件/总大小/条目数上限（抗 zip-bomb）。

**代码插件信任边界**（AGENTS.md + opengrep 规则强制）：插件只经 `openclaw/plugin-sdk/*` barrel 跨入核心。

---

## 11. 静态分析（防回归火墙）

- `security/opengrep/precise.yml`：**147 条 precise 规则**，每条 id 为 `<source>.<original>`，GHSA 规则带 `advisory-url`/`ghsa`/`source-rule-id` 元数据（`pnpm check:opengrep-rule-metadata` 强制）。
- 规则**直接映射真实已修漏洞**，编码上述信任边界。示例：`prompt-unsanitized-literal-interpolation`、`channel-metadata-into-trusted-system-prompt`、`ipv6-transition-special-use-ssrf-guard-bypass`、`exec-allowlist-double-quoted-command-substitution`、`session-transcript-path-traversal`、`skill-env-host-injection`、`oauth-state-reuses-pkce-verifier`、`webhook-loopback-passwordless-fallback`。
- CI：`opengrep-precise.yml`（PR diff，`--error` 失败即拦）+ 全量扫描 + CodeQL（含 Android/macOS 关键安全）+ `security-sensitive-guard.yml`。
- `.semgrepignore` 是唯一排除真相源（仅 test/fixture/mock），README 警告「产品代码绝不应匹配此文件」。

---

## 12. 限流（Rate Limiting）

内存滑窗限流（`auth-rate-limit.ts`），按 `{scope, clientIp}`。默认 10 次/60s 窗/300s 锁定；**loopback 默认豁免**（本地 CLI 不被锁）。**分 scope**：`shared-secret`/`device-token`/`node-pairing`/`bootstrap-token`/`hook-auth` —— 一类凭证失败不锁另一类。bootstrap-token 单独 scope 因其 verify 是 mutex 串行 + fs 读写，防 DoS。**缺凭证不消耗额度**。浏览器 origin 请求即使 loopback 也用 origin-keyed 身份不豁免。

---

## 13. 安全设计总评与已知短板

### 强项
1. **强默认拒绝**：未分类方法需 admin；无设备客户端 scope 清空；scope 密码学绑定进设备签名。
2. **全链路反提权守卫**：批准者须持被授 scope、重连竞态保护、node 自动批准窄化、break-glass `dangerouslyDisableDeviceAuth` 仅 operator role。
3. **一致的常量时间比较**（`safeEqualSecret` 先填充等长再比，比后才拒长度不匹配以防泄漏长度）+ `O_NOFOLLOW`/0600 凭证文件。
4. **SSRF 全面**：私有/loopback/link-local/CGNAT/元数据/IPv6 过渡全默认阻止，失败关闭，抗 DNS-rebind，逃生口显式 dangerous。
5. **secrets 纵深防御**：语法校验 → 文件/exec 的 owner+权限+非符号链接+大小/超时 → exec `shell:false`+净化 env → exec 需 `--allow-exec` 同意。
6. **安装管线全程失败关闭** + 源权威分类。
7. **静态分析作回归火墙**：147 规则映射真实已修漏洞。

### 已知短板 / 硬化机会（诚实标注）
1. **🔸 MCP 输出信任缺口**：`details.untrustedMcpOutput:true` 已设（`agent-bundle-mcp-materialize.ts:125`）但**无运行时强制消费**（仅测试断言）。MCP server 输出**未经** `wrapExternalContent` 的边界标记/特殊 token 净化，而 web/email 内容经过了。MCP 输出直达模型 —— 是相对 web/email 强路径的硬化缺口。
2. **🔸 动态方法 scope 回退**：独立 CLI 进程看不到插件注册表时，动态方法回退授予含 admin 的 `CLI_DEFAULT_OPERATOR_SCOPES`（`method-scopes.ts:107`）—— 可用性优先于严格性的有意取舍，依赖调用者已是认证操作者。
3. **🔸 device-token 字段重载**：无显式 `deviceToken` 时回退用 `token` 字段试 device token（`auth-context.ts:118`）—— 校验路径仍要求匹配配对设备，不削弱信任，但是非显然的字段重载。
4. **🔸 限流仅进程内**：无分布式限流，多进程/重启重置计数。
5. **🔸 tailscale/trusted-proxy 无设备级凭证**：作「共享认证」允许跳过设备身份，安全完全依赖网络层代理信任配置。
6. **`allow: []` = 允许全部**：有意保留的 footgun 语义，配置失误可能意外放开。

### 一句话结论
OpenClaw 的安全是**「在受信任本地操作者模型下、对真实可达边界（auth/scope/approval/sandbox/tool）做纵深防御 + 静态分析回归火墙」**。它**不承诺**多租户对抗隔离与纯 prompt-injection 防护（明确划在威胁模型外）。在其定位内，安全工程成熟度高（常量时间比较、SSRF 抗绕过、失败关闭、Ed25519 设备绑定、147 条映射真实漏洞的静态规则）；主要硬化机会是 MCP 输出未走最强输入净化路径。
