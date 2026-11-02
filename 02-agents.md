# 02 · Agent 体系

## 一句话定位
MiMo-Code 用"3 主 + 7 隐藏 subagent"的多 agent 模型替代 OpenCode 的"单 build agent"——3 个用户可切换的主 agent（build/plan/compose）覆盖开发/分析/编排三大场景，7 个 subagent 在后台被自动唤起，专门负责标题生成、会话摘要、上下文压缩、checkpoint 写入、dream 整理、distill 提炼、自适应任务完成检查等单一职责。

## 涉及文件
- `packages/opencode/src/agent/agent.ts` (554 行) — **所有 agent 的唯一注册中心**（已读全文）
- `packages/opencode/src/agent/config.ts` (21 行) — `SYSTEM_SPAWNED_AGENT_TYPES = { "checkpoint-writer", "dream", "distill" }`（runtime 主动 spawn 的白名单）
- `packages/opencode/src/agent/prompt/` — 7 个 prompt 文件
  - `checkpoint-writer.txt` (167 行)
  - `compaction.txt` (9 行)
  - `distill.txt` (199 行)
  - `dream.txt` (155 行)
  - `explore.txt` (18 行)
  - `summary.txt` (11 行)
  - `title.txt` (44 行)
- `packages/opencode/src/agent/generate.txt` — `/general` 命令的 prompt
- `packages/opencode/src/agent/schema.ts` — `Agent.Info` zod schema

## 核心数据流

### Agent 类型矩阵

| Agent | mode | hidden | 触发方式 | 工具白名单 | 主要权限 |
|-------|------|--------|----------|------------|----------|
| **build** | primary | ❌ | 用户 Tab 切换 / 默认 | 全工具 | 全部 allow + ask（doom_loop/external/*.env） |
| **plan** | primary | ❌ | 用户 Tab 切换 | 全工具但 edit deny | 全部 deny edit，只能写到 plans/*.md |
| **compose** | primary | ❌ | 用户 Tab 切换 | 全工具 + skill allow | 编排 skill 驱动流程 |
| **max** | primary | ❌ | `experimental.maxMode` 开启时出现 | 全工具 | 同 build（实验性） |
| general | subagent | ❌ | 主 agent `task` 工具调用 | 全工具（无 change_directory） | 默认 + user |
| explore | subagent | ❌ | 主 agent `task` 工具调用 | grep/glob/list/bash/webfetch/websearch/codesearch/read | 全部 deny 编辑 |
| **title** | subagent | ✅ | 每次 user message 后 | 空（`toolAllowlist: []`） | 全部 deny |
| **summary** | subagent | ✅ | 会话结束 / checkpoint 触发 | 空 | 全部 deny |
| **compaction** | subagent | ✅ | 上下文即将溢出 | 空 | 全部 deny |
| **checkpoint-writer** | subagent | ✅ | **runtime 自动 spawn**（见 SYSTEM_SPAWNED） | **空 + 继承 parent 工具 schema** | `defaults + user`，无独立 block（设计见下） |
| **dream** | subagent | ✅ | runtime 自动 spawn | read/write/edit/glob/grep/memory/bash | 写到 `~/.local/share/mimocode/memory/` |
| **distill** | subagent | ✅ | runtime 自动 spawn | read/write/edit/glob/grep/memory/bash | 写到 `~/.local/share/mimocode/memory/` |

### 数据流

```
用户键入 "修复登录页 bug"
   ↓
[主 agent: build] 读到 prompt → 决定用 explore subagent 找代码
   ↓
spawn(actor, agent="explore", contextMode="fork", background=false)
   ↓
explore subagent 跑 ReAct 循环（受 toolAllowlist 限制为只读）
   ↓
返回 "**Status**: success\n**Summary**: 找到 login.tsx 第 42 行..."
   ↓
[主 agent: build] 用代码定位工具
   ↓
…（循环）…
   ↓
checkpoint-writer subagent（runtime 在 token 阈值时自动 spawn）
   ↓ 写 checkpoint.md
[主 agent: build] 继续
```

## 关键设计

### 1. **统一注册中心**（`agent.ts:111-369`）
**所有 agent 在一个 `agents` 对象字面量中定义**——不像 OpenCode 把每个 agent 拆成单独文件。这种"大字典"风格的优劣：
- **优点**：一眼看完全部 agent 矩阵、permission diff 便于审查、新增 agent 改 1 个文件
- **缺点**：单文件 554 行（含 Effect Layer 包装），大型改动易冲突

```typescript
// agent.ts:117-369 核心结构
const agents: Record<string, Info> = {
  build: { ..., mode: "primary", native: true },
  plan:  { ..., mode: "primary", native: true },
  compose: { ..., mode: "primary", native: true },
  ...(cfg.experimental?.maxMode ? { max: { ... } } : {}),
  general: { ..., mode: "subagent", native: true },
  explore: { ..., mode: "subagent", native: true },
  title:      { ..., mode: "subagent", hidden: true, toolAllowlist: [] },
  summary:    { ..., mode: "subagent", hidden: true, toolAllowlist: [] },
  compaction: { ..., mode: "subagent", hidden: true, toolAllowlist: [] },
  "checkpoint-writer": { ..., mode: "subagent", hidden: true },
  dream:    { ..., mode: "subagent", hidden: true, toolAllowlist: [...] },
  distill:  { ..., mode: "subagent", hidden: true, toolAllowlist: [...] },
}
```

### 2. **三层 permission 合成**（`agent.ts:120-128, 161-172`）
```typescript
const defaults = Permission.fromConfig({
  "*": "allow",
  doom_loop: "ask",
  external_directory: { "*": "ask", ...skill 白名单... },
  question: "deny",
  plan_enter: "deny",
  read: { "*": "allow", "*.env": "ask", "*.env.*": "ask" },
})
const user = Permission.fromConfig(cfg.permission ?? {})
// 每个 agent 的 permission:
permission: Permission.merge(defaults, Permission.fromConfig({...}), user)
```
**关键点**：`Permission.merge` 是顺序合并，**后写覆盖前写**。`user` 总在最后，保证用户配置永远赢。

### 3. **checkpoint-writer 的"零 prompt 零工具白名单"设计**（`agent.ts:285-314`）
这是一段非常关键的注释（已读）：

```typescript
"checkpoint-writer": {
  mode: "subagent",
  hidden: true,
  // 没有 prompt 字段 —— fork agent contract：
  // spawn 时 tryStartCheckpointWriter 把 parent 的 LLM request prefix
  // （system + tools + messages-to-watermark） 捕获到 ForkContext，
  // 存在 Actor 服务的内存 map 里。fork 的 runLoop 从那个快照读，
  // 而不是从该 agent identity 重新算。
  //
  // 没有 toolAllowlist —— fork agent 必须 mirror parent 的 tool schema
  // 以保 prefix-cache 对齐。运行时工具限制由 actor.tools 白名单强制
  // （在 tryStartCheckpointWriter 设置）。
  // Permission 只继承 defaults（+ user），无自定义 block ——
  // fork 的 LLM 可见 tool schema 是按 PARENT agent 的 permission 过滤的
  // （ForkContext.parentPermission 喂给 prompt.ts 的 fork 分支），
  // 所以它和 parent 一致（prompt-cache parity）。
  permission: Permission.merge(defaults, user),
}
```

**这个设计的目的**：让 fork 出去的 checkpoint-writer 和 parent 共用同一份**带缓存前缀的 LLM 请求**——省 token、省延迟、保语义一致。

### 4. **用户配置覆盖**（`agent.ts:371-402`）
```typescript
for (const [key, value] of Object.entries(cfg.agent ?? {})) {
  if (value.disable) { delete agents[key]; continue }
  // 增量更新已有 agent
  item.model = ...   // "anthropic/claude-sonnet-4" → 自动 parse
  item.temperature = value.temperature ?? item.temperature
  item.prompt = value.prompt ?? item.prompt
  item.toolAllowlist = value.tool_allowlist ?? item.toolAllowlist
  item.permission = Permission.merge(item.permission, Permission.fromConfig(value.permission ?? {}))
}
```
- `disable: true` → 移除（不是 disable 标记）
- 顶层 config 的 `agent:` 配置覆盖 native agent
- 任意名字的自定义 agent 会**新建**（`mode: "all"`）

### 5. **Truncate.GLOB 强制 allow**（`agent.ts:404-418`）
```typescript
for (const name in agents) {
  const agent = agents[name]
  const explicit = agent.permission.some((r) =>
    r.permission === "external_directory" && r.action === "deny" && r.pattern === Truncate.GLOB
  )
  if (explicit) continue
  agents[name].permission = Permission.merge(
    agents[name].permission,
    Permission.fromConfig({ external_directory: { [Truncate.GLOB]: "allow" } }),
  )
}
```
**含义**：除非 agent 显式 deny 截断目录（`Truncate.GLOB`），否则一律 allow——保证 memory 写入不受 external_directory 限制。

### 6. **defaultAgent 选举**（`agent.ts:440-453`）
```typescript
const defaultAgent = Effect.fnUntraced(function* () {
  const c = yield* config.get()
  if (c.default_agent) {
    const agent = agents[c.default_agent]
    if (agent.mode === "subagent") throw new Error(...)
    if (agent.hidden === true) throw new Error(...)
    return agent.name
  }
  const visible = Object.values(agents).find((a) => a.mode !== "subagent" && a.hidden !== true)
  return visible.name
})
```
**优先级**：用户 `default_agent` 配置 > 第一个 `primary + !hidden` agent。

### 7. **Agent.generate：用 LLM 生成新 agent 配置**（`agent.ts:483-538`）
- 调用 `streamObject` / `generateObject` 让模型输出 `{ identifier, whenToUse, systemPrompt }`
- 系统 prompt = `PROMPT_GENERATE`（`agent/generate.txt`）
- 用户消息 = `"Create an agent configuration based on this request: \"${input.description}\""` + 已存在 agent 名（防重名）
- OpenAI OAuth 走 `streamObject` + `providerOptions.instructions` 走 system

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| Agent 数量 | 1 主 + 2 隐藏 (title/summary) | **3 主 + 5 隐藏** | 多出来的 plan/compose/max/checkpoint-writer/compaction/dream/distill |
| 可见性 | 用 mode: "subagent" 区分 | **+ `hidden: true`** | hidden 的 subagent 不出现在 `list()` |
| 自定义 agent | 不支持 | **支持**（`cfg.agent.<name>`） | 可用 disable 移除 native |
| 默认 agent | 写死 build | **可配置**（`cfg.default_agent`） | + 校验 primary + !hidden |
| agent 生成 | 无 | **LLM-driven `Agent.generate`** | 用 LLM 自动产出新 agent 配置 |
| Permission 合成 | 单一字典 | **`Permission.merge(defaults, specific, user)`** | 三段式合并、user 总在最后 |
| max 模式 | 无 | **`experimental.maxMode` 开关** | 跑 N 路并行推理候选 + 选最佳（实验性） |
| fork 协议 | 无 | **checkpoint-writer 零 prompt 零工具白名单** | 复用 parent LLM prefix cache |

## 关键引用

- `packages/opencode/src/agent/agent.ts:117-369` — 全部 8 个 native agent 定义
- `packages/opencode/src/agent/agent.ts:285-314` — checkpoint-writer 的 fork protocol 注释
- `packages/opencode/src/agent/agent.ts:440-453` — defaultAgent 选举
- `packages/opencode/src/agent/agent.ts:483-538` — `Agent.generate` LLM-driven 创建
- `packages/opencode/src/agent/config.ts:11-12` — `SYSTEM_SPAWNED_AGENT_TYPES`
- `packages/opencode/src/agent/prompt/checkpoint-writer.txt:1-50` — 写入 11 段式 checkpoint 的指令
- `packages/opencode/src/agent/prompt/dream.txt:1-50` — dream 的两阶段 prompt

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 一共注册了几个 native agent？哪些是 primary、哪些是 subagent？
- [ ] checkpoint-writer 为什么"没有 prompt 字段、没有 toolAllowlist"？这个设计的目的是什么？
- [ ] Permission 的三层 merge 顺序是什么？为什么 user 总在最后？
- [ ] `experimental.maxMode` 开启时会多注册哪个 agent？它的语义是什么？
- [ ] 如何用 config 注册一个自定义 agent 并 disable 一个 native agent？
