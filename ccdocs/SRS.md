# OpenClaw 软件需求规格说明书（SRS）

> 文档类型：软件需求规格说明书（Software Requirements Specification）
> 结构：敏捷/精简 SRS（Epic → Feature → 用户故事 + 验收标准）+ 模块级功能需求（FR-ID）
> 目标系统：OpenClaw v2026.6.9（个人 AI 助理运行时，~59 万行 TypeScript）
> 证据基线：源码级分析（配套 `ccdocs/` 各分析文档 + `architecture.svg`），需求以实现事实为依据，以 `file:line` 佐证。
> 版本：1.0 · 状态：基线（Baseline）

---

## 目录

1. [引言](#1-引言)
2. [总体描述](#2-总体描述)
3. [分层模块与功能清单](#3-分层模块与功能清单)
4. [功能需求](#4-功能需求)（见「功能需求」章）
5. [非功能需求](#5-非功能需求)（见 NFR 章）
6. [数据需求](#6-数据需求)
7. [外部接口需求](#7-外部接口需求)
8. [需求可追溯矩阵](#8-需求可追溯矩阵)
9. [附录](#9-附录)

---

# 1. 引言

## 1.1 目的
本 SRS 定义 OpenClaw 的功能与非功能需求，供以下角色使用：
- **研发团队**：据以实现/重构模块（配合 `ccdocs/agent-core/` 等重写级规格）。
- **测试团队**：据以设计验收测试（每条 FR 附验收标准）。
- **产品/架构**：据以评估完成度、规划演进（配合各 `*-assessment.md`）。
- **安全审计**：据以核对安全边界（配合 `security-analysis.md`）。

本文档是**逆向工程 SRS** —— 从已实现系统提炼需求，而非前瞻设计。因此需求均可追溯到源码。

## 1.2 范围
- **产品名**：OpenClaw（CLI/包/路径/配置标识为 `openclaw`）。
- **一句话定位**：用户在自有设备上运行的、接管其现有消息渠道的、能执行真实任务的个人 AI 助理运行时。
- **范围内**：Gateway 控制平面、Agent 运行时与循环引擎、渠道接入、工具/MCP/技能、记忆、沙箱、插件系统、模型接入、CLI/UI/SDK、跨设备节点、调度。
- **范围外**：LLM 模型本体（外部依赖）、ClawHub 市场后端（独立仓库）、伴侣 App 原生实现细节（本 SRS 只到其与 Gateway 的接口）。

## 1.3 术语与缩写
| 术语 | 定义 |
|---|---|
| **Gateway** | 控制平面进程，WebSocket 服务端，承载认证/路由/RPC/会话 |
| **Agent** | 一个可执行任务的 AI 实体；一次运行=一个 agent run |
| **Harness** | (a) agent-core 库级会话外壳 CoreAgentHarness；(b) 运行时可插拔执行后端契约 |
| **Session（会话树）** | append-only 树形转写存储，含 parentId + leaf 指针 |
| **Compaction（压缩）** | 上下文超窗时把旧历史结构化摘要替换 |
| **ReAct** | Reasoning+Acting，思考→行动→观察循环 |
| **Tool** | 模型可调用的函数（schema + execute） |
| **MCP** | Model Context Protocol，工具互操作协议 |
| **Skill** | SKILL.md 定义的、模型按需读取的指令包 |
| **Channel** | 消息渠道 transport-only 适配器 |
| **Memory Slot** | 单一激活的记忆插件槽 |
| **Sandbox** | 工具执行的隔离环境（Docker/SSH 后端） |
| **Scope** | 操作者权限范围（operator.admin/read/write/...） |
| **Node（节点）** | 连接 Gateway、执行远程命令的设备 |
| **FR / NFR** | 功能需求 / 非功能需求 |

## 1.4 文档约定
- 需求 ID：`FR-<域>-<序号>`（功能）、`NFR-<类>-<序号>`（非功能）。
- 优先级：**M**(必须/Must)、**S**(应该/Should)、**C**(可以/Could)。
- 每条需求含：描述 + 验收标准 + 源码佐证（`file:line`）。
- 关键字「必须/应/可」遵循 RFC 2119 语义。

## 1.5 参考
- 配套分析：`ccdocs/README.md`（21 部分深度分析）、`modules-and-architecture.md`、`architecture.svg`、`agent-core/`（12 份）、`security-analysis.md`、`sandbox-assessment.md`、`memory-assessment.md`。
- 项目文档：`AGENTS.md`（硬策略）、`VISION.md`、`SECURITY.md`、`docs/`。

---

# 2. 总体描述

## 2.1 产品愿景
「The AI that actually does things. It runs on your devices, in your channels, with your rules.」（`VISION.md:3`）。核心价值 = 全渠道广度 × 真实执行深度 × 生产级可靠性 × 系统化可扩展。

## 2.2 用户画像（Persona）
| Persona | 描述 | 关键需求 |
|---|---|---|
| **P1 极客/自托管者** | 在自有机器/NAS 跑常驻服务，用终端 | 易 setup、可靠常驻、可观测 |
| **P2 多渠道用户** | 活跃于多个 IM，要一个跨渠道助理 | 全渠道接入、统一体验、路由 |
| **P3 插件/工具开发者** | 给助理加能力并发布 | 稳定插件 API、扩展点、边界清晰 |
| **P4 隐私敏感用户** | 不愿数据/凭证上云 | 本地存储、凭证保护、沙箱 |
| **P5 编码 Agent 用户** | 把 Codex 等当后端驱动 | 可插拔 harness、审批中继 |

## 2.3 运行环境
- **运行时**：Node ≥22.19（推荐 24），保留 Bun 路径。
- **存储**：SQLite（Kysely + node:sqlite），本地文件。
- **部署**：npm/pnpm/bun、Docker、Nix、Fly、Render；守护进程 launchd/systemd/schtasks。
- **客户端**：23+ IM 渠道、Web UI（Lit）、伴侣 App（macOS/iOS/Android/Windows）、CLI、SDK。

## 2.4 分层架构总览
系统分 10 层（详见 `architecture.svg`）：接入层 → 控制平面(Gateway) → 路由 → Agent 运行时 → 循环引擎(agent-core) → 模型层/工具层 → 支撑层 → 基础设施 → 插件系统 → 外部依赖。控制流：入站消息 → Gateway(认证) → 路由 → Agent 运行时 → 循环引擎 → 模型/工具 → 回复投递。

## 2.5 设计约束（来自 AGENTS.md）
- **C1**：核心保持插件无关；能力经 `openclaw/plugin-sdk/*` 跨入。
- **C2**：运行时只消费 canonical 配置；遗留形态仅在 doctor 迁移归一化。
- **C3**：存储默认 SQLite；不新增 JSON sidecar 运行时状态。
- **C4**：无静态+动态同模块 import；用 `*.runtime.ts` 懒边界；保持无循环依赖。
- **C5**：TS ESM strict，避免 `any`。
- **C6**：兼容性 opt-in；破坏性变更需版本化 + doctor 迁移。

## 2.6 假设与依赖
- **A1**：用户是受信任的本地操作者（威胁模型前提，`SECURITY.md`）—— 系统不承诺多租户对抗隔离。
- **A2**：LLM provider 可用（外部）；无 provider 时部分能力降级。
- **A3**：Docker 可用时沙箱 Docker 后端可用；否则需 SSH 后端或关闭沙箱。
- **A4**：渠道平台 API/ToS 稳定（外部风险）。

---

# 3. 分层模块与功能清单

> 目标1：分层模块梳理。下表为权威模块清单（LOC 已核实），按重要等级 P0(骨架)→P3(可选)。完整依赖关系见 `modules-and-architecture.md`。

## 3.1 模块清单（代码对应 + 等级 + 功能）

| 等级 | 模块 | 代码位置 | LOC | 核心功能 |
|---|---|---|---:|---|
| P0 | Agent 运行时 | `src/agents/` | 278K | 循环编排、会话、工具、子代理、系统提示、重试/压缩 |
| P0 | 循环引擎 | `packages/agent-core/` | 8.1K | ReAct 循环 + 会话树 + 压缩（provider 无关） |
| P0 | Gateway 控制平面 | `src/gateway/` | 114K | WS 服务、认证、RPC 分发、会话方法、热重载 |
| P0 | 网关协议 | `packages/gateway-protocol/` | 8.4K | Protocol v4 wire 契约（TypeBox + 惰性编译） |
| P0 | 插件加载器 | `src/plugins/` | 93K | manifest-first 发现/激活/加载/注册表 |
| P0 | LLM 契约+分发 | `packages/llm-core`+`llm-runtime`+`src/llm/` | 14.8K | 消息/模型/事件流契约 + api-id 分发 + adapter |
| P1 | 入站分发 | `src/auto-reply/` | 71K | 门控→去抖→分发→回复投递 |
| P1 | 渠道抽象 | `src/channels/` | 38K | transport-only 适配 + 可移植动作 + 入站归一化 |
| P1 | 配置系统 | `src/config/` | 55K | canonical schema/校验/类型 |
| P1 | 基础设施 | `src/infra/` | 95K | SQLite 状态、system-events、heartbeat、迁移 |
| P1 | CLI | `src/cli/`+`src/commands/` | 147K | Commander 入口 + 命令（onboard/doctor/...） |
| P1 | 记忆系统 | `memory-host-sdk`+扩展 | 56.9K | 三级记忆 + 混合检索 + dreaming（单槽） |
| P2 | ACP | `src/acp/`+`acp-core` | 13K | 驱动外部 agent 后端（Codex） |
| P2 | 路由 | `src/routing/` | 1.8K | 入站→agentId/sessionKey 解析 |
| P2 | 上下文引擎 | `src/context-engine/` | 2K | 可插拔上下文管理 + quarantine 容错 |
| P2 | 沙箱 | `src/agents/sandbox/` | 9.3K | Docker/SSH 隔离 + FS 安全 |
| P2 | 调度 cron | `src/cron/` | 19K | SQLite 定时唤醒 |
| P2 | 技能 | `src/skills/` | 15K | SKILL.md 加载/广告/分发 |
| P2 | MCP | `src/mcp/`+bundle-mcp | 1.5K | MCP 双向（server+client） |
| P2 | 密钥 | `src/secrets/` | 14K | 凭证管理 + secret refs |
| P2 | 模型目录 | `model-catalog`+core | 2.3K | 模型合并权威 |
| P2 | 工具调用修复 | `tool-call-repair` | 2.3K | 修复文本泄漏工具调用 |
| P3 | 守护/节点/SDK/UI/媒体/网络策略 | 各处 | — | 服务管理/跨设备/编程接口/控制台/媒体/SSRF |
| P3 | 扩展 | `extensions/` (139) | 180K | 70 provider + 25 channel + 2 memory + 42 其他 |

## 3.2 模块功能矩阵（能力域 → 模块）
| 能力域 | 承载模块 |
|---|---|
| 对话循环 | agent-core + embedded-agent-runner |
| 多渠道接入 | channels + extensions/&lt;channel&gt; + auto-reply + routing |
| 工具执行 | agents/tools + mcp + skills + 工具策略管线 |
| 长期记忆 | memory-host-sdk + memory-core/lancedb/wiki/active-memory |
| 隔离执行 | sandbox（Docker/SSH） |
| 模型接入 | llm-core/runtime + src/llm + extensions/&lt;provider&gt; |
| 扩展能力 | plugins + plugin-sdk + extensions |
| 控制/管理 | gateway + cli + config + secrets + doctor |
| 多 Agent | subagent-spawn + acp + sessions_send |
| 跨设备 | node-host + gateway node-registry |
| 定时/主动 | cron + heartbeat-wake |

---

# 4. 功能需求

> 组织：12 个 Epic，每个含 Feature → FR-ID（描述+验收+源码佐证）。优先级 M/S/C。

## Epic 1：Agent 对话循环（EP-LOOP）

**用户故事**：作为用户，我发一条消息，助理能多轮思考、调用工具、返回结果，并能中途纠偏。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-LOOP-01 | M | 系统必须以 ReAct 循环处理 prompt：流式调模型→解析工具调用→执行→回灌→判停 | 单 turn 无工具时事件序为 agent_start→turn_start→message_*→turn_end→agent_end；含工具时插入 tool_execution_* | `agent-loop.ts:258` |
| FR-LOOP-02 | M | 助理消息含工具调用时必须执行工具并把结果回灌下一轮 | 工具结果作为 ToolResultMessage 进入上下文，触发续轮 | `agent-loop.ts:540` |
| FR-LOOP-03 | M | 工具执行必须支持串行/并行；任一工具声明 sequential 则整批串行 | 并行模式 tool_execution_end 按完成序、result 按源序；含 sequential 则串行 | `agent-loop.ts:665,600` |
| FR-LOOP-04 | M | 工具执行前必须经 beforeToolCall 钩子（可 block），执行后经 afterToolCall（可改写） | block 返回错误工具结果；afterToolCall 字段级覆盖 | `agent-loop.ts:833,958` |
| FR-LOOP-05 | M | 系统必须支持 steering（中途注入）与 follow-up（停下后追加）两类队列 | steering 每轮工具后注入；follow-up 在无工具无 steering 时注入续跑 | `agent-loop.ts:413,419` |
| FR-LOOP-06 | M | 循环必须支持中止；中止时持久化 aborted 助手消息并补全事件序 | signal.aborted 时产出 stopReason=aborted 消息，事件序完整 | `agent-loop.ts:273` |
| FR-LOOP-07 | S | turn 间必须可换模型/上下文/thinking 级别（prepareNextTurn） | 返回的 model/context 应用到下一轮，reasoning 重算 | `agent-loop.ts:379` |
| FR-LOOP-08 | S | 整批工具结果都标 terminate 时循环必须停止 | shouldTerminateToolBatch 全 true → 停 | `agent-loop.ts:768` |
| FR-LOOP-09 | M | 流式失败必须编码进事件流而非抛出（StreamFn 契约） | 请求/模型失败产出 stopReason=error 消息，不抛异常 | `types.ts:201` |

## Epic 2：会话与上下文管理（EP-SESSION）

**用户故事**：作为用户，助理能记住本次会话历史、在超长时自动压缩、并支持回溯/分支。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-SESS-01 | M | 会话必须以 append-only 树形存储（parentId + leaf 指针） | 条目只增不改；leaf 移动=切换分支 | `harness/session/session.ts:106` |
| FR-SESS-02 | M | 系统必须能从会话树构建模型上下文（含压缩重放） | 无压缩全回放；有压缩=摘要+firstKeptEntryId 尾部+压缩后新增 | `session.ts:28` |
| FR-SESS-03 | M | 上下文超阈值必须触发压缩：contextTokens > window − reserveTokens(16384) | shouldCompact 返回 true | `compaction.ts:236` |
| FR-SESS-04 | M | 压缩必须保留约 keepRecentTokens(20000) 近期上下文，切点不切散工具调用/结果 | findCutPoint 吸附合法切点，排除 toolResult | `compaction.ts:388,313` |
| FR-SESS-05 | M | 压缩摘要必须用结构化模板（Goal/Progress/...）并保留文件操作列表 | 摘要含结构化段 + read/modified 文件 | `compaction.ts:441` |
| FR-SESS-06 | S | Token 估算必须优先用 provider 真实 usage，无则字符启发式(char/4) | estimateContextTokens 用 usage + 尾部估算 | `compaction.ts:205` |
| FR-SESS-07 | S | 系统必须支持分支导航（navigateTree）与被弃分支摘要 | moveTo 切叶 + 可生成 branch_summary | `agent-harness.ts:887` |
| FR-SESS-08 | M | 会话必须持久化到 SQLite（或 JSONL 存储实现） | 进程重启后可从叶恢复 | `harness/session/jsonl-storage.ts` |
| FR-SESS-09 | S | 压缩前必须支持 checkpoint 快照以便 fork/恢复（每会话≤25，128MB 上限） | 快照可读到 stopAfterEntryId | `gateway/session-compaction-checkpoints.ts` |

## Epic 3：多渠道接入（EP-CHANNEL）

**用户故事**：作为多渠道用户，我在任意支持的 IM 上都能与同一助理对话。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-CHAN-01 | M | 系统必须支持 23+ 消息渠道接入（Telegram/Discord/Slack/WhatsApp/iMessage/微信等） | 每渠道为 transport-only 适配器，25 个 channel 扩展 | `src/channels/`+`extensions/*` |
| FR-CHAN-02 | M | 渠道必须实现统一的可移植动作词表（~60 动作：send/reply/react/edit/poll...） | ChannelMessageActionAdapter 映射动作到原生 | `channels/plugins/message-action-names.ts` |
| FR-CHAN-03 | M | 渠道为纯传输层，不得拥有产品命令树或 provider 策略 | 命令/策略在核心/owner 插件，渠道只映射 | `channels/AGENTS.md` |
| FR-CHAN-04 | M | 入站消息必须归一化并分类为 user_request / room_event | classifyChannelInboundEvent 输出分类 | `inbound-event/classification.ts:26` |
| FR-CHAN-05 | M | 入站必须经门控（mention/command/allowlist）与去抖 | 未提及群消息按策略；快速同键消息批处理 | `mention-gating.ts`+`inbound-debounce.ts` |
| FR-CHAN-06 | M | 入站文本必须包 sanitized 信封头 `[channel from host ip ts]`（防注入） | CR/LF 剥离、`[]`中和 | `auto-reply/envelope.ts:60` |

## Epic 4：工具、MCP、技能（EP-TOOL）

**用户故事**：作为用户，助理能读写文件、执行命令、调用外部 MCP 工具、按需读取技能。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-TOOL-01 | M | 系统必须提供内置工具集（read/exec/process/edit/write/grep/find/ls/apply_patch 等 ~70） | 工具经 createOpenClawCodingTools 组装 | `agent-tools.ts:394` |
| FR-TOOL-02 | M | 工具参数必须经 TypeBox schema 校验 | validateToolArguments 校验 | `agent-loop.ts:873` |
| FR-TOOL-03 | M | 系统必须支持消费外部 MCP server（stdio/sse/streamable-http），工具物化为 AgentTool | tools/list → 物化，名字前缀防冲突 | `agent-bundle-mcp-materialize.ts:231` |
| FR-TOOL-04 | S | 系统必须能作 MCP server 暴露渠道会话/插件工具 | `openclaw mcp serve` 启动 | `src/mcp/channel-server.ts` |
| FR-TOOL-05 | M | 系统必须支持 SKILL.md 技能：广告给模型、模型按需 read 加载 | `<available_skills>` 列出，模型读文件加载 | `system-prompt.ts:269` |
| FR-TOOL-06 | M | 工具必须经策略管线过滤（profile/global/agent/group/sender/sandbox/subagent/inherited，严格相减） | 每层只减不增，deny 永胜 | `tool-dispatch.ts:210` |
| FR-TOOL-07 | S | 系统应支持延迟工具 schema 加载（Tool Search）+ 运行时水合 | resolveDeferredTool 按需水合 | `agent-loop.ts:805` |

## Epic 5：记忆系统（EP-MEMORY）

**用户故事**：作为用户，助理能跨会话记住我的偏好/决策/事实，并在需要时召回。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-MEM-01 | M | 系统必须支持单一激活的记忆插件槽（默认 memory-core） | 非选中 memory 插件禁用 | `config-activation-shared.ts:342` |
| FR-MEM-02 | M | 记忆必须支持混合检索：FTS5 关键词 + 向量，加权合并 | mergeHybridResults 合并 vector+keyword | `hybrid.ts:52` |
| FR-MEM-03 | S | 检索应支持可选时间衰减重排（默认关，halfLife 30d）与 MMR 多样性（默认关，λ0.7） | 显式开启后生效 | `temporal-decay.ts:11`,`mmr.ts:27` |
| FR-MEM-04 | M | 向量/embedding 不可用时必须优雅降级为纯 FTS 关键词 | 无 provider/sqlite-vec 时仍可检索 | `manager.ts:1210` |
| FR-MEM-05 | S | 系统应支持短期→长期 promotion（score≥0.75 & recall≥3 & 唯一查询≥2） | 满足门槛的片段晋升 MEMORY.md | `short-term-promotion.ts:47` |
| FR-MEM-06 | S | 长期记忆文件必须预算淘汰（≤10000 字符，优先丢最老自动 promotion 段） | 超限淘汰，保护用户手写 | `memory-budget.ts:25` |
| FR-MEM-07 | C | 系统可支持 dreaming 离线巩固（短期→长期叙事） | dreaming 管线运行 | `dreaming.ts` |
| FR-MEM-08 | M | 记忆索引必须持久化到每 Agent SQLite（FTS5 + sqlite-vec 表） | schema 建 7 张表 | `memory-schema.ts:7-13` |

继续见 Epic 6-12 →

<!-- APPEND_EPICS_6_12 -->

## Epic 6：隔离执行/沙箱（EP-SANDBOX）

**用户故事**：作为用户，助理执行命令/文件操作时被限制在工作区，不误伤主机。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-SBX-01 | M | 系统必须支持沙箱执行，后端可选 Docker（默认）/SSH，可插拔 | registerSandboxBackend；backend=docker 默认 | `sandbox/backend.ts:36` |
| FR-SBX-02 | M | 沙箱 mode 必须支持 off(默认)/non-main/all（控制哪些会话隔离） | shouldSandboxSession 按 mode 判定 | `sandbox/runtime-status.ts:22` |
| FR-SBX-03 | M | Docker 后端必须默认硬化：no-new-privileges(始终)+cap-drop ALL+read-only 根+network none | buildSandboxCreateArgs 含这些默认 | `sandbox/docker.ts:470,467,441,447` |
| FR-SBX-04 | M | 沙箱必须强制路径限制（拒 `..`/绝对逃逸 + 符号/硬链接别名逃逸） | resolveSandboxPath/assertNoPathAliasEscape | `sandbox-paths.ts:66,88` |
| FR-SBX-05 | M | bind-mount 必须校验主机路径黑名单（/etc,/proc,/sys,Docker sock,.ssh,.aws...） | validateSandboxSecurity 拒黑名单 | `validate-sandbox-security.ts:23` |
| FR-SBX-06 | S | 沙箱化浏览器必须 CDP token 认证 + 仅 loopback 端口 + noVNC 单次 token | 随机 token，端口 127.0.0.1 | `sandbox/browser.ts` |
| FR-SBX-07 | S | 沙箱必须支持 idle/age 剪枝（默认 24h/7d） | maybePruneSandboxes 惰性触发 | `sandbox/prune.ts` |
| FR-SBX-08 | C | env 注入容器前必须净化（阻 null 字节，警告大值/base64 凭证） | sanitizeExplicitSandboxEnvVars | `sandbox/sanitize-env-vars.ts` |

## Epic 7：多 Agent 协作（EP-MULTI）

**用户故事**：作为用户，助理能派生子代理并行处理长任务并汇总，能与对等 agent 通信。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-MULTI-01 | M | 系统必须支持父 agent 派生子代理（sessions_spawn，runtime=subagent/acp） | 子会话独立运行，登记 SQLite | `tools/sessions-spawn-tool.ts:253` |
| FR-MULTI-02 | M | 子代理默认深度必须为 1（叶子），除非配置显式开启嵌套 | DEFAULT_SUBAGENT_MAX_SPAWN_DEPTH=1 | `config/agent-limits.ts:13` |
| FR-MULTI-03 | M | 子代理并发必须有界（≤8 并发，≤5 子/agent） | 超限拒绝 | `config/agent-limits.ts` |
| FR-MULTI-04 | S | 子代理完成必须唤醒父会话并投递结果（直投→steer 兜底） | announce-dispatch 投递 | `subagent-announce-dispatch.ts:63` |
| FR-MULTI-05 | S | 系统应支持有界 A2A ping-pong（sessions_send，maxPingPongTurns） | 对等会话有界往返 | `tools/sessions-send-tool.a2a.ts:75` |
| FR-MULTI-06 | C | 系统应支持 ACP 驱动外部 agent 后端（Codex，stdio/ws） | AcpSessionManager 管理外部会话 | `acp/control-plane/manager.core.ts:65` |
| FR-MULTI-07 | M | 系统不提供 manager-of-managers/嵌套规划树作默认架构（定位约束） | 无中心调度器/规划树 | `VISION.md:122` |

## Epic 8：模型接入（EP-MODEL）

**用户故事**：作为用户，我能配置并使用多家 LLM provider，系统按模型路由。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-MODEL-01 | M | 系统必须按 model.api 分发到对应 adapter（8 内置 API 家族） | stream.ts 按 api 查注册表 | `llm-runtime/stream.ts` |
| FR-MODEL-02 | M | 系统必须支持 70+ provider 扩展（多复用 OpenAI/Anthropic 家族 + compat） | provider 清单声明 + auth + catalog | `extensions/*/openclaw.plugin.json` |
| FR-MODEL-03 | S | 系统必须支持 thinking/reasoning 级别（off/minimal/low/medium/high/xhigh/max） | 级别映射到 provider reasoning | `agent-core/reasoning.ts` |
| FR-MODEL-04 | S | 系统应修复文本泄漏的工具调用（bracketed/XML/Harmony 三语法） | 泄漏文本转真 toolCall | `tool-call-repair/stream-normalizer.ts:1055` |
| FR-MODEL-05 | S | 模型目录必须按权威合并（config>manifest>cache>provider-index） | 低数值权威胜 | `model-catalog/authority.ts:9` |
| FR-MODEL-06 | S | 系统应支持短寿命 OAuth token 动态解析（getApiKey） | 每请求解析 key | `agent-loop.ts:466` |

## Epic 9：插件与扩展（EP-PLUGIN）

**用户故事**：作为开发者，我能通过清单+register API 给系统加工具/provider/渠道/记忆等能力。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-PLUG-01 | M | 插件必须 manifest-first：核心仅凭 `openclaw.plugin.json` 发现/激活规划，零运行时导入 | activation-planner 纯清单规划 | `plugins/activation-planner.ts:74` |
| FR-PLUG-02 | M | 插件激活决策必须默认拒绝优先（denylist>slot>allowlist>默认） | resolvePluginActivationDecisionShared | `config-activation-shared.ts:128` |
| FR-PLUG-03 | M | 插件必须经 `register(api)` 注入能力（~60 register 方法） | api.registerTool/Provider/Channel/... | `plugins/registry.ts` |
| FR-PLUG-04 | M | 插件仅可经 `openclaw/plugin-sdk/*` 跨入核心，禁止 deep import 内部 | 边界规则强制（opengrep） | `extensions/AGENTS.md` |
| FR-PLUG-05 | S | memory/context-engine 必须为单槽 kind | slots 机制 | `plugins/slots.ts` |
| FR-PLUG-06 | S | 系统应支持 bundle 插件（codex/claude/cursor 外部生态） | bundle-manifest 读外部清单 | `plugins/bundle-manifest.ts` |
| FR-PLUG-07 | M | 插件安装必须失败关闭（策略 exec/before_install/依赖黑名单/归档限额） | 缺配置/block/超时均阻止 | `security/install-policy.ts` |

## Epic 10：安全与访问控制（EP-SEC）

**用户故事**：作为操作者，我的凭证受保护、连接受认证、权限最小化、执行受审批。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-SEC-01 | M | Gateway 必须认证连接（device-token/password/bootstrap/tailscale/trusted-proxy），常量时间比较 | safeEqualSecret；缺凭证不消耗限流 | `gateway/auth.ts:405` |
| FR-SEC-02 | M | 系统必须用 scope 最小权限门控方法（admin/read/write/approvals/pairing/talk.secrets），默认拒绝 | 未分类方法需 admin | `method-scopes.ts:177` |
| FR-SEC-03 | M | 自声明 scope 不得被信任；无设备身份必须清空 scope | clearUnboundScopes | `message-handler.ts:846` |
| FR-SEC-04 | M | 设备身份必须 Ed25519；签名覆盖 role+scope+nonce（防重放/越权） | verifyDeviceSignature | `infra/device-identity.ts:329` |
| FR-SEC-05 | M | exec 必须支持三级 security(deny/allowlist/full) + human-in-the-loop 审批 | 审批阻塞在 waitDecision | `infra/exec-approvals.ts` |
| FR-SEC-06 | M | 密钥必须支持 SecretRef(env/file/exec)，file/exec 强制 owner+权限+非符号链接 | assertSecurePath 校验 | `secrets/resolve.ts` |
| FR-SEC-07 | M | 日志必须脱敏密钥（~80 厂商 token 前缀 + 抗混淆 + 工具 UI 强制） | redactSecrets/redactToolPayloadText | `logging/redact.ts` |
| FR-SEC-08 | M | 网络请求必须 SSRF 防护（阻私有/loopback/元数据/IPv6 过渡，DNS pinning） | 两阶段检查 + pinned lookup | `infra/net/ssrf.ts` |
| FR-SEC-09 | S | 外部内容必须包边界标记 + 剥 LLM 特殊 token（wrapExternalContent） | 随机 id 标记 + 特殊 token 净化 | `security/external-content.ts` |
| FR-SEC-10 | M | 限流必须分 scope、loopback 豁免、缺凭证不计 | 每 scope 独立滑窗 | `gateway/auth-rate-limit.ts` |

## Epic 11：CLI/配置/运维（EP-OPS）

**用户故事**：作为自托管者，我能用终端完成 setup、诊断修复、常驻运行。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-OPS-01 | M | 系统必须提供 CLI（onboard/configure/gateway/doctor/mcp/agent 等），冷启动优化 | Commander + help fast-path | `cli/run-main.ts:610` |
| FR-OPS-02 | M | 系统必须提供终端优先 onboard 向导 | runSetupWizard 引导 | `wizard/setup.ts` |
| FR-OPS-03 | M | 运行时必须只读 canonical 配置；遗留形态仅 doctor --fix 迁移 | 迁移在 doctor，运行时不兼容 | `commands/doctor/shared/legacy-config-migrations.ts` |
| FR-OPS-04 | M | 状态必须存 SQLite（共享库 + 每 Agent 库）；不新增 JSON sidecar 运行时态 | Kysely 访问 | `src/infra/` |
| FR-OPS-05 | S | 系统必须支持守护进程（launchd/systemd/schtasks） | daemon 安装/状态 | `src/daemon/` |
| FR-OPS-06 | S | 配置变更必须支持热重载 | config-reload | `gateway/config-reload*.ts` |
| FR-OPS-07 | S | 系统应支持 cron 定时唤醒 agent（避开整点） | CronService 调度 | `cron/service.ts:15` |

## Epic 12：客户端与跨设备（EP-CLIENT）

**用户故事**：作为用户，我能通过 Web UI 控制台、伴侣 App、SDK 使用助理，并把多设备暴露为可调用节点。

| FR-ID | 优先级 | 描述 | 验收标准 | 佐证 |
|---|---|---|---|---|
| FR-CLIENT-01 | S | 系统必须提供 Web Control UI（会话/日志/渠道配置/Canvas 沙箱 iframe） | Lit UI 经 WS 连 Gateway | `ui/` |
| FR-CLIENT-02 | S | 系统必须提供公开 SDK（agents/sessions/runs/tasks 命名空间） | @openclaw/sdk 客户端 | `packages/sdk/` |
| FR-CLIENT-03 | S | 系统应支持伴侣 App（macOS/iOS/Android/Windows）经 Gateway Protocol 连接 | 设备配对 + 节点能力 | `apps/` |
| FR-CLIENT-04 | S | 系统应支持 Node Host 跨设备命令执行（exec 审批/allowlist/env 清洗） | evaluateSystemRunPolicy 门控 | `node-host/exec-policy.ts:55` |
| FR-CLIENT-05 | M | 网关协议必须版本化（v4），additive-first，破坏性需版本协商 | PROTOCOL_VERSION=4 | `gateway-protocol/version.ts:2` |
| FR-CLIENT-06 | S | Canvas 必须为白名单沙箱 iframe | resolveCanvasIframeUrl 校验路径 | `ui/src/ui/canvas-url.ts` |

<!-- APPEND_NFR -->

---

# 5. 非功能需求

## 5.1 安全（NFR-SEC）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-SEC-01 | M | 威胁模型明确为「受信任本地操作者」，不承诺多租户对抗隔离 | 文档化于 SECURITY.md；边界=auth/scope/approval/sandbox/tool |
| NFR-SEC-02 | M | 所有密钥/token 比较必须常量时间 | safeEqualSecret/timingSafeEqual，无长度泄漏 |
| NFR-SEC-03 | M | 凭证文件必须 owner-only（0600）+ 目录 0700 + O_NOFOLLOW | 文件权限核验 |
| NFR-SEC-04 | M | 默认拒绝：未分类方法需 admin；无设备身份清空 scope | 回归测试 silent-scope-upgrade POC |
| NFR-SEC-05 | M | SSRF 默认阻止全私有段+元数据+IPv6 过渡，失败关闭 | net-policy 单测覆盖 |
| NFR-SEC-06 | M | 静态分析必须作 CI 回归火墙（147 opengrep 规则映射已修漏洞） | PR diff 扫描 --error |
| NFR-SEC-07 | S | 沙箱应对不可信执行提供隔离（当前：Docker 硬化，默认防误伤级） | 见 sandbox-assessment（多租户级需补强） |

## 5.2 性能（NFR-PERF）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-PERF-01 | M | CLI 冷启动必须优化（help/version fast-path 不加载完整程序） | 无 fast-path 才动态 import |
| NFR-PERF-02 | M | 插件激活必须惰性（零运行时导入规划），热路径不加载重运行时 | activation-planner 纯清单 |
| NFR-PERF-03 | M | 网关协议校验必须惰性编译（首次调用才编译 TypeBox） | lazyCompile |
| NFR-PERF-04 | S | 系统提示必须分缓存稳定前缀 + 动态后缀（命中 prompt cache） | 前缀逐 turn 字节稳定 |
| NFR-PERF-05 | S | 热路径必须携带已备妥事实（provider/model/channel id）前向传递，不重复发现 | 无热路径广查 |
| NFR-PERF-06 | S | 无静态+动态同模块 import；用 *.runtime.ts 懒边界 | pnpm build 无 INEFFECTIVE_DYNAMIC_IMPORT |
| NFR-PERF-07 | S | CPU 密集（embedding/向量）应 worker 线程隔离 | embeddings-worker |

## 5.3 可靠性（NFR-REL）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-REL-01 | M | Agent 运行必须有跨 attempt 重试/失败转移状态机（overflow/限流/auth/空响应/超时） | run.ts 计数器 + failover-policy |
| NFR-REL-02 | M | 上下文溢出必须自动压缩并续写（非重启） | 注入 continue 提示 |
| NFR-REL-03 | M | 必须有 idle 断路器防成本失控 + post-compaction 死循环守卫 | issue #76293/#77474 守卫 |
| NFR-REL-04 | M | 运行必须可取消（AbortController 逐检查点） | 中止持久化 aborted 消息 |
| NFR-REL-05 | S | 终态优先级必须确定（hard_timeout>cancelled>completed，黏性） | mergeAgentRunTerminalOutcome |
| NFR-REL-06 | S | 记忆/向量不可用必须优雅降级（纯 FTS） | probeVectorAvailability |
| NFR-REL-07 | S | 每会话必须串行化（避免并发写会话树） | SessionActorQueue / phase 守卫 |

## 5.4 可扩展性（NFR-EXT）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-EXT-01 | M | 核心必须插件无关；新能力经 SDK 子路径而非改核心 | 扩展点矩阵覆盖 tool/provider/channel/memory/context-engine/harness |
| NFR-EXT-02 | M | 后端（沙箱/记忆/上下文引擎/harness）必须可插拔 | 注册表 + 单槽机制 |
| NFR-EXT-03 | S | provider 扩展应复用 API 家族 adapter + compat，降新增成本 | 70 provider 多复用 ~8 adapter |

## 5.5 可维护性（NFR-MAINT）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-MAINT-01 | M | 代码必须 TS ESM strict，避免 any | tsgo 类型检查 |
| NFR-MAINT-02 | M | 必须保持无循环依赖 | pnpm check:import-cycles 绿 |
| NFR-MAINT-03 | S | 重构应减少非测试 LOC 或以更大架构收益抵偿 | git diff --numstat 审查 |
| NFR-MAINT-04 | S | 文件宜在 ~700 LOC 拆分（清晰/可测时） | （核心运行时有超标，见技术债） |
| NFR-MAINT-05 | M | 行为/回归必须有测试覆盖 | Vitest колocated 测试 |

## 5.6 可移植性（NFR-PORT）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-PORT-01 | M | 必须支持 Node ≥22.19 且保留 Bun 路径 | 双运行时 CI |
| NFR-PORT-02 | M | 必须支持 macOS/Linux/Windows（onboard 三平台） | 守护进程三后端 |
| NFR-PORT-03 | S | 必须支持 Docker/Nix/Fly/Render 部署 | Dockerfile/fly.toml/render.yaml |
| NFR-PORT-04 | C | agent-core 应可独立复用（仅依赖 llm-core+typebox） | 零 OpenClaw src 依赖 |

## 5.7 可用性（NFR-USE）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-USE-01 | M | setup 必须终端优先、显式暴露 auth/权限/安全决策 | onboard 向导 |
| NFR-USE-02 | S | 错误/诊断必须可操作（doctor 给修复建议） | doctor --fix |
| NFR-USE-03 | S | 回复必须支持流式投递 | onBlockReply |
| NFR-USE-04 | C | 非交互 setup 必须显式 --accept-risk | onboard-non-interactive |

## 5.8 兼容性（NFR-COMPAT）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-COMPAT-01 | M | 配置破坏性变更必须配 doctor 迁移 | legacyConfigRules |
| NFR-COMPAT-02 | M | 网关协议必须版本协商（minProtocol/maxProtocol） | HelloOk 协商 |
| NFR-COMPAT-03 | M | 公开 CLI setup 流（onboard/configure）变更须 additive/弃用窗口 | 向后保留迁移 |
| NFR-COMPAT-04 | S | 插件 SDK 公开面破坏须新 API + 弃用 + 迁移内部调用 | 同变更迁移 bundled |

## 5.9 合规/隐私（NFR-PRIV）
| ID | 优先级 | 需求 | 度量/验收 |
|---|---|---|---|
| NFR-PRIV-01 | M | 凭证必须本地存储（~/.openclaw/credentials，不上云） | 本地文件 |
| NFR-PRIV-02 | M | 代码/密钥/用户数据不得未经允许外传第三方 | 仅用户显式请求（部署/推送）时 |
| NFR-PRIV-03 | M | 绝不提交真实电话/凭证/live 配置 | 提交前审查 |
| NFR-PRIV-04 | S | PII 在示例/日志用占位符 | 日志脱敏 |

---

# 6. 数据需求

## 6.1 持久化数据实体
| 实体 | 存储 | 说明 |
|---|---|---|
| 会话树条目 | SQLite/JSONL | SessionTreeEntry（message/compaction/branch_summary/...），append-only |
| 全局运行时状态 | `state/openclaw.sqlite` | 插件 KV、cron、节点注册 |
| 每 Agent 状态 | `agents/<id>/agent/openclaw-agent.sqlite` | agent 态、记忆索引 |
| 记忆索引 | 每 Agent SQLite | 7 表（sources/chunks+embedding/fts/vec/cache/meta/state） |
| 长期记忆 | `MEMORY.md`+`memory/*.md` | 用户可见命名产物 |
| 渠道/provider 凭证 | `~/.openclaw/credentials/` | owner-only |
| 模型 auth profile | `~/.openclaw/agents/<id>/agent/auth-profiles.json` | SQLite-backed |
| 设备身份 | `<stateDir>/identity/device.json` | Ed25519 私钥，owner-only |
| 配对/bootstrap 状态 | `<pairingDir>/devices/*.json` | 设备信任 |
| 子代理运行登记 | SQLite | subagent-registry |
| 沙箱注册 | SQLite | sandbox_registry_entries |
| 配置 | `openclaw.json` | canonical only |

## 6.2 数据约束
- **DR-01**：运行时只读 canonical 形态；遗留仅 doctor 迁移（C2）。
- **DR-02**：OpenClaw 拥有的运行时状态默认 SQLite；文件存储须为命名产物（导入导出/附件/日志/备份/外部工具契约）。
- **DR-03**：SQLite 访问用 Kysely（DDL/迁移/低层原语除外）。
- **DR-04**：prompt cache 相关序列化须确定性排序。

---

# 7. 外部接口需求

## 7.1 用户接口
- **IF-U1 CLI**：`openclaw <command>`（onboard/configure/gateway/doctor/mcp/agent/chat/...）。
- **IF-U2 Web UI**：Lit 控制台，WS 连 Gateway，含 Canvas。
- **IF-U3 消息渠道**：23+ IM 作为对话界面。
- **IF-U4 伴侣 App**：macOS/iOS/Android/Windows。

## 7.2 软件接口
- **IF-S1 LLM Provider**：8 API 家族（anthropic-messages/openai-*/google-*/mistral），HTTP/SSE。
- **IF-S2 MCP**：server + client（stdio/sse/streamable-http）。
- **IF-S3 Gateway Protocol v4**：WebSocket + TypeBox 帧（req/res/event），版本协商。
- **IF-S4 Plugin SDK**：`openclaw/plugin-sdk/*` 子路径 + `register(api)`。
- **IF-S5 SDK**：`@openclaw/sdk` 编程式客户端。
- **IF-S6 Node Host**：设备作 Gateway client 执行 system.run。

## 7.3 通信接口
- **IF-C1**：WebSocket（Gateway 控制平面，主）。
- **IF-C2**：HTTP(S)（插件路由、健康检查）。
- **IF-C3**：stdio（MCP、Codex app-server、SSH）。

---

# 8. 需求可追溯矩阵

## 8.1 Epic → 模块 → 分析文档
| Epic | 主模块 | 分析文档 |
|---|---|---|
| EP-LOOP | agent-core | `agent-core/06-loop-algorithm.md` |
| EP-SESSION | agent-core session/compaction | `agent-core/08,09` |
| EP-CHANNEL | channels/auto-reply/routing | `part-02,part-12` |
| EP-TOOL | agents/tools + mcp + skills | `part-10` |
| EP-MEMORY | memory-* | `memory-assessment.md`,`part-09` |
| EP-SANDBOX | agents/sandbox | `sandbox-assessment.md` |
| EP-MULTI | subagent/acp | `part-11` |
| EP-MODEL | llm-* | `part-02` |
| EP-PLUGIN | plugins | `part-13` |
| EP-SEC | gateway/secrets/net-policy | `security-analysis.md` |
| EP-OPS | cli/config/daemon/cron | `part-15,part-03` |
| EP-CLIENT | ui/sdk/apps/node-host | `part-02` |

## 8.2 需求 → 验证方法
| 需求类 | 验证方法 |
|---|---|
| FR-LOOP/SESSION | Vitest 单测（agent-loop/compaction 测试）+ 事件序断言 |
| FR-CHANNEL | 渠道 e2e + 入站分类/门控测试 |
| FR-MEMORY | 混合检索测试 + promotion 门槛测试（63 测试） |
| FR-SANDBOX | validate-security/path-safety/docker 测试（59 测试） |
| FR-SEC/NFR-SEC | opengrep 147 规则 + auth/scope POC 回归 + CodeQL |
| NFR-PERF | 冷启动 profile + import-cycle 检查 |
| NFR-REL | 重试/压缩/断路器测试 |

## 8.3 已知差距（需求 vs 实现完成度）
| 领域 | 完成度 | 差距（来自评估文档） |
|---|---|---|
| 沙箱（受信任定位） | 高 85/100 | 达标可发布 |
| 沙箱（多租户） | 中 60/100 | 无默认非 root/资源限制、运维自动化缺失 |
| 记忆 | 高 87/100 | 单槽双实现未收敛、dreaming 复杂、SQL/JSON 与策略张力 |
| 安全 | 高 | MCP 输出未走最强净化、限流仅进程内 |
| 多 Agent | 中（刻意） | 无嵌套编排（定位约束，非缺陷） |
| 可维护性 | 中 | 巨型文件（run/attempt.ts 5804 行等）超 700 LOC 指引 |

---

# 9. 附录

## 9.1 优先级统计
- 功能需求 FR：约 70 条（12 Epic）。
- 非功能需求 NFR：约 40 条（9 类）。
- Must(M) 占比高（骨架+核心能力），Should(S)/Could(C) 为增强/可选。

## 9.2 与配套文档关系
本 SRS 是需求层；实现层规格见 `ccdocs/agent-core/`（重写级）、评估层见 `*-assessment.md`、架构层见 `architecture.svg`+`part-02`、模块层见 `modules-and-architecture.md`。

## 9.3 变更管理
- 需求变更须更新对应 FR/NFR + 追溯矩阵。
- 破坏性需求变更须评估兼容性（NFR-COMPAT）+ doctor 迁移。

## 9.4 局限声明
本 SRS 为**逆向工程需求**，从 v2026.6.9 实现提炼。需求措辞（必须/应/可）反映**当前实现的事实优先级**，非产品路线承诺。行号为分析时点快照，随代码演进可能漂移，以实际源码为准。host-sdk embedding 内部细节因子代理调查未竟，相关 FR（FR-MEM-02/04）以 schema/接口为据。

---

*文档结束。共 12 Epic / ~70 FR / ~40 NFR / 完整数据与接口需求 + 可追溯矩阵。*



