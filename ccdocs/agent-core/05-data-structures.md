# 05 · 核心数据结构与类型契约

重写时这些类型须 1:1 复刻（含字段语义）。来源：`types.ts`、`harness/types.ts`、消费的 `@openclaw/llm-core`。

## 5.1 消息体系

### 来自 llm-core（agent-core 消费，不定义）
```ts
type Message = UserMessage | AssistantMessage | ToolResultMessage;

UserMessage   = { role:"user"; content: string | (TextContent|ImageContent)[]; timestamp }
AssistantMessage = {
  role:"assistant";
  content: (TextContent | ThinkingContent | ToolCall)[];
  api; provider; model;            // 身份
  usage: Usage; stopReason;        // "stop"|"length"|"toolUse"|"error"|"aborted"
  errorMessage?; errorCode?; ...;
  timestamp;
}
ToolResultMessage = { role:"toolResult"; toolCallId; toolName; content:(TextContent|ImageContent)[]; isError; details?; timestamp }
ToolCall = { type:"toolCall"; id; name; arguments: Record<string,unknown>; executionMode? }
Usage = { input; output; cacheRead; cacheWrite; totalTokens; cost{...} }
```

### agent-core 自定义消息（`types.ts:313-393`）
通过**声明合并**扩展（`CustomAgentMessages` 接口，应用可加）：
```ts
BashExecutionMessage = { role:"bashExecution"; command; output; exitCode?; cancelled; truncated; fullOutputPath?; timestamp; excludeFromContext? }
CustomMessage<T>     = { role:"custom"; customType; content: string|(Text|Image)[]; display; details?:T; timestamp }
BranchSummaryMessage = { role:"branchSummary"; summary; fromId; timestamp }
CompactionSummaryMessage = { role:"compactionSummary"; summary; tokensBefore; timestamp: number|string; tokensAfter?; firstKeptEntryId?; details? }

type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```
`HarnessMessage`（`messages.ts:20`）= AgentMessage ∪ 上述四类，用于内部 normalize。

## 5.2 工具契约（`types.ts:428-485`）

```ts
interface AgentTool<TParameters extends TSchema = TSchema, TDetails = unknown> extends Tool<TParameters> {
  name; description; parameters: TParameters;   // 来自 Tool
  label: string;                                 // UI 显示名
  prepareArguments?: (args: unknown) => Static<TParameters>;  // 校验前兼容 shim
  execute: (toolCallId, params: Static<TParameters>, signal?, onUpdate?) => Promise<AgentToolResult<TDetails>>;
  executionMode?: "sequential" | "parallel";    // 单工具覆盖
}

interface AgentToolResult<T> {
  content: (TextContent|ImageContent)[];   // 回模型的内容
  details: T;                               // 结构化细节（日志/UI）
  progress?: AgentToolProgress;             // 公开进度（绝非模型内容）
  terminate?: boolean;                      // 早停提示（整批都 true 才停）
}
interface AgentToolProgress { text; visibility:"channel"; privacy:"public"; id? }
type AgentToolUpdateCallback<T> = (partial: AgentToolResult<T>) => void;
type AgentToolCall = Extract<AssistantMessage["content"][number], { type:"toolCall" }>;
```

## 5.3 循环配置（`types.ts:143-304`）

```ts
interface AgentLoopConfig extends SimpleStreamOptions {
  model: Model;
  thinkingLevel?: ThinkingLevel;               // off|minimal|low|medium|high|xhigh|max
  convertToLlm: (msgs: AgentMessage[]) => Message[] | Promise<...>;   // 必填
  transformContext?: (msgs, signal?) => Promise<AgentMessage[]>;      // 上下文改写
  getApiKey?: (provider) => Promise<string|undefined>|...;            // 动态 key（过期 token）
  shouldStopAfterTurn?: (ctx) => boolean|Promise<boolean>;            // turn 后优雅停
  prepareNextTurn?: (ctx) => AgentLoopTurnUpdate|undefined|Promise<...>; // 换模型/上下文/thinking
  getSteeringMessages?: () => Promise<AgentMessage[]>;                // 中途纠偏队列
  getFollowUpMessages?: () => Promise<AgentMessage[]>;                // 停下后追加队列
  toolExecution?: "sequential"|"parallel";     // 默认 parallel
  beforeToolCall?: (ctx, signal?) => Promise<{block?;reason?}|undefined>;  // 拦截（权限）
  resolveDeferredTool?: (ctx, signal?) => Promise<AgentTool|undefined>;    // 延迟工具水合
  afterToolCall?: (ctx, signal?) => Promise<AfterToolCallResult|undefined>; // 结果改写
}
interface AgentLoopTurnUpdate { context?: AgentContext; model?: Model; thinkingLevel?: ThinkingLevel }
```
回调上下文：`BeforeToolCallContext`（assistantMessage/toolCall/args/context）、`AfterToolCallContext`（+result/isError）、`ShouldStopAfterTurnContext`（message/toolResults/context/newMessages）、`DeferredToolCallContext`。

## 5.4 上下文与状态

```ts
interface AgentContext { systemPrompt: string; messages: AgentMessage[]; tools?: AgentTool[] }  // 传入循环的快照

interface AgentState {                          // Agent 公开状态
  systemPrompt; model; thinkingLevel;
  get/set tools: AgentTool[];                   // 赋值时拷贝顶层数组
  get/set messages: AgentMessage[];
  readonly isStreaming;                          // 处理中（直到 agent_end 监听器结算）
  readonly streamingMessage?;                    // 当前流式消息
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?;
}
```

## 5.5 事件联合（`types.ts:504-533`）

```ts
type AgentEvent =
  | { type:"agent_start" }
  | { type:"agent_end"; messages: AgentMessage[] }
  | { type:"turn_start" }
  | { type:"turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type:"message_start"; message }
  | { type:"message_update"; message; assistantMessageEvent: AssistantMessageEvent }  // 仅流式 assistant
  | { type:"message_end"; message }
  | { type:"tool_execution_start"; toolCallId; toolName; args }
  | { type:"tool_execution_update"; toolCallId; toolName; args; partialResult }
  | { type:"tool_execution_end"; toolCallId; toolName; result; isError; executionStarted? };
```
**事件序契约**（重写必须保持）：`agent_start` → (每 turn: `turn_start` → `message_start`/`message_update*`/`message_end`（assistant）→ `tool_execution_start`/`update*`/`end`（每工具）→ tool result 的 `message_start`/`message_end` → `turn_end`) → `agent_end`。`agent_end` 是终态事件，但 await 监听器结算后 Agent 才 idle。

## 5.6 会话树条目（`harness/types.ts:352-453`）

```ts
interface SessionTreeEntryBase { type; id; parentId: string|null; timestamp: string; appendMode?:"side" }

type SessionTreeEntry =
  | MessageEntry              { type:"message"; message: AgentMessage }
  | ThinkingLevelChangeEntry  { type:"thinking_level_change"; thinkingLevel }
  | ModelChangeEntry          { type:"model_change"; provider; modelId }
  | CompactionEntry<T>        { type:"compaction"; summary; firstKeptEntryId; tokensBefore; details?; fromHook? }
  | BranchSummaryEntry<T>     { type:"branch_summary"; fromId; summary; details?; fromHook? }
  | CustomEntry<T>            { type:"custom"; customType; data? }       // 不回放进上下文
  | CustomMessageEntry<T>     { type:"custom_message"; customType; content; details?; display }  // 可回放
  | LabelEntry                { type:"label"; targetId; label? }
  | SessionInfoEntry          { type:"session_info"; name? }
  | LeafEntry                 { type:"leaf"; targetId: string|null; appendParentId? };

interface SessionContext { messages: AgentMessage[]; thinkingLevel: string; model: {provider;modelId}|null }
interface SessionMetadata { id; createdAt }
interface JsonlSessionMetadata extends SessionMetadata { cwd; path; parentSessionPath? }
```

### SessionStorage 接口（`harness/types.ts:472`）
```ts
interface SessionStorage<TMetadata> {
  getMetadata(); getLeafId(); getAppendParentId?(); setLeafId(id);
  createEntryId(); appendEntry(entry); getEntry(id);
  findEntries<TType>(type); getLabel(id); getPathToRoot(leafId); getEntries();
}
```

## 5.7 执行环境（`harness/types.ts:113-350`）

```ts
type FileKind = "file"|"directory"|"symlink";
type FileErrorCode = "aborted"|"not_found"|"permission_denied"|"not_directory"|"is_directory"|"invalid"|"not_supported"|"unknown";
class FileError extends Error { code; path? }
class ExecutionError extends Error { code: "aborted"|"timeout"|"shell_unavailable"|"spawn_error"|"callback_error"|"unknown" }

interface FileSystem {           // 所有方法返回 Result，绝不抛
  cwd;
  absolutePath/joinPath/readTextFile/readTextLines/readBinaryFile/writeFile/appendFile/
  fileInfo/listDir/canonicalPath/exists/createDir/remove/createTempDir/createTempFile/cleanup;
}
interface Shell { exec(command, options?): Promise<Result<{stdout;stderr;exitCode}, ExecutionError>>; cleanup() }
interface ExecutionEnv extends FileSystem, Shell {}
interface ExecutionEnvExecOptions { cwd?; env?; timeout?; abortSignal?; onStdout?; onStderr? }
```

## 5.8 Harness 类型

### 资源/选项
```ts
interface Skill { name; description; content; filePath; promptVersion?; disableModelInvocation? }
interface PromptTemplate { name; description?; content }
interface AgentHarnessResources<TSkill,TPromptTemplate> { promptTemplates?; skills? }
interface AgentHarnessStreamOptions { transport?; timeoutMs?; maxRetries?; maxRetryDelayMs?; headers?; metadata?; cacheRetention? }
type AgentHarnessPhase = "idle"|"turn"|"compaction"|"branch_summary"|"retry";
type PendingSessionWrite = Omit<SessionTreeEntry, "id"|"parentId"|"timestamp">;  // 运行中累积待写
```

### 钩子事件与返回值映射（`harness/types.ts:705`）
```ts
type AgentHarnessEventResultMap = {
  before_agent_start: BeforeAgentStartResult|undefined;       // 改 prompt/系统提示
  context: ContextResult|undefined;                            // 改上下文消息
  before_provider_request: BeforeProviderRequestResult|undefined;  // patch 请求选项
  before_provider_payload: BeforeProviderPayloadResult|undefined;  // 改 payload
  after_provider_response: undefined;
  tool_call: ToolCallResult|undefined;                         // block 工具
  tool_result: ToolResultPatch|undefined;                      // 改工具结果
  session_before_compact: SessionBeforeCompactResult|undefined;
  session_compact: undefined;
  session_before_tree: SessionBeforeTreeResult|undefined;
  session_tree: undefined;
  model_select|thinking_level_select|resources_update|queue_update|save_point|abort|settled: undefined;
};
```
另有 own 事件：`QueueUpdateEvent`/`SavePointEvent`/`AbortEvent`/`SettledEvent`/`ModelSelectEvent`/`ThinkingLevelSelectEvent`/`ResourcesUpdateEvent` 等。

## 5.9 压缩类型（`compaction.ts`）

```ts
interface CompactionSettings { enabled; reserveTokens; keepRecentTokens }
const DEFAULT_COMPACTION_SETTINGS = { enabled:true, reserveTokens:16384, keepRecentTokens:20000 }
interface CompactionResult<T> { summary; firstKeptEntryId; tokensBefore; details? }
interface CompactionPreparation { firstKeptEntryId; messagesToSummarize; turnPrefixMessages; isSplitTurn; tokensBefore; previousSummary?; fileOps; settings }
interface CompactionDetails { readFiles: string[]; modifiedFiles: string[] }
interface ContextUsageEstimate { tokens; usageTokens; trailingTokens; lastUsageIndex: number|null }
interface CutPointResult { firstKeptEntryIndex; turnStartIndex; isSplitTurn }
interface FileOperations { read: Set<string>; written: Set<string>; edited: Set<string> }
```

## 5.10 Result 与错误（`harness/types.ts:14-237`）

```ts
type Result<V,E> = { ok:true; value:V } | { ok:false; error:E };
ok(v) / err(e) / toError(unknown): Error
class CompactionError    { code:"aborted"|"summarization_failed"|"invalid_session"|"unknown" }
class BranchSummaryError { code:"aborted"|"summarization_failed"|"invalid_session" }
class SessionError       { code:"not_found"|"invalid_session"|"invalid_entry"|"invalid_fork_target"|"storage"|"unknown" }
class AgentHarnessError  { code:"busy"|"invalid_state"|"invalid_argument"|"session"|"hook"|"auth"|"compaction"|"branch_summary"|"unknown" }
```

## 5.11 注入运行时（`runtime-deps.ts`）
```ts
interface AgentCoreRuntimeDeps { streamSimple: StreamFn; completeSimple: CompleteSimpleFn }
type AgentCoreStreamRuntimeDeps     = Pick<..., "streamSimple">;
type AgentCoreCompletionRuntimeDeps = Pick<..., "completeSimple">;
```
