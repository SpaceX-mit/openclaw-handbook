# 08 · 会话树存储

来源：`harness/session/session.ts`、`storage-base.ts`、`jsonl-storage.ts`、`memory-storage.ts`、`uuid.ts`、`timestamps.ts`。

## 8.1 核心设计：append-only 树

会话树是**只增不改**的树形存储，每条 `SessionTreeEntry` 有 `{id, parentId, timestamp}`，构成有向无环树（根节点 `parentId=null`）。当前活跃分支由**叶子指针**（leafId）标记，切换叶子即等于切换对话分支。

设计理由：
- prompt cache 字节稳定：历史条目不改，缓存命中率高。
- 可审计：完整历史保留，压缩也只是追加一条 compaction 条目。
- 分支探索：可任意跳转到历史条目重新分叉，而不是覆盖历史。

## 8.2 Session 类（`session.ts:106`）

高级 API，包装 `SessionStorage`：

| 方法 | 说明 |
|---|---|
| `buildContext(): Promise<SessionContext>` | 从当前叶到根的路径构建 `{messages, thinkingLevel, model}` |
| `getBranch(fromId?): Promise<SessionTreeEntry[]>` | 叶到根的有序路径（祖先序） |
| `appendMessage(msg)` | 追加 message 条目，parentId = appendParentId |
| `appendThinkingLevelChange/appendModelChange` | 追加状态标记 |
| `appendCompaction(summary, firstKeptEntryId, tokensBefore, details?, fromHook?)` | 追加压缩条目 |
| `appendCustomEntry/appendCustomMessageEntry` | 追加自定义条目（后者可回放进上下文） |
| `appendLabel(targetId, label)` | 追加标签（targetId 必须存在） |
| `appendSessionName(name)` | 追加会话名称 |
| `moveTo(entryId, summary?)` | 切换叶子 + 可选追加 branch_summary 条目 |
| `getEntry/getEntries/getLabel/getSessionName` | 读取 |

**appendParentId 设计**：默认用 `getLeafId()`；若存储实现了 `getAppendParentId()`，则优先用之（`appendMode:"side"` 支持旁路游标，即新条目挂在旁路而非可见叶）。

## 8.3 buildSessionContext（压缩感知重放，`session.ts:28`）

从祖先序路径条目构建上下文消息。这是理解「会话树如何变成 LLM 上下文」的关键算法：

```
buildSessionContext(pathEntries):
  thinkingLevel = "off"; model = null; compaction = null
  # 扫描全路径，取最后的 thinking_level_change/model_change/assistant-role-message/compaction
  for entry in pathEntries:
    if thinking_level_change → thinkingLevel = entry.thinkingLevel
    if model_change          → model = {provider, modelId}
    if message && role=assistant → model = {provider: msg.provider, modelId: msg.model}
    if compaction            → compaction = entry   # 记录最后一次（应该是最近一次）

  messages = []
  appendMessage(entry): 把 message/custom_message/branch_summary 条目转成 AgentMessage 推进 messages

  if compaction:
    # 1. 先推一条 compactionSummaryMessage（摘要，替代旧历史）
    messages.push(createCompactionSummaryMessage(summary, tokensBefore, timestamp))
    compactionIdx = pathEntries.findIndex(e => e.id === compaction.id)
    # 2. 从压缩条目之前，找 firstKeptEntryId 起的条目（保留尾部）
    foundFirstKept = false
    for i in [0..compactionIdx):
      if entries[i].id === compaction.firstKeptEntryId: foundFirstKept = true
      if foundFirstKept: appendMessage(entries[i])
    # 3. 压缩条目之后的所有条目（新增历史）
    for i in [compactionIdx+1..end): appendMessage(entries[i])
  else:
    # 无压缩：直接重放全部
    for entry in pathEntries: appendMessage(entry)

  return {messages, thinkingLevel, model}
```

**关键点**：压缩条目把它之前的旧历史替换为摘要消息，但保留 `firstKeptEntryId` 之后的「保留尾部」直接重放，使上下文窗口只含：摘要 + 保留尾部 + 压缩后新增条目。

## 8.4 SessionStorage 接口（必须实现，`harness/types.ts:472`）

```ts
interface SessionStorage<TMetadata extends SessionMetadata> {
  getMetadata(): Promise<TMetadata>;
  getLeafId(): Promise<string | null>;
  getAppendParentId?(): Promise<string | null>;   // 可选旁路游标
  setLeafId(leafId: string | null): Promise<void>;
  createEntryId(): Promise<string>;               // 通常用 uuidv7()
  appendEntry(entry: SessionTreeEntry): Promise<void>;
  getEntry(id: string): Promise<SessionTreeEntry | undefined>;
  findEntries<TType extends SessionTreeEntry["type"]>(type): Promise<Array<...>>;
  getLabel(id: string): Promise<string | undefined>;
  getPathToRoot(leafId: string | null): Promise<SessionTreeEntry[]>;   // 从叶到根，祖先序
  getEntries(): Promise<SessionTreeEntry[]>;
}
```
`getPathToRoot` 需要把叶到根的路径**按从根到叶的顺序**（祖先序）返回，以便 buildSessionContext 正向遍历。

## 8.5 两种内置存储实现

### JsonlSessionStorage（`jsonl-storage.ts`）
文件格式：每行一个 JSON 序列化的 `SessionTreeEntry`（JSONL）。
- 构造：`new JsonlSessionStorage(path, metadata)`。
- 元数据（`JsonlSessionMetadata`）：`{id, createdAt, cwd, path, parentSessionPath?}`。
- `appendEntry`：`fs.appendFile`（每次追加一行）。
- `getPathToRoot`：读全部条目，从叶按 parentId 链爬到根，reverse 成祖先序。
- `createEntryId`：uuidv7()。
- 叶子指针：持久化在单独的 `<path>.leaf` 文件（或内嵌 `LeafEntry` 条目中最后一条）。

### MemorySessionStorage（`memory-storage.ts`）
全内存，测试/短生命周期会话用。实现 SessionStorage 接口，用 Map 存条目，用变量存 leafId。

## 8.6 uuidv7（`session/uuid.ts`）
基于 RFC 9562 UUIDv7（时间有序）。算法：取 `Date.now()`（48 位毫秒时间戳）+ 4 位版本（0x7）+ 12 位随机 + 2 位变体（0b10）+ 62 位随机，以连字符分隔十六进制格式输出。重写时可用标准库实现，须保证时间单调递增且全局唯一。

## 8.7 timestamp 格式（`session/timestamps.ts`）
条目 timestamp 用 ISO 8601 字符串存储（`new Date().toISOString()`），但 `CompactionSummaryMessage.timestamp` 接受 `number|string`（历史兼容）。`parseSessionTimestampMs(str): number|null` 用 `Date.parse`；`requireSessionTimestampMs(str, label)` 解析失败时返回 `0`（而非抛错，注释说明「不要因脏数据中止上下文转换」）。

## 8.8 会话树操作示意

```
初始:  [root: user"hello"] ← leaf
压缩:  [root: user"hello"] → [compaction{firstKeptId=root}] ← leaf
       （context: compactionSummary + root.message）
分支:  user B' = navigateTree(root)
       [root] → [compaction] ← branch_summary{fromId=compaction,summary="..."} ← leaf(B')
       原分支: [root] → [compaction] ← [assistant] ← [user C]（历史保留不删）
```
