# Cloud Scheduled Task 接纳的任务对话准入

本文件登记 [任务对话准入决策](../../../decisions/cloud/scheduled-task/20261027-task-conversation-admission.md)
的核心验证义务。实现尚未合入 main，义务如实标为 `Missing`；每项落地完成时按直接证据更新状态。

## Admission Must Refuse While The Task Conversation Refuses Agent Input

- 风险：定时接纳绕过任务对话准入：对话已暂停或有未解决的失败消息时，occurrence 记为接纳、新 IssueRun
  永久停在 `queued`，此后每个到期点都记 `skipped_busy`；有先前的 waiting 消息时，定时运行越过它先执行。
- 前置状态：任务已启用且资格表其余各项满足；绑定 issue 的任务对话处于下列之一：已暂停、有未解决的
  失败消息、有先前接收仍在等待的消息。
- 触发：调度 worker 评估到期点，并在之后重复评估。
- 必须成立：该到期点恰有一条 `failed` occurrence，已暂停或有失败消息时原因为 `task_paused`，有 waiting
  消息时为 `task_handoff_pending`；不创建 IssueRun；重复评估不改写这条记录，也不改记为 `missed`；任务
  对话的消息、暂停点与队列顺序不变；对话恢复后，下一个到期点按正常流程接纳。
- 禁止结果：接纳后留下永久排队的运行、越过 waiting 消息执行、记成 `target_unavailable` 或
  `skipped_busy`、重试或补跑被拒的到期点、为定时输入改写或排入任务对话队列。
- 决策依据：本 ADR D1、D2 与不变量 1、3、4；[同一任务 Agent 执行串行决策](../../../decisions/cloud/issue-run/20261018-serialize-task-agent-handoffs.md)
  D1/D3 的准入与错误码语义。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 对话已暂停时到期，一次写成 `failed / task_paused`，不建 IssueRun，重复评估不变 | 尚未合入 main | `Missing` |
| 有未解决的失败消息时到期，一次写成 `failed / task_paused`，不建 IssueRun，重复评估不变 | 尚未合入 main | `Missing` |
| 有 waiting 消息时到期，一次写成 `failed / task_handoff_pending`，不建 IssueRun，waiting 消息仍在队首 | 尚未合入 main | `Missing` |
| 对话正常时同一夹具照常接纳（对照） | 尚未合入 main | `Missing` |

## Scheduled Admission And Direct Input Must Agree On The Conversation Gate

- 风险：定时入口另写一套对话判断，与直接输入入口的条件或错误码逐渐分叉，同一对话状态在两条路径上
  得到不同结论。
- 前置状态：同一绑定 issue 的任务对话处于任一拒绝或接受状态。
- 触发：分别经直接输入入口发送消息与定时接纳评估到期点。
- 必须成立：直接输入以 `409 task_paused` / `409 task_handoff_pending` 拒绝的状态，定时接纳以同名原因
  记失败；直接输入接受的状态，定时接纳不因对话被拒。
- 禁止结果：两条路径对同一状态一拒一收，或使用不同的原因码。
- 决策依据：本 ADR D1、D2 与不变量 2。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 两条路径共用同一准入判断，拒绝原因同名 | 尚未合入 main | `Missing` |

## History Must Point Conversation Refusals At The Task Conversation

- 风险：任务历史只显示原始原因码，创建者看不出该去任务对话里处理。
- 前置状态：任务历史中有原因为 `task_paused` 或 `task_handoff_pending` 的失败记录。
- 触发：创建者打开任务的运行历史。
- 必须成立：两个原因分别展示为指向任务对话的说明；其余失败原因的展示不变。
- 禁止结果：两个原因显示为原始代码，或被展示成目标、模型配置不可用。
- 决策依据：本 ADR D2 的选择依据与落地顺序第 2 步。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 两个原因在任务历史中展示为任务对话说明 | 尚未合入 main | `Missing` |
