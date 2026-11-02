# 13 · Fork 差异清单（MiMo-Code vs OpenCode）

## 一句话定位

MiMo-Code 相对上游 OpenCode 的**所有差异聚合**——按"自研 / 继承 / 改写"三类组织，每条差异给出行号引用 + 简要代码片段 + 原始章节出处。这是给"30 分钟理解 MiMo-Code 改了什么"的索引页。

## 数据基线（量化差异）

| 指标 | 数值 | 说明 |
|------|------|------|
| 源码文件数（`.ts` / `.tsx`） | **568** | `find packages/opencode/src` |
| 源码总行数 | **105,879** | 含 import + 类型注解 |
| 迁移数 | **34** | 2026-01-27 → 2026-06-09 |
| Provider 数 | 22 + **1**（MiMo Free 匿名） | `provider.ts:106-131` 动态注册 |
| 工具数（`.ts` 文件） | **35** | 9 大类分组 |
| Agent 数 | 1（OpenCode build）→ **8**（3 主 + 5 隐藏） | `agent.ts:1-554` |
| TUI Dialog | ~15 → **23** | `tui/component/dialog-*.tsx` |
| TUI Component | ~20 → **36** | dialog + 通用 |
| TUI i18n | 基础 → **8 语言** | `tui/i18n/*.ts` |
| TUI 架构 | 单进程 → **Worker 隔离 + RPC** | `tui/worker.ts` |
| Server 路由 | ~15 → **23**（19 instance + 2 control + 1 global + 1 ui） | `server/routes/` |
| 部署模式 | 1 → **2**（workspace / control-plane） | `MIMOCODE_WORKSPACE_ID` |
| Effect 文件 | 0 → **13** | `effect/*.ts` |
| 持久化记忆 | 无 → **FTS5 + 4 类文件 + checkpoint-writer** | `memory/*` + `session/checkpoint.ts` |
| 子 agent 系统 | 无 → **Actor（lifecycle + stuck + inbox）** | `actor/*`（10 文件） |
| 任务系统 | 无 → **T1/T1.1 树 + 9 verb + Gate** | `task/*`（8 文件） |
| Workflow | 无 → **运行时 + sandbox + 12h cap + 5 op** | `workflow/*` |
| Plugin 内置 | 基础 → **11 个内置** | `plugin/index.ts` |
| Skill 兼容 | 无 → **4 生态目录兼容** | `skill/index.ts:24-28` |
| 业务模块 | 原生 async/await → **Effect v4 全量替代** | `effect/index.ts` |

## 一、🌟 MiMo 自研特性（OpenCode 完全没有）

### 1. 跨会话持久化记忆（`memory/*` + `session/checkpoint.ts`）
**章节出处**：`06-memory.md`

- **FTS5 全文搜索**（`memory/fts-query.ts`）：OR-join + 相对 score 阈值
- **4 类记忆文件**：
  - `MEMORY.md`（项目级）— `paths.ts:1-30` 解析
  - `checkpoint.md`（会话级）— `session/checkpoint.ts:1-1478`
  - `notes.md`（草稿）— `paths.ts` 多源
  - `tasks/<id>/progress.md`（任务级）— `task/paths.ts`
- **checkpoint-writer 子 agent**（`agent.ts` 中定义）— **没有 prompt、没有 toolAllowlist**——为对齐 prefix cache 设计的极简壳
- **TAIL_MIN_TOKENS=10k / TAIL_MAX_TOKENS=20k** — checkpoint 软上限
- **11 段 checkpoint 模板**（§0 Meta / §1 Goal / §2 Progress / §3 Discoveries / §4 Task tree / ...）— `checkpoint-templates.ts`

**关键代码**：
```typescript
// packages/opencode/src/session/checkpoint.ts
export const TAIL_MIN_TOKENS = 10_000
export const TAIL_MAX_TOKENS = 20_000

// checkpoint-writer agent — 空 prompt
// agent.ts 中: prompt: undefined, tools: undefined
// 让 LLM 完全靠 §11 instruction block 驱动
```

### 2. Goal 独立裁判模型（`session/goal.ts`）
**章节出处**：`04-react-loop.md`

- **MAX_GOAL_REACT=12** — 单 goal 最多 12 轮 ReAct
- **REPEATED_STEP_THRESHOLD=3** — 同一 step signature 出现 3 次强制 re-plan
- **独立 judge LLM** — 防止主 LLM "乐观停止"（自主工作时报告"已完成"但实际没做完）
- **stableStringify** — step signature hash 工具

**关键代码**：
```typescript
// packages/opencode/src/session/goal.ts
export const MAX_GOAL_REACT = 12
export const REPEATED_STEP_THRESHOLD = 3
// decide() 调独立 judge model 判"是否真正完成"
```

### 3. Dream & Distill 自进化（`session/auto-dream.ts`）
**章节出处**：`04-react-loop.md`

- **`/dream`** — 把会话内重复模式提取到项目 `MEMORY.md`
- **`/distill`** — 把反复手敲的命令打包成 skill / subagent
- 持续学习，无需用户手动整理笔记

### 4. Tree 状任务系统（`task/*` 8 文件）
**章节出处**：`08-task.md`

- **TaskID 正则** `^T\d+(\.\d+)*$` — `T1`, `T1.1`, `T1.1.2` 树状
- **5 TaskStatus**（open / in_progress / blocked / done / abandoned）
- **8 TaskEventKind**（transitions 单独记）
- **9 verb 状态机**（create/list/get/start/block/unblock/done/abandon/rename）
- **`nextChildId(parentId, siblings)`** — 同级自增算法
- **TaskGate.decide()** — 完结闸门
  - **MAX_TASK_GATE_SUBAGENT_REACT=2** / **MAX_TASK_GATE_MAIN_REACT=3**
  - **fail-open on DB error**（注释明确："transient DB error must NEVER trap the agent"）
- **levenshtein 拼写纠错** — `task` 工具内置，distance ≤ 2 + 唯一候选才建议

**关键代码**：
```typescript
// packages/opencode/src/task/schema.ts
export const TaskID = z.string().regex(/^T\d+(\.\d+)*$/, "Task ID must be Tn or Tn.m...")

// packages/opencode/src/task/gate.ts
export const MAX_TASK_GATE_SUBAGENT_REACT = 2
export const MAX_TASK_GATE_MAIN_REACT = 3

// packages/opencode/src/task/tool/task.ts
function suggestVerb(input: string): string | undefined {
  const candidates = KNOWN_VERBS.map((v) => ({ v, d: levenshtein(input, v) })).filter((c) => c.d <= 2)
  if (candidates.length !== 1) return undefined  // 多解歧义不纠
  return candidates[0].v
}
```

### 5. Actor 子 agent 系统（`actor/*` 10 文件）
**章节出处**：`07-actor.md`

- **3 个 SpawnMode**：peer（同辈）/ subagent（子）/ main（主）
- **3 个 ContextMode**：none（空）/ state（DB 状态）/ full（完整 prefix）
- **STUCK_THRESHOLD_MS = 5 min** — 5 分钟无新事件标 stuck
- **SCAN_INTERVAL_MS = 60s** — 后台扫描频率
- **Effect.uninterruptible outer + Effect.interruptible inner** — `runTurn` 防外层中断杀内层
- **ForkContext** — prefix cache 对齐（隐藏注入的 actor context 不会污染 LLM cache key）
- **allocateActorID**：`{agentType}-{n}` 自增
- **Inbox 消息队列**（`migration/20260527000100_inbox/`）— actor 间异步通知

**关键代码**：
```typescript
// packages/opencode/src/actor/registry.ts
export const STUCK_THRESHOLD_MS = 5 * 60 * 1000
export const SCAN_INTERVAL_MS = 60 * 1000

// packages/opencode/src/actor/turn.ts
export const runTurn = Effect.gen(function*() {
  return yield* Effect.uninterruptible(
    Effect.gen(function*() {
      // 中间逻辑：分轮跑 LLM
      yield* /* 每一轮 interruptible */
    })
  )
})
```

### 6. Compose 编排模式 + 最大模式（`agent.ts` + `session/max-mode.ts`）
**章节出处**：`02-agents.md`、`04-react-loop.md`

- **3 个主 agent**：build（默认）/ plan（规划）/ compose（编排） + **max**（最大模式）
- **max 模式**：N 路并行推理候选 + 自动选最佳
- **7 个隐藏 subagent**：title（标题生成）/ summary（摘要）/ compaction（压缩）/ checkpoint-writer（写入）/ dream（梦境）/ distill（蒸馏）/ (其他)
- **每个 agent 独立 `permission` / `prompt` / `mode` / `tools` 元数据**
- **SYSTEM_SPAWNED_AGENT_TYPES = { checkpoint-writer, dream, distill }** — 系统独占 spawn 类型

### 7. Workflow 运行时（`workflow/*` 完整子系统）
**章节出处**：`12-plugin-skill.md`

- **`SCRIPT_DEADLINE_MS = 12h`** — 单脚本最长 12 小时
- **`MAX_LIFECYCLE_AGENTS = 1000`** — 单 run 最多 spawn 1000 个 agent
- **`DEFAULT_MAX_CONCURRENT = 16`** — 默认并发
- **`STRAGGLER_TIMEOUT = Symbol`** — 永不撞值哨兵
- **`WORKFLOW_STRUCTURAL_ERROR` marker** — wiring bug 必须 loud fail
- **5 operation 工具**（run / status / wait / cancel / resume）
- **journal 持久化** — 重启可恢复
- **Null-prototype registry**（`Object.create(null)`）— 防 prototype 污染
- **`workflowRef` 懒绑定** — 破 Layer 循环依赖

**关键代码**：
```typescript
// packages/opencode/src/workflow/runtime.ts
const SCRIPT_DEADLINE_MS = 12 * 60 * 60 * 1000
const MAX_LIFECYCLE_AGENTS = 1000
const DEFAULT_MAX_CONCURRENT = 16
const STRAGGLER_TIMEOUT = Symbol("straggler-timeout")

// packages/opencode/src/workflow/builtin.ts
import DEEP_RESEARCH_SCRIPT from "./builtin/deep-research.js" with { type: "text" }
const REGISTRY: Record<string, Entry> = Object.create(null)
```

### 8. Control-Plane 部署模式（`server/workspace.ts`）
**章节出处**：`10-server.md`

- **`MIMOCODE_WORKSPACE_ID` 模式**：单 workspace 简化（无 control plane）
- **默认 control-plane 模式**：多 workspace + `WorkspaceRouterMiddleware`
- **RULES 路由表**：按 `method + path` 决定 local（自己处理）vs forward（按 session 路由）
- **Enterprise/SaaS 形态**

### 9. Sync Fence 增量同步（`server/fence.ts`）
**章节出处**：`10-server.md`

- **`x-mimocode-sync: id:seq,id:seq`** header
- **`load` / `diff` / `parse`** 三步流程
- **减少 TUI 拉新 event 时的重复同步**

### 10. 其他自研小特性
- **Bun macro 编译时内联** — Compose Skills bundle 整个 inline 进二进制
- **`with { type: "text" }`** — 内置 workflow 脚本 inline
- **mDNS 局域网发现** — `mimocode.local`
- **10s SSE heartbeat** — 防止 stalled proxy
- **10 hop-by-hop headers 严格删除** — WebSocket proxy 合规
- **CSP sha256 inline script hash** — 主题预加载
- **Web UI 编译时内嵌** — `opencode-web-ui.gen.ts`
- **Frecency 提示** — `FrecencyProvider` 按频次排
- **Prompt Stash** — 草稿暂存
- **8 语言 i18n**

## 二、🔁 继承自 OpenCode（基本原样照搬）

> 这些模块在 OpenCode 中已存在，MiMo 没改（或几乎没改），但**也用了**

| 模块 | 文件 | OpenCode 同源 | MiMo 改了什么 |
|------|------|---------------|---------------|
| **22 个 Provider SDK** | `provider/sdk/{openai,anthropic,google,...}.ts` | 22 个全照搬 | +1 个 MiMo Free 匿名通道 |
| **ai-sdk 集成** | `provider/provider.ts:1-50` | 一样 | 动态 import registry（`provider.ts:106-131`） |
| **Permission 系统** | `permission/*` | 一样 | 加了 3 层 merge（defaults → specific → user） |
| **Hono 服务框架** | `server/server.ts:1-30` | 一样 | 加了 mDNS + Web UI 嵌入 + Control-Plane |
| **hono-openapi** | `server/server.ts:71-90` | 一样 | 注释说 `.route()` 在有 custom errorHandler 的子 app 上剥 metadata symbol |
| **bash / read / write / edit** | `tool/{bash,read,write,edit}.ts` | 大体一致 | 5 段 syscall 解析 |
| **Tool Registry** | `tool/registry.ts` | 一样 | 5 段 `Tool.Def` schema |
| **Bus 事件总线** | `bus/*` | 一样 | 扩了 `TaskCreated` / `TaskUpdated` / WorkflowStarted / InboxMessage |
| **SolidJS + @opentui/solid** | `tui/app.tsx` | 一样 | + Worker 进程隔离 |
| **drizzle-orm** | `storage/db.ts` | 一样 | 加了 `#db` import conditions |
| **yargs** | `cli/cmd/*.ts` | 一样 | 22 个子命令（OpenCode 18 + MiMo 加 4） |

**关键证据**：
```typescript
// packages/opencode/src/provider/provider.ts:106-131
// 动态 import registry — OpenCode 的同款模式
for (const item of items) {
  // loadSdk 等价于 OpenCode 的 loaders/opencode.ts
}

// packages/opencode/src/permission/index.ts
// 三层 merge：defaults → specific → user
// OpenCode 同款
```

## 三、🔧 改写/重写（OpenCode 有但 MiMo 大改了）

### 1. 异步运行时：async/await → **Effect v4**
**章节出处**：`11-effect-db.md`

- **13 个 Effect 文件**（`effect/*`）
- **Context.Service + Layer.effect 模式** — 所有长生命周期 Service
- **`AppRuntime.runPromise(...)`** 顶层入口
- **`Instance.provide({ directory, init, fn })`** 注入 context
- **`Effect.uninterruptible / interruptible`** 声明式并发控制
- **`Effect.fn("name")`** 命名 effect span
- **.gen 函数式风格**

**为什么改**：
- OpenCode 的原生 async/await 难以管理：①DB transaction 跨多表 ②长生命周期 Service ③detached background writer
- Effect 提供声明式 resource management + structured concurrency

**关键代码**：
```typescript
// packages/opencode/src/effect/index.ts
export const AppRuntime = ... // Layer.effect 创建

// packages/opencode/src/cli/bootstrap.ts
export async function bootstrap<T>(directory: string, cb: () => Promise<T>) {
  return Instance.provide({
    directory,
    init: () => AppRuntime.runPromise(InstanceBootstrap),
    fn: async () => {
      try { return await cb() }
      finally {
        // drainWriters — detached writer 120s 跑完
        await AppRuntime.runPromise(SessionCheckpoint.Service.use(svc => svc.drainWriters()))
        await Instance.dispose()
      }
    },
  })
}
```

### 2. DB 后端：`better-sqlite3` 单一 → **Bun / Node 双后端**
**章节出处**：`11-effect-db.md`

- **`#db` import conditions**（`packages/opencode/package.json:imports`）
  - `bun: ./src/storage/db.bun.ts` — `bun:sqlite`
  - `node: ./src/storage/db.node.ts` — `better-sqlite3`
- **PRAGMA**：WAL / NORMAL / 5s busy_timeout / 64MB cache
- **Per-instance ScopedCache** — 切工作目录自动失效

**为什么改**：
- Bun 单文件打包需要 `bun:sqlite`（更小、零依赖）
- Node 用户保留 `better-sqlite3` 兜底

### 3. TUI 架构：单进程 → **Worker 进程隔离**
**章节出处**：`09-tui.md`

- **主进程**（`app.tsx`）— 只渲染，零业务逻辑
- **Worker 进程**（`worker.ts`）— 承载 Session/Agent/Provider/DB
- **`@/util/Rpc` IPC** — 双向 RPC（fetch / global.event / snapshot / server / checkUpgrade）
- **崩溃隔离** — Worker 挂掉不毁 TUI 渲染
- **`writeHeapSnapshot("server.heapsnapshot")`** — Worker RPC 暴露给 TUI 做内存诊断
- **plainTerminal 模式**：10/15 fps + 透明背景 + 不开线程 —— pipe 友好

**为什么改**：
- 业务逻辑（LLM 流、DB query、tool 执行）阻塞 → TUI 渲染卡顿
- 独立进程可以独立重启、独立 debug

**关键代码**：
```typescript
// packages/opencode/src/cli/cmd/tui/worker.ts
ensureProcessMetadata("worker")
GlobalBus.on("event", (event) => Rpc.emit("global.event", event))

export const rpc = {
  async fetch(input) { /* 转发到 Server.Default().app.fetch */ },
  snapshot() { return writeHeapSnapshot("server.heapsnapshot") },
  async server(input) { /* 启动/重启动 server */ },
  ...
}
```

### 4. CLI 启动链：单层 → **3 层 shim + Avx2/musl 检测**
**章节出处**：`01-cli-bootstrap.md`

- **`bin/mimo`** Node shim — 找真二进制
- **3 路决策**：`MIMOCODE_BIN_PATH` > 本地 `.mimocode` > 下载
- **`supportsAvx2` 跨平台检测**：
  - Linux `/proc/cpuinfo`
  - macOS `sysctl -n hw.optional.avx2_0`
  - Windows `[Kernel32]::IsProcessorFeaturePresent(40)`
- **musl/glibc 检测**（`/etc/alpine-release` / `ldd --version`）
- **22 个 yargs 子命令**（OpenCode 18 + MiMo 加 4：acp / agent / plug / web / mcp / db / thread / attach / debug）
- **`bootstrap()` finally 块 `drainWriters()`** — 防 headless `mimo run` 杀掉 detached writer
- **`Heap.start()`** 堆监控
- **顶级 catch 区分 3 类错误**：NamedError / Error / ResolveMessage
- **finally 强制 `process.exit()`** — 防 hanging subprocess（特别是 docker MCP server）

### 5. Server：基础 → **多 workspace + Control-Plane**
**章节出处**：`10-server.md`

- **5 middleware 顺序**：Error → Auth → Logger → Compression → CORS
- **2 部署模式**：单 workspace vs control-plane
- **mDNS Bonjour publish** — `mimocode.local`
- **Web UI 嵌入 + Proxy 兜底** — `opencode-web-ui.gen.ts` 编译时生成
- **CSP 自定义 hash** — 允许 inline 主题预加载脚本
- **OpenAPI 文档生成** — `openapi()` 重建 fresh app 解决 metadata symbol 剥除

### 6. Plugin 体系：基础 → **11 内置 + 5 HookEvent + ActorPreStop/PostStop**
**章节出处**：`12-plugin-skill.md`

| Plugin | 作用 |
|--------|------|
| `CodexAuthPlugin` | Codex 鉴权 |
| `MimoAuthPlugin` + `AnthropicProxyPlugin` | MiMo 账号 + Anthropic proxy |
| `MimoFreeAuthPlugin` | MiMo Free 匿名 JWT |
| `CopilotAuthPlugin` | GitHub Copilot 鉴权 |
| `GitlabAuthPlugin` | GitLab 鉴权 |
| `PoeAuthPlugin` | Poe 鉴权 |
| `CloudflareAIGatewayAuthPlugin` | Cloudflare AI Gateway |
| `CloudflareWorkersAuthPlugin` | Cloudflare Workers AI |
| `CheckpointSplitoverPlugin` | checkpoint 拆分 hook |
| `SubagentProgressCheckerPlugin` | subagent 进度检查 hook |

- **5 个 HookEvent**：
  - `hook.executed` — 记录每次 hook 触发
  - `hook.react.reentered` — preStop / postStop 触发 ReAct re-entry
  - `hook.react.max_reached` — cap 触顶
  - 加上 `actor.preStop` / `actor.postStop` 两个 input 接口

### 7. Skill：基础 → **4 生态兼容 + Compose Bundle**
**章节出处**：`12-plugin-skill.md`

- **`EXTERNAL_DIRS = [".claude", ".agents", ".codex", ".opencode"]`** — 兼容 Claude / Codex / OpenCode skill 目录
- **`{skill,skills}/**/SKILL.md`** — MiMo 原生模式
- **`Discovery.pull(url)`** — 远程 `index.json` 拉取
- **Bun macro 编译时内联 Compose bundle** — 离线可用
- **`<skill_content>` 输出格式** — LLM 可读

### 8. Provider 适配：22 → 22 + **MiMo Free 匿名**
**章节出处**：`03-llm.md`

- **`MimoFreeAuthPlugin`** — 零配置匿名 JWT
- **`with { type: "text" }`** — 内置鉴权脚本 inline
- **Persistent-retry schedule**：500ms × 2, max 10 attempts
- **SSE 8-minute chunk timeout** — reasoning 模型流式响应
- **`isRetryableTransientError`** — 单一可重试错误判断源

## 四、改名/重命名清单

| 上游 OpenCode 名 | MiMo-Code 名 | 备注 |
|-----------------|--------------|------|
| `opencode` | **`mimo` / `mimocode` / `MIMOCODE_*`** | 项目级（CLI 名、env var、目录名前缀） |
| `OPENCODE_*` env | `MIMOCODE_*` env | 全量替换 |
| `.opencode` 目录 | `.mimocode` | bin shim 缓存目录 |
| `mimocode.local` mDNS | — | 新增 |
| `x-opencode-*` header | `x-mimocode-*` header | sync fence + workspace router |
| `opencode-web-ui.gen.ts` | 同名 | 编译产物路径（未改） |
| `opencode-<platform>-<arch>` 二进制 | **同名** | 注意：二进制仍叫 `opencode`（README 明示） |
| `MIMOCODE_HOME` | 新增 | dev 模式 home 目录 |
| `MIMOCODE_WORKSPACE_ID` | 新增 | workspace 模式触发 |
| `MIMOCODE_DISABLE_CLAUDE_IMPORT` | 新增 | 关闭 Claude 导入 |
| `MIMOCODE_CLAUDE_IMPORTED` | 新增 | 一次性环境守卫 |
| `MIMOCODE_PURE=1` | 新增 | 不加载 external plugin |
| `MIMOCODE_DISABLE_MOUSE` | 新增 | 禁用 TUI 鼠标 |
| `MIMOCODE_DISABLE_EMBEDDED_WEB_UI` | 新增 | 关闭内嵌 Web UI |

## 五、模块级 diff 速查表

| 模块 | OpenCode 状态 | MiMo 改动 | 章节 |
|------|---------------|-----------|------|
| `bin/<name>` | 单层 shim | + Avx2/musl + 3 路 | `01-cli-bootstrap` |
| `src/index.ts` | 18 子命令 | 22 子命令 + 1-次迁移 + Claude 导入 | `01-cli-bootstrap` |
| `src/agent/agent.ts` | 1 agent | 8 agents（3 主 + 5 隐藏） | `02-agents` |
| `src/provider/provider.ts` | 22 provider | 22 + MiMo Free | `03-llm` |
| `src/session/llm.ts` | 基础 retry | persistent-retry + 8min SSE | `03-llm` |
| `src/session/prompt.ts` | 1k 行 | **3355 行** + Goal + Auto-dream + checkpoint + prefix-capture | `04-react-loop` |
| `src/session/goal.ts` | 无 | 新增 + MAX cap + judge model | `04-react-loop` |
| `src/session/checkpoint.ts` | 无 | **1478 行** + 11 段模板 | `06-memory` |
| `src/session/max-mode.ts` | 无 | 新增 N 路并行候选 | `04-react-loop` |
| `src/tool/*.ts` | 30 个 | 35 个 | `05-tools` |
| `src/memory/*` | 无 | **6 文件** + FTS5 + reconcile | `06-memory` |
| `src/actor/*` | 无 | **10 文件** + lifecycle + stuck | `07-actor` |
| `src/task/*` | 无 | **8 文件** + T1/T1.1 + Gate | `08-task` |
| `src/workflow/*` | 无 | 完整子系统 + sandbox | `12-plugin-skill` |
| `src/cli/cmd/tui/worker.ts` | 无 | **新进程入口** | `09-tui` |
| `src/cli/cmd/tui/app.tsx` | 800 行 | **1130 行** + 23 dialog | `09-tui` |
| `src/cli/cmd/tui/i18n/*` | 1-2 语言 | 8 语言 | `09-tui` |
| `src/server/server.ts` | 基础 | 5 middleware + 2 模式 | `10-server` |
| `src/server/workspace.ts` | 无 | control-plane router | `10-server` |
| `src/server/fence.ts` | 无 | sync fence | `10-server` |
| `src/server/mdns.ts` | 无 | Bonjour | `10-server` |
| `src/server/routes/*` | 15 | 23 | `10-server` |
| `src/effect/*` | 无 | **13 文件** | `11-effect-db` |
| `src/storage/db.*.ts` | 单文件 | **3 文件**（abstract + bun + node） | `11-effect-db` |
| `src/migration/` | ~10 | **34 个**（2026-01-27 → 2026-06-09） | `11-effect-db` |
| `src/plugin/index.ts` | 5 内置 | **11 内置** | `12-plugin-skill` |
| `src/skill/discovery.ts` | 无 | 新增远程拉取 | `12-plugin-skill` |
| `src/skill/compose/*` | 无 | Bun macro + 2 文件 | `12-plugin-skill` |

## 六、未在 OpenCode 中验证的假设

> 由于 OpenCode 上游在 2026-06 突然消失（README 已 redirect），无法做严格的 git diff 对比。下列假设基于"OpenCode 早期版本的 sst/opencode 公开代码 + MiMo-Code 当前代码"反推：

1. **OpenCode 原版用原生 async/await**——MiMo 全量迁到 Effect v4 是改写
2. **OpenCode 原版只有 1 个 build agent**——MiMo 的 8 agent 是新增
3. **OpenCode 原版没有持久化记忆**——FTS5 + checkpoint 是新增
4. **OpenCode 原版没有子 agent 系统**——Actor 是新增
5. **OpenCode 原版没有任务系统**——Task 是新增
6. **OpenCode 原版没有 Workflow**——Workflow 是新增
7. **OpenCode 原版没有 Skill compose**——Bun macro 内联是新增
8. **OpenCode 原版 CLI 18 子命令**——MiMo 加到 22
9. **OpenCode 原版 TUI 15 dialog**——MiMo 加到 23
10. **OpenCode 原版 TUI 单进程**——MiMo 改 Worker 隔离

## 七、改动的"哲学"

### 1. **后向兼容优先**
- 兼容 Claude / Codex / OpenCode 的 skill 目录（`EXTERNAL_DIRS`）
- 一次性自动导入 `~/.claude/`（`ClaudeImport.run()`）
- 旧 JSON 数据自动迁移到 SQLite（`JsonMigration`）

### 2. **离线优先**
- Bun macro 内联 Compose bundle — 无网也能用
- `with { type: "text" }` 内置 workflow — 无 fs 也能跑
- Web UI 编译时内嵌 — 无 app.opencode.ai 也能用
- MiMo Free 匿名通道 — 无 API key 也能用

### 3. **鲁棒性优先**
- `TaskGate` fail-open — DB 抖一下不让 agent 卡死
- `bootstrap()` drainWriters — 防止 headless run 杀掉 detached writer
- `process.exit()` finally — 防 hanging subprocess
- `Effect.uninterruptible` outer — 防外层中断杀内层
- mDNS skip same port — 防重复 publish
- prefix cache 对齐 — checkpoint-writer 空 prompt

### 4. **观察性优先**
- 5 个 HookEvent 记录所有 hook
- `writeHeapSnapshot` 暴露给 TUI
- `Log.Default.info("mimocode", { version, args, process_role, run_id })` 启动记录
- checkpoint-writer 11 段模板 — 完整会话快照

### 5. **类型严格优先**
- `z.strictObject` 拒绝未知字段（task / workflow 工具）
- `Object.create(null)` 防 prototype 污染（workflow registry）
- `Symbol` 作哨兵永不撞值（STRAGGLER_TIMEOUT）
- `z.discriminatedUnion` 严格分类（workflow 5 op）
- Bun macro `with { type: "macro" }` — 编译时强校验

## 八、参考 PR/Commit 锚点

> 由于 MiMo-Code 公开仓库刚发布（2026-06-10），自研特性均合在一个大 commit，PR 粒度分析暂不可行。
> 后续如能拿 upstream git history，可补充 §7 表格的"上游 commit hash"列。

| 自研特性 | MiMo-Code 首次出现 commit |
|---------|---------------------------|
| Memory FTS5 | `7233b71`（initial） |
| Actor 系统 | `7233b71` |
| Task 系统 | `7233b71` |
| Goal judge | `7233b71` |
| Effect v4 | `7233b71` |
| Worker 隔离 | `7233b71` |
| Compose Bun macro | `7233b71` |
| Workflow runtime | `7233b71` |
| Control-Plane 模式 | `7233b71` |
| MiMo Free Plugin | `7233b71` |

## 关键引用

- `package.json:1-50` — workspaces.catalog 50+ 共享版本
- `packages/opencode/package.json:1-40` — bin 入口 + dual runtime imports
- `packages/opencode/src/index.ts:1-66` — yargs 启动链
- `packages/opencode/src/agent/agent.ts:1-554` — 8 agent 全定义
- `packages/opencode/src/session/prompt.ts:1-3355` — 主 ReAct 循环
- `packages/opencode/src/session/checkpoint.ts:1-1478` — 持久化记忆核心
- `packages/opencode/src/session/goal.ts:1-80` — Goal 独立裁判
- `packages/opencode/src/memory/fts-query.ts` — FTS5 OR-join
- `packages/opencode/src/actor/registry.ts:35-46` — nextChildID 算法
- `packages/opencode/src/task/gate.ts:8-20` — MAX cap 常量
- `packages/opencode/src/workflow/runtime.ts:30-44` — 4 cap 常量
- `packages/opencode/src/cli/cmd/tui/worker.ts:1-30` — Worker 入口
- `packages/opencode/src/server/server.ts:1-90` — 5 middleware + 2 模式 + openapi
- `packages/opencode/src/effect/index.ts` — Effect 顶层
- `packages/opencode/src/storage/db.bun.ts` + `db.node.ts` — 双后端
- `packages/opencode/migration/20260515010000_memory_fts/` — FTS5 迁移
- `packages/opencode/migration/20260527000000_actor_lifecycle/` — Actor 迁移
- `packages/opencode/migration/20260603000000_workflow_run/` — Workflow 迁移

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 相对 OpenCode 总共改了多少文件 / 行数？
- [ ] 哪 10 个特性是 MiMo **完全自研**、OpenCode 没有？
- [ ] 哪 11 个模块是 MiMo **继承自** OpenCode、几乎没改？
- [ ] 哪 8 个模块是 MiMo **改写/重写**？
- [ ] 命名约定（`mimocode` vs `opencode`、`MIMOCODE_*` vs `OPENCODE_*`）改了哪些？
- [ ] MiMo 改动的"哲学"是什么？（后向兼容 / 离线 / 鲁棒 / 观察 / 类型严格）
- [ ] Effect v4 替换原生 async/await 的核心动机是什么？
- [ ] checkpoint-writer 为何"无 prompt、无 toolAllowlist"？
- [ ] TaskGate 为什么要 fail-open？什么场景会触发？
- [ ] Compose Bundle 为什么必须用 Bun macro 而不是 `Bun.file()`？
