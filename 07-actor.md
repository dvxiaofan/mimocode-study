# 07 · 子 Agent 系统

## 一句话定位
MiMo-Code 的子 agent 系统（"Actor"）是一个**完整的并发单元生命周期管理框架**——每个 actor 在 SQLite 行里持久化（崩溃可恢复）、有 lifecycle（ephemeral / persistent）、有 contextMode（none / state / full 三种上下文继承）、有 stuck 检测（5 分钟无 turn 自动 bus 事件）、有 3 种 spawn 模式（peer / subagent / main）和 3 种 outcome（success / failure / cancelled），配套 `actor` 工具让主 agent 能 spawn / wait / cancel / status 任意子 agent。

## 涉及文件
- `packages/opencode/src/actor/schema.ts` (已读) — `Actor / ActorStatus / ActorOutcome / Lifecycle / ContextMode / SpawnMode / ToolWhitelist` zod schemas
- `packages/opencode/src/actor/registry.ts` (已读) — Actor CRUD + **stuck 检测 fiber** + orphan recovery
- `packages/opencode/src/actor/spawn.ts` (已读) — `spawn` / `cancel` / `getForkContext` 三个核心 + `ForkContext` / `SpawnInput` / `AgentOutcome` 类型
- `packages/opencode/src/actor/turn.ts` (已读) — `runTurn` Effect 包装：保证 status cleanup 一定跑
- `packages/opencode/src/actor/waiter.ts` (已读) — `wait` API（轮询 / 阻塞）+ 6 种结果状态
- `packages/opencode/src/actor/events.ts` (已读) — 4 个 BusEvent：Registered / StatusChanged / Stuck / WriterCachePerf + InboxArrived
- `packages/opencode/src/actor/return-header.ts` — 解析子 agent 返回的 `**Status**: ...` / `**Summary**: ...` 协议
- `packages/opencode/src/actor/spawn-ref.ts` — module-level ref 懒绑定
- `packages/opencode/src/actor/actor.sql.ts` — `actor_registry` 表 schema
- `packages/opencode/src/tool/actor.ts` — `actor` 工具（spawn / wait / cancel / status 4 个 operation）
- `packages/opencode/src/inbox/*` — 子 agent 通知队列
- `packages/opencode/migration/20260527000000_actor_lifecycle/` + `20260515000000_actor_rename/` + `20260515010000_actor_registry_v6/` — schema 演进

## 核心数据流

### Actor 生命周期状态机
```
[注册] → register()
  status: "pending"   (在 actor_registry 表里 INSERT)
  ↓
[被调度] → runAgentLoop()
  status: "running"   (turnCount: 0 → 1)
  ↓
[执行中] 每次 turn 完成
  status: "running" (turnCount++)
  ↓
[本轮结束]
  status: "idle"
  lastOutcome: "success" / "failure" / "cancelled"
  time.completed: now
  ↓
[可能被重新调度] (lifecycle=persistent)
  status: "pending" → "running" → "idle"
  ↓
[孤儿] 进程崩溃但 actor 还在 "running"
  下次启动 orphan recovery：status: "idle" + lastOutcome: "failure" + lastError: "orphaned: process restarted"
```

### 3 种 spawn 模式
| 模式 | 说明 | 关系 |
|------|------|------|
| `peer` | 同级 agent（独立任务、共享 session 上下文） | 不挂在 parent 下 |
| `subagent` | 子 agent（任务可能调用子 agent） | `parent_actor_id` 记录 |
| `main` | 主 session（单例，actorID = "main"） | `parent_actor_id = null` |

### 3 种 ContextMode（contextMode.ts）
| Mode | 含义 | 用例 |
|------|------|------|
| `none` | 不继承任何上下文 | 全新独立任务 |
| `state` | 只继承 state（checkpoint 摘要、task 树） | 同 session 延续上下文 |
| `full` | 继承完整 messages 到 watermark（含 prefix cache） | checkpoint-writer fork |

### 2 种 Lifecycle
- `ephemeral`（默认）：执行完就清理
- `persistent`：保留在表里等被 reuse

### Spawn 流程
```
主 agent 调 actor 工具 { operation: "spawn", task: "...", agent: "explore" }
  ↓
Actor.Service.spawn({ mode, sessionID, agentType, task, context, tools, ... })
  ├─ allocateActorID(sessionID, agentType) → "explore-1"
  ├─ ActorRegistry.register(...) → INSERT + bus.publish(ActorRegistered)
  ├─ forkWork (在 actor scope 内 fork fiber)
  │    ├─ runAgentLoop → SessionPrompt.prompt
  │    ├─ runTurn wraps: 状态更新一定跑（success/failure/cancelled）
  │    └─ Deferred<AgentOutcome>.succeed / fail / interrupt
  └─ 返回 { actorID, sessionID, outcome: Deferred }
  ↓
主 agent 拿到 actorID，可调 actor 工具 wait
```

## 关键设计

### 1. **register() 一手 INSERT + bus event**（`registry.ts:128-156`）
```typescript
const register = Effect.fn("ActorRegistry.register")(function* (input) {
  const now = Date.now()
  const row = { session_id, actor_id, mode, parent_actor_id: input.parentActorID ?? null,
                status: "pending" as const, last_outcome: null, lifecycle, agent, description,
                context_mode, context_watermark: input.contextWatermark ?? null, background,
                tools: input.tools ?? null, last_turn_time: now, turn_count: 0, last_error: null, ... }
  yield* Effect.sync(() => Database.use((db) => db.insert(ActorRegistryTable).values(row).run()))
  yield* bus.publish(Events.ActorRegistered, { ... })
  return fromRow(row)
})
```
- **同步 INSERT**（不包裹事务——写一行就够）
- **紧接 bus.publish**：TUI / 监控能立即看到 actor 出现

### 2. **updateStatus() 重读后 publish**（`registry.ts:158-200`）
```typescript
const updateStatus = Effect.fn("ActorRegistry.updateStatus")(function* (...) {
  // 1. UPDATE
  // 2. SELECT 拿最新 row
  if (!row) return  // 行没了就不 publish
  yield* bus.publish(Events.ActorStatusChanged, { ...row... })
})
```
**关键注释**："a dropped event beats a misleading one"——行消失就不发事件，避免误报。

### 3. **Orphan Recovery**（`registry.ts:355-378`）
```typescript
yield* Effect.sync(() =>
  Database.use((db) => {
    const now = Date.now()
    db.update(ActorRegistryTable)
      .set({ status: "idle", last_outcome: "failure", last_error: "orphaned: process restarted", ... })
      .where(inArray(ActorRegistryTable.status, ["pending", "running"]))
      .run()
  }),
)
log.info("orphan recovery complete")
```
- **启动时跑一次**（layer 初始化时）
- **不自动 revive**——下次 sender send 才唤醒
- 标记 `lastError: "orphaned: process restarted"`——可区分

### 4. **Stuck Detection Fiber**（`registry.ts:13, 380-403`）
```typescript
const STUCK_THRESHOLD_MS = 5 * 60 * 1000   // 5 minutes
const SCAN_INTERVAL_MS = 60 * 1000         // 每 60s 扫一次
const scanStuck = Effect.gen(function* () {
  const cutoff = Date.now() - STUCK_THRESHOLD_MS
  const stuck = yield* Effect.sync(() => Database.use((db) =>
    db.select().from(ActorRegistryTable)
      .where(and(eq(ActorRegistryTable.status, "running"), lte(ActorRegistryTable.last_turn_time, cutoff)))
      .all()))
  for (const row of stuck) {
    yield* bus.publish(Events.ActorStuck, { sessionID, actorID, description, lastTurnTime, stuckDuration })
  }
})
yield* scanStuck.pipe(Effect.repeat(Schedule.fixed(SCAN_INTERVAL_MS)), Effect.ignore, Effect.forkScoped)
```
- **5 分钟无 turn** → 发 `ActorStuck` 事件
- **TUI 监听** → 渲染"stuck 5m" 提示
- **fork 在 scope 内**——layer 关掉时自动停

### 5. **runTurn 强制 status cleanup**（`turn.ts:14-44`）
```typescript
export const runTurn = <A, E>(sessionID, actorID, work) =>
  Effect.uninterruptible(Effect.gen(function* () {
    const reg = yield* ActorRegistry.Service
    yield* reg.updateStatus(sessionID, actorID, { status: "running" }).pipe(Effect.ignore)
    const exit = yield* work.pipe(Effect.interruptible, Effect.exit)
    // ... 根据 exit 写 status + lastOutcome + lastError
  }))
```
- **`Effect.uninterruptible` 包裹**：外层不可中断
- **`work.pipe(Effect.interruptible)`**：内层可被 `Fiber.interrupt` 取消
- **保证 outcome 一定写入**：cancelled vs failure 由 `Cause.hasInterruptsOnly(cause)` 区分

### 6. **ForkContext 完整快照**（`spawn.ts:80-105`）
```typescript
export interface ForkContext {
  readonly system: string[]                                 // 父 system prompt 全部
  readonly tools: Record<string, AITool>                     // 父 tools schema snapshot
  readonly parentPermission: Permission.Ruleset             // 父 permission
  readonly inheritedMessages: ModelMessage[]                 // 父 messages 全部到 watermark
  readonly watermarkMsgID: MessageID                         // 边界 marker
  readonly model: { providerID: ProviderID; modelID: ModelID }
}
```
- **"fork" 指 checkpoint-writer**：用 ForkContext 让 fork 出去的 LLM request 和 parent **完全一致**（保 prefix cache）
- **per-actor `Map<actorID, ForkContext>`**：内存里，不进 DB（snapshot 太大）
- **terminate 时清理**

### 7. **allocateActorID 自增**（`registry.ts:331-347`）
```typescript
const allocateActorID = Effect.fn("ActorRegistry.allocateActorID")(function* (sessionID, agentType) {
  const existing = yield* Database.use((db) =>
    db.select({ actor_id: ... }).from(ActorRegistryTable)
      .where(and(eq(session_id), eq(agent, agentType))).all())
  const prefix = `${agentType}-`
  let max = 0
  for (const row of existing) {
    if (row.actor_id.startsWith(prefix)) {
      const n = parseInt(row.actor_id.slice(prefix.length), 10)
      if (Number.isFinite(n) && n > max) max = n
    }
  }
  return `${agentType}-${max + 1}`
})
```
- **形式**：`explore-1`、`explore-2`...
- **agentType 区分**：`build-1`、`plan-1` 互不冲突

### 8. **renderForAgent 给 LLM 看**（`registry.ts:280-298`）
```typescript
const renderForAgent = Effect.fn("ActorRegistry.renderForAgent")(function* (sessionID) {
  const actors = yield* listBySession(sessionID)
  const active = actors.filter((actor) => actor.background && (actor.status === "pending" || actor.status === "running"))
  if (active.length === 0) return ""
  const lines: ["## Active Actors", "", `You have ${active.length} background actor(s) registered. ...`, ...]
})
```
- 只渲染**后台运行**的 actor
- 注入到 system prompt——LLM 知道"我还有 subagent 在跑"

### 9. **isSystemSpawned**（`registry.ts:316-326`）
```typescript
const isSystemSpawned = Effect.fn("ActorRegistry.isSystemSpawned")(function* (...) {
  if (actorID === "main") return false
  const actor = yield* get(sessionID, actorID)
  if (!actor) return false
  return SYSTEM_SPAWNED_AGENT_TYPES.has(actor.agent)  // { "checkpoint-writer", "dream", "distill" }
})
```
- 用于 permission ask 时 `interactive: false`（没人答）

### 10. **Wait 6 种结果**（`waiter.ts:8-19`）
```typescript
export interface WaitResult {
  status: Actor["status"] | "timeout" | "unknown"
  actor_id: string
  description?: string
  agent?: string
  background?: boolean
  turnCount?: number
  lastTurnTime?: number
  result?: string
  structured?: unknown
  error?: string
  lastOutcome?: Actor["lastOutcome"]
  reportedStatus?: ReturnStatus        // 子 agent 头里的 "**Status**: success"
  reportedSummary?: string             // 子 agent 头里的 "**Summary**: ..."
  time?: { ... }
}
```
- `isWaitResolving()` 决定何时 resolve：
  - ephemeral → status=idle
  - persistent → status=idle **且** lastOutcome !== undefined && lastOutcome !== "success"
- `DEFAULT_TIMEOUT_MS = 600_000`（10 分钟）

### 11. **onActorID 同步回调**（`spawn.ts:170-180`）
```typescript
// Fired SYNCHRONOUSLY with the freshly-allocated actorID inside the spawn
// Effect — right after the actor is registered, BEFORE its work fiber detaches
// (forkWork forks into the actor scope). Lets a caller record the child id the
// instant the actor exists, closing the window where an in-flight spawn would
// otherwise be invisible to a concurrent cancel/reclaim.
onActorID?: (actorID: string) => void
```
- **WorkflowRuntime 用**：spawn 完立刻 add 到 reclaim set——关掉"in-flight spawn 不可见"的窗口
- **best-effort**：try/catch 吞 throw 防止 buggy callback 杀 spawn

### 12. **actor 工具 4 operation**（`tool/actor.ts`）
- `operation: "spawn"` — 触发 Actor.Service.spawn
- `operation: "wait"` — 触发 ActorWaiter.wait
- `operation: "cancel"` — 触发 Actor.Service.cancel(mode: graceful/forced)
- `operation: "status"` — 查 actor_registry 状态

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 子 agent | 无正式抽象 | **Actor 系统**（注册表 + 生命周期 + 状态机） | 完整自研 |
| spawn 模式 | 无 | **peer / subagent / main 3 种** | 灵活分工 |
| ContextMode | 无 | **none / state / full 3 种** | 上下文继承粒度 |
| Lifecycle | 无 | **ephemeral / persistent 2 种** | 是否持久化 |
| Fork protocol | 无 | **ForkContext 完整 snapshot + prefix cache 对齐** | checkpoint-writer 用 |
| Stuck 检测 | 无 | **5 min 阈值 + 60s 扫描 + bus 事件** | 自动发现卡死 |
| Orphan recovery | 无 | **启动时把 pending/running 标 failed** | 进程崩溃恢复 |
| allocateActorID | 无 | **`{agentType}-{n}` 自增** | 人类可读 |
| renderForAgent | 无 | **后台 actor 状态注入 system prompt** | LLM 知道有活在做 |
| 6 种 WaitResult | 无 | **structured / reportedStatus / reportedSummary** | 子 agent 头协议 |
| onActorID 同步回调 | 无 | **fiber detach 前 fire** | 修 in-flight 不可见窗口 |
| MAX_PRE_REACT / MAX_POST_REACT | 无 | **actor 3 / main 12** | 不同 scope 步数上限 |

## 关键引用

- `packages/opencode/src/actor/schema.ts:1-50` — 7 个枚举 + Actor 类型
- `packages/opencode/src/actor/registry.ts:128-156` — register
- `packages/opencode/src/actor/registry.ts:158-200` — updateStatus（重读后 publish）
- `packages/opencode/src/actor/registry.ts:355-378` — Orphan Recovery
- `packages/opencode/src/actor/registry.ts:380-403` — Stuck Detection
- `packages/opencode/src/actor/spawn.ts:80-105` — ForkContext
- `packages/opencode/src/actor/spawn.ts:170-180` — onActorID 同步回调
- `packages/opencode/src/actor/turn.ts:14-44` — runTurn 强制 cleanup
- `packages/opencode/src/actor/waiter.ts:8-19` — WaitResult 6 种状态
- `packages/opencode/src/actor/events.ts:1-50` — 4 个 BusEvent
- `packages/opencode/migration/20260515000000_actor_rename/` + `20260527000000_actor_lifecycle/` + `20260515010000_actor_registry_v6/` — schema 演进

## 学习检查清单

读完后应能回答：
- [ ] Actor 的 3 种 spawn 模式、3 种 ContextMode、2 种 Lifecycle 各自含义和用例？
- [ ] Actor 的 5 个状态（pending / running / idle + lastOutcome）怎么转移？
- [ ] orphan recovery 在什么时候跑、做什么？
- [ ] stuck detection 的阈值和扫描间隔分别是多少？发什么 bus 事件？
- [ ] runTurn 为什么要用 `Effect.uninterruptible` 包裹外层、内层 `Effect.interruptible`？
- [ ] ForkContext 包含哪 6 个字段？它在 fork 协议中起什么作用？
- [ ] `allocateActorID` 为什么按 agentType 区分自增？
- [ ] `onActorID` 同步回调的用途是什么？哪个调用方用到？
