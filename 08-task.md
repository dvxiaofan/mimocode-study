# 08 · 任务系统

## 一句话定位
MiMo-Code 的任务系统是一个**树状 ID + 9 verb 状态机 + 持久化 + 完结闸门（task gate）**的工作追踪框架——LLM 用 `task` 工具的 9 个 action 操作任务，`T1 / T1.1 / T1.2` 树状 ID 自动按同级自增；当子 agent 想 stop 时，`TaskGate.decide()` 检查该 owner 名下未完成的任务，**最多 2 次**给子 agent 推 nudge 让它继续完成（subagent）/ **最多 3 次**给 main session 推 nudge（main 路径）。

## 涉及文件
- `packages/opencode/src/task/schema.ts` (已读) — TaskID 正则 `^T\d+(\.\d+)*$` + TaskStatus 5 态 + TaskEventKind 8 态
- `packages/opencode/src/task/registry.ts` (已读) — TaskTable CRUD + `nextChildId(parentId, siblings)` 自动 ID
- `packages/opencode/src/task/gate.ts` (已读) — `decide()` 纯函数 + `MAX_TASK_GATE_SUBAGENT_REACT=2` / `MAX_TASK_GATE_MAIN_REACT=3`
- `packages/opencode/src/task/gate-state.ts` (已读) — per-session ReAct 计数器（InstanceState）
- `packages/opencode/src/task/index.ts` — 1 行 re-export
- `packages/opencode/src/task/events.ts` (已读) — `TaskCreated` / `TaskUpdated(kind: updated only)` BusEvent
- `packages/opencode/src/task/task.sql.ts` — `task` / `task_event` 两表
- `packages/opencode/src/tool/task.ts` (已读) — `task` 工具 + 9 verb + `levenshtein` 拼写纠错 + shell-mode

## 核心数据流

### 9 verb 状态机
```
create  → status: "open"          (TaskEventKind: "created")
list    → 查
get     → 查
start   → "open"     → "in_progress"   ("started")
block   → "open|in_progress" → "blocked"   ("blocked")
unblock → "blocked"  → "open"          ("unblocked")
done    → "in_progress|open" → "done"  ("done")
abandon → 任意        → "abandoned"     ("abandoned")
rename  → 改 summary                ("renamed")
```

### TaskID 树状结构
```
T1                      ← 根
├── T1.1               ← 第 1 个子
│   ├── T1.1.1
│   └── T1.1.2
└── T1.2               ← 第 2 个子
T2
T3
```

`TaskID` 正则：`/^T\d+(\.\d+)*$/`
- `T1`、`T1.1`、`T1.1.2` 都合法
- `T`、`T.1`、`T1..1` 不合法

### `nextChildId(parentId, siblings)` 算法（`registry.ts:35-46`）
```typescript
const prefix = parentId ? `${parentId}.` : "T"
const used = siblings
  .filter((s) => (parentId ? s.startsWith(prefix) : /^T\d+$/.test(s)))
  .map((s) => {
    const tail = s.slice(prefix.length)
    return /^\d+$/.test(tail) ? Number(tail) : 0
  })
const next = used.length > 0 ? Math.max(...used) + 1 : 1
return `${prefix}${next}`
```
- 无 parent → `T1, T2, T3`...
- 有 parent `T1` → `T1.1, T1.2, T1.3`...
- 取同级最大 + 1（不补空缺）

### TaskGate 完结闸门
```
子 agent 想 stop / main session 完结
  ↓
TaskGate.decide({ session_id, owner, reactCount, maxReact, mode })
  ├─ mode=subagent: 查 owner=actorID 的非终态任务
  └─ mode=main:     查 session 下所有非终态任务（含 subagent-orphaned）
  ↓
3 种结果：
  1. { needReentry: false, capExceeded: false, incompleteTasks: [] }   → 允许 stop
  2. { needReentry: true, reentryText, incompleteTasks, capExceeded: false } → 注入 reminder 继续
  3. { needReentry: false, capExceeded: true, incompleteTasks: [...] } → 强制 stop（标 incompleteTasks）
```

## 关键设计

### 1. **TaskID 强校验**（`schema.ts:3`）
```typescript
export const TaskID = z.string().regex(/^T\d+(\.\d+)*$/, "Task ID must be Tn or Tn.m...")
```
- 提前在 zod 层面 reject `T1.` / `T` / `T.a`
- `t.id` 类型自动收窄为 `T\d+(\.\d+)*`

### 2. **5 态 TaskStatus**（`schema.ts:5-7`）
| Status | 含义 |
|--------|------|
| `open` | 创建后默认 |
| `in_progress` | start 后 |
| `blocked` | block 后（unblock 回到 open） |
| `done` | done 后（终态） |
| `abandoned` | abandon 后（终态） |

### 3. **8 态 TaskEventKind**（`schema.ts:23-32`）
`created / started / unstarted / blocked / unblocked / done / abandoned / renamed`
- **8 态 ≠ 5 态**：transitions 单独记（`created` / `started` 等）
- `TaskEventKind.exclude(["created"])` → `UpdatedKind`（不让"创建"和"变更"混在一起发事件，注释明确说"matching actor.registered vs actor.status"）
- `TaskCreated` / `TaskUpdated(kind: UpdatedKind)` 两个独立 bus event

### 4. **Task 5 个时间字段**（`schema.ts:10-19`）
- `created_at` / `last_event_at` — 必填
- `ended_at` / `cleanup_after` — 终态才有
- `last_event_at` 每次 verb 操作更新——用于按活跃度排序

### 5. **task 工具 9 verb + zod strictObject**（`tool/task.ts:30-110`）
```typescript
const KNOWN_VERBS = ["create", "list", "get", "start", "block", "unblock", "done", "abandon", "rename"]
const createOperation = z.strictObject({ action: z.literal("create"), summary: z.string().min(1), ... })
const listOperation   = z.strictObject({ action: z.literal("list"), status: statusSchema.optional(), ... })
const getOperation    = z.strictObject({ action: z.literal("get"), id: z.string().min(1), ... })
const startOperation  = z.strictObject({ action: z.literal("start"), id: z.string().min(1), event_summary: z.string().min(1).optional(), ... })
const blockOperation  = z.strictObject({ action: z.literal("block"), id: z.string().min(1), event_summary: z.string().min(1).optional(), ... })
// ... unblock / done / abandon / rename
```
- **`z.strictObject`**：拒绝未知字段
- **9 verb 全部严格类型化**

### 6. **拼写纠错：levenshtein**（`tool/task.ts:9-28`）
```typescript
function levenshtein(a: string, b: string): number { /* DP */ }
function suggestVerb(input: string): string | undefined {
  const candidates = KNOWN_VERBS.map((v) => ({ v, d: levenshtein(input, v) })).filter((c) => c.d <= 2)
  if (candidates.length !== 1) return undefined  // 多个候选项时不建议（避免误纠）
  return candidates[0].v
}
```
- LLM 写错 `tasl done` → 建议 `task done`
- **distance ≤ 2** + **唯一候选**才建议（防多解歧义）

### 7. **task 工具支持 shell-mode**（`tool/task.ts:1-2, tokenize`）
- `DESCRIPTION` (JSON 形态) + `SHELL_DESCRIPTION` (shell 形态) 两个 prompt
- `tokenize` 把 shell 命令切分成 verb + args
- `task list` / `task done T1 完成` 之类

### 8. **TaskGate.decide() 纯函数**（`gate.ts:38-100`）
```typescript
export const decide = Effect.fn("TaskGate.decide")(function* (input: DecideInput) {
  // 1. 查 non-terminal tasks
  // 2. reactCount < maxReact && 有 incomplete → needReentry: true
  // 3. reactCount >= maxReact → capExceeded: true
  // 4. 空 → 允许 stop
})
```
**关键注释**（`gate.ts:84-90`）：
> "orElseSucceed on registry failure: a transient DB error must NEVER trap the agent in the gate — fail open by reporting empty so the caller stops cleanly."

**transient DB 错误 fail-open**——避免因为 DB 抖一下 agent 永远跑不完。

### 9. **buildReentryText 的 owner 语义**（`gate.ts:38-60`）
```typescript
const headline =
  mode === "subagent"
    ? "You are about to finish, but these tasks you own are still unfinished:"
    : "You are about to finish, but these tasks in this session are still unfinished:"
```
**关键差异**：
- subagent：`owner=actorID` — 列出的任务**都属于**收件人 → "you own" 准确
- main：`owner=undefined` — 列出**session 内所有**任务（含 subagent-orphaned）→ 不能说 "you own"，否则会误导 main agent 倾向 abandon 而不是 complete

### 10. **MAX_TASK_GATE_SUBAGENT_REACT=2 vs MAIN_REACT=3**（`gate.ts:8-20`）
- **subagent 2 次**：短生命周期，2 次 nudge 还没完成就标 "partial"/"blocked" 返回
- **main 3 次**：长生命周期，但比 `MAX_GOAL_REACT=12` 低
- **关键理由**（注释）：main 没有 judge 模型（只有 DB state），无法区分"model 还在工作" vs "model 正在卡死"——所以 cap 严一些

### 11. **TaskGateState per-session counter**（`gate-state.ts:1-50`）
- InstanceState 里 `Map<sessionID, count>` —— 切 instance 自动失效
- `get` / `bump` / `clear` 三个方法
- 模式跟 `Goal` 完全一样（注释明确说"mirrors Goal.bumpReact"）

### 12. **Tool + Gate 联动**
- checkpoint-writer 在写 §4 Task tree 时，**唯一来源** = `task` 工具的返回
- 注释明确说："If a section is empty, render it empty — never invent."
- 命名规则："Use ONLY the IDs returned by `task` tool... Never write `T6.1 (T7)`"——agent 在对话里说的 logical label 不可信，DB ID 是 source of truth

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 任务系统 | 无 | **TaskTable + TaskEventTable + 9 verb 状态机** | 完整自研 |
| 树状 ID | 无 | **`T1 / T1.1` + nextChildId 自增** | 层级化 |
| TaskGate | 无 | **decide() 纯函数 + subagent=2 / main=3 cap** | 完结前检查 |
| 8 态 TaskEventKind | 无 | **transitions 单独记 + `UpdatedKind = exclude(["created"])`** | 区分"创建"与"变更" |
| zod strictObject | 无 | **9 verb 全部严格类型化** | 拒绝未知字段 |
| levenshtein 纠错 | 无 | **distance ≤ 2 + 唯一候选才建议** | 防误纠 |
| Shell 形态 | 无 | **`task list` / `task done T1 完成`** | 双形态 |
| owner 语义 | 无 | **subagent 模式 "you own" / main 模式 "in this session"** | 准确引导 |
| Fail-open on DB error | 无 | **"transient DB error must NEVER trap the agent"** | 鲁棒性 |
| Checkpoint §4 联动 | 无 | **"Use ONLY IDs returned by `task` tool"** | 防 LLM 编造 |

## 关键引用

- `packages/opencode/src/task/schema.ts:3-19` — TaskID 正则 + TaskStatus 5 态 + Task 字段
- `packages/opencode/src/task/schema.ts:23-32` — TaskEventKind 8 态
- `packages/opencode/src/task/registry.ts:35-46` — `nextChildId` 自增算法
- `packages/opencode/src/task/gate.ts:8-20` — MAX cap 常量
- `packages/opencode/src/task/gate.ts:38-60` — `buildReentryText` owner 语义
- `packages/opencode/src/task/gate-state.ts:1-50` — per-session ReAct counter
- `packages/opencode/src/task/events.ts:1-40` — Created / Updated BusEvent
- `packages/opencode/src/tool/task.ts:1-110` — 9 verb + levenshtein
- `packages/opencode/src/agent/prompt/checkpoint-writer.txt:60-90` — §4 渲染规则
- `packages/opencode/migration/20260422170000_task_registry/` + `20260515020000_user_task/` + `20260603000000_task_in_progress_owner/` — schema 演进

## 学习检查清单

读完后应能回答：
- [ ] TaskID 的正则约束是什么？为什么用 strictObject 而不是 object？
- [ ] 9 verb 各自对应什么状态转移？哪些是终态？
- [ ] `nextChildId` 怎么保证同级唯一、跨级不冲突？
- [ ] `TaskEventKind` 8 态和 `TaskStatus` 5 态的区别是什么？为什么 `UpdatedKind` 要 `exclude(["created"])`？
- [ ] `TaskGate.decide()` 的 3 种 Decision 含义？`subagent=2 / main=3` cap 为什么不一致？
- [ ] `buildReentryText` 在 subagent 模式说 "you own"、main 模式说 "in this session" 为什么不一致？
- [ ] `decide()` 失败时为什么 fail-open 报 empty？背后的设计哲学？
- [ ] checkpoint-writer 写 §4 时为什么说"Use ONLY IDs returned by `task` tool"？
