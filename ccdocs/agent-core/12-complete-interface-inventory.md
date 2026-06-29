# 12 · agent-core 完整接口清单（无遗漏 · 导出 × 调用方）

> 本文是 02/03 章的**穷尽版**：列出 `@openclaw/agent-core` 经 `index.ts`/`node.ts` 暴露的**每一个**导出符号（含此前摘要遗漏项），以及每个调用方导入的**确切符号**。
> 提取方式：对全部 25 个非测试源文件做 `export` 抽取 + 对全仓库 import 做符号聚合。

---

## A. 完整导出符号清单（按源文件）

### A.1 循环与 Agent
| 文件 | 导出符号 |
|---|---|
| `agent-loop.ts` | `agentLoop`, `agentLoopContinue`, `runAgentLoop`, `runAgentLoopContinue`, `AgentEventSink`(type) |
| `agent.ts` | `Agent`(class), `AgentOptions`(type), `QueueMode`(re-export) |
| `reasoning.ts` | `resolveAgentReasoningOption` |
| `runtime-deps.ts` | `AgentCoreRuntimeDeps`, `AgentCoreStreamRuntimeDeps`, `AgentCoreCompletionRuntimeDeps`(types), `resolveAgentCoreStreamFn`, `resolveAgentCoreCompleteFn` |

### A.2 类型（types.ts，全部）
`AfterToolCallContext`, `AfterToolCallResult`, `AgentContext`, `AgentEvent`, `AgentLoopConfig`, `AgentLoopTurnUpdate`, `AgentMessage`, `AgentState`, `AgentTool`, `AgentToolCall`, `AgentToolProgress`, `AgentToolResult`, `AgentToolUpdateCallback`, `BashExecutionMessage`, `BeforeToolCallContext`, `BeforeToolCallResult`, `BranchSummaryMessage`, `CompactionSummaryMessage`, `CustomAgentMessages`, `CustomMessage`, `DeferredToolCallContext`, `PrepareNextTurnContext`, `QueueMode`, `ShouldStopAfterTurnContext`, `StreamFn`, `ThinkingLevel`, `ToolExecutionMode`。

### A.3 Harness
| 文件 | 导出符号 |
|---|---|
| `harness/agent-harness.ts` | `CoreAgentHarness`(class, 别名 `AgentHarness`) |
| `harness/types.ts` | 见下方 A.7（最大类型集） |
| `harness/messages.ts` | `convertToLlm`, `asAgentMessage`, `bashExecutionToText`, `createBranchSummaryMessage`, `createCompactionSummaryMessage`, `createCustomMessage`, `HarnessMessage`(type), `COMPACTION_SUMMARY_PREFIX/SUFFIX`, `BRANCH_SUMMARY_PREFIX/SUFFIX` |
| `harness/skills.ts` | `formatSkillInvocation` |
| `harness/prompt-template-arguments.ts` | `formatPromptTemplateInvocation`, **`parseCommandArgs`**, **`substituteArgs`** |

### A.4 会话存储（session/*）
| 文件 | 导出符号 |
|---|---|
| `session/session.ts` | `Session`(class), `buildSessionContext` |
| `session/jsonl-storage.ts` | `JsonlSessionStorage`, **`loadJsonlSessionMetadata`** |
| `session/memory-storage.ts` | **`InMemorySessionStorage`**（注意：实际类名是 `InMemorySessionStorage`，02 章曾写作 `MemorySessionStorage`，以此为准） |
| `session/storage-base.ts` | **`BaseSessionStorage`**, **`appendParentIdAfterEntry`**, **`leafIdUpdateAfterEntry`** |
| `session/timestamps.ts` | **`parseSessionTimestampMs`**, **`requireSessionTimestampMs`** |
| `session/uuid.ts` | `uuidv7` |

> **注**：`index.ts` 显式只 re-export 了 `jsonl-storage`/`memory-storage`/`session`/`uuid` 四个 session 文件 + `uuidv7`。`storage-base.ts`/`timestamps.ts` 的符号是否对外可见取决于 `index.ts` 的 `export *`——实测 `index.ts` 未直接导出 `storage-base`/`timestamps`，它们是内部模块（被 jsonl/session 复用）。

### A.5 压缩（compaction/*）
| 文件 | 导出符号 |
|---|---|
| `compaction/compaction.ts` | `compact`, `prepareCompaction`, `shouldCompact`, `findCutPoint`, `findTurnStartIndex`, `generateSummary`, `estimateTokens`, `estimateContextTokens`, `calculateContextTokens`, `getLastAssistantUsage`, `serializeConversation`(re-export), `DEFAULT_COMPACTION_SETTINGS`, 类型 `CompactionDetails`/`CompactionPreparation`/`CompactionResult`/`CompactionSettings`/`ContextUsageEstimate`/`CutPointResult`，常量 `SUMMARIZATION_SYSTEM_PROMPT`(模块内导出，但 **index.ts 未 re-export**) |
| `compaction/branch-summarization.ts` | `generateBranchSummary`, `collectEntriesForBranchSummary`, `collectEntriesForBranchSummaryFromBranches`, `prepareBranchEntries`, 类型 `BranchPreparation`/`BranchPathEntry`/`BranchSummaryDetails`/`CollectBranchPathEntriesResult`/`CollectEntriesResult`/`GenerateBranchSummaryOptions` |
| `compaction/utils.ts` | **`computeFileLists`**, **`createFileOps`**, **`extractFileOpsFromMessage`**, **`formatFileOperations`**, `serializeConversation`, `FileOperations`(type) —— 注：index.ts 未直接 re-export utils，多为内部用，`serializeConversation` 经 compaction.ts re-export 暴露 |

### A.6 执行环境与工具
| 文件 | 导出符号 |
|---|---|
| `harness/env/nodejs.ts` | `NodeExecutionEnv`, **`resolveExecTimeoutMs`**（经 `node.ts` 导出 `NodeExecutionEnv`） |
| `harness/env/kill-tree.ts` | `killProcessTree`, **`signalProcessTree`**, `KillProcessTreeOptions`(type) |
| `harness/utils/truncate.ts` | `truncateHead`, `truncateTail`, `truncateLine`, `formatSize`, `DEFAULT_MAX_BYTES`, `DEFAULT_MAX_LINES`, `GREP_MAX_LINE_LENGTH`, `TruncationOptions`, `TruncationResult`(types) |
| `validation.ts` | `validateToolArguments`, `validateToolCall`（re-export llm-core） |
| `llm.ts` | `export * from "@openclaw/llm-core"`（整个 llm-core 透传） |

### A.7 harness/types.ts 全部类型（最大集）
**会话条目**：`SessionTreeEntry`, `SessionTreeEntryBase`, `MessageEntry`, `ThinkingLevelChangeEntry`, `ModelChangeEntry`, `CompactionEntry`, `BranchSummaryEntry`, `CustomEntry`, `CustomMessageEntry`, `LabelEntry`, `SessionInfoEntry`, `LeafEntry`。
**会话**：`Session`(re-export), `SessionContext`, `SessionMetadata`, `JsonlSessionMetadata`, `SessionStorage`, `PendingSessionWrite`。
**执行环境**：`ExecutionEnv`, `FileSystem`, `Shell`, `ExecutionEnvExecOptions`, `FileInfo`, `FileKind`, `FileError`, `FileErrorCode`, `ExecutionError`, `ExecutionErrorCode`。
**Harness 选项/资源**：`AgentHarnessOptions`, `AgentHarnessResources`, `AgentHarnessStreamOptions`, `AgentHarnessStreamOptionsPatch`, `AgentHarnessPhase`, `Skill`, `PromptTemplate`, `NavigateTreeResult`。
**钩子事件**：`AgentHarnessEvent`, `AgentHarnessOwnEvent`, `AgentHarnessEventResultMap`, `QueueUpdateEvent`, `SavePointEvent`, `AbortEvent`, `SettledEvent`, `BeforeAgentStartEvent`, `ContextEvent`, `BeforeProviderRequestEvent`, `BeforeProviderPayloadEvent`, `AfterProviderResponseEvent`, `ToolCallEvent`, `ToolResultEvent`, `SessionBeforeCompactEvent`, `SessionCompactEvent`, `SessionBeforeTreeEvent`, `SessionTreeEvent`, `ModelSelectEvent`, `ThinkingLevelSelectEvent`, `ResourcesUpdateEvent`。
**钩子返回**：`BeforeAgentStartResult`, `ContextResult`, `BeforeProviderRequestResult`, `BeforeProviderPayloadResult`, `ToolCallResult`, `ToolResultPatch`, `SessionBeforeCompactResult`, `SessionBeforeTreeResult`, `AbortResult`, `CompactResult`。
**压缩/分支**：`CompactionSettings`, `CompactionPreparation`, `CompactionEntry`, `TreePreparation`, `GenerateBranchSummaryOptions`, `BranchSummaryResult`, `FileOperations`。
**错误与 Result**：`Result`, `ok`, `err`, `toError`, `AgentHarnessError`, `AgentHarnessErrorCode`, `SessionError`, `SessionErrorCode`, `CompactionError`, `CompactionErrorCode`, `BranchSummaryError`, `BranchSummaryErrorCode`。

---

## B. 此前 02/03 章遗漏/需更正项（勘误）

| 项 | 更正 |
|---|---|
| 内存存储类名 | 实际是 **`InMemorySessionStorage`**（非 `MemorySessionStorage`） |
| `parseCommandArgs`/`substituteArgs` | prompt-template-arguments.ts 还导出这两个，02 章未列 |
| `loadJsonlSessionMetadata` | jsonl-storage.ts 额外导出，未列 |
| `resolveExecTimeoutMs` | nodejs.ts 额外导出，未列 |
| `signalProcessTree` | kill-tree.ts 额外导出，未列 |
| `truncate.ts` 全套 | `truncateHead/Tail/Line`/`formatSize`/常量/类型，02 章仅泛指 |
| `compaction/utils.ts` | `computeFileLists/createFileOps/extractFileOpsFromMessage/formatFileOperations` 为压缩内部 API，02 章未列 |
| `SUMMARIZATION_SYSTEM_PROMPT` | compaction.ts 内 export，但 **index.ts 未 re-export**（内部常量，非公开面） |
| `storage-base.ts`/`timestamps.ts` | 内部模块，**index.ts 未 re-export**，不属公开 API |

> 即：**真正的对外公开面 = index.ts 显式 re-export 的集合**（02 章 2.1 的清单 + 本文 A 中标注「index 未 re-export」的需排除）。本文把「文件级导出」与「包级公开」区分清楚，重写时以 `index.ts` 的 re-export 为对外契约边界。

---

## C. 完整调用方清单（经 `openclaw/plugin-sdk/agent-core`，按文件聚合）

| 调用方目录 | 文件数 | 角色 |
|---|---:|---|
| `extensions/discord` | 7 | 渠道扩展（取消息/工具类型） |
| `extensions/xai` | 4 | provider 扩展 |
| `extensions/slack` | 3 | 渠道扩展 |
| `extensions/anthropic-vertex` | 2 | provider |
| **`src/agents/sessions`** | 1 | **核心：AgentSession 取 `Agent`** |
| **`src/agents/embedded-agent-runner`** | 1 | **核心：运行编排** |
| `extensions/{xiaomi,whatsapp,vllm,telegram,qwen,openrouter,openai,ollama,memory-lancedb,matrix,lmstudio,kimi-coding,google,github-copilot,fireworks,codex,cloudflare-ai-gateway,browser,anthropic,amazon-bedrock-mantle,amazon-bedrock}` | 各 1 | provider/渠道扩展（多为 import type） |

**合计**：经此说明符 ~37 个文件（核心 2 + 扩展 ~35）。另有 14 个文件经相对路径 `../../packages/agent-core/src/*` 导入（facade 自身 + sessions 层 + process/kill-tree）。

## D. 调用方实际导入的符号频次（聚合）

| 符号 | 出现次数 | 类别 |
|---|---:|---|
| `StreamFn` | 59 | 类型（provider 扩展实现流式） |
| `AgentMessage` | 47 | 类型（消息） |
| `AgentToolResult` | 21 | 类型（工具结果） |
| `AgentTool` | 7 | 类型（工具定义） |
| `AgentEvent` | 2 | 类型 |
| `Agent` | 2 | 类（核心驱动者 new Agent） |
| `AfterToolCallContext` | 1 | 类型 |

**结论**：
- **绝大多数调用方只 `import type`**（StreamFn/AgentMessage/AgentToolResult/AgentTool 占绝对多数）——它们消费**类型契约**，不实例化。
- **唯一实例化 `Agent` 类的是核心 2 个文件**（sessions + embedded-runner，经 facade）。
- `StreamFn` 高居榜首（59）说明：**70 个 provider 扩展实现的都是 `StreamFn` 契约**，这是 agent-core 与 provider 生态的核心耦合类型。

---

## E. 重写时的「对外契约冻结清单」（最小不可破坏集）

按调用方实际依赖频次，重写**必须**保持签名/字段不变的符号（破坏即上层断）：
1. **`StreamFn`**（59 处）—— provider 生态契约，最高优先。
2. **`AgentMessage`** 及其联合成员（47 处）。
3. **`AgentToolResult`** / **`AgentTool`**（28 处）—— 工具契约。
4. **`Agent` 类**（构造 + `prompt`/`continue`/`steer`/`followUp`/`subscribe`/`abort`/`state`）—— 核心驱动。
5. **`AgentEvent`** 联合 + 事件序 —— AgentSession 事件处理依赖。
6. **`AgentCoreRuntimeDeps`** —— facade 注入点。
7. **`Session`/`buildSessionContext`/`convertToLlm`/`prepareCompaction`/`compact`/`estimateTokens`** —— sessions 层与压缩复用。
8. **`index.ts` re-export 清单**本身 —— SDK 子路径稳定面。

其余（truncate/kill-tree/uuid/prompt-template 等工具函数、内部 storage-base/timestamps/utils）属低耦合，重写自由度高。
