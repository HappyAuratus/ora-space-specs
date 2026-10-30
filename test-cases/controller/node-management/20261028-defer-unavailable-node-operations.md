# 不可用 Node 延后与 clone 分轮等待

依据[不可用 Node 延后决策](../../../decisions/controller/node-management/20261028-defer-unavailable-node-operations.md)
（`implemented`，下称“ADR”）。本文件镜像该 ADR 的领域和文件名，同时登记 Controller 轮次、Cloud 持久计数、
有限原因、结果接管与跨进程验证义务。Cloud 实现已落在
[`1cfaed3`](https://github.com/ora-space/cloud/commit/1cfaed320d653f0f2689e165f98cf3c705f3dcfa)；
Controller 实现与直接测试以 [desktop#750](https://github.com/ora-space/desktop/pull/750) 的
[`9078570`](https://github.com/ora-space/desktop/commit/9078570ad26b644bd5e91d80d310815ebd1e1f0c) 为审阅基线。
本规格登记 PR 必须等该 Desktop PR 先合入，随后补充其 merge commit；PR head 不表示 Desktop main 已落地。
ADR 的实现状态与验证覆盖分别登记，仍未覆盖的完整分支保留 `Partial` 或 `Missing`。
以下分别列出直接测试覆盖与缺口，不能用 PR 全绿或历史 #698 数据代替完整义务。

最终 PR head 为 [`65fc457`](https://github.com/ora-space/desktop/commit/65fc45714bdcdbcc71f2eb9263b963ab54c6a26a)。本轮多失联项复用由 `9078570` 构建的镜像并保留 revision 标签；两提交仅 `#[cfg(test)]` 夹具变化，生产运行源码与版本输入未变。前期部署与测试报告仍按其实际 `9078570` 基线保留，不声称镜像由新 head 重建。

证据入口（`文件:测试名` 中的文件缩写均指以下固定版本）：

- **C-db**：[Cloud `internal/core/operation_wait_db_test.go`](https://github.com/ora-space/cloud/blob/1cfaed320d653f0f2689e165f98cf3c705f3dcfa/internal/core/operation_wait_db_test.go)。
- **C-review**：[Cloud `internal/core/operation_wait_review_test.go`](https://github.com/ora-space/cloud/blob/1cfaed320d653f0f2689e165f98cf3c705f3dcfa/internal/core/operation_wait_review_test.go)。
- **C-migration**：[Cloud `internal/core/operation_wait_migration_test.go`](https://github.com/ora-space/cloud/blob/1cfaed320d653f0f2689e165f98cf3c705f3dcfa/internal/core/operation_wait_migration_test.go)。
- **C-codec**：[Cloud `internal/controlgrpc/operation_wait_codec_test.go`](https://github.com/ora-space/cloud/blob/1cfaed320d653f0f2689e165f98cf3c705f3dcfa/internal/controlgrpc/operation_wait_codec_test.go)。
- **D-wait**：[Desktop `apps/ora-controller/src/cloud/operations/wait.rs`](https://github.com/ora-space/desktop/blob/9078570ad26b644bd5e91d80d310815ebd1e1f0c/apps/ora-controller/src/cloud/operations/wait.rs)。
- **D-joint**：[Desktop `apps/ora-controller/tests/workspaces/short_rounds_joint.rs:production_worker_yields_and_delivers_original_late_results_to_persistent_go_cloud`](https://github.com/ora-space/desktop/blob/9078570ad26b644bd5e91d80d310815ebd1e1f0c/apps/ora-controller/tests/workspaces/short_rounds_joint.rs)，由 [Cloud `internal/controlgrpc/short_rounds_joint_test.go:TestShortRoundsJointServer`](https://github.com/ora-space/cloud/blob/1cfaed320d653f0f2689e165f98cf3c705f3dcfa/internal/controlgrpc/short_rounds_joint_test.go) 提供真实 gRPC/Store/PostgreSQL 边界。

D-joint 使用真实 Rust 普通 worker 和协议 Node 替身；夹具预置步骤，并用 `/expire` 将原 execution 的
`created_at` 调老至 1801 秒。它证明受控跨进程状态和原结果交付，不是完整 Project 创建、真实 Git、
不修改 operation 行的部署复现或真实 30 分钟等待。PR750 报告 48 个 lib 测试及 6 个联合场景通过；
各项状态仍由直接断言的范围决定。部署记录及最终运行附件位置见文末，未提供的分支保留缺口。

## Unavailable Nodes Yield The Ordinary Worker Within The Handshake Grace

### 风险

失联 Node 每轮占住全局普通 worker 60 秒，阻止其他租户推进；重连反复刷新宽限，导致本轮永远不结束。

### 前置状态

Controller 持有有效 lease；A 的 Node 不可用，B 有健康可推进的 operation；领取顺序为当前 `created_at,id`。

### 触发

A 执行 node 就绪检查、clone 等待或 quiesce 的 Node 就绪检查；Node 缺失、持续不可用、在宽限内就绪，或反复连接却不能推进。

### 必须成立

目标已就绪时立即推进，没有可连接目标时立即延后；否则同一轮所有就绪检查共用一次最多 2 秒宽限。
宽限不因重连或重复检查续期。A 以 `retry_wait/node_unavailable` 提交未来的 `retry_at` 后结束本轮，
被排除出可领取集合，B 可以领取；会话重连和全局 lease 继续，quiesce 的停止派发与 idle 判定不变。

### 禁止结果

轮内等待 60 秒、等 retry_at 到期才让出 worker、重复获取宽限，或把 Node 不可用写成失败、idle 或终止事实。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 零等待与有限宽限 | 受控时钟分别证明已就绪立即推进、无目标立即延后、宽限内就绪正常推进、不可用最多等 2 秒且连接抖动不续期 | Partial | `D-wait:handshake_grace_is_shared_and_cannot_renew` 断言同轮 2 秒截止不续期；D-joint `unavailable` 断言生产 worker 不足 4 秒延后。已就绪零等待、无目标零等待、宽限内握手及真实连接抖动尚无完整直接断言 |
| 当前排序下让出处理位置 | Rust Controller 与持久 Cloud 联合路径中，在 A 本轮创建 B；断言未来 retry_at 排除 A、B 被领取，且没有 60 秒等待 | Covered | D-joint `unavailable` 在 A running 后创建 B，断言 A 未来 retry_at、B 成功且 A 无 60 秒等待；`C-db:TestOperationWaitPendingYieldsAndKeepsExecution` 直接断言 B 被领取。B 是预置 stop operation，部署 Project 延迟另见后文 |
| quiesce 只缩短就绪等待 | 验证同一 2 秒宽限，既有停止派发、pending/idle 与 Cloud 接受条件保持原样 | Partial | D-joint `quiesce` 断言有限等待、无 idle/terminated 伪证据且 B 推进；正常停止派发、pending→idle 与完整 Cloud 接受条件未在该联合路径全部覆盖 |

### 决策依据

ADR D1、D2，不变量 1、2、6、7；specs#95 D3 合并后原样叠加，不能作为当前排序测试的前置条件。

## Cloud Persists Backoff Counts And Bounded Reasons

### 风险

重启或手动 retry 重置预算；次数与 retry_at 分开提交；Node 不可用轮次误耗 clone 三次预算，或新原因在内部契约中丢失。

### 前置状态

同一 operation 连续因 Node 不可用延后；Cloud 有持久计数，存在其他可领取项和可能已登记的 clone execution。

### 触发

多次延后、Controller 重启或 epoch 换手、手动 retry、事务回滚、RPC 重放、旧 version/epoch，以及真实步骤推进。

### 必须成立

Cloud 每接受一个新不可用轮次，原子保存次数与数据库时间计算的 5/10/20/40/60/60 秒退避，长期封顶 60 秒。
重领、重启、连接恢复、手动 retry 不清零；真实推进或完成时清零，其他原因不增减该计数。
`node_unavailable`、`clone_result_pending`、`clone_result_timeout` 与 `clone_result_unknown` 在持久状态和内部契约中明确区分。
新增迁移只追加，旧行初始次数为 0，不从 version 或错误历史猜造次数，也不改写旧执行、结果和资源绑定。

### 禁止结果

Controller 覆盖 Cloud 退避、部分提交、重放多计、陈旧写入被接受、次数耗尽写成失败、公开内部身份，或等待新增 clone execution。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 持久次数与恢复 | 真实持久事务及跨进程恢复断言完整退避序列、上限、重启/换手/手动 retry 后保持、仅真实推进清零 | Partial | `C-db:TestOperationWaitBackoffPersistsAndResetsOnAdvance` 覆盖 5/10/20/40/60/60、Store 重建、手动 retry 和推进清零；`TestOperationWaitReplayFenceAndRollback` 覆盖换 epoch；`TestOperationWaitCompletionResetsCounter` / `TestOperationWaitCounterClampsBeforeExponent` 覆盖完成清零及长期封顶。尚非真实 Cloud/Controller 进程重启后的完整序列 |
| 原子提交与围栏 | 注入 defer 回滚、响应重放和陈旧 epoch/version，断言次数与时间共同回滚、重放不多计、陈旧写入拒绝 | Covered | `C-db:TestOperationWaitReplayFenceAndRollback` 直接断言 submission 重放相同、旧 version/epoch 拒绝、注入回滚后计数与时间不部分提交、新 epoch 保留计数 |
| 有限原因与迁移边界 | 新库和升级路径及内部契约往返测试证明四种原因不混淆、旧次数为 0、旧身份与结果保留、无内部身份泄漏 | Partial | `C-codec:TestOperationWaitSnapshotFieldsRoundTrip` 覆盖四原因、计数与 createdAt 往返；`C-migration:TestOperationWaitMigrationPreservesLegacyExecutionAndDefaults` 覆盖 0073→0074、重复应用、旧次数 0、Node/结果保持及负值拒绝；C-db 新库 fixture 和 public retry 不泄漏计数；`C-review:TestOperationWaitPendingRejectsNonCloneStep` / `TestOperationWaitPendingRequiresExecution` 覆盖错误步骤/缺执行拒绝。全部旧资源绑定与全部内部身份未逐字段断言；blocked/node_unavailable 参数边界见文末 |
| 不消耗已知失败预算 | 不可用与其他等待原因交错后，断言执行行数及已知 clone 三次预算不因这些轮次变化，不因不可用次数进入 failed | Covered | `C-db:TestOperationWaitPendingYieldsAndKeepsExecution` 断言多轮仍一条原执行及 pending 不改计数；`TestOperationWaitLateFailureKeepsThreeAttemptBudget` 交错 unavailable/pending/timeout，仅真实失败消耗三次预算 |

### 决策依据

ADR D2、D3，不变量 1、3、4、6、7、8。

## Pending Clones Keep One Execution And A Positive Reclaim Interval

### 风险

连接正常但无结果的 clone 无限占用 worker；零秒延后立即再领最老项；每轮让出后重新派发产生重复副作用。

### 前置状态

A 已登记原 execution，Node 保持连接但没有结果；B 有可推进项，当前按创建时间领取。

### 触发

本轮结果等待超过 5 秒，随后重领 A；在轮内注入重连、pending 或结果到达事件。

### 必须成立

5 秒单调时钟窗口不被事件续期，也不额外叠加 D1 宽限；到期持久保存 `retry_wait/clone_result_pending`，
retry_at 为 Cloud 时间 1 秒后。B 能在 A 暂不符合领取条件时进入；A 跨轮只查询同一 execution、Node 与冻结输入，
不新增派发或改变 Node 不可用次数。已知结果到达可立即按原规则接管，不必等满本轮窗口。

### 禁止结果

pending 再领取间隔为 0 或 5 秒，轮内窗口无限延长，把 pending 当 clone_failed/Unknown，或创建替代 execution。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 5 秒轮次与未来 1 秒 | 受控时钟和联合领取序列证明 5 秒到期让出、1 秒后才可再领、B 获得一次机会；把间隔改成 0 时最老项立即再领，断言失败 | Covered | `D-wait:clone_window_uses_monotonic_time` 断言 4999/5000ms；D-joint `pending` 断言不足 7 秒让出、retry_remaining 为正且≤1.1 秒、B 成功；`C-db:TestOperationWaitPendingYieldsAndKeepsExecution` 断言 1 秒延后及立即 claim 得 B。未执行改为 0 的变异运行，现有断言会拒绝 A 立即再领 |
| 原执行与早到结果 | 跨多轮、重启和连接抖动断言派发一次、输入/Node/执行身份不变，早到已知结果即时接管，D2 计数不因 pending 改变 | Partial | D-joint `pending`/`late_success`/`late_failure` 断言跨轮 executions 深相等且派发一次；`C-db:TestOperationWaitPendingYieldsAndKeepsExecution` 覆盖三轮原执行与计数。尚未直接覆盖真实进程重启/连接抖动后的保持，及 5 秒内早到结果的立即接管时间 |

### 决策依据

ADR D3，不变量 2、3、4、7；同样适用于 specs#95 D3 后续轮转，当前不依赖该轮转。

## The Total Result Window Uses Durable Registration Wall Clock Time

### 风险

总截止随重启、排队或连接恢复续期；把等待超时等同 Node Unknown 或失败；超时事务覆盖并发到达的已知结果。

### 前置状态

原 clone execution 已在 Cloud 登记，持久 created_at 已知；没有可接管结果，或结果正与超时判定并发到达。

### 触发

跨过登记后 1800 秒墙钟边界，期间排队、断线、退避、重启、重领或手动 retry。

### 必须成立

总窗口始终从该 execution 的登记 created_at 算起，不从 operation 创建或实际 Git 开始算起，不暂停、不续期。
到期仍无结果时进入 `blocked/clone_result_timeout`，离开普通领取集合；已知结果优先接管，持久化 timeout 前再次核验。
超时保留执行、资源与结果交付，不制造成功/失败事实；仅原预算允许的真实已知失败新 execution 才有新起点。

### 禁止结果

用本进程时间重建总起点、因断线暂停计时、改为 clone_result_unknown、重派、伪造终态、删除目录或终止 Node。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 持久墙钟边界 | 受控时钟及跨重启恢复分别断言 1800 秒前后分类、所有等待计入、不因手动 retry 续期、新已知失败执行独立起算 | Partial | `D-wait:total_window_uses_registration_wall_clock_at_the_exact_boundary` 覆盖 1799.999/1800 秒；`C-db:TestOperationWaitTimeoutBoundaryAndKnownResultRace` 覆盖 Cloud 登记时间判定及 blocked 不领取；`C-review:TestOperationWaitUnavailableExpiredExecutionPreservesCounter` 覆盖到期不增计数。手动 retry、真实重启/退避组合及新执行独立起算尚未全部直接覆盖；调老时间不是实际等满 30 分钟；E-C 补充未调时钟的真实 1801.26 秒边界及原结果随后提交，其他恢复组合仍保留缺口 |
| 超时与结果竞争 | 在结果接管/timeout 提交交界处构造并发，断言已有已知结果不被超时覆盖，执行记录与资源均保留且不新增派发 | Partial | `C-db:TestOperationWaitTimeoutBoundaryAndKnownResultRace` 在旧观察后先提交结果，再提交旧 timeout，断言已知结果不被覆盖；`C-review:TestOperationWaitRefusedKnownResultTakeoverPreservesRound` 断言接管拒绝时完整 operation/execution 保持。尚无屏障控制的真实并发交错及全部资源/派发联合断言 |

### 决策依据

ADR D3，不变量 3、4、5、8。

## A Late Known Success Automatically Recovers A Timed Out Operation

### 风险

operation 已 blocked 后不再被普通领取，晚到成功因此永久不能推进，必须由人手动 retry。

### 前置状态

原 execution 仍受保护，operation 为 `blocked/clone_result_timeout`；没有人工动作或普通重新领取。

### 触发

原执行的真实成功结果晚到并经既有结果交付路径送达 Cloud。

### 必须成立

Cloud 按原 execution 身份与围栏自动接管成功、解除 timeout 阻塞并推进，保留真实结果和原尝试次数。
恢复不依赖人工 retry 或 blocked 项再次普通领取，不新建执行、不重新派发。

### 禁止结果

仍停在 timeout，要求人工操作，因总截止拒绝已知成功，或改写原输入、Node、历史结果与预算。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 晚到成功自动推进 | 真实 Cloud 与 Rust Controller 结果交付联合测试先进入 timeout，再送达原成功；禁止人工 retry 和普通重领，断言自动推进、原 execution 与派发次数保持 | Covered | D-joint `late_success` 先 timeout 再经原 Node 消息交付成功，无 retry 即 succeeded，仍原执行一条且派发一次；`C-db:TestOperationWaitLateSuccessAndUnknownIsolation/clone_result_timeout` 断言断连/心跳过期仍自动恢复。联合时间由夹具调老 |

### 决策依据

ADR D3、不变量 3、5、6、8；owner 要求的晚到成功直接测试。

## A Late Known Failure Follows The Existing Clone Failed Budget

### 风险

timeout 吞掉晚到失败，或恢复时重置/绕过原 clone 三次预算，造成无限新执行或提前失败。

### 前置状态

operation 为 `blocked/clone_result_timeout`，当前原执行分别是同一 operation 的第 1、第 2、第 3 次真实尝试。

### 触发

原 execution 的已知失败结果晚到，由 Cloud 自动接管，不执行人工 retry。

### 必须成立

解除 timeout 后沿 `clone_failed` 规则处理：前两次分别 5/10 秒退避，第三次 failed，保留执行与历史。
只有真实已知失败后依原预算创建的新执行消耗尝试；pending、Node 不可用及 timeout 本身都不耗预算。

### 禁止结果

继续留在 timeout、依赖人工解锁、将原次数清零、超过三次新派发，或把未知等待计为已知失败。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 晚到失败自动恢复与次数上限 | 持久 Cloud 与真实结果交付测试覆盖第 1/2/3 次 timeout 后晚到失败，断言自动进入原 5/10 秒退避或第三次 failed，无人工 retry、不重置次数 | Partial | `C-db:TestOperationWaitLateFailureKeepsThreeAttemptBudget` 覆盖第 1/2/3 次晚到失败、5/10 秒或 failed 及执行行数；D-joint `late_failure` 覆盖首次 timeout 后真实 Rust→gRPC 原结果交付且派发一次。第 2/3 次尚未在该联合交付路径覆盖；E-C late-result-detail.md 补充真实 1800 秒后首次晚到 clone_failed、约 5 秒退避、第二次执行成功；第 2/3 次真实部署分支仍未覆盖 |

### 决策依据

ADR D2、D3，不变量 3、4、8；owner 要求的晚到已知失败直接测试。

## A Node Reported Unknown Remains Blocked

### 风险

把 Node Unknown 与单纯等不到结果混为一谈，套用 timeout 自动恢复而解除未知执行的保护。

### 前置状态

Node 为原 execution 报告 Unknown，Cloud 已按 clone 契约 D6 记录 `blocked/clone_result_unknown`。

### 触发

跨过登记后 30 分钟、重连、重启、轮询或普通调度唤醒，但没有原契约要求的新证据。

### 必须成立

继续 `blocked/clone_result_unknown`，不改成 timeout，不由 timeout 恢复机制解除；执行与资源责任保留，恢复仍遵守原契约。

### 禁止结果

仅因截止到期或重新连接自动解除 Unknown、重新领取派发，或伪造失败、空闲、清理证据。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| Unknown 不走 timeout 恢复 | Node 报告真实 Unknown 后覆盖墙钟到期、重连和重启，断言原因仍是 clone_result_unknown、保持 blocked、无新派发或删除 | Partial | `D-wait:fresh_unknown_cannot_be_reclassified_as_timeout`；D-joint `unknown_at_deadline` 由协议 Node 回 Unknown，到期仍 blocked/clone_result_unknown 且原执行一条；`C-db:TestOperationWaitLateSuccessAndUnknownIsolation/clone_result_unknown` 覆盖过期与 Store 重建后不领取。真实断线重连与 Controller 进程重启尚未在该 Unknown 联合路径覆盖 |

### 决策依据

ADR D3、不变量 5、6、8；clone 契约 D6；owner 要求的 Unknown 保持阻塞直接测试。

## Yielding Preserves Fencing Recovery And Force Stop

### 风险

让出 worker 被当作放弃租约、擦除恢复责任，或普通轮次修改干扰独立 force-stop 路径。

### 前置状态

存在原 execution、旧 epoch 下待对账的 effect，以及与普通 operation 分开的 force-stop 工作。

### 触发

延后、epoch 换手、旧快照推进、结果重放、租约丢失以及 force-stop。

### 必须成立

每轮以最新快照先对账再推进，写入继续受原 epoch/version 与身份围栏保护；失去租约停止普通推进，
会话和未知执行按原规则恢复。force-stop 的独立任务、idle/终止证据与行为保持原样。

### 禁止结果

因短轮次放弃全局 lease、接受旧 epoch/version 写入、换 execution/输入或清空未知账本，或绕过 force-stop 原有证据。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 围栏、对账与强停保持 | 联合故障测试覆盖旧 epoch/version、租约丢失、恢复先对账和普通项等待时的独立 force-stop；直接断言原身份及证据边界保持 | Partial | `C-db:TestOperationWaitReplayFenceAndRollback` 覆盖旧 epoch/version；Desktop `apps/ora-controller/tests/workspaces/force_stop.rs:force_stop_progresses_while_an_ordinary_operation_is_waiting`、`force_stop_preserves_all_cloud_effect_targets_and_evidence`、`absence_and_unavailable_nodes_are_not_termination_confirmation` 覆盖既有独立强停规则，但使用替身 Cloud/受阻 effect。尚非短轮次联合路径同时覆盖租约丢失、先对账和完整强停证据 |

### 决策依据

ADR 继承表、D1、D3，不变量 3、5、6。

## The Single Unavailable Node Reproduction Records Before And After Latency

### 风险

仅证明“最终成功”，没有证明其他租户不再遭遇反复 60 秒阻塞；正常快照或插件耗时被全部归因于故障。

### 前置状态

每个场景固定 Cloud/Controller/Node/cluster 版本，使用新隔离环境和空卷、单 Cloud 副本、租户 A/B。
同版本无故障基线已单独定义；不运行工作流或模型，不修改 operation 行。

### 触发

沿 #698 分别在 A 的 clone 阶段暂停 Node；或 SIGKILL，Node 消失 61 秒后创建 B 的 Project。

### 必须成立

并列记录修复前、修复后及各自无故障基线；采集 B 创建到首次领取、每轮领取/延后/推进和完成耗时，
A 每轮占用、次数/retry_at、原 execution 身份与结果。修复后不能再有每条不可用项 60 秒本地等待。

### 禁止结果

把历史 173.5/168.6 秒与 52.0/66.5 秒当作新版本门槛或修复证据，把组件测试或最终成功替代延迟记录。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 两种故障的前后对照 | 当前创建时间排序下分别完成上述部署路径，提供固定版本、真实时间线和各自无故障基线，区分正常准备耗时；后续轮转采用同一场景 | Covered | 部署 E-P/E-K：P 基线/B 为 54.66/55.85 秒，K 为 43.34/57.43 秒；K 在故障后 61 秒建 B。两租户及 B 后续 Workspace 均推进，原 clone 各一条；次数、retry_at、轮次占用和历史对照见 E-D。首次领取为采样上下界，快速步骤的版本跳跃原样保留，不声称逐事务完整日志或严格延迟保证 |

### 决策依据

ADR 背景、D1/D2、落地顺序与验证第 4 项；#698 的历史数据仅作对照。

## Several Unavailable Operations Leave Progress Opportunities For Healthy Tenants

### 风险

单失联项改善不能证明多条旧项同时存在时，健康租户的领取及后续步骤仍能推进。

### 前置状态

至少三个不同 Project 的旧 operation 各自等待本 Project 不可用的 Workspace Node，三条旧项同时位于全局队列，另有健康租户；使用固定版本与新隔离环境。

### 触发

旧项持续循环时创建健康租户 B 的 Project，并在旧项执行轮次内加入健康租户 C 的 Project。

### 必须成立

先在当前创建时间排序下记录各旧项 2 秒宽限、Cloud 退避、健康项每次领取和后续推进，
比较旧实现与同版本无故障基线，没有每条旧项 60 秒的阻塞；不以 specs#95 D3 合并为验收前提。

### 禁止结果

只报告首次领取或最终成功而遗漏中间阻塞，或把估算 N×2 秒当作测得的严格延迟上界。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 多失联项的真实排队时间 | 上述部署场景中各项领取、延后、退避和健康项完整推进的时间线及对照；轮转落地后复用场景 | Covered | 部署 E-M：三个不同租户/Project 各自的 Workspace Node 被 SIGKILL，旧操作同时循环，各完成 4 个不可用轮次；观测领取→延后为 2.007–3.024 秒（含 RPC），次数 1/2/3/4，退避约 5/10/20/40 秒，无约 60 秒占用。同栈基线 50.52 秒；B 首次领取 0–0.46 秒、完成 47.50 秒；C 首次领取 13.67–14.16 秒、完成 99.21 秒。C 的 DB 创建时刻落在旧项 running→defer 区间，POST 返回后仍采到同版本 running。逐轮 hold/counter/retry_at 和健康项后续排队段见 E-M；首次领取是采样界限，快速步骤漏版本不补造，不把总耗时差全部当作排队或租户延迟保证 |

### 决策依据

ADR D1/D2、不变量 7、后果与兼容性、落地顺序与验证第 5 项。

## A Connected Node Without A Result Does Not Hold The Worker Indefinitely

### 风险

断连测试不能证明连接和心跳正常时无结果的循环会退出；人工推进时钟不能证明真实 30 分钟部署等待。

### 前置状态

固定版本真实栈中 A 的原 clone 已派发，Node 保持会话与心跳却不交付结果；健康租户 B 有工作。

### 触发

跨越多个 5 秒轮次与 1 秒再领取间隔，等待真实总窗口到期；分别交付原执行的晚到成功、已知失败或 Node Unknown。

### 必须成立

各轮让出普通 worker，B 取得推进机会，派发始终一次；真实 30 分钟后 timeout 与 Unknown 保持区分。
晚到已知结果无人工步骤自动接管，Unknown 仍阻塞。记录本地等待、健康结果可能约多 1 秒加排队、
claim/defer 写入与多条慢 clone 约 N×5 秒轮内占用的实际观察，不把估算当保证。

### 禁止结果

用断线代替持续连接、人工预置 deadline 冒充真实等待、拼接无关执行结果，或未覆盖本场景就宣布完整实现。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 | 代表性证据与边界 |
| --- | --- | --- | --- |
| 持续连接下的跨轮与结果恢复 | 真实 Cloud/Rust Controller/Node 联合记录覆盖持续会话、B 推进、同一执行多轮、真实总窗口及晚到结果自动恢复/Unknown 阻塞的各分支；受控时钟测试单独标明范围 | Partial | 部署 E-C：Node 保持 ['connected']，同原 execution/Node 与两条安全 input ref，B 在 60.80 秒完成且后续 Workspace 推进；记录 178 个延后/阻塞轮次。真实 1801.26 秒观察到 clone_result_timeout；原结果 clone_failed 于登记后 1802.71 秒提交。原结果提交前只有一条业务执行；晚到 clone_failed 接管后按原预算生成第二次执行并成功（见 E-C late-result-detail.md），不是 pending 重派；业务执行身份不等于网络帧发送次数。完整 input 的直接深相等由 D-joint 支持，部署只投影两条安全 ref。第 2/3 次晚到失败、真实晚到成功/Unknown、多慢 clone 排队及真实重启组合仍未覆盖；D-joint 的调老时钟路径单独保留范围 |

### 决策依据

ADR D3、不变量 2–8、后果与兼容性、落地顺序与验证第 6 项。

## 证据位置与保留缺口

- 固定产品与工具：Cloud #206 merge `1cfaed3`、Desktop #750 最终 PR head `65fc457`（合入后补 merge commit）、Cluster `562351d`。E-T/E-P/E-K/E-C 保留实际 `9078570` 基线；E-M 复用其生产源码相同的镜像并保留原 revision 标签。本规格 PR 须等 Desktop #750 先合入。
- E-T 为测试范围摘要；48 个 lib 测试及 6 个联合场景、79.83 秒的运行结果转述自 PR750，本轮未重跑。SHA-256 标识该摘要文件，不冒充原始测试日志摘要。
- E-D 为部署总报告；E-P/E-K/E-C 分别对应 `p/`、`k/`、`c/` 的 case-report.json 及同目录安全时间线。初次 C 捕获漏过快速重试的记录保留在 E-D，未冒充 30 分钟通过。
- E-M 为三个不同 Project 各自 Node 失联、健康 B/C 的新部署证据，取代旧版同 Node 限制。
- 以下文件位于生成候选的 cluster 工作区，未提交。路径均相对 cluster 仓库根目录；SHA-256 从保存的安全报告文件一次计算，没有增加运行判定门槛或重算比对。

| 证据 | cluster 相对位置（未提交） | SHA-256 |
| --- | --- | --- |
| E-T | `.acceptance/landing803/test-evidence.md` | `ad335076d6626c125ff5f8b10f122ca36ee73228d343abea7b382f04f605893c` |
| E-D | `.acceptance/landing803/cleanup-summary.json` | `a111a0f3d4e796bd7e9bc387f29372f326f9984d898cdd8302a0bab9c9657c48` |
| E-P | `.acceptance/landing803/p/case-report.json` | `ffe76ab72276b009054267050c9fc26c670467789a42dc264773d80ddc350c07` |
| E-P | `.acceptance/landing803/p/operation-rounds.jsonl` | `0aa4dccdceb412235a278169766ae244237d5bb113c6a32290a6e525dd238a1e` |
| E-P | `.acceptance/landing803/p/hol2-operations.jsonl` | `621fae1a35bac5597135132a4c81b8ff834c1c00e88638216d1bdc74d5121430` |
| E-P | `.acceptance/landing803/p/hol2-controller-safe.json` | `7683ee5893cb6706c7cf691c62155ceb5de65863d6f26edd74e5eb0904eaa188` |
| E-K | `.acceptance/landing803/k/case-report.json` | `73fb3c5a0a2a70187bd89e3918429bf2e20d13a8e1c4a7a592da872229844886` |
| E-K | `.acceptance/landing803/k/operation-rounds.jsonl` | `822d252d506be59950b81daf69e81c15ea02e62ac227cc50240caf71f8be7246` |
| E-K | `.acceptance/landing803/k/hol2-operations.jsonl` | `2744f329038d55c82c40f607d80ad7f7cfff3d74ada5f9889005bcfe74080ba4` |
| E-K | `.acceptance/landing803/k/hol2-controller-safe.json` | `591de96afee5a4a02003a072ccad60e3e23fca6bddc0b8b7c9d5490a4a23b53d` |
| E-C | `.acceptance/landing803/c/case-report.json` | `3320f933344b32c8674d1a5c5b0a4f530b559734e050225f7f1bb8068ac6e905` |
| E-C | `.acceptance/landing803/c/operation-rounds.jsonl` | `bf1f109a4e763263432f91dd6b054e524966cbd890f19b1d7a8eedf0ba7e8b9a` |
| E-C | `.acceptance/landing803/c/hol2-operations.jsonl` | `9e1451edbbf06b1825d897f230b6deed5726da93c6e578ec384fed9e2341b8ee` |
| E-C | `.acceptance/landing803/c/hol2-controller-safe.json` | `694f78f31f95b3edd0af3020e5be91ac83124bdc676891dff65576f73c44ed8b` |
| E-D | `.acceptance/landing803/deployment-report.md` | `10e5aa298b6489502d2c1e3e68fb4fba63a7c5ad3dad6c764930f68bf13ccc00` |
| E-C | `.acceptance/landing803/c/late-result-detail.md` | `6e69f506a31242e6664df2cd64b135a91aeaa5d3d96f6430b09953e60ba92acb` |
| E-M | `.acceptance/landing803/multiple-unavailable/report.md` | `5a9667825287018801a5f02019420102893e2b8bdfe304bb4856b6ac8ee60d0c` |
| E-M | `.acceptance/landing803/multiple-unavailable/case-report.json` | `21043825cb71e4bbfad18049620438c6b2aa03d0811d1cfc2b9d8a87ed432714` |
| E-M | `.acceptance/landing803/multiple-unavailable/cleanup.json` | `c6c466ba496228f71fe4c1f7a804d766bca38d38fdea27d1363a332619337dd3` |
| E-M | `.acceptance/landing803/multiple-unavailable/candidate-note.md` | `9c01648c4b1fd46517785bd774fc8619400231b4a5f76c44bc6ac23d4fb2a1c6` |
| E-M | `.acceptance/landing803/multiple-unavailable/operations.jsonl` | `100f8ef63a0b2a78979eda6ea6e7c65581f61dd19404913f73e3d7ff3da62dfb` |
| E-M | `.acceptance/landing803/multiple-unavailable/rounds.jsonl` | `8f2eac989412d9a2798e29f5192af7757914f8de5af762257609363163e482cd` |
| E-M | `.acceptance/landing803/multiple-unavailable/controller-safe.json` | `461aa3b6e228ec592a0a04a5475b7864f3e2218cd6297892052ac714b703aecd` |

多失联项的拓扑：每个 Project 使用各自的 Workspace Node，不要求多个 Project 共用一个 Node。E-M 已提供这一拓扑的直接部署时间线；E-D 旧报告中的“同 Node 无法构造”对应旧场景定义，不再构成本义务的限制。旧报告和摘要保持原样。

已知独立后续边界：Cloud `internal/core/control.go:operationCommand` 对 `node_unavailable` 豁免 `retrySeconds` 的 1..3600 校验，只有 `state=retry_wait` 进入 Cloud 计算退避。
`blocked/node_unavailable` 仍可能落入通用更新并使用调用方的无界 delay，所列测试未覆盖该组合。有限原因白名单不等于所有原因/状态/参数组合均已封闭；本 ADR 的正常 Controller 路径不会据此获得新的 blocked 延后语义。
