# OpenClaw 深度分析文档索引

## 文档概览

本目录包含对 OpenClaw 项目的完整深度分析文档，涵盖源码级、架构级、产品级和生态级的全面分析。

## 文档列表

### 核心分析文档

| 文档 | 说明 | 大小 |
|------|------|------|
| [01-project-positioning.md](01-project-positioning.md) | 项目定位、架构、核心对象、运行机制等21个维度的完整分析 | ~80KB |
| [02-core-modules.md](02-core-modules.md) | 核心模块详细分析（Agent Core、Harness、Tool、Session等） | ~27KB |
| [03-extension-guide.md](03-extension-guide.md) | 扩展点指南（Plugin、Tool、Skill、Channel、Provider、MCP） | ~17KB |
| [04-api-reference.md](04-api-reference.md) | API参考手册 | ~14KB |

## 文档内容概览

### 第一部分：21维度深度分析 (01-project-positioning.md)

1. **项目定位分析** - 解决什么问题、为什么诞生、对标产品、核心竞争力
2. **整体架构分析** - 完整架构图（Mermaid）、层间调用关系
3. **代码目录逆向分析** - src/、packages/、extensions/ 结构详解
4. **核心对象模型分析** - Agent、Session、Tool等核心对象
5. **Agent运行机制分析** - 启动、规划、推理、Tool调用、反思
6. **Runtime分析** - 上下文管理、Token管理、模型调用、事件循环
7. **Harness分析** - Harness架构、生命周期、资源管理
8. **Loop Engineering分析** - 循环结构、终止条件、错误恢复
9. **Memory系统分析** - 短期记忆、工作记忆、长期记忆、RAG
10. **Tool/MCP分析** - Tool调用、注册、发现、MCP支持
11. **多Agent分析** - Subagent架构、ACP协议、任务分发
12. **数据流分析** - 完整数据流、Prompt变化、状态变化
13. **扩展机制分析** - 新增Agent、Tool、Skill、Memory等
14. **技术选型分析** - TypeScript、SQLite、LLM SDK选择
15. **源码关键路径分析** - 入口、核心函数、调用链
16. **复刻指南** - 6阶段开发计划Roadmap
17. **Agent OS映射分析** - OS模块映射、缺失能力
18. **竞品对比** - vs LangGraph、CrewAI、AutoGen等
19. **优缺点分析** - TOP20优点、TOP20缺点
20. **最终结论** - 创新点、最难复刻、评分（架构/工程/Agent能力/生态/潜力）
21. **Agentic OS映射** - 层次定位、上下层能力、缺失接口

### 第二部分：核心模块详解 (02-core-modules.md)

- Agent Core模块详解
- Harness模块详解
- Tool系统模块详解
- Session管理模块详解
- ACP协议模块详解
- Channel系统模块详解
- Plugin SDK模块详解
- LLM Runtime模块详解

### 第三部分：扩展点指南 (03-extension-guide.md)

- 创建Plugin完整指南
- 创建Tool（Descriptor、Executor、可用性）
- 创建Skill（SKILL.md格式、注册）
- 创建Channel（Transport、Session）
- 创建Model Provider
- 创建MCP Server
- 最佳实践

### 第四部分：API参考 (04-api-reference.md)

- Agent Core API
- Tool API
- Session API
- Plugin SDK API
- Channel API
- LLM Runtime API
- 常见错误码

## 快速导航

### 按主题查找

| 主题 | 文档位置 |
|------|----------|
| 项目概述 | [01-project-positioning.md - 第一部分](01-project-positioning.md#第一部分项目定位分析) |
| 架构图 | [01-project-positioning.md - 第二部分](01-project-positioning.md#第二部分整体架构分析) |
| Agent循环 | [01-project-positioning.md - 第五部分](01-project-positioning.md#第五部分agent运行机制分析) |
| Harness | [01-project-positioning.md - 第七部分](01-project-positioning.md#第七部分harness分析) |
| Tool系统 | [01-project-positioning.md - 第十部分](01-project-positioning.md#第十部分tool-mcp分析) |
| 扩展开发 | [03-extension-guide.md](03-extension-guide.md) |
| API文档 | [04-api-reference.md](04-api-reference.md) |

### 关键文件索引

| 模块 | 核心文件 |
|------|----------|
| Agent Core | `packages/agent-core/src/agent.ts`, `packages/agent-core/src/agent-loop.ts` |
| Harness | `packages/agent-core/src/harness/agent-harness.ts` |
| Tool | `src/tools/planner.ts`, `src/tools/types.ts` |
| Session | `packages/agent-core/src/harness/session/session.ts` |
| ACP | `src/acp/translator.ts`, `src/agents/subagent-registry.ts` |
| Channel | `src/channels/registry.ts` |
| Plugin SDK | `packages/plugin-sdk/src/plugin-runtime.ts` |

## 关键架构图

### 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenClaw Architecture                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐       │
│  │     CLI      │     │     TUI      │     │   Channel    │       │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘       │
│         │                    │                    │                 │
│         └────────────────────┼────────────────────┘                 │
│                              ▼                                        │
│                    ┌──────────────────┐                             │
│                    │     Gateway      │                             │
│                    └────────┬─────────┘                             │
│                             │                                        │
│         ┌───────────────────┼───────────────────┐                   │
│         ▼                   ▼                   ▼                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │
│  │   Session   │    │   Agent     │    │   Channel   │            │
│  │   Manager   │    │   Harness  │    │   Manager   │            │
│  └──────┬──────┘    └──────┬──────┘    └─────────────┘            │
│         │                  │                                        │
│         │         ┌────────┴────────┐                              │
│         │         ▼                 ▼                              │
│         │    ┌─────────┐      ┌─────────┐                         │
│         │    │ Agent   │      │  Tool   │                         │
│         │    │  Loop   │      │ Planner │                         │
│         │    └────┬────┘      └────┬────┘                         │
│         │         │                 │                               │
│         │         └────────┬────────┘                               │
│         │                  ▼                                        │
│         │           ┌────────────┐                                 │
│         │           │   Model    │                                 │
│         │           │  Runtime   │                                 │
│         │           └─────┬──────┘                                 │
│         │                 │                                         │
│         │    ┌────────────┼────────────┐                            │
│         │    ▼            ▼            ▼                            │
│         │ ┌──────┐  ┌──────┐  ┌──────┐                           │
│         │ │OpenAI│  │Anthro│  │Google│                           │
│         │ └──────┘  └──────┘  └──────┘                           │
│         │                                                       │
│         ▼                                                       │
│   ┌─────────────┐                                              │
│   │   SQLite    │                                              │
│   └─────────────┘                                              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## 评分总结

| 维度 | 评分 | 说明 |
|------|------|------|
| **架构评分** | 92/100 | 模块化优秀，Harness模式创新 |
| **工程评分** | 88/100 | 代码质量高，测试完善 |
| **Agent能力** | 90/100 | Tool系统完整，Loop设计优秀 |
| **生态评分** | 95/100 | Plugin生态丰富，渠道支持最多 |
| **未来潜力** | 93/100 | 本地优先+多Agent是趋势 |

**综合评分**: 91.6/100

---

*文档生成时间: 2026-06-23*
*分析版本: OpenClaw main branch*
*项目仓库: https://github.com/openclaw/openclaw*
