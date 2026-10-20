# Cloud Scheduled Task 接纳、会话边界与历史保留

本文件登记 [Cloud Scheduled Task 根决策](../../decisions/cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md)
的核心验证义务。实现尚未合入 main，义务如实标为 `Missing`；每项落地完成时按直接证据更新状态。

## Occurrence Admission Must Be Idempotent And Bounded

- 风险：多副本调度、重启重放或重复评估为同一到期点接纳多个运行，或长时间停机恢复后逐点补记 `missed`。
- 前置状态：任务已启用，历史中已有若干运行与 `missed` 记录。
- 触发：调度 worker 的重复扫描、进程重启、部署更替、时钟回拨与迟到观察。
- 必须成立：同一 `(task_id, scheduled_at)` 至多接纳一个 Scheduled Run，重放返回既有结果；观察晚于
  `scheduled_at` 超过 10 分钟的到期点记 `missed(late_trigger)`，不执行不补跑；一次停机（跨过任意多个
  到期点）恢复后每个任务至多新增一条 `missed`（最近一个已过迟到窗口的到期点），它按一条普通记录
  参与 100 条保留；停用（含自动停用）与重新启用之间的到期点不评估、不补记。
- 禁止结果：同一 occurrence 两个运行、逐点补记 `missed`、把停用区间当作停机、失败或超时自动重试。
- 决策依据：根决策 D2 与不变量 2、5；根决策 D4/D7 的 `missed` 语义与保留计数。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 重复扫描/重启/多副本对同一到期点只接纳一次 | 尚未实现 | `Missing` |
| 迟到超阈值记 `missed` 且一次停机至多一条，计入 100 条保留 | 尚未实现 | `Missing` |
| 停用与重新启用之间的到期点不评估、不补记 | 尚未实现 | `Missing` |

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
- 决策依据：根决策 D4；IssueRun 根决策 D6 与有界启动收尾的既有语义。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 截止只在会话未结束且 IssueRun 未终态时写入，事实幂等只写一次 | 尚未实现 | `Missing` |
| 投影为 `failed · timeout` 当且仅当截止事实存在且终态为 `cancelled` | 尚未实现 | `Missing` |
| 用户取消先于截止时不产生截止事实，交付中的运行不被追改 | 尚未实现 | `Missing` |

## Headless Sessions Must End At The First Successful Turn

- 风险：无人值守运行沿用交互式空闲窗口，空等与 60 分钟截止相撞把成功运行投影成超时；或调度器伪造
  终态绕过会话收尾。
- 前置状态：无头单会话运行已接纳并开始执行任务 Prompt。
- 触发：会话执行接管到首个成功 `turnEnded`（`stop_reason` 非 `cancelled`）回执。
- 必须成立：调度器按[工作流会话结束决策](../../decisions/cloud/workflow/20261019-workflow-session-end-after-initial-turn.md)
  的同判据与同机制放出 `EndSession{idle_timeout}`，有界等待会话终态；IssueRun 终态推导为
  `completed`；非成功的 `TurnEnded` 不构成结束证据；创建者的追加请求在会话进入 `ending` 后按
  `409 thread_closed` 拒绝；结束请求经既有 Controller 中继，不新增契约。
- 禁止结果：调度器直接伪造会话终态或运行状态、复用交互式空闲窗口空等、为无头运行新增 reason 或
  第二套结束命令。
- 决策依据：根决策 D3；[工作流会话结束决策](../../decisions/cloud/workflow/20261019-workflow-session-end-after-initial-turn.md)
  （同判据与同机制）；[个人模型发起者决策](../../decisions/cloud/thread/20261010-personal-model-initiator-controls-input.md)
  的结束权边界。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 首个成功 `turnEnded` 后放出 `EndSession{idle_timeout}`，IssueRun 终态 `completed` | 尚未实现 | `Missing` |
| 非成功 `TurnEnded` 不触发结束，运行由截止收敛 | 尚未实现 | `Missing` |
| 无头运行的追加请求被既有 `409 thread_closed` 拒绝 | 尚未实现 | `Missing` |

## Eligibility Must Be Rechecked At Save And Every Admission

- 风险：保存时的资格（成员资格、模型配置、同意、目标可达性）在接纳时已失效，运行仍被接纳，或失败
  后回退、换模型、自动重试。
- 前置状态：任务定义已保存；创建者资格、模型配置或目标可能已变化。
- 触发：每次接纳的事务内校验，以及创建者成员资格失效、绑定 issue 删除、同意撤销。
- 必须成立：保存与每次接纳都执行同一资格表；失败映射到 `target_unavailable / model_unavailable /
  approval_policy_invalid` 并终止该 occurrence，不回退不重试；Agent 插件版本由既有 IssueRun 用例在
  接纳时按 Space 当时值快照；创建者成员资格失效或绑定 issue 删除时任务自动停用，重新满足条件不
  自动恢复；同意撤销自动停用且既有运行事实不改写。
- 禁止结果：跳过校验直接接纳、失败换用其他模型或执行位置、把资格失效改判为其他原因、自动转移任务。
- 决策依据：根决策 D5、D1 与不变量 6、8、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 保存与接纳的资格表逐项校验，失败按原因映射并终止 | 尚未实现 | `Missing` |
| 插件版本在接纳时按 Space 当时值快照，任务编辑不影响已接纳运行 | 尚未实现 | `Missing` |
| 创建者资格失效 / 绑定 issue 删除 / 同意撤销分别自动停用任务 | 尚未实现 | `Missing` |

## History Retention Must Stay Bounded Per Task

- 风险：历史无限增长，或清理误删刚写入的事实、活跃运行与审计关联。
- 前置状态：任务的 occurrence / 运行 / missed 记录已接近或超过 100 条。
- 触发：第 101 条事实写入。
- 必须成立：写入第 101 条时在同一事务从最旧开始清理；活跃（`accepted / running`）运行不清理；被清理
  的 `missed` 按一条普通记录计数；IssueRun 及其 Thread / Revision 审计事实不受清理影响。
- 禁止结果：清理刚写入的事实、清理活跃运行、把 Cloud 侧清理扩散到 IssueRun 审计数据。
- 决策依据：根决策 D7 与不变量 10。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 第 101 条写入后同事务从最旧清理，活跃运行与刚写入事实保留 | 尚未实现 | `Missing` |
| 清理不触碰 IssueRun / Thread / Revision 审计事实 | 尚未实现 | `Missing` |

## Visibility And Ownership Must Follow The Bound Issue

- 风险：任务定义与历史泄漏给无关成员，或绕过创建者管理边界修改无人值守授权。
- 前置状态：任务已创建，成员资格与归档状态可能变化。
- 触发：读取、创建、编辑、启停、删除、撤销同意及归档 Space。
- 必须成立：任务与历史对能读取绑定 issue 的有效租户成员可见，不存在与无权限统一 `404 not_found`；
  管理操作仅限任务创建者；绑定 issue 与执行归属创建时固定、不可修改；Space 归档后不产生新接纳。
- 禁止结果：把管理权开放给其他成员、按客户端提供的 scope 判定归属、归档 Space 后继续接纳运行。
- 决策依据：根决策 D1 与不变量 9、11；[Space API 根决策](../../decisions/cloud/api-boundary/0-space-api-and-event-stream.md)
  的 `404` 语义与 [Project 归属决策](../../decisions/cloud/project/0-project-space-scoping.md) 的跨 Space 禁止。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 读取跟随 issue 可见性，`404` 语义与 Space API 根决策一致 | 尚未实现 | `Missing` |
| 管理操作仅限创建者；绑定 issue 与执行归属不可修改 | 尚未实现 | `Missing` |
| Space 归档后不再接纳新的运行 | 尚未实现 | `Missing` |
