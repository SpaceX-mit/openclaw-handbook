# OpenClaw API参考手册

## 目录

1. [Agent Core API](#1-agent-core-api)
2. [Tool API](#2-tool-api)
3. [Session API](#3-session-api)
4. [Plugin SDK API](#4-plugin-sdk-api)
5. [Channel API](#5-channel-api)
6. [LLM Runtime API](#6-llm-runtime-api)

---

## 1. Agent Core API

### 1.1 Agent Class

```typescript
// packages/agent-core/src/agent.ts

class Agent {
  /**
   * 创建Agent实例
   */
  constructor(options?: AgentOptions)
  
  /**
   * 同步运行Agent
   * @param prompts - 用户输入消息
   * @returns 最终的Assistant消息列表
   */
  async run(prompts: AgentMessage[]): Promise<AgentMessage[]>
  
  /**
   * 流式运行Agent
   * @param prompts - 用户输入消息
   * @returns EventStream用于监听事件
   */
  runStream(prompts: AgentMessage[]): EventStream<AgentEvent, AgentMessage[]>
  
  /**
   * 从当前状态继续运行
   * @returns 新产生的消息
   */
  async continue(): Promise<AgentMessage[]>
  
  /**
   * 停止当前运行
   */
  stop(): void
  
  /**
   * 检查是否正在运行
   */
  isRunning(): boolean
}
```

### 1.2 AgentOptions

```typescript
interface AgentOptions {
  /** 初始状态 */
  initialState?: Partial<AgentState>
  
  /** 消息转换为LLM格式 */
  convertToLlm?: (messages: AgentMessage[]) => Message[] | Promise<Message[]>
  
  /** 上下文转换 */
  transformContext?: (
    messages: AgentMessage[],
    signal?: AbortSignal
  ) => Promise<AgentMessage[]>
  
  /** 流函数 */
  streamFn?: StreamFn
  
  /** API Key获取 */
  getApiKey?: (provider: string) => Promise<string | undefined>
  
  /** Tool Call前钩子 */
  beforeToolCall?: (
    context: BeforeToolCallContext
  ) => Promise<BeforeToolCallResult | undefined>
  
  /** Tool Call后钩子 */
  afterToolCall?: (
    context: AfterToolCallContext
  ) => Promise<AfterToolCallResult | undefined>
  
  /** 准备下一轮 */
  prepareNextTurn?: () => Promise<AgentLoopTurnUpdate | undefined>
  
  /** 工具执行模式 */
  toolExecution?: "sequential" | "parallel"
  
  /** 会话ID */
  sessionId?: string
  
  /** Reasoning预算 */
  thinkingBudgets?: ThinkingBudgets
}
```

### 1.3 AgentEvent

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AssistantMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_call_start"; toolCall: AgentToolCall }
  | { type: "tool_call_end"; toolCall: AgentToolCall; result: AgentToolResult }
  | { type: "error"; error: Error }
  | { type: "abort" }
```

### 1.4 AgentMessage

```typescript
interface AgentMessage {
  role: "user" | "assistant" | "toolResult"
  content: ContentBlock[]
  timestamp: number
  usage?: Usage
  stopReason?: StopReason
  errorMessage?: string
  model?: string
  provider?: string
  api?: string
}

type ContentBlock =
  | { type: "text"; text: string }
  | { type: "image"; url: string }
  | { type: "toolCall"; id: string; name: string; arguments: unknown }
  | { type: "toolResult"; toolCallId: string; content: ContentBlock[]; isError?: boolean }
```

---

## 2. Tool API

### 2.1 ToolDescriptor

```typescript
// src/tools/types.ts

interface ToolDescriptor {
  /** 工具名称 */
  name: string
  
  /** 显示标题 */
  title?: string
  
  /** 描述 */
  description: string
  
  /** 输入Schema */
  inputSchema: JsonObject
  
  /** 输出Schema */
  outputSchema?: JsonObject
  
  /** 所有者 */
  owner: ToolOwnerRef
  
  /** 执行器 */
  executor?: ToolExecutorRef
  
  /** 可用性条件 */
  availability?: ToolAvailabilityExpression
  
  /** 注解 */
  annotations?: JsonObject
  
  /** 排序键 */
  sortKey?: string
}

type ToolOwnerRef =
  | { kind: "core" }
  | { kind: "plugin"; pluginId: string }
  | { kind: "channel"; channelId: string; pluginId?: string }
  | { kind: "mcp"; serverId: string }

type ToolExecutorRef =
  | { kind: "core"; executorId: string }
  | { kind: "plugin"; pluginId: string; toolName: string }
  | { kind: "channel"; channelId: string; actionId: string }
  | { kind: "mcp"; serverId: string; toolName: string }
```

### 2.2 ToolAvailabilityExpression

```typescript
type ToolAvailabilityExpression =
  | ToolAvailabilitySignal
  | { allOf: readonly ToolAvailabilityExpression[] }
  | { anyOf: readonly ToolAvailabilityExpression[] }

type ToolAvailabilitySignal =
  | { kind: "always" }
  | { kind: "auth"; providerId: string }
  | { kind: "config"; path: readonly string[]; check?: "exists" | "non-empty" | "available" }
  | { kind: "env"; name: string }
  | { kind: "plugin-enabled"; pluginId: string }
  | { kind: "context"; key: string; equals?: JsonPrimitive }
```

### 2.3 ToolPlan

```typescript
interface ToolPlan {
  visible: readonly ToolPlanEntry[]
  hidden: readonly HiddenToolPlanEntry[]
}

interface ToolPlanEntry {
  descriptor: ToolDescriptor
  executor: ToolExecutorRef
}

interface HiddenToolPlanEntry {
  descriptor: ToolDescriptor
  diagnostics: readonly ToolAvailabilityDiagnostic[]
}

interface ToolAvailabilityDiagnostic {
  reason: ToolUnavailableReason
  signal?: ToolAvailabilitySignal
  message: string
}
```

### 2.4 ToolPlanner Functions

```typescript
// src/tools/planner.ts

/**
 * 构建工具计划
 */
function buildToolPlan(options: BuildToolPlanOptions): ToolPlan

interface BuildToolPlanOptions {
  descriptors: readonly ToolDescriptor[]
  availability?: ToolAvailabilityContext
}

// src/tools/availability.ts

/**
 * 评估工具可用性
 */
function evaluateToolAvailability(params: {
  descriptor: ToolDescriptor
  context: ToolAvailabilityContext
}): ToolAvailabilityDiagnostic[]
```

---

## 3. Session API

### 3.1 Session Interface

```typescript
// packages/agent-core/src/harness/session/session.ts

interface Session {
  id: string
  createdAt: number
  updatedAt: number
  messages: AgentMessage[]
  tools: AgentTool[]
  systemPrompt: string
  model: Model
  metadata?: SessionMetadata
  
  addMessage(message: AgentMessage): void
  pruneMessages(options: PruneOptions): AgentMessage[]
  checkpoint(): SessionSnapshot
  restore(snapshot: SessionSnapshot): void
  getContext(): AgentContext
}
```

### 3.2 SessionManager

```typescript
// src/agents/sessions/

interface SessionManager {
  create(params: CreateSessionParams): Promise<Session>
  get(id: string): Promise<Session | null>
  getOrCreate(id: string): Promise<Session>
  update(session: Session): Promise<void>
  delete(id: string): Promise<void>
  list(filter?: SessionFilter): Promise<Session[]>
}

interface CreateSessionParams {
  id?: string
  systemPrompt?: string
  model?: Model
  tools?: AgentTool[]
  metadata?: SessionMetadata
}

interface SessionFilter {
  agentId?: string
  channelId?: string
  status?: "active" | "archived"
  createdAfter?: number
  createdBefore?: number
}
```

### 3.3 SessionSnapshot

```typescript
interface SessionSnapshot {
  id: string
  timestamp: number
  messages: AgentMessage[]
  systemPrompt: string
  model: Model
  metadata?: SessionMetadata
}
```

---

## 4. Plugin SDK API

### 4.1 Plugin Interface

```typescript
// packages/plugin-sdk/src/plugin-runtime.ts

interface Plugin {
  id: string
  name: string
  version: string
  description?: string
  
  setup(runtime: PluginRuntime): Promise<void> | void
  teardown?(): Promise<void> | void
}
```

### 4.2 PluginRuntime

```typescript
interface PluginRuntime {
  /** 注册工具 */
  registerTools(tools: ToolDescriptor[]): void
  
  /** 注册单个工具的executor */
  registerToolExecutor(executor: ToolExecutor): void
  
  /** 注册技能 */
  registerSkills(skills: Skill[]): void
  
  /** 注册Provider */
  registerProvider(provider: ModelProvider): void
  
  /** 注册Channel */
  registerChannel(channel: ChannelPlugin): void
  
  /** 注册Hook */
  registerHook(hook: Hook): void
  
  /** 获取配置 */
  getConfig<T>(path: string[]): T | undefined
  
  /** 设置配置 */
  setConfig<T>(path: string[], value: T): void
  
  /** 获取状态 */
  getState(): PluginState
  
  /** 设置状态 */
  setState(state: Partial<PluginState>): void
  
  /** 日志 */
  logger: Logger
}
```

### 4.3 ToolExecutor

```typescript
interface ToolExecutor {
  pluginId: string
  toolName: string
  execute(
    args: unknown,
    context: ToolExecutionContext
  ): Promise<ToolExecutionResult>
}

interface ToolExecutionContext {
  session: Session
  userId: string
  channelId?: string
  metadata?: Record<string, unknown>
}

interface ToolExecutionResult {
  success: boolean
  output?: unknown
  error?: string
  details?: Record<string, unknown>
}
```

### 4.4 Hook System

```typescript
interface Hook {
  name: string
  beforeAgentRun?: (params: BeforeAgentRunParams) => Promise<BeforeAgentRunResult | undefined>
  afterAgentRun?: (params: AfterAgentRunParams) => Promise<void>
  beforeToolCall?: (params: BeforeToolCallParams) => Promise<BeforeToolCallResult | undefined>
  afterToolCall?: (params: AfterToolCallParams) => Promise<AfterToolCallResult | undefined>
  onMessage?: (params: OnMessageParams) => Promise<void>
}

interface BeforeAgentRunParams {
  harness: AgentHarness
  prompts: AgentMessage[]
  context: AgentContext
}

interface BeforeAgentRunResult {
  block?: boolean
  reason?: string
  overridePrompts?: AgentMessage[]
}
```

---

## 5. Channel API

### 5.1 ChannelPlugin

```typescript
// packages/plugin-sdk/src/channel.ts

interface ChannelPlugin {
  id: string
  name: string
  
  setup(runtime: PluginRuntime): Promise<void> | void
  teardown?(): Promise<void> | void
  
  createTransport(config: ChannelConfig): ChannelTransport
  createSession(config: ChannelSessionConfig): ChannelSession
}
```

### 5.2 ChannelTransport

```typescript
interface ChannelTransport {
  /** 发送消息 */
  send(message: OutboundMessage): Promise<void>
  
  /** 接收消息 */
  receive(handler: MessageHandler): void
  
  /** 关闭连接 */
  close(): Promise<void>
  
  /** 连接状态 */
  getStatus(): ConnectionStatus
}

type MessageHandler = (message: InboundMessage) => void | Promise<void>

interface ConnectionStatus {
  connected: boolean
  lastConnected?: number
  error?: string
}
```

### 5.3 ChannelSession

```typescript
interface ChannelSession {
  id: string
  channelId: string
  senderId: string
  
  /** 发送消息 */
  send(message: OutboundMessage): Promise<void>
  
  /** 更新typing状态 */
  updateTyping(typing: boolean): Promise<void>
  
  /** 获取会话信息 */
  getInfo(): SessionInfo
}

interface SessionInfo {
  id: string
  channelId: string
  senderId: string
  senderName?: string
  threadId?: string
  metadata?: Record<string, unknown>
}
```

### 5.4 Message Types

```typescript
interface InboundMessage {
  channelId: string
  sender: SenderInfo
  content: string
  attachments?: Attachment[]
  timestamp: number
  threadId?: string
  replyTo?: string
  raw?: unknown
}

interface OutboundMessage {
  content: string
  attachments?: Attachment[]
  replyTo?: string
  threadId?: string
  extra?: Record<string, unknown>
}

interface SenderInfo {
  id: string
  name?: string
  username?: string
  avatar?: string
}

interface Attachment {
  type: "image" | "video" | "audio" | "file"
  url?: string
  content?: string
  filename?: string
  mimeType?: string
}
```

---

## 6. LLM Runtime API

### 6.1 ModelProvider

```typescript
// packages/plugin-sdk/src/provider-runtime.ts

interface ModelProvider {
  id: string
  api: "openai" | "anthropic" | "google" | "custom"
  
  /** 发现可用模型 */
  discover(): Promise<Model[]>
  
  /** 创建运行时 */
  createRuntime(config: ProviderConfig): ProviderRuntime
  
  /** 验证配置 */
  validateConfig(config: unknown): ValidationResult
  
  /** 获取认证说明 */
  getAuthInstructions(): AuthInstructions
}

interface ValidationResult {
  valid: boolean
  error?: string
  warnings?: string[]
}

interface AuthInstructions {
  type: "api-key" | "oauth" | "env"
  url?: string
  instructions?: string
  envVar?: string
}
```

### 6.2 ProviderRuntime

```typescript
interface ProviderRuntime {
  /** 流式调用 */
  stream(
    model: Model,
    messages: Message[],
    options: StreamOptions
  ): Promise<AsyncIterable<ProviderEvent>>
  
  /** 完整调用（非流式） */
  complete?(
    model: Model,
    messages: Message[],
    options: CompleteOptions
  ): Promise<ProviderResponse>
}

interface StreamOptions {
  temperature?: number
  maxTokens?: number
  reasoning?: ReasoningOption
  stop?: string[]
  seed?: number
  presencePenalty?: number
  frequencyPenalty?: number
  headers?: Record<string, string>
  metadata?: Record<string, unknown>
  timeout?: number
}

interface CompleteOptions extends StreamOptions {
  responseFormat?: "text" | "json" | "json-schema"
  jsonSchema?: object
}
```

### 6.3 Model

```typescript
interface Model {
  id: string
  name: string
  api: string
  provider: string
  baseUrl?: string
  reasoning?: boolean
  input: ContentFormat[]
  cost: ModelCost
  contextWindow: number
  maxTokens: number
  supportedFeatures?: ModelFeatures
}

interface ModelCost {
  input: number
  output: number
  cacheRead?: number
  cacheWrite?: number
}

type ContentFormat = "text" | "images" | "audio" | "video"

interface ModelFeatures {
  streaming?: boolean
  functionCalling?: boolean
  jsonMode?: boolean
  vision?: boolean
}
```

### 6.4 ProviderEvent

```typescript
type ProviderEvent =
  | { type: "start" }
  | { type: "text_start"; contentIndex: number }
  | { type: "text_delta"; contentIndex: number; delta: string }
  | { type: "text_end"; contentIndex: number }
  | { type: "thinking_start"; contentIndex: number }
  | { type: "thinking_delta"; contentIndex: number; delta: string }
  | { type: "thinking_end"; contentIndex: number }
  | { type: "toolcall_start"; contentIndex: number; id: string; name: string }
  | { type: "toolcall_delta"; contentIndex: number; delta: string }
  | { type: "toolcall_end"; contentIndex: number }
  | { type: "done"; reason: StopReason; usage: Usage }
  | { type: "error"; error: string; code?: string }

type StopReason = "stop" | "length" | "toolUse" | "contentFilter" | "error" | "aborted"

interface Usage {
  input: number
  output: number
  cacheRead?: number
  cacheWrite?: number
  totalTokens?: number
  cost?: {
    input: number
    output: number
    cacheRead?: number
    cacheWrite?: number
    total: number
  }
}
```

---

## 附录: 常见错误码

| 错误码 | 说明 | 处理建议 |
|--------|------|----------|
| `tool_not_found` | 工具不存在 | 检查工具名称和注册 |
| `tool_execution_failed` | 工具执行失败 | 检查工具实现和参数 |
| `auth_missing` | 缺少认证 | 配置API Key |
| `rate_limit_exceeded` | 超出速率限制 | 减慢请求频率 |
| `context_overflow` | 上下文溢出 | 启用Compaction |
| `model_not_found` | 模型不存在 | 检查模型ID |
| `invalid_config` | 配置无效 | 检查配置文件 |

---

*文档生成时间: 2026-06-23*
