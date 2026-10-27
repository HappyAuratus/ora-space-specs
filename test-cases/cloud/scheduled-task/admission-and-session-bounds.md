# Cloud Scheduled Task 接纳、会话边界与历史保留

本文件登记 [Cloud Scheduled Task 根决策](../../decisions/cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md)
的核心验证义务。存储与调度 worker、公开 API 与 Web 管理面已随 [cloud#111](https://github.com/ora-space/cloud/pull/111)
合入 main，worker/API 领域测试随 [cloud#141](https://github.com/ora-space/cloud/pull/141)、
[cloud#145](https://github.com/ora-space/cloud/pull/145)、[cloud#152](https://github.com/ora-space/cloud/pull/152)、
[cloud#153](https://github.com/ora-space/cloud/pull/153)、[cloud#156](https://github.com/ora-space/cloud/pull/156)、
[cloud#159](https://github.com/ora-space/cloud/pull/159)、[cloud#183](https://github.com/ora-space/cloud/pull/183)、
[cloud#188](https://github.com/ora-space/cloud/pull/188)、[cloud#190](https://github.com/ora-space/cloud/pull/190) 合入。

证据分两类：

- 领域测试：cloud main [`0b2cc14`](https://github.com/ora-space/cloud/commit/0b2cc14e4b6dd717697b05019dd08afbce4cad20)
  上的直接测试（真实 PostgreSQL；`integration/` 下的测试用文件名简写，`internal/core/` 与 `frontend/` 下的单独注明）。
- 真实栈：2026-10-26 在 cluster Compose 栈上的端到端验收，部署为 cluster `28bac76`、cloud `da2125f`、desktop `d913440`
  的干净检出，真实 Controller / Node / Sandbox、官方 Claude 插件 0.4.1 与真实模型（经个人模型连接接入的
  Anthropic 协议端点）；验收开关只在该部署打开，任务经公开 API 创建并启用，不声明产物。播种或直接改库的步骤在对应行写明。

17 项义务全部 `Covered`。

## Occurrence Admission Must Be Idempotent And Bounded

- 风险：多副本调度、重启重放或重复评估为同一到期点接纳多个运行，或长时间停机恢复后逐点补记 `missed`。
- 前置状态：任务已启用，历史中已有若干运行与 `missed` 记录。
- 触发：调度 worker 的重复扫描、进程重启、部署更替、时钟回拨与迟到观察。
- 必须成立：同一 `(task_id, scheduled_at)` 至多接纳一个 Scheduled Run，重放返回既有结果；观察晚于
  `scheduled_at` 超过 10 分钟的到期点记 `missed(late_trigger)`，不执行不补跑；一次停机（跨过任意多个
  到期点）恢复后每个任务至多新增一条 `missed`（最近一个已过迟到窗口的到期点），它按一条普通记录
  参与 100 条保留；停用（含自动停用）与重新启用之间的到期点不评估、不补记。
- 禁止结果：同一 occurrence 两个运行、逐点补记 `missed`、把停用区间当作停机、失败或超时自动重试。
- 决策依据：本 ADR D2 与不变量 2、5；desktop 根决策 D4/D7 的 `missed` 语义与保留计数。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 重复扫描/重启/多副本对同一到期点只接纳一次 | `scheduled_task_admission_db_test.go`：`TestScheduledTaskAdmissionCreatesARun`（同一到期点重复评估只留一条 `admitted` occurrence 与一个 IssueRun）。`scheduled_task_admission_boundaries_test.go`：`TestScheduledTaskReplicasAdmitADuePointOnce`（两个 Store 各用独立连接池，8 个评估 pass 同时开始，只有一条 `admitted` occurrence 与一个 IssueRun，没有评估报错）、`TestScheduledTaskRestartReplayKeepsTheAdmission`（运行结束、任务不再 busy 后，新进程重放仍返回原结果）。真实栈：worker 每 10 秒扫描一次，每个到期点只有一条 occurrence 和一个 IssueRun | `Covered` |
| 迟到超阈值记 `missed` 且一次停机至多一条，计入 100 条保留 | `scheduled_task_admission_db_test.go`：`TestScheduledTaskMissedRecoverySingleOccurrence`（跨过多个到期点的停机两次评估后只有一条 `missed`）；`scheduled_task_schedule_retention_test.go`：`TestScheduledTaskHistoryRetentionPrunesOldestNonActive`（100 条 `missed` 按普通记录参与保留计数） | `Covered` |
| 停用与重新启用之间的到期点不评估、不补记 | `scheduled_task_schedule_retention_test.go`：`TestScheduledTaskDisabledIntervalIsNotEvaluated`（停用期间不写 occurrence，重新启用后不回填 `missed`）；`scheduled_task_admission_boundaries_test.go`：`TestScheduledTaskEnableAPIReanchorsTheDisabledInterval`（经公开 `disable` / `enable` 端点：手动停用保留旧锚点，停用区间跨过两个到期点，`enable` 把 `enabled_at` 重新锚定到数据库 `now()`，之后不接纳、不补记） | `Covered` |

## Deadline Cancellation Must Respect Session And Terminal Boundaries

- 风险：截止取消写入者对已在收尾或已终态的运行重复写取消/事实，或把成功完成、交付中的运行投影成超时。
- 前置状态：运行已接纳并持久化截止时刻；会话可能已结束、运行可能停留在交付重试。
- 触发：截止扫描到期，或用户与截止取消竞争。
- 必须成立：截止取消请求与截止取消事实只在会话尚未进入 `ending | ended` 且 IssueRun 未到终态时写入，
  事实以 occurrence 身份幂等且只写一次；任务历史展示为 `failed · timeout` 当且仅当截止取消事实存在
  **且** IssueRun 终态为 `cancelled`，其余一律按 IssueRun 自身终态展示（会话结束不等于终态，交付中的
  运行不被截止追改）；用户先于截止的取消不产生截止事实。
- 禁止结果：从 `cancel_requested_at` 反推超时、对已完成/失败/交付中的运行写超时投影、同一 occurrence
  两条截止事实、绕过取消请求直接伪造终态。
- 决策依据：本 ADR D4；IssueRun 根决策 D6 与有界启动收尾的既有语义。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 截止只在会话未结束且 IssueRun 未终态时写入，事实幂等只写一次 | `scheduled_task_worker_boundaries_test.go`：`TestScheduledTaskDeadlineWriterCancelsDueRunAndIsIdempotent`、`TestScheduledTaskDeadlineWriterSkipsTerminalRun`；`scheduled_task_deadline_boundaries_test.go`：`TestScheduledTaskDeadlineWriterSkipsEndingOrEndedThread`（`ending` / `ended` 不写）、`TestScheduledTaskDeadlineWriteKeepsIdentityBinding`（不改任务与 occurrence 身份）、`TestScheduledTaskDeadlineWriteRollsBackAsOneTransaction`（写事实失败时取消请求一并回滚，恢复后两项一起写入）。真实栈：截止到期后的下一轮扫描在同一时刻写入取消请求与截止事实 | `Covered` |
| 投影为 `failed · timeout` 当且仅当截止事实存在且终态为 `cancelled` | `scheduled_task_deadline_boundaries_test.go`：`TestScheduledTaskHistoryProjectsTimeoutOnlyForDeadlineCancelledRuns`（截止事实 + `cancelled` 才是 `failed / timeout`；截止事实配 `completed` / `failed` / 非终态、只有用户取消请求的 `cancelled` 都按 IssueRun 自身终态展示）。真实栈：被截止取消的运行在历史中为 `failed · timeout`，同栈正常完成与会话失败的运行分别为 `succeeded` 与 `failed · agent_failed` / `failed · agent_turn_failed` | `Covered` |
| 用户取消先于截止时不产生截止事实，交付中的运行不被追改 | `scheduled_task_deadline_boundaries_test.go`：`TestScheduledTaskDeadlineWriterSkipsUserCancelledRun`（已有用户取消请求时不写截止事实、不改写取消请求）、`TestScheduledTaskDeadlineWriterSkipsEndingOrEndedThread`（会话已 `ended`、运行在交付阶段时不写） | `Covered` |

## Headless Sessions Must End At The First Successful Turn

- 风险：无人值守运行沿用交互式空闲窗口，空等与 60 分钟截止相撞把成功运行投影成超时；或调度器伪造
  终态绕过会话收尾。
- 前置状态：无头单会话运行已接纳并开始执行任务 Prompt。
- 触发：会话执行接管到首个成功 `turnEnded`（`stop_reason` 非 `cancelled`）回执。
- 必须成立：定时运行的空闲窗口为 0——调度器按 Thread D4 原判据（成功 `turnEnded` 且没有 `queued`
  用户轮次）确认进入 `idle` 后，立即放出 `EndSession{idle_timeout}`，有界等待会话终态（既有结束确认
  窗口）；IssueRun 终态推导为 `completed`；非成功的 `TurnEnded` 不构成结束证据；定时运行整个生命
  周期 `canAppend = false`，任何 Thread 状态下追加一律 `409 thread_closed`，前端禁用输入并显示只读提示；结束
  请求经既有 Controller 中继，不新增契约。
- 禁止结果：调度器直接伪造会话终态或运行状态、复用交互式空闲窗口空等、为无头运行新增 reason 或
  第二套结束命令、在有 `queued` 用户轮次时结束会话。
- 决策依据：本 ADR D3（对 [Thread 根决策 D4](../../decisions/cloud/thread/0-durable-agent-thread-with-user-turns.md)
  的修改，仅限定时运行）；[Workflow 会话结束决策](../../decisions/cloud/workflow/20261019-workflow-session-end-after-initial-turn.md)
  为同判据先例；[个人模型发起者决策](../../decisions/cloud/thread/20261010-personal-model-initiator-controls-input.md)
  的结束权边界。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 首个成功 `turnEnded` 后放出 `EndSession{idle_timeout}`，IssueRun 终态 `completed` | `internal/core/agent_run_thread_end_db_test.go`：`TestScheduledIdleThreadEndsImmediately`。真实栈：定时运行的首个回合以 `turnEnded{stop_reason: end_turn}` 结束后，Cloud 恰好放出一条 `end_session{reason: idle_timeout}`，会话 `agent_session_ended / idle_timeout`，IssueRun `completed`，Revision 保存后 Workspace 删除，历史为 `succeeded` | `Covered` |
| 非成功 `TurnEnded` 不触发结束，运行由截止收敛 | `internal/core/scheduled_task_deadline_convergence_db_test.go`：`TestScheduledDeadlineConvergesAfterANonSuccessfulTurn`（回合以 `stopReason: cancelled` 结束后会话保持 `active`、没有 `EndSession{idle_timeout}`；截止写入者写入取消请求与截止事实，既有取消反应把 Thread 推到 `ending` 并放出恰好一条 `EndSession{cancelled}`）。之后会话以 `cancelled` 结束、IssueRun 推导为 `cancelled`（`integration/agent_run_release_giveup_test.go`：`TestG032_8_BusinessStatusMatrixSurvivesTheGiveUp` 的 `cancelled` 行），历史再按上表的投影测试展示为 `failed · timeout`。真实栈：回合进行中撤销模型凭据，回合以 `turnEnded{stop_reason: cancelled}` 结束，Cloud 不放出 `EndSession`、不进入 `idle` | `Covered` |
| 定时运行任何状态下追加均被 `409 thread_closed` 拒绝（`canAppend = false`），前端禁用输入 | `scheduled_task_headless_thread_test.go`：`TestScheduledThreadIsClosedInEveryState`（经真实 HTTP，`pending` / `active` / `idle` / `ending` / `ended` 下 Thread 读取均为 `canAppend: false`，追加均为 `409 thread_closed`，不写用户轮次、不放出命令）；`integration/agent_run_thread_message_test.go`：`TestThreadMessagePostRejectsScheduledRun`。前端只按 `canAppend` 渲染：`frontend/src/features/issues/thread/thread-panel.test.tsx` 的 “shows the frozen model and prevents non-owners from sending while administrators can end” 断言 `active` 而 `canAppend: false` 时输入框与发送按钮禁用并显示只读提示 | `Covered` |

## Eligibility Must Be Rechecked At Save And Every Admission

- 风险：保存时的资格（成员资格、模型配置、同意、目标可达性）在接纳时已失效，运行仍被接纳，或失败
  后回退、换模型、自动重试。
- 前置状态：任务定义已保存；创建者资格、模型配置或目标可能已变化。
- 触发：每次接纳的事务内校验，以及创建者成员资格失效、绑定 issue 删除、同意撤销。
- 必须成立：保存（启用的创建与 `enable`）逐项执行资格表，失败以该项的错误码拒绝且不留部分写入；
  每次接纳重新校验目标、成员资格与模型配置，失败映射到 `target_unavailable / model_unavailable` 并
  终止该 occurrence，不回退不重试；同意只在保存时校验——`enable` 缺少有效同意为 `409
  approval_policy_invalid`，启用的创建缺少或带了不支持的策略为 `400 invalid_request`，撤销同意即
  自动停用、接纳不再单独校验；Agent 插件版本由既有 IssueRun 用例在接纳时按 Space 当时值快照；
  创建者成员资格失效或绑定 issue 删除时任务自动停用，重新满足条件不自动恢复；同意撤销自动停用且
  既有运行事实不改写。
- 禁止结果：跳过校验直接接纳、失败换用其他模型或执行位置、把资格失效改判为其他原因、自动转移任务。
- 决策依据：本 ADR D5、D1 与不变量 6、8、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 保存与接纳的资格表逐项校验，失败按原因映射并终止 | `scheduled_task_admission_db_test.go`：`TestScheduledTaskNonModelBindingFailureIsTargetUnavailable`、`TestScheduledTaskSingleFailureDoesNotBlock`、`TestScheduledTaskModelBindingFailureIsDurable`；`scheduled_task_model_protocol_test.go`：`TestScheduledTaskIncompatibleModelOverrideFailsOccurrence`；`scheduled_task_contract_obligations_test.go`：`TestScheduledTaskDisabledCreateDoesNotPersistConsent`、`TestScheduledTaskReconsentAfterRevoke`。`scheduled_task_management_boundaries_test.go`：`TestScheduledTaskSaveChecksEachEligibilityRow`（启用的创建与 `enable` 各自逐项破坏资格表：issue 删除、issue 无 Project、Space 归档、Agent 退役、创建者模型未配置、模型覆盖不在目录，各得到该项错误码且不留部分写入）、`TestScheduledTaskSaveRequiresTheSupportedConsent`（同意这一项）。真实栈：接纳后、会话启动前撤销模型凭据，会话申请模型授权被拒（`model_access_denied`），运行显式 `failed / agent_failed`，不换模型、不重试 | `Covered` |
| 插件版本在接纳时按 Space 当时值快照，任务编辑不影响已接纳运行 | `scheduled_task_admission_boundaries_test.go`：`TestScheduledTaskAdmissionSnapshotsTheSpacePluginVersion`（Space 把 `desired_version` 从 `1.0.0` 改选为 `1.1.0` 后，下一次接纳的 `input.agentPluginVersion` 为 `1.1.0`，已接纳运行仍为 `1.0.0`）；`TestScheduledTaskModelBindingFreezesDefinitionSelection` 另证模型绑定在接纳时冻结。真实栈：经公开 API 修改任务定义（revision 1 → 2）后，已接纳运行的插件快照与 occurrence 的 `task_revision` 不变，之后的接纳使用新 revision | `Covered` |
| 创建者资格失效 / 绑定 issue 删除 / 同意撤销分别自动停用任务 | `scheduled_task_worker_boundaries_test.go`：`TestScheduledTaskCreatorInactiveAutoDisables`、`TestScheduledTaskIssueDeletedAutoDisables`；`scheduled_task_api_boundaries_test.go`：`TestScheduledTaskRevokeConsentDisables`；`scheduled_task_deadline_boundaries_test.go`：`TestScheduledTaskAutoDisableKeepsIdentityBinding`（自动停用只改启用状态与原因，创建者、绑定 issue、Agent 与定义不变）；`scheduled_task_admission_boundaries_test.go`：`TestScheduledTaskAutoDisableDoesNotRecover`（恢复成员资格或恢复 issue 后任务仍停用、原因不变、不接纳）、`TestScheduledTaskRevokeConsentKeepsExistingRunFacts`（撤销同意后已接纳运行的状态、phase、取消请求、input、version 与 occurrence 均不变） | `Covered` |

## History Retention Must Stay Bounded Per Task

- 风险：历史无限增长，或清理误删刚写入的事实、活跃运行与审计关联。
- 前置状态：任务的 occurrence / 运行 / missed 记录已接近或超过 100 条。
- 触发：第 101 条事实写入。
- 必须成立：写入第 101 条时在同一事务从最旧开始清理；活跃（`accepted / running`）运行不清理；被清理
  的 `missed` 按一条普通记录计数；IssueRun 及其 Thread / Revision 审计事实不受清理影响。
- 禁止结果：清理刚写入的事实、清理活跃运行、把 Cloud 侧清理扩散到 IssueRun 审计数据。
- 决策依据：本 ADR D6 与不变量 10。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 第 101 条写入后同事务从最旧清理，活跃运行与刚写入事实保留 | `scheduled_task_schedule_retention_test.go`：`TestScheduledTaskHistoryRetentionPrunesOldestNonActive`、`TestScheduledTaskHistoryRetentionSkipsActiveRun`。真实栈：任务已有 100 条历史（1 条真实接纳 + 99 条在验收库播种的 `missed`），由 worker 真实接纳第 101 条后总数仍为 100，被清理的是最旧一条，新接纳保留 | `Covered` |
| 清理不触碰 IssueRun / Thread / Revision 审计事实 | 真实栈：被清理的最旧 occurrence 关联一个已完成并保存 Revision 的定时运行；清理后该运行的 IssueRun、Thread 条目、Thread 命令、Revision 与 Node 执行记录数量不变，Git Server 上的交付 ref 仍指向原提交 | `Covered` |

## Visibility And Ownership Must Follow The Bound Issue

- 风险：任务定义与历史泄漏给无关成员，或绕过创建者管理边界修改无人值守授权。
- 前置状态：任务已创建，成员资格与归档状态可能变化。
- 触发：读取、创建、编辑、启停、删除、撤销同意及归档 Space。
- 必须成立：任务与历史对能读取绑定 issue 的有效租户成员可见；租户内找不到或无权限统一 `404
  not_found`，非成员由租户门禁 `403 membership_required` 拒绝；管理操作仅限任务创建者；绑定 issue
  与创建者创建时固定、不可修改（执行归属固定为 `cloud`、没有可写字段；Agent 可经 PATCH 修改）；
  Space 归档后不产生新接纳，保存按 Space API 根决策的统一 `404 not_found` 整体回滚。
- 禁止结果：把管理权开放给其他成员、按客户端提供的 scope 判定归属、归档 Space 后继续接纳运行。
- 决策依据：本 ADR D1 与不变量 9、11；[Space API 根决策](../../decisions/cloud/api-boundary/0-space-api-and-event-stream.md)
  的 `404` 语义与 [Project 归属决策](../../decisions/cloud/project/0-project-space-scoping.md) 的跨 Space 禁止。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 读取跟随 issue 可见性，拒绝语义与 Space API 根决策一致 | `scheduled_task_api_boundaries_test.go`：`TestScheduledTaskMissingTaskIsNotFound`（不存在的任务读写均为 `404 not_found`）；`scheduled_task_contract_obligations_test.go`：`TestScheduledTaskReadRequiresBoundIssue`（绑定 issue 删除后读取为 `404 not_found`）；`scheduled_task_management_boundaries_test.go`：`TestScheduledTaskReadFollowsTenantMembership`（非创建者成员可读任务与历史；已停用成员与外租户成员访问本租户路径为 `403 membership_required`；外租户成员在自己租户路径下指名该任务为 `404 not_found`） | `Covered` |
| 管理操作仅限创建者；绑定 issue 与创建者不可修改 | `scheduled_task_api_boundaries_test.go`：`TestScheduledTaskManagementIsCreatorOnly`（非创建者的停用、删除、PATCH 均为 `404 not_found` 且不改任务，创建者仍可操作）、`TestScheduledTaskVersionConflict`（过期 `version` 以 `409 version_conflict` 拒绝且无部分写入）；`scheduled_task_management_boundaries_test.go`：`TestScheduledTaskBindingIsImmutable`（PATCH 带 `issueId` 或 `creatorUserId` 时传输层为 `400 unknown_field`、任务不变；绕过传输层交给领域层也不改绑定） | `Covered` |
| Space 归档后不再接纳新的运行 | `scheduled_task_admission_db_test.go`：`TestScheduledTaskArchivedSpaceRejectsOpenCode`（未声明配置的插件，Space 归档后到期点记 `failed / target_unavailable`，不建 IssueRun）、`TestScheduledTaskArchivedSpaceRejectsDeclaredClaude`（已声明配置的 Claude 插件：同一夹具在 Space 未归档时正常接纳，归档后同样 `failed / target_unavailable` 且不建 IssueRun）。真实栈：Space 归档（公开 API 没有归档入口，经数据库设置 `archived_at`）后到期的 Claude 任务，由 worker 评估为 `failed / target_unavailable`，不建 IssueRun。保存侧：`TestScheduledTaskSaveChecksEachEligibilityRow` 的 Space 归档行，启用的创建与 `enable` 均为 `404 not_found` 且整体回滚 | `Covered` |
