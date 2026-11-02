# 01 · 启动链 + CLI 框架

## 一句话定位
MiMo-Code 的启动链分**三层 shim**：`bin/mimo` (Node) → 下载/缓存的预编译二进制（Bun 单文件）→ `src/index.ts`（yargs 22 个子命令）；yargs `.command()` 注册 + `.fail()` 优雅处理未知参数 + `Heap.start()` 启用堆监控 + `JsonMigration` 老 JSON → SQLite 一次性迁移 + `ClaudeImport` 自动从 `~/.claude/` 导入数据。

## 涉及文件
- `packages/opencode/bin/mimo` (120+ 行) — **npm bin shim**（Node 脚本，spawnSync 真二进制）
- `packages/opencode/src/index.ts` (335+ 行) — CLI 唯一入口（yargs 注册 22 个子命令）
- `packages/opencode/src/cli/cmd/cmd.ts` — `cmd({ command, describe, builder, handler })` 工厂
- `packages/opencode/src/cli/bootstrap.ts` — `bootstrap<T>(directory, cb)` Effect ctx 包装
- `packages/opencode/src/cli/heap.ts` — `Heap.start()` 堆监控
- `packages/opencode/src/cli/ui.ts` — UI 输出辅助
- `packages/opencode/src/cli/error.ts` — `FormatError` 错误格式化
- `packages/opencode/src/cli/heap.ts` — 堆分析
- 22 个 yargs 子命令（`packages/opencode/src/cli/cmd/*.ts`）

## 22 个 yargs 子命令清单

| Command | 用途 | 文件 |
|---------|------|------|
| `acp` | Agent Communication Protocol 适配 | `acp.ts` |
| `agent` | agent CRUD（create/list/permission） | `agent.ts` |
| `attach` | 附加到运行中的 TUI | `tui/attach.ts` |
| `completion` | 生成 shell 补全脚本 | yargs built-in |
| `console` | 控制台登录/账号管理 | `account.ts` |
| `db` | 数据库维护（migrate/status） | `db.ts` |
| `debug` | 调试（file / lsp / ripgrep / scrap / skill / snapshot） | `debug/*.ts` |
| `export` | 导出会话 | `export.ts` |
| `generate` | 跑 agent 生成器（`/general`） | `generate.ts` |
| `github` | GitHub 集成（PR/Issue） | `github.ts` |
| `import` | 导入会话 | `import.ts` |
| `mcp` | MCP server 列表/管理 | `mcp.ts` |
| `models` | 列模型 | `models.ts` |
| `plug` | 插件管理（install/list/remove） | `plug.ts` |
| `pr` | Pull Request 操作 | `pr.ts` |
| `providers` | 列 provider | `providers.ts` |
| `run` | headless 跑 agent（核心入口） | `run.ts` |
| `serve` | 启动 HTTP server | `serve.ts` |
| `session` | session 列表/CRUD | `session.ts` |
| `stats` | 统计 | `stats.ts` |
| `thread` | 启动 TUI 新线程 | `tui/thread.ts` |
| `tui` | 启动 TUI（默认） | `tui/*` |
| `uninstall` | 卸载 | `uninstall.ts` |
| `upgrade` | 升级 | `upgrade.ts` |
| `web` | 启动 web UI | `web.ts` |

## 核心数据流

### 启动链 3 层
```
$ mimo run "fix bug"   # 用户
   ↓
1. bin/mimo（Node shim）— 找真二进制
   ├─ MIMOCODE_BIN_PATH 环境变量
   ├─ 同目录 .mimocode（缓存的预编译）
   └─ 否则按 platform/arch/avx2 下载到 .mimocode
       darwin-arm64 / darwin-x64 / linux-x64 / linux-arm64 / windows-x64
       musl / glibc 检测（linux）
   ↓
2. 真二进制（mimo，单 ~80MB）— 跑 src/index.ts
   ├─ 解析 yargs
   ├─ middleware：Log.init / pure mode
   ├─ Heap.start()
   ├─ 设环境变量 AGENT=1 MIMOCODE=1 MIMOCODE_PID
   ├─ Log.Default.info("mimocode", { version, args, process_role, run_id })
   ├─ 一次性 DB 迁移（如果 marker 不存在）
   ├─ ClaudeImport.run()（一次性）
   └─ dispatch 到 cli/cmd/<subcommand>.ts
   ↓
3. cli/cmd/run.ts（例）
   ├─ AppRuntime.runPromise(...)
   ├─ bootstrap(directory, () => SessionPrompt.run(...))
   │    ├─ Instance.provide({ directory, init: () => AppRuntime.runPromise(InstanceBootstrap) })
   │    └─ finally: drainWriters() + Instance.dispose()
   └─ 输出结果
```

### bin/mimo 决策
```
MIMOCODE_BIN_PATH 存在
  → run(MIMOCODE_BIN_PATH)              # 自定义路径（debug 用）
否则 scriptDir/.mimocode 存在
  → run(.mimocode)                       # 已缓存的预编译
否则
  → 拼 base = "opencode-" + platform + "-" + arch
  → binary = "opencode.exe" (windows) | "opencode"
  → supportsAvx2() 检测
  → 检测 musl/glibc
  → 从 release URL 下载 → 解压 → 落到 .mimocode
  → run(.mimocode)
```

## 关键设计

### 1. **bin/mimo 的 3 路 shim**（已读前 120 行）
```javascript
const envPath = process.env.MIMOCODE_BIN_PATH
if (envPath) run(envPath)

const cached = path.join(scriptDir, ".mimocode")
if (fs.existsSync(cached)) run(cached)

const platformMap = { darwin, linux, win32 }
const archMap = { x64, arm64, arm }
const base = "opencode-" + platform + "-" + arch
const binary = platform === "windows" ? "opencode.exe" : "opencode"
```
- **优先级**：环境变量 > 本地缓存 > 下载
- 跨平台：darwin / linux / windows × x64 / arm64 / arm
- **musl/glibc 检测**（`/etc/alpine-release` / `ldd --version` 输出）—— Alpine 用 musl build

### 2. **supportsAvx2 跨平台检测**（bin/mimo:50-90）
- Linux：`/proc/cpuinfo` 搜 `avx2`
- macOS：`sysctl -n hw.optional.avx2_0` → `"1"`
- Windows：PowerShell `[Kernel32]::IsProcessorFeaturePresent(40)` —— 40 是 AVX2 constant
- **avx2 不支持的 x64 走 baseline build** ——兼容性兜底

### 3. **一次性 DB 迁移 UI**（index.ts:120-160）
```typescript
const marker = path.join(Global.Path.data, "mimocode.db")
if (!(await Filesystem.exists(marker))) {
  process.stderr.write("Performing one time database migration, may take a few minutes..." + EOL)
  const width = 36
  const orange = "\x1b[38;5;214m"
  const muted = "\x1b[0;2m"
  // ... 进度条
  await JsonMigration.run(drizzle({ client: Database.Client().$client }), {
    progress: (event) => {
      const percent = Math.floor((event.current / event.total) * 100)
      // TTY 模式：彩色进度条
      // 非 TTY 模式：`sqlite-migration:50` 单行（可解析）
    },
  })
}
```
- **marker 文件 = 旧 JSON 存储存在**——存在说明需要从 JSON → SQLite 迁移
- 进度条**只在 tty 显示**；pipe / log 收集场景用 `sqlite-migration:50` 单行格式

### 4. **Claude Code 一次性导入**（index.ts:175-184）
```typescript
if (!process.env.MIMOCODE_DISABLE_CLAUDE_IMPORT && !process.env.MIMOCODE_CLAUDE_IMPORTED) {
  process.env.MIMOCODE_CLAUDE_IMPORTED = "1"
  try { await ClaudeImport.run() }
  catch (e) { Log.Default.warn("claude-import failed", { e: errorMessage(e) }) }
}
```
- **环境守卫**：子进程继承 `MIMOCODE_CLAUDE_IMPORTED=1` 不会重跑
- **best-effort**：失败只 warn，不阻塞启动
- 数据来源：`~/.claude/projects/<slug>/memory/**/*.md` + `~/.claude/CLAUDE.md`（CC 路径在 `memory/paths.ts:48-65` 解析）

### 5. **22 子命令 + strict() + fail()**（index.ts:200-240）
```typescript
const cli = yargs(args)
  .parserConfiguration({ "populate--": true })
  .scriptName("mimo")
  .wrap(100)
  .help("help", "show help")
  .version("version", "show version number", InstallationVersion)
  .option("print-logs", { ... })
  .option("log-level", { type: "string", choices: ["DEBUG", "INFO", "WARN", "ERROR"] })
  .option("pure", { type: "boolean" })
  .middleware(async (opts) => {
    if (opts.pure) process.env.MIMOCODE_PURE = "1"
    await Log.init({ ... })
  })
  ...
  .command(AcpCommand)
  .command(McpCommand)
  ...
  .fail((msg, err) => {
    if (msg?.startsWith("Unknown argument") || msg?.startsWith("Not enough non-option arguments") ...) {
      if (err) throw err
      cli.showHelp(show)
    }
    if (err) throw err
    process.exit(1)
  })
  .strict()
```
- **`.strict()`**：未知参数直接 fail（不静默接受）
- **`.fail()` 自定义**：常见错误显示 help 而非 throw
- **`populate--`**：保留 `--foo` 之后的 arg 给子命令

### 6. **cmd 工厂**（`cli/cmd/cmd.ts`）
```typescript
export function cmd<T>({ command, describe, builder, handler }) { ... }
```
- 包装 yargs 常见 pattern——统一 import / 错误处理 / 退出码

### 7. **bootstrap 包装 Effect ctx**（`cli/bootstrap.ts:1-30`）
```typescript
export async function bootstrap<T>(directory: string, cb: () => Promise<T>) {
  return Instance.provide({
    directory,
    init: () => AppRuntime.runPromise(InstanceBootstrap),
    fn: async () => {
      try {
        return await cb()
      } finally {
        // Give detached background checkpoint writers a chance to finish
        // before teardown. Headless `mimo run` would otherwise exit right
        // after the main response, killing any forked writer mid-LLM-call
        // and leaving zero checkpoint files on disk.
        await AppRuntime.runPromise(SessionCheckpoint.Service.use(svc => svc.drainWriters()))
          .catch(err => log.warn("checkpoint drain failed", { error: String(err) }))
        await Instance.dispose()
      }
    },
  })
}
```
**关键设计**：**drainWriters() 在 finally**——给 detached 后台 checkpoint writers 最多 120s 跑完，避免 headless `mimo run` 跑完主响应就退、杀掉 fork 出去的 writer

### 8. **Heap.start() 堆监控**（`cli/heap.ts`）
- 启 V8 heap snapshot
- 周期性 dump（OOM 排查）
- 注释明确：防内存泄漏快速定位

### 9. **顶级错误处理**（index.ts:280-330）
```typescript
} catch (e) {
  let data: Record<string, any> = {}
  if (e instanceof NamedError) { Object.assign(data, e.toObject().data) }
  if (e instanceof Error) { Object.assign(data, { name, message, cause, stack }) }
  if (e instanceof ResolveMessage) { Object.assign(data, { name, message, code, specifier, referrer, position, importKind }) }
  Log.Default.error("fatal", data)
  const formatted = FormatError(e)
  if (formatted) UI.error(formatted)
  if (formatted === undefined) {
    UI.error("Unexpected error, check log file at " + Log.file() + " for more details" + EOL)
    process.stderr.write(errorMessage(e) + EOL)
  }
  process.exitCode = 1
} finally {
  // Some subprocesses don't react properly to SIGTERM ... Explicitly exit
  process.exit()
}
```
- **3 类错误类型**特判：NamedError / 普通 Error / 解析错误
- **`finally` 强制 `process.exit()`**——防 hanging subprocess（特别是 docker MCP server）

### 10. **`-h` / `--help` 特殊路径**（index.ts:280-290）
```typescript
if (args.includes("-h") || args.includes("--help")) {
  await cli.parse(args, (err, _argv, out) => {
    if (err) throw err
    if (!out) return
    show(out)  // 加 logo + 颜色
  })
} else {
  await cli.parse()
}
```
- help 走 `show()` 自定义渲染
- 普通走默认 parse

### 11. **Agent 子命令的 LLM-driven create**（`cli/cmd/agent.ts:1-40`）
```typescript
const AgentCreateCommand = cmd({
  command: "create",
  describe: "create a new agent",
  builder: (yargs) => yargs
    .option("path", { type: "string" })
    .option("description", { type: "string" })
    .option("mode", { type: "string", choices: ["all", "primary", "subagent"] })
    .option("tools", { type: "string", describe: `...Available: "${AVAILABLE_TOOLS.join(", ")}"` }),
  ...
})
const AVAILABLE_TOOLS = ["bash", "read", "write", "edit", "glob", "grep", "webfetch", "actor", "task"]
```
- **9 个内置工具名**——可被 `--tools` 子集

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 启动链 | 同 | **3 层 shim + Avx2/musl 检测** | 平台兼容 |
| 子命令数 | ~18 | **22** | + acp / agent / plug / web / mcp / db / thread / attach / debug |
| `pure` 模式 | 无 | **`MIMOCODE_PURE=1` 不加载 external plugin** | 调试隔离 |
| 一次性 DB 迁移 UI | 无 | **TTY 彩色进度条 + 非 TTY 单行格式** | 旧 JSON 兼容 |
| Claude 导入 | 无 | **`ClaudeImport.run()` 一次性** | 平滑迁移 |
| drainWriters finally | 无 | **给 detached writer 120s 跑完** | 防止 headless run 杀掉 writer |
| Heap 监控 | 无 | **`Heap.start()` V8 snapshot** | 内存分析 |
| `ResolveMessage` 错误 | 无 | **import 解析错误特判** | 友好提示 |
| `process.exit()` finally | 无 | **强制退出防 hanging subprocess** | 防止 docker MCP 卡死 |
| 平台 build 名 | `opencode` | **同 `opencode` (未改名)** | 注意：二进制仍叫 `opencode` |

## 关键引用

- `packages/opencode/bin/mimo:1-120` — 3 路 shim + Avx2/musl 检测
- `packages/opencode/src/index.ts:1-66` — 顶部 import + unhandled rejection
- `packages/opencode/src/index.ts:100-185` — middleware + 一次性迁移 + Claude import
- `packages/opencode/src/index.ts:200-240` — 22 command 注册 + strict + fail
- `packages/opencode/src/index.ts:280-335` — 顶级 catch + finally process.exit
- `packages/opencode/src/cli/cmd/cmd.ts` — cmd 工厂
- `packages/opencode/src/cli/bootstrap.ts:1-30` — drainWriters finally
- `packages/opencode/src/cli/cmd/agent.ts:1-40` — AgentCreate + AVAILABLE_TOOLS

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 的 3 层启动 shim 是什么？每层职责？
- [ ] `bin/mimo` 的 3 路决策顺序（MIMOCODE_BIN_PATH / 缓存 / 下载）？
- [ ] `supportsAvx2` 在三大平台分别怎么检测？为什么不直接禁用？
- [ ] musl/glibc 怎么判断？Alpine 用户会下哪个 build？
- [ ] 22 个 yargs 子命令大致怎么分类？哪些是核心入口（run/serve/web/tui）？
- [ ] 一次性 DB 迁移 UI 怎么区分 TTY / 非 TTY？
- [ ] `bootstrap()` 函数的 `finally` 块为什么要 `drainWriters()`？
- [ ] `Heap.start()` 的作用是什么？什么时候开？
- [ ] 顶级 catch 块区分的 3 类错误类型（NamedError / Error / ResolveMessage）各自什么场景？
- [ ] `finally` 块强制 `process.exit()` 的目的是什么？
