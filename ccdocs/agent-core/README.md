# `@openclaw/agent-core` 模块完整规格

> 目标：一份**可据以从零重写整个 `agent-core` 模块**的规格文档。涵盖：模块定位、对上提供的接口、调用方（谁来调用它）、完整功能与架构、业务逻辑、核心算法、数据结构、流程时序、以及验收标准。
> 分析对象：`packages/agent-core/`（25 个非测试 `.ts` 文件，约 8129 行源码 + 测试），version `0.0.0-private`（workspace 内部包）。
> 证据全部以仓库根相对路径 `file:line` 标注，均经直接阅读核对。

---

## 文档导航

| # | 文件 | 内容 |
|---|------|------|
| 00 | [README.md](README.md) | 本文：总览 + 导航 + 速查 |
| 01 | [01-overview-positioning.md](01-overview-positioning.md) | 模块定位、职责边界、在系统中的层级、依赖 |
| 02 | [02-public-api.md](02-public-api.md) | 完整公开 API 面（导出清单 + 每个导出的契约） |
| 03 | [03-consumers-callgraph.md](03-consumers-callgraph.md) | **谁调用它**：调用方图谱、SDK facade、注入运行时 |
| 04 | [04-architecture.md](04-architecture.md) | 内部架构：三层结构、文件职责、依赖图 |
| 05 | [05-data-structures.md](05-data-structures.md) | 全部核心数据结构与类型契约 |
| 06 | [06-loop-algorithm.md](06-loop-algorithm.md) | ReAct 循环引擎算法（runLoop 全流程） |
| 07 | [07-harness.md](07-harness.md) | CoreAgentHarness：会话/钩子/队列/phase 状态机 |
| 08 | [08-session-tree.md](08-session-tree.md) | 会话树存储、上下文构建、分支导航 |
| 09 | [09-compaction-algorithm.md](09-compaction-algorithm.md) | 压缩与分支摘要算法（切点/token/摘要） |
| 10 | [10-flows-sequences.md](10-flows-sequences.md) | 端到端流程与时序图 |
| 11 | [11-acceptance-criteria.md](11-acceptance-criteria.md) | **验收标准**：可据以验证重写正确性 |

---

## 一句话定位

`@openclaw/agent-core` 是 **provider-agnostic（与模型供应商无关）的 Agent 循环引擎 + 会话外壳库**。它把「思考→行动→观察」的 ReAct 循环、工具执行、会话树持久化、上下文压缩、分支摘要做成**可复用、可注入、不依赖 OpenClaw 主体**的纯库。它**不知道**任何具体的 LLM provider、渠道、插件 —— 这些通过 `AgentCoreRuntimeDeps`（注入的 `streamSimple`/`completeSimple`）和各种回调钩子从外部传入。

它对上提供三个层次的 API：
1. **底层循环**：`runAgentLoop` / `agentLoop`（纯函数 ReAct 循环）。
2. **有状态 Agent**：`Agent` 类（持有转写 + 队列 + 生命周期事件）。
3. **会话化 Harness**：`CoreAgentHarness` 类（会话树 + 压缩 + 分支 + 17 类钩子）。
外加：会话存储（JSONL/内存）、压缩算法、分支摘要、消息转换、执行环境（Node FS/Shell）、技能/模板格式化、工具校验等工具函数。

---

## 调用方速览（详见 03）

```
                    ┌─────────────────────────────────────────┐
                    │  @openclaw/agent-core (本模块)           │
                    │  runAgentLoop / Agent / CoreAgentHarness │
                    └───────────────▲─────────────────────────┘
                                    │ 注入 streamSimple/completeSimple
          ┌─────────────────────────┴──────────────────────────┐
          │ src/plugin-sdk/agent-core.ts  (SDK facade)          │
          │ src/agents/runtime/index.ts   (运行时 facade)        │
          │  → class Agent extends CoreAgent (预绑 OpenClaw 运行时)│
          └─────────────────────────▲──────────────────────────┘
                                    │ openclaw/plugin-sdk/agent-core (78 处导入)
       ┌────────────────────────────┼────────────────────────────┐
       │ src/agents/sessions/*       │ src/agents/embedded-agent-runner/* │
       │ (AgentSession 包装 Agent)    │ (运行编排，调 AgentSession)         │
       │ src/sessions/*              │ extensions/* (codex-supervisor 等) │
       │ src/process/kill-tree.ts    │                                    │
       └─────────────────────────────────────────────────────────┘
```

**关键事实**：
- 外部**从不直接** `import "@openclaw/agent-core"`（0 处），而是经两条 facade：
  - `openclaw/plugin-sdk/agent-core`（78 处，SDK 子路径）
  - 相对路径 `../../packages/agent-core/src/*.js`（14 处，核心内部 + facade 自身）
- 两条 facade（`src/plugin-sdk/agent-core.ts` 与 `src/agents/runtime/index.ts`）做同一件事：**把 OpenClaw 的 `streamSimple`/`completeSimple` 注入成 `AgentCoreRuntimeDeps`，导出预配置的 `Agent` 子类，并 re-export 整个 agent-core**。
- 真正驱动循环的是 `src/agents/sessions/agent-session.ts` 的 `AgentSession`（持有 `CoreAgent` 实例），再上层是 `src/agents/embedded-agent-runner/`。

---

## 核心证据锚点

| 主题 | 位置 |
|---|---|
| 循环引擎 | `packages/agent-core/src/agent-loop.ts:258`（`runLoop`） |
| 公开循环入口 | `agent-loop.ts:87/126/164/191`（`agentLoop`/`agentLoopContinue`/`runAgentLoop`/`runAgentLoopContinue`） |
| Agent 状态机 | `packages/agent-core/src/agent.ts:204`（`Agent`） |
| Harness | `packages/agent-core/src/harness/agent-harness.ts:217`（`CoreAgentHarness`） |
| 注入运行时契约 | `packages/agent-core/src/runtime-deps.ts:5`（`AgentCoreRuntimeDeps`） |
| 类型契约 | `packages/agent-core/src/types.ts`（`AgentLoopConfig`/`AgentTool`/`AgentEvent`/`AgentMessage`） |
| Harness 类型 | `packages/agent-core/src/harness/types.ts`（`Session`/`SessionTreeEntry`/钩子事件） |
| 会话树 | `packages/agent-core/src/harness/session/session.ts:106`（`Session`） |
| 压缩 | `packages/agent-core/src/harness/compaction/compaction.ts`（`shouldCompact`/`findCutPoint`/`compact`） |
| 公开导出清单 | `packages/agent-core/src/index.ts` |
| OpenClaw facade | `src/agents/runtime/index.ts:19` / `src/plugin-sdk/agent-core.ts:17` |

---

## 依赖（极小）

```json
"dependencies": {
  "@openclaw/llm-core": "workspace:*",   // 消息/模型/事件流契约
  "typebox": "1.1.39"                     // 工具参数 schema
}
```
零第三方运行时依赖（除 Node 内置 + llm-core + typebox）。这是其「可独立复用」的基础。
