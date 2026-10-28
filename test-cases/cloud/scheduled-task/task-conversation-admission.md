# Cloud Scheduled Task 接纳的任务对话准入

本文件登记 [任务对话准入决策](../../../decisions/cloud/scheduled-task/20261027-task-conversation-admission.md)
的核心验证义务。接纳预检、两个失败原因与任务历史文案已随 [cloud#193](https://github.com/ora-space/cloud/pull/193)
合入 main，对话不变、恢复后接纳与两条入口联合验证的测试随 [cloud#199](https://github.com/ora-space/cloud/pull/199) 补齐；
证据以 cloud [`60c1ae7`](https://github.com/ora-space/cloud/commit/60c1ae7b721caf2d2c65ee03ff77e8a68bab7d94)
为准，测试文件路径相对 cloud 仓库根目录。8 项义务全部 `Covered`。

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
| 对话已暂停时到期，一次写成 `failed / task_paused`，不建 IssueRun，重复评估不变 | `integration/scheduled_task_admission_db_test.go`：`TestScheduledTaskPausedConversationRejectsAdmission`（`paused_run_id` 非空，连续两次评估后恰有一条 `failed / task_paused` occurrence，`run_id` 为空，没有以该任务为触发的 IssueRun；去掉预检时记 `admitted` 并建出运行，测试失败） | `Covered` |
| 有未解决的失败消息时到期，一次写成 `failed / task_paused`，不建 IssueRun，重复评估不变 | 同文件：`TestScheduledTaskFailedMessageRejectsAdmission`（先前一条 `failed` 消息，断言同上） | `Covered` |
| 有 waiting 消息时到期，一次写成 `failed / task_handoff_pending`，不建 IssueRun，waiting 消息仍在队首 | 同文件：`TestScheduledTaskWaitingMessageRejectsAdmission`（断言同上，且 `seq=1` 的消息仍为 `waiting`） | `Covered` |
| 对话正常时同一夹具照常接纳（对照） | 同文件：`TestScheduledTaskAdmissionCreatesARun`（同一 `scheduledTaskStore` / `seedEnabledScheduledTask` 夹具、没有任务对话，记 `admitted` 并建出一个 IssueRun） | `Covered` |
| 拒绝不改写任务对话的消息、暂停点与队列顺序 | 上面三个 `TestScheduledTask*RejectsAdmission`：两次评估前后比较整个任务对话（对话行与全部消息）的快照，必须完全相同；另外分别断言 `paused_run_id` 仍指向原运行、失败消息仍为 `failed / agent_unavailable`、`seq=1` 的消息仍为 `waiting`（拒绝时清除暂停点或改写失败消息，测试失败） | `Covered` |
| 对话恢复后，下一个到期点按正常流程接纳，被拒的到期点不补跑 | `integration/scheduled_task_conversation_gate_test.go`：`TestScheduledTaskAdmitsTheNextDuePointAfterConversationRecovery`（经公开接口 `POST …/conversation/resume` 恢复已暂停的对话、`POST …/conversation/messages/{id}/cancel` 结清失败消息；被拒后恢复，下一个到期点连续两次评估恰好接纳一次并建出一个 IssueRun，被拒的到期点仍为 `failed / task_paused` 且没有运行；恢复后定时侧仍拒绝时测试失败）。waiting 消息不经恢复接口，它不再等待后的接纳由下一组的已结清对话用例证明 | `Covered` |

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
| 两条路径共用同一准入判断，拒绝原因同名 | `integration/scheduled_task_conversation_gate_test.go`：`TestScheduledAdmissionAndDirectInputAgreeOnTheConversationGate`（同一夹具、同一对话状态上先经直接输入 `POST …/runs` 发起 Agent 运行，再由调度 worker 评估到期点；覆盖无对话、已结清、已暂停、有失败消息、有 waiting 消息五种状态：拒绝的状态两侧分别为 `409 <原因>` 与 `failed / <原因>`，原因同名且对话不变；接受的状态两侧都接纳。只在一侧增加拒绝条件或改变原因码时测试失败） | `Covered` |

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
| 两个原因在任务历史中展示为任务对话说明，其余原因保持原码 | `frontend/src/features/scheduled-tasks/scheduled-task-history.test.tsx`：`points a task conversation refusal at the conversation instead of a raw code`（两条说明文案出现，原始 `task_paused` 不出现）、`keeps other failure reasons as their code`（`model_unavailable` 仍显示原码） | `Covered` |
