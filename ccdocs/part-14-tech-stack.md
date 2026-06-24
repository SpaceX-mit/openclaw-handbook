# 第十四部分：技术选型分析

## 14.1 核心技术栈一览（来自 `package.json` v2026.6.9）

| 维度 | 选型 | 版本 |
|---|---|---|
| 语言 | **TypeScript**（ESM, strict） | 6.0.3 |
| 运行时 | **Node** ≥22.19（推荐 24），保留 Bun 路径 | — |
| 包管理 | **pnpm** workspace | 11.2.2 |
| Schema/校验 | **typebox**（主）+ **zod**（边界） | typebox 1.1.39 / zod 4.4.3 |
| 存储 | **SQLite**（`node:sqlite` + **Kysely** 0.29.2） | — |
| 向量 | **sqlite-vec** + **@lancedb/lancedb**（可选） | — |
| 本地 embedding | **node-llama-cpp** | — |
| 传输 | **ws**（WebSocket） | 8.21.0 |
| LLM SDK | **@anthropic-ai/sdk** 0.100.1、**openai** 6.39.1、google genai 等 | — |
| MCP | **@modelcontextprotocol/sdk** | 1.29.0 |
| 渠道 SDK | **grammy**（Telegram）、discord（carbon）、slack、baileys（WhatsApp）等 | grammy 1.43 |
| Web UI | **Lit 3** + **Vite 8** | — |
| 原生 App | **Swift 6.2**（iOS/macOS）、**Kotlin/Compose**（Android）、**WinUI**（Windows） | — |
| 构建 | **tsdown**（不用 tsc emit）、**tsgo**（typecheck） | — |
| 格式/lint | **oxfmt** + **oxlint**（不用 Prettier/ESLint） | — |
| 测试 | **vitest** | — |

## 14.2 为什么选 TypeScript

VISION.md「Why TypeScript?」段直接回答：**OpenClaw 主要是编排系统（prompts/tools/protocols/integrations），TypeScript 让它「默认可黑客化」** —— 广为人知、迭代快、易读易改易扩展。

**优点**：
- 全栈统一语言（核心 + UI + SDK + 插件），降低贡献门槛。
- 强类型 + 判别联合（discriminated union）对协议/事件/消息建模极合适（`AssistantMessageEvent`、`SessionTreeEntry`、`AgentEvent` 都是判别联合）。
- 生态最大的 IM SDK 都在 JS/TS（grammy/discord.js/baileys），渠道适配天然契合。
- ESM + 动态 import 支持懒加载（冷启动优化）。

**缺点**：
- CPU 密集（embedding、向量）需 worker 线程 + 原生扩展（node-llama-cpp、sqlite-vec）绕开单线程瓶颈。
- 类型系统复杂度高（看 `PendingSessionWrite` 的条件类型映射）。
- 与 Rust/Go 比，常驻服务的内存/GC 开销更大。

## 14.3 为什么不选 Python（对比）

Python 是 Agent 框架主流（LangChain/CrewAI/AutoGen 都是 Python）。OpenClaw 不选的隐含理由：
- 它是**产品/运行时**而非**研究框架**，需常驻服务 + 多渠道 SDK + 跨平台分发，JS/TS 生态在「服务 + IM 集成 + 打包分发」上更强。
- 终端优先 + npm/pnpm/bun 分发，单 `npx openclaw` 即可跑，Python 的环境/依赖地狱不利于个人自托管。
- 类型安全对协议契约（gateway-protocol v4）至关重要。

## 14.4 为什么选 SQLite（而非 Postgres/Redis）

AGENTS.md 硬策略「Storage default: SQLite only」。
- **个人单用户**：无需多租户并发，SQLite 足够且零运维。
- **本地优先/隐私**：数据落本地文件，无需外部服务。
- 双层库设计：共享态 `state/openclaw.sqlite`（全局运行时 + 插件 KV）+ 每 Agent 库 `agents/<id>/agent/openclaw-agent.sqlite`（agent 态/缓存/记忆索引）。
- 运行时访问用 **Kysely**（类型安全 query builder），但记忆索引用 `node:sqlite` 原始 SQL（DDL/低层原语豁免）+ FTS5 + sqlite-vec。
- **缺点**：不适合多写并发与水平扩展；OpenClaw 用 WAL + 写锁 + 每会话串行队列缓解，但这决定了它**不是多用户云平台**。

**为什么不用 Postgres/Redis**：会引入外部依赖、违背「个人自托管零运维」定位。向量也不用专用 VectorDB（Pinecone 等），而是 sqlite-vec/LanceDB 嵌入式，保持自包含。

## 14.5 为什么不用 LangGraph / 现成 Agent 框架

OpenClaw **自研 agent-core 循环**而非用 LangGraph：
- LangGraph 是 graph-based 编排，OpenClaw 定位是「无重编排层」（`VISION.md:124`），单 ReAct 循环 + 工程化重试已满足个人助理。
- 自研循环可深度定制 steering/follow-up 队列、会话树、压缩、provider-agnostic streamFn —— 这些是现成框架难以提供的细粒度控制。
- 自研避免重依赖与版本绑定，保持「可黑客化」。

## 14.6 其他关键选型

| 选型 | 理由 |
|---|---|
| **typebox 而非纯 zod** | typebox 直接产 JSON Schema（工具参数即 LLM tool schema），运行时校验 + 编译期类型双赢；zod 用于外部边界 |
| **oxfmt/oxlint 而非 Prettier/ESLint** | Rust 实现，速度快，适合 59 万行大仓 |
| **tsdown/tsgo 而非 tsc** | 构建/类型检查更快；动态 import 边界检测（`[INEFFECTIVE_DYNAMIC_IMPORT]`） |
| **WebSocket 而非 HTTP/gRPC** | 双向流式（事件推送 + RPC），适合常驻控制平面 + 实时 UI |
| **Gateway Protocol 自研（TypeBox schema）** | 强类型 wire 契约，惰性编译省冷启动，版本协商（v4） |
| **ACP（Agent Client Protocol）** | 标准化驱动外部 agent 后端（Codex/Gemini），复用而非重造 |
| **MCP 双向** | 拥抱生态标准工具协议 |

## 14.7 技术选型的内在一致性

所有选型服务于三个一致目标：
1. **个人自托管零运维** → SQLite、本地凭证、单二进制、Docker/Nix。
2. **可黑客化/易贡献** → TypeScript 全栈、manifest 插件、精简核心。
3. **生产级可靠 + 冷启动快** → 惰性编译、动态 import 懒加载、worker 隔离、Rust 工具链。

代价是放弃了「多用户云规模」与「CPU 密集本地推理性能」，但这与定位完全自洽。
