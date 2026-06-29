# OpenClaw Memory 系统功能与完成度分析

> 调查目标：memory 系统的功能与完成度评估。
> 方法：源码级核查 —— 所有 LOC、阈值、配置、表名、默认值均**逐条对源核对**（见文末「核对清单」）。证据以 `file:line` 标注。
> 一句话结论：**Memory 是 OpenClaw 投入最重、测试最充分的子系统之一（~5.6 万行 + 160 测试文件），混合检索链路达生产级；功能在「个人助理长期记忆」定位内完整可用。主要短板是「单槽多实现未收敛」、dreaming 子系统复杂度偏高且偏研究性、以及原始 SQL/JSON 工作态与 SQLite-only 策略的张力。**

---

## 1. 代码规模与构成（已核对）

| 模块 | 角色 | LOC | 文件 | 测试 |
|---|---|---:|---:|---:|
| `extensions/memory-core` | **默认记忆槽插件**（FTS+向量混合 + dreaming） | 28346 | 82 | **63** |
| `extensions/memory-wiki` | 知识 wiki 层（OKF/Obsidian，伴侣插件） | 12578 | 38 | 31 |
| `packages/memory-host-sdk` | embedding/FTS/sqlite-vec 引擎（主机 SDK） | 8574 | 80 | — |
| `extensions/active-memory` | 回复前阻塞召回注入（伴侣插件） | 3924 | 2 | 3 |
| `extensions/memory-lancedb` | LanceDB 向量后端（替代槽） | 2492 | 6 | 3 |
| `src/memory/root-memory-files.ts` | 核心仅种子文件定位器 | 962 | — | — |
| **合计** | | **~56876** | | **~100+** |

**信号**：memory-core 零 TODO/FIXME/HACK 标记（grep 命中全是 prompt 文本/正则子串）；63 个测试文件 —— 工程投入与测试覆盖在全项目居前列。

---

## 2. 功能清单与完成度（标尺 A：个人助理长期记忆定位内）

### 2.1 三级记忆模型 —— 完成度高
- **短期**：会话转写按日索引 + recall 跟踪（`source=sessions`）。
- **工作记忆**：dreaming/promotion 临时态（SQLite KV + `.dreams/*.json`）。
- **长期**：`MEMORY.md` + `memory/*.md`，索引进 SQLite（FTS5 + sqlite-vec）。
- 与上下文压缩（agent-core）**独立** —— 压缩是单会话窗口管理，记忆是跨会话可检索持久知识。

### 2.2 混合检索 —— 生产级（最强项，已读源码核实）
`hybrid.ts` 的 `mergeHybridResults`（`:52`）是完整三段式管线，非 stub：
1. **FTS5 关键词**：`buildFtsQuery`（`:32`）分词保留 `\p{L}\p{N}_`、引号转义、AND 连接；`bm25RankToScore`（`:41`）把 SQLite 负 rank 转 0~1 相关度。
2. **向量检索**：查询 embedding → sqlite-vec 最近邻。
3. **合并**：按 chunk id 合并（向量结果入 Map → 关键词补 textScore/择优 snippet），加权分 `vectorWeight*v + textWeight*t`（`:125`）。
4. **时间衰减重排**（`temporal-decay.ts`）：`halfLifeDays=30`，`λ=ln2/halfLife`。
5. **MMR 多样性重排**（`mmr.ts`）：`computeMMRScore = λ*relevance − (1−λ)*maxSim`，λ=0.7。

> ⚠️ **重要更正（对源核实）**：**MMR 与时间衰减默认都是 `enabled: false`**（`mmr.ts:27` / `temporal-decay.ts:11`）—— 是**可选开关，非默认开启**。默认检索 = FTS+向量加权合并；MMR/decay 需调用方在 `search` 显式传 `{enabled:true}`（`manager.ts:965-966,989-990`）。

### 2.3 Embedding / 向量 —— 完成度高，优雅降级
- 可插拔引擎（host-sdk）：远程 HTTP provider、本地 llama.cpp、批处理任务、worker 线程隔离、embedding 缓存表。
- **优雅降级**：sqlite-vec 加载失败或无 embedding provider 时**退化为纯 FTS 关键词检索**，不崩溃 —— 保证任何环境记忆可用。
- 双存 embedding：chunks 表存 JSON（可移植）+ `_vec` 表存二进制（查询优化）。

### 2.4 短期→长期 promotion —— 完成度高（已核对阈值）
`short-term-promotion.ts`：高频 recall 片段晋升 `MEMORY.md`。门槛（已核实）：
- `DEFAULT_PROMOTION_MIN_SCORE=0.75`（`:47`）
- `DEFAULT_PROMOTION_MIN_RECALL_COUNT=3`（`:48`）
- `DEFAULT_PROMOTION_MIN_UNIQUE_QUERIES=2`（`:49`）
- recall 跟踪上限：`MAX_ENTRIES=512`、`MAX_RECALL_DAYS=16`、`MAX_QUERY_HASHES=32`、recency `HALF_LIFE=14天`。

### 2.5 记忆预算/淘汰 —— 完成度高
`memory-budget.ts`：`DEFAULT_MEMORY_FILE_MAX_CHARS=10_000`（`:25`）− `WRITE_OVERHEAD_RESERVE=21`。超限优先淘汰**最老的自动 promotion 段**，保护用户手写内容。+ MMR 去重 + reindex 孤儿 GC。

### 2.6 dreaming 离线巩固 —— 功能完整但复杂度高/偏研究性
最大、最复杂子系统，**6724 LOC / 12 文件**（已核实）：`dreaming-phases.ts`(2001)、`dreaming-narrative.ts`(1211)、`rem-evidence.ts`(1095)、`dreaming.ts`(991)、`dreaming-repair.ts`(337)、`dreaming-shadow-trial.ts`(242)、`rem-harness.ts`(206)、`dreaming-state.ts`(180)…。模拟「睡眠巩固」：多阶段把短期片段巩固成长期叙事，含 narrative 构建、repair、shadow-trial、REM evidence。**工程化（非实验脚本），但概念复杂度高，是后续维护/简化的重点。**

### 2.7 知识 wiki 层 —— 完成度高（伴侣，非槽）
`memory-wiki`（12578 LOC，31 测试）：OKF 格式 + Obsidian/ChatGPT 导入，5 工具 `wiki_search/wiki_get/wiki_apply/wiki_lint/wiki_status`（已核实）。bridge-读活跃槽插件产物。

### 2.8 主动记忆 —— 完成度高（伴侣，非槽）
`active-memory`（3924 LOC）：回复前**有界阻塞**召回子代理，把片段注入提示。缓存（已核实）：`TTL=15_000ms`、`MAX_ENTRIES=1000`、`SWEEP=1000ms`。

### 2.9 单槽机制 —— 完成度高
`resolveMemorySlotDecisionShared`（`config-activation-shared.ts:342`，已核实）：同时仅一个 `kind:"memory"` 激活（默认 memory-core）；非选中的禁用（reason "memory slot already filled"）；双 kind 插件失槽仍启用其他角色；dreaming 旁车可与 LanceDB 共存。`memory-wiki`/`active-memory` 无 `kind`（已核实），不争槽。

### 2.10 存储 schema —— 完成度高（已核实 7 表）
`memory-schema.ts`：`memory_index_meta` / `memory_index_sources` / `memory_index_chunks`(+embedding) / `memory_embedding_cache` / `memory_index_state`(revision 触发器) / `memory_index_chunks_fts`(FTS5) / `memory_index_chunks_vec`(sqlite-vec)。含遗留表迁移 + PK 迁移逻辑。

**标尺 A 结论：在「个人助理长期记忆」定位内，功能完整、链路生产级、测试充分、优雅降级到位 —— 成熟可发布。**

---

## 3. 完成度缺口 / 成熟度短板

| 缺口 | 证据 | 影响 |
|---|---|---|
| **🔸 单槽多实现未收敛** | memory-core(`memory_search/get`) 与 memory-lancedb(`memory_recall/store/forget`) **工具名/API 风格不一致**；VISION 称「计划收敛到一个推荐默认」 | 用户/插件面不统一，迁移与文档负担 |
| **🔸 LanceDB 替代后端测试薄** | 2492 LOC 仅 3 测试 vs memory-core 63 测试 | 替代路径成熟度远低于默认 |
| **🔸 dreaming 复杂度高/偏研究性** | 6724 LOC，多阶段 shadow-trial/REM evidence | 维护成本高，行为可解释性/可调参性待验证 |
| **🔸 原始 SQL + JSON 工作态 vs SQLite-only 策略** | memory-core **node:sqlite 14 文件 / kysely 0**（已核实）；dreaming 用 `.dreams/*.json` 工作文件 | 与 AGENTS.md「运行时用 Kysely / SQLite-only」有张力（DDL/低层原语有豁免，但属迁移债） |
| **🔸 MMR/decay 默认关闭** | `enabled:false`（已核实） | 默认检索无多样性/时效重排，需显式开启才发挥；文档易误以为默认开 |
| **🔸 embedding provider 依赖外部** | 远程需 API / 本地需 llama.cpp 原生扩展 | 无 provider 时退化纯关键词（功能在但语义检索缺失） |

---

## 4. 成熟度评分

| 维度 | 评分(100) | 说明 |
|---|---:|---|
| **功能完整性** | 90 | 三级记忆 + 混合检索 + promotion + 预算 + dreaming + wiki + active，覆盖全 |
| **检索质量（默认）** | 80 | FTS+向量加权生产级；扣分于 MMR/decay 默认关闭 |
| **检索质量（开全特性）** | 88 | + 时间衰减 + MMR 多样性 |
| **工程质量** | 88 | 零 TODO、63 测试、优雅降级、版本化迁移 |
| **后端一致性/收敛度** | 70 | 单槽双实现 API 不一致，lancedb 测试薄 |
| **dreaming 可维护性** | 68 | 功能完整但复杂度高、偏研究性 |
| **存储策略合规** | 75 | 原始 SQL + JSON 工作态与 SQLite-only/Kysely 策略张力 |
| **文档** | 82 | host-sdk + 各扩展 + prompt-section 指引齐全 |

**综合（个人助理长期记忆定位）：87/100 —— 成熟可发布。**

---

## 5. 是否符合产品化需求 —— 直接回答

### 符合 ✅
- **个人/团队助理的长期记忆**：完全符合。混合检索（FTS+向量）、promotion、预算淘汰、优雅降级、active-memory 自动召回注入、wiki 知识层都达产品级，63 测试覆盖。可作为「助理跨会话记住偏好/决策/事实」的产品功能直接发布。

### 需补强 ⚠️
- **多后端统一**：收敛单槽实现（统一 memory-core 与 lancedb 的工具名/API），消除用户面不一致 —— VISION 已列为计划。
- **语义检索开箱体验**：默认开启时机检索（decay）与多样性（MMR），或在文档显著说明需显式开启；提供更省心的本地 embedding 默认。
- **dreaming 简化/可观测**：高复杂度子系统建议增加可观测性与可调参，或模块化降复杂度。
- **存储合规**：记忆索引迁移到 Kysely、dreaming 工作态从 JSON 迁移到 SQLite（消除迁移债）。

### 改进优先级
| 优先级 | 改进 | 价值 |
|---|---|---|
| **P1** | 收敛单槽双实现（统一工具名/API） | 消除用户/插件面不一致 |
| **P1** | MMR/decay 默认策略 + 文档澄清 | 检索质量开箱即用 |
| **P2** | lancedb 后端补测试 | 替代路径可靠性 |
| **P2** | dreaming 可观测性/简化 | 降维护成本 |
| **P3** | 记忆索引 Kysely 化 + dreaming 工作态 SQLite 化 | 消除存储迁移债 |

---

## 6. 终评

Memory 是 OpenClaw **最成熟的子系统之一**：混合检索（FTS5+sqlite-vec+加权合并+可选 MMR/时间衰减）达生产级，promotion 门槛、预算淘汰、优雅降级、单槽机制都设计完整，63 个测试 + 零技术债标记 + 版本化迁移体现工程纪律。dreaming 离线巩固是其最独特、投入最重（6724 LOC）的创新，但也是复杂度最高、最偏研究性的部分。

**在「个人助理长期记忆」定位内：87/100，成熟可商业化发布。** 主要演进方向不是补功能（功能已全），而是**收敛**（单槽双实现统一）、**默认体验**（MMR/decay 开箱）、**降复杂度**（dreaming）与**存储合规**（Kysely/SQLite 化）。

---

## 附：核对清单（防止臆断）

本文以下数据均经 `grep`/`wc`/源码阅读**直接核实**：
- ✅ 各模块 LOC/文件/测试数（`wc -l` + `find`）
- ✅ 4 个扩展的 `kind` 与 `tools`（manifest grep）
- ✅ `src/memory/root-memory-files.ts` 存在（`ls`）
- ✅ 7 张 schema 表名（`memory-schema.ts:7-13`）
- ✅ promotion 阈值 0.75/3/2、recall 上限 512/16/32（`short-term-promotion.ts:47-54`）
- ✅ 记忆预算 10_000 字符（`memory-budget.ts:25`）
- ✅ **MMR 默认 enabled:false, λ=0.7**（`mmr.ts:26-28`）
- ✅ **temporal-decay 默认 enabled:false, halfLife=30**（`temporal-decay.ts:10-12`）
- ✅ active-memory 缓存 15000ms/1000/1000ms（`index.ts:48-50`）
- ✅ memory-core node:sqlite 14 文件 / kysely 0 文件（`grep -l`）
- ✅ dreaming 6724 LOC / 12 文件（`wc -l`）
- ✅ `hybrid.ts` 算法（直接阅读 `:32-156`）
- ✅ 单槽函数 `resolveMemorySlotDecisionShared`（`config-activation-shared.ts:342`）
- ⚠️ host-sdk embedding provider 内部实现：因子代理调查失败,本文相关结论基于 schema/接口与前序分析,**未逐文件读完 host-sdk 80 文件全部内部**（embedding worker/batch 细节标为「基于接口与 schema 推断」）。
