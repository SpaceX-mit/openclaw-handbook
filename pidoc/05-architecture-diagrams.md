# OpenClaw 架构图表集

> 本文档包含 OpenClaw 项目的关键架构图和流程图，便于快速理解系统设计。

## 目录

1. [整体架构图](#1-整体架构图)
2. [Agent Loop流程图](#2-agent-loop流程图)
3. [Tool调用时序图](#3-tool调用时序图)
4. [Session生命周期图](#4-session生命周期图)
5. [Harness架构图](#5-harness架构图)
6. [ACP协议时序图](#6-acp协议时序图)
7. [多渠道架构图](#7-多渠道架构图)
8. [Memory分层图](#8-memory分层图)
9. [Plugin架构图](#9-plugin架构图)

---

## 1. 整体架构图

```mermaid
flowchart TB
    subgraph UI["用户界面层"]
        CLI[CLI Interface]
        TUI[Terminal UI]
        WEB[Web Dashboard]
        APP[Companion Apps]
    end

    subgraph Gateway["Gateway层"]
        GW[Gateway Server]
        CHANNEL_MGR[Channel Manager]
        SESSION_MGR[Session Manager]
        PLUGIN_MGR[Plugin Manager]
    end

    subgraph AgentLayer["Agent层"]
        HARNESS[Agent Harness]
        LOOP[Agent Loop]
        PLANNER[Tool Planner]
        HOOK[Hook System]
    end

    subgraph RuntimeLayer["运行时层"]
        CORE[Agent Core]
        EXEC[Tool Executor]
        SANDBOX[Sandbox]
        COMPACT[Compaction]
    end

    subgraph Integration["集成层"]
        MCP[MCP Runtime]
        CHANNELS[Channel Plugins]
        PROVIDERS[Model Providers]
    end

    subgraph DataLayer["数据层"]
        SESSION_DB[(Session DB)]
        MEMORY[(Memory)]
        STATE[(State)]
    end

    UI --> Gateway
    Gateway --> AgentLayer
    AgentLayer --> RuntimeLayer
    RuntimeLayer --> Integration
    RuntimeLayer --> DataLayer
```

## 2. Agent Loop流程图

```mermaid
flowchart TD
    START([开始]) --> INIT
    
    subgraph INIT["初始化"]
        INIT1[创建上下文] --> INIT2[构建Prompt]
        INIT2 --> INIT3[emit agent_start]
    end
    
    INIT3 --> TURN
    
    subgraph TURN["Turn循环"]
        TURN_START[emit turn_start] --> PENDING{有Pending?}
        PENDING -->|Yes| INJECT[注入消息]
        INJECT --> STREAM
        PENDING -->|No| STREAM
        STREAM[流式LLM响应] --> PARSE[解析响应]
        PARSE --> CHECK{stopReason?}
        
        CHECK -->|error| ERROR[emit error]
        CHECK -->|stop| CHECK_TOOLS{有Tool Calls?}
        CHECK -->|toolUse| CHECK_TOOLS
        
        CHECK_TOOLS -->|Yes| EXEC[执行工具]
        EXEC --> RESULTS[收集结果]
        RESULTS --> TURN_END[emit turn_end]
        TURN_END --> PREPARE[prepareNextTurn]
        PREPARE --> STEERING{有Steering?}
        STEERING -->|Yes| PENDING
        STEERING -->|No| STOP_CHECK
        
        CHECK_TOOLS -->|No| TURN_END2[emit turn_end]
        TURN_END2 --> STOP_CHECK
    end
    
    STOP_CHECK{应停止?} -->|Yes| END
    STOP_CHECK -->|No| FOLLOWUP{有FollowUp?}
    FOLLOWUP -->|Yes| ROUTE[路由消息]
    ROUTE --> PENDING
    FOLLOWUP -->|No| FINAL_END[emit agent_end]
    
    ERROR --> END
    FINAL_END --> END([结束])
    
    style START fill:#90EE90
    style END fill:#FFB6C1
    style TURN fill:#E6F3FF
    style INIT fill:#FFF8DC
```

## 3. Tool调用时序图

```mermaid
sequenceDiagram
    participant LLM as LLM
    participant Loop as Agent Loop
    participant Planner as Tool Planner
    participant Executor as Tool Executor
    participant Sandbox as Sandbox
    participant Session as Session

    LLM-->>Loop: ToolCall响应
    Loop->>Planner: 规划可用工具
    Planner-->>Loop: ToolPlan (visible/hidden)

    Loop->>Executor: 执行Tool (name, args)
    
    alt 需要沙箱
        Executor->>Sandbox: 沙箱执行
        Sandbox-->>Executor: 执行结果
    else 直接执行
        Executor-->>Executor: 直接执行
    end
    
    Executor-->>Loop: ToolResult
    
    alt 执行失败
        Loop->>Session: 记录错误
        Session-->>Loop: 确认
    else 执行成功
        Loop->>Session: 更新状态
        Session-->>Loop: 确认
    end
    
    Loop->>Loop: 继续循环或结束
```

## 4. Session生命周期图

```mermaid
stateDiagram-v2
    [*] --> Created: new Session()

    Created --> Active: start()

    Active --> Active: addMessage()
    Active --> Active: update()

    Active --> Checkpointed: checkpoint()
    Checkpointed --> Restored: restore()

    Active --> Compacting: needsCompaction()
    Compacting --> Active: compaction complete

    Active --> Paused: pause()
    Paused --> Active: resume()

    Paused --> Archived: timeout
    Archived --> Active: restore()

    Active --> Completed: close()
    Completed --> [*]

    Active --> Error: error
    Error --> Active: retry
    Error --> [*]: fatal

    note right of Active
        状态: messages, tools, prompt
    end note

    note right of Checkpointed
        快照: 可用于恢复
    end note
```

## 5. Harness架构图

```mermaid
flowchart TB
    subgraph Harness["AgentHarness"]
        direction TB
        
        subgraph Resources["资源"]
            SYSTEM[System Prompt]
            SKILLS[Skills]
            TEMPLATES[Prompt Templates]
        end
        
        subgraph SessionMgmt["会话管理"]
            SESSION[Session]
            STORE[Session Store]
            CHECKPOINT[Checkpoint]
        end
        
        subgraph ToolPlanning["工具规划"]
            DESCRIPTORS[Tool Descriptors]
            PLANNER[Tool Planner]
            AVAIL[Availability Checker]
        end
        
        subgraph Execution["执行"]
            EXEC[Tool Executor]
            SANDBOX[Sandbox]
            VALIDATOR[Validator]
        end
        
        subgraph Memory["内存管理"]
            COMPACT[Compaction Engine]
            PRUNER[Context Pruner]
        end
    end

    Resources --> SessionMgmt
    DESCRIPTORS --> PLANNER
    PLANNER --> AVAIL
    AVAIL --> Execution
    SESSION --> Memory
    COMPACT --> PRUNER

    style Harness fill:#F0F8FF,stroke:#4169E1
    style SessionMgmt fill:#E8F5E9,stroke:#4CAF50
    style ToolPlanning fill:#FFF3E0,stroke:#FF9800
    style Memory fill:#FCE4EC,stroke:#E91E63
```

## 6. ACP协议时序图

```mermaid
sequenceDiagram
    participant Manager as Manager Agent
    participant Bus as ACP Bus
    participant Worker1 as Worker 1
    participant Worker2 as Worker 2

    Note over Manager: Task Creation
    Manager->>Bus: Announce Task (ACPAnnounce)
    Bus->>Bus: Route to workers

    par 并行分发
        Bus->>Worker1: Forward Task
        Bus->>Worker2: Forward Task
    end

    Note over Worker1: Processing
    Worker1->>Bus: Progress Update (50%)
    Bus->>Manager: Progress

    Note over Worker2: Processing
    Worker2->>Bus: Progress Update (75%)
    Bus->>Manager: Progress

    Worker1->>Bus: Complete Result
    Bus->>Manager: Partial Result

    Worker2->>Bus: Complete Result
    Bus->>Manager: Final Result

    Note over Manager: Aggregation Complete
```

## 7. 多渠道架构图

```mermaid
flowchart TB
    subgraph Sources["消息来源"]
        TG[Telegram]
        DC[Discord]
        WA[WhatsApp]
        SL[Slack]
        EM[Email]
    end

    subgraph Gateway["Gateway"]
        ROUTE[Message Router]
        SESSION[Session Resolver]
        CHANNEL_MGR[Channel Manager]
    end

    subgraph Agent["Agent System"]
        HARNESS[Agent Harness]
        LOOP[Agent Loop]
        TOOLS[Tool Executor]
    end

    subgraph Destinations["消息目的地"]
        TG_OUT[Telegram]
        DC_OUT[Discord]
        WA_OUT[WhatsApp]
        SL_OUT[Slack]
    end

    Sources --> Gateway
    Gateway --> Agent
    Agent --> Gateway
    Gateway --> Destinations

    style Gateway fill:#E3F2FD,stroke:#1976D2
    style Agent fill:#E8F5E9,stroke:#388E3C
```

## 8. Memory分层图

```mermaid
flowchart TB
    subgraph Input["输入"]
        USER[用户消息]
        TOOL[工具结果]
        SYSTEM[系统信息]
    end

    subgraph Layer1["L1: Short-term"]
        SHORT[Messages]
    end

    subgraph Layer2["L2: Working"]
        WORK[System Prompt]
        SUMMARY[Compressed Summary]
    end

    subgraph Layer3["L3: Long-term"]
        LONG[Persistent Memory]
        KB[Knowledge Base]
    end

    subgraph Retrieval["检索"]
        QUERY[Query]
        EMBED[Embedding]
        SEARCH[Vector Search]
    end

    Input --> Layer1
    Layer1 --> Layer2
    Layer2 --> Layer3

    QUERY --> EMBED
    EMBED --> SEARCH
    SEARCH --> Layer3
    Layer3 --> CONTEXT[Context]

    style Layer1 fill:#BBDEFB
    style Layer2 fill:#C8E6C9
    style Layer3 fill:#FFE0B2
```

## 9. Plugin架构图

```mermaid
flowchart LR
    subgraph Core["OpenClaw Core"]
        SDK[Plugin SDK]
        REGISTRY[Registry]
    end

    subgraph Plugins["Plugins"]
        TOOL_PLUGIN[Tool Plugin]
        CHANNEL_PLUGIN[Channel Plugin]
        PROVIDER_PLUGIN[Provider Plugin]
        MEMORY_PLUGIN[Memory Plugin]
    end

    subgraph External["外部系统"]
        TOOLS[Tools]
        CHANNELS[Channels]
        MODELS[Models]
        STORAGE[Storage]
    end

    TOOL_PLUGIN --> SDK
    CHANNEL_PLUGIN --> SDK
    PROVIDER_PLUGIN --> SDK
    MEMORY_PLUGIN --> SDK

    SDK --> REGISTRY
    REGISTRY --> TOOLS
    REGISTRY --> CHANNELS
    REGISTRY --> MODELS
    REGISTRY --> STORAGE

    style Core fill:#E1F5FE
    style Plugins fill:#FFF3E0
    style External fill:#F3E5F5
```

---

*图表生成时间: 2026-06-23*
