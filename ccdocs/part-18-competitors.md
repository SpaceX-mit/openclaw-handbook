# 第十八部分：竞品对比

## 18.1 对比维度总表

| 维度 | OpenClaw | LangGraph | CrewAI | AutoGen | OpenHands | Claude Code | OpenCode |
|---|---|---|---|---|---|---|---|
| **定位** | 多渠道个人助理运行时（产品） | Agent 编排框架（库） | 多 Agent 协作框架（库） | 多 Agent 对话框架（库/研究） | 自主软件工程 Agent（产品） | 终端编码 Agent（产品） | 开源编码 Agent（产品） |
| **语言** | TypeScript | Python（+JS） | Python | Python | Python | TS | Go/TS |
| **架构核心** | ReAct 单循环 + 控制平面 + 插件 | 有状态图（StateGraph） | Crew/Agent/Task/Process | ConversableAgent 对话 | Agent + runtime + 浏览器/CLI | ReAct harness | ReAct harness |
| **扩展性** | manifest 插件 + MCP 双向（极强） | 节点/边自定义（强） | 自定义 Agent/Tool（中） | 自定义 Agent（中） | microagents + MCP（中） | MCP + skills（中） | provider/MCP（中） |
| **性能** | 常驻服务，懒加载冷启动优化 | 取决于宿主 | 取决于宿主 | 取决于宿主 | 重（含浏览器/沙箱） | 轻（CLI） | 轻 |
| **Agent 能力** | 真实执行（70 工具）+ 渠道 + 设备 | 取决于实现 | 角色分工协作 | 多 Agent 对话/群聊 | 端到端软件工程 | 编码 + 终端 | 编码 |
| **Memory** | 三级 + 混合检索 + dreaming（强） | checkpointer + store（中） | 短/长期 + RAG（中） | 可配 memory（中） | 会话 + condenser（中） | 会话 + 压缩（中） | 会话（中） |
| **Tool** | 70 内置 + MCP + 技能 + 策略沙箱（强） | 工具节点（中） | 工具（中） | 工具/代码执行（中） | 工具 + 浏览器（强） | 工具 + MCP（强） | 工具 + MCP（中） |
| **Multi-Agent** | 单层委派 + A2A（克制） | 图编排（强） | Crew 协作（强） | 群聊/嵌套（强） | 单 + 委派（中） | 子代理（中） | 弱 |
| **工程复杂度** | 极高（59 万行，全栈+多渠道+App） | 中 | 中低 | 中 | 高 | 中 | 中 |
| **多渠道 IM** | ✅ 23+（独家强项） | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

## 18.2 架构对比

- **LangGraph**：以**显式状态图**为核心，节点 = 步骤，边 = 转移，自带 checkpointer/store。OpenClaw 故意**不做图编排**（`VISION.md:124`），用单 ReAct 循环 + 工程化重试，复杂度更低但表达力也更受限。
- **CrewAI**：一等对象是 Crew/Agent/Task/Process（sequential/hierarchical）。OpenClaw **没有 Task/Crew 一级对象**，任务内化为会话/子代理。CrewAI 擅长「角色分工」，OpenClaw 擅长「单 agent 真实执行 + 多渠道」。
- **AutoGen**：以**多 Agent 对话**（ConversableAgent、GroupChat）为范式。OpenClaw 的 A2A 是有界 ping-pong，远不及 AutoGen 的群聊编排，但 OpenClaw 不追求这个。
- **OpenHands**：与 OpenClaw 都是「产品级执行 Agent」，但 OpenHands 聚焦软件工程（含浏览器、代码沙箱、端到端 PR），OpenClaw 聚焦「通用个人助理 + 多渠道」。两者的 harness/压缩/工具思想同源。
- **Claude Code / OpenCode**：与 OpenClaw 共享 coding-agent harness 范式（ReAct + 工具 + 压缩 + steering）。OpenClaw 实际**复用了这套范式并泛化到个人助理**（工具集叫 `createOpenClawCodingTools`），并能**把 Codex 当后端 harness 驱动**。区别：Claude Code/OpenCode 是终端编码工具，OpenClaw 是多渠道常驻助理。

## 18.3 各维度细分对比

### 扩展能力
OpenClaw 的 manifest-first + 懒激活 + MCP 双向 + 单槽记忆/可插拔上下文引擎/可插拔 harness，是这批里**最系统化的扩展架构**。LangGraph 的图节点扩展灵活但偏「写代码」；CrewAI/AutoGen 偏「定义 Agent」。

### Memory 能力
OpenClaw 的三级记忆 + FTS5/向量混合检索 + dreaming promotion + 时间衰减 + MMR，**深度领先**多数框架（LangGraph 的 store/checkpointer 是基础设施级，需自己搭检索）。

### Multi-Agent 能力
**OpenClaw 最弱的一维**（刻意）。LangGraph（图）、CrewAI（crew）、AutoGen（群聊）都远强于 OpenClaw 的单层委派。若需复杂多 Agent 编排，OpenClaw 不是选择。

### 工程复杂度
OpenClaw **最高**：59 万行、23+ 渠道、4 个平台原生 App、Gateway 协议、插件生态。这是产品级广度的代价，也是其他纯框架不具备的。

## 18.4 选型建议

| 需求 | 推荐 |
|---|---|
| 多渠道常驻个人助理 + 真实执行 | **OpenClaw**（独家） |
| 复杂多 Agent 工作流编排 | LangGraph / CrewAI / AutoGen |
| 端到端自主软件工程 | OpenHands |
| 终端内编码助理 | Claude Code / OpenCode |
| 嵌入自己应用的 Agent 库 | LangGraph |
| 角色分工的团队式 Agent | CrewAI |

## 18.5 OpenClaw 的差异化护城河（综合对比）

唯一同时具备「多渠道广度（23+）× 真实执行深度（70 工具 + 设备）× 生产级可靠性（重试/压缩/失败转移）× 系统化扩展（插件 + MCP）」的开源项目。其他项目各强一隅，但没有一个覆盖这个组合 —— 因为这个组合需要**产品级长期工程投入**，而非框架级抽象设计。
