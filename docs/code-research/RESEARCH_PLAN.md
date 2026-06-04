# 研究计划

> **研究完成日期**：2026-06-04
> **项目**：Paperclip —— 开源 AI Agent 编排系统

---

## 研究摘要

### 项目核心价值

Paperclip 是一个以"公司管理"隐喻为核心的 AI Agent 编排系统。它将多个外部 AI 代理（Claude Code、OpenClaw、Codex、Cursor 等）组织成虚拟公司，通过组织架构、目标分解、审批流程、预算控制和心跳调度来协调代理团队自主完成业务目标。其最大创新在于：**不是让开发者管理代码提交，而是让管理者经营 AI 公司**。

### 文档索引

| 序号 | 专题 | 文件 | 核心内容 |
|------|------|------|---------|
| A | 架构全景 | [`01_architecture.md`](01_architecture.md) | 项目模块划分、四层架构、模块依赖关系、核心抽象、扩展机制 |
| B | Agent 编排与心跳执行 | [`02_mechanism_agent_heartbeat.md`](02_mechanism_agent_heartbeat.md) | Agent 生命周期、心跳调度队列、Adapter 可插拔架构、Workspace 隔离、Routine 定时触发 |
| C | 数据流与状态管理 | [`03_data_flow.md`](03_data_flow.md) | Company/Agent/Goal/Issue 等核心实体建模、状态机、预算控制三级模型、前端实时更新 |
| D | 依赖与生态 | [`04_dependencies.md`](04_dependencies.md) | Express/Drizzle/Better Auth/Zod 等核心依赖选型理由、适配器体系、插件 JSON-RPC 架构、MCP 集成 |
| E | Issue 生命周期与审批 | [`05_workflow.md`](05_workflow.md) | Goal→Issue 分解、Issue 分配与执行、状态转换、异常处理 |
| F | 学习路径 | [`06_learning_path.md`](06_learning_path.md) | 快速阅读顺序、按目标索引、值得学习的代码片段、易混淆设计解释 |

### 设计亮点

1. **心跳是唯一的调度入口**：所有 Agent 执行——无论定时触发、任务分配、手动唤醒还是自动化回调——都必须经过 `heartbeatRuns` 队列。这种统一抽象让调度逻辑高度内聚，同时通过数据库状态机实现持久化和容错。

2. **Issue 双重关系模型**：用 `parentId` 自引用表达树形分解结构，用独立的 `issueRelations` 表表达阻塞依赖（有向图）。一套 schema 同时支持简单的父子层级和复杂的任务依赖网络。

3. **Adapter 与 Plugin 的双层扩展**：Adapter 解决"连接外部 AI 代理"的问题（进程内执行），Plugin 解决"扩展 Paperclip 功能"的问题（进程外 JSON-RPC 隔离）。两者分离让它们各自保持简单，生命周期和安全边界清晰。

4. **预算控制三级模型**：Company / Agent / Project 三级预算策略通过 `budgetPolicies` 统一配置，`costEvents` 实时触发评估，超预算时直接修改状态为 `paused` 从源头阻止新执行。

5. **Bring Your Own Agent**：通过 `adapter-utils` 统一接口，已支持 10+ 种 AI 代理运行时。核心调度逻辑完全不感知底层是 Claude 还是 Codex，只通过 `ServerAdapterModule.execute()` 调用。

---

## 项目概述

**它是什么**：Paperclip 是一个开源的 AI Agent 编排系统，以 Node.js 服务器 + React UI 的形式提供，用于管理 AI 代理团队完成业务目标。

**解决什么问题**：当用户同时运行多个 AI 代理（Claude Code、OpenClaw、Codex、Cursor 等）时，会面临代理 coordination、目标对齐、成本失控、工作不可审计等问题。Paperclip 将代理管理转化为"公司管理"视角——用组织架构、目标分解、审批流程、预算控制来治理 AI 代理团队。

**谁在使用**：需要构建自主 AI 公司或协调多代理团队的开发者和组织。通过 Dashboard 管理代理，通过 CLI 触发代理执行，通过适配器接入不同 AI 运行时。

## 研究专题

### 专题 A：架构全景
- 目标：理解项目由哪些模块组成，每个模块是什么、为什么存在，模块间依赖关系
- 范围：根目录结构、pnpm workspace 划分、server/ui/cli/packages 各模块职责、适配器体系、插件体系
- 输出：`docs/code-research/01_architecture.md`

### 专题 B：核心机制 - Agent 编排与心跳执行
- 目标：理解 Agent 是什么、为什么需要心跳机制、Agent 如何被唤醒和执行任务、完整调用链
- 范围：`server/src/services/agents.ts`、`server/src/services/heartbeat.ts`、`cli/src/commands/heartbeat-run.ts`、各 adapter 实现、`server/src/services/workspace-runtime.ts`
- 输出：`docs/code-research/02_mechanism_agent_heartbeat.md`

### 专题 C：数据流与状态管理
- 目标：核心数据结构是什么、为什么这样建模、关键业务数据如何流动
- 范围：`packages/db/src/schema/`、`server/src/services/` 中的核心表操作、Issue/Goal/Agent/Approval 等核心实体的状态机、前端状态管理
- 输出：`docs/code-research/03_data_flow.md`

### 专题 D：依赖与生态
- 目标：核心依赖是什么、为什么选择它们、适配器如何连接外部 AI 系统、插件如何扩展功能
- 范围：`package.json`、各 package 依赖、`packages/adapters/*`、`packages/plugins/*`、`packages/mcp-server/`、`packages/skills-catalog/`
- 输出：`docs/code-research/04_dependencies.md`

### 专题 E：核心工作流 - Issue 生命周期与审批
- 目标：追踪从 Goal 分解到 Issue 创建、分配、执行、审批、完成的完整端到端流程
- 范围：`server/src/routes/issues.ts`、`server/src/routes/approvals.ts`、`server/src/routes/goals.ts`、`server/src/services/approvals.ts`、`ui/src/pages/IssueDetail.tsx`、`ui/src/pages/GoalDetail.tsx`
- 输出：`docs/code-research/05_workflow.md`

### 专题 F：学习路径（最后执行，依赖以上专题）
- 目标：为想要贡献代码的开发者总结推荐阅读顺序和关键入口
- 范围：综合以上所有专题结论
- 输出：`docs/code-research/06_learning_path.md`

## 待解决疑问

（研究过程中添加，不要跳过）
