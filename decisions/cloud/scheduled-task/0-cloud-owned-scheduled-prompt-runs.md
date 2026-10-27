---
status: implemented
---

# Cloud Scheduled Task 由服务端持久调度产生幂等触发，每次触发经既有 IssueRun 链执行绑定 issue 下的无头 Prompt 单会话

> 历史决策：D5 的资格表与接纳失败原因已由[任务对话准入决策](20261027-task-conversation-admission.md)
> 修改：每次接纳还要校验绑定 issue 的任务对话接受新的 Agent 输入，被拒时记 `task_paused` 或
> `task_handoff_pending`。其余规则保持有效。

用户可以在 Web 端创建周期性的定时任务（Scheduled Task）：把一条随任务版本冻结的 Prompt、一个执行
Agent 与一份无人值守同意，绑定到租户内的一个 Issue 上；由 Cloud 的持久调度器在到期时产生稳定的
occurrence，并在同一事务里幂等接纳为该 issue 下的一次 agent IssueRun。运行完全复用既有的
IssueRun 编排（运行 Workspace、Thread、Revision、交付与释放）、有界启动收尾与残留回收，调度器
不新增第二套 Agent 运行入口，也不解释 Prompt 之外的执行语义。每次运行固定 60 分钟，超时取消依赖
Cloud 侧的取消写入者；任务所有者的个人模型连接承担模型费用，运行以任务所有者的 Git 身份提交。

本 ADR 处于 `implemented`：取消写入者、存储与调度 worker、公开 API 与 Web 管理面均已合入 cloud main，
2026-10-26 在 cluster Compose 真实栈上完成端到端验收；正文已按最终实现同步，偏离 approved 决策之处
在对应决策中按实际行为改写，并汇总在[落地结果](#落地结果)。它承接 [desktop Scheduled Task 根决策](../../desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md)
显式留给 Cloud 决策的契约：任务的账号与存储归属、可见范围、无头单会话 API、通知、发起者归属与
费用，以及 Desktop `cloud` 选项的解除禁用条件。首期只做无头 Prompt 单会话：不接 Workflow 目标、
不解析 `executorSlot`、不接入 [Workflow 根决策](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md)
的 Server Workflow 执行链。核心测试用例见
[`test-cases/cloud/scheduled-task/admission-and-session-bounds.md`](../../../test-cases/cloud/scheduled-task/admission-and-session-bounds.md)，
17 项义务全部 `Covered`。

涉及 Cloud 的 `internal/core`（定时任务定义、occurrence、调度 worker）、PostgreSQL 迁移与公开 API。
兼容性边界：manual run、`@` Confirm、Workflow 路径与 desktop 决策全部不变；`issue_runs` 只增加
可选的触发来源标识，不改变既有列义与状态机。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [desktop Scheduled Task 根决策](../../desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md) D2：`ScheduledOccurrence{taskId, scheduledAt}` 唯一标识一次计划触发，同一 occurrence 至多接纳一个 Scheduled Run，幂等是各端内部规则 | 继承，不变：Cloud 侧以 `(task_id, scheduled_at)` 唯一约束在同一事务内幂等接纳（D2、D3） |
| 同上 D3：Cloud 首期复用既有 Agent `IssueRun` / `Thread` 应用用例，每个任务绑定一个 issue，每次触发在该 issue 下新建 agent IssueRun，不接 Workflow / `executorSlot`，不另建运行入口 | 继承，不变：这是本文 D3 的硬边界 |
| 同上 D4/D7：迟到超 10 分钟记 `missed` 不补跑；失败、超时与 `missed` 不自动重试；禁用只阻止未来接纳；删除任务保留审计事实 | 继承，不变；Cloud 侧 `missed` 原因只有 `late_trigger`（D2） |
| 同上 D5：`cloud` 的未来 occurrence 只由 Cloud 持久调度器产生；创建后不可迁移为 `local`；60 分钟超时取消依赖 Cloud 侧取消写入者，写入者就绪前 `cloud` 不得可选 | 继承：D4 落实截止与取消写入者的接缝；D8 冻结解除禁用条件 |
| 同上 D7：运行记录关联 task、revision、occurrence 与目标运行 ID；状态集 `accepted/running/succeeded/failed/cancelled/skipped_busy/skipped_capacity`；每任务保留最近 100 条 | 继承并落地：Cloud 侧任务历史记录投影既有 IssueRun 终态，不复制状态机；100 条保留只作用于任务历史视图（D6） |
| 同上 D8：保存与每次接纳均校验目标、模型配置与无人值守授权；失败明确结束，不回退 | 继承并落地为 Cloud 侧的资格校验表与失败原因映射（D5） |
| 同上 D1：schedule 首版只有三种受限周期，不存储或解释 cron 字符串；时区作为持久化字段留给 Cloud | 继承并落定：任务持久化创建时固定的 IANA 时区，DST 规则沿用根决策 D6（D2） |
| [cloud/issue-run 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)：每次 Agent IssueRun 一个一次性运行 Workspace，`phase` 状态机、`status` 与 Revision 分别记录、取消是请求、`one_project_operation` 不被绕过 | 继承，全部不变：调度触发的运行与手动运行走同一条编排，本文不新增阶段或状态值（D3） |
| [有界启动与未派发撤回](../issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)：启动预算、结束确认窗口与残留账本按数据库时间有界收尾 | 继承，不变：定时运行不重置、不续期任何窗口，也不把业务终态当清理证据 |
| [残留 Agent Run 回收](../issue-run/20261018-residual-agent-run-reclamation.md)：业务终态与时间流逝都不触发物理终止 | 继承，不变：60 分钟截止只产生取消请求，物理终止只经既有证据链 |
| [cloud/model-access 根决策](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) D3：创建运行的同一事务冻结连接版本、密钥引用、模型与发起者 Git 身份；撤销的绑定不可复活 | 继承，不变：每次接纳以创建者当时的个人配置接纳并冻结（D5）；被否定的配置不回退、不替换 |
| [个人模型发起者决策](../thread/20261010-personal-model-initiator-controls-input.md)：只有发起者可追加输入，发起者与当前租户管理员可结束 | 修改，仅限定时运行：定时运行整个生命周期 `canAppend = false`，任何 Thread 状态下追加均 `409 thread_closed`，前端禁用输入并显示只读提示；结束权仍归发起者与当前租户管理员 |
| [Thread 根决策 D4](../thread/0-durable-agent-thread-with-user-turns.md)：会话在用户结束或空闲期满（默认 15 分钟）时结束；成功 `TurnEnded` 且没有 `queued` 用户轮次是 `idle` 判据；其替代方案表否决「Agent 首轮结束即结束会话」 | 修改，仅限定时运行：判据不变，空闲窗口为 0——进入 `idle` 即 `ending` 并放出 `EndSession{idle_timeout}`；该否决针对交互式 Thread，无人值守定时运行没有人会回复，空闲等待没有收益 |
| [Workflow 会话结束决策](../workflow/20261019-workflow-session-end-after-initial-turn.md)：单回合会话由调度器读取成功 `turnEnded` 回执后放出 `EndSession{idle_timeout}` | 先例、同判据：它处理的 WorkflowRun 没有 Thread，因此「继承判据、不继承机制」且明确 IssueRun 与 Thread 路径不变；本文对 IssueRun 的修改走上方 Thread D4 行 |
| [取消写入者（G-026）](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md)：取消是请求加有界终态，首期必须有取消的写入者 | 继承并复用：定时运行的 60 分钟截止是该写入者的一个调用方（D4） |
| [tenancy 一 Space 一租户](../tenancy/20260923-one-space-per-tenant.md)：有效租户成员资格是授权边界；被禁用成员的新访问立即拒绝 | 继承，不变：任务读取与触发资格都按当前成员资格判定（D1、D5） |
| [project Space 归属](../project/0-project-space-scoping.md)：跨 Space 引用硬禁止，归属校验必须复用同一不变量 | 继承，不变：任务只存 `issue_id`，Project 一律由 issue 的 `project_ref` 派生（D1） |
| [Space API 根决策](../api-boundary/0-space-api-and-event-stream.md)：REST 是权威状态源，SSE 只承载提交后的失效提示；`404` 统一不存在与无权限 | 继承，不变：任务与历史的读写、事件按同一契约（D6、D7）；租户内找不到或无权限统一 `404`，非成员由所有租户请求共用的租户门禁 `403 membership_required` 拒绝（D1） |
| [Workflow 根决策](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md) D5：首期执行只支持匿名公开 Git | 继承为首期支持范围：端到端验收只覆盖匿名公开 Git 的目标 Project；资格表不单独校验仓库可见性（见[落地结果](#落地结果)） |

## 决策

### D1：任务定义归属创建者、绑定一个 issue，读取跟随 issue 可见性，管理仅限创建者

定时任务是租户内的持久定义，至少持久化：稳定 ID、`tenant_id`、绑定 `issue_id`、名称、随版本冻结的
Prompt、执行 Agent（`space_agents` 引用；插件版本不在此冻结，由既有 IssueRun 用例在接纳时按 Space
当时值快照，见 D3 与不变量 6）、创建者的模型选择、schedule
与创建时固定的 IANA 时区、无人值守同意（策略名与同意时间）、定义 revision、启用状态与停用原因、
软删除墓碑、创建/更新时间。执行归属固定为 `cloud`：Cloud 任务只能由 Cloud 调度，不存储也不暴露
可写字段。修改 schedule、时区、Prompt、Agent 或模型选择增加 revision；纯展示修改（如名称）不增加。

- **归属与费用**：任务属于创建者。每次触发产生的 IssueRun 以创建者为运行发起者：个人模型连接承担
  模型费用，Git 提交身份是创建者（沿用既有运行时身份规则）。任务不支持转移或共享管理。无人值守
  授权的持有人不在场时任务自动停用：创建者失去有效租户成员资格、或绑定 issue 被删除时，调度器在
  下一次评估时把任务置为停用（与撤销同意的停用语义一致），既有运行事实不改写；重新满足条件不会
  自动恢复触发，需创建者显式重新启用。
- **可见性**：任务定义与运行历史对「能读取该绑定 issue 的有效租户成员」可见，与 Thread 读取权限
  先例一致；租户内的 issue 对全部有效成员可读，因此任务对租户内全部有效成员可见。租户内找不到或
  无权限统一返回 `404 not_found`；非成员（含已停用成员）先被所有租户请求共用的租户门禁以 `403
  membership_required` 拒绝，与 Thread 读取一致。创建、编辑、启停、删除与撤销同意仅限任务创建者，
  其他成员一律得到 `404 not_found`：无人值守同意是个人授权，多人可改会造成授权来源不清晰。
- **绑定与派生**：任务只持久化 `issue_id`，绑定在创建时固定、不可修改——需要换目标时复制为新任务
  并停用旧任务（与执行归属的不可切换一致）；Project 由 issue 的 `project_ref` 派生，跨 Space 引用
  按 [Project 归属决策](../project/0-project-space-scoping.md) 硬禁止。绑定 issue 删除后任务保留但
  自动停用，不自动解绑或改绑。编辑接口不接受 `issueId` 与 `creatorUserId`（传输层以 `400
  unknown_field` 拒绝），领域层也不从请求体读取绑定；Prompt、Agent、模型选择、schedule 与时区可由
  创建者修改。

### D2：schedule 按任务时区求值，occurrence 以 UTC 落盘，迟到阈值固定

schedule 首版只有三种受限周期：每 N 小时（最短间隔 1 小时）、每天某时刻、每周某几天某时刻；不存储
或解释 cron 字符串。任务持久化创建时固定的 IANA 时区；occurrence 落盘后只以 UTC `scheduled_at`
标识。夏令时跳过的本地时间不补跑；回拨造成同一当地时间重复时，只接纳第一个对应的 UTC 时间点。
系统后续调整不影响已落盘的 occurrence。

- **迟到阈值**：调度器观察到某到期点时，若观察时刻晚于 `scheduled_at` 超过 10 分钟（固定阈值，不
  提供配置），该 occurrence 记为 `missed(late_trigger)`，不执行也不补跑。Cloud 侧 `missed` 原因只有
  `late_trigger`：调度器停机（部署、故障）表现为观察到迟到，不另设「服务未运行」原因。
- **长时间停机至多记一条 `missed`**：`missed` 是任务级观察，不是逐点事实。一次停机（无论跨过多少
  个到期点）恢复后，只为最近一个已过迟到窗口的到期点记录一条 `missed`，更早的到期点不补记；停机
  期间仍在迟到窗口内的最近到期点照常接纳。这条 `missed` 在 100 条保留里按一条普通历史记录计数，
  与运行、跳过事实同样参与最旧删除。
- **锚点**：每 N 小时任务以启用（或从禁用重新启用）时刻为锚，首个到期点为锚点加间隔，对齐 desktop
  的启用锚语义；daily / weekly 按任务时区的日历求值。**停用（或自动停用）与重新启用之间的到期点
  一律不评估、不补记 `missed`**：调度器只求值启用（重新启用）时刻之后的到期点，停用区间不视作
  停机。手动停用保留原锚点，重新启用由 `enable` 把锚点改为数据库当时时刻；自动停用同时清除锚点。
- **身份**：`scheduled_task_occurrences` 以 `(task_id, scheduled_at)` 唯一。调度器多副本、重启、
  部署或重复扫描都按同一身份幂等：先到的事务接纳，后到的读取既有结果原样返回，不产生第二个运行。

### D3：接纳在同一事务内完成资格校验、occurrence 写入与 agent IssueRun 创建

Cloud 运行一个持久调度 worker（数据库时间为准、无跨事务必需内部状态，复用既有周期扫描先例），
周期评估到期任务。某到期点到期且未超迟到阈值时，在一个事务内完成：

1. 读取任务当前定义并执行 D5 的资格校验；
2. 以 `(task_id, scheduled_at)` 幂等写入 occurrence；
3. 校验通过且未超限时，在同一事务创建该 issue 下的 agent IssueRun，occurrence 记录该运行 ID 与
   冻结的 task revision，之后由既有编排驱动 `provisioning → starting → running → delivering →
   releasing → done`。

- **运行入口唯一**：调度器只调用既有 IssueRun 应用用例，不直接调用模型、不拼接未版本化输入、不
  建立第二个 Agent 运行入口。运行记录其触发来源（任务与 occurrence）；调度器不写评论、不伪造用户
  交互。首条 prompt 沿用既有的首条 prompt 模板路径，其中任务文本是随 task revision 冻结的 Prompt。
- **每次触发新建会话**：不复用既有会话，不续接上一次运行的 Thread。
- **无头会话在首个成功回合后结束（修改 Thread D4，仅限定时运行）**：无人值守运行没有人回复追加
  轮次，若沿用交互式空闲窗口，每次运行都会空等空闲期满并可能与 60 分钟截止相撞。本文**修改
  [Thread 根决策 D4](../thread/0-durable-agent-thread-with-user-turns.md)，仅限定时运行**：Thread 仍按
  D4 原有判据进入 `idle`（成功 `TurnEnded` 且没有 `queued` 用户轮次），但定时运行的空闲窗口为
  0——进入 `idle` 即转入 `ending`，经既有 `thread_commands`（`run_id`）路径放出
  `EndSession{idle_timeout}`，会话终态后按 IssueRun D4 推导为 `completed`。这仍是 Thread D4 的状态机
  与 IssueRun 既有路径，只是把空闲窗口改为 0；判据不省略「没有 `queued` 用户轮次」。[Workflow 会话
  结束决策](../workflow/20261019-workflow-session-end-after-initial-turn.md) 是同判据的先例（它处理的
  WorkflowRun 没有 Thread，因此「继承判据、不继承机制」，且其兼容边界明确 IssueRun 与 Thread 路径
  不变），本文不搬用其机制。会话进入 `ending` 后的收敛只走 IssueRun 既有的结束确认窗口（最早一个
  `end_session` 的 `created_at` 加 `AgentEndConfirmTimeout`，不续期），60 分钟截止不再是这段时间的
  兜底（见 D4）。
- **定时运行整个生命周期不可追加**：修改 [个人模型发起者决策](../thread/20261010-personal-model-initiator-controls-input.md)，
  仅限定时运行——定时运行的 `canAppend = false`，任何 Thread 状态下追加一律 `409 thread_closed`，
  前端按 `canAppend` 禁用输入并显示只读提示。若只在会话结束后拒绝，首轮执行期间接受的追加轮次会被静默丢弃；若允许排队
  执行，运行会变成多回合并与 60 分钟截止相撞——两者都不符合「无头 Prompt 单会话」，因此整体禁用。
- **跳过与失败**：同一任务上一次接纳的运行尚未到达终态时，该到期点记 `skipped_busy`，不排队不补跑。
  首期不设跨任务全局并发上限：并发已受既有编排约束（每运行独立 Workspace、`one_project_operation`
  串行建删、沙盒容量），全局配额数值属于后续配额决策（见「未解决的问题」）。
- **失败的 occurrence**：资格校验失败的到期点记为失败的调度事实并写明原因（映射见 D5），不自动
  重试该 occurrence，也不回退到任何其他执行位置。

### D4：每次运行固定 60 分钟，截止取消由 Cloud 取消写入者执行并持久化截止事实

接纳事务为运行持久化截止时刻 = 接纳时刻 + 60 分钟，固定不可配置。到期时由 Cloud 侧的取消写入者
（[Workflow 根决策 D6](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md) 承诺首期交付的
同一写入者，G-026）对运行写入取消请求；会话收尾、交付与释放沿既有有界路径进行，实际终态在收尾后
写入，物理终止只遵循[残留回收](../issue-run/20261018-residual-agent-run-reclamation.md)的证据链。截止取消不是
清理证据，不绕过任何既有保护。

- **截止取消事实由调度层持久化**：`issue_runs.cancel_requested_at` 分不清用户取消与截止取消，历史
  投影不得从它反推。取消写入者对定时运行写取消请求的同一事务里，调度层为该 occurrence 记录一条
  截止取消事实（含写入时间）。事实只写一次：以 occurrence 身份幂等，重复的截止扫描、重启重放都
  不覆盖、不重复记录；用户先于截止取消的运行不产生该事实。
- **截止写入的边界**：截止扫描只在会话尚未进入 `ending | ended`、且 IssueRun 尚未到达终态时写取消
  请求与截止事实。会话已经结束的运行（例如正停留在 `delivering` 的交付重试）不再被截止取消，其
  交付按既有 D5 的上限收敛；截止扫描对这类运行是空操作，不补写事实。
- **投影规则**：任务历史展示为 `failed · timeout` 当且仅当**截止取消事实存在且 IssueRun 终态为
  `cancelled`**；其余情况一律按 IssueRun 自身终态展示（`completed → succeeded`、`failed → failed`、
  无截止事实的 `cancelled → cancelled`）。运行是否「先于截止自然结束」以 IssueRun 终态为准——会话
  结束只说明收尾开始，终态在收尾完成后写入；两层事实分别记录、互不改写。
- **前置门禁**：Cloud 取消写入者（G-026）随 cloud#100（cloud commit `f405823e`）合入 main，定时运行
  场景的截止取消已在端到端验收中直接验证。会话进入 `ending` 后截止扫描即空操作，这段时间的收敛
  只依赖 IssueRun 既有的结束确认窗口（见 D3）——60 分钟截止对收尾阶段不再兜底。非成功回合后会话
  仍存活时，由截止取消经既有取消反应放出 `EndSession{cancelled}` 收敛。

### D5：保存与每次接纳都校验资格，失败明确到原因，不回退不重试

保存指启用的创建与 `enable`；停用状态的草稿创建同样校验目标与模型配置，只是不取得同意。

| 校验 | 保存时 | 每次接纳 |
|---|---|---|
| 绑定 issue 存在、未删除且有 `project_ref` | ✓（issue 不存在为 `404`，无 Project 为 `409 issue_project_required`） | ✓（按当时状态） |
| Space 未归档 | ✓（写入时解析 Space 失败，按 Space API 根决策的统一 `404 not_found` 整体回滚） | ✓ |
| Agent（`space_agents`）active | ✓ | ✓（插件版本由既有用例按 Space 当时的 desired 版本快照） |
| 创建者是有效租户成员 | ✓ | ✓（失效即自动停用任务） |
| 创建者个人模型配置可用（连接启用、密钥已配置、模型在目录、能力满足） | ✓ | ✓（以当时配置接纳并在运行创建事务中冻结） |
| 无人值守同意有效且策略受支持 | ✓ | —（撤销同意即自动停用，接纳不再单独校验） |

- 保存失败以该项的错误码拒绝且不留部分写入：目标侧为 `target_unavailable`，模型侧为
  `model_unavailable`；`enable` 缺少有效同意为 `409 approval_policy_invalid`，启用的创建缺少或带了
  不支持的策略按畸形请求 `400 invalid_request` 拒绝。
- 接纳失败映射到根决策 D7 的失败原因集：issue / Project / Space / Agent 不可用记
  `target_unavailable`，模型配置不可用记 `model_unavailable`。失败 occurrence 不自动重试，不换用其他
  模型或执行位置。创建者成员资格失效或绑定 issue 删除时，评估直接自动停用任务（见 D1），不写
  occurrence。
- **同意只在保存时校验**：撤销同意在同一操作里清除策略与同意时间并停用任务，停用的任务不再接纳，
  所以接纳路径不会出现 `approval_policy_invalid`。重新启用必须携带新的显式同意。
- **无人值守同意**：首版唯一策略为 `full_auto_in_cloud_workspace`——在一次性隔离运行 Workspace 中
  全自动批准工具调用。创建任务时必须以清晰风险说明取得创建者显式同意，并把策略名与同意时间随任务
  持久化；说明至少覆盖：运行 Workspace 不是用户可见的数据边界承诺、Agent 全自动执行任意工具调用、
  消耗创建者个人模型额度、以创建者 Git 身份提交。仅当目标 issue / Project 或策略变更时要求重新
  同意；只改 Prompt 或计划不要求。创建者撤销同意后任务自动停用，既有运行事实不改写。
- 与 [model-access 根决策 D3](../model-access/0-personal-model-connections-and-run-scoped-proxy.md)
  一致：接纳时的配置校验与运行创建在同一事务，失败整体回滚；冻结后的绑定不随后续配置变化漂移，
  被撤销的绑定不可复活。

### D6：任务历史投影运行事实，每任务保留最近 100 条

任务历史记录 = occurrence（outcome、原因、接纳时间、冻结的 task revision、IssueRun 引用、D4 的截止
取消事实）加上对该 IssueRun 终态的投影：`completed → succeeded`、`cancelled → cancelled`、
`failed → failed`；截止取消按 D4 的持久事实展示为 `failed · timeout`，不从 `cancel_requested_at`
反推。调度层不复制 IssueRun 状态机：历史读取按需联查，运行中的运行展示为 `running`，终态以
IssueRun 的持久事实为准。

- **100 条保留**作用于任务历史视图的记录（occurrence 行）：写入第 101 条时在同一事务删除最旧的
  记录；若最旧的记录是尚未到达终态的活跃运行，则顺延到下一条非活跃记录。清理只移除任务侧的展示
  与关联行，不删除、不改写 IssueRun 及其 Thread / Revision 审计事实；被清理的运行在 issue 上仍完整
  可查。首期没有跨任务全局上限，历史不产生 `skipped_capacity`；该状态名保留用于两端展示对齐。
- 禁用任务只阻止未来接纳；删除任务写软删除墓碑并保留既有记录语义。
- 两端只对齐展示用的字段名与状态名，不共享 ID、存储或同步协议；本地任务与 Cloud 任务互不同步、
  不迁移，需要换执行位置时复制为新任务并停用旧任务。

### D7：通知走既有事实通道，不新建通知系统

首期不新增独立的通知或外发渠道。运行与触发的可见性由既有机制承担：IssueRun 的 Timeline 活动
（`run.completed / failed / cancelled`）继续落在绑定 issue 上；Space 事件流新增定时任务的失效事件
（任务列表与历史的刷新提示），按 Space API 根决策的事务后发布与只带失效信息的语义执行。接纳失败
与 `missed` 在任务历史中可见。邮件、推送等外发通知属于后续决策；通知是交互辅助，不是运行事实源。

### D8：首期管理面只在 Web，Desktop 保持禁用直到三项条件同时满足

首期任务的创建、编辑、启停、删除、撤销同意与历史查看只通过 Web 公开 API 与前端提供。Desktop 不
合并展示 Cloud 任务；Desktop 的 `cloud` 执行位置选项在以下三项全部满足前保持禁用，不得保存、不得
以本地计时器代跑、不得展示模拟成功：

1. 本 ADR 评审通过并置为 `approved`；
2. Cloud 取消写入者（G-026）已合入 main（cloud#100，`f405823e`），定时运行场景的截止取消已直接验证；
3. 无头 Prompt 单会话链路按 D3 的真实编排完成端到端验收。

三项条件已于 2026-10-26 的端到端验收后同时满足。Cloud 侧另有部署配置的验收开关：关闭时创建、编辑
与 `enable` 返回 `409 scheduled_task_acceptance_pending`，调度 worker 不评估任何任务，只有显式打开
的部署才开放定时任务。Desktop 端 `cloud` 选项的解除禁用、创建入口、任务展示与历史界面由后续决策
承接，复用同一公开 API 与同一授权边界；本 ADR 不预置其形状。

## 不变量

1. 任务的执行归属在创建时确定且只可为 `cloud`，之后不可切换；每个任务的未来触发始终只由 Cloud
   持久调度器产生。
2. 同一 `(task_id, scheduled_at)` 至多接纳一个 Scheduled Run；重复评估幂等返回既有结果。
3. 调度器只调用既有 IssueRun / Thread 应用用例；不新增第二套 Agent 运行入口，不接 Workflow、
   不解析 `executorSlot`，不解释 Prompt 之外的执行语义。
4. 每次触发新建会话，不复用既有会话或 Thread。
5. 观察晚于 `scheduled_at` 超过 10 分钟的到期点记 `missed`，不执行不补跑；失败、超时与 `missed`
   都不自动重试。
6. 已接纳运行冻结接纳时的 task revision、Prompt、Agent 插件版本与模型绑定；任务的后续编辑只影响
   未来触发。
7. 每次运行固定 60 分钟，截止不可配置；超时取消必须经 Cloud 取消写入者，写入者就绪前 `cloud` 不
   可成为可选执行位置。
8. 没有有效无人值守同意或可用模型配置时不接纳运行；撤销同意自动停用任务；接纳失败不回退、不替换
   模型或执行位置。
9. 创建者失去有效租户成员资格、或绑定 issue 被删除时，任务自动停用；任务不自动转移、不自动改绑，
   重新满足条件不自动恢复触发。
10. 禁用与删除不改写既有运行事实；100 条保留只作用于任务历史视图，IssueRun 及其审计事实不受影响。
11. 任务与运行历史只存于其租户内，不跨设备同步到本地。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 存储/解释 cron 表达式 | 表达能力强；但调度语义与校验面随之膨胀，违反根决策「不得存储或解释 cron 字符串」的冻结 |
| Web 前端计时器触发 | 实现快；但刷新、休眠与多副本会漏触发或重复触发，且违背「服务端拥有未来触发」的已冻结语义 |
| Desktop 本地计时器代跑 Cloud 任务 | Desktop 关机即漏跑，还会重复产生有费用的运行；被根决策 D5 明确禁止 |
| 每任务一个常驻会话，到期续发消息 | 省去建 Workspace；但会话状态跨触发漂移、失败面扩大，违反「每次触发新建会话」与一次性 Workspace 的隔离契约 |
| 触发时写一条 @ Agent 评论走既有评论路径 | 完全复用评论链路；但伪造用户评论污染时间线与权限语义，且 Prompt 冻结与评论编辑语义冲突 |
| 首期设跨任务全局并发上限 | 防止配额耗尽；但数值是配额决策，现有编排已约束并发面，先交由既有机制、配额另立决策 |
| 截止后直接终止会话并伪造终态 | 收尾快；但绕过取消请求与有界收尾，伪造运行事实，违反残留回收的不变量 |
| 邮件/推送通知随首期交付 | 及时性好；但通知渠道、地址与退订是独立产品面，先以既有 Timeline 与事件流承担可见性 |
| 允许把任务转移给其他成员或共享管理 | 协作友好；但费用归属、同意与 Git 身份都绑定创建者，转移等于重签授权，留待后续决策 |
| 首期在 Desktop 同步展示与管理 Cloud 任务 | 两端一致；但 Desktop 侧存储、同步与展示边界是独立决策面，先冻结 Web 单端以缩小首期范围 |

## 本决策未解决的问题

- 跨任务全局并发与配额的数值及策略（首期依赖既有编排的隐式约束）。
- 外发通知（邮件、推送）的渠道、地址与退订契约。
- 任务转移、协作管理与创建者离开后的承接。
- Desktop 解除禁用后的创建入口与展示边界（解除条件已在本 D8 冻结）。
- 私有仓库凭据链路就绪后，定时任务目标的扩展。
- `missed` 的可选补跑、合并与过期策略（继承根决策「未来若增加 catch-up 必须另行决定」）。
- 时区修改是否要求重新同意（首版按「仅目标或策略变更要求重新同意」处理，未单独计时区）。
- 定时任务相关的用量统计与账单展示口径。

## 落地顺序

1. **ADR 评审与批准**：按评审结论纠正本文后置为 `approved`，并核对
   `test-cases/cloud/scheduled-task/admission-and-session-bounds.md` 的义务与最终同意的决策一致。
2. **取消写入者（G-026）**：确认或交付 agent 运行侧的取消写入者及其直接验证（与 Workflow 落地共用
   同一收尾路径）；未就绪前不开放保存 `cloud` 任务。
3. **存储与调度 worker**：任务与 occurrence 的迁移、幂等接纳事务、周期扫描与 D5 资格校验；完成
   条件：多副本与重复扫描只产生一次接纳，迟到阈值、`skipped_busy` 与失败原因映射有直接验证。
4. **公开 API 与 Web 端**：任务的 CRUD、启停、撤销同意、历史读取与失效事件；完成条件：管理仅限
   创建者、读取跟随 issue 可见性、`404` 语义与幂等键规则与 Space API 根决策一致。
5. **端到端验收**：真实栈跑通「到期 → 接纳 → 运行 Workspace → Thread → Revision → 释放」，覆盖
   60 分钟截止取消、同意撤销停用、创建者成员失效失败与 100 条保留；验收通过并满足 D8 三条件后，
   才解除 `cloud` 选项的禁用。

## 落地结果

- 取消写入者（第 2 步）：[cloud#100](https://github.com/ora-space/cloud/pull/100)，cloud commit `f405823e`。
- 存储、调度 worker、公开 API 与 Web 管理面（第 3、4 步）：[cloud#111](https://github.com/ora-space/cloud/pull/111)。
  各决策的直接测试随 cloud#141、#145、#152、#153、#156、#159、#183、#188 合入；核心用例的领域
  证据以 cloud main `44daf7c` 为准。
- 端到端验收（第 5 步）：2026-10-26 在 cluster Compose 真实栈上完成，部署为 cluster `28bac76`、cloud
  `da2125f`、desktop `d913440` 的干净检出，真实 Controller / Node / Sandbox、官方 Claude 插件 0.4.1
  与真实模型，目标 Project 为匿名公开 Git。逐项结果登记在核心用例中。

与 approved 决策的偏离（正文已按实际行为改写）：

1. **同意的校验时点（D5）**：approved 版本要求每次接纳都校验同意、撤销后记 `approval_policy_invalid`。
   实现在撤销同意的同一操作里停用任务，接纳不再单独校验；`approval_policy_invalid` 只在 `enable`
   时作为 `409` 返回，启用的创建缺少或带了不支持的策略为 `400 invalid_request`。撤销后已不会再有
   接纳，「没有有效同意不接纳」的承诺不变。
2. **自动停用不记失败事实（D5）**：approved 版本要求自动停用前观察到的最后一次触发记失败事实。
   实现在评估时直接停用任务，不写 occurrence；停用原因记在任务的 `disabled_reason` 上。
3. **Space 归档时的保存（D5）**：资格表不单列 Space 归档。保存时写入要解析租户当前的 Space，已归档
   时按 Space API 根决策的统一 `404 not_found` 整体回滚；接纳时记 `target_unavailable`。
4. **读取的拒绝码（D1）**：approved 版本写的是「不存在与无权限统一 `404`」。实现中非成员先被租户
   门禁以 `403 membership_required` 拒绝，租户内找不到或无权限才是 `404`，与 Thread 读取一致。
5. **前端输入框（D3）**：approved 版本写的是「前端不显示输入框」。实现按 `canAppend` 禁用输入并显示
   只读提示；追加的权威拒绝仍是服务端的 `409 thread_closed`。
6. **仓库可见性（D5、不变量 11）**：approved 版本把「Project 的仓库满足首期仅匿名公开 Git」列为保存与
   接纳的校验项。实现没有单独校验仓库可见性，与 [Workflow 根决策 D5](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md)
   一样只作为首期支持范围；端到端验收只覆盖匿名公开 Git。
7. **插件版本可解析（D5）**：approved 版本要求 Agent「当时 desired 版本可解析」。实现只校验 Agent
   为 active，插件版本由既有 IssueRun 用例按 Space 当时的 desired 版本快照，资格表不单独校验它能否
   解析。
8. **执行归属与下一次触发投影（D1、D2）**：执行归属固定为 `cloud`，没有持久化列，也没有可写字段；
   任务不持久化、也不展示「下一次触发」投影。
