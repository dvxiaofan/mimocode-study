# 11 · Effect Runtime + DB

## 一句话定位
MiMo-Code 用 **Effect v4** 替代了 OpenCode 的裸 Promise/async-await 风格——所有长生命周期 Service 走 `Context.Service` + `Layer.effect` 模式，按"per-instance"作用域懒加载、按"per-instance teardown"清理；DB 层用 **drizzle-orm + bun:sqlite**（Node 兜底 better-sqlite3）+ 34 个有序迁移做单一真相源，WAL/NORMAL/5s busy_timeout 是性能基线。

## 涉及文件
### Effect 运行时（13 文件）
- `packages/opencode/src/effect/index.ts` — 5 行 re-export
- `packages/opencode/src/effect/runtime.ts` — ManagedRuntime 单例
- `packages/opencode/src/effect/run-service.ts` — `makeRuntime<I, S, E>(service, layer)` 工厂
- `packages/opencode/src/effect/instance-state.ts` — `ScopedCache<string, A>` 跨 instance 缓存
- `packages/opencode/src/effect/instance-ref.ts` — `InstanceRef` / `WorkspaceRef` Effect Refs
- `packages/opencode/src/effect/instance-registry.ts` — `registerDisposer` teardown 注册
- `packages/opencode/src/effect/bridge.ts` — Effect ↔ 同步桥（`Effect.runPromise` / `Effect.runFork`）
- `packages/opencode/src/effect/cross-spawn-spawner.ts` — Effect ChildProcessSpawner 实现
- `packages/opencode/src/effect/logger.ts` — Effect Logger → util/Log 适配
- `packages/opencode/src/effect/observability.ts` — OpenTelemetry OTLP exporter
- `packages/opencode/src/effect/memo-map.ts` — Runtime memoMap（防 fiber 重复）
- `packages/opencode/src/effect/bootstrap-runtime.ts` — 启动期 runtime（DB init 前）
- `packages/opencode/src/effect/app-runtime.ts` — 应用期 runtime（DB init 后）

### DB 抽象（4 文件 + 34 迁移）
- `packages/opencode/src/storage/index.ts` — re-export Database / Storage / drizzle operators
- `packages/opencode/src/storage/db.ts` — `Database.use()` / `Database.transaction()` / `Database.effect()` 上下文
- `packages/opencode/src/storage/db.bun.ts` — Bun SQLite 后端
- `packages/opencode/src/storage/db.node.ts` — Node better-sqlite3 后端
- `packages/opencode/src/storage/storage.ts` — JSON 文件存储（draft/update 模式）
- `packages/opencode/src/storage/json-migration.ts` — 老 JSON 存储 → SQLite 迁移
- `packages/opencode/src/storage/schema.ts` — 11 个表统一 re-export
- `packages/opencode/src/storage/schema.sql.ts` — drizzle sqliteTable 定义
- `packages/opencode/migration/2026*.sql` — 34 个有序迁移（2026-01-27 → 2026-06-09）

## 核心数据流

### Effect Service 注入模式
```
agent.ts 定义:
  export class Service extends Context.Service<Service, Interface>()("@opencode/Agent") {}
  export const layer = Layer.effect(Service, Effect.gen(function*() { ... }))

  export const defaultLayer = layer.pipe(
    Layer.provide(Plugin.defaultLayer),
    Layer.provide(Provider.defaultLayer),
    Layer.provide(Auth.defaultLayer),
    Layer.provide(Config.defaultLayer),
    Layer.provide(Skill.defaultLayer),
  )

调用方:
  const agent = yield* Agent.Service                    // 拿 Service
  const info = yield* agent.get("build")                // 调方法
```

### Database 上下文流
```
启动:
  Database.init() 读 Flag.MIMOCODE_DB 路径
  Client() lazy singleton
  PRAGMA: WAL / NORMAL / busy_timeout=5s / cache=64MB / foreign_keys=ON

每次访问:
  Database.use((db) => db.select().from(...).get())     // 自动注入 ctx
  ├─ ctx 已存在（事务中）→ 用 ctx.tx
  └─ ctx 不存在（首次） → ctx.provide({tx: Client(), effects}, callback)
                          for (effect of effects) effect()   // 异步副作用同步触发

事务:
  Database.transaction((tx) => ..., { behavior: "deferred" | "immediate" | "exclusive" })
```

### 迁移应用
```
db.bun.ts init() 返回 drizzle client
  ↓
Client() lazy init
  ├─ typeof OPENCODE_MIGRATIONS !== "undefined"（Bun --bundle 注入的常量）
  │   → 用 bundled 模式（生产）
  └─ 否则 → 读 migration/ 目录（dev）
  ↓
drizzle migrate(db, entries)  // 按 timestamp 升序
```

## 关键设计

### 1. **Per-Instance ScopedCache**（`instance-state.ts:38-60`）
```typescript
export const make = <A, E = never, R = never>(
  init: (ctx: InstanceContext) => Effect.Effect<A, E, R | Scope.Scope>,
) => Effect.gen(function* () {
  const cache = yield* ScopedCache.make<string, A, E, R>({
    capacity: Number.POSITIVE_INFINITY,    // 不限容量
    lookup: () => Effect.gen(function* () {
      return yield* init(yield* context)   // 用当前 instance ctx 初始化
    }),
  })
  const off = registerDisposer((directory) =>
    Effect.runPromise(ScopedCache.invalidate(cache, directory).pipe(Effect.provide(EffectLogger.layer))),
  )
  yield* Effect.addFinalizer(() => Effect.sync(off))
  return { [TypeId]: TypeId, cache }
})
```
- **per-instance key = directory**：`cd /Users/me/projA` 是一个 instance；切到 `/projB` 重建
- **容量无限**：因为每个 instance 通常只有 1 个 entry，OOM 不会发生
- **自动清理**：`registerDisposer` 把清理逻辑注册到全局，instance 切换时触发

### 2. **makeRuntime 单例**（`runtime.ts:18-28`）
```typescript
export function makeRuntime<I, S, E>(service: Context.Service<I, S>, layer: Layer.Layer<I, E>) {
  let rt: ManagedRuntime.ManagedRuntime<I, E> | undefined
  const getRuntime = () =>
    (rt ??= ManagedRuntime.make(Layer.provideMerge(layer, Observability.layer), { memoMap }))
  return {
    runSync: ..., runPromiseExit: ..., runPromise: ..., runFork: ..., runCallback: ...
  }
}
```
- **`rt ??=` 懒初始化**：第一次调用才创建 runtime
- **`Layer.provideMerge(layer, Observability.layer)`**：所有 runtime 自动获得 OTel 层
- **`{ memoMap }`**：effect v4 提供的 fiber 共享 memoMap，跨调用复用

### 3. **Effect ↔ 同步世界桥**（`run-service.ts:9-32`）
```typescript
export function attachWith<A, E, R>(effect: Effect.Effect<A, E, R>, refs: Refs): Effect.Effect<A, E, R> {
  if (!refs.instance && !refs.workspace) return effect
  if (!refs.instance) return effect.pipe(Effect.provideService(WorkspaceRef, refs.workspace))
  if (!refs.workspace) return effect.pipe(Effect.provideService(InstanceRef, refs.instance))
  return effect.pipe(
    Effect.provideService(InstanceRef, refs.instance),
    Effect.provideService(WorkspaceRef, refs.workspace),
  )
}
```
- **核心问题**：CLI 入口是 `yargs` callback（同步），但内部代码是 Effect.gen
- **解决**：在切到 Effect 前**捕获 Instance.current**（同步 API），attach 到 fiber context
- `WorkspaceContext.restore(...)` 模式：ctx-based scope 切换

### 4. **Effect Logger 桥**（`logger.ts`）
```typescript
export const logger = Logger.make((opts) => {
  const extra = clean(opts.fiber.getRef(References.CurrentLogAnnotations))
  const now = opts.date.getTime()
  for (const [key, start] of opts.fiber.getRef(References.CurrentLogSpans)) {
    extra[`logSpan.${key}`] = `${now - start}ms`
  }
  if (opts.cause.reasons.length > 0) extra.cause = Cause.pretty(opts.cause)
  const svc = typeof extra.service === "string" ? extra.service : undefined
  if (svc) delete extra.service
  const log = svc ? Log.create({ service: svc }) : Log.Default
  // ... format and write to log
})
```
- 把 Effect 的 structured log 翻译成内部 `util/Log` 格式
- 关键转换：`sessionID` → `session.id`（点号分隔的 OpenTelemetry 风格）

### 5. **DB PRAGMA 调优**（`db.ts:113-120`）
```typescript
db.run("PRAGMA journal_mode = WAL")            // 并发读 + 写不互锁
db.run("PRAGMA synchronous = NORMAL")          // WAL 模式下 NORMAL 即可（FULL 太慢）
db.run("PRAGMA busy_timeout = 5000")           // 5s 等待锁（默认 0 = 立刻失败）
db.run("PRAGMA cache_size = -64000")           // 64MB cache（负数 = KB）
db.run("PRAGMA foreign_keys = ON")             // 启用外键
db.run("PRAGMA wal_checkpoint(PASSIVE)")      // 启动时温和做一次 checkpoint
```

### 6. **Database.use 上下文注入**（`db.ts:148-180`）
```typescript
export function use<T>(callback: (trx: TxOrDb) => T): T {
  try {
    return callback(ctx.use().tx)  // 在事务中：用当前 tx
  } catch (err) {
    if (err instanceof LocalContext.NotFound) {
      const effects: (() => void | Promise<void>)[] = []
      const result = ctx.provide({ effects, tx: Client() }, () => callback(Client()))
      for (const effect of effects) effect()  // 同步触发"应该异步但被同步调用"的副作用
      return result
    }
    throw err
  }
}
```
- **LocalContext 模式**：用 sync throw NotFound 表示"上下文未设置"——比传 ctx 参数干净
- **`effects` 队列**：让 `Database.effect()` 排队的副作用在 use 完后同步触发——避免 magic promise

### 7. **Bundled vs Dev Migrations**（`db.ts:84-93`）
```typescript
declare const OPENCODE_MIGRATIONS: { sql: string; timestamp: number; name: string }[] | undefined

// Client() 内：
const entries = typeof OPENCODE_MIGRATIONS !== "undefined"
  ? OPENCODE_MIGRATIONS                  // 生产：Bun --bundle 注入的全局常量
  : migrations(path.join(import.meta.dirname, "../../migration"))   // dev：扫目录
```
- **生产模式**：用 `Bun.build({ define: { "OPENCODE_MIGRATIONS": "..." } })` 编译进二进制
- **dev 模式**：`readdirSync` + 解析 `20260323234822_events/` 目录名时间戳
- **跳过**：`Flag.MIMOCODE_SKIP_MIGRATIONS=true` 时所有 SQL 变 `select 1;`（测试用）

### 8. **34 个迁移时间线**（节选）
- `20260127222353_familiar_lady_ursula` — 最早一个（2026-01-27）
- `20260211171708_add_project_commands` — 加 project commands
- `20260225215848_workspace` — workspace 系统
- `20260312043431_session_message_cursor` — session message cursor
- `20260410174513_workspace-name` — workspace 改名
- `20260422160000_context_inheritance` — **子 agent contextMode 基础**
- `20260422170000_task_registry` — **任务系统上线**
- `20260515000000_actor_rename` — actor 改名
- `20260515010000_memory_fts` + `20260521010000_memory_fts_v6` — **FTS5 记忆搜索**
- `20260515020000_user_task` — user-level task
- `20260521000100_actor_registry_v6` — actor v6 schema
- `20260526000000_agent_id_main` — main session agent_id
- `20260527000000_actor_lifecycle` — actor lifecycle
- `20260527000100_inbox` — **Inbox 队列**
- `20260603000000_task_in_progress_owner` — task 状态机扩展
- `20260603000000_workflow_run` — workflow run 表
- `20260604000000_workflow_script_sha` — workflow 脚本 SHA
- `20260608000000_claude_import` — Claude Code 导入
- `20260609000000_history_fts` — **历史 FTS**
- `20260609230000_workflow_agent_timeout` — workflow agent timeout

### 9. **observability.ts：OpenTelemetry OTLP**
- 触发：`Flag.OTEL_EXPORTER_OTLP_ENDPOINT` 存在
- 资源属性：`service.name=opencode`、`service.version=InstallationVersion`、`deployment.environment.name=InstallationChannel`
- 运行时 trace：`@effect/opentelemetry/Tracer` + `OtlpLogger`

### 10. **bootstrap-runtime vs app-runtime**
- `bootstrap-runtime.ts`：DB 没初始化前用的 runtime（只包含 Console、Log 等无 DB 依赖的 Service）
- `app-runtime.ts`：DB 初始化后用的 runtime（包含 Config、Agent、Session 等所有 Service）
- **顺序保证**：先 bootstrap → init DB → migrate → 切到 app runtime

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 异步运行时 | 原生 Promise / async-await | **Effect v4** 全量 | 资源管理、并发、错误处理 |
| Service 注入 | class + singleton | **`Context.Service` + `Layer.effect`** | 声明式依赖、可测试 |
| DB 后端 | better-sqlite3 单后端 | **Bun SQLite + Node better-sqlite3 双后端** | import conditions |
| 迁移方式 | drizzle migrate 文件 | **bundled 模式**（生产）+ dev 目录模式 | Bun --bundle 注入常量 |
| Schema 数 | ~6 表 | **11 表 + FTS5 虚表** | session / message / part / todo / permission / share / account / control_account / project / workspace / workflow_run / history_fts |
| 迁移数 | < 10 | **34 个** | 6 个月密集开发 |
| PRAGMA | 默认 | **WAL + NORMAL + 5s busy + 64MB cache** | 高并发优化 |
| OpenTelemetry | 无 / 基础 | **完整 OTLP exporter** | 资源属性、log span、cause |
| Per-Instance 缓存 | 无 | **`ScopedCache<string, A>`** | 切 instance 自动失效 |

## 关键引用

- `packages/opencode/src/effect/instance-state.ts:30-60` — `make()` + `ScopedCache` 缓存
- `packages/opencode/src/effect/runtime.ts:18-28` — `makeRuntime` 单例
- `packages/opencode/src/effect/run-service.ts:9-32` — `attachWith` ctx 注入
- `packages/opencode/src/effect/logger.ts:20-50` — Effect Log → util Log 桥
- `packages/opencode/src/effect/observability.ts:1-50` — OTLP exporter 配置
- `packages/opencode/src/storage/db.ts:85-130` — PRAGMA + migration load
- `packages/opencode/src/storage/db.ts:148-180` — `use()` / `transaction()` ctx 注入
- `packages/opencode/src/storage/db.bun.ts:1-10` — Bun SQLite 极简 init
- `packages/opencode/migration/` — 34 个迁移（`ls` 验证）

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 的异步运行时用的是什么框架？和 OpenCode 有什么不同？
- [ ] `Context.Service` + `Layer.effect` 的"声明式依赖注入"模式怎么工作？
- [ ] `makeRuntime` 为什么用 `rt ??=` 懒初始化？什么时候才创建 runtime？
- [ ] 34 个迁移里哪几个是**自研特性**的奠基（actor_lifecycle、memory_fts、task_in_progress_owner、inbox、history_fts、workflow_run）？
- [ ] DB 的 PRAGMA 配置针对什么场景优化？busy_timeout=5000 是什么意思？
- [ ] `Database.use()` 的 `LocalContext.NotFound` + `effects` 队列模式解决了什么同步/异步混用问题？
- [ ] bootstrap-runtime 和 app-runtime 为什么要分开？
