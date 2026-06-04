# 核心工作流：Issue 生命周期

## 端到端业务流程

### Goal 到 Issue 分解

**它是什么**：将高层业务目标（Goal）逐层拆解为可执行的 Issue 树，形成从战略到任务的工作单元层级。

**触发方式**：用户操作（Board 用户通过 UI 创建 Goal，或在 Issue 详情中基于 Plan 分解创建子 Issue）。

**流程全景**：

```mermaid
sequenceDiagram
    actor User
    participant UI as IssueDetail.tsx
    participant API as routes/goals.ts
    participant Svc as goalService
    participant DB as goals / issues

    User->>UI: 创建 Goal / 分解 Plan
    UI->>API: POST /companies/:id/goals
    API->>Svc: goalService.create(companyId, data)
    Svc->>DB: INSERT goals
    DB-->>Svc: 返回 goal 记录
    Svc-->>API: goal
    API-->>UI: 201 + goal

    alt 基于 Plan 分解创建子 Issue
        UI->>API: POST /issues/:id/accepted-plan-decompositions
        API->>DB: INSERT issue_plan_decompositions + issues(children)
        DB-->>API: 返回子 Issue 列表
    end
```

**关键步骤说明**：

| 步骤 | 负责模块 | 说明 |
|------|---------|------|
| Goal 创建 | `routes/goals.ts` | 接收 `createGoalSchema`，校验公司权限后创建 |
| Goal 查询 | `goalService` | 支持 list / getById / create / update / remove |
| Issue 分解 | `routes/issues.ts` | `POST /issues/:id/accepted-plan-decompositions` 将计划转化为子 Issue |
| 树形关联 | `issues` schema | `parentId` 自引用形成 Issue 树；`goalId` 关联目标 |

---

### Issue 分配与执行

**它是什么**：将 Issue 分配给 Agent（或用户），通过 Heartbeat 机制驱动 Agent 在独立工作空间中实际执行代码/任务。

**触发方式**：
- **Heartbeat 定时唤醒**：Agent 的 timer 触发 `enqueueWakeup`
- **用户分配**：用户通过 UI 修改 assignee，或 Agent checkout Issue
- **手动唤醒**：`POST /agents/:id/wakeup` 或 `POST /agents/:id/heartbeat/invoke`

**流程全景**：

```mermaid
sequenceDiagram
    actor User
    participant UI as IssueDetail.tsx
    participant IR as routes/issues.ts
    participant AR as routes/agents.ts
    participant HS as heartbeatService
    participant IS as issueService
    participant DB as issues / heartbeat_runs
    participant CLI as cli/heartbeat-run.ts

    User->>UI: 分配 Agent / Checkout Issue
    UI->>IR: POST /issues/:id/checkout
    IR->>IS: checkout(id, agentId, expectedStatuses, runId)
    IS->>DB: UPDATE issues status=>in_progress, checkoutRunId, executionRunId
    DB-->>IS: updated issue
    IS-->>IR: issue
    IR->>HS: heartbeat.wakeup(agentId, {source: "assignment", payload: {issueId}})
    HS->>DB: INSERT agent_wakeup_requests + heartbeat_runs(status=queued)
    DB-->>HS: queued run
    HS->>HS: startNextQueuedRunForAgent(agentId)
    HS->>HS: claimQueuedRun(run) → status=running
    HS->>HS: executeRun(runId)
    HS->>DB: UPDATE issues executionRunId=run.id, executionLockedAt=now
    CLI->>AR: POST /agents/:id/wakeup (CLI 轮询/触发)
    AR->>HS: heartbeat.wakeup(...)
    HS-->>CLI: 返回 run 状态
    CLI->>CLI: 本地执行 Adapter 任务
    CLI->>AR: 上报 run 结果
```

**关键步骤说明**：

| 步骤 | 负责模块 | 说明 |
|------|---------|------|
| Issue Checkout | `issueService.checkout` | 将 Issue 状态设为 `in_progress`，绑定 `checkoutRunId` 与 `executionRunId`；检查依赖阻塞和 Pause Hold |
| Agent 唤醒入队 | `heartbeatService.enqueueWakeup` | 写入 `agent_wakeup_requests` 和 `heartbeat_runs(queued)`；支持幂等、合并(coalesce)、延迟(defer) |
| 运行认领 | `heartbeatService.claimQueuedRun` | 将 `queued` 转为 `running`，检查预算、依赖、树暂停等 |
| 执行运行 | `heartbeatService.executeRun` | 准备执行上下文（workspace、session、adapter config），调用 Adapter 实际运行 |
| 执行锁 | `issues` schema | `executionRunId` + `executionLockedAt` 防止并发执行；同一 Agent 可合并运行 |
| 释放 Issue | `issueService.release` | 将状态回退为 `todo`，清空 assignee 和运行绑定 |

---

### Issue 状态转换

**状态转换图**：

```mermaid
stateDiagram-v2
    [*] --> backlog: 创建 Issue
    backlog --> todo: 分配 / 准备
    todo --> in_progress: checkout / 开始执行
    in_progress --> in_review: 提交评审 / 执行策略
    in_review --> in_progress: 评审未通过 / 返回执行
    in_progress --> done: 完成 / 审批通过
    in_review --> done: 审批通过
    in_progress --> blocked: 遇到阻塞
    blocked --> todo: 阻塞解除
    todo --> cancelled: 取消
    in_progress --> cancelled: 取消
    in_review --> cancelled: 取消
    done --> [*]
    cancelled --> [*]
```

| 状态 | 含义 | 进入条件 | 退出条件 |
|------|------|---------|---------|
| backlog | 待办池，尚未分配 | Issue 创建默认状态 | 被分配或手动移入 todo |
| todo | 已就绪，等待执行 | 用户分配、release 后、阻塞解除 | Agent checkout 进入 in_progress |
| in_progress | 正在执行 | checkout 成功、Agent 认领运行 | 完成(done)、取消(cancelled)、进入评审(in_review) |
| in_review | 等待评审/审批 | 执行策略进入 review 阶段、用户提交 | 审批通过(done)、返回修改(in_progress)、取消 |
| blocked | 被依赖阻塞 | 依赖 Issue 未解决 | 依赖解决后回到 todo |
| done | 已完成 | 执行成功且审批通过 | 无 |
| cancelled | 已取消 | 用户或系统取消 | 无 |

---

## 模块内部执行流程

### Issue 路由处理

**它是什么**：`routes/issues.ts` 是 Issue 领域的 HTTP 入口，负责 CRUD、状态变更、文档/评论/附件/审批/工作产物等全生命周期操作，并协调 `issueService`、`heartbeatService`、`approvalService` 等下游服务。

**执行流程**：

```mermaid
flowchart TD
    A[HTTP Request] --> B{路由匹配}
    B -->|GET /issues/:id| C[查询 Issue 详情]
    B -->|POST /companies/:id/issues| D[创建 Issue]
    B -->|POST /issues/:id/checkout| E[Checkout Issue]
    B -->|POST /issues/:id/release| F[Release Issue]
    B -->|PATCH /issues/:id| G[更新 Issue]
    B -->|POST /issues/:id/approvals| H[关联审批]
    B -->|POST /issues/:id/comments| I[添加评论]
    B -->|POST /issues/:id/documents| J[文档操作]
    B -->|POST /issues/:id/work-products| K[工作产物]
    C --> L[authz + issueService.getById]
    D --> M[applyCreateIssueStatusDefault + validate + issueService.create]
    E --> N[assertCompanyAccess + issueService.checkout + heartbeat.wakeup]
    F --> O[assertAgentIssueMutationAllowed + issueService.release]
    G --> P[applyIssueExecutionPolicyTransition + issueService.update]
    H --> Q[issueApprovalService.linkApproval]
```

**关键决策点**：

| 决策点 | 位置 | 逻辑 |
|--------|------|------|
| 创建时状态默认值 | `applyCreateIssueStatusDefault` | 根据请求体解析默认 status，注入 body |
| 执行策略转换 | `applyIssueExecutionPolicyTransition` | 根据 `executionPolicy` / `executionState` 计算状态、assignee、monitor 等 patch |
| Checkout 权限 | `routes/issues.ts ~5045` | 检查 project 是否 paused、Agent 只能 checkout 自己、assignee 权限 |
| Release 权限 | `routes/issues.ts ~5129` | 只有 assignee 或同 run 可 release；admin 可 force-release |
| 评论/文档/附件 | 各子路由 | 统一走 `issueService` 或独立 service，最后 `logActivity` |

---

## 流程间的关联

```mermaid
graph LR
    subgraph 用户层
        UI[IssueDetail.tsx]
        GOAL_UI[Goal 管理页]
    end

    subgraph 路由层
        IR[routes/issues.ts]
        GR[routes/goals.ts]
        AR[routes/agents.ts]
    end

    subgraph 服务层
        IS[issueService]
        GS[goalService]
        HS[heartbeatService]
        IEP[issue-execution-policy]
    end

    subgraph 数据层
        ISS[issues schema]
        GOS[goals schema]
        HRS[heartbeat_runs schema]
    end

    UI -->|CRUD / checkout / release| IR
    GOAL_UI -->|CRUD| GR
    IR -->|调用| IS
    IR -->|wakeup| HS
    IR -->|策略转换| IEP
    GR -->|调用| GS
    IS -->|读写| ISS
    GS -->|读写| GOS
    HS -->|读写| HRS
    HS -->|checkout / release| IS
    AR -->|wakeup / invoke| HS
    CLI[cli/heartbeat-run.ts] -->|wakeup| AR
```
