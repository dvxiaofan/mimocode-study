# 06 · 持久化记忆

## 一句话定位
MiMo-Code 的**核心差异化**——用 4 类记忆文件（`MEMORY.md` / `checkpoint.md` / `notes.md` / `tasks/<id>/progress.md`）+ SQLite FTS5 全文索引 + **checkpoint-writer 子 agent 自动维护**机制，让 agent 在跨会话、跨上下文压缩、跨任务交接时都能恢复工作上下文；这是 OpenCode 完全没有的能力。

## 涉及文件
- `packages/opencode/src/memory/service.ts` (已读) — `Memory.Service` 提供 `root() / reconcile() / search()` Effect API
- `packages/opencode/src/memory/fts-query.ts` (已读) — `buildFtsQuery` 把自由文本转 FTS5 MATCH 表达式
- `packages/opencode/src/memory/paths.ts` (已读) — 路径解析 / 类型检测 / CC 兼容
- `packages/opencode/src/memory/reconcile.ts` (已读) — 扫盘 → 入库
- `packages/opencode/src/memory/index.ts` — 1 行 re-export
- `packages/opencode/src/memory/fts.sql.ts` (已读) — `memory_fts` 表 schema
- `packages/opencode/src/session/checkpoint.ts` (1478 行) — **checkpoint-writer 系统核心**（已读 100-400 行）
- `packages/opencode/src/session/checkpoint-paths.ts` — `metaDir / checkpointPath / memoryPath / notesPath / globalMemoryPath` 路径工具
- `packages/opencode/src/session/checkpoint-templates.ts` — 11 段式 checkpoint 模板 + section budgets
- `packages/opencode/src/session/checkpoint-context.ts` — 上下文重建逻辑
- `packages/opencode/src/session/checkpoint-align.ts` — 对齐到 non-tool-result user 消息
- `packages/opencode/src/session/checkpoint-retry.ts` — 重试 + 已发现标题去重
- `packages/opencode/src/session/checkpoint-validator.ts` — 11 段式格式校验
- `packages/opencode/src/session/checkpoint-progress-reconcile.ts` — 任务进度 diff
- `packages/opencode/src/session/budgeted-read.ts` — token-budget 读文件
- `packages/opencode/src/session/boundary.ts` — `adjustBoundaryForApiInvariants` 工具
- `packages/opencode/src/tool/memory.ts` — `memory` 工具（BM25 搜索）
- `packages/opencode/src/tool/memory-path-guard.ts` — 路径白名单保护

## 核心数据流

### 4 类记忆文件
```
~/.local/share/mimocode/memory/   (Memory.root() = Global.Path.data + "/memory")
├── global/
│   └── MEMORY.md          ← 跨项目用户偏好（用户名: <user> 等）
├── projects/
│   └── <projectID>/
│       ├── MEMORY.md      ← 项目级持久知识（架构决策、规则）
│       └── tasks/
│           └── <taskID>/
│               ├── progress.md   ← 任务进度
│               └── notes.md      ← 任务笔记
└── sessions/
    └── <sessionID>/
        ├── checkpoint.md  ← 会话结构化状态（11 段式）
        └── notes.md       ← 会话临时记录
```

### 写入路径
```
用户在主 session 编辑文件 / 完成子任务
  ↓
checkpoint-writer subagent（runtime 自动 spawn）被触发：
  - 触发条件：token 使用超过阈值 / session 即将关闭
  - 调度：SessionCheckpoint.tryStartCheckpointWriter()
  - 协议：复用 parent LLM request prefix（见 02-agents.md §3）
  ↓
checkpoint-writer 读取：
  - 对话历史
  - 现有 checkpoint.md / MEMORY.md / notes.md
  - 任务的 tasks/<id>/progress.md
  ↓
writer 用 LLM 把内容按 11 段式 §1-§11 重组写回 checkpoint.md
  ↓
把跨任务持久的事实 promote 到 MEMORY.md（项目级）
  ↓
side effect：tasks/<id>/progress.md 由 writer 整合（write-derived splitover）
```

### 读取路径
```
主 agent 想搜索记忆
  ↓
调 `memory` 工具 { operation: "search", query: "..." }
  ↓
Memory.Service.search()
  ├─ 可选：reconcile() — 扫盘补 FTS 索引（覆盖 off-tool 写）
  ├─ buildFtsQuery() — token-level 短语化 + OR-join
  ├─ 跑 FTS5 MATCH + BM25 排序
  ├─ 应用 score floor（top hit × 0.X，避免 BM25 corpus-size-dependent 误伤）
  └─ 返回 top N：[{ path, snippet, score, scope, scope_id, type }]
  ↓
LLM 拿到结果（snippet 截断；可 Read 全文）
```

### 上下文压缩时的 Rebuild
```
LLM 上下文接近上限
  ↓
computeBoundary() — 选保留 tail 的起点
  ├─ 起点 = 最近完成 assistant 索引 - 1
  ├─ 算 tail token 数（估）
  ├─ < TAIL_MIN_TOKENS (10k) 或 < TAIL_MIN_TEXT_BLOCK_MESSAGES (5) → 往前扩
  ├─ > TAIL_MAX_TOKENS (20k) → 不动（避免断 tool_use/tool_result pair）
  └─ 用 Token.estimate() 估每条 message token
  ↓
adjustBoundaryForApiInvariants() — 调边界避开 tool_use/tool_result 拆对 + thinking-block 拆块
  ↓
checkpoint-writer spawn（带 prefix cache + ForkContext）
  ↓
rebuild 上下文 = checkpoint.md 内容 + MEMORY.md budgeted 节选 + tail (boundary 之后)
  ↓
LLM 在 rebuild 上下文里继续工作（感觉"什么都没变"，实际是从 snapshot 恢复）
```

## 关键设计

### 1. **FTS5 查询构造**（`fts-query.ts:22-50`）
```typescript
export function buildFtsQuery(raw: string): string | null {
  const tokens = raw.match(/[\p{L}\p{N}_]+/gu)?.map(t => t.trim()).filter(Boolean) ?? []
  if (tokens.length === 0) return null
  const quoted = tokens.map(t => `"${t.replaceAll('"', "")}"`)
  return quoted.join(" OR ")
}
```
- **Unicode 感知**：`\p{L}` 包含 CJK
- **短语化包裹**：`"token"` 防止 FTS5 特殊字符（`*` `:` `(` `)` `^` 等） crash parser
- **OR-join 而非 AND**：注释明确说 AND 几乎 0 结果，OR 配 BM25 ranking + score floor 实际更精准
- 移植自 openclaw 仓库（`extensions/memory-core/src/memory/hybrid.ts:30`）

### 2. **Score Floor（相对阈值）**（`service.ts:60-80`）
```typescript
// OR-join means a doc matching only a common word (e.g. every
// checkpoint.md matches "checkpoint") still matches, but BM25 ranks it
// far below a doc matching several rare query words. We drop the
// common-word noise with a RELATIVE floor: keep results scoring at
// least `ratio` of the top hit's score.
```
- **不能用绝对 floor**：BM25 数值 corpus-size dependent
- **小 corpus 全部 score → 0**：绝对 floor 会全砍
- **相对 floor**：`top_score * ratio`（注释在 `service.ts:60-80`）

### 3. **MemoryType 自动检测**（`paths.ts:23-40`）
```typescript
const TYPE_PATTERNS = [
  { match: /^memory$/i, type: "memory" },        // 大小写不敏感
  { match: /^memory-/i, type: "memory" },
  { match: /^checkpoint$/, type: "checkpoint" },
  { match: /^checkpoint-/, type: "checkpoint" },
  { match: /^tasks\/[^/]+\/progress$/, type: "progress" },
  { match: /^tasks\/[^/]+\/notes$/, type: "notes" },
]
function detectType(key: string): MemoryType { ... }
```
- **`memory` 大小写不敏感**：迁移 `memory.md → MEMORY.md` 期间必须桥接
- **`checkpoint` / `tasks` 大小写敏感**：注释说"如果 writer 漂到 `CHECKPOINT.md` 不应静默归类为 checkpoint"
- 文件名 → type 的硬编码规则

### 4. **CC (Claude Code) 兼容**（`paths.ts:48-65`, `reconcile.ts:40-70`）
- 路径模式：`/.claude/projects/<slug>/memory/<key>.md`
- `CC_TYPES = ["feedback", "project", "reference", "user"]`
- `parseCcFrontmatterType(body)` 读 YAML frontmatter 判定 type
- 用户开 `mimocode.json: { memory: { cc_index: true } }` 启用

### 5. **reconcile() 双向同步**（`reconcile.ts:60-90`）
```typescript
export async function indexFromDisk(absPath, loc, bodyType, oldFingerprint): Promise<"hit" | "updated" | "skipped"> {
  const stat = await fs.stat(absPath)
  const fingerprint = `${stat.size}-${stat.mtimeMs}`  // mtime+size 哈希
  if (oldFingerprint === fingerprint) return "hit"     // 没变
  // ... 读 body → upsert FTS
}
```
- **触发**：
  - 启动时（bootstrap）
  - `Memory.search()` 之前（lazy，cover off-tool writes）
  - 可由 `mimocode.json: { checkpoint: { memory_reconcile_on_search: false } }` 关掉 lazy reconcile
- **fingerprint 增量更新**：`stat.size - stat.mtimeMs` 比 hash 整个文件快
- **`hit` / `updated` / `skipped`** 三态返回

### 6. **11 段式 Checkpoint**（`checkpoint-templates.ts` + `checkpoint-writer.txt`）
| 段号 | 标题 | 用途 |
|------|------|------|
| §1 | Active intent | 用户原始请求 block-quote |
| §2 | Next concrete action | 下一步具体动作（含 verbatim quote） |
| §3 | Directives (this session) | session-specific working style |
| §4 | Task tree | 从 task 工具拉的树（🔵/🔄/🟡/✅/❌） |
| §5 | Current work | checkpoint 之前在做什么 |
| §6 | Files and code sections | 读/写的文件 + 用途 |
| §7 | Discovered knowledge (cross-task) | 跨任务事实（promote candidate） |
| §8 | Errors and fixes | 错误 + 修复 |
| §9 | Live resources | runtime 状态（branch / processes） |
| §10 | Design decisions and discussion outcomes | 设计决策 |
| §11 | Open notes | writer-curated 杂项 |

### 7. **Token-Budget 边界选择**（`checkpoint.ts:106-200`）
```typescript
const TAIL_MIN_TOKENS = 10_000
const TAIL_MAX_TOKENS = 20_000
const TAIL_MIN_TEXT_BLOCK_MESSAGES = 5
```
**算法**（`computeBoundary`）：
1. 起点 = `lastAsstIdx - 1`
2. 累计 tail token
3. 已超 TAIL_MAX → 不动（避免断 tool_use/result 配对）
4. 不够 TAIL_MIN → 往前扩
5. 用 `Token.estimate(JSON.stringify(part))` 估每段

**`TAIL_MAX` 是软上限**：注释明确说"20K 是经验甜点"，超过不强制切。

### 8. **microcompact 工具白名单**（`checkpoint.ts:75-90`）
```typescript
const COMPACTABLE_TOOL_NAMES = new Set([
  "read", "bash", "grep", "glob", "webfetch", "websearch",
  "edit", "write", "multiedit", "apply_patch", "codesearch",
])
```
**规则**：rebuild 时这些 tool 的 tool_result 内容用占位符替换：
- 大且可再生成（read/bash/grep/glob/webfetch/websearch）
- 几乎是 "done" 确认（edit/write/multiedit/apply_patch）
- **actor / task / question / skill / memory** 不在白名单——这些的 tool_result 携带后续 LLM 需要的状态

### 9. **tryStartCheckpointWriter 1-slot 队列**（`checkpoint.ts:347-360`）
```typescript
export type TryStartCheckpointWriterResult = "started" | "queued" | "skipped"
```
- **"started"**：本次启动了新 writer
- **"queued"**：writer 已在跑，1-slot 队列里压一个；旧的 pending 被 evict（"newest wins"——新请求的 range 是旧的 superset）
- **"skipped"**：空 session / SYSTEM_SPAWNED subagent / Actor service 不可用 → 完全跳过

### 10. **Writer Prompt 的"绝对路径"硬约束**（`checkpoint.ts:268-295`）
```typescript
function composeWriterPrompt(input) {
  return [
    "<system-reminder>",
    "ABSOLUTE PATHS — USE THESE VERBATIM. NEVER COMPUTE, INFER, OR MODIFY.",
    `CHECKPOINT_PATH = ${input.checkpointFile}`,
    `MEMORY_PATH     = ${input.memoryFile}`,
    `TASK_MEM_DIR    = ${input.taskMemDir}`,
    `NOTES_PATH      = ${input.notesFile}`,
    "",
    "If you find yourself typing '/data/checkpoints/' as a parent, STOP —",
    "that is the legacy v2 layout and is wrong.",
    ...
  ].join("\n")
}
```
**目的**：防止 LLM 用训练数据里"看起来像"的路径（v2 layout）写错位置。

### 11. **Active Recall Protocol**（`llm.ts:78-110`）
- 注入到 main agent 的 system prompt
- 教 agent：
  - MEMORY.md 是它**自己**的 curator（project-scoped writer）
  - checkpoint.md writer EXCLUSIVE，main agent 永远不写
  - tasks/<id>/progress.md 是 writer-derived，**subagent 可以读不能写**
- 防止 main agent 重复 Read 已经进 rebuild 的文件
- 子 agent 返回格式契约（**Status** / **Summary** / **Files touched** / **Findings worth promoting**）

### 12. **CLAUDE.md → MEMORY.md 迁移**（`claude-import.ts`）
- `migration/20260608000000_claude_import` 加了 Claude Code 数据导入
- 把 `~/.claude/CLAUDE.md` 等用户级配置并入 `global/MEMORY.md`
- 让用户从 Claude Code 平滑迁到 MiMo

## 与上游 OpenCode 的差异

| 项 | OpenCode | MiMo-Code | 说明 |
|----|---------|-----------|------|
| 跨会话记忆 | 无 | **4 类文件 + FTS5 索引** | 完整自研 |
| Checkpoint 系统 | 无 | **checkpoint-writer subagent + 11 段式模板 + token-budget 边界** | 1478 行核心 |
| Dream / Distill | 无 | **`.txt` prompt + auto-dream.ts 间隔检查** | 自进化 |
| Active Recall | 无 | **注入到 main agent system prompt** | 教 agent 怎么用 |
| FTS5 score floor | 无 | **相对 top_score ratio** | 修小 corpus 误伤 |
| Token-budget 重建 | 无 | **TAIL_MIN/MAX/MIN_TEXT_BLOCK + soft cap** | 上下文压缩后无缝继续 |
| microcompact | 无 | **11 工具白名单** | 重建时压缩 tool_result |
| CC (Claude Code) 兼容 | 无 | **路径解析 + frontmatter type** | 迁移友好 |
| ABSOLUTE PATHS 硬约束 | 无 | **writer prompt 顶部 5 行 system-reminder** | 防止 LLM 写错位置 |
| 1-slot queue | 无 | **`started` / `queued` / `skipped` 三态** | 防 writer 风暴 |
| 11 段式模板 | 无 | **`CHECKPOINT_TEMPLATE` + `§1`-`§11` 严格校验** | 写一致 |

## 关键引用

- `packages/opencode/src/memory/service.ts:1-100` — root / search 接口 + score floor 注释
- `packages/opencode/src/memory/fts-query.ts:1-50` — `buildFtsQuery` + 移植说明
- `packages/opencode/src/memory/paths.ts:23-65` — TYPE_PATTERNS + CC 路径解析
- `packages/opencode/src/memory/reconcile.ts:60-90` — fingerprint 增量更新
- `packages/opencode/src/memory/fts.sql.ts:1-20` — `memory_fts` 表 + 索引
- `packages/opencode/src/session/checkpoint.ts:75-90` — COMPACTABLE_TOOL_NAMES
- `packages/opencode/src/session/checkpoint.ts:106-200` — `computeBoundary` 算法
- `packages/opencode/src/session/checkpoint.ts:268-295` — `composeWriterPrompt` 绝对路径
- `packages/opencode/src/session/checkpoint.ts:347-360` — `tryStartCheckpointWriter` 三态
- `packages/opencode/src/session/llm.ts:78-110` — Active Recall protocol
- `packages/opencode/src/tool/memory.ts:1-80` — memory 工具 + 0-result 引导
- `packages/opencode/migration/20260515010000_memory_fts/` + `20260521010000_memory_fts_v6/` — FTS 上线

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 的 4 类记忆文件分别是什么？分别放哪个 scope 下？
- [ ] `buildFtsQuery` 为什么用 OR-join 不用 AND？相对 score floor 修的什么问题？
- [ ] checkpoint-writer 是怎么被 spawn 的？触发的 token 阈值在哪里定义？
- [ ] 11 段式 checkpoint 每一段对应什么内容？
- [ ] `tryStartCheckpointWriter` 返回值 `started` / `queued` / `skipped` 各自含义？
- [ ] microcompact 工具白名单 11 个工具分类是什么？为什么 actor / task 不在里面？
- [ ] Active Recall Protocol 注入到哪一层 system prompt？教 agent 哪 3 件事？
- [ ] CC (Claude Code) 兼容做了哪两件事？
