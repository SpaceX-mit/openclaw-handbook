# 10 · 端到端流程与时序

## 10.1 L2 路径：AgentSession 驱动一次对话（OpenClaw 实际路径）

```mermaid
sequenceDiagram
  autonumber
  participant Up as embedded-agent-runner
  participant AS as AgentSession (OpenClaw)
  participant AG as Agent (agent-core L2)
  participant LP as runLoop (L1)
  participant SF as streamFn (注入)
  participant T as AgentTool.execute

  Up->>AS: prompt(text, options)
  AS->>AG: agent.subscribe(handleAgentEvent)  %% 持久化/转发
  AS->>AG: agent.prompt(messages)
  AG->>LP: runAgentLoop(prompts, context, loopConfig, emit, signal, streamFn)
  LP-->>AG: emit agent_start / turn_start / message_*(prompt)
  AG-->>AS: handleAgentEvent (落盘 user 消息)
  loop 每 turn
    LP->>LP: transformContext (钩子) → convertToLlm
    LP->>SF: streamFn(model, {systemPrompt, messages, tools}, opts)
    SF-->>LP: start / text_delta / thinking_delta / toolcall_* / done
    LP-->>AG: emit message_start/update/end (assistant)
    AG-->>AS: 落盘 assistant 消息
    alt 含 toolCall
      LP->>LP: beforeToolCall (钩子: 权限/策略)
      LP->>T: execute(id, args, signal, onUpdate)
      T-->>LP: tool_execution_update (流式)
      T-->>LP: AgentToolResult
      LP->>LP: afterToolCall (钩子: 改写)
      LP-->>AG: emit tool_execution_* + toolResult message_*
      AG-->>AS: 落盘 toolResult
    end
    LP-->>AG: emit turn_end
    LP->>LP: prepareNextTurn / shouldStopAfterTurn / getSteering / getFollowUp
  end
  LP-->>AG: emit agent_end
  AG-->>AS: handleAgentEvent(agent_end) → 结算
  AS-->>Up: 返回最终 assistant 消息
```

## 10.2 L3 路径：CoreAgentHarness.prompt

```mermaid
sequenceDiagram
  autonumber
  participant C as 调用方
  participant H as CoreAgentHarness
  participant S as Session (会话树)
  participant LP as runAgentLoop
  participant SF as createStreamFn

  C->>H: prompt(text)
  Note over H: phase idle→turn (否则 busy)
  H->>S: buildContext() → createTurnState (系统提示+工具快照)
  H->>H: emitHook(before_agent_start) 可改 messages/系统提示
  H->>LP: runAgentLoop(messages, context, createLoopConfig, handleAgentEvent, signal, createStreamFn)
  loop 每 turn
    LP->>H: transformContext → emitHook(context)
    LP->>SF: streamFn
    SF->>H: getApiKeyAndHeaders + emitHook(before_provider_request/payload)
    SF-->>LP: 事件流
    LP->>H: beforeToolCall → emitHook(tool_call) / afterToolCall → emitHook(tool_result)
    LP->>H: handleAgentEvent(message_end) → session.appendMessage
    LP->>H: handleAgentEvent(turn_end) → flushPendingSessionWrites + emitOwn(save_point)
    LP->>H: prepareNextTurn → flushPending + createTurnState(重建上下文)
  end
  LP->>H: handleAgentEvent(agent_end) → phase=idle + emitOwn(settled)
  H-->>C: 返回最终 assistant 消息
```

## 10.3 压缩触发流程（在 harness 中）

```mermaid
sequenceDiagram
  participant Up as 上层(或自动判定)
  participant H as CoreAgentHarness
  participant S as Session
  participant CP as compaction
  participant LLM as completeSimple(注入)

  Up->>H: compact(customInstructions?)
  Note over H: phase idle→compaction
  H->>S: getBranch() → branchEntries
  H->>CP: prepareCompaction(branchEntries, settings)
  CP-->>H: CompactionPreparation (firstKeptEntryId, messagesToSummarize, ...)
  H->>H: emitHook(session_before_compact) 可 cancel/提供摘要
  alt 钩子未提供摘要
    H->>CP: compact(prep, model, apiKey, ...)
    CP->>LLM: completeSummarization (SUMMARIZATION_PROMPT)
    LLM-->>CP: 摘要文本
    CP-->>H: CompactionResult(summary, firstKeptEntryId, tokensBefore, details)
  end
  H->>S: appendCompaction(summary, firstKeptEntryId, tokensBefore, details, fromHook)
  H->>H: emitOwn(session_compact)
  Note over H: phase compaction→idle
  Note over S: 下次 buildContext: 摘要替换旧史 + 保留尾部
```

## 10.4 状态转换汇总

| 实体 | 状态 | 触发转换 |
|---|---|---|
| Agent run | idle → streaming → idle | prompt()/continue() → agent_end + 监听器结算 |
| Agent 内 | streaming ↔ toolExecuting | 助手消息含/不含 toolCall |
| Harness phase | idle/turn/compaction/branch_summary | prompt/compact/navigateTree |
| 会话树 leaf | 移动 | appendMessage（指向新条目）/ moveTo（分支） |
| 上下文 | 增长 → 压缩 | shouldCompact 触发 → appendCompaction |

## 10.5 关键不变量（流程级）

1. **事件成对**：每个 message_start 必有 message_end；每个 tool_execution_start 必有 tool_execution_end。
2. **turn 包裹**：turn_start...turn_end 包裹一次 assistant 响应 + 其工具批。
3. **agent_end 唯一终态**：每次 run 恰好一个 agent_end，携带本次 newMessages。
4. **toolResult 紧跟其 toolCall**：工具结果消息在对应 assistant 消息之后，且压缩切点不切散二者。
5. **abort 持久化**：中止时持久化 aborted 助手消息，事件序仍完整（补 turn_start/message/turn_end/agent_end）。
6. **会话写在 turn 边界 flush**：运行中的 model_change/thinking_change 等累积在 pendingSessionWrites，turn_end 时落盘（保证会话树时序正确）。
