# 04 · 主 ReAct 循环 + Goal 停判

## 一句话定位
`session/prompt.ts`（3355 行）实现了 MiMo-Code 的**单条消息 → 多次 LLM call → 工具执行 → 反馈**的 ReAct 循环中枢——它把 Agent、Provider、Tool、Memory、Task、Actor、Permission 全部串起来，是整个项目最复杂的文件；`session/goal.ts` 在此基础上加了**Goal 停止条件**和**独立裁判模型**，防止 LLM 自主工作时的"乐观停止"。

## 涉及文件
- `packages/opencode/src/session/prompt.ts` (3355 行) — **核心 ReAct 循环**（已读前 800 行）
- `packages/opencode/src/session/goal.ts` — Goal 状态机 + 独立裁判模型
- `packages/opencode/src/session/auto-dream.ts` — Dream/Distill 自动触发判断
- `packages/opencode/src/session/llm.ts` — LLM 流式调用封装
- `packages/opencode/src/session/system.ts` — system prompt 拼装
- `packages/opencode/src/session/instruction.ts` — instruction 注入
- `packages/opencode/src/session/prompt/build-switch.txt` — build 模式切换 system-reminder
- `packages/opencode/src/session/prompt/max-steps.txt` — 最大步数 system-reminder
- `packages/opencode/src/session/prompt/compose.txt` — compose 模式 system-reminder
- `packages/opencode/src/session/prompt/{anthropic,beast,codex,copilot-gpt-5,default,gemini,gpt,kimi,trinity}.txt` — per-provider system 模板
- `packages/opencode/src/session/llm-request-prefix.ts` — LLM request prefix 构造
- `packages/opencode/src/session/processor.ts` — message 状态管理
- `packages/opencode/src/session/retry.ts` — 可重试错误判定
- `packages/opencode/src/session/compaction.ts` — 上下文压缩
- `packages/opencode/src/session/checkpoint.ts` — checkpoint-writer 调度

## 核心数据流

### ReAct 循环主流程
```
用户键入消息 → Session.Prompt 入口
  ↓
1. resolvePromptParts() — 解析 @file / @agent 引用
2. ensureTitle() — 第一次消息时调 "title" subagent 生成标题
3. predict() — 预测用户下一条消息（TUI 占位用）
4. 构造 message 持久化（user / assistant）
  ↓
5. 【OUTER LOOP】loop()  — 最多 MAX_GOAL_REACT (12) 次
   ↓
   5.1 buildLLMRequestPrefix() — 拼装 system + tools + messages-to-watermark
   5.2 llm.stream() — 流式调用 LLM
   5.3 解析返回：text / tool_calls / reasoning
   5.4 工具执行（带 permission 询问、whitelist 检查、bus metrics 发布）
   5.5 如果有 goal — 调 Goal.judge() 让独立模型判停
        ├─ ok=true → 退出
        ├─ impossible=true → 退出（标 impossible）
        └─ ok=false → 继续（re-enter OUTER LOOP）
   ↓
6. 返回 final assistant message
```

### Goal 停判（`goal.ts`）
```
用户在主 session 调 /goal "条件"
  ↓
Goal.set(condition) — 存到 InstanceState
  ↓
每次主 agent 想 stop：
  ├─ 无 goal → 正常 stop
  └─ 有 goal → 调 Judge
       ↓
       独立模型（复用同一个 provider 但 smallModel）生成 Verdict:
         { ok: bool, impossible?: bool, reason: string }
       ↓
       ok=true → 正常 stop
       ok=false → 重新进 OUTER LOOP（最多 MAX_GOAL_REACT=12）
       impossible=true → 正常 stop，标 impossible
```

### 关键常量（`prompt.ts:97-100`）
```typescript
const MAX_GOAL_REACT = 12                  // 主 loop 最大 re-entry
const REPEATED_STEP_THRESHOLD = 3          // 连续 3 步相同 tool_call → 推 nudge
```

## 关键设计

### 1. **Prompt Parts 解析**（`prompt.ts:308-360`）
```typescript
const resolvePromptParts = Effect.fn("SessionPrompt.resolvePromptParts")(function* (template: string) {
  const ctx = yield* InstanceState.context
  const parts: PromptInput["parts"] = [{ type: "text", text: template }]
  const files = ConfigMarkdown.files(template)   // 正则 @filename 提及
  const seen = new Set<string>()
  yield* Effect.forEach(files, ..., { concurrency: "unbounded" })
})
```
- 用户消息里的 `@file.ts` / `@~/notes.md` / `@agent-name` 被解析成 part
- 提到 agent 名 → 推 `{ type: "agent" }` part（让 LLM 知道这个 agent 存在）
- 提到文件路径 → 推 `{ type: "file", url: fileURL }` part
- `concurrency: "unbounded"` 并行 stat

### 2. **Title 自动生成**（`prompt.ts:362-419`）
- 触发：第一次 user message 且 session.title 还是默认（"New session"）
- 用 **"title" subagent**（modelRef 走 small model，temperature=0.5）
- prompt = `"Generate a title for this conversation:\n"` + 历史
- 流式拼接 → 清洗掉 `<think>...</think>` → 取第一行非空 → 截断 100 字符
- 写回 session.title

### 3. **predict() 预测下一条消息**（`prompt.ts:421-499`）
- TUI 用：在用户发完当前消息后，预测**他下一条**会发什么
- 复用 title agent 的 model 配置但换 prompt 为 `PREDICT_SYSTEM`
- Bail 条件：assistant 还没完成 / 找不到 user message
- 截断 120 字符

### 4. **Reminders 注入**（`prompt.ts:501-580`）
- 三个 reminders：
  1. **compose mode**：第一次进入 compose 模式时插入 `PROMPT_COMPOSE` + 当前 skill 列表
  2. **build 切换**：从 plan → build 模式时插入 `BUILD_SWITCH` 提示 plan file 已存在
  3. **plan 进入**：从 build → plan 模式时插入完整 plan 模式 system-reminder
- 用 `synthetic: true` 标记是自动注入（不算 user 实际发言）

### 5. **resolveTools() 工具解析**（`prompt.ts:582-825`）
```typescript
const resolveTools = Effect.fn("SessionPrompt.resolveTools")(function* (input) {
  // 1. per-actor whitelist（subagent 受 actor.tools 限制）
  const whitelist = yield* whitelistFor()
  const rejectionFor = (toolID: string) => ({ ...rejection output... })

  // 2. context 工厂：每次 tool.execute 都有 metadata / ask / abortSignal
  const context = (args, options) => ({
    sessionID, abort, messageID, callID,
    extra: { model, bypassAgentCheck, promptOps },
    agent: input.agent.name,
    actorID: input.agentID,
    taskId: input.task_id,
    messages: input.messages,
    metadata: (val) => /* 更新 tool call 状态 */,
    ask: (req) => permission.ask({ ..., interactive: !SYSTEM_SPAWNED_AGENT_TYPES.has(input.agent.name) })
  })

  // 3. 遍历 ToolRegistry.tools() + MCP.tools()
  for (const item of registry.tools({ ... })) {
    tools[item.id] = tool({
      description: item.description,
      inputSchema: jsonSchema(ProviderTransform.schema(input.model, z.toJSONSchema(item.parameters))),
      execute(args, options) { /* whitelist check + execute + metrics */ }
    })
  }
})
```
**关键点**：
- **`SYSTEM_SPAWNED_AGENT_TYPES` 白名单**：`checkpoint-writer / dream / distill` 没有人类可回答，permission ask 立刻失败
- **per-actor tool whitelist**：subagent 被 actor.tools 限制（继承 parent 时为空 = 全部允许）
- **whitelist 拒绝响应**：`{ title, output, metadata: { rejected: true, reason: "tool-whitelist" } }`——LLM 看到 reason 后会切换工具
- **MCP 工具**：走 `ctx.ask()` 先过 permission 再调 `execute()`

### 6. **稳定 step signature**（`prompt.ts:106-145`）
```typescript
function stableStringify(value: unknown): string {
  if (value === null || typeof value !== "object") return JSON.stringify(value) ?? "null"
  if (Array.isArray(value)) return "[" + value.map(stableStringify).join(",") + "]"
  const keys = Object.keys(value as Record<string, unknown>).sort()  // 关键：排序
  return "{" + keys.map((k) => JSON.stringify(k) + ":" + stableStringify(...)).join(",") + "}"
}

function stepSignature(parts: MessageV2.Part[]): string | undefined {
  // 只看 tool 部分的 name + input；忽略 text 和 reasoning
  // 返回 undefined 当 step 无 tool call
  return segments.join("\n")
}
```
- **目的**：检测 LLM 是否"原地打转"——连续 3 步相同 tool_call signature → 推 nudge
- **为什么 stableStringify**：模型对同一 input 可能 emit 键序不同的 JSON，必须归一化

### 7. **JSON 模式输出**（`prompt.ts:155-162`）
```typescript
const STRUCTURED_OUTPUT_DESCRIPTION = `Use this tool to return your final response in the requested structured format...`
const STRUCTURED_OUTPUT_SYSTEM_PROMPT = `IMPORTANT: The user has requested structured output. You MUST use the StructuredOutput tool...`
```
- 当用户要求结构化输出时，注入 `StructuredOutput` 工具
- TUI 用于在聊天里展示富表格

### 8. **Prefix Capture 懒绑定**（`prompt.ts:230-275`）
```typescript
// 注释明确说：避免 ToolRegistry → SessionCheckpoint → ToolRegistry 形成 layer cycle
// 解决：prompt.ts 自己实现 capture，存到 module-level prefixCaptureRef.current
// SessionCheckpoint.tryStartCheckpointWriter 调用时 capture(input) 拿 prefix
const capture: typeof prefixCaptureRef.current = (input) =>
  Effect.gen(function* () {
    const empty = { system: [], tools: {}, inheritedMessages: [], parentPermission: [] }
    const ag = yield* agents.get(input.agentName).pipe(Effect.catch(() => Effect.succeed(undefined)))
    if (!ag) return empty
    const model = yield* provider.getModel(...).pipe(Effect.catch(() => Effect.succeed(undefined)))
    if (!model) return empty
    const [skills, env, instructions] = yield* Effect.all([...])
    const prefix = yield* buildLLMRequestPrefix({...}).pipe(...)
    return { ...prefix, parentPermission: ag.permission }
  })
prefixCaptureRef.current = capture
yield* Effect.addFinalizer(() => Effect.sync(() => {
  if (prefixCaptureRef.current === capture) prefixCaptureRef.current = undefined
}))
```
- **核心问题**：Effect Layer 不允许循环依赖（`A → B → A`）
- **解决**：用 module-level 引用 + Effect finalizer 清理

### 9. **RUNNER 抽象**（`prompt.ts:283-300`）
```typescript
const runner = Effect.fn("SessionPrompt.runner")(function* () {
  return yield* EffectBridge.make()  // 返回 { promise, fork } shape
})
const ops = Effect.fn("SessionPrompt.ops")(function* () {
  const run = yield* runner()
  return {
    cancel: (sessionID) => run.fork(cancel(sessionID)),
    resolvePromptParts: (template) => resolvePromptParts(template),
    prompt: (input) => prompt(input),
  } satisfies ActorPromptOps
})
```
- `ActorPromptOps` 暴露给子 agent 用
- `EffectBridge.make()` 在 `effect/bridge.ts`：用 `InstanceRef` / `WorkspaceRef` 注入

### 10. **Goal 独立裁判**（`goal.ts`）
- `Judge` prompt：要求引用 transcript 中**具体证据**——不是凭印象判断
- `Verdict` schema：`{ ok: bool, impossible?: bool, reason: string }`
- 状态：`Goal { condition: string; react: number }`——react 计数受 `MAX_GOAL_REACT=12` 限制
- bus event `session.goal`：TUI 监听渲染"active goal indicator"
- 关键设计：**裁判只读 transcript，不做工作**——所以"cold"、不会被工作 agent 的乐观态度感染

### 11. **Auto-Dream / Auto-Distill**（`auto-dream.ts`）
- `shouldAutoRun({ enabled, intervalDays, title, label })`：
  - 查 session 表里 `title = "Auto Dream"` / `"Auto Distill"` 的最近一条
  - `now - lastRun > intervalDays * DAY_MS` 才返回 true
- 默认间隔：dream=7 天、distill=30 天
- **MIN_SPAWN_GAP_MS = 10_000**：两次 spawn 至少间隔 10s（避免误触发刷屏）
- `DREAM_TASK` / `DISTILL_TASK` 是 spawn 时给子 agent 的 prompt 模板

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 循环控制 | 普通 for-await | **MAX_GOAL_REACT=12 OUTER LOOP** | Goal 停判的安全阀 |
| 停判机制 | LLM 自决 | **独立 judge 模型** + 自决双轨 | 防止乐观停止 |
| Title 生成 | 用同一 agent | **专用 title subagent**（hidden, temperature=0.5） | 不污染主 agent |
| Predict | 无 | **复用 title subagent 改 prompt** | TUI 占位用 |
| Compose 模式 | 无 | **`compose` agent + `PROMPT_COMPOSE` + composeSkillsBlock()** | 整段 system-reminder 注入 |
| Plan 切换 | 无 | **`BUILD_SWITCH` reminder** | 退出 plan → build 时提示 plan 文件位置 |
| Stable step signature | 无 | **stableStringify + stepSignature** | 检测原地打转 |
| Whitelist 拒绝 | 无 | **`{ title, output, metadata: { rejected, reason: "tool-whitelist" } }`** | LLM 收到 reason 后可换工具 |
| SYSTEM_SPAWNED_AGENT_TYPES | 无 | **`{ checkpoint-writer, dream, distill }`** 强制 `interactive: false` | 无人值守 agent 不卡死 |
| Prefix capture 懒绑定 | 无 | **module-level ref + finalizer** | 破 Effect Layer 循环依赖 |
| Persistent-retry | Schedule.recurs(5) | **Schedule.recurs(10) + 5min cap** | 见 `03-llm.md` |
| Auto-dream / distill | 无 | **间隔检查 + 标题查 session 表** | 自进化能力 |

## 关键引用

- `packages/opencode/src/session/prompt.ts:1-77` — 顶部常量 + imports
- `packages/opencode/src/session/prompt.ts:97-100` — MAX_GOAL_REACT + REPEATED_STEP_THRESHOLD
- `packages/opencode/src/session/prompt.ts:106-145` — stableStringify + stepSignature
- `packages/opencode/src/session/prompt.ts:230-275` — prefix capture 懒绑定
- `packages/opencode/src/session/prompt.ts:283-300` — RUNNER / ops 抽象
- `packages/opencode/src/session/prompt.ts:362-419` — ensureTitle
- `packages/opencode/src/session/prompt.ts:501-580` — insertReminders
- `packages/opencode/src/session/prompt.ts:582-825` — resolveTools + whitelist
- `packages/opencode/src/session/goal.ts:1-80` — Judge prompt + Verdict schema
- `packages/opencode/src/session/auto-dream.ts:1-60` — shouldAutoRun 间隔逻辑

## 学习检查清单

读完后应能回答：
- [ ] ReAct 循环的 OUTER LOOP（`MAX_GOAL_REACT=12`）什么时候 re-enter？什么时候真正停？
- [ ] Goal 独立裁判模型和 LLM 自决双轨制如何协同？为什么裁判要"只读 transcript"？
- [ ] `stepSignature` 为什么用 `stableStringify`？它解决了什么具体问题？
- [ ] `SYSTEM_SPAWNED_AGENT_TYPES = { checkpoint-writer, dream, distill }` 在 permission ask 时怎么 `interactive: false`？
- [ ] Compose 模式进入时的 system-reminder 注入流程？
- [ ] `prefix-capture-ref` 懒绑定怎么破 Effect Layer 循环依赖？
- [ ] Auto-dream 默认间隔 7 天、Auto-distill 默认间隔 30 天，触发条件是查 session 表的什么字段？
