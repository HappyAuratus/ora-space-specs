---
status: proposed
---

# Internal Control gRPC 增加插件、Agent 会话与 Revision 交付三类执行，新增 AgentRunService 承载 Thread 事件、Thread 命令与上传授权

`ExecutionService` 的执行输入与结果以只增不改的方式增加插件安装/移除、Agent 会话与 Revision 交付；
`ClaimWork` 除租户级 clone 外还返回 IssueRun 放出的会话与交付工作项，并带上目标 Node。新增
`AgentRunService`：`TakeOverThreadEvents` 按序批量接管会话执行的非终态 Thread 事件，
`ClaimThreadCommands`/`RecordThreadCommandDelivered` 把用户消息与结束请求交给 Controller，
`GrantRevisionUpload` 签发上传授权。`Watch` 增加 `ThreadCommandAvailable`。

本 ADR 处于 `proposed`，继承 [gRPC 契约根决策](0-cloud-owned-internal-grpc-contract.md)、
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
- 执行登记的持久化由 Cloud 实现决定（可新增通用执行表，`clone_executions` 保持原义）；本文要求的是
  规则：派发前登记、输入与目标由 Cloud 校验、结果冲突不覆盖、恢复读取（`GetDispatch`、
  `ListPendingDispatches`）覆盖全部新种类。

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

## 不变量

1. `v1` 已发布的消息与方法不变。
2. 会话与交付执行的目标 Node 由 Cloud 决定。
3. Thread 事件只按序号连续接管；终态事件在其前面全部事件接管之后接管。
4. 任何 `EventAck` 都晚于对应事件的 Cloud 提交。
5. Thread 命令在 Cloud 持久化后才可领取；重复投递不改变 Node 状态。
6. 上传授权不在 Cloud、Controller 或 Node 持久化。

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

## 落地

proto 变更与生成物、`internal/controlgrpc` 映射、PostgreSQL 集成测试覆盖：批量接管的原子性与顺序冲突、
终态事件先于 Thread 事件时拒绝、命令重复投递登记幂等、授权只对未结束交付签发。desktop 按 commit 锁定新契约。
