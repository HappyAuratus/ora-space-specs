# Node 恢复截止、取消与停止确认

依据[已批准的 Node 恢复截止与停止确认决策](../../../decisions/node/revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)；其中 D6 的停止确认证据另由[恢复停止确认接受 host 权威 NotStarted 决策](../../../decisions/node/revision/20261021-stop-confirmation-accepts-not-started.md)扩展（下称“扩展决策”）。下文的 D 编号与不变量编号未加限定时指前一篇 ADR，引用扩展决策时写明其 D 编号或“扩展决策不变量”；“现行恢复决策”指[Node 恢复前序 Revision 决策](../../../decisions/node/revision/20261010-restore-prior-revision-before-session.md)，“内部线决策”指[Node 交付推送与 Git 恢复决策](../../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)，“启动决策”指[有界启动与未派发撤回决策](../../../decisions/cloud/issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)，“残留回收决策”指[残留资源回收决策](../../../decisions/cloud/issue-run/20261018-residual-agent-run-reclamation.md)。

本文覆盖完整恢复截止与预算裁剪、停止确认的两种证据（直接进程退出与 host 权威 `NotStarted`）、取消点、停止证据未知与上限、窗口衔接和启动清理交接。恢复已按内部线决策改为从 Git Server 拉取前序交付 ref，不再有 bundle 恢复；它的拉取、检出与失败分类证据登记在 [Node 经 Git Server 交付与恢复前序 Revision](git-delivery-push-and-restore.md)。那些证据不能替代本文新增的验证义务。host 权威 `NotStarted` 扩展的直接证据登记在下方同名用例，其余截止、取消与 Cloud/Node 联合故障义务保持各自的实际证据状态；扩展已实现不表示整份截止决策已经完成部署级验收。

## A Restore Deadline Covers Every Step And Clips Command Budgets To Its Remainder

### 风险

某个子步骤重置截止，或一条命令按自身预算越过截止继续运行，恢复步骤在 Cloud 窗口之后才结束，失败被折叠成超时。

### 前置状态

恢复输入已校验；截止可配置，默认 5 分钟；一条网络命令的自身预算是 10–20 分钟；恢复在各步骤间重试与退避。

### 触发

在截止前一刻开始慢拉取；重试与退避跨越截止；检出与祖先检查在截止内完成；截止到期。

### 必须成立

- 一个截止从恢复第一步计到检出与祖先检查完成，任何子步骤、重试、退避或新连接都不重置它。
- 每条受管 Git 命令的预算等于 min(自身预算, 开始时的剩余截止)；截止到期后不再开始新命令，等待类步骤立即中止，运行中的命令被请求关闭。
- 截止约束的是恢复步骤的执行与等待预算，不是会话返回时长：停止确认、有界清理、迟到补报和上限内的等待在截止之后另行进行，不被该截止压缩，也不得为了满足截止而提前终结或跳过停止证据。
- 取得停止确认后，会话以 `agent_failed` 加 `prior_revision_*` 结束；停止确认与 D6 的证据定义一致。
- 默认 5 分钟与 Cloud 6 分钟恢复窗口的关系只作为常态目标，不构成“Node 必然先报告”的保证。

### 禁止结果

- 重试、退避或子步骤把截止推后；预算未被裁剪时命令用满自身 10–20 分钟预算越过截止。
- 截止到期后仍开始新的 Git 命令；未取得停止确认就发终态。
- 以“deadline 已过”推断 Git 已停止写入。
- 把“恢复或会话在截止前返回”当作验收判据，为此提前发终态、跳过停止确认或推迟已发出的关闭请求。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 截止贯穿全程、子步骤不重置 | 注入跨步骤的重试与退避，使各步骤单独都在截止内、合计超过截止：断言恢复在截止处（以恢复第一步为唯一起点）不再开启新命令、等待类步骤立即中止，且截止不因第 N 次重试、退避或新连接顺延；非法配置（零、负、无限）拒绝启动恢复 | Missing |
| 命令预算裁剪到剩余截止 | 在截止前一刻注入慢拉取，断言命令在截止处被关闭请求而不是用满自身预算；等待类步骤立即中止 | Missing |
| 到期终态与停止确认顺序 | 截止到期且取得停止确认后才发 `agent_failed`/`prior_revision_*`；未取得时按 `An Unconfirmed Stop Never Yields A Terminal State Or A Deliverable Checkout` 处理 | Missing |
| 停止确认与终态时点不并入截止 | 注入截止处发出关闭请求、原 Scope `Closed` 与直接进程退出晚于截止返回（例如 5 分 2 秒）：断言不再开启新命令、关闭请求已发出，取得停止确认后才清理并报终态，且该路径不按“超过截止”判失败；再注入上限内始终取不到原停止证据，断言会话保持未终结并转交残留回收。停止确认、清理与上限到期分别按 `Stop Confirmation Requires Scope Closure And Direct Process Exit`、`The Unconfirmed-Stop Bound Starts At The First Stop Request And Never Resets` 验证 | Missing |

### 决策依据

D1、D6；不变量 1、2、4、11。

## Stop Confirmation Requires Scope Closure And Direct Process Exit

### 风险

把执行预算到期、关闭请求被接受或清理预算耗尽当作“Git 已停止”，在进程仍可能写入时发终态或开始交付。

### 前置状态

恢复中有一条受管 Git 命令在运行；可注入 guardian 正常、关闭请求失败、guardian 不可达与直接进程状态未知。

### 触发

截止到期向受管执行请求关闭；host 接受关闭但 guardian 尚未观测到 `Closed`；清理预算耗尽；直接进程已退出但 Scope 未观测到 `Closed`。

### 必须成立

- 基础证据是“受管执行观测到原 Scope 已 `Closed` 且该 Run 的直接进程已退出”；扩展决策在此之外接受 host 权威的 `NotStarted`。两者的共同前提都是原 Scope 已 `Closed`，`UnknownIdentity` 只说明查不到记录，不成立停止确认；`NotStarted` 的来源、fencing 前提与矛盾检测按 `Stop Confirmation Also Accepts A Host-Authoritative NotStarted` 验证。
- 预算到期只表示恢复不再等这条命令，关闭请求只表示关闭已被提出；两者都不是停止确认，也都不能推导 `NotStarted`。
- 停止确认按实际纳管保证报告：Strong 在声明范围内给出完整退出证据；当前 Linux BestEffort 的尽力清理完成不能排除未跟踪后代，不得写成已确认静止。

### 禁止结果

- 用关闭请求被接受、清理预算耗尽、`process_attempts` 记录缺失或 Node 重启推断停止，或用它们推断 host 将报告 `NotStarted`。
- 把 `UnknownIdentity` 当作停止确认，或把“查不到记录”解释为“从未启动”。
- 在未取得停止确认时发出作为停止确认的终态，或用清理结果替代停止确认。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 三种事实分开 | 注入“关闭请求成功但未观测到 `Closed`”“直接进程退出但 Scope 未 `Closed`”“清理预算耗尽”，断言前两者不产生停止确认、第三者不产生终态，且三者都不能被当成 `NotStarted` 的依据 | Missing |
| 两侧结论 | 同一「原 Scope `Closed`」前置下分别注入 `NotStarted` 与 `UnknownIdentity`：断言前者构成停止确认并进入正常路径，后者不构成停止确认、不进入正常路径；`UnknownIdentity` 的具体禁令按 `An Unconfirmed Stop Never Yields A Terminal State Or A Deliverable Checkout` 与 `Stop Confirmation Also Accepts A Host-Authoritative NotStarted` 验证 | Missing |
| 证据分级 | BestEffort 下尽力清理完成不报告“已确认静止”；Strong 启用时只在声明范围内报告；两者的结果可区分 | Missing |

### 决策依据

D6；不变量 3；扩展见[扩展决策](../../../decisions/node/revision/20261021-stop-confirmation-accepts-not-started.md)。

## Stop Confirmation Also Accepts A Host-Authoritative NotStarted

### 风险

把“查不到记录”或“还没看到进程”当作“没有进程在跑”，或者把平台/guardian 合成的 `NotStarted` 用在没有 fencing 保证的路径上，在仍可能有 `Start` 到达或进程仍可能写入时发终态、开始交付。

### 前置状态

恢复命令的 attempt 已持久化，`Start` 未送达 host 或已被取消；host 可能返回 `DirectProcessState::NotStarted`（平台 spawn 明确未启动，或 worker 对 `Stop` 的 `UnknownRun` 合成 `cancelled()`）或 `HostRejection::UnknownIdentity`；可注入已确认启动的 Run（用于矛盾检测）、guardian 在接受到启动请求前后的两种时机，以及 host 记录不完整（journal 缺失、`process_attempts` 被清理）。

### 触发

原 Scope 已 `Closed` 后查询该 Run：host 报告 `NotStarted`；或报告 `UnknownIdentity`；或对一个已确认启动的 Run 报告 `NotStarted`。

### 必须成立

- 原 Scope 已 `Closed` 且 host 权威报告该 Run 为 `NotStarted`（`direct` 为 `DirectProcessState::NotStarted`、`launch` 为 `LaunchFact::NotStarted`、cleanup 为 `CleanupState::Complete`）构成停止确认，走与直接进程退出相同的正常路径：在有界清理预算内清理临时目录与传输仓库（失败只记录），报告 `prior_revision_unavailable`（取消先生效时按取消原因），会话以 `agent_failed` 结束；不进入 `stop_unconfirmed`。
- `NotStarted` 的来源限于平台 spawn 明确报告未启动与 guardian 在接受到启动请求前取消两类；后者由 worker 在发 `Stop` 收到 `GuardianRunRejection::UnknownRun` 后合成，可信前提是 fencing：当前 host binding 挡住旧请求，且 worker 发出 `Stop` 之后不再排 `Start`。
- `UnknownIdentity` 仍不是停止确认：不发终态、不交付、不读 HEAD、不重开或复制执行、不解除 checkout 冲突责任，停止证据上限到期后在原执行上登记 `stop_unconfirmed`，迟到证据仍按原执行身份补报一次。
- 对已确认启动的 Run 报 `NotStarted` 是矛盾，只拒绝该快照、保留原事实并公开协调错误；同批其他 Run、Scope 观测和其他 Scope 的协调继续，不能当作停止确认。
- cleanup 证据按实际取得的结果分级报告：平台 spawn 路径取 `cleanup_evidence(guarantee)`，guardian 取消路径固定 `CleanupEvidence::BestEffortComplete`；BestEffort 的尽力清理完成不等于已确认静止。

### 禁止结果

- 由预算到期、关闭请求、清理预算耗尽或等待上限推导 `NotStarted`。
- 接受 `UnknownIdentity` 作为停止确认，或把“查不到记录”解释为“从未启动”。
- 在没有 fencing 前提（旧请求可能到达，或 `Stop` 之后仍可能排入 `Start`）的路径上合成 `NotStarted`。
- 把已确认启动的 Run 的 `NotStarted` 静默当成停止确认。
- 把 `BestEffortComplete` 报告成已确认静止。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| `NotStarted` 走正常路径 | [`not_started_requires_closed_scope_consistent_facts_and_original_identity`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/apps/ora-node/src/managed/not_started_tests.rs) 经真实 Unix IPC、SQLite 与生产停止确认/私有输入清理入口，断言一致 NotStarted 被接受、清理前输入保留、确认后传输目录清除、原 attempt 解除且 `agent_failed` / `prior_revision_unavailable` 终态只提交一次；不一致、cleanup 未完成及身份错配均不能终结，确认期间不执行 Git | Covered |
| 两种来源各自成立 | [`platform_nonlaunch_preserves_cleanup_evidence_for_each_guarantee`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/tests/launch.rs) 从生产 ScopeRuntime 的平台 spawn 未启动路径取得快照与 Closed；[`nonlaunch_requires_bound_stop_and_never_sends_start_after_stop`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/src/host/worker_tests.rs) 经生产 worker 与真实 Unix IPC 从已绑定的 Stop/UnknownRun 取得一致 NotStarted。两类产出的事实由上述 Node 判定回归验证；这是分层行为证据，不是 Strong 平台部署验收 | Covered |
| fencing 前提 | [`nonlaunch_requires_bound_stop_and_never_sends_start_after_stop`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/src/host/worker_tests.rs) 断言管理 binding 先于 Close/Stop、Stop 后无 Start；旧 binding、旧 session 和 Start/UnknownRun 均无 NotStarted；[`queued_request_checks_binding_after_takeover`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/src/guardian/management_tests.rs) 在实际执行互斥门禁排队后接管，旧 session 被拒。未尝试 guardian 的关闭路径由 [`restart_before_first_dispatch_distinguishes_recorded_and_unknown_runs`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/src/host/non_launch_tests.rs) 验证原 Scope 封闭、拒绝后续 Start 且未登记 Run 仍未知 | Covered |
| 矛盾检测与错误隔离 | [`confirmed_launch_rejects_not_started_before_and_after_host_recovery`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/src/host_state/observations.rs) 验证 Running/Exited 与后续 Unknown 不能抹掉已启动事实；[`contradictory_nonlaunch_preserves_facts_and_keeps_tick_serving`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/process-runtime/src/host/non_launch_tests.rs) 经生产 tick 与真实 SQLite，在重启前后断言 tick 成功、原观测不变、同批其他 Run 与另一 Scope 的观测/进度继续提交、查询暴露 Rejected(ConflictingRun)，不制造 settled 证据；Node 的一致性回归拒绝矛盾快照、无终态 | Covered |
| `UnknownIdentity` 保守 | Node 一致性回归的 UnknownIdentity 分支验证无终态、输入与原身份保留；[`unknown_stop_matrix_expires_without_terminal_then_original_proof_settles_once`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/apps/ora-node/src/managed/restore_tests.rs) 通过真实 IPC/SQLite 验证到期登记 stop_unconfirmed、停止轮询且原身份迟到终态只提交一次；[`unknown_stop_retains_clock_terminal_and_checkout_responsibility`](https://github.com/ora-space/desktop/blob/8caecc59f857bb9a14568abbec06e75674645365/crates/node-db/src/tests/session/restore.rs) 验证重启后原时钟与身份保留、拒绝新会话复用 checkout。尚缺把 UnknownIdentity 注入完整交付入口并直接断言无 HEAD/快照/推送的组合证据；该联合隔离验证随截止决策独立承接，不将较低层状态拒绝写成全链路 Covered | Partial |
| cleanup 证据分级 | 平台 nonlaunch 回归逐一断言 BestEffort/Strong 的实际快照与 Closed 证据；worker 的 Stop/UnknownRun 回归及 host 重启回归固定断言 BestEffortComplete，不升级为 ConfirmedQuiescence。这证明分级规则，不声明 Strong 部署已经启用 | Covered |

### 决策依据

扩展决策 D1、D2；扩展决策不变量 3、4、5。

## An Unconfirmed Stop Never Yields A Terminal State Or A Deliverable Checkout

### 风险

停止证据未知时发出终态，同时虚构“已停止”和“checkout 可交付”；或交付在仍可能变化的 checkout 上读 HEAD、快照，把半恢复内容做成成果。

### 前置状态

恢复的检出或取入正在进行；guardian 不可达、关闭失败、直接进程状态未知或 Node 在恢复期间崩溃；该会话带前序输入，Agent 从未启动。

### 触发

在上述状态下进入 D6 的未知路径；Cloud 结束确认窗口耗尽并已业务收尾；停止证据上限到期。

### 必须成立

- 会话保持未终结，不发送 `AgentSessionEnded` 或任何伪装成停止的终态；Cloud 的 D4 业务收尾是业务结算，不是停止证明。
- 不把 `process_attempts` 标记 cleaned，不删除原 Scope/Run，不放弃原 host/guardian 身份，不为同一会话新建第二条恢复或第二个执行身份。
- 交付被禁止：既不探查工作区改动，也不读 HEAD、不快照、不推新内容；checkout 不被新会话、新恢复或交付复用，即使 Cloud 已把 Run 记为 `done`。
- 临时目录、传输仓库等本地清理失败只记录，并与进程未知分别报告，不合并成同一个结论。

### 禁止结果

- 终态或任何“已停止”记录；交付返回 `unchanged`/`delivered`；checkout 被复用；删除原责任记录。
- 用“Cloud 已 `done`”“清理预算耗尽”“Node 已重启”解除交付或复用禁令。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未知路径不发终态、不交付 | 四种注入（guardian 不可达、关闭失败、未返回 `Closed`、直接进程状态未知）下无终态事件、无交付读 HEAD/快照/推送、原 Run 记录保留、checkout 复用被拒 | Missing |
| 清理与进程未知分开 | 临时目录删除失败只产生清理结果，不改变停止证据或终态判断 | Missing |
| Cloud 业务收尾不解除保护 | Cloud 已 `failed/done` 后，同一 checkout 的交付与复用仍被拒绝 | Missing |

### 决策依据

D4、D6；不变量 7、8。

## The Unconfirmed-Stop Bound Starts At The First Stop Request And Never Resets

### 风险

上限在重启、重试或对账时被重置，原 Scope/Run、checkout 冲突责任与 Workspace 再次被无限期保留；或到期被当成停止证据，提前交付或强停。

### 前置状态

首次停止请求已持久化；配置的恢复截止与上限；可注入 Node 重启、Controller 重连、Cloud 窗口到期与重复对账。

### 触发

首次停止请求后仍取不到停止确认；时钟越过上限；Node 在到期前或到期后重启；Cloud 的 2/6/5 窗口分别到期；清理重试与对账重复执行。

### 必须成立

- 上限从首次停止请求起算（不等同于“发现证据未知”的时间），默认 10 分钟，可配置且为正的有限时长；配置校验要求大于恢复截止。
- 不因恢复截止、Cloud 窗口、重试、清理、退避、Node 重启、Controller 重连或对账重置；起点缺失时只取第一次观测值并持久化，之后沿用。
- 到期后停止本会话的等待与轮询，不发终态、不标记 cleaned、不删除原记录；在原执行上持久登记命名原因 `stop_unconfirmed` 与首次停止请求、到期时间及实际纳管保证。
- 该命名原因通过已有执行查询可见；Cloud 的残留账本条目在结束确认窗口耗尽、仍无真实终态时，以同一命名原因登记命名证据（`reason`/`responsibility` 不改写）；Node 的本地记录与账本条目按同一 Run/Workspace 身份对应。
- 交付与复用禁令不因到期解除；之后取得原停止证据仍按原执行身份补报一次，作为迟到事实接管。
- 资源回收按残留回收决策的证据分层：到期不是证据，不授权自动强停或数据删除。

### 禁止结果

- 上限按重启、重试、窗口或对账重新计时；或把上限并入 Cloud 的结束确认窗口。
- 到期即发终态、解除交付/复用禁令、删除或改写原记录、自动强停。
- 重复补报终态、重开会话、重新结算或复活资源。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 起点持久化与不重置 | 首次停止请求先落盘再发关闭请求；重启、重连、窗口到期与重复对账后上限不变；起点缺失时只锚定一次 | Missing |
| 到期登记与可观测 | 到期后无终态、原记录保留，执行查询返回 `stop_unconfirmed` 与两个时间；Cloud 在 D4 结算时把 `stop_unconfirmed` 写成命名证据且 `reason`/`responsibility` 不变；两者在同一个 Run/Workspace 身份上可对应 | Missing |
| 安全边界与迟到补报 | 到期后交付/复用仍拒绝；随后取得原停止证据时按原执行身份补报恰好一次；残留回收仍按证据分层，未因到期强停 | Missing |
| 配置校验 | 非法值（零、负、无限、小于等于恢复截止）拒绝启动恢复 | Missing |

### 决策依据

D6；不变量 8、11；残留回收决策 D1、D2、D5。

## Cancellation Stops At Git Command Boundaries And Keeps The Checkout Consistent

### 风险

取消时打断运行中的 Git 留下锁或半写状态；或取消后仍写 checkout、在没有停止确认时结束会话；崩溃后交付探查混合工作区。

### 前置状态

恢复进行到取入、检出或下载；取消来源为用户结束、Cloud 策略结束、Node 优雅停止；另有崩溃重启。

### 触发

命令边界收到取消；命令运行中收到取消；崩溃重启后继续交付。

### 必须成立

- 取消在命令边界或无副作用的等待中生效；已开始的命令等它结束或按裁剪预算被关闭，之后不再开始新步骤、不再写 checkout。
- 取消点的 checkout 停在 clone 基线或前序最终提交；检出中途失败、被截止终止或崩溃可能留下混合工作区，但 HEAD 只可能是这两个提交之一。
- 终态原因按先到者（用户结束/取消/`interrupted`），并且只在取得停止确认后发出；取不到时走未知路径，不在崩溃重启时发 `interrupted`。
- 带前序输入且 Agent 从未启动的已确认停止会话，交付不探查工作区、最终提交取 HEAD、结果恒为 `unchanged`，不推新内容。

### 禁止结果

- 取消时打断 Git 进程；取消后继续写 checkout；没有停止确认就发终态。
- 在停止确认未知时交付或返回 `unchanged`；把混合工作区快照成新提交。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 取消点矩阵 | 取入前、下载中、检出后分别取消：不开始新命令、不再写 checkout、终态在停止确认后发出 | Missing |
| 检出故障的交付隔离 | 注入文件锁使 `checkout --force -B` 非原子失败、检出中被截止终止、崩溃重启，三种情况下交付均不探查工作区、最终提交取 HEAD、结果 `unchanged`、无推送 | Missing |
| 崩溃重启不虚构 `interrupted` | 前进程未取得停止确认时，重启后不发 `interrupted`，按未知路径保留责任 | Missing |

### 决策依据

D3、D4；不变量 5、6、7、10。

## Cloud And Node Windows Settle By First Arrival Without Resetting Each Other

### 风险

两侧重复计窗、互相续期，或把 Cloud 的业务收尾、结束请求送达、确认窗口耗尽当作 Node 已停止；正确性依赖“Node 必然先报告”。

### 前置状态

Cloud 的 2/6/5 分钟窗口与 Node 的恢复截止、停止证据上限独立计时；可注入各窗口的到期顺序。

### 触发

按 D2 矩阵逐行触发：Node 在恢复窗口内失败、Cloud 恢复窗口先耗尽、结束请求送达且确认在窗口内、确认晚于窗口、停止证据未知、上限到期、结束请求不可达、恢复成功。

### 必须成立

- 每一行按“谁先到按谁结算”执行；Cloud 窗口耗尽不取消 Node 截止，Node 截止不为 Cloud 窗口续期；停止证据上限同样不重置任何窗口。
- 矩阵任何一行都不把结束请求送达、预算到期、确认窗口耗尽或上限到期当作 Node 已停止；Cloud 的业务收尾与其物理停止证明分开。
- Node 的迟到终态按原执行身份接管，不重开会话、不再次结算。

### 禁止结果

- 重复计窗或互相续期；以时序假设“Node 先报告”作为正确性前提。
- 把 Cloud 的 `failed/done` 当作停止证据；在 Node 未知路径下把结束请求视为已停止。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 矩阵逐行 | 联合故障测试覆盖 D2 矩阵每一行，含“报告晚于恢复窗口”“确认晚于确认窗口”“上限到期”“结束请求不可达” | Missing |
| 业务收尾与停止证明分离 | Cloud 结算后 Node 未知路径仍不发终态、不交付；迟到终态只补事实 | Missing |
| 上限不重置窗口 | 上限到期前后 Cloud 窗口的起点、时长与结算不变 | Missing |

### 决策依据

D2、D6；不变量 9、11。

## Startup Cleanup Follows The Original Execution Handover And Keeps Evidence Tiers

### 风险

Node 重启时清掉仍可能被原 Git 进程写入的临时目录或传输仓库；或把 BestEffort 的尽力清理完成当成已确认静止，掩盖未跟踪后代。

### 前置状态

Node 重启后存在 `.ora-transport/` 或 `revision-restores/` 残留；其中一条原执行已取得停止确认，另一条仍未知；实际纳管保证为 Linux BestEffort（Strong 未启用）。

### 触发

Node 启动清理私有目录；原执行交接查询返回 `Closed` 或未知；Strong 能力启用时重复同一场景。

### 必须成立

- 启动清理以资源交接决策 D3 为前提：只清理原执行已取得停止确认的目录；未确认的保留并登记命名残留，不用 Node 重启、DB 锁或目录空闲推断停止。
- 清理结果按实际纳管保证报告：BestEffort 的尽力清理完成不写成已确认静止；Strong 只有在声明范围内取得完整退出证据时才报告。
- 临时目录删除失败只记录并由启动清理兜底，不影响停止证据；进程未知也不因目录已删而缓解。

### 禁止结果

- 删除原执行未确认停止的目录；以 Node 重启推断原 Git 已停止；把 BestEffort 结果当作 Strong 证据或停止确认。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 启动清理经原执行交接 | 重启注入：已确认停止的目录被清理，未知原执行的目录保留且登记命名原因 | Missing |
| 证据分级 | BestEffort 与 Strong 两条路径的清理结果与报告可区分，BestEffort 不报告已确认静止 | Missing |
| 清理失败与进程未知分开 | 删除失败只产生清理结果；进程未知不因清理成功而解除 | Missing |

### 决策依据

D1、D4、D6；[资源交接决策](../../../decisions/node/process/integration/0-shared-process-entry-and-resource-handoff.md) D3；[纳管保证根决策](../../../decisions/node/process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D2。
