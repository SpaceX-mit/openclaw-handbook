# OpenClaw Sandbox 功能与完成度评估（产品化/商业化视角）

> 调查目标：sandbox 的功能、完成度，是否符合产品化、商业化需求。
> 方法：源码级深查（核心 ~9272 LOC + 集成 ~3272 LOC + 59 个测试文件 + 4 篇文档）+ 2 路并行后端深查。证据以 `file:line` 标注。
> 一句话结论：**沙箱在「受信任本地操作者」定位下已是成熟、设计严谨、测试充分的功能性子系统；但要满足「多租户/不可信代码执行」级别的商业化需求，存在默认硬化不足、SSH 后端工程化偏薄、运维自动化缺失三类硬缺口。**

---

## 1. 沙箱要回答的第一个问题：它的定位

OpenClaw 沙箱**不是**用来「在对抗环境里安全执行不可信代码」的强隔离平台。代码注释直接写明威胁模型：**"local-trusted config, but protect against foot-guns and config injection"**（`validate-sandbox-security.ts:3-5`）。它的目标是：
- 把 agent 的文件/命令执行**限制在工作区**，防止误伤主机。
- 防御**配置注入 / 路径逃逸 / 符号链接攻击**等 foot-gun。
- 给 non-main 会话（如子代理、群组会话）一层隔离。

这与 SECURITY.md 的总威胁模型一致（不承诺多租户对抗隔离）。**评估完成度必须分两个标尺**：(A) 在其声明定位内是否完整；(B) 若要拔高到商业化「不可信多租户」需求还差什么。

---

## 2. 功能清单与完成度（标尺 A：声明定位内）

### 2.1 双后端架构 —— 完成度高
- **Docker 后端**（`docker.ts` 685 + `docker-backend.ts`）：真容器隔离。**默认硬化全开**：`--security-opt no-new-privileges`（始终、不可配，`docker.ts:470`）、`--cap-drop ALL`（默认，`config.ts:122`）、`--read-only` 根 fs（默认，`:118`）、`--network none`（默认，`:120`）、tmpfs 可写临时区。
- **SSH 后端**（`ssh.ts` 849 + `ssh-backend.ts`）：远程主机目录隔离（**非**进程/内核隔离），用于把执行放到远程盒子。
- **可插拔后端注册表**（`backend.ts`）：进程级 `Symbol.for` 全局 Map，`registerSandboxBackend` 返回 restore 回调，core 仅自动注册 docker/ssh，插件可加自定义后端 —— 契约中立（`BackendHandle`），符合插件无关核心架构。

### 2.2 文件系统安全 —— 完成度很高（最强项）
- **共享的 Python mutation helper**（`fs-bridge-mutation-helper.ts`，526 行，docker+ssh 共用）：`O_NOFOLLOW`+`O_DIRECTORY` dir-fd 遍历、拒 `..`、原子 temp+`os.replace`+fsync 写、硬链接拒绝（`st_nlink>1`→EPERM）、`EXDEV` 跨设备 copy+校验+删除并重检 inode 身份。**这是生产级、TOCTOU 感知的实现。**
- **bind-mount 安全校验**（`validate-sandbox-security.ts` 435 行）：主机路径硬黑名单（`/etc`/`/proc`/`/sys`/`/dev`/`/root`/`/boot`/`/run`/Docker socket 全别名）+ HOME 子路径黑名单（`.ssh`/`.aws`/`.docker`/`.gnupg`/`.npm`/`.cargo`/`.config`/`.netrc`）+ 双向包含检查（拒绝覆盖黑名单的父挂载）+ 符号链接逃逸重检（解析最深存在祖先后再查）+ 保留目标守卫（`/workspace`/`/agent` 不可被 bind 遮蔽）。
- **路径限制三层**（见安全分析文档）：词法逃逸 → 别名逃逸断言 → 挂载再校验。
- **上传安全**（`ssh.ts:790`）：tar 前遍历拒绝逃逸符号链接；远程解包前 `ENSURE_REMOTE_REAL_DIRECTORY_SCRIPT` 拒非绝对/`..`/符号链接根。

### 2.3 网络隔离 —— 完成度高
默认 `network=none`；`host` 模式始终阻止；`container:*` 命名空间 join 默认阻止（需 `dangerouslyAllowContainerNamespaceJoin`）。注意：裸 `bridge`（全出站）若操作者显式配置则允许 —— 仅 `host`/`container:` 被强制管控。

### 2.4 沙箱化浏览器 —— 完成度高（设计精细）
`browser.ts`（541 行）：独立镜像 + 契约 epoch 校验、专用 bridge 网络、**CDP 随机 24 字节 token 认证 + 仅 loopback 端口绑定**、noVNC **60s 单次性 observer token**（密码不进 URL）、host 侧 bridge 应用 SSRF 策略。Chrome 以 `--no-sandbox` 跑（依赖容器隔离）。默认 `enabled=false`。

### 2.5 配置面 —— 完成度高
`types.sandbox.ts` 暴露丰富配置：镜像/workdir/网络/user/capDrop/env/setupCommand/pidsLimit/memory/cpus/gpus/ulimits/seccomp/apparmor/dns/extraHosts/binds + 3 个 `dangerously*` 显式逃生口。SSH 侧：target/key/cert/known_hosts（file/inline 三态）/strictHostKeyChecking。

### 2.6 生命周期与运维 —— 完成度中
- **scope**：`session`/`agent`/`shared`（`shared.ts:32`）。
- **容器复用 via config-hash**（仅 Docker）：配置变更检测 → 不匹配则 `rm -f` 重建；「热容器」（5 分钟内用过）保留并仅记日志。
- **SQLite 注册表**（`registry.ts`，符合 SQLite-only 策略，含遗留 JSON 迁移）。
- **剪枝**（`prune.ts`）：idle（默认 24h）/age（默认 7d）；**惰性触发**（会话解析时，5 分钟节流），非定时调度。
- **媒体进沙箱**（`stage-sandbox-media.ts`，364 行）：把媒体安全暂存进沙箱路径。

### 2.7 测试与文档 —— 完成度高
- **59 个测试文件**（32 在 sandbox/ 目录内），覆盖 validate-security、host-paths、config-hash 重建、bind-spec、browser create、fs-bridge e2e、ssh、remote-fs-bridge 边界等。
- **4 篇文档**：`docs/gateway/sandboxing.md`（546 行）、`docs/cli/sandbox.md`、`docs/gateway/sandbox-vs-tool-policy-vs-elevated.md`。
- **零 TODO/FIXME/HACK** 标记（两后端文件全清），版本化契约（mount format v3、browser image epoch、env policy epoch）—— 真实迭代与升级纪律。

**标尺 A 结论：在「受信任本地操作者」定位内，沙箱功能完整、设计严谨、测试充分、文档齐全 —— 是一个成熟的功能性子系统。**

---

## 3. 完成度缺口（标尺 B：商业化/不可信多租户视角）

### 3.1 默认硬化不足（最关键，影响安全商业化）
| 缺口 | 证据 | 商业化影响 |
|---|---|---|
| **无默认非 root 用户** | `--user` 默认 undefined（`config.ts:121`），容器以镜像默认用户（通常 root）运行；无 user namespace remap | root-in-container + cap-drop ALL 有缓解，但非纵深防御；不可信代码场景不达标 |
| **无默认资源限制** | memory/cpus/pidsLimit/ulimits 全默认 undefined | **开箱即遭 fork-bomb/OOM/CPU 耗尽主机 DoS** —— 多租户致命 |
| **seccomp/apparmor 仅用 Docker 默认** | 仅拒 `unconfined`，不附自定义硬化 profile | 系统调用面未收紧 |
| **`allow:[]` = 允许全部** | 工具策略 footgun | 配置失误意外放权 |

→ **结论：默认姿态是「防误伤」级，不是「防恶意」级。商业化要跑不可信代码，必须先把非 root、资源限制、自定义 seccomp 设为安全默认。**

### 3.2 SSH 后端工程化偏薄（影响远程执行商业化）
| 缺口 | 证据 |
|---|---|
| **无连接复用/多路复用** | 每次 exec/FS 守卫/mutation 都 spawn 新 ssh 进程；一次 writeFile ≈ 3-4 次握手；无 `ControlMaster`/`ControlPersist` | 
| **无重连/退避** | 首次非零退出即 reject；瞬时网络抖动直接报错 |
| **无 config-drift 重建** | Docker 有 config-hash 重建，**SSH 仅靠远程目录存在性复用**，配置/工作区变更后静默复用陈旧副本（`ssh-backend.ts:221`，`configHash` 字段存在但 SSH 路径从不设置/比对） |
| **SecretRef 不支持** | `identityData`/`knownHostsData` 经 `normalizeSecretInputString` 只接受字面量，`SecretRef`（env/file/exec）被静默丢弃 |
| **无浏览器支持** | SSH 后端不广告 `capabilities.browser`，浏览器沙箱抛错 |
| **无远程并发锁** | 两进程共享 `shared` scope 可竞争同一 `runtimeRootDir` |
| **远程依赖未预检** | 需远程 `python3`/`tar`/`stat -c`/`readlink -f`（GNU 语义），失败是运行时而非 setup 时 |

→ **结论：SSH 后端「安全默认正确但运维薄」。最高价值改进：(1) ControlMaster 多路复用消除握手风暴，(2) configHash 接入 SSH 复用，(3) SecretRef 解析。**

### 3.3 运维自动化缺失（影响交付商业化）
| 缺口 | 证据 |
|---|---|
| **无镜像管理** | `ensureDockerImage` 从不 pull/build（`docker.ts:326`），要求操作者跑 `scripts/sandbox-setup.sh`；而**该脚本未随 npm 包发布**（`docs/gateway/sandboxing.md:383`）→ npm 用户须手动构建 |
| **无 rootless Docker/Podman 支持** | 无检测；`--gpus`/`--user`/bind UID 映射在 rootless 下行为不同 |
| **无多架构** | 镜像 tag 硬编 `:bookworm-slim`，create 无平台 pin |
| **剪枝仅惰性** | 空闲主机不再收到会话则永不剪枝；节流 per-process，重启丢失 |
| **bridge 网络泄漏** | browser bridge 网络创建后无清理空网络 |
| **热容器陈旧** | 配置变更但容器近用则**保留陈旧容器**仅记日志（`docker.ts:646`）—— 配置漂移可静默持续 |

---

## 4. 产品化/商业化成熟度评分

| 维度 | 评分(100) | 说明 |
|---|---:|---|
| **功能完整性** | 88 | 双后端 + FS 安全 + 网络隔离 + 浏览器 + 配置面 + 剪枝，覆盖全；SSH 缺浏览器/config-drift |
| **隔离强度（声明定位内）** | 85 | Docker 默认硬化全开 + FS 安全生产级；扣分于无默认非 root/资源限制 |
| **隔离强度（不可信多租户）** | 55 | 默认姿态防误伤非防恶意；无资源限制=DoS 风险；root-in-container |
| **工程质量** | 90 | 零 TODO、版本化契约、59 测试、强注释、显式 dangerously 守卫 |
| **运维/可交付性** | 60 | 无镜像自动管理、无 rootless/Podman/多架构、剪枝惰性、npm 缺构建脚本 |
| **SSH 后端成熟度** | 62 | 安全默认正确，但无连接复用/重连/config-drift/SecretRef |
| **文档** | 85 | 4 篇文档 + 代码内威胁模型 |

**综合（按定位加权）**：
- **作为「受信任本地操作者的工作区隔离」**：**85/100 —— 已达产品化标准**，可随产品发布。
- **作为「不可信代码/多租户安全执行」商业化平台**：**约 60/100 —— 不达标**，需补默认硬化 + 资源限制 + 运维自动化。

---

## 5. 是否符合产品化/商业化需求 —— 直接回答

### 符合的场景 ✅
- **个人/团队自托管助理的工作区隔离**：完全符合。Docker 默认硬化、FS 安全、路径限制、剪枝、配置面、文档、测试都达产品级。可直接作为「让 agent 在容器里安全干活、不碰主机敏感路径」的产品功能发布。
- **把执行下放到受信任远程机**（SSH 后端）：功能可用、安全默认正确，适合内部受控场景。

### 不符合 / 需补强的场景 ⚠️
- **跑不可信/第三方代码的多租户 PaaS**：**不符合**。三块硬缺口必须先补：
  1. **安全默认**：默认非 root 用户 + user namespace、默认资源限制（memory/cpus/pids/ulimits）、自定义 seccomp/apparmor profile。
  2. **DoS 防护**：资源限制设为强制默认（当前全 opt-in = 开箱可被 fork-bomb 打挂主机）。
  3. **运维自动化**：镜像自动 pull/build + 随包发布构建脚本、rootless/Podman 支持、多架构、定时剪枝。
- **大规模远程执行（SSH）**：需补 ControlMaster 多路复用（否则握手风暴）、重连退避、config-drift 重建、SecretRef。

### 商业化优先级建议（投入产出排序）
| 优先级 | 改进 | 价值 |
|---|---|---|
| **P0** | 资源限制安全默认（memory/cpus/pids） | 消除开箱 DoS，多租户前提 |
| **P0** | 默认非 root 用户 + user namespace | root-in-container 纵深防御 |
| **P1** | 镜像自动管理 + 构建脚本随包发布 | 消除「装完不能用」的交付摩擦 |
| **P1** | 自定义 seccomp/apparmor 硬化 profile | 收紧系统调用面 |
| **P2** | SSH ControlMaster 多路复用 + config-drift 重建 | 远程执行性能与正确性 |
| **P2** | rootless Docker/Podman + 多架构 | 部署环境覆盖 |
| **P3** | 定时剪枝（替代惰性）+ 网络/资源泄漏清理 | 长期运维健康 |

---

## 6. 终评

OpenClaw 沙箱是一个**工程质量高、在其声明威胁模型内成熟可发布**的子系统 —— 文件系统安全层（符号链接/硬链接/遍历抗性、原子 mutation、挂载边界）甚至达到生产级水准，bind-mount 校验与浏览器 token 认证设计精细，零技术债标记，测试充分。

但它的**默认姿态是「防 foot-gun」而非「防恶意」**，且**运维自动化与 SSH 后端工程化偏薄**。因此：
- **作为产品功能**（受信任操作者工作区隔离）：**已达标，可商业化发布**。
- **作为安全卖点**（不可信代码/多租户隔离平台）：**尚不达标**，需先补默认硬化（非 root + 资源限制 + 自定义 seccomp）和运维自动化，否则开箱即存在主机 DoS 与 root-in-container 风险。

这与项目自身定位完全自洽 —— OpenClaw 从未声称多租户对抗隔离（SECURITY.md 明确排除）。所以「是否符合商业化需求」取决于商业化方向：**做受信任个人/团队助理 → 符合；做不可信多租户执行平台 → 需按上表 P0/P1 补强。**
