# 10 · Server / API

## 一句话定位
MiMo-Code 的 Server 层用 **Hono + hono-openapi** 暴露 RESTful API + WebSocket 事件流；分**两种部署模式**——`MIMOCODE_WORKSPACE_ID` 模式（单 workspace 直挂 instance routes）和**默认 control-plane 模式**（workspace router middleware 按 session 自动路由）；支持 **mDNS 局域网发现**、**Web UI 内嵌代理**（CSP 沙箱）、**sync fence 增量同步**（`x-mimocode-sync` header）和 **OpenAPI 文档自动生成**。

## 涉及文件
### 核心
- `packages/opencode/src/server/server.ts` (已读) — `create(opts)` 工厂 + `Default` lazy + `openapi()` 文档生成
- `packages/opencode/src/server/middleware.ts` — `AuthMiddleware` / `CompressionMiddleware` / `CorsMiddleware` / `ErrorMiddleware` / `LoggerMiddleware`
- `packages/opencode/src/server/adapter.{bun,node,bun}.ts` — Hono adapter（Bun/Node）
- `packages/opencode/src/server/error.ts` — errors helper
- `packages/opencode/src/server/event.ts` — `Connected` / `Disposed` BusEvent
- `packages/opencode/src/server/fence.ts` (已读) — sync fence（`x-mimocode-sync` header + `EventSequenceTable`）
- `packages/opencode/src/server/proxy.ts` (已读) — WebSocket proxy（hop headers 删除 + workspace 路由）
- `packages/opencode/src/server/workspace.ts` (已读) — `WorkspaceRouterMiddleware` + `RULES` 路由表
- `packages/opencode/src/server/mdns.ts` (已读) — Bonjour mDNS `mimocode.local`
- `packages/opencode/src/server/projectors.ts` — `initProjectors()` 启动同步投影

### Routes
- `routes/global.ts` — 跨 instance 路由（SSE 事件流 / config / provider / installation）
- `routes/control/index.ts` + `routes/control/workspace.ts` — control plane 路由
- `routes/instance/index.ts` + 19 个子文件
  - `bash-interactive.ts`, `config.ts`, `event.ts`, `experimental.ts`, `file.ts`, `httpapi/`, `mcp.ts`, `middleware.ts`, `permission.ts`, `project.ts`, `provider.ts`, `pty.ts`, `question.ts`, `session.ts`, `sync.ts`, `trace.ts`, `tui.ts`, `workflows.ts`
- `routes/ui.ts` (已读) — **Web UI 嵌入 / 代理 + CSP**

## 核心数据流

### Server 启动
```
mimo serve
  ↓
Server.Default() = lazy(() => create({}))
  ↓
create({ cors? }):
  ├─ 初始化 Hono
  ├─ 5 middleware: Error → Auth → Logger → Compression → CORS
  ├─ /global/* → GlobalRoutes
  ├─ runtime = adapter.create(app)
  ├─ if MIMOCODE_WORKSPACE_ID:
  │    └─ InstanceMiddleware + FenceMiddleware + InstanceRoutes (无 control plane)
  └─ else (默认):
       ├─ / → ControlPlaneRoutes
       ├─ /experimental/workspace → WorkspaceRoutes
       ├─ WorkspaceRouterMiddleware
       ├─ / → InstanceRoutes
       └─ / → UIRoutes
  ↓
Server.listen({ port, hostname, mdns?, cors? }) → Listener
  ↓
  ├─ mDNS: Bonjour publish("mimocode.local", "mimocode-<port>", "http")
  └─ Worker 模式下被 `worker.ts:server` 调用
```

### 两种部署模式
| 模式 | 触发 | 路由 |
|------|------|------|
| **Workspace 模式** | `Flag.MIMOCODE_WORKSPACE_ID` 存在 | InstanceMiddleware 直挂 InstanceRoutes（无 control plane） |
| **Control-Plane 模式** | 默认 | ControlPlaneRoutes + WorkspaceRouterMiddleware + InstanceRoutes + UIRoutes |

### WorkspaceRouterMiddleware RULES（已读）
```typescript
const RULES: Rule[] = [
  { path: "/session/status", action: "forward" },  // 跨 workspace 转发
  { method: "GET", path: "/session", action: "local" },  // 本 workspace
]
```
- 按 `method + path` 决定 local（自己处理）还是 forward（按 session 路由到对应 workspace）
- `getSessionID(url)` 提取 `/session/<id>/...` 中的 id
- `getSessionWorkspace(url)` 查 session 拿到 workspaceID

## 关键设计

### 1. **5 Middleware 顺序**（`server.ts:35-42`）
```typescript
const app = new Hono()
  .onError(ErrorMiddleware)
  .use(AuthMiddleware)
  .use(LoggerMiddleware)
  .use(CompressionMiddleware)
  .use(CorsMiddleware(opts))
```
- **ErrorMiddleware**（`.onError`）：统一错误响应
- **AuthMiddleware**：`Authorization` header 校验
- **LoggerMiddleware**：请求日志
- **CompressionMiddleware**：gzip
- **CorsMiddleware**：跨域

### 2. **Web UI 嵌入 + Proxy 兜底**（`routes/ui.ts:1-30`）
```typescript
const embeddedUIPromise = Flag.MIMOCODE_DISABLE_EMBEDDED_WEB_UI
  ? Promise.resolve(null)
  : // @ts-expect-error - generated file at build time
    import("opencode-web-ui.gen.ts").then((module) => module.default as Record<string, string>).catch(() => null)

const DEFAULT_CSP = "default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; media-src 'self' data:; connect-src 'self' data:"
```
- **生产**：编译时生成的 `opencode-web-ui.gen.ts` 把 web UI 内嵌到 `Record<path, content>`
- **dev/无嵌入**：proxy 到 `https://app.opencode.ai<path>`
- **CSP 安全**：HTML 注入 `default-src 'self'` + `script-src 'self' 'wasm-unsafe-eval'`
- **`@ts-expect-error`** 注释明确说"generated file at build time"——tsgo 没这文件

### 3. **CSP 自定义 hash**（`routes/ui.ts:30-50`）
```typescript
const csp = (hash = "") => `default-src 'self'; script-src 'self' 'wasm-unsafe-eval'${hash ? ` 'sha256-${hash}'` : ""}; ...`
```
- 动态加 inline script 的 sha256 hash——允许内联主题预加载脚本
- 注释明确说 "oc-theme-preload-script" id 的 inline script

### 4. **mDNS 局域网发现**（`mdns.ts:1-30`）
```typescript
bonjour = new Bonjour()
const service = bonjour.publish({ name, type: "http", host, port, txt: { path: "/" } })
```
- 域名：`mimocode.local`（可配置）
- 服务名：`mimocode-<port>`（防冲突）
- TXT 记录：`path: "/"`
- **`currentPort === port` skip**：同端口重复 publish 自动跳过

### 5. **Sync Fence 增量同步**（`fence.ts:1-40`）
```typescript
const HEADER = "x-mimocode-sync"

export function load(ids?: string[]) {
  // 查 EventSequenceTable: { aggregate_id, seq }
  return Object.fromEntries(rows.map(r => [r.aggregate_id, r.seq]))
}

export function diff(prev: State, next: State) {
  // 返回 { aggregate_id: new_seq } for ids that changed
}

export function parse(headers: Headers) {
  // 解析 "id1:seq1,id2:seq2" 格式
}
```
- 客户端发请求带 `x-mimocode-sync: <id>:<seq>,<id>:<seq>` 记录已见版本
- 服务端返回 diff → 客户端 merge
- **减少重复同步**——TUI 拉新 event 不会重发已收到的

### 6. **WebSocket Proxy**（`proxy.ts:1-50`）
```typescript
const hop = new Set([
  "connection", "keep-alive", "proxy-authenticate", "proxy-authorization",
  "proxy-connection", "te", "trailer", "transfer-encoding", "upgrade", "host",
])
function headers(req, extra) {
  // 删 hop-by-hop headers + accept-encoding + x-mimocode-*（路由用）
}
```
- **hop-by-hop header 删除**：HTTP 1.1 spec，proxy 不能转
- **删除 `accept-encoding`**：避免下游响应被压后无法流式
- **删除 `x-mimocode-directory` / `x-mimocode-workspace`**：这些 header 只在 router middleware 里看，proxy 不应外传
- 接收 `sec-websocket-protocol` header 解析子协议

### 7. **5 个 Worker RPC 方法**（`worker.ts:30-80`）
```typescript
export const rpc = {
  async fetch(input) { /* 转发到 Server.Default().app.fetch */ },
  snapshot() { return writeHeapSnapshot("server.heapsnapshot") },
  async server(input) { /* 启动/重启动 server */ },
  async checkUpgrade(input) { /* 检查升级 */ },
  ...
}
```
- TUI 全部请求走 `fetch`
- `snapshot` 给调试
- `server` 支持动态切换端口/hostname

### 8. **OpenAPI 文档生成**（`server.ts:71-90`）
```typescript
export async function openapi() {
  // Build a fresh app with all routes registered directly so
  // hono-openapi can see describeRoute metadata (`.route()` wraps
  // handlers when the sub-app has a custom errorHandler, which
  // strips the metadata symbol).
  const { app } = create({})
  const result = await generateSpecs(app, {
    documentation: { ... }
  })
  return result
}
```
- **关键注释**：`.route()` 在有 custom errorHandler 的子 app 上会**剥掉 metadata symbol**——hono-openapi 看不到
- 解决：build fresh app 让所有 routes 直接 register

### 9. **19 个 instance 路由**（已 ls 验证）
- bash-interactive / config / event / experimental / file / mcp / permission / project / provider / pty / question / session / sync / trace / tui / workflows / httpapi/ / middleware

### 10. **Adapter Bun/Node 双后端**（`adapter.bun.ts` / `adapter.node.ts`）
- Hono 的 server adapter：bottleneck 在 `Bun.serve` vs `node:http`
- `#hono` import condition 走对应后端

### 11. **流式 SSE**（`routes/global.ts:30-70`）
- `streamSSE` from `hono/streaming`
- 10s heartbeat 防止 stalled proxy
- 初始 `server.connected` 事件
- `AsyncQueue<string | null>` 缓冲

### 12. **PROJECT_NAME → control-plane 命名变化**
- `routes/control/`（**非** `routes/management/`）
- `routes/control/workspace.ts` 单独处理 workspace meta

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 路由数 | ~15 | **19 instance + 2 control + 1 global + 1 ui = 23** | 更完整 |
| Workspace 模式 | 无 | **`MIMOCODE_WORKSPACE_ID` 单 workspace 简化模式** | 企业部署 |
| Control-Plane 模式 | 无 | **多 workspace + WorkspaceRouterMiddleware** | SaaS 形态 |
| Sync Fence | 无 | **`x-mimocode-sync: id:seq,id:seq` 增量同步** | 减少重复 |
| CSP | 无 | **sha256 hash inline script** | 主题预加载 |
| Web UI 嵌入 | 基础 | **`opencode-web-ui.gen.ts` 编译时内嵌** | 离线可用 |
| mDNS | 基础 | **Bonjour publish + txt:path** | 局域网发现 |
| 10s SSE heartbeat | 无 | **防止 stalled proxy** | 鲁棒性 |
| hop-by-hop header 严格删除 | 无 | **10 个标准 hop 集合** | HTTP 1.1 合规 |
| accept-encoding 删 | 无 | **让流式不被压** | TUI streaming |
| `x-mimocode-*` 删 | 无 | **路由头不外泄** | 多 workspace 安全 |
| `writeHeapSnapshot("server.heapsnapshot")` | 无 | **Worker RPC 暴露** | 远程 debug |
| 控制平面 | 无 | **control/index + control/workspace** | enterprise/saas |

## 关键引用

- `packages/opencode/src/server/server.ts:30-70` — 5 middleware + 两种模式
- `packages/opencode/src/server/server.ts:71-90` — openapi() 文档生成
- `packages/opencode/src/server/routes/ui.ts:1-50` — Web UI 嵌入 + CSP
- `packages/opencode/src/server/mdns.ts:1-30` — Bonjour publish
- `packages/opencode/src/server/fence.ts:1-40` — sync fence
- `packages/opencode/src/server/proxy.ts:1-50` — WebSocket proxy + hop 删除
- `packages/opencode/src/server/workspace.ts:1-50` — RULES + workspace 路由
- `packages/opencode/src/server/routes/global.ts:30-70` — SSE stream + heartbeat
- `packages/opencode/src/server/routes/` — 19 instance + 2 control + 1 global + 1 ui

## 学习检查清单

读完后应能回答：
- [ ] Server 启动的 5 个 middleware 顺序是什么？每个的职责？
- [ ] 两种部署模式（workspace vs control-plane）的触发条件和路由差异？
- [ ] Web UI 嵌入的 3 种情况（生产 / dev / 失败）怎么走？
- [ ] CSP 里 `'wasm-unsafe-eval'` 和动态 `'sha256-<hash>'` 各允许什么？
- [ ] mDNS 在 Bonjour 服务的 name / type / host / port / txt 分别是什么？
- [ ] Sync Fence 的 `x-mimocode-sync` header 格式？`load` / `diff` / `parse` 三步流程？
- [ ] WebSocket Proxy 删的 10 个 hop-by-hop headers 是哪 10 个？为什么要单独删 `accept-encoding`？
- [ ] 10s SSE heartbeat 解决什么问题？
- [ ] 23 个路由大致按什么分类？
