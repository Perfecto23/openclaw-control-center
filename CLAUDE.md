# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# 构建
npm run build                    # tsc 编译到 dist/

# 测试（Node.js 内置 test 框架）
npm test                         # 运行全部测试
node --test test/usage-cost.test.ts   # 运行单个测试文件

# 开发运行
npm run dev                      # 基础监控模式
npm run dev:ui                   # UI 模式（HTTP 服务器 :4310）
npm run dev:continuous           # 连续监控

# 验证
npm run validate                 # 验证 task-store + budget 计算
npm run smoke:ui                 # UI 烟雾测试

# 运维命令（通过 APP_COMMAND 环境变量触发）
npm run command:backup-export    # 导出状态快照
npm run command:import-validate  # 验证导入包（无变更）
npm run command:acks-prune       # 清理过期确认记录
npm run command:task-heartbeat   # 任务心跳
npm run release:audit            # 发布审计
```

## 架构

### 核心原则

**官方优先**：优先使用 OpenClaw 官方接口，仅在必要时添加薄适配器。所有写操作严格限制在 `runtime/` 目录，不得修改 OpenClaw 运行时配置。

### 运行模式

项目有两种主要运行模式，由环境变量控制：

- **监控模式**（`src/index.ts`）：轮询 OpenClaw Gateway（WebSocket `GATEWAY_URL`），写入 `runtime/timeline.log`、`runtime/last-snapshot.json`、`runtime/digests/`
- **UI 模式**（`UI_MODE=true`）：启动 HTTP 服务器（`src/ui/server.ts`，端口 4310），提供所有 REST API 和前端页面

**命令模式**（`APP_COMMAND=<cmd>`）：单次执行后退出，用于 `backup-export`、`import-validate`、`acks-prune`、`task-heartbeat`。

### 目录结构

```
src/
├── index.ts              # 入口：模式分发（monitor / ui / command）
├── config.ts             # 所有环境变量的单一读取点
├── types.ts              # 全局类型定义
├── clients/              # OpenClaw Gateway 客户端（WebSocket）
│   ├── factory.ts        # 根据 READONLY_MODE 选择客户端实现
│   └── openclaw-live-client.ts  # 实时 WS 客户端
├── adapters/             # 只读适配器（无 Gateway 时的 stub）
├── contracts/            # OpenClaw 工具调用契约定义
├── mappers/              # API 响应 → 内部类型映射
├── runtime/              # 核心业务逻辑模块（每模块单一职责）
│   ├── commander.ts      # 告警生成与路由
│   ├── task-store.ts     # 任务 CRUD（runtime/tasks.json）
│   ├── project-store.ts  # 项目 CRUD（runtime/projects.json）
│   ├── budget-governance.ts  # 预算阈值计算
│   ├── audit-timeline.ts # 聚合多源审计日志
│   ├── approval-action-service.ts  # 审批动作（默认 dry-run）
│   └── ...               # 其他运营模块
└── ui/
    └── server.ts         # 单文件 HTTP 服务器（原生 Node.js http 模块）
```

### 本地持久化（runtime/ 目录）

所有写操作仅限于此目录：

| 文件 | 用途 |
|------|------|
| `runtime/projects.json` | 项目状态 |
| `runtime/tasks.json` | 任务状态 |
| `runtime/budgets.json` | 预算策略 |
| `runtime/acks.json` | 通知确认记录（支持 TTL/snooze） |
| `runtime/timeline.log` | 监控事件 JSONL |
| `runtime/approval-actions.log` | 审批操作审计 JSONL |
| `runtime/operation-audit.log` | 导入/导出操作审计 JSONL |
| `runtime/last-snapshot.json` | 最新 Gateway 快照 |
| `runtime/digests/YYYY-MM-DD.{json,md}` | 日摘要 |
| `runtime/exports/*.json` | 备份导出包 |
| `runtime/export-snapshots/*.json` | 状态导出快照 |

### 安全默认值（不得随意修改）

```
READONLY_MODE=true                  # 只读模式
LOCAL_TOKEN_AUTH_REQUIRED=true      # 所有变更 API 需要 x-local-token
APPROVAL_ACTIONS_ENABLED=false      # 审批动作关闭
APPROVAL_ACTIONS_DRY_RUN=true       # 审批动作默认 dry-run
IMPORT_MUTATION_ENABLED=false       # 实时导入关闭
```

受保护路由（需要 `x-local-token` 或 `Authorization: Bearer`）：所有 `POST`/`PATCH` 端点，以及 `/api/import/*`、`/export/state.json`。

### 告警路由

```
info          → timeline（runtime/timeline.log）
warn          → operator-watch（GET /exceptions）
action-required → action-queue（GET /api/action-queue）
```

### 预算治理

作用域：`agent` / `project` / `task`。输入来自 `session_status` 快照（`tokensIn`、`tokensOut`、`cost`）。状态：`ok` / `warn` / `over`，由 `budgets.json` 中的阈值和 `warnRatio` 计算。

### 技术栈

- TypeScript 5.8（严格模式，CommonJS 输出）
- Node.js 原生 `http` 模块（无 Express/Fastify）
- Node.js 内置 `test` 框架（无 Jest/Vitest）
- 最小依赖：仅 `@types/node`、`tsx`、`typescript`
- 外部连接：仅 OpenClaw Gateway WebSocket（`ws://127.0.0.1:18789`）
