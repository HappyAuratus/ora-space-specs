---
status: approved
---

# Internal Control gRPC 增加插件、Agent 会话与 Revision 交付三类执行，新增 AgentRunService 承载 Thread 事件、Thread 命令与上传授权

`ExecutionService` 的执行输入与结果以只增不改的方式增加插件安装/移除、Agent 会话与 Revision 交付；
`ClaimWork` 除租户级 clone 外还返回 IssueRun 放出的会话与交付工作项，并带上目标 Node。新增
`AgentRunService`：`TakeOverThreadEvents` 按序批量接管会话执行的非终态 Thread 事件，
`ClaimThreadCommands`/`RecordThreadCommandDelivered` 把用户消息与结束请求交给 Controller，
`GrantRevisionUpload` 签发上传授权。`Watch` 增加 `ThreadCommandAvailable`。

本 ADR 处于 `approved`（已评审，待实现），继承 [gRPC 契约根决策](0-cloud-owned-internal-grpc-contract.md)、
[控制监听器保活](20260924-control-listener-keepalive.md)、[Workspace 操作与 Node 代报](20260926-workspace-operations-and-controller-reported-nodes.md)
与 [ListLiveSandboxes](20260926-list-live-sandboxes.md)。业务语义分别由
[IssueRun](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)、
[Thread](../thread/0-durable-agent-thread-with-user-turns.md)、
[Revision](../revision/0-cloud-owned-object-store-and-verified-revisions.md) 与
[operation 后续决策](../operation/20260928-plugin-step-and-run-workspace-release.md)决定，本文只定契约形状与
事务边界。兼容性边界：`v1` 已发布的消息与方法不变，buf breaking 检查必须通过。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D1：proto 是唯一契约源，`v1` 只增不改 | 继承，不变 |
| 根 D3：写提交携带 `submission_id`，重传返回原结果 | 继承，扩展到本文全部写方法 |
| 根 D4：派发前登记、终态事件与收据原子接管、`ClaimWork` 纯读、每请求一个执行 | 继承；`ClaimWork` 增加工作项种类，“每请求一个执行”只适用于租户级 clone（D1） |
| 根 D5：`Watch` 至多一次信号 | 继承，新增 `ThreadCommandAvailable`（D5） |
| 根 D6：错误映射一处产生，七个枚举值 | 继承，不新增枚举值 |
| Workspace 操作 D3：operation 的 clone 步骤经 `RecordDispatch` 登记 | 继承，扩展到 plugin 步骤（D1） |
| 根 D4：事件只有终态结果 | 修改：会话执行有非终态 Thread 事件，由新方法接管（D2） |

## 决策

### D1：执行种类与工作项

`ExecutionInput.spec` 与 `ExecutionResult.outcome` 增加 oneof 成员：

| 执行 | 输入 | 终态结果 | 如何领取 |
|---|---|---|---|
| 插件安装/移除 | `InstallPlugins`/`RemovePlugins`（插件载荷列表） | `PluginsResult{items[]}`；执行整体无法运行时 `PluginsFailed{plugin_root_unavailable \| interrupted}` | `ClaimOperation`，处于 `plugin` 步骤的 operation |
| Agent 会话 | `AgentSession{agent_plugin_id, agent_plugin_version, checkout_execution_id, git_identity, initial_turn{turn_id, content}}` | `AgentSessionEnded{reason, detail}` | `ClaimWork` |
| Revision 交付 | `DeliverRevision{session_execution_id, checkout_execution_id, base_commit, revision_ref, bundle_key, history_key}` | `RevisionDelivered`/`RevisionUnchanged`/`RevisionFailed` | `ClaimWork` |

- `WorkItem` 增加 `target{workspace_id, sandbox_instance_id, node_id}`：会话与交付的目标 Node 由 Cloud 按运行
  Workspace 当前 generation 决定，`RecordDispatch` 的 `node_id` 必须与之相等。租户级 clone 的工作项不带 target。
- 会话与交付工作项的 `operation_id` 是 IssueRun ID；一个运行同时至多一个会话执行，交付执行失败后重试是
  新执行（IssueRun D5）。
- `checkout_execution_id` 引用运行 Workspace 的 clone 执行：Node 由自己的账本解析 checkout 路径，Controller
  与 Cloud 不传递 Node 本地路径（协议根不变量 5、6）。
- `OperationSnapshot` 增加 `plugin_input`（进入 `plugin` 步骤时固定的 `InstallPlugins`/`RemovePlugins` 输入，
  集合为空时不设置）与 `plugin_executions`（该步骤已登记的执行）；`DeferReason` 增加
  `PLUGIN_EXECUTION_FAILED`（重试新执行）与 `PLUGIN_RESULT_UNKNOWN`（保持 `blocked`），与 clone 的两个原因对称。
- `RevisionFailed.reason` 另有 `VERIFICATION_FAILED`，只由 Cloud 在对象校验失败时记录，Node 不会报告。
- `ClaimWork` 仍是纯读，返回最早的可领取工作项；归属由 `RecordDispatch` 决定。
- 新种类的登记存放在通用执行表中（D6），`clone_executions` 保持原义；规则与 clone 相同：派发前登记、
  输入与目标由 Cloud 校验、结果冲突不覆盖、恢复读取（`GetDispatch`、`ListPendingDispatches`）覆盖全部新种类。

### D2：Thread 事件按序批量接管

`AgentRunService.TakeOverThreadEvents{submission_id, epoch, operation_id, execution_id, events[{sequence, record, turn_id?}]}`：

- 一次最多 64 个事件，在一个事务中全部写入 Thread 或全部不写；返回已接管的最大 `sequence`。
- 顺序约束：批次的第一个事件必须是该执行已接管的最大序号 + 1；序号已接管且内容相同的事件视为重放，
  跳过；内容不同或出现缺口时 `ABORTED`/`CONFLICT`，不写入任何事件。Controller 收到冲突时不确认，
  由 Node 重放恢复顺序。
- 会话执行的终态事件仍经 `TakeOverNodeEvent` 接管，且必须在其前面所有序号都已接管之后；否则 `CONFLICT`。
  这保证 Thread 的最后一条记录先于“会话已结束”落库。
- 只有返回成功后 Controller 才能对批次中的序号发送 `EventAck`，与根 D4 相同。

为什么批量：定型记录在工具密集时每秒可达数十条，逐条一次 RPC 与事务会让 Thread 落后于 Agent；批量不改变
“先持久化再确认”的边界。为什么要求连续：Thread 条目的 `seq` 由 Cloud 连续分配（Thread D1），乱序接管会
把 Node 的顺序重排成错误的对话顺序。

### D3：Thread 命令由 Cloud 持久化，Controller 投递后登记

- `ClaimThreadCommands{epoch, limit}` 返回尚未登记投递的命令，按运行分组、组内按创建顺序：
  `SubmitUserTurn{command_id, run_id, turn_id, content}` 与 `EndSession{command_id, run_id, reason}`，以及该运行
  会话执行的 `execution_id` 与目标 Node。纯读，要求当前租约 epoch。
- Controller 把命令发给 Node，收到 Node 的持久受理确认后调用
  `RecordThreadCommandDelivered{submission_id, epoch, command_id, execution_id}`；之后该命令不再返回。
- 投递可能重复（登记前 Controller 重启）：Node 按 `turn_id`/`command_id` 去重，重复投递无副作用。
- 会话执行尚未登记派发时命令留在 Cloud；首条 prompt 不是命令，它在会话执行输入中。

### D4：上传授权

`AgentRunService.GrantRevisionUpload{epoch, execution_id}` 返回
`grants[{object_key, url, method, headers, expires_at}]`：只对处于登记状态、尚无结果的交付执行签发，
对象键必须是该执行输入中的键。它不是写提交，不携带 `submission_id`，也不在 Cloud 留下记录；重复调用签发
新的授权。授权内容不得写入 Controller 日志。

### D5：`Watch` 增加 `ThreadCommandAvailable{run_id}`

命令写入事务提交后发送，语义同其他信号；会话与交付工作项沿用 `WorkAvailable{operation_id = run_id}`。
Controller 在流不可用时按周期调用 `ClaimThreadCommands` 兜底。

### D6：Cloud 内的持久化与事务边界

Cloud 内部分为两类表。**控制面表**承载“交给 Controller 的工作与 Node 的证据”，只在 internal control 动作与
下表列出的业务事务中写入；**业务表**承载 IssueRun、Thread、Agent 等领域状态，只在业务转换中写入。两类表
之间只通过下面的命名函数交互，函数都在调用方的事务内执行，不开启自己的事务。

| 表 | 类别 | 内容 |
|---|---|---|
| `execution_work` | 控制面 | `id uuid`、`run_id`、`kind`（`agent_session \| deliver_revision`）、`input jsonb`、`target jsonb`、`available_at`、`execution_id`（登记后写入）。同一运行同时至多一个未登记的工作项（部分唯一索引） |
| `node_executions` | 控制面 | 新种类的执行登记：`execution_id`、`kind`、`operation_id`（operation 或 IssueRun）、`work_id`（可空）、`node_id`、`input`、`result`、`dispatched_epoch`、`last_event_sequence` |
| `node_event_receipts` | 控制面 | `(execution_id, sequence)` 主键与原始事件；Thread 事件与终态事件共用，是确认的唯一依据 |
| `thread_commands` | 控制面 | `id uuid`（即 `command_id`）、`run_id`、`kind`、`body jsonb`、`created_at`、单调 `queue_sequence`、`delivered_at`、`delivered_execution_id`。同事务的 created_at 相同，领取按 queue_sequence 保持接受顺序；升级保留旧 created_at/id 顺序 |
| `revisions` | 控制面 | 见 [Revision D4](../revision/0-cloud-owned-object-store-and-verified-revisions.md) |
| `issue_runs` 新列、`space_agents`、`thread_entries`、`users` 的 git 身份列 | 业务 | 见 IssueRun、Thread、git 身份决策 |

控制面提供给业务转换调用的函数：

| 函数 | 作用 | 提交后 |
|---|---|---|
| `enqueueExecutionWork(t, run, kind, input, target, availableAt)` | 放出会话或交付工作项；输入在此固定 | `WorkAvailable` |
| `enqueueThreadCommand(t, run, command)` | 放出 `SubmitUserTurn`/`EndSession`，返回 `command_id` | `ThreadCommandAvailable` |
| `createRunWorkspace(t, run)` / `deleteRunWorkspace(t, run)` | 以 `(issue_run_id, kind)` 幂等地创建运行 Workspace 及其 operation；Project 忙时返回 `busy` 值而不是错误，事务照常提交 | `OperationAvailable` |

业务提供给控制面事务调用的钩子：

| 钩子 | 在哪个控制面事务中调用 | 业务上做什么 |
|---|---|---|
| `runWorkspaceSettled(t, run, ready \| failed)` | 运行 Workspace 的 `create_workspace` 进入终态的事务 | `starting` 或按 IssueRun D3 进入 `releasing` |
| `threadEventsTakenOver(t, run, execution, events)` | `TakeOverThreadEvents`，在收据写入之后 | 分配 `seq` 写 `thread_entries`、推进 `thread_state`、把轮次标为 `delivered`、`running` |
| `sessionEnded(t, run, execution, ended)` | 会话终态的 `TakeOverNodeEvent` | Thread `ended`、排队轮次 `discarded`、进入 `delivering` 并放出交付工作项 |
| `deliverySettled(t, run, execution, saved(revisionId) \| unchanged(revisionId) \| failed(reason))` | 交付终态的接管事务（`revisions` 行已写入之后） | 进入 `releasing` 或按 IssueRun D5 退避放出新交付 |
| `runWorkspaceDeleted(t, run)` | 运行 Workspace 的 `delete_workspace` 成功的事务 | `done` |

- 顺序、重放与冲突只由控制面判断：钩子只收到按序号连续、首次接管的事件；重放不会再次调用钩子。
- 钩子返回错误时整个事务回滚，Controller 不确认事件，Node 会重放；因此钩子必须对同一事件集合是确定的。
- 业务表的 `seq`、`thread_state` 与阶段只由钩子与业务 API 写；控制面代码不直接写业务表，反之亦然。

运行 Workspace 初始化成功后，生命周期 maintenance operation 交接为该 IssueRun 的 maintenance binding，
提升 control epoch，Node 确认后才签发会话或交付许可。这个占用持续到运行 Workspace 释放，不占
`one_project_operation`；用户操作会话与系统运行互斥。许可仍校验当前 Node/sandbox、运行和执行身份、
epoch、触发者权限及仓库凭据可用性。触发者来自同租户同 Issue 的 `run.enqueued` 用户证据，固定为
Workspace creator；不能改用 Issue 创建者。缺失或歧义证据返回 `run_actor_required`，交给业务侧补齐。

控制面提供上述五个命名钩子的同事务接缝；业务实现必须使用调用方事务，不能在钩子里再次调用会
开启事务的 Store 包装方法。`deliverySettled(saved/unchanged)` 仍须在 Revision 校验登记完成之后调用；
仅接管 Node 上传成功声明不能触发成功业务转换。cloud#41 目前覆盖 failed 分支，成功分支留待 M3。

为什么用命名函数而不是共享表：Thread 的顺序规则（连续序号、重放）与业务规则（轮次状态、空闲判定）由
不同的人演进，若双方都直接写同一张表，任何一侧改动都可能破坏另一侧的不变量；函数边界让每张表只有一个
写入者，事务仍然只有一个。

## 不变量

1. `v1` 已发布的消息与方法不变。
2. 会话与交付执行的目标 Node 由 Cloud 决定。
3. Thread 事件只按序号连续接管；终态事件在其前面全部事件接管之后接管。
4. 任何 `EventAck` 都晚于对应事件的 Cloud 提交。
5. Thread 命令在 Cloud 持久化后才可领取；重复投递不改变 Node 状态。
6. 上传授权不在 Cloud、Controller 或 Node 持久化。
7. 每张表只有一类写入者：控制面表只由控制面代码写，业务表只由业务转换写；二者只经 D6 的函数与钩子交互，且都在同一事务内。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Thread 事件复用 `TakeOverNodeEvent`，每个事件一次调用 | 不新增方法；吞吐不足（D2），且该方法的语义是终态结果 |
| Cloud 经 `Watch` 流推送用户消息给 Controller | 延迟最低；流至多一次、不持久化，丢消息无法发现（根 D5） |
| 会话与交付作为 operation 步骤 | 统一在 `ClaimOperation`；会话持续期间占用 `one_project_operation`（IssueRun D3） |
| 上传授权在工作项中一起返回 | 少一次调用；授权会过期，而工作项在 Cloud 中可能长期等待领取 |

## 本决策未解决的问题

- Thread 事件接管的限速与单运行的事件量上限。
- 多 Controller worker 时命令投递的分区。
- `node_executions` 是否在后续把 clone 并入（本文保持 `clone_executions` 不变）。

## 落地

proto 变更与生成物、`internal/controlgrpc` 映射、PostgreSQL 集成测试覆盖：批量接管的原子性与顺序冲突、
终态事件先于 Thread 事件时拒绝、命令重复投递登记幂等、授权只对未结束交付签发。desktop 按 commit 锁定新契约。
