# 12 · Plugin / Skill / Workflow

## 一句话定位
MiMo-Code 用**三层扩展机制**把"如何发现能力"分成不同优先级：
- **Plugin**（`src/plugin/`）— 进程级，编译到二进制（Copilot / MiMo / GitLab / Poe / Cloudflare / Codex 鉴权 plugin）
- **Skill**（`src/skill/`）— 用户/项目级，`SKILL.md` 文件 + `discovery.ts` 远程拉取，含 `compose` 子系统（Bun macro 编译时内联）
- **Workflow**（`src/workflow/`）— 脚本级，内置 deep-research 等 + 用户 inline JS（`export const meta = ...`）

每层有独立的 tool（plugin 自动注入、skill 用 `skill` 工具调用、workflow 用 `workflow` 工具调用）。

## 涉及文件
### Plugin
- `packages/opencode/src/plugin/index.ts` (已读) — Plugin Loader + 11 内置 auth plugin + 5 HookEvent
- `packages/opencode/src/plugin/loader.ts` — 动态 plugin 加载
- `packages/opencode/src/plugin/matcher.ts` — actor 匹配
- `packages/opencode/src/plugin/meta.ts` — plugin 元数据
- `packages/opencode/src/plugin/mimo.ts` — MiMo 鉴权 plugin
- `packages/opencode/src/plugin/mimo-free.ts` — MiMo Free 匿名 JWT
- `packages/opencode/src/plugin/codex.ts` — Codex 鉴权
- `packages/opencode/src/plugin/cloudflare.ts` — Cloudflare AI Gateway / Workers
- `packages/opencode/src/plugin/github-copilot/` — Copilot 鉴权
- `packages/opencode/src/plugin/checkpoint-splitover.ts` — checkpoint writer 拆分 hook
- `packages/opencode/src/plugin/subagent-progress-checker.ts` — subagent 进度检查
- `packages/opencode/src/plugin/shared.ts` — 公共解析
- `packages/opencode/src/plugin/install.ts` — npm 安装

### Skill
- `packages/opencode/src/skill/index.ts` (已读) — `Info` schema + EXTERNAL_DIRS + SKILL_PATTERN
- `packages/opencode/src/skill/discovery.ts` (已读) — 远程 index.json + HTTP 拉取
- `packages/opencode/src/skill/compose/extract.ts` (已读) — Bun macro + extract Compose bundle
- `packages/opencode/src/skill/compose/bundle.macro.ts` (已读) — `loadComposeBundle` 扫 `.bundle/`
- `packages/opencode/src/skill/compose/LICENSE-karpathy` + `LICENSE-superpowers` — 第三方 skill 引用

### Workflow
- `packages/opencode/src/workflow/builtin.ts` (已读) — 内置 workflow registry（`deep-research.js` + `with { type: "text" }` import）
- `packages/opencode/src/workflow/runtime.ts` (已读) — 12h 脚本 deadline + 1000 agent cap + sandbox eval
- `packages/opencode/src/workflow/sandbox.ts` — 脚本安全沙箱
- `packages/opencode/src/workflow/meta.ts` — `parseMeta` 解析 `export const meta = {...}`
- `packages/opencode/src/workflow/resolve.ts` — script 解析（inline / file / named）
- `packages/opencode/src/workflow/persistence.ts` — journal + 重启恢复
- `packages/opencode/src/workflow/events.ts` — WorkflowStarted / Finished / Failed / Log / Phase / ChildFailed / AgentFailed
- `packages/opencode/src/workflow/workspace.ts` — workspace 沙箱（`makeFileHooks`）
- `packages/opencode/src/workflow/workflow.sql.ts` — workflow_run 表

### Tools
- `packages/opencode/src/tool/skill.ts` (已读) — `skill` 工具
- `packages/opencode/src/tool/workflow.ts` (已读) — `workflow` 工具（5 operation: run/status/wait/cancel/resume）

## 核心数据流

### Plugin 加载
```
启动
  ↓
Plugin.Layer 初始化
  ├─ 内置 plugin 直接 import（CodexAuthPlugin / MimoAuthPlugin / AnthropicProxyPlugin / MimoFreeAuthPlugin / CopilotAuthPlugin / GitlabAuthPlugin / PoeAuthPlugin / CloudflareAIGatewayAuthPlugin / CloudflareWorkersAuthPlugin / CheckpointSplitoverPlugin / SubagentProgressCheckerPlugin）
  └─ PluginLoader.loadUserPlugins() 读 mimocode.json 的 plugin 列表
  ↓
register all tools / hooks / events / auth providers / matchers
  ↓
TUI 启动时通过 ActorPreStopInput / ActorPostStopInput 注入 hook 上下文
```

### Skill 发现
```
启动
  ↓
Skill.dirs() 扫 4 个位置：
  - .claude/skills/**/SKILL.md
  - .agents/skills/**/SKILL.md
  - .codex/skills/**/SKILL.md
  - .opencode/skills/**/SKILL.md
  - {skill,skills}/**/SKILL.md（mimocode 原生）
  ↓
extractComposeBundle() — 用 Bun macro 读预编译的 compose bundle
  ↓
Discovery.pull(url) — 远程 index.json → download skill
  ↓
LLM 通过 `skill` 工具 { name: "..." } 加载
  ↓
返回 `<skill_content name="...">` + base directory + 文件列表
```

### Workflow 执行
```
LLM 调 `workflow` 工具 { operation: "run", name: "deep-research", args: {...} }
  ↓
BuiltinWorkflow.get("deep-research") 拿内置脚本
  ↓
WorkflowRuntime.run({ runID, sessionID, name, script, args, workspace })
  ├─ evalScript(script, hostFns) 沙箱执行
  ├─ 12h SCRIPT_DEADLINE_MS 计时
  ├─ 1000 MAX_LIFECYCLE_AGENTS 上限
  ├─ 16 DEFAULT_MAX_CONCURRENT 并发
  └─ 持久化 journal（重启可 resume）
  ↓
返回 { runID, status: "running", deferred }
  ↓
LLM 调 `workflow` 工具 { operation: "wait" / "status" / "cancel" / "resume" }
```

## 关键设计

### 1. **5 个 HookEvent**（`plugin/index.ts:33-58`）
- `hook.executed` — 记录每次 hook 触发
- `hook.react.reentered` — preStop / postStop 触发 ReAct re-entry
- `hook.react.max_reached` — cap 触顶
- 加上 `actor.preStop` / `actor.postStop` 两个 input 接口
- 让 plugin 能：
  - 监听 actor 状态变化
  - 决策 "actor 该不该继续"（continueRequested）

### 2. **Plugin 11 个内置**（`plugin/index.ts:1-30`）
| Plugin | 作用 |
|--------|------|
| CodexAuthPlugin | Codex 鉴权 |
| MimoAuthPlugin + AnthropicProxyPlugin | MiMo 账号 + Anthropic proxy |
| MimoFreeAuthPlugin | MiMo Free 匿名 JWT |
| CopilotAuthPlugin | GitHub Copilot 鉴权 |
| GitlabAuthPlugin | GitLab 鉴权 |
| PoeAuthPlugin | Poe 鉴权 |
| CloudflareAIGatewayAuthPlugin | Cloudflare AI Gateway |
| CloudflareWorkersAuthPlugin | Cloudflare Workers AI |
| CheckpointSplitoverPlugin | checkpoint 拆分 hook |
| SubagentProgressCheckerPlugin | subagent 进度检查 hook |

### 3. **Skill EXTERNAL_DIRS 跨生态兼容**（`skill/index.ts:24-28`）
```typescript
const EXTERNAL_DIRS = [".claude", ".agents", ".codex", ".opencode"]
const EXTERNAL_SKILL_PATTERN = "skills/**/SKILL.md"
const MIMOCODE_SKILL_PATTERN = "{skill,skills}/**/SKILL.md"
const SKILL_PATTERN = "**/SKILL.md"
```
- 兼容 Claude / Codex / OpenCode skill 目录
- 用户从其它 agent 迁过来不用改文件结构

### 4. **Discovery 远程拉取**（`skill/discovery.ts:23-95`）
- `pull(url)`：
  - 读 `index.json`（URL `base/index.json`）
  - 遍历 `skills[]`，逐个下载
  - **缓存到 `Global.Path.cache/skills/`**（命中直接返回）
- `download(url, dest)` 用 `withTransientReadRetry` 包装的 HTTP client
- 失败只 log 不 throw（不让一个 skill 拉失败杀掉整体）

### 5. **Compose Bundle 用 Bun Macro 编译时内联**（`skill/compose/extract.ts:1-30`）
```typescript
import { loadComposeBundle } from "./bundle.macro" with { type: "macro" }
import { loadComposeBundle as loadComposeBundleDev } from "./bundle.macro"

function safeLoadComposeBundle() {
  try {
    return loadComposeBundle()
  } catch(e) {
    if (e instanceof ReferenceError) {
      return loadComposeBundleDev()  // dev 模式兜底
    }
    throw e
  }
}
```
- **Bun macro** 编译时把 `.bundle/` 目录整个读进内存
- dev 模式（plugin test）宏不可用 → catch ReferenceError 走 `loadComposeBundleDev()`
- **注释明确说**："Bun macros only resolve in the static import graph of an entry point"

### 6. **`bundle.macro.ts` 实现**（`skill/compose/bundle.macro.ts:1-20`）
```typescript
export function loadComposeBundle(): Record<string, Record<string, string>> {
  const dir = path.resolve(import.meta.dir, ".bundle")
  const result: Record<string, Record<string, string>> = {}
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    if (!entry.isDirectory()) continue
    const files: Record<string, string> = {}
    walkDir(path.join(dir, entry.name), "", files)
    if (Object.keys(files).length > 0) result[entry.name] = files
  }
  return result
}
```
- 返回 `{ skillName: { "SKILL.md": content, "scripts/foo.sh": content, ... } }`
- 编译后整个 bundle inline 进二进制

### 7. **Compose Skills Block 渲染**（`skill/compose/extract.ts:50-90`）
```typescript
export function composeSkillsBlock(): string {
  const root = path.join(GlobalPath.data, "compose", InstallationVersion)
  const entries: string[] = []
  for (const [skillName, files] of Object.entries(COMPOSE_BUNDLE)) {
    const skillMd = files["SKILL.md"]
    if (!skillMd) continue
    const parsed = parseSkillMeta(skillMd)  // gray-matter 解析 frontmatter
    if (!parsed?.data?.name || !parsed?.data?.description) continue
    const location = pathToFileURL(path.join(root, "skills", skillName, "SKILL.md")).href
    entries.push(
      `  <skill>`,
      `    <name>${parsed.data.name}</name>`,
      `    <description>${parsed.data.description}</description>`,
      `    <location>${location}</location>`,
      `  </skill>`,
    )
  }
  return ... + 拼成 "## Available skills\n" + 全部 <skill> 块
}
```
- **prompt.ts:541** 在 compose 模式 reminder 里调 `composeSkillsBlock()` —— 注入 `<skill>` XML 块给 LLM

### 8. **Workflow 内置 `with { type: "text" }`**（`workflow/builtin.ts:1-15`）
```typescript
// @ts-expect-error TS1192: import-attribute text loader, resolved by Bun not tsgo
import DEEP_RESEARCH_SCRIPT from "./builtin/deep-research.js" with { type: "text" }
```
- **`with { type: "text" }`** — Bun 编译时把整个 `.js` 源码 inline 进二进制
- **`@ts-expect-error`** — tsgo 报 TS1192 但 Bun 接受
- **注释明确说**：`Bun.file(...).text()` 不行——编译后没文件系统

### 9. **Null-prototype Registry**（`workflow/builtin.ts:30-35`）
```typescript
const REGISTRY: Record<string, Entry> = Object.create(null)
```
- 防 `get("constructor")` 拿 `Object.prototype.constructor` 误命中
- 注释："self-evidently closed set"

### 10. **WorkflowRuntime 4 个 cap**（`workflow/runtime.ts:30-44`）
```typescript
const SCRIPT_DEADLINE_MS = 12 * 60 * 60 * 1000   // 12h 整脚本
const MAX_LIFECYCLE_AGENTS = 1000                 // 单 run 最多 spawn 1000 个 agent
const DEFAULT_MAX_CONCURRENT = 16                  // 默认并发
const STRAGGLER_TIMEOUT = Symbol("straggler-timeout")  // 超时哨兵
```
- **STRAGGLER_TIMEOUT** 是 Symbol：永远不会和 agent 返回的 object / string / null 撞——用于区分"超时" vs "正常返回"

### 11. **workflow 工具 5 operation**（`tool/workflow.ts:14-50`）
- `run` — 启动（name 或 script 二选一）
- `status` — 查
- `wait` — 阻塞等（带 timeout_ms）
- `cancel` — 取消
- `resume` — 恢复（journal 持久化）
- **`z.strictObject` + `z.discriminatedUnion("operation", [...])`** — 类型严格

### 12. **workflowRef 懒绑定**（`tool/workflow.ts:1-50`）
- 用 module-level `workflowRef.current` 拿到 runtime
- **关键注释**："mirrors the actor tool's spawnRef"
- 避免把 `WorkflowRuntime.Service` 拉进 `ToolRegistry.layer` 产生循环

### 13. **WORKFLOW_STRUCTURAL_ERROR marker**（`workflow/runtime.ts:42-44`）
```typescript
const WORKFLOW_STRUCTURAL_ERROR = "WorkflowStructuralError"
```
- 标 workflow wiring bug（cycle / over-depth / unknown name）—— **必须 loud fail**
- 不会降级为"child 失败返 null"—— 整 tree fail

### 14. **Skill 工具的 `<skill_content>` 输出**（`tool/skill.ts:30-65`）
- 读 SKILL.md frontmatter（`name` / `description` 必填）
- 列 10 个文件（ripgrep `SKILL.md` 之外）
- 包成 `<skill_content name="...">` + base directory + 文件列表
- LLM 拿到后可以 Read 任何文件

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| Plugin 体系 | 基础 | **11 个内置 + 5 HookEvent + ActorPreStop/PostStop input** | 更强生命周期 hook |
| Skill 系统 | 基础 | **4 生态兼容（Claude/Codex/OpenCode/MiMo）+ 远程 Discovery** | 平滑迁移 |
| Compose Skills | 无 | **Bun macro 编译时内联 + extract to `~/.local/share/mimocode/compose/<version>/`** | 离线可用 |
| Workflow | 无 | **5 operation + journal 持久化 + 12h deadline + sandbox eval** | 完整 spec |
| 内置 Workflow | 无 | **`deep-research.js` + `with { type: "text" }` inline** | 开箱即用 |
| Null-prototype Registry | 无 | **`Object.create(null)` 防 prototype 污染** | 安全 |
| STRAGGLER_TIMEOUT Symbol | 无 | **超时哨兵永不撞值** | 类型安全 |
| WORKFLOW_STRUCTURAL_ERROR | 无 | **wiring bug 必须 loud fail** | 防止 silent 降级 |
| `with { type: "macro" }` | 无 | **Bun macro 编译时读盘** | 离线部署 |
| workflowRef 懒绑定 | 无 | **module-level ref 破 Layer 循环** | 与 actor 一致 |
| 4 cap 数字 | 无 | **12h / 1000 / 16 / Symbol** | 防御性 |

## 关键引用

- `packages/opencode/src/plugin/index.ts:1-58` — 11 内置 plugin + 5 HookEvent
- `packages/opencode/src/skill/index.ts:24-30` — EXTERNAL_DIRS 4 兼容
- `packages/opencode/src/skill/discovery.ts:23-95` — 远程 index.json pull
- `packages/opencode/src/skill/compose/extract.ts:1-30` — Bun macro + safeLoadComposeBundle
- `packages/opencode/src/skill/compose/bundle.macro.ts:1-20` — `loadComposeBundle` 实现
- `packages/opencode/src/skill/compose/extract.ts:50-90` — `composeSkillsBlock` 渲染
- `packages/opencode/src/workflow/builtin.ts:1-50` — `with { type: "text" }` + null-prototype registry
- `packages/opencode/src/workflow/runtime.ts:30-44` — 4 cap 常量
- `packages/opencode/src/tool/skill.ts:30-65` — `<skill_content>` 输出
- `packages/opencode/src/tool/workflow.ts:14-50` — 5 operation zod

## 学习检查清单

读完后应能回答：
- [ ] Plugin / Skill / Workflow 三层扩展机制的区别是什么？分别由什么工具调用？
- [ ] `EXTERNAL_DIRS = [".claude", ".agents", ".codex", ".opencode"]` 为什么能兼容 4 个生态？
- [ ] Bun macro `with { type: "macro" }` 的作用是什么？dev 模式怎么兜底？
- [ ] `with { type: "text" }` 怎么把内置 workflow 脚本 inline 进二进制？为什么不能用 `Bun.file()`？
- [ ] Workflow 4 个 cap（12h / 1000 / 16 / Symbol）各自防御什么？
- [ ] `WORKFLOW_STRUCTURAL_ERROR` marker 为什么必须 loud fail？
- [ ] `workflowRef` 懒绑定怎么破 Layer 循环依赖？
- [ ] `composeSkillsBlock()` 在哪个时间点被调用、注入到哪？
