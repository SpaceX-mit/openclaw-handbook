# 第十五部分：源码关键路径分析

## 15.1 程序入口

**文件**：`src/index.ts:1`（bin，shebang）→ `src/entry.ts:1`（Node CLI 进程入口）。
**核心函数**：
- `src/index.ts`：作为 main 时装异常处理器（`:93-118`），调 `runLegacyCliEntry` → `src/cli/run-main.js`；作为库导入时懒绑 `library.ts` re-export。
- `src/entry.ts:56-132`：compile-cache 重生检查（`:66`）→ `process.title` → env 归一化 → 版本 fast-path / `runMainOrRootHelp`。help/version fast-path 避免加载完整 CLI（`:137`/`:208`），无 fast-path 才动态 import `./cli/run-main.js`（`:279`）。

**调用链**：
```
openclaw.mjs → src/index.ts → src/entry.ts
  → (fast-path: root-help / version) 直接返回
  → src/cli/run-main.ts:610 runCli
      → command-catalog 决定启动策略(是否加载插件)
      → buildProgram (Commander) → program.parseAsync
      → 或 gateway-run fast-path → 启动 WS Gateway
```

## 15.2 Gateway 入口

**文件**：`src/daemon/gateway-entrypoint.ts`（服务管理器启动点）→ `src/gateway/server.impl.ts`（startup）。
**启动序列**（`server.impl.ts`，行号近似）：
```
bootstrap 插件安装记录 + 网络运行时 (~551)
→ 加载 config + secrets + auth (~607)
→ 加载插件注册表 + 解析方法描述符 + 建方法索引 (~704)
→ 解析 bind host/port/TLS/auth-mode + 限流 (~770)
→ 创建 channel manager + live state(cron/hooks) + node session runtime (~876)
→ 创建 HTTP(S) server + WebSocketServer (~927)
→ attachGatewayWsHandlers (~1540)
→ listen + 激活 scheduled services(heartbeat/pricing/cron/hooks) + 启动 channels (~1577)
```
（注：`server.impl.ts` 行号来自子代理调查，应视为近似锚点。）

## 15.3 Agent 入口

**文件**：`src/agents/embedded-agent-runner/run.ts:597`（`runEmbeddedAgent`）。
**核心函数链**：
```
runEmbeddedAgent (run.ts:597)
  → runEmbeddedAgentInternal (run.ts:617)
      → ensureContextEnginesInitialized + resolveContextEngine
      → resolveModelAsync (model.ts)
      → buildAgentRuntimeAuthPlan
      → selectAgentHarness (harness/selection.ts)
      → runEmbeddedAttemptWithBackend (run.ts:2031)
          → [openclaw harness] runEmbeddedAttempt (run/attempt.ts:837)
```

## 15.4 Loop 入口

**文件**：`packages/agent-core/src/agent-loop.ts:258`（`runLoop`）。
**进入路径**：
```
runEmbeddedAttempt (attempt.ts:837)
  → createOpenClawCodingTools (agent-tools.ts:394)  # 工具
  → buildSystemPromptParams (attempt.ts:1919)        # 系统提示
  → createAgentSession (attempt.ts:2471)             # 会话
  → assembleHarnessContextEngine (context-engine-lifecycle.ts:133)  # 上下文
  → activeSession.prompt() (attempt.ts:3426)
      → AgentSession.runAgentPrompt (agent-session.ts:1089)
          → this.agent.prompt() → runAgentLoop (agent.ts:443)
              → runLoop (agent-loop.ts:258)   ← LOOP 入口
```

## 15.5 Tool 入口

**文件**：`packages/agent-core/src/agent-loop.ts:540`（`executeToolCalls`）。
**调用链**：
```
runLoop → 检测 toolCall → executeToolCalls (agent-loop.ts:540)
  → 并行/串行分派 (executeToolCallsParallel:665 / Sequential:600)
  → prepareToolCall (:833): prepareArguments → validateToolArguments → beforeToolCall
  → executePreparedToolCall (:921): tool.execute(id,args,signal,onUpdate)
  → finalizeExecutedToolCall (:958): afterToolCall
  → createToolResultMessage (:1025)
```
OpenClaw 工具策略挂在 `beforeToolCall`/`afterToolCall`（`agent-session.ts:498/521`）。

## 15.6 Memory 入口

**文件**：`extensions/memory-core/src/memory/manager.ts:599`（`MemoryIndexManager.search`）。
**两条路径**：
- **检索**：模型调 `memory_search` 工具 → `MemoryIndexManager.search` → `searchVector`（`:849`）+ `searchKeyword`（`:879`）→ `mergeHybridResults`（`:960`）→ MMR + 时间衰减。
- **注入**：系统提示 `buildMemorySection`（`system-prompt.ts:286`）+ `active-memory` 回复前阻塞召回注入。
- **索引**：会话转写 / `MEMORY.md` → SQLite（FTS5 + sqlite-vec），DDL 在 `packages/memory-host-sdk/src/host/memory-schema.ts`。

## 15.7 UI 入口

**文件**：`ui/src/main.ts`（bootstrap）→ `<openclaw-app>` 根组件。
**连接链**：
```
ui/src/main.ts → 注册 SW(prod) → 挂载 <openclaw-app>
  → ui/src/ui/app-gateway.ts connectGateway
      → GatewayBrowserClient (ui/src/ui/gateway.ts)
      → device-auth 握手(ed25519) → 订阅事件(seq/gap检测) → 指数退避重连
  → Canvas: resolveCanvasIframeUrl (ui/src/ui/canvas-url.ts) 白名单沙箱iframe
```

## 15.8 入站消息入口（端到端）

**文件**：`src/auto-reply/dispatch.ts:501`（`dispatchInboundMessage`）。
**调用链**：
```
渠道 → buildChannelInboundEventContext (inbound-event/context.ts:459)
  → classifyChannelInboundEvent (classification.ts:26)
  → 门控: resolveInboundMentionDecision / resolveControlCommandGate / allowlist
  → createInboundDebouncer (inbound-debounce.ts:63)
  → resolveAgentRoute (resolve-route.ts:611) → {agentId, sessionKey}
  → dispatchInboundMessage (dispatch.ts:501)
      → dispatchReplyFromConfig → runEmbeddedAgent
  → 回复: turn/kernel.ts:353 dispatchAssembledChannelTurn → dispatcher.deliver
```

## 15.9 关键路径速查表

| 入口 | 文件:行 | 核心函数 |
|---|---|---|
| 程序 | `src/entry.ts:56` / `src/cli/run-main.ts:610` | `runCli` |
| Gateway | `src/gateway/server.impl.ts` | startup 序列 |
| Agent | `src/agents/embedded-agent-runner/run.ts:597` | `runEmbeddedAgent` |
| Attempt | `src/agents/embedded-agent-runner/run/attempt.ts:837` | `runEmbeddedAttempt` |
| Loop | `packages/agent-core/src/agent-loop.ts:258` | `runLoop` |
| Tool | `packages/agent-core/src/agent-loop.ts:540` | `executeToolCalls` |
| LLM 分发 | `packages/llm-runtime/src/stream.ts` | `stream`（按 model.api） |
| Memory | `extensions/memory-core/src/memory/manager.ts:599` | `search` |
| 压缩 | `packages/agent-core/src/harness/compaction/compaction.ts:236` | `shouldCompact` / `findCutPoint` |
| 系统提示 | `src/agents/system-prompt.ts:682` | `buildAgentSystemPrompt` |
| 入站分发 | `src/auto-reply/dispatch.ts:501` | `dispatchInboundMessage` |
| 路由 | `src/routing/resolve-route.ts:611` | `resolveAgentRoute` |
| UI | `ui/src/main.ts` | `connectGateway` |
| 插件加载 | `src/plugins/loader.ts:1821` | `loadOpenClawPlugins` |
| 子代理派生 | `src/agents/tools/sessions-spawn-tool.ts:253` | `createSessionsSpawnTool` |
