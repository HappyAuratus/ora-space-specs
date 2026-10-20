# 调度串行、节点提交与恢复

依据[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)（D1 条件②、D4、D6）。
本用例立项时 Cloud 没有工作流调度器：生产里的运行停在 `pending`，唯一会写节点状态的是开发夹具
`MockWorkflowRunSimulator`（[`internal/collab/collab.go`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/collab/collab.go)），
而根决策明确模拟 trace 不能作为验收证据。

Cloud 侧调度器已在 [cloud#67](https://github.com/ora-space/cloud/pull/67)（认领、从已提交状态重算就绪集合、节点结果比较交换）、
[cloud#72](https://github.com/ora-space/cloud/pull/72)（按会话派发与结算）、[cloud#91](https://github.com/ora-space/cloud/pull/91)
（已派发会话的 Node 不可达时结算失败）、[cloud#100](https://github.com/ora-space/cloud/pull/100)（agent 运行的取消写入者）、
[cloud#108](https://github.com/ora-space/cloud/pull/108)（已派发会话的回合窗口，merge `a4a13e8`）与
[cloud#109](https://github.com/ora-space/cloud/pull/109)（工作流的取消写入者与有界终态，merge `27e7f48`）的堆叠里落地，
下表因此不再全是 `Missing`。仍为 `Missing` 的是两类需要真实部署的证据——**多副本**与**崩溃接手**。
要求"真实执行侧"的行只记 `Partial`：现有证据走控制面接缝或真实 PostgreSQL，真实 Node / Agent 回调
仍属 D1 落地顺序第 3 步（ora-space/circus#70）。

## At Most One Serial Executor Advances A Workflow Run

- 风险：多副本或多请求同时推进同一个运行，两个执行者各自派发同一批就绪节点，同一运行的 Workspace 与节点结果互相踩踏。
- 前置状态：运行已创建并冻结身份；部署有多个 Cloud 副本；运行有多个可派发的就绪节点。
- 触发：两个副本同时收到推进请求；一次推进的租约或排他权在事务中途失效；同一运行内多个 agent 节点。
- 必须成立：同一 WorkflowRun 在同一时刻至多有一个串行执行者，排他权是跨进程的（按 run 取排他事务权，不依赖任何进程内状态）；首期同一运行内的多个 agent 节点串行执行，共用该运行的 Workspace；排他权随事务结束释放，不靠心跳维持。
- 禁止结果：两个副本同时派发、用进程内锁表当跨进程锁、把"并行执行的多个 agent 节点"当作优化、锁未释放时就开始第二批派发。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 多副本下同一运行只有一个串行执行者 | 多副本真实 PostgreSQL：同步竞争的推进请求，检查只有一个副本派发、另一个要么等待要么幂等退出 | Missing |
| 同一运行内的多个 agent 节点串行且共用同一 Workspace | 真实执行侧：一个运行内两个 agent 节点的派发与回调顺序，Workspace 归属唯一 | Partial |

- 决策依据：D1 条件②（无状态调度、单 run 串行、多副本与租约），D4（同 run 内 agent 节点首期串行），不变量 3。
- desktop 参考实现（`8909f07`）一侧：**Partial** — 同一 run 只有一个推进者有直接证据：per-run 排他锁是 backend composition 里的 `KeyedResourceLocks`（[`crates/backend/src/git_cleanup/keyed_locks.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/git_cleanup/keyed_locks.rs) 的 `exclusive_waits_for_shared_holders`），[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `concurrent_resume_from_failure_reruns_the_failed_node_once`（两线程各持排他权，结果恰为 (resumed, refused)=(1,1)）与 `concurrent_cancel_and_complete_report_the_committed_outcome`。**但"同一 run 内多个 agent 节点串行"与参考实现相反**：desktop 在同一次调度 pass 里并行派发 sibling agent，并允许它们同时 in-flight（[`crates/application/src/workflow_run/engine/aggregator_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/aggregator_tests.rs) 的 `parallel_fan_in_uses_the_declared_priority`、`d2_tests.rs` 的 `failed_run_binds_in_flight_sibling_and_does_not_dispatch_join`），其文档也写明并行 agent 共用 Workspace 只提供时间窗 diff、不提供改动归属。首期串行是本决策相对参考实现的**偏离**，没有 desktop 证据可依托，只能靠 Cloud 自己的测试证明。
- 本次直接证据：**Partial** — 第 2 行已部分就位：[cloud#72](https://github.com/ora-space/cloud/pull/72) 的
  `TestWorkflowSessionRunsTwoAgentNodesInOrder` 在真实 PostgreSQL schema 上跑完同一运行内的两个 agent 节点，断言
  a1 结算后 a2 才派发、a2 在途时运行保持 `running` 且下游 `output` 不跑、两个节点各拿自己的执行与自己的结果
  （a2 的 `reason` 不是 a1 的）、每个 agent 节点恰好一条注册会话。第 1 行仍为 **Missing**：没有按 run 取跨进程
  排他权的代码路径，参考实现用进程内锁表（desktop `workflow_run/engine/engine.rs:43-49`），根决策明确要求 Cloud
  换成跨进程等价物并在运行 issue 记录参数。
- 仍缺：两个节点确实落在**同一个** Workspace 的断言（用例只把同一个 Workspace 交给两次派发，没有断言归属唯一）；
  锁的键、租约时长、续租与失效窗口的具体数值由实现给出；多副本联合验证属 D1 落地顺序第 3 步（ora-space/circus#70）。

## A Node Result Commit Is A Compare And Swap And Late Callbacks Are No-ops

- 风险：重复或迟到的回调推进图，或者旧执行代的回调被当成当前结果，同一节点被结算两次、后续节点被派发两次。
- 前置状态：节点已派发并带有派发时的执行代；执行侧可能重发回调，或在代已过期后回调。
- 触发：同一结果重复提交；代过期（新的派发或控制代已经产生）后提交；两个回调几乎同时到达。
- 必须成立：节点结果提交带 from→to 的当前状态，是状态比较交换；重复或迟到的回调是空操作，不改变节点状态、不推进图；回调携带派发时的执行代，代过期的回调被拒绝；被拒绝的回调不产生任何副作用。
- 禁止结果：无条件覆写节点状态、把过期的回调当当前结果、重复回调触发第二次派发、拒绝回调时留下部分写入。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 结果提交是比较交换，重复与迟到回调为空操作 | 真实 PostgreSQL：并发与重复提交的允许结果集合，检查状态只前进一次 | Partial |
| 回调受派发代约束，过期代被拒绝且无副作用 | 真实执行侧：延迟到过期代之后的回调被拒，节点状态与派发记录都不变 | Missing |

- 决策依据：D1 条件②（节点结果提交是状态比较交换、回调带执行代），不变量 3。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 重复完成是空操作（[`crates/backend/src/workflow/run/engine.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/engine.rs) 的 `duplicate_completion_reports_are_idempotent_no_ops`）；被替换 attempt 的迟到回调与重复唤醒不改变状态、不派发（[`crates/backend/src/workflow/run/retry_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/retry_tests.rs) 的 `late_callbacks_and_repeated_wakes_are_no_ops_and_concurrent_waits_fire_independently`）；非 Running 节点完成返回 `NotRunning` 且不会派发 join（[`crates/application/src/workflow_run/engine/d2_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/d2_tests.rs) 的 `failed_run_binds_in_flight_sibling_and_does_not_dispatch_join`）。契约写在 [`crates/application/src/workflow_run/engine/ports.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/ports.rs) 的 `transition_node_run_status` 注释里（仅当当前状态恰为 `from` 才转移）。仍缺：DB 层没有"from 不匹配则不改变"的直接测试，行为证据在引擎与集成层。
- 本次直接证据：**Partial** — 第 1 行已部分就位：[cloud#67](https://github.com/ora-space/cloud/pull/67) 的
  `TestWorkflowAdvanceIsIdempotent` 连跑三次推进，断言已结算的运行不被二次认领、版本不漂移；[cloud#72](https://github.com/ora-space/cloud/pull/72)
  的 `TestWorkflowSessionLateReceiptIsANoOp` 重放同一个终态回执，断言运行状态、节点状态与版本逐字节不变，`TestWorkflowSessionDispatchIsNotRepeatable`
  断言同一节点的工作项被派发消耗掉后重放派发是同一个事实而不是第二次会话；[cloud#91](https://github.com/ora-space/cloud/pull/91)（merge
  `068d6718`）的 `TestWorkflowSessionResultAfterTheNodeSettledIsKept` 覆盖另一种迟到回调：节点已被 Cloud 按 Node 不可达结算、
  运行已终态后，真实会话结果到达仍不改写运行状态、错误与版本，只保留在执行行的 `result` 上。第 2 行仍为 **Missing**：控制面已有代机制的
  列（[`0024_agent_run_control_plane.sql`](https://github.com/ora-space/cloud/blob/7704089d453f88d66e06d462aafe554e03a11855/internal/core/migrations/0024_agent_run_control_plane.sql)
  的 `node_executions.dispatched_epoch`、`0014_clone_coordination.sql` 的同类列），但没有用例证明过期代的回调被拒且无副作用。
- 仍缺：两个回调**并发**提交同一节点结果的允许结果集合；过期代回调被拒且状态与派发记录都不变的用例。

## Recovery Recomputes The Ready Set From Committed State

- 风险：调度器把图状态放在内存里，副本崩溃或重启后进程内状态丢失，已完成的节点被重跑、就绪节点再也派发不出去，或运行停在中间状态无人推进。
- 前置状态：运行有已提交结果的节点、在途未提交结果的节点，以及尚未就绪的节点；调度副本随时可能崩溃。
- 触发：推进过程中副本崩溃；崩溃后另一个副本（或重启后的同一副本）接手；派发已发出但结果未知。
- 必须成立：每次推进都从 Cloud 已提交的状态重算已完成、在途与就绪集合，不缓存图状态；副本崩溃后不丢失进度，接手者从已提交状态继续；已经提交结果的节点不重复派发；在途没有结果的节点按中断结算为失败。
- 禁止结果：把内存中的进度当成权威、崩溃后重跑已提交的节点、崩溃后运行永久卡在中间状态、把"派发过"当作"已完成"。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 就绪集合只从已提交状态重算 | 真实 PostgreSQL：以任意中间状态作为起点（等价于崩溃后接手），重算结果与线性推进一致 | Partial |
| 崩溃接手不重跑已提交节点、不丢进度 | 真实执行侧：推进中途杀死调度副本后由另一副本接手，检查派发集合与最终结果 | Missing |

- 决策依据：D1 条件②（每次推进从已提交状态重算、崩溃后不恢复内部状态），D4（崩溃恢复只依据已提交的节点结果），不变量 4。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 分支激活与就绪集合只从持久化行推导（[`crates/application/src/workflow_run/engine/branch_projection.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/engine/branch_projection.rs) 的 `selected_branch_activates_only_the_matching_successor`、`fan_in_with_an_inactive_branch_still_becomes_ready`、`unresolved_condition_leaves_successors_unreached` 等），并且经编码 → 真实 SQLite 重载 → 再解析后执行仍成立（[`crates/backend/src/workflow/run/aggregator_persistence_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/aggregator_persistence_tests.rs) 的 `persisted_branch_outputs_reach_downstream`、`persisted_false_value_reaches_downstream`）；崩溃后只重做失败节点与其后继、已成功 sibling 保留（[`crates/backend/src/workflow/run/resume_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/resume_tests.rs) 的 `resume_from_failure_after_orphaned_crash_recovery`、`resume_from_failure_redoes_failed_join_and_finishes`）。说明：desktop 的串行执行者是进程内的，跨进程多副本不适用，这一条的多副本部分在参考实现里没有对应证据。
- 本次直接证据：**Partial** — 第 1 行已部分就位：就绪集合是纯函数，[cloud#67](https://github.com/ora-space/cloud/pull/67) 的
  `TestWorkflowProjectionLinearGraph`、`TestWorkflowProjectionPendingUpstreamWaits`、`TestWorkflowProjectionRootlessNodeIsInactive`、
  `TestWorkflowProjectionUnlabeledEdgeIsElse` 覆盖它的四类结果，`TestWorkflowAdvanceSucceedsLinearRun` 与
  `TestWorkflowAdvanceIsIdempotent` 证明推进每次都从行上的已提交状态重算、不缓存图状态（参考语义为 desktop 的
  `resume`，`engine.rs:328-330`："Recomputes the ready set from persisted state"）。第 2 行仍为 **Missing**：没有用例以
  任意已提交中间状态为起点重启推进（等价于崩溃接手），也没有杀掉副本后由另一副本接手的部署级验证。
- 仍缺：从任意已提交中间状态重启推进的 DB 用例（投影本身目前是纯函数单测）；崩溃接手的部署级验证，与上一条共同构成
  D1 落地顺序第 3 步的验收面（ora-space/circus#70："真实跑通 Start → Agent → Output，fixtures=false"）。

## A Node With No Committed Result Is Never Recorded Successful

- 风险：未知结果被当作成功，运行的终态谎报了一次没有发生或没有留下结果的执行；下游节点基于不存在的输出继续跑。
- 前置状态：节点已派发，回调未到、结果丢弃，或进程在写入结果前消失。
- 触发：执行侧超时、结果响应丢失、运行被中断后恢复、回调从未到达。
- 必须成立：没有已提交结果的节点永不被记为成功；恢复时这种节点按中断结算为失败，失败随运行终态对外可见；只有提交了结果的成功才允许推进下游节点。
- 禁止结果：把"已派发"或"未知"记成成功、用 HTTP 200 或夹具 trace 代替真实结果、恢复后把中断节点静默留成在途、把失败改写成成功以让运行通过。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 无已提交结果的节点不进入成功状态 | 真实 PostgreSQL + 执行侧：丢弃回调、超时与中断后的节点状态与运行终态 | Partial |
| 运行终态不早于最后一个节点的结果提交 | 真实执行侧：运行进入终态的时点与节点结果提交顺序 | Partial |

- 决策依据：D1 条件②（在途未提交结果的节点按中断结算为失败、绝不记为成功），D4（崩溃恢复），D7（首期节点产出落在节点状态里），不变量 5。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 恢复把在途节点结算为 `interrupted_by_restart` 并让运行失败，已成功的节点保持成功（[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `recovery_fails_interrupted_turns_and_resumes_stalled_runs`、[`crates/backend/src/workflow/run/recovery.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/recovery.rs) 的 `an_interrupted_outer_row_still_fails_the_run` 与 `a_crashed_round_inside_a_fail_iteration_fails_the_node_and_the_run`、[`crates/backend/src/workflow/run/resume_gap_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/resume_gap_tests.rs) 的 `boot_sweep_interrupted_node_resumes_as_attempt_two`）。
- 本次直接证据：**Partial** — 第 1 行已部分就位：[cloud#72](https://github.com/ora-space/cloud/pull/72) 的
  `TestWorkflowAdvanceFailsRunOnProjectlessAgentNode`（#67 时代的 `TestWorkflowAdvanceFailsRunOnUnwiredAgentNode` 随真实会话接线改名）
  断言缺 Project 的 agent 节点以 `workflow_run_project_required` 结算为失败、既不进成功状态也不让下游提交状态；
  同 PR 的 `TestWorkflowSessionRunsTwoAgentNodesInOrder` 断言在途无结果的
  节点保持 `running` 而不被记成功、`TestWorkflowSessionStartDeadlineFailsNode` 与 `TestWorkflowSessionFailureFailsRun`
  断言超时与失败的节点连同运行一起进入失败终态。[cloud#91](https://github.com/ora-space/cloud/pull/91)（merge `068d6718`）的
  `TestWorkflowSessionNodeGoneSettlesTheNodeFailed` 用两种 Node 消失形态（心跳超过 `delivery_unreachable_after`、强制停止把沙箱
  与其中未完成的执行一起终止）断言已派发而无结果的节点以 `node_unavailable` 失败、运行随之进入失败终态并声明 Workspace 删除；
  `TestWorkflowSessionNodeSilenceWithinWindowHolds` 与 `TestWorkflowSessionUnconfiguredWindowHoldsTheNode` 断言窗口内或窗口
  未配置时会话保持 `running`，不把沉默记成成功；`TestWorkflowSessionResultAfterTheNodeSettledIsKept` 断言结算后到达的真实结果
  留在执行行，不改写节点与运行的终态。[cloud#108](https://github.com/ora-space/cloud/pull/108)（merge `a4a13e8`，ora-space/circus#299
  选项 A）补上已派发而既无结果也无成功 `turnEnded` 的会话：`TestWorkflowSessionTurnDeadlineFailsNode` 断言执行行注册时刻超过
  `workflow.session_turn_timeout`（默认 2 小时）时，节点与运行以 Cloud 自己的 `session_turn_deadline_exceeded` 进入失败终态并
  声明 Workspace 删除，重复推进不再扫描终态运行；`TestWorkflowSessionTurnDeadlineHoldsWithinWindow` 与
  `TestWorkflowSessionTurnDeadlineUsesTheConfiguredWindow` 断言窗口内、以及部署配置的窗口值下会话保持 `running`，不把沉默记成
  成功；`TestWorkflowSessionTurnDeadlineDoesNotOverrideSuccessfulTurn` 与 `TestWorkflowSessionTurnDeadlineDoesNotOverrideAResult`
  断言成功 `turnEnded` 与已提交结果分别优先于超窗结算（判定顺序为结果 → Node 不可达 → 回合窗口）；
  `TestWorkflowSessionTurnDeadlineResultAfterTheNodeSettledIsKept` 断言超窗结算后到达的真实结果留在执行行、不改写节点与运行的
  终态；`TestWorkflowSessionTurnDeadlineIsPerExecution` 断言窗口按执行行计时，重试的新执行行重新计时。
  第 2 行已部分就位：同 PR 的 `TestWorkflowRunningCarriesNoFinishedAt`
  断言运行仍在执行时不带 `finishedAt`。唯一会写 `node_states` 成功的 `MockWorkflowRunSimulator` 在没有真实结果的
  情况下直接产出成功状态，根决策已把这类模拟 trace 排除在验收之外。
- 仍缺：显式断言"终态写入发生在最后一个节点的结果提交之后"（现有用例只覆盖运行中不带 `finishedAt`）；回调丢失或
  中断后重试仍不成功的用例；Node 不可达与回合窗口仍只在控制面接缝上验证，真实执行侧的中断结算与终态时点验证仍属 D1 落地顺序第 3 步。

## Cancellation Is A Recorded Request With A Bounded Terminal State

- 风险：取消请求被当成可立即生效的终态，正在收尾的资源与执行失去归属；或者根本没有取消入口，用户只能看着运行跑完。
- 前置状态：运行处于可执行状态并已派发节点；运行拥有 Workspace 等需要释放的资源。
- 触发：发起者请求取消；同一运行在取消请求后又收到节点回调；取消写入者与清理路径并发。
- 必须成立：取消先记录请求，运行进入有界的收尾路径，实际终态在清理完成后才写入；清理期间到达的回调不把运行拉回执行中；首期必须有工作流的取消写入者，agent 运行侧复用同一条收尾路径并一并补上；取消不泄漏运行的 Workspace。
- 禁止结果：收到请求就直接写终态、运行卡在收尾状态、清理后仍留资源、取消后回调把运行改回 `running`、"没有取消入口"被当成首期可接受状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 取消是请求加有界终态，终态在清理后写入 | 真实 PostgreSQL + 执行侧：取消与清理、回调的并发顺序与最终行状态 | Partial |
| 工作流与 agent 两侧都有取消写入者 | 真实 HTTP 入口：两侧的取消请求都落到持久状态并走同一条收尾路径 | Partial |

- 决策依据：D4（取消是请求加有界终态），D6（取消写入者纳入首期，agent 侧一并补上），不变量 3；继承 [issue-run 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md) D6。
- desktop 参考实现（`8909f07`）一侧：**Covered** — 取消是有界终态而非即时终态：清理失败时仍提交 `Cancelled`（[`crates/backend/src/workflow/run/operations/tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/operations/tests.rs) 的 `cancellation_commits_even_when_session_cleanup_fails`）、取消与完成并发时以已提交结果为准（同文件的 `concurrent_cancel_and_complete_report_the_committed_outcome`）、取消后迟到唤醒是空操作（[`crates/backend/src/workflow/run/retry_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/retry_tests.rs) 的 `cancel_during_the_wait_cancels_immediately_and_the_late_wake_is_a_no_op`）、已取消运行不能被清理复活（[`crates/backend/src/workflow/run/loop_exit_tests.rs`](https://github.com/ora-space/desktop/blob/8909f07/crates/backend/src/workflow/run/loop_exit_tests.rs) 的 `loop_exit_cleanup_cannot_revive_a_cancelled_run`）。desktop 侧缺的是入口本身：取消由本地界面/命令触发，Cloud 需要的 HTTP 写入者没有参考实现。
- 本次直接证据：**Partial** — 第 2 行的 agent 侧已就位：[cloud#100](https://github.com/ora-space/cloud/pull/100)（merge `f405823e`）落地
  `POST /tenants/:tid/issues/:iid/runs/:rid/cancel`。真实 HTTP + PostgreSQL 的 `TestCancelRunRecordsTheRequest` 断言请求只记录
  `issue_runs.cancel_requested_at`、返回 202 与记录时刻，运行的 phase、status、Thread 状态与 Workspace 都不动；
  `TestCancelRunIsIdempotentUnderTheSameKey` 与 `TestCancelRunKeepsTheFirstInstantAcrossFreshKeys` 断言重复请求返回同一时刻且不再写入；
  `TestCancelRunOnlyAcceptsTheAgentExecutionWindow`、`TestCancelRunIsNotFoundOutsideItsScope` 与 `TestCancelRunAuthorizesLikeEndingTheThread`
  断言越过 agent 执行窗口、越界与越权的调用者被拒；`TestCancelledRequestEndsTheDeclaredSession` 与
  `TestCancelledRequestReleasesTheUndeclaredSession` 断言端点写入的请求正是 `ReactToCancelledAgentRunsOnce` 消费的事实，
  分别落到 EndSession{cancelled} 与 `deliveryState=skipped` 的释放。工作流侧已由 [cloud#109](https://github.com/ora-space/cloud/pull/109)
  （merge `27e7f48`）落地 `POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/cancel`，第 1 行由 **Missing** 推进到 **Partial**，
  第 2 行仍为 **Partial**：`TestWorkflowRunCancelRequestRecordsTheInstant` 断言请求只写 `workflow_runs.cancel_requested_at`、
  返回 202 与数据库记录的时刻，运行保持 `pending` 且 `started_at` 为空，换幂等键重复请求返回同一时刻；
  `TestWorkflowRunCancelRequiresTheInitiatorOrAnAdmin` 断言有冻结执行者绑定时非发起者的普通成员被拒
  （`workflow_run_cancel_forbidden`）且不落库，发起者可以取消；`TestWorkflowRunCancelAfterTheRunSettledIsRefused` 断言终态运行
  返回 `run_not_cancellable` 而不是被记录；`TestWorkflowRunCancelRacesToOneRequest` 断言两个并发请求只记录一个时刻、只发生一次
  请求引起的版本变化。`TestWorkflowRunCancelBeforeStartNeverStarts` 断言未开始的运行在推进时直接结算 `cancelled`：不认领、
  不写 `started_at`、不建 Workspace、不派生删除操作，请求在结算后保留且后续推进不再改动终态运行；
  `TestWorkflowRunCancelStopsTheRunningSession` 断言在途会话经既有 EndSession 中继停止（`end_session`、`reason=cancelled`，
  指向该运行的执行），重复推进是重放而不是第二条请求，会话终态后节点与运行结算 `cancelled` 并释放 Workspace；
  `TestWorkflowRunCancelEndWindowSettlesCancelled` 断言结束请求超过 `AgentEndConfirmTimeout` 仍未获确认时，节点与运行结算
  `cancelled`、Workspace 释放，之后到达的结果只保留在执行行、不改写终态；`TestWorkflowRunCancelKeepsACommittedSessionResult`
  与 `TestWorkflowRunCancelReusesTheExistingEndRequest` 断言已提交结果与会话自己上报的终态优先于取消，运行仍结算 `cancelled`
  并释放 Workspace；`TestWorkflowRunCancelWinsOverAnUnreachableNode` 断言 Node 不可达时由取消而不是 `node_unavailable` 决定
  终态，且不再向已消失的 Node 释放结束请求，Workspace 照常释放。
- 仍缺：工作流侧只有核心层 `Store.Public` 用例，尚无真实 HTTP 边界的集成用例（agent 侧的入口用例在
  `integration/agent_run_cancel_test.go`，工作流侧是 `internal/core/workflow_run_cancel_db_test.go`）；两侧取消与清理、迟到回调的
  **并发**顺序证据（G-026），以及真实执行侧下的取消时点验证，属 D1 落地顺序第 5 步，归 ora-space/circus#75。
