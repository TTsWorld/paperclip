# 代码阅读路径

## 快速理解（按顺序阅读这些文件）

1. **`README.md`** — 这是项目门面，先读前 80 行 → 你会理解 Paperclip 的核心理念："如果 OpenClaw 是员工，Paperclip 就是公司"，以及它解决的多代理协调问题。

2. **`server/src/app.ts`** — Express 应用的组装中心 → 读完后你会知道系统有哪些 REST API 端点、哪些中间件、插件和适配器是如何挂载到服务器上的。这是理解服务端全貌的最佳入口。

3. **`packages/db/src/schema/` 下的核心 schema 文件** — 优先阅读 `issues.ts`、`goals.ts`、`agents.ts`、`companies.ts` → 读完后你会理解系统的核心实体和它们之间的关系。Paperclip 的整个业务逻辑都是围绕这些表展开的。

4. **`server/src/services/heartbeat.ts` 前 200 行** — 心跳服务的核心入口 → 读完后你会理解 Agent 是如何被唤醒、排队、执行和完成的最核心调度逻辑。这是整个系统的"发动机"。

5. **`cli/src/commands/heartbeat-run.ts` 前 150 行** — CLI 触发心跳的入口 → 读完后你会理解外部如何与 Paperclip 交互来驱动 Agent 执行，以及 CLI 如何轮询执行事件。

---

## 按目标索引

### 想理解整体架构

| 文件 | 关键位置 | 说明 |
|------|---------|------|
| `README.md` | 开头 | 项目定位和设计哲学 |
| `pnpm-workspace.yaml` | 全文 | monorepo 模块划分 |
| `server/src/app.ts` | 路由注册区 | 服务端模块组装 |
| `server/src/index.ts` | 启动流程 | 服务器初始化顺序 |
| `packages/shared/src/index.ts` | 导出列表 | 共享类型和工具 |
| `ui/src/App.tsx` | 路由定义 | 前端页面结构 |
| `cli/src/index.ts` | 命令注册 | CLI 命令体系 |

### 想理解 Agent 编排与心跳执行

| 文件 | 关键位置 | 说明 |
|------|---------|------|
| `server/src/services/heartbeat.ts` | `wakeup()`、`enqueueWakeup()`、`executeRun()` | 心跳调度的核心 |
| `server/src/services/agents.ts` | `create()`、`updateStatus()` | Agent 生命周期管理 |
| `server/src/services/agent-start-lock.ts` | 全文 | 并发控制锁 |
| `server/src/services/workspace-runtime.ts` | 前 150 行 | 执行工作区准备 |
| `server/src/adapters/registry.ts` | 适配器注册 | 如何根据 `adapterType` 获取适配器 |
| `cli/src/commands/heartbeat-run.ts` | `heartbeatRun()` | CLI 触发入口 |

### 想理解 Issue 工作流

| 文件 | 关键位置 | 说明 |
|------|---------|------|
| `server/src/routes/issues.ts` | 前 300 行 | Issue CRUD 和状态变更 |
| `server/src/routes/goals.ts` | 前 150 行 | Goal 创建和查询 |
| `server/src/services/heartbeat.ts` | `claimQueuedRun()`、`releaseIssueExecutionAndPromote()` | Issue 执行锁管理 |
| `packages/db/src/schema/issues.ts` | 表定义 | Issue 的所有字段含义 |
| `ui/src/pages/IssueDetail.tsx` | 前 150 行 | Issue 详情页结构 |

### 想理解插件系统

| 文件 | 关键位置 | 说明 |
|------|---------|------|
| `server/src/services/plugin-worker-manager.ts` | 前 150 行 | 插件 Worker 进程管理 |
| `server/src/services/plugin-job-scheduler.ts` | 前 100 行 | 插件任务调度 |
| `server/src/services/plugin-tool-dispatcher.ts` | 前 100 行 | 插件工具分发 |
| `packages/plugins/sdk/src/index.ts` | 导出列表 | 插件 SDK 接口 |
| `packages/plugins/create-paperclip-plugin/` | 脚手架模板 | 如何创建新插件 |

### 想理解适配器体系

| 文件 | 关键位置 | 说明 |
|------|---------|------|
| `packages/adapter-utils/src/index.ts` | 导出列表 | 适配器通用接口 |
| `packages/adapters/claude-local/` | 实现文件 | Claude Code 适配器示例 |
| `packages/adapters/openclaw-gateway/` | 实现文件 | OpenClaw 适配器示例 |
| `server/src/adapters/` | 注册和通用逻辑 | 服务端适配器层 |

### 想扩展 / 贡献代码

| 目标 | 需要理解的模块 | 潜在入口点 |
|------|--------------|-----------|
| 添加新的 AI 代理适配器 | `packages/adapter-utils/` + 任意现有适配器 | 复制 `claude-local` 目录，实现 `ServerAdapterModule` 接口 |
| 添加新的后端 API | `server/src/routes/` + `server/src/services/` | 在 `app.ts` 注册新路由，在 `services/` 添加业务逻辑 |
| 添加新的前端页面 | `ui/src/pages/` + `ui/src/api/` | 在 `App.tsx` 添加路由，在 `api/` 添加后端调用 |
| 添加新的插件功能 | `packages/plugins/sdk/` + `server/src/services/plugin-*.ts` | 使用 `create-paperclip-plugin` 脚手架创建 |
| 修改数据库 schema | `packages/db/src/schema/` | 添加表定义后运行 `pnpm db:generate` 生成迁移 |

---

## 值得深入学习的代码片段

1. **心跳调度队列（`server/src/services/heartbeat.ts`）**
   - 为什么值得学：它展示了一个生产级的异步任务队列是如何实现的——包括唤醒请求的合并（coalesce）、延迟执行（deferred）、并发控制（`maxConcurrentRuns`）、有界重试和续跑。这些模式在分布式调度系统中非常通用。
   - 能学到什么：如何设计一个既支持实时响应又支持批量合并的任务队列；如何用数据库状态机（`queued` → `running` → `succeeded`）代替内存队列实现持久化和容错。

2. **Issue 树形结构 + 阻塞关系（`packages/db/src/schema/issues.ts` + `issueRelations`）**
   - 为什么值得学：它同时用 `parentId`（自引用，树形结构）和 `issueRelations`（独立表，有向图）表达两种关系，这是非常精妙的建模。
   - 能学到什么：何时应该用自引用外键、何时应该用独立关系表；如何用最小 schema 复杂度支持树形分解和复杂依赖网络。

3. **适配器统一接口（`packages/adapter-utils/` + 各适配器实现）**
   - 为什么值得学：10 个不同的 AI 代理运行时（Claude、Codex、Cursor、OpenClaw 等）通过同一套接口接入，展示了"策略模式"在真实项目中的落地。
   - 能学到什么：如何设计一个稳定的插件接口，让第三方实现者只需要关注"做什么"而不需要关注"怎么被调度"。

4. **插件 JSON-RPC over stdio 架构（`server/src/services/plugin-worker-manager.ts`）**
   - 为什么值得学：进程外扩展是保障系统稳定性的经典设计，Paperclip 用 Node.js 子进程 + JSON-RPC 实现了完整的插件运行时隔离。
   - 能学到什么：如何用 stdio 作为轻量级 IPC 通道；如何管理子进程生命周期（启动、健康检查、优雅退出、崩溃重启）。

5. **预算控制三级模型（`server/src/services/budgets.ts` + `costs.ts`）**
   - 为什么值得学：Company / Agent / Project 三级预算策略，结合 `budgetPolicies` 表和 `costEvents` 实时触发评估，是一个完整的资源配额系统。
   - 能学到什么：如何在多租户系统中设计资源配额；如何用事件驱动的方式实现实时预算告警和硬停。

---

## 令人困惑的地方

1. **为什么 `adapterConfig` 和 `runtimeConfig` 要分开？**
   - 初看困惑：Agent 表中有两个 JSONB 配置字段，容易混淆。
   - 背后原因：`adapterConfig` 描述如何连接外部 AI 运行时（如 Claude Code 的路径、API 密钥），属于"外部系统配置"；`runtimeConfig` 描述 Paperclip 如何调度这个 Agent（如心跳间隔、最大并发运行数、环境变量），属于"内部调度配置"。这种分离让 adapter 层完全不需要理解 Paperclip 的调度策略。

2. **为什么 Issue 同时有 `checkoutRunId` 和 `executionRunId`？**
   - 初看困惑：一个 Issue 为什么需要两个运行 ID？
   - 背后原因：`checkoutRunId` 表示 Agent 已"认领"这个 Issue（checkout），此时 Issue 进入 `in_progress` 状态；`executionRunId` 表示 Agent 正在实际执行这个 Issue。分离它们是为了支持"认领后准备环境"和"真正开始执行"之间的异步间隔——例如 Agent 需要克隆仓库、安装依赖才能开始工作。

3. **为什么 Org Chart 不直接参与任务路由？**
   - 初看困惑：系统有完整的组织架构（`reportsTo`），但 heartbeat 调度似乎不根据 org chart 来决定谁执行什么。
   - 背后原因：Org Chart 在 Paperclip 中主要是"治理和汇报"概念，而非"任务路由"概念。Issue 的分配是通过 `assigneeAgentId` 显式指定的。Org Chart 的影响体现在：Agent 创建权限继承、目标对齐（Goal ownership）、审批链流转等治理层面，而不是执行层面的任务分发。这样设计保持了执行调度的简单性和可预测性。

4. **为什么插件和适配器是两个独立的扩展体系？**
   - 初看困惑：两者都是扩展机制，为什么不能统一？
   - 背后原因：适配器解决"如何连接外部 AI 代理"的问题，每个适配器对应一种运行时协议（Process spawn、HTTP、SSH）；插件解决"如何扩展 Paperclip 自身功能"的问题，如自定义工具、自定义 UI、自定义存储后端。它们的生命周期、运行环境、安全边界完全不同——适配器在 server 主进程内执行，插件在独立子进程中执行。分离让它们各自保持简单。
