# 02 · 完整公开 API 面

本节列出 agent-core 通过 `src/index.ts`（及 `node.ts`）导出的**全部公开 API**，每项给出签名要点与契约。重写时这是必须 1:1 复刻的对外契约。

## 2.1 导出总览（来自 `index.ts` + `node.ts`）

```
agent.js            → Agent 类, AgentOptions, QueueMode
agent-loop.js       → agentLoop, agentLoopContinue, runAgentLoop,
                       runAgentLoopContinue, AgentEventSink
node.js             → NodeExecutionEnv (Node 专用入口)
runtime-deps.js     → AgentCoreRuntimeDeps, AgentCoreStreamRuntimeDeps,
                       AgentCoreCompletionRuntimeDeps,
                       resolveAgentCoreStreamFn, resolveAgentCoreCompleteFn
types.js            → 全部循环类型（见 05）
validation.js       → validateToolArguments, validateToolCall (re-export llm-core)
harness/agent-harness.js → CoreAgentHarness (别名 AgentHarness)
harness/env/kill-tree.js → killProcessTree 等
harness/messages.js → convertToLlm, asAgentMessage, bashExecutionToText,
                       createBranchSummaryMessage, createCompactionSummaryMessage,
                       createCustomMessage, COMPACTION_SUMMARY_PREFIX/SUFFIX,
                       BRANCH_SUMMARY_PREFIX/SUFFIX, HarnessMessage
harness/prompt-template-arguments.js → formatPromptTemplateInvocation
harness/skills.js   → formatSkillInvocation
harness/types.js    → 全部 harness 类型（见 05）
harness/session/jsonl-storage.js  → JSONL 会话存储
harness/session/memory-storage.js → 内存会话存储
harness/session/session.js → Session 类, buildSessionContext
harness/session/uuid.js → uuidv7
harness/compaction/branch-summarization.js → generateBranchSummary,
   collectEntriesForBranchSummary, prepareBranchEntries, + 类型
harness/compaction/compaction.js → compact, prepareCompaction, shouldCompact,
   findCutPoint, findTurnStartIndex, generateSummary, estimateTokens,
   estimateContextTokens, calculateContextTokens, getLastAssistantUsage,
   serializeConversation, DEFAULT_COMPACTION_SETTINGS, + 类型
harness/utils/truncate.js → 文本截断工具
```

## 2.2 循环 API（最重要）

### `runAgentLoop(prompts, context, config, emit, signal?, streamFn?, runtime?): Promise<AgentMessage[]>`
（`agent-loop.ts:164`）以新 prompt 启动一次循环，事件经 `emit: AgentEventSink` 回调推送，返回本次产生的新消息。
- **契约**：先 emit `agent_start`/`turn_start`，再为每条 prompt emit `message_start`/`message_end`，然后进入 `runLoop`。
- `streamFn`/`runtime` 二选一提供流式实现（`resolveAgentCoreStreamFn` 优先用 `streamFn`，否则 `runtime.streamSimple`，都没有则抛错）。

### `runAgentLoopContinue(context, config, emit, signal?, streamFn?, runtime?): Promise<AgentMessage[]>`
（`agent-loop.ts:191`）从现有 context 继续（不加新消息），**最后一条消息必须能转成 user/toolResult**（不能是 assistant，否则抛错）。

### `agentLoop(...) / agentLoopContinue(...)`
（`agent-loop.ts:87/126`）上述两者的 **EventStream 版本**：返回 `EventStream<AgentEvent, AgentMessage[]>`（异步可迭代），内部把失败编码为 `pushLoopFailure`（emit message_start/end + turn_end + agent_end 的 error 消息），不抛出。

### `type AgentEventSink = (event: AgentEvent) => Promise<void> | void`
（`agent-loop.ts:27`）事件回调类型。

## 2.3 `Agent` 类（`agent.ts:204`）

有状态封装。**公开成员**：
- 构造 `new Agent(options: AgentOptions)`。
- `subscribe(listener): () => void` — 订阅事件，listener 收 `(event, signal)`，按订阅序 await，计入运行结算。
- `get state(): AgentState` — 当前状态（systemPrompt/model/thinkingLevel/tools/messages/isStreaming/streamingMessage/pendingToolCalls/errorMessage）。
- `prompt(message | message[])` / `prompt(text, images?)` — 启动新 prompt（已有 activeRun 则抛错）。
- `continue()` — 从当前转写继续（末条须为 user/toolResult；若末条为 assistant 则尝试 drain steering/follow-up）。
- `steer(msg)` / `followUp(msg)` — 入队（中途纠偏 / 停下后追加）。
- `clearSteeringQueue()` / `clearFollowUpQueue()` / `clearAllQueues()` / `hasQueuedMessages()`。
- `get/set steeringMode` / `get/set followUpMode`（`QueueMode = "all" | "one-at-a-time"`）。
- `get signal` — 当前 run 的 AbortSignal。`abort()` — 中止当前 run。
- `waitForIdle(): Promise<void>` — 等到 run + 所有 await 监听器结算。
- `reset()` — 清转写/状态/队列。
- 公开可写字段：`convertToLlm`/`transformContext`/`runtime`/`streamFn`/`getApiKey`/`onPayload`/`onResponse`/`beforeToolCall`/`resolveDeferredTool`/`afterToolCall`/`prepareNextTurn`/`sessionId`/`thinkingBudgets`/`transport`/`maxRetryDelayMs`/`toolExecution`。

### `AgentOptions`（`agent.ts:104`）
构造选项：`initialState`（transcript/tools/model/thinkingLevel/systemPrompt）、`convertToLlm`、`transformContext`、`runtime`、`streamFn`、`getApiKey`、`onPayload`、`onResponse`、`beforeToolCall`、`resolveDeferredTool`、`afterToolCall`、`prepareNextTurn`、`steeringMode`、`followUpMode`、`sessionId`、`thinkingBudgets`、`transport`、`maxRetryDelayMs`、`toolExecution`。

## 2.4 `CoreAgentHarness` 类（`harness/agent-harness.ts:217`，别名 `AgentHarness`）

泛型 `<TSkill, TPromptTemplate, TTool>`。**公开方法**：
- `prompt(text, {images?}): Promise<AssistantMessage>` — 跑一个 prompt turn（须 idle 否则抛 `busy`）。
- `skill(name, additionalInstructions?)` — 按技能名跑一次（用 `formatSkillInvocation`）。
- `promptFromTemplate(name, args[])` — 按模板名跑一次（用 `formatPromptTemplateInvocation`）。
- `steer(text,{images?})` / `followUp(text,{images?})` / `nextTurn(text,{images?})` — 三类队列入队。
- `appendMessage(message)` — idle 时直接落盘，运行中入 `pendingSessionWrites`。
- `compact(customInstructions?)` — 压缩会话（须 idle）。
- `navigateTree(targetId, {summarize?, customInstructions?, replaceInstructions?, label?})` — 分支导航。
- `getModel()`/`setModel(model)`、`getThinkingLevel()`/`setThinkingLevel(level)`、`setActiveTools(names)`、`getResources()`/`setResources(res)`、`getStreamOptions()`/`setStreamOptions(opts)`、`setTools(tools, activeNames?)`、`get/set SteeringMode`/`FollowUpMode`。
- `abort(): Promise<AbortResult>` — 中止 + 清队列 + 等待 idle。
- `waitForIdle()`、`subscribe(listener)`、`on(type, handler)`（注册带类型返回值的钩子）。

### `AgentHarnessOptions`（`harness/types.ts:806`）
`env: ExecutionEnv`、`session: Session`、`tools?`、`resources?`、`systemPrompt?`（字符串或回调）、`getApiKeyAndHeaders?`、`runtime?`、`streamOptions?`、`model`、`thinkingLevel?`、`activeToolNames?`、`steeringMode?`、`followUpMode?`。

## 2.5 注入运行时 API（`runtime-deps.ts`）

```ts
interface AgentCoreRuntimeDeps {
  streamSimple: StreamFn;        // 流式（普通 turn）
  completeSimple: CompleteSimpleFn; // 非流式（摘要用）
}
resolveAgentCoreStreamFn(runtime?, streamFn?): StreamFn   // streamFn 优先
resolveAgentCoreCompleteFn(runtime?): CompleteSimpleFn
```
**契约**：未配置时抛明确错误「runtime dependency "X" is not configured」。

## 2.6 会话 API（`harness/session/session.js`）

- `class Session<TMetadata>`：`getMetadata`/`getLeafId`/`getEntry`/`getEntries`/`getBranch(fromId?)`/`buildContext()`/`getLabel`/`getSessionName`/`appendMessage`/`appendThinkingLevelChange`/`appendModelChange`/`appendCompaction`/`appendCustomEntry`/`appendCustomMessageEntry`/`appendLabel`/`appendSessionName`/`moveTo(entryId, summary?)`/`getStorage`。
- `buildSessionContext(pathEntries): SessionContext` — 从分支条目构建 `{messages, thinkingLevel, model}`，处理压缩重放。
- 存储实现：`JsonlSessionStorage`（文件）、`MemorySessionStorage`（内存），均实现 `SessionStorage` 接口。

## 2.7 压缩 API（`harness/compaction/compaction.js`）

- `shouldCompact(contextTokens, contextWindow, settings): boolean` — `contextTokens > contextWindow - reserveTokens`。
- `prepareCompaction(pathEntries, settings): Result<CompactionPreparation | undefined, CompactionError>` — 计算切点、待摘要消息、文件操作。
- `compact(preparation, model, apiKey, headers?, customInstructions?, signal?, thinkingLevel?, streamFn?, runtime?): Promise<Result<CompactionResult, CompactionError>>` — 生成摘要。
- `generateSummary(...)` / `findCutPoint(...)` / `findTurnStartIndex(...)` / `estimateTokens(msg)` / `estimateContextTokens(msgs)` / `calculateContextTokens(usage)` / `getLastAssistantUsage(entries)` / `serializeConversation(...)`。
- `DEFAULT_COMPACTION_SETTINGS = { enabled: true, reserveTokens: 16384, keepRecentTokens: 20000 }`。
- `SUMMARIZATION_SYSTEM_PROMPT`（导出于 compaction.ts:441，但未在 index 重导出 —— 内部常量）。

## 2.8 分支摘要 API（`harness/compaction/branch-summarization.js`）

- `generateBranchSummary(entries, options): Promise<Result<BranchSummaryResult, ...>>`。
- `collectEntriesForBranchSummary(session, oldLeafId, targetId)` / `collectEntriesForBranchSummaryFromBranches` / `prepareBranchEntries`。

## 2.9 消息转换 API（`harness/messages.js`）

- `convertToLlm(messages: AgentMessage[]): Message[]` — 把 harness 消息（bash/custom/branchSummary/compactionSummary）转成 LLM 可读 Message，过滤 `excludeFromContext`。
- `asAgentMessage`、`bashExecutionToText`、`createBranchSummaryMessage`/`createCompactionSummaryMessage`/`createCustomMessage`、前后缀常量、`HarnessMessage` 类型。

## 2.10 其他工具 API

- `formatSkillInvocation(skill, additional?)`（`harness/skills.ts`）→ `<skill name location>...</skill>`。
- `formatPromptTemplateInvocation(template, args)`（`harness/prompt-template-arguments.ts`）。
- `validateToolArguments`/`validateToolCall`（re-export llm-core）。
- `uuidv7()`（会话条目 id）。
- `NodeExecutionEnv`（`harness/env/nodejs.ts`）：FileSystem + Shell 的 Node 实现。
- `killProcessTree`（`harness/env/kill-tree.ts`）。
- 文本截断工具（`harness/utils/truncate.ts`）。
