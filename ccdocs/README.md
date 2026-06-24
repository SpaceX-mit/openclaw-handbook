# OpenClaw 深度源码分析报告

> 目标：达到「能独立复刻 / 能重构 / 能设计同类竞品 / 能输出技术架构与 PRD / 能指导团队实现」的深度。
> 分析对象：`openclaw/openclaw`（commit on `main`，version `2026.6.9`），约 **59 万行 TypeScript**（src 36.5 万 + extensions 18 万 + packages 4.6 万）。
> 分析方法：源码级直接阅读 + 多路并行子代理调查，全部结论以仓库根相对路径 `file:line` 为证据。

---

## 文档导航（21 部分）

| # | 文件 | 主题 |
|---|------|------|
| 01 | [part-01-positioning.md](part-01-positioning.md) | 项目定位：解决什么问题、用户画像、竞争力、护城河 |
| 02 | [part-02-architecture.md](part-02-architecture.md) | 整体架构：分层架构图、职责、数据流/控制流 |
| 03 | [part-03-directory.md](part-03-directory.md) | 代码目录逆向：逐目录作用、核心类、依赖图 |
| 04 | [part-04-object-model.md](part-04-object-model.md) | 核心对象模型：ER/类图、生命周期、状态变化 |
| 05 | [part-05-agent-mechanism.md](part-05-agent-mechanism.md) | Agent 运行机制：ReAct/状态机/时序图 |
| 06 | [part-06-runtime.md](part-06-runtime.md) | Runtime：上下文/Token/并发/取消/Checkpoint |
| 07 | [part-07-harness.md](part-07-harness.md) | Harness 分析：是什么、为何需要、状态机 |
| 08 | [part-08-loop-engineering.md](part-08-loop-engineering.md) | Loop Engineering：循环结构、终止、重试、反思 |
| 09 | [part-09-memory.md](part-09-memory.md) | Memory 系统：短期/工作/长期/RAG/压缩/淘汰 |
| 10 | [part-10-tool-mcp.md](part-10-tool-mcp.md) | Tool / MCP / Skill / Plugin / 权限 / 沙箱 |
| 11 | [part-11-multi-agent.md](part-11-multi-agent.md) | 多 Agent：子代理、ACP、A2A、Node 模式 |
| 12 | [part-12-data-flow.md](part-12-data-flow.md) | 数据流：从用户输入到结果的完整时序 |
| 13 | [part-13-extension.md](part-13-extension.md) | 扩展机制：扩展点矩阵 |
| 14 | [part-14-tech-stack.md](part-14-tech-stack.md) | 技术选型分析 |
| 15 | [part-15-key-paths.md](part-15-key-paths.md) | 源码关键路径：各入口、调用链 |
| 16 | [part-16-reproduction.md](part-16-reproduction.md) | 复刻指南：六阶段 Roadmap |
| 17 | [part-17-agent-os-mapping.md](part-17-agent-os-mapping.md) | Agent OS 映射：Kernel/Process/IPC/... |
| 18 | [part-18-competitors.md](part-18-competitors.md) | 竞品对比：LangGraph/CrewAI/AutoGen/OpenHands/Claude Code |
| 19 | [part-19-pros-cons.md](part-19-pros-cons.md) | 优缺点 TOP20、技术债、架构风险 |
| 20 | [part-20-conclusion.md](part-20-conclusion.md) | 最终结论 + 五维评分 |
| 21 | [part-21-agentic-os-layer.md](part-21-agentic-os-layer.md) | Agentic OS 分层定位 |

---

## 一句话总览

**OpenClaw 是一个「个人 AI 助理」编排系统**：你在自己的设备上运行一个 **Gateway（控制平面）**，它把你已经在用的几十个消息渠道（WhatsApp/Telegram/Slack/Discord/iMessage/微信…）接入一个统一的 Agent 运行时；Agent 通过一个**可插拔、事件驱动的 ReAct 循环引擎**（`@openclaw/agent-core`）调用工具完成真实任务，并通过**会话树（Session Tree）+ 压缩（Compaction）+ 记忆插件槽（Memory Slot）**管理长期上下文。整个系统以 **manifest-first 的插件架构**为骨架，核心保持精简，能力（70+ 模型提供商、25+ 渠道、记忆/上下文引擎/技能/MCP）全部外移为插件。

**它最接近的类比是「Android Framework + System Server」**，而不是某个 Agent 框架库：它不是给你写代码用的 SDK，而是一个**常驻、多渠道、多 Agent、面向个人的 Agent 操作系统级运行时**。

---

## 核心证据锚点（贯穿全文反复引用）

- **Loop 引擎**：`packages/agent-core/src/agent-loop.ts:258`（`runLoop`）— 外层 follow-up 循环 + 内层工具调用循环。
- **Agent 状态机**：`packages/agent-core/src/agent.ts:204`（`Agent` 类）。
- **Harness（会话/压缩/分支/钩子）**：`packages/agent-core/src/harness/agent-harness.ts:217`（`CoreAgentHarness`）。
- **会话树类型**：`packages/agent-core/src/harness/types.ts:443`（`SessionTreeEntry` 联合）。
- **OpenClaw 实际运行时**：`src/agents/embedded-agent-runner/run/attempt.ts`（`runEmbeddedAttempt`，5804 行）。
- **可插拔上下文引擎**：`src/context-engine/types.ts:298`（`ContextEngine` 接口）。
- **网关协议**：`packages/gateway-protocol/src/version.ts:2`（`PROTOCOL_VERSION = 4`）。
- **插件清单**：`src/plugins/manifest.ts:297`（`PluginManifest`）。
- **记忆插件槽（单槽）**：`src/plugins/config-activation-shared.ts:342`（`resolveMemorySlotDecisionShared`）。

> 说明：本报告中标注 `file:line` 的位置绝大多数经直接阅读核对；少量来自并行子代理的行号（已在相应章节注明）应视为「近似锚点」，复刻/重构时以实际文件为准。
