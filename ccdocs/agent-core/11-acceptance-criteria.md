# 11 · 验收标准（可据以验证重写正确性）

本节给出**可执行的验收清单**。若要让 AI 从零重写 agent-core，重写产物须逐条满足以下标准。分为：契约一致性、循环行为、工具执行、会话/压缩、Harness、错误/中止、非功能。

---

## A. 对外契约一致性（破坏即上层全断）

- [ ] **A1** 导出 `AgentCoreRuntimeDeps { streamSimple: StreamFn; completeSimple: CompleteSimpleFn }` 及 `resolveAgentCoreStreamFn`/`resolveAgentCoreCompleteFn`；未配置时抛含 "runtime dependency" 的明确错误。
- [ ] **A2** `index.ts` 导出清单与现状一致（Agent/runAgentLoop/agentLoop/CoreAgentHarness/Session/buildSessionContext/compact/prepareCompaction/shouldCompact/findCutPoint/estimateTokens/convertToLlm/DEFAULT_COMPACTION_SETTINGS/uuidv7/NodeExecutionEnv/... 见 02 章）。
- [ ] **A3** `Agent` 类方法签名不变：`prompt(msg|msg[])`/`prompt(text,images?)`、`continue()`、`steer(msg)`、`followUp(msg)`、`subscribe(listener)`、`abort()`、`get state`、`get signal`、`waitForIdle()`、`reset()`、队列 mode getter/setter。
- [ ] **A4** `AgentEvent` 联合的所有 9 个变体与字段不变；事件类型字符串不变。
- [ ] **A5** `AgentLoopConfig`/`AgentTool`/`AgentToolResult`/`AgentContext`/`AgentState`/`ThinkingLevel`/`AgentMessage`/`SessionTreeEntry`/`SessionStorage`/`ExecutionEnv` 类型字段不变。
- [ ] **A6** 仅依赖 `@openclaw/llm-core` + `typebox` + Node 内置；不引入 OpenClaw `src/**` 依赖。
- [ ] **A7** `node.ts` 仍导出 `NodeExecutionEnv` + `index.ts` 全部。

## B. 循环行为（runLoop）

- [ ] **B1** 事件序：`agent_start` → 每 prompt 的 `message_start`/`message_end` → 每 turn(`turn_start` → assistant `message_start`/`message_update*`/`message_end` → 每工具 `tool_execution_start`/`update*`/`end` + toolResult `message_start`/`message_end` → `turn_end`) → `agent_end`。
- [ ] **B2** `runAgentLoopContinue` 在 messages 空或末条为 assistant 时抛错。
- [ ] **B3** assistant `stopReason ∈ {error, aborted}` 时立即 `turn_end([])` + `agent_end` 并返回。
- [ ] **B4** 开局先 `getSteeringMessages()`；每 turn 后 `prepareNextTurn` → `shouldStopAfterTurn` → `getSteeringMessages`；内层耗尽后 `getFollowUpMessages` 非空则续跑外层。
- [ ] **B5** `prepareNextTurn` 返回的 context/model/thinkingLevel 被应用到下一 turn，且 reasoning 经 `resolveAgentReasoningOption` 重算。
- [ ] **B6** steering/follow-up 消息注入时 emit `message_start`/`message_end` 并入 context 与 newMessages。
- [ ] **B7** `shouldStopAfterTurn` 返回 true → emit `agent_end` 并退出，不再发起 LLM 请求。
- [ ] **B8** `agentLoop`/`agentLoopContinue` 返回 `EventStream`，内部异常经 `pushLoopFailure` 转成完整 error 事件序，**不抛出**。
- [ ] **B9** `transformContext` 在 `convertToLlm` 之前应用；`getApiKey` 解析的 key 覆盖 `config.apiKey`。
- [ ] **B10** 流式增量：带 `partial` 用 partial；`text_delta` 无 partial 时追加到对应 contentIndex 的 text 块。

## C. 工具执行

- [ ] **C1** 默认 `parallel`；任一待执行工具 `executionMode==="sequential"` 或 config `toolExecution==="sequential"` 时整批串行。
- [ ] **C2** 并行模式：preflight（resolve + beforeToolCall）按源序串行；execute 并发；`tool_execution_end` 按完成序；toolResult message 按 assistant 源序。
- [ ] **C3** 流水线：`prepareArguments`(可选) → `validateToolArguments`(TypeBox) → `beforeToolCall`(可 block) → `execute`(signal+onUpdate) → `afterToolCall`(可改写)。
- [ ] **C4** tool 未在 context.tools 找到时调 `resolveDeferredTool`；水合工具名须匹配请求名（否则抛）；水合后加入 context.tools。
- [ ] **C5** `beforeToolCall` 返回 `{block:true}` → 错误 toolResult（reason 文本）；`signal.aborted` → 「Operation aborted」错误结果。
- [ ] **C6** `afterToolCall` 字段级覆盖（content/details/isError/terminate），未提供字段保留原值，无深合并。
- [ ] **C7** 整批工具结果全 `terminate:true` 时循环停止（`shouldTerminateToolBatch`）。
- [ ] **C8** `onUpdate` 推 `tool_execution_update`；execute 异常 → 错误 toolResult（不抛出循环外）。
- [ ] **C9** toolResult message 结构：`{role:"toolResult", toolCallId, toolName, content, details, isError, timestamp}`。

## D. 会话树与上下文构建

- [ ] **D1** `SessionTreeEntry` 全 10 个变体可持久化与读取；条目 append-only，含 `{id, parentId, timestamp}`。
- [ ] **D2** `buildSessionContext`：无压缩 → 全路径回放；有压缩 → compactionSummary 消息 + firstKeptEntryId 起的保留尾部 + 压缩条目后新增条目。
- [ ] **D3** `buildSessionContext` 正确解析最后的 thinkingLevel/model（model_change 或 assistant 消息）。
- [ ] **D4** `convertToLlm` 正确转换 bash/custom/branchSummary/compactionSummary（前后缀包裹），过滤 `excludeFromContext` 与未知 role。
- [ ] **D5** `moveTo(entryId, summary?)` 切换 leaf 并可追加 branch_summary 条目（parentId=entryId）。
- [ ] **D6** `appendParentId` 优先 `getAppendParentId()`（旁路游标），否则 `getLeafId()`。
- [ ] **D7** `JsonlSessionStorage` 每条目一行 JSON；`getPathToRoot` 返回祖先序（根→叶）。
- [ ] **D8** `uuidv7` 时间单调递增且唯一；timestamp 解析失败回退 0 不抛。

## E. 压缩与分支摘要

- [ ] **E1** `shouldCompact = contextTokens > contextWindow - reserveTokens`（enabled 时）；`DEFAULT_COMPACTION_SETTINGS = {true, 16384, 20000}`。
- [ ] **E2** `estimateContextTokens` 优先用最后成功 assistant 的真实 usage + 其后消息字符估算；无 usage 则全字符估算（char/4，图片块 4800）。
- [ ] **E3** `findValidCutPoints` 排除 toolResult 作切点；`findCutPoint` 从尾累加到 `keepRecentTokens` 后吸附到合法切点，并回退避免切在状态标记中。
- [ ] **E4** split turn 检测：切点落在非 user 消息处且能找到 turnStart → 单独摘要 turn 前缀（0.5×reserve maxTokens）。
- [ ] **E5** 增量压缩：存在前次 compaction 时从其 firstKeptEntryId 起算，previousSummary 传入用 UPDATE 模板。
- [ ] **E6** 文件操作（read/written/edited）跨压缩保留并追加到摘要末尾；前次 details 并入（非 fromHook）。
- [ ] **E7** 摘要用固定结构模板（Goal/Constraints/Progress/Key Decisions/Next Steps/Critical Context）；maxTokens = min(0.8×reserve, model.maxTokens)。
- [ ] **E8** 压缩/摘要返回 `Result`；aborted → `CompactionError("aborted")`，失败 → `summarization_failed`，缺 id → `invalid_session`。
- [ ] **E9** `prepareCompaction` 在空路径或末条为 compaction 时返回 `ok(undefined)`。
- [ ] **E10** `generateBranchSummary` 收集公共祖先到目标的放弃路径条目并摘要；aborted → cancelled。

## F. Harness（CoreAgentHarness）

- [ ] **F1** phase 守卫：非 idle 时 prompt/skill/promptFromTemplate/compact/navigateTree 抛 `AgentHarnessError("busy")`。
- [ ] **F2** `createTurnState` 每 turn 从 `session.buildContext()` 重建上下文 + 解析系统提示（字符串或回调）+ active 工具子集。
- [ ] **F3** `prepareNextTurn` flush pendingSessionWrites 后重建 turnState（保证多 turn 从会话树取最新上下文）。
- [ ] **F4** `handleAgentEvent`：message_end→appendMessage；turn_end→flush+save_point；agent_end→phase=idle+settled。
- [ ] **F5** `createStreamFn` 叠加 getApiKeyAndHeaders + before_provider_request/payload 钩子 + after_provider_response。
- [ ] **F6** 17 类钩子（`AgentHarnessEventResultMap`）按 type 调用，返回最后一个非 undefined 结果；广播钩子（subscribe）异常归一成 hook 错误。
- [ ] **F7** `compact`：getBranch→prepareCompaction→session_before_compact 钩子(可 cancel/提供)→compact→appendCompaction→session_compact。
- [ ] **F8** `navigateTree`：目标为 user/custom_message 时 newLeafId=parentId 且回填 editorText（编辑重跑）；否则 newLeafId=targetId；可选生成 branch_summary。
- [ ] **F9** 三队列（steer/followUp/nextTurn）+ QueueMode（all/one-at-a-time）drain 语义正确；steer/followUp 非 idle 才允许，nextTurn 任意时。
- [ ] **F10** `setModel/setThinkingLevel` idle 时落盘、运行中入 pendingSessionWrites；emit model_select/thinking_level_select。
- [ ] **F11** `abort` 清两队列 + 中止 + waitForIdle + emit abort，返回 cleared 队列。
- [ ] **F12** 错误归一：SessionError/CompactionError/BranchSummaryError → 对应 code 的 AgentHarnessError。

## G. 错误与中止

- [ ] **G1** `streamFn` 契约：失败编码进流（stopReason error/aborted + errorMessage），不抛。
- [ ] **G2** 循环多检查点检测 `signal.aborted`，中止时持久化 aborted 助手消息并补全事件序（turn_start/message/turn_end/agent_end）。
- [ ] **G3** 钩子（convertToLlm/transformContext/getSteering/getFollowUp）不抛、返回安全兜底。
- [ ] **G4** Agent 一次只允许一个 activeRun；prompt/continue 在运行中抛错。

## H. 非功能

- [ ] **H1** 无内部循环依赖（madge/import-cycles 绿）。
- [ ] **H2** 不可变更新：pendingToolCalls 用新 Set 拷贝；state.tools/messages 赋值拷贝顶层数组。
- [ ] **H3** TypeScript strict，无 `any`（用 unknown + 窄化）；TypeBox schema 用于工具参数。
- [ ] **H4** 全部现有单测通过（`agent-loop.test.ts`/`reasoning.test.ts`/`messages.test.ts`/`prompt-templates.test.ts` 等）。

---

## 验收测试建议（重写后必跑）

1. **循环金路径**：单 turn 无工具 → 事件序与最终消息正确。
2. **工具循环**：含 1 工具 → before/execute/after/回灌/续 turn → 终止。
3. **并行 vs 串行**：多工具，验证 end 完成序 + result 源序；含 sequential 工具时整批串行。
4. **steering/follow-up**：运行中 steer → 下 turn 注入；停下后 followUp → 续跑。
5. **abort**：turn 中途 abort → aborted 消息持久化 + 完整事件序。
6. **会话往返**：append 多消息 → buildContext 还原 → 压缩 → buildContext 验证摘要替换。
7. **压缩**：构造超窗会话 → prepareCompaction 切点正确（不切 toolResult）→ compact 摘要含文件列表 → 增量压缩保留 previousSummary。
8. **split turn 压缩**：切点落 turn 中间 → 双摘要拼接。
9. **分支导航**：navigateTree 到历史 user 消息 → editorText 回填 + branch_summary 生成 + leaf 移动。
10. **注入缺失**：未配置 runtime 且无 streamFn → 明确抛错。
