# 09 · TUI 终端 UI

## 一句话定位
MiMo-Code 的 TUI 是用 **SolidJS + @opentui/solid** 写的终端 UI——主进程只渲染 UI，**所有业务逻辑（agent / session / provider / DB）通过 `Worker` 进程 RPC 调用**（GlobalBus event 转发），TUI 拥有 23 个 dialog、3 个 route、36 个 component、6 种语言 i18n、完整 plugin 扩展点（slot / theme / route）。

## 涉及文件
### 核心入口
- `packages/opencode/src/cli/cmd/tui/app.tsx` (1130 行，已读头 100 行) — TUI 入口
- `packages/opencode/src/cli/cmd/tui/thread.ts` — 启动 TUI 新线程
- `packages/opencode/src/cli/cmd/tui/attach.ts` — 附加到运行中的 TUI
- `packages/opencode/src/cli/cmd/tui/worker.ts` — **Worker 进程入口**（业务逻辑全在这）
- `packages/opencode/src/cli/cmd/tui/layer.ts` — Effect CliLayer = Observability + TuiConfig + Npm
- `packages/opencode/src/cli/cmd/tui/win32.ts` — Windows 特殊处理（Ctrl-C guard）
- `packages/opencode/src/cli/cmd/tui/event.ts` — TuiEvent bus 事件
- `packages/opencode/src/cli/cmd/tui/internal.ts` — INTERNAL_TUI_PLUGINS
- `packages/opencode/src/cli/cmd/tui/slots.tsx` — Plugin slot 系统

### Context（10 个）
- `args.tsx`, `cwd.ts`, `event.ts`, `exit.tsx`, `helper.tsx`, `keybind.tsx`, `kv.tsx`, `language.tsx`, `local.tsx`, `plugin-keybinds.ts`, `project.tsx`, `prompt.tsx`, `route.tsx`, `sdk.tsx`, `sync.tsx`, `theme.tsx`, `thinking.ts`, `tui-config.tsx`, `tui-migrate.ts`, `tui-schema.ts`, `tui.ts`

### UI（7 个）
- `dialog.tsx`, `dialog-alert.tsx`, `dialog-confirm.tsx`, `dialog-export-options.tsx`, `dialog-help.tsx`, `dialog-prompt.tsx`, `dialog-select.tsx`, `link.tsx`, `spinner.tsx`, `toast.tsx`

### Component（36 个）
- 23 个 dialog
- 9 个其他（background-image, bg-pulse, border, error-component, logo, plugin-route-missing, spinner, starry-background, startup-loading, task-item, textarea-keybindings, todo-item）
- 4 个 prompt 子目录

### Routes
- `home.tsx` — 首页（最近 session / 项目列表）
- `session/index.tsx` — Session route 聚合
- `session/{dialog-fork-from-timeline, dialog-message, dialog-subagent, dialog-timeline, footer, index, permission, question, sidebar, subagent-footer}.tsx` — 10 个 session 子组件

### Plugin
- `plugin/index.ts` — Plugin 导出
- `plugin/runtime.ts` — Plugin 加载运行时
- `plugin/api.tsx` — TUI plugin API
- `plugin/slots.tsx` — slot 视图
- `plugin/internal.ts` — 内置 plugin

### i18n
- 8 个 locale：`en.ts` / `es.ts` / `fr.ts` / `ja.ts` / `ru.ts` / `zh.ts` / `zht.ts` + `locales.ts`（注册表）

## 核心数据流

### TUI 进程架构
```
┌──────────────────────────────────────┐
│  主进程（TUI 渲染）                   │
│  - SolidJS + @opentui/solid          │
│  - 36 component / 23 dialog          │
│  - 全在 app.tsx + route              │
│                                      │
│  RPC emit("global.event", event)     │ ──→ Worker 进程
│  RPC call("fetch", ...)              │ ←── Server.handle
└──────────────────────────────────────┘
                  ↕ RPC（@/util/Rpc）
┌──────────────────────────────────────┐
│  Worker 进程（业务逻辑）               │
│  - Session / Agent / Provider / DB  │
│  - GlobalBus.on("event") 订阅        │
│  - Server.listen() 启动 HTTP         │
│  - Effect 全套                       │
└──────────────────────────────────────┘
```

### TUI 启动
```
$ mimo                                  # 启动 TUI
   ↓
bin/mimo → 缓存二进制 → src/index.ts
   ↓
yargs dispatch `tui` 命令
   ↓
TuiThreadCommand / 直接 src/cli/cmd/tui/app.tsx
   ↓
1. ensureProcessMetadata("main")  // 标记主进程
2. rendererConfig(plainTerminal?) — 决定 fps 60/15、mouse on/off
3. render(<App />) — SolidJS 挂载
   ↓
App.tsx 内部：
   ├─ RouteProvider
   ├─ DialogProvider
   ├─ KeybindProvider
   ├─ ThemeProvider
   ├─ SDKProvider (rpc = worker rpc)
   ├─ SyncProvider (RPC 拉全状态)
   ├─ ProjectProvider
   ├─ LocalProvider
   ├─ PromptHistoryProvider
   ├─ FrecencyProvider
   ├─ PromptStashProvider
   ├─ ArgsProvider
   ├─ ToastProvider
   ├─ ExitProvider
   ├─ TuiConfigProvider
   ├─ LanguageProvider
   ├─ PromptRefProvider
   └─ Switch/Match 路由 home / session
```

## 关键设计

### 1. **Worker 隔离**（`worker.ts:1-30`）
```typescript
ensureProcessMetadata("worker")
await Log.init({ ... })
Heap.start()
process.on("unhandledRejection", e => Log.Default.error("rejection", { e: e instanceof Error ? e.message : e }))
process.on("uncaughtException", e => Log.Default.error("exception", { e }))

// Subscribe to global events and forward them via RPC
GlobalBus.on("event", (event) => {
  Rpc.emit("global.event", event)
})

let server: Awaited<ReturnType<typeof Server.listen>> | undefined

export const rpc = {
  async fetch(input) { /* 转发到 Server.Default().app.fetch */ },
  snapshot() { return writeHeapSnapshot("server.heapsnapshot") },
  async server(input) { if (server) await server.stop(true); server = await Server.listen(input); return { url: server.url.toString() } },
  async checkUpgrade(input) { ... },
  ...
}
```
- **TUI 主进程零业务逻辑**——只渲染
- **Worker 进程承载所有 Session/Agent/Provider/DB**——崩溃不毁 TUI
- **GlobalBus event 转发**：Worker 业务事件 → `Rpc.emit("global.event", event)` → TUI 渲染

### 2. **`createTuiApi` 把 TUI 暴露给 plugin**（`plugin/runtime.ts`）
- TUI plugin 可以挂 slot（视图位点）/ theme / route
- **`@opentui/solid/runtime-plugin-support`** 装饰器
- `setupSlots(Slot as View)` 装配 plugin 提供的视图

### 3. **rendererConfig 双形态**（`app.tsx:80-100`）
```typescript
function rendererConfig(_config, plainTerminal): CliRendererConfig {
  const mouseEnabled = !plainTerminal && !Flag.MIMOCODE_DISABLE_MOUSE && (_config.mouse ?? true)
  return {
    externalOutputMode: "passthrough",
    targetFps: plainTerminal ? 10 : 60,
    gatherStats: false,
    exitOnCtrlC: false,
    useKittyKeyboard: plainTerminal ? null : {},
    autoFocus: false,
    openConsoleOnError: false,
    enableMouseMovement: mouseEnabled,
    useMouse: mouseEnabled,
    ...(plainTerminal
      ? { maxFps: 15, screenMode: "main-screen", useThread: false, backgroundColor: "transparent" }
      : { maxFps: 60 }),
    consoleOptions: { keyBindings: [{ name: "y", ctrl: true, action: "copy-selection" }], onCopySelection: ... },
  }
}
```
- **plainTerminal 模式**：10/15 fps、不用线程、透明背景、kitty keyboard off —— pipe / log 友好
- **完整模式**：60 fps、mouse 全开、kitty keyboard on

### 4. **23 个 dialog**（已 ls 验证）
dialog-agent / dialog-command / dialog-console-org / dialog-go-upsell / dialog-image-list / dialog-logo-design / dialog-mcp / dialog-mimo-login / dialog-model / dialog-provider / dialog-session-delete-failed / dialog-session-list / dialog-session-rename / dialog-skill / dialog-stash / dialog-status / dialog-tag / dialog-theme-list / dialog-variant / dialog-workflows / dialog-workspace-create / dialog-workspace-unavailable / dialog-worktree

### 5. **Session 路由子组件**（10 个）
- `index.tsx` — Session 主视图
- `footer.tsx` — 底部状态栏
- `sidebar.tsx` — 侧边栏
- `permission.tsx` — 权限询问 UI
- `question.tsx` — 问用户问题 UI
- `dialog-fork-from-timeline.tsx` — 时间线 fork 对话框
- `dialog-message.tsx` — 消息详情
- `dialog-subagent.tsx` — 子 agent 详情
- `dialog-timeline.tsx` — 时间线
- `subagent-footer.tsx` — 子 agent 状态栏

### 6. **3 套 plugin 注册机制**（`plugin/runtime.ts`）
- **TuiPlugin** — 通用 plugin
- **TuiSlotPlugin** — 只挂 slot 的轻量 plugin
- **INTERNAL_TUI_PLUGINS** — 编译内置的 plugin（`./internal.ts`）

### 7. **i18n 8 语言**（`i18n/{locales,en,es,fr,ja,ru,zh,zht}.ts`）
- `Locale` type
- `LOCALES` 注册表
- `LanguageProvider` context
- `UiI18nBridge` 桥接 UI 组件

### 8. **TuiConfig 持久化配置**（`context/tui-config.tsx` + `config/tui.ts`）
- 用户级 TUI 配置（mouse / fps / theme / keybind）
- 走 KV 存储
- `TuiConfigProvider` 注入

### 9. **Plugin 加载容错**（`plugin/runtime.ts`）
- `PluginLoader.loadUserPlugins()` + 错误恢复
- 缺主题 / 缺 slot 不影响主程序
- `errorData(err) / errorMessage(err)` 详细记录

### 10. **10 个 context 提供方**
- **args**：启动参数
- **cwd**：当前 worktree
- **event**：bus 事件流
- **exit**：退出流程
- **keybind**：快捷键映射
- **kv**：本地 KV（设置 / 历史）
- **language**：i18n
- **local**：本地状态
- **plugin-keybinds**：plugin 提供的快捷键
- **project**：项目元数据
- **prompt**：输入框状态
- **route**：当前路由
- **sdk**：RPC client（连 worker）
- **sync**：全状态同步
- **theme**：主题
- **thinking**：thinking 状态
- **tui-config**：用户配置

### 11. **Win32 Ctrl-C Guard**（`win32.ts`）
- `win32DisableProcessedInput` — 关闭 Windows 模式下的 Ctrl-C 处理
- `win32InstallCtrlCGuard` — 装 Ctrl-C 守护避免子进程接管

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| TUI 框架 | SolidJS | **同 SolidJS + @opentui/solid** | 继承 |
| Worker 隔离 | 无 / 简单 | **完整 Worker 进程 + RPC 转发** | 主进程零业务 |
| Dialog 数 | ~15 | **23** | + logo-design / workspace-* / go-upsell / tag / variant / image-list |
| Session 子组件 | 5 | **10** | + fork-from-timeline / subagent-footer / dialog-timeline / dialog-subagent |
| 国际化 | 基础 | **8 语言**（en/es/fr/ja/ru/zh/zht） | 全球化 |
| Plugin 体系 | 基础 | **TuiPlugin + TuiSlotPlugin + INTERNAL** | 完整扩展点 |
| TuiConfig | 无 | **用户级 TUI 配置 KV 持久化** | 鼠标 / fps / 主题 |
| Frecency | 无 | **FrecencyProvider** | 输入建议按频次排 |
| Prompt Stash | 无 | **PromptStashProvider** | 草稿暂存 |
| 路由 | home/session | **home + session + dialog-timeline + subagent 详情** | 深度嵌套 |
| `writeHeapSnapshot` | 无 | **"server.heapsnapshot" 周期 dump** | 内存诊断 |
| plainTerminal 模式 | 无 | **10/15 fps + 透明背景 + 不开线程** | pipe 友好 |

## 关键引用

- `packages/opencode/src/cli/cmd/tui/worker.ts:1-30` — Worker 进程入口
- `packages/opencode/src/cli/cmd/tui/worker.ts:30-80` — rpc 桥接 Server
- `packages/opencode/src/cli/cmd/tui/app.tsx:80-100` — rendererConfig 双形态
- `packages/opencode/src/cli/cmd/tui/app.tsx:1-50` — Provider 注入链
- `packages/opencode/src/cli/cmd/tui/layer.ts:1-10` — CliLayer = Observability + TuiConfig + Npm
- `packages/opencode/src/cli/cmd/tui/plugin/runtime.ts:1-30` — Plugin 加载
- `packages/opencode/src/cli/cmd/tui/i18n/` — 8 语言
- `packages/opencode/src/cli/cmd/tui/component/` — 23 dialog + 9 其他 = 36 files

## 学习检查清单

读完后应能回答：
- [ ] TUI 进程架构分几层？Worker 进程承担什么职责？
- [ ] GlobalBus event 怎么从 Worker 转发到 TUI 主进程？
- [ ] 23 个 dialog 大致按什么功能分类？
- [ ] `plainTerminal` 模式渲染参数和完整模式有什么区别？什么场景用 plainTerminal？
- [ ] TUI Plugin 体系有哪 3 种类型？分别用于什么场景？
- [ ] Session 路由有哪 10 个子组件？哪些是用户交互、哪些是状态展示？
- [ ] 国际化支持哪 8 种语言？locale 怎么注册？
- [ ] 10+ 个 context provider 大致按职责分哪几类？
