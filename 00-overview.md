# 00 · 项目全景

## 一句话定位
MiMo-Code 是小米 2026-06-10 开源的终端原生 AI 编程智能体，基于 OpenCode（sst/opencode）的 fork，新增了**跨会话持久化记忆、Goal 停止条件、Dream/Distill 自进化、Tree 任务追踪、Compose 编排**等自研特性；用 TypeScript + Bun 写成，单包 ~110k 行（`packages/opencode/` 估算）。

## 涉及文件
- `package.json`（root） — workspace 拓扑、catalog 依赖、scripts
- `turbo.json` — turbo 编排（typecheck、build、test）
- `bun.lock` — 锁定依赖
- `bunfig.toml` — Bun 运行时配置（含 `test.root` 防根目录跑测试）
- `AGENTS.md` — 仓库风格指南（强调并行工具调用、避免 try/catch、snake_case schema）
- `README.md` / `README.zh.md` — 用户向文档
- `sst.config.ts` — 部署配置
- `flake.nix` / `nix/` — Nix 打包
- `install/` — 一键安装脚本

## 核心数据流

### Monorepo 拓扑
```
MiMo-Code/                              ← root（脚本+配置）
├── packages/
│   ├── opencode/        ⭐ 核心 — CLI + TUI + Server 全部塞在这
│   │   ├── src/         ← 业务代码（agent / session / tool / ...）
│   │   ├── bin/mimo     ← npm bin shim（找本地 .mimocode 二进制 → 否则下载）
│   │   ├── cli/         ← yargs 子命令 + TUI
│   │   ├── migration/   ← 34 个 drizzle 迁移（2026-01 ~ 2026-06）
│   │   └── script/      ← build/fix-node-pty 等工具
│   ├── app/             ← Web UI（SolidJS）
│   ├── desktop/         ← Tauri 桌面壳
│   ├── console/         ← Console web app
│   ├── containers/      ← 容器化部署配置
│   ├── enterprise/      ← 企业版
│   ├── extensions/      ← 扩展点
│   ├── function/        ← serverless 函数
│   ├── identity/        ← OAuth / auth
│   ├── plugin/          ← 插件定义
│   ├── script/          ← 仓库级脚本
│   ├── sdk/js           ← 自动生成的 JS SDK（`script/build.ts`）
│   ├── shared/          ← 跨包工具（effect 类型、util/error）
│   ├── slack/           ← Slack 集成
│   ├── storybook/       ← UI 组件库文档
│   └── ui/              ← 共享 UI 组件
├── sdks/                ← 各语言 SDK（可能多语言）
├── infra/               ← SST 基础设施
├── patches/             ← 依赖 patch
├── docs/                ← 官方文档
├── script/              ← 全局脚本
├── install/             ← 安装脚本
├── nix/                 ← Nix flake
└── sst-env.d.ts         ← SST 类型
```

### Fork 关系
```
sst/opencode（MIT）   ←── sst/opencode-v0.x（2025）
       │  2026-06 突然消失 → XiaomiMiMo/MiMo-Code 接管
       ↓
XiaomiMiMo/MiMo-Code（MIT，1976 stars @ 2026-06-10 首发）
       │  持续追平上游 + 加自研特性
       ↓
（猜测）未来可能有 sst/opencode 重新上线 或 MiMo 走向自研路线
```

### 启动链（概览，详见 01）
```
$ mimo [command]
  ↓
bin/mimo（Node shim）
  ├── 查 MIMOCODE_BIN_PATH 环境变量
  ├── 查本机 .mimocode（缓存的预编译二进制）
  └── 否则下载 + 解压到 .mimocode
  ↓
mimo 编译后的二进制（单文件 ~80MB）
  ↓
src/index.ts
  ├── yargs 解析 CLI
  ├── Log.init
  ├── Database.use（init + migrate）
  ├── Plugin 加载
  └── dispatch 到 cli/cmd/<command>.ts
```

## 关键设计

### 1. 单包核心（`packages/opencode/` 集中 99% 业务）
- **是什么**：把 CLI / TUI / Server / Agent / Tool / DB 全部塞进 `packages/opencode/`，其它 `packages/*` 都是壳/UI/SDK
- **为什么**：单包便于 Bun 单文件打包发布；Bun 条件 imports (`#db` / `#pty`) 让单个入口适配 Bun/Node 双运行时
- **关键证据**：
  - `packages/opencode/src/index.ts` — CLI 唯一入口（已读，66 行 import + 22 子命令）
  - `packages/opencode/package.json` `bin: { "mimo": "./bin/mimo" }` — npm 安装后可直接 `mimo` 调起
  - `package.json:scripts.dev` — `MIMOCODE_HOME=$PWD/.dev-home bun run --cwd packages/opencode --conditions=browser src/index.ts`

### 2. TypeScript + Bun 优先
- **技术栈**：
  - **运行时**：Bun 1.3.11（`packageManager: "bun@1.3.11"`）；可选 Node 22（`@tsconfig/node22`）
  - **数据库**：SQLite via `drizzle-orm/bun-sqlite`（Bun 优先）/ `better-sqlite3`（Node 兜底）
  - **LLM SDK**：`ai` (Vercel ai-sdk v4) + 22 个 provider 适配
  - **TUI**：`@opentui/core` + `@opentui/solid`（SolidJS 渲染）
  - **Web UI**：`@opentui/solid`（共享 SolidJS 组件）
  - **服务框架**：`hono` + `@hono/zod-validator` + `hono-openapi`（自动 OpenAPI 文档）
  - **异步运行时**：`effect` v4 beta（4.0.0-beta.48）+ `@effect/platform-node`
  - **Schema 校验**：`zod`（全量）
  - **依赖管理**：`workspaces.catalog`（50+ 共享版本统一在 root `package.json`）
- **关键证据**：
  - `bunfig.toml` — `test.root` 防根目录跑测试（强制从 package 目录跑）
  - `packages/opencode/package.json` `imports."#db"` — `bun: ./src/storage/db.bun.ts`, `node: ./src/storage/db.node.ts`

### 3. 三层架构
```
┌─────────────────────────────────────────────┐
│  入口层 (entry)                              │
│  src/index.ts  +  src/cli/cmd/*  +  src/server/*  +  src/cli/cmd/tui/*  │
├─────────────────────────────────────────────┤
│  业务核心 (domain)                            │
│  src/agent/*  +  src/session/*  +  src/task/*  +  src/actor/*  +  src/memory/*  +  src/workflow/*  +  src/tool/*  +  src/skill/*  +  src/plugin/*  │
├─────────────────────────────────────────────┤
│  基础设施 (infra)                             │
│  src/effect/*  +  src/storage/*  +  src/provider/*  +  src/permission/*  +  src/bus/*  +  src/util/*  +  src/config/*  │
└─────────────────────────────────────────────┘
```

### 4. 数据库即真相（Database-of-Record）
- **是什么**：34 个 drizzle 迁移（`packages/opencode/migration/`）定义完整 schema
- **为什么**：所有跨会话状态都进 SQLite；checkpoint / memory / task / actor / session 全在 DB 里
- **关键证据**：
  - `migration/20260515010000_memory_fts/` — FTS5 全文搜索（v6 之前还有 20260521010000_memory_fts_v6）
  - `migration/20260609000000_history_fts/` — 命令历史 FTS
  - `migration/20260527000000_actor_lifecycle/` — 子 agent 生命周期
  - `migration/20260527000100_inbox/` — Inbox 消息队列（actor 通知）
  - `migration/20260603000000_task_in_progress_owner/` — 任务状态机

## 与上游 OpenCode 的差异

> 详细差异清单见 `13-fork-diff.md`，这里只点出**架构级**差异

| 维度 | OpenCode（上游） | MiMo-Code | 影响 |
|------|------------------|-----------|------|
| 异步运行时 | 原生 Promise + async/await | **Effect v4** 全量替代 | 资源管理、错误处理、并发声明式 |
| 持久化 | 仅 session 表 | **FTS5 全文搜索 + 4 类记忆文件** | 跨会话记忆成为核心特性 |
| 子 agent | 无正式抽象 | **Actor 系统**（lifecycle、contextMode、stuck 检测） | 多 agent 并行工作 |
| 任务系统 | 无 | **Tree 状 T1/T1.1** + 9 verb 状态机 | 长任务可恢复 |
| 编译产物 | JS bundle | **Bun 单文件二进制** + 跨平台预编译 | 安装即用、零依赖 |
| Provider | 22 个 | 22 个 + 1 个 **MiMo Free** 匿名通道 | 零配置开箱即用 |
| Agent 数量 | 1 个 build | 3 主（build/plan/compose）+ 7 隐藏 subagent | 专业化分工 |
| Goal 停判 | 无 | **独立裁判模型** | 防止自主工作"乐观停止" |

## 关键引用

- `package.json:1-50` — workspaces.catalog 依赖定义
- `packages/opencode/package.json:1-40` — bin 入口 + dual runtime imports
- `packages/opencode/src/index.ts:1-66` — yargs 启动链
- `packages/opencode/bin/mimo:1-50` — npm bin shim 逻辑
- `packages/opencode/migration/` — 34 个迁移（`ls` 验证）
- `README.zh.md:1-50` — 项目中文简介（已读，验证核心特性 5 项）

## 学习检查清单

读完后应能回答：
- [ ] MiMo-Code 的 monorepo 怎么组织？核心包是哪个？
- [ ] 与 OpenCode 的核心架构差异是什么？
- [ ] 单包策略 + Bun 编译的优缺点？
- [ ] 哪 5 个核心特性是 MiMo 自研、OpenCode 没有的？
