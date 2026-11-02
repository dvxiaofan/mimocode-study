# MiMo-Code 源码学习手册

> 目标读者：希望读懂 `XiaomiMiMo/MiMo-Code`（小米 2026-06-10 开源的 AI 编程 Agent，OpenCode 的 fork）核心架构并能向他人复述其差异化设计的中高级工程师。
>
> 上游仓库：`MiMo-Code/`
> 本手册：`mimocode-study/`

## 阅读路径

学习手册分 14 章，**按推荐顺序读**效果最佳（每章都建立在前几章的概念之上）：

```
┌──────────────────────────────────────────────────────────────────────┐
│  入门三件套（必须先读，建立心智模型）                                  │
│  00-overview → 02-agents → 03-llm                                    │
├──────────────────────────────────────────────────────────────────────┤
│  基础运行时（Effect/DB 是全局底座）                                   │
│  11-effect-db                                                        │
├──────────────────────────────────────────────────────────────────────┤
│  核心循环（这里是项目灵魂）                                            │
│  04-react-loop ─┬─→ 05-tools（tool 集合）                            │
│                 ├─→ 06-memory（跨会话记忆）                           │
│                 ├─→ 07-actor（子 agent）                              │
│                 └─→ 08-task（任务追踪）                                │
├──────────────────────────────────────────────────────────────────────┤
│  上层能力                                                              │
│  12-plugin-skill                                                      │
├──────────────────────────────────────────────────────────────────────┤
│  三个入口                                                              │
│  01-cli-bootstrap → 09-tui（TUI） / 10-server（HTTP）                 │
├──────────────────────────────────────────────────────────────────────┤
│  差异化总结                                                            │
│  13-fork-diff                                                         │
└──────────────────────────────────────────────────────────────────────┘
```

## 章节索引

| 章节 | 主题 | 关键文件 | 一句话定位 |
|------|------|----------|------------|
| 00 | 项目全景 | `package.json`、`packages/*` | MiMo-Code 是什么、Fork 自哪里、为什么重要 |
| 01 | 启动链 + CLI 框架 | `src/index.ts`、`src/cli/cmd/*.ts` | 22 个 yargs 子命令如何挂到 bin shim 上 |
| 02 | Agent 体系 | `src/agent/agent.ts` | 3 主 agent + 5 隐藏 subagent 的权限/工具矩阵 |
| 03 | LLM 调用层 | `src/provider/*` | ai-sdk 适配、22 provider、persistent-retry |
| 04 | ReAct 主循环 | `src/session/prompt.ts` (3355 行) | 用户消息 → 工具调用 → 反馈的中枢 |
| 05 | Tools 工具集 | `src/tool/*.ts` (35 文件) | read/write/bash/edit/webfetch/memory/... |
| 06 | 持久化记忆 | `src/memory/*`、`src/session/checkpoint.ts` (1478 行) | 核心差异化：跨会话 FTS5 记忆 + checkpoint-writer |
| 07 | 子 agent 系统 | `src/actor/*` | lifecycle、contextMode、stuck 检测 |
| 08 | 任务系统 | `src/task/*` | T1/T1.1 树状 ID、9 verb 状态机 |
| 09 | TUI | `src/cli/cmd/tui/*` | SolidJS + opentui + Worker 隔离 |
| 10 | Server / API | `src/server/*` | Hono + control/instance 双模式 |
| 11 | Effect + DB | `src/effect/*`、`src/storage/*` | 13 个 effect 文件 + 39 个 drizzle 迁移 |
| 12 | Plugin / Skill / Workflow | `src/plugin/*`、`src/skill/*`、`src/workflow/*` | 上层扩展机制 |
| 13 | Fork 差异 | — | MiMo 自研 / OpenCode 继承 / 改写的差异清单 |

## 关键发现速查表

> 详细分析见 `13-fork-diff.md`，这里是给"30 分钟理解 MiMo-Code"的速记版

### ⭐ 核心差异化（自研）
1. **跨会话持久化记忆**（`06-memory.md`）
   - SQLite FTS5 全文搜索
   - 4 类记忆文件：`MEMORY.md`（项目）、`checkpoint.md`（会话）、`notes.md`（草稿）、`tasks/<id>/progress.md`（任务）
   - **checkpoint-writer 子 agent** 自动维护 checkpoint——保证上下文溢出后能从压缩状态恢复
2. **Goal 停止条件**（`04-react-loop.md` → `goal.ts`）
   - 独立裁判模型评估"是否真正完成"，避免 LLM 自主工作时的"乐观停止"
3. **Dream & Distill**（`04-react-loop.md` → `auto-dream.ts`）
   - `/dream` 提取持久知识到项目记忆
   - `/distill` 把重复手动工作打包成 skill/subagent
4. **Tree 状任务系统**（`08-task.md`）
   - T1 / T1.1 / T1.2 树状 ID，9 verb 状态机（create/claim/start/block/done/...）
   - 与 checkpoint 联动，恢复会话后任务进度不丢
5. **Compose 编排模式**（`02-agents.md`、`12-plugin-skill.md`）
   - Bun macro 编译时把 spec 内联到二进制
   - 内置 plan/execute/review/TDD/debug/verify/merge 技能

### 🔁 继承自 OpenCode（基本原样）
- 22 provider 适配（`src/provider/sdk/`）
- ai-sdk 集成方式
- 工具注册表机制（`src/tool/registry.ts`）
- Permission 系统（`src/permission/*`）
- Hono 服务框架 + hono-openapi

### 🔧 改写/重写
- 引入 **Effect Runtime**（OpenCode 没用）——所有长生命周期 Service 用 `Layer.effect` + `Effect.gen`
- DB 抽象层 `#db` import conditions（Bun / Node 双后端）
- TUI 走 **Worker 隔离**（`src/cli/cmd/tui/worker.ts`），主线程只渲染
- 加了**最大模式**（max agent）：N 路并行推理候选 + 选最佳

## 复盘使用建议

- **第一次读**：按"阅读路径"顺序通读 00-04 即可建立完整心智模型
- **想理解某个具体特性**：用"章节索引"直接跳到对应章节
- **想对比 OpenCode**：先读 00-overview 的"Fork 关系"节，再读 13-fork-diff
- **想自己改代码**：先读 11-effect-db 了解 Service/Layer 模式，否则会看懵

## 验证

- ✅ 全 14 章按统一模板写完（每章含"涉及文件 / 核心数据流 / 关键设计 / 与 OpenCode 差异 / 关键引用"）
- ✅ `bun turbo typecheck` 在 `packages/opencode/` 通过
- ✅ 关键差异化章节（02/06/07/13）含详细代码片段引用
- ✅ 纯分析任务，**未修改任何源码**

## 致谢

- `XiaomiMiMo/MiMo-Code` 上游仓库（MIT）
- `sst/opencode` 上游 OpenCode 项目（MIT）
- 笔记中引用的官方文档路径：`MiMo-Code/docs/`
