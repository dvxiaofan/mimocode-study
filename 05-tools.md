# 05 · Tools 工具集

## 一句话定位
MiMo-Code 把 LLM 可调用的"动作"抽象成 35 个 tool 文件——按"读 / 写 / 跑命令 / 子 agent / 任务 / 工作流 / 记忆 / 外部"九大类组织；每个 tool 走统一的 `Tool.define(id, initFn)` 工厂返回 `Def<z.ZodType, Metadata>`，运行时被 `ToolRegistry` 聚合并按 agent 的 `toolAllowlist` 过滤、按 MCP 协议扩展、按 `shell` 字段支持 shell-mode 简化调用。

## 涉及文件
- `packages/opencode/src/tool/tool.ts` — **Tool.Def / Info / Context 类型基类**（已读）
- `packages/opencode/src/tool/registry.ts` — 工具注册表 + plugin 扩展点
- `packages/opencode/src/tool/index.ts` — 4 行 re-export
- `packages/opencode/src/tool/schema.ts` — ToolDefinition zod schema
- `packages/opencode/src/tool/invocation-style.ts` — `resolveInvocationStyle(toolCfg, toolName)` 决定 shell / JSON
- 35 个 tool 文件（按类组织见下）

## 核心数据流

### 工具分类（按文件）
| 类别 | 工具 | 文件 | 作用 |
|------|------|------|------|
| **读** | read | `read.ts` | 读文件（带 line range / glob / image） |
| | glob | `glob.ts` | glob 模式匹配文件 |
| | grep | `grep.ts` | ripgrep 全文搜索 |
| | codesearch | `codesearch.ts` | 语义代码搜索 |
| | lsp | `lsp.ts` | LSP 跳转定义 / 引用 |
| | webfetch | `webfetch.ts` | HTTP GET 转 markdown |
| | websearch | `websearch/*.ts` | 网络搜索 |
| | history | `history.ts` | 历史消息搜索（FTS5） |
| **写** | write | `write.ts` | 写文件 |
| | edit | `edit.ts` | 字符串替换编辑 |
| | multiedit | `multiedit.ts` | 多次 edit 一次提交 |
| | apply_patch | `apply_patch.ts` | git-apply 风格 patch |
| **执行** | bash | `bash.ts` | 跑命令（带 permission 询问、文件操作预检） |
| | bash-interactive | `bash-interactive.ts` | 交互式 shell |
| **Agent 控制** | task | `task.ts` | 任务追踪（与 TaskRegistry 联动） |
| | actor | `actor.ts` | spawn / wait / cancel 子 agent |
| | plan | `plan.ts` | 进入/退出 plan 模式 |
| | workflow | `workflow.ts` | 调用内置 workflow |
| **上下文控制** | change-directory | `change-directory.ts` | 切 session cwd |
| | session-cwd | `session-cwd.ts` | 查/设 session cwd |
| | truncate | `truncate.ts` | 大输出截断 |
| | truncation-dir | `truncation-dir.ts` | 截断文件目录 |
| **交互** | question | `question.ts` | 向用户提问 |
| **记忆** | memory | `memory.ts` | BM25 搜 MEMORY/CHECKPOINT/NOTES |
| | memory-path-guard | `memory-path-guard.ts` | 防止子 agent 越权改记忆 |
| **Skill** | skill | `skill.ts` | 调用已加载 skill |
| **杂** | external-directory | `external-directory.ts` | external 目录访问询问 |
| | invalid | `invalid.ts` | 解析失败的 tool call 兜底 |
| | shell-tokenize | `shell-tokenize.ts` | shell 命令 tokenize 工具 |
| | shell-wrap | `shell-wrap.ts` | shell 模式包装 |
| | mcp-exa | `mcp-exa.ts` | Exa MCP 客户端（搜索） |
| **元** | tool | `tool.ts` | Tool.Def 基类 |

### Tool 生命周期
```
ToolRegistry 启动时聚合所有 Tool.Info
  ↓
为每个 Info 调 init() 拿 DefWithoutID
  ↓
session/prompt.ts:resolveTools() 调 registry.tools({ model, agent })
  ↓
返回 Info[] 列表（按 agent.toolAllowlist 过滤、provider capability 过滤）
  ↓
遍历 Info[] 构造 ai-sdk 的 tool({ description, inputSchema, execute })
  ↓
LLM 调 tool → execute(args, ctx) → Effect<ExecuteResult>
  ↓
ctx 里有 permission.ask / metadata() / abort / messages
  ↓
tool 返回 { title, metadata, output, attachments? } 给 LLM
```

## 关键设计

### 1. **Tool.Def 强类型契约**（`tool.ts:30-50`）
```typescript
export interface Def<Parameters extends z.ZodType = z.ZodType, M extends Metadata = Metadata> {
  id: string
  description: string
  parameters: Parameters                                            // zod schema
  execute(args: z.infer<Parameters>, ctx: Context): Effect.Effect<ExecuteResult<M>>
  formatValidationError?(error: z.ZodError): string
  shell?: {
    description: string
    parse(script: string): Effect.Effect<z.infer<Parameters>[], unknown>
    recover?(rawArgs: unknown): z.infer<Parameters> | undefined
  }
}
```
- **强类型推导**：`z.infer<Parameters>` 让 execute 自动拿到验证后的 args
- **`shell` 字段**：定义 shell-mode 简化调用形式（详见 §3）
- **`formatValidationError`**：自定义 zod 校验失败时的 LLM 可见错误信息

### 2. **Tool.Context 运行时信息**（`tool.ts:19-28`）
```typescript
export type Context<M extends Metadata = Metadata> = {
  sessionID: SessionID
  messageID: MessageID
  agent: string
  actorID?: string       // 当前是否在 subagent 中
  taskId?: string        // 当前是否绑定到 task
  abort: AbortSignal
  callID?: string
  extra?: { [key: string]: unknown }
  messages: MessageV2.WithParts[]
  metadata(input: { title?: string; metadata?: M }): Effect.Effect<void>
  ask(input: Omit<Permission.Request, "id" | "sessionID" | "tool">): Effect.Effect<void>
}
```
- `ask()` 触发 permission 系统询问
- `metadata()` 实时更新 tool call 在 UI 上的状态（title / metadata）
- `actorID` / `taskId` 让 tool 知道"我正在为谁服务"

### 3. **Invocation-Style 双形态**（`invocation-style.ts`）
- 工具可声明 `shell` 字段——允许**shell-mode** 简化调用
- 例如 `task` 工具：
  - JSON 形态：`task({ operation: "list" })`
  - Shell 形态：`task list`（简化）
- `resolveInvocationStyle(toolCfg, "task")` 决定走哪种
- **关键**（`prompt.ts:85-93`）：shell-mode 永远不会返回 JSON-shape 示例（防止 LLM 输出 JSON 让 shell 解析挂掉）
- `memory` 工具**没有** shell 形态——只走 JSON

### 4. **registry.tools() 过滤链**（`registry.ts`）
```typescript
registry.tools({
  modelID: ModelID.make(input.model.api.id),
  providerID: input.model.providerID,
  agent: input.agent,
})
```
- 按 provider capability 过滤（vision / tool call）
- 按 agent.toolAllowlist 过滤
- 按 plan / compose 等模式过滤（plan agent 看不到 edit 工具）
- 合并 plugin 提供的 tool

### 5. **bash 工具的"危险命令预检"**（`bash.ts`）
```typescript
const FILES = new Set([
  "rm", "cp", "mv", "mkdir", "touch", "chmod", "chown", "cat",  // Unix
  // PowerShell 等价物
  "remove-item", "copy-item", "move-item", "new-item", "rename-item", ...
])
const FLAGS = new Set(["-destination", "-literalpath", "-path"])
const SWITCHES = new Set(["-confirm", "-debug", "-force", "-recurse", "-verbose", "-whatif"])
```
- **预检逻辑**：识别文件操作命令 → 提前问 permission（避免执行到一半才 fail）
- PowerShell 兼容：注释明确说"alias 标准化留一处处理、避免双询问"
- `DEFAULT_TIMEOUT = 2 * 60 * 1000`（2 分钟），可 `Flag.MIMOCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS` 覆盖

### 6. **memory 工具的"无结果不放弃"提示**（`memory.ts:42-66`）
```typescript
if (results.length === 0) {
  return {
    title: `Memory search: 0 results`,
    output: [
      `No matches for "${args.query}".`,
      `0 results does NOT mean it was never recorded. Escalate before giving up:`,
      `1. Retry with FEWER / more distinctive terms ...`,
      `2. For a LITERAL string the tokenizer splits (URLs, ports, paths) — Grep the memory dir directly`,
      `3. For VERBATIM recall of something a summary may have glossed over — use the history tool`,
      `Widen scope progressively: session → project → global → history.`,
    ].join("\n"),
  }
}
```
**设计哲学**：LLM 拿到 0 结果时容易被"放弃"——prompt 明确告诉它"扩大范围 + 换工具"。

### 7. **memory-path-guard 防越权**（`memory-path-guard.ts`）
- 子 agent（特别是 checkpoint-writer）想 edit 记忆文件时
- 检查路径是否在 `Memory.root()` 子树内
- 不在 → 拒；是 → 跳过 permission ask（memory write 是合法的）
- 解决 `02-agents.md` §3 提到的 "fork 时继承 parent permission 冲突" 问题

### 8. **plugin tool 扩展点**（`registry.ts`）
```typescript
import { type ToolContext as PluginToolContext, type ToolDefinition } from "@mimo-ai/plugin"
// ...
for (const pluginTool of plugin.tools()) {
  // 包装成 Tool.Info 注册进 registry
}
```
- plugin 包可以提供自定义 tool
- `ToolDefinition` schema 限定了 plugin tool 的形状

### 9. **MCP 工具动态合并**（`prompt.ts:770-820`）
```typescript
for (const [key, item] of Object.entries(yield* mcp.tools())) {
  const execute = item.execute
  if (!execute) continue
  const schema = yield* Effect.promise(() => Promise.resolve(asSchema(item.inputSchema).jsonSchema))
  const transformed = ProviderTransform.schema(input.model, schema)
  // ...
  tools[key] = tool({ ...item, inputSchema: jsonSchema(transformed) })
}
```
- 调 `mcp.tools()` 拿当前连接的 MCP server 提供的所有 tool
- 走 `ProviderTransform.schema()` 归一化（不同 provider 接受不同 JSON schema 格式）
- **MCP 工具必须 `ctx.ask()` 先过 permission**——比 native tool 更严

### 10. **truncate 工具（大输出截断）**（`truncate.ts`）
- LLM 调某个工具，输出超过阈值 → 自动写到 truncation 目录
- 给 LLM 返回"已截断，完整内容在 `~/.local/share/mimocode/truncate/<hash>`"
- 解决"模型卡上下文"问题

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| Tool 数 | ~25 | **35** | + actor / task / workflow / plan / memory / memory-path-guard / history / mcp-exa / invalid / change-directory / session-cwd / shell-tokenize / shell-wrap / truncation-dir / apply_patch / multiedit |
| Invocation Style | 仅 JSON | **JSON + Shell 双形态** | `resolveInvocationStyle` 决定 |
| Memory 工具 | 无 | **`memory` + `memory-path-guard`** | FTS5 搜索 + 防越权 |
| Task 工具 | 无 | **`task`**（绑定 TaskRegistry） | 树状 ID 任务 |
| Actor 工具 | 无 | **`actor`**（spawn/wait/cancel） | 子 agent 控制 |
| Workflow 工具 | 无 | **`workflow`**（+ `renderWorkflowCatalog`） | 内置 workflow 列表 |
| Bash 危险预检 | 简单 | **FILES + FLAGS + SWITCHES 三套集合** | 提前询问 PowerShell / Unix |
| MCP 强制 permission | 无 | **所有 MCP tool 必须 `ctx.ask()`** | 防止恶意 MCP 工具 |
| Plugin 扩展 | 基础 | **`PluginToolContext` + 严格 schema** | 标准化 |
| 0-result 引导 | 无 | **memory 工具的"无结果不放弃"** | 教 LLM 扩大范围 |
| Plan 工具 | 无 | **`plan` + `plan-enter.txt` / `plan-exit.txt`** | 切模式专用工具 |

## 关键引用

- `packages/opencode/src/tool/tool.ts:1-100` — Tool.Def / Context / Info 类型
- `packages/opencode/src/tool/registry.ts:1-50` — registry 聚合 + renderWorkflowCatalog
- `packages/opencode/src/tool/memory.ts:1-80` — memory 工具 + 0-result 引导
- `packages/opencode/src/tool/bash.ts:1-60` — FILES/FLAGS/SWITCHES 危险预检
- `packages/opencode/src/tool/invocation-style.ts` — resolveInvocationStyle
- `packages/opencode/src/session/prompt.ts:582-825` — resolveTools 聚合逻辑
- `packages/opencode/src/session/prompt.ts:770-820` — MCP 工具合并
- `packages/opencode/src/tool/memory-path-guard.ts` — 越权保护

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 一共 35 个 tool 文件，按几大类组织？每类举 1-2 个例子？
- [ ] Tool.Def 的强类型契约是什么？`parameters` 和 `shell` 字段分别起什么作用？
- [ ] `resolveInvocationStyle` 决定 JSON 还是 Shell 形态的依据是什么？哪个工具**没有** shell 形态？
- [ ] bash 工具的"危险命令预检"用哪三套集合提前询问 permission？
- [ ] memory 工具在 0 结果时给 LLM 的 prompt 引导是什么？
- [ ] `memory-path-guard` 解决了什么子 agent 越权问题？
- [ ] MCP 工具为什么必须 `ctx.ask()` 先过 permission？和 native tool 的 permission 处理有什么不同？
