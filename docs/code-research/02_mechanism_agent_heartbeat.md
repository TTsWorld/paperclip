# 核心机制：Agent 编排与心跳执行

## 这是什么机制

**它是什么**：Paperclip 的 Agent 编排与心跳执行机制是一套将多个外部 AI 代理（Claude Code、Codex、Cursor 等）组织成“虚拟公司”并持续驱动其工作的调度系统。每个 Agent 拥有独立配置、状态、预算和汇报关系，系统通过“心跳（Heartbeat）”周期性地或在特定事件触发下唤醒 Agent，使其执行分配的任务（Issue），并通过 Adapter 层将命令翻译为对应 AI 代理可理解的协议。

**为什么需要它**：
- 没有心跳机制，AI 代理只是静态配置，无法被自动唤醒执行任务；
- 没有 Agent 编排，多代理之间缺乏协调，会出现任务冲突、重复执行、预算失控；
- 没有 Adapter 抽象，每接入一种新 AI 代理都需要重写核心调度逻辑；
- 没有 Workspace 隔离，不同 Agent 执行时可能污染彼此的工作目录或环境。

**设计核心**：
- 将“公司管理”隐喻贯穿到底：Agent 有角色（role）、有上级（reportsTo）、有预算（budgetMonthlyCents）、有状态（idle/running/paused/terminated）；
- 心跳是唯一的调度入口：所有 Agent 执行都必须经过 heartbeatRuns 队列，无论是定时触发、任务分配、手动唤醒还是自动化回调；
- Adapter 是可插拔的翻译层：核心调度逻辑完全不感知底层是 Claude 还是 Codex，只通过统一的 `ServerAdapterModule.execute()` 接口调用；
- Workspace 提供执行隔离：每次运行都可能创建独立的 git worktree 或目录，确保 Agent 的执行环境干净且可复现。

---

## 调用链

### 1. Agent 创建与配置

```
用户通过 UI/CLI 创建 Agent
  → POST /api/companies/:companyId/agents (server/src/routes/agents.ts:2123)
    → agentService.create() (server/src/services/agents.ts:413)
      → 写入 agents 表，生成默认配置
      → materializeDefaultInstructionsBundleForNewAgent() (server/src/routes/agents.ts:1065)
        → agentInstructionsService.materializeManagedBundle() (server/src/services/agent-instructions.ts:685)
          → 在磁盘创建 AGENTS.md 指令文件
```

### 2. 心跳触发与执行（核心链路）

```
触发源：定时器 / 任务分配 / 手动唤醒 / 自动化回调 / Routine 触发
  → heartbeatService.wakeup() (server/src/services/heartbeat.ts:8986)
    → enqueueWakeup() (server/src/services/heartbeat.ts:8986)
      → 检查 Agent 状态、预算、并发限制
      → 创建 agentWakeupRequests 记录 + heartbeatRuns 记录（状态 queued）
      → publishLiveEvent("heartbeat.run.queued")
      → startNextQueuedRunForAgent() (server/src/services/heartbeat.ts:6943)
        → withAgentStartLock() (server/src/services/agent-start-lock.ts:32)
          → 获取 Agent 并发策略（maxConcurrentRuns）
          → 按优先级排序 queued runs
          → claimQueuedRun() (server/src/services/heartbeat.ts:6033)
            → 校验预算、依赖、Issue 状态
            → 将 run 状态改为 running
            → 锁定 Issue 执行权（executionRunId）
          → executeRun(run.id) (server/src/services/heartbeat.ts:7016)
            → 解析 Workspace → resolveWorkspaceForRun() (server/src/services/heartbeat.ts:3665)
            → 构建 Adapter 配置 → resolveExecutionRunAdapterConfig() (server/src/services/heartbeat.ts:358)
            → 获取 ServerAdapter → getServerAdapter(agent.adapterType) (server/src/adapters/registry.ts:653)
            → adapter.execute() (例如 server/src/adapters/process/execute.ts 或各包 adapter)
              → 启动子进程 / HTTP 请求 / SSH 远程执行
            → 收集 stdout/stderr、事件、成本
            → 运行结束后更新 heartbeatRuns 状态（succeeded/failed/cancelled/timed_out）
            → finalizeAgentStatus() 更新 agents 状态
            → releaseIssueExecutionAndPromote() 释放 Issue 锁并提升 deferred wake
            → startNextQueuedRunForAgent() 继续处理队列
```

### 3. CLI 触发心跳执行

```
CLI: paperclipai heartbeat-run --agent-id <id>
  → cli/src/commands/heartbeat-run.ts:58 heartbeatRun()
    → 调用 API POST /api/agents/:id/wakeup
    → 轮询 /api/heartbeat-runs/:runId/events 获取实时输出
    → 通过 getCLIAdapter(adapterType) 格式化 stdout 事件
```

### 4. Routine 定时触发心跳

```
系统定时任务或 webhook 触发
  → routineService.tickTimers() / firePublicTrigger() (server/src/services/routines.ts)
    → 解析 cron 表达式，匹配触发条件
    → 创建 routineRuns 记录
    → 调用 heartbeatService.wakeup() 唤醒关联 Agent
      → 后续进入标准心跳执行链路
```

### 5. Adapter 执行细节（以 Process Adapter 为例）

```
adapter.execute() (server/src/adapters/process/execute.ts:14)
  → runChildProcess() (server/src/adapters/utils.ts)
    → spawn 子进程（如 claude、codex 命令）
    → 通过 onLog 回调将 stdout/stderr 流式写入 runLogStore
    → 进程结束后返回 exitCode、signal、timedOut、errorMessage
```

### 6. Workspace 准备与隔离

```
executeRun() 中
  → resolveWorkspaceForRun() (server/src/services/heartbeat.ts:3665)
    → 优先使用 projectWorkspace（项目级工作区）
    → 若无则创建 agent_home 默认目录
  → realizeExecutionWorkspace() (server/src/services/workspace-runtime.ts)
    → 若策略为 git_worktree：创建 git worktree、checkout 分支
    → 若策略为 project_primary：使用项目主目录
    → 记录 workspaceOperations（provision、teardown 等操作）
  → 执行完成后 cleanupExecutionWorkspaceArtifacts() 清理
```

### 7. Org Chart 影响任务委派

```
API GET /api/companies/:companyId/org (server/src/routes/agents.ts:1681)
  → agentService.orgForCompany() (server/src/services/agents.ts:672)
    → 按 reportsTo 关系递归构建树形结构
    → 返回每个 Agent 的 reports（下属）列表
  → UI OrgChart.tsx (ui/src/pages/OrgChart.tsx) 渲染交互式组织架构图
  → 任务分配时，issueService 根据 assigneeAgentId 直接绑定到具体 Agent
    → 与 org chart 无直接执行耦合，但 org chart 决定了“谁可以管谁”的权限视图
```

---

## 关键实现

### Agent 生命周期管理

Agent 的状态机由 `agents.status` 字段驱动：
- `pending_approval`：新创建待审批，不可执行；
- `idle`：空闲，可接受心跳唤醒；
- `running`：至少有一个 heartbeat run 在执行中；
- `paused`：被手动/预算/系统暂停，唤醒会被拒绝；
- `terminated`：已终止，不可恢复，API key 被吊销。

关键代码：`server/src/services/agents.ts:438-499`，`pause/resume/terminate` 方法分别更新状态并记录原因。

### 心跳队列与并发控制

每个 Agent 的并发度由 `runtimeConfig.heartbeat.maxConcurrentRuns` 控制（默认 1）。`startNextQueuedRunForAgent()` 会：
1. 查询当前 running 数量；
2. 计算可用槽位；
3. 按优先级排序 queued runs（in_progress 的 Issue > 依赖已就绪 > 创建时间早）；
4. 逐个 claim 并异步执行。

关键代码：`server/src/services/heartbeat.ts:6943-7014`。

### 唤醒请求合并与延迟执行

当同一 Agent 的同一 Issue 被多次唤醒时（例如用户连续评论），`enqueueWakeup()` 不会创建重复 run，而是：
- 若已有同 Agent 同 Issue 的 running run，则将新 wake **合并（coalesce）**到现有 run 的 contextSnapshot 中；
- 若已有 queued run，则更新其 context；
- 若 Issue 被其他 Agent 执行中，则将 wake **延迟（deferred）**到 `agentWakeupRequests` 表，待当前执行结束后通过 `releaseIssueExecutionAndPromote()` 提升为新的 queued run。

关键代码：`server/src/services/heartbeat.ts:9160-9562`。

### 有界重试与最大轮次续跑

当 Agent 执行因上游瞬态错误（transient upstream）或最大轮次耗尽（max turns exhausted）失败时，系统会自动调度重试：
- `scheduleBoundedRetryForRun()` 实现指数退避重试（2min -> 10min -> 30min -> 2h）；
- 对于 max-turn 耗尽，支持立即续跑（continuation），保持同一 Issue 和 session；
- 重试前通过 `evaluateScheduledRetryGate()` 校验 Issue 是否仍被当前 Agent 拥有、是否仍在 in_progress 状态。

关键代码：`server/src/services/heartbeat.ts:5357-5763`。

### Adapter 注册与可插拔架构

所有 Adapter 在 `server/src/adapters/registry.ts` 中注册为 `ServerAdapterModule`，包含：
- `execute`: 实际执行入口；
- `sessionCodec`: 会话序列化/反序列化；
- `testEnvironment`: 环境探测；
- `listSkills/syncSkills`: 技能同步；
- `getRuntimeCommandSpec`: 生成安装/检测命令（如 `npm install -g @anthropic-ai/claude-code`）。

外部 Adapter 插件可在运行时通过 `registerServerAdapter()` 热加载，覆盖内置类型时会保留 builtin fallback，支持通过 `setOverridePaused()` 回退。

关键代码：`server/src/adapters/registry.ts:502-776`。

### Workspace 运行时隔离

Workspace 的创建策略由 `executionWorkspacePolicy` 决定：
- `shared_workspace`：复用项目主目录；
- `isolated_workspace`：为每次执行创建独立目录或 git worktree；
- `operator_branch`：基于操作者分支隔离。

`workspace-runtime.ts` 实现了完整的 git worktree 生命周期：
- `ensureManagedProjectWorkspace()` 克隆仓库；
- `findRegisteredGitWorktreeByBranch()` 查找已有 worktree；
- `validateLinkedGitWorktree()` 校验 worktree 有效性；
- 执行前后通过 `workspaceOperationRecorder` 记录 provision/teardown 操作。

关键代码：`server/src/services/workspace-runtime.ts:631-780`。

### Routine 触发心跳

Routine 是“定时任务”的抽象，支持 cron 调度、手动触发、webhook 触发。其核心逻辑：
- `routineService` 维护 `routines` 和 `routineTriggers` 表；
- 每次触发创建 `routineRuns` 记录，并通过 `dispatchFingerprint` 去重（同一 Routine 同一参数不重复创建 Issue）；
- 触发后最终调用 `heartbeatService.wakeup()` 将执行权交给 Agent。

关键代码：`server/src/services/routines.ts:473-850`。

---

## 设计决策

### 1. 为什么用“心跳（Heartbeat）”作为统一调度抽象？

Paperclip 将 Agent 视为“需要定期唤醒的员工”，而不是“常驻进程”。这种设计的收益：
- **资源节约**：Agent 不执行时不占用进程或连接；
- **状态集中**：所有执行历史、日志、成本都集中在 `heartbeatRuns` 表中，便于审计；
- **容错简单**：如果服务器重启，只需恢复 queued runs 和 scheduled retries，无需恢复进程状态。

代价是引入了队列延迟和并发控制的复杂性，但通过 `agentWakeupRequests` 的合并/延迟机制缓解了重复唤醒问题。

### 2. 为什么 Issue 执行锁采用“懒锁定（lazy locking）”？

在 `enqueueWakeup()` 中，新 run 被创建时**不立即**写入 `issues.executionRunId`，而是在 `claimQueuedRun()` 状态变为 running 时才锁定。这避免了：
- 队列中多个 runs 竞争同一 Issue 时的死锁；
- run 被快速取消后 Issue 仍被锁定的资源浪费。

对应代码注释明确标注为 “Fix A (lazy locking)”，见 `server/src/services/heartbeat.ts:6140`。

### 3. 为什么 Adapter 配置要区分 `adapterConfig` 和 `runtimeConfig`？

- `adapterConfig`：静态配置，如 API key、模型选择、指令文件路径，变更会触发配置版本记录（`agentConfigRevisions`）；
- `runtimeConfig`：动态策略，如心跳间隔、并发数、最大轮次续跑策略，允许 Agent 自身或系统动态调整而不产生配置快照。

这种分离使得“配置审计”和“运行时策略调优”互不干扰。

### 4. 为什么 Org Chart 不直接参与任务路由？

Org Chart（`reportsTo` 关系）主要用于：
- UI 可视化（OrgChart.tsx / org-chart-svg.ts）；
- 权限推导（CEO 拥有最高权限，普通 Agent 权限由 role + grants 决定）；
- 任务分配时由用户或自动化逻辑显式指定 `assigneeAgentId`，而非通过组织架构自动路由。

这种解耦避免了“组织架构变动导致任务路由错误”的风险，同时保留了组织视角的管理能力。

### 5. 为什么 Session 采用 TaskKey + TaskSession 双重机制？

- `agentRuntimeState.sessionId`：全局 fallback session，用于无 Issue 上下文的心跳（如定时自检）；
- `agentTaskSessions`：按 `taskKey`（通常是 Issue ID）隔离的会话，确保同一 Agent 处理不同 Issue 时拥有独立上下文；
- 显式 resume 支持：通过 `resumeFromRunId` 可以精确恢复到某次运行的会话状态。

这种设计让 Agent 既能“单线程专注一个任务”，也能“多任务间切换不丢上下文”。
