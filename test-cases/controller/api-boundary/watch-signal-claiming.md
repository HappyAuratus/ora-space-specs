# Controller 消费 Watch 信号流验收

依据[已实现的后续决策](../../../decisions/controller/api-boundary/20260924-controller-consumes-watch-signals.md)。
证据核对 desktop 主干 `03d521b`（含 [desktop#614](https://github.com/ora-space/desktop/pull/614)），
`cargo test -p ora-controller` 已在该提交上本地重跑，PR CI 在三个平台通过。行为测试位于 `apps/ora-controller/tests/cloud.rs`：以内存假 Cloud
（`apps/ora-controller/tests/support/fake_cloud.rs`，由 `ora-controller-proto` 的 `test-server` feature 生成的
服务端桩托管真实契约并记录调用序列）为对端、以 `ControllerRuntime` 驱动云端持久模式。状态迁移由
`apps/ora-controller/src/cloud/signals.rs` 中对流类型泛型的纯逻辑单元测试逐一覆盖。
与真实 Cloud（`ora-space/cloud` `34d8067`）的手工端到端验证记录在 desktop#614 的描述中。Cloud 服务端
一侧的信号、排空与保活证据见
[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md#watch-signals-only-accelerate-claiming)。

## Claiming Follows The Watch Stream State

- **风险**：流在线时仍按 `claim_interval_ms` 空转查询，流建立前已接受的工作因无人发信号而滞留，信号被丢弃后
  积压无人领取，或断流后不再领取。
- **前置状态**：云端模式 Controller 持有租约；假 Cloud 可入队工作、发送 `WorkAvailable`、以错误状态结束流，
  并记录全部调用。
- **触发**：`claim_interval_ms` 取远大于测试时长的值后发送 `WorkAvailable`；在 Controller 启动前入队工作；
  流在线且无信号时经过续约节拍；流以错误状态结束。
- **必须成立**：流建立后立即领取一次；`WorkAvailable` 立即触发领取；在线时领取节拍不领取，续约后兜底领取一次；
  流以错误结束后进入无流，按 `claim_interval_ms` 领取并在领取与续约节拍上重开流。
- **禁止结果**：在线时按 `claim_interval_ms` 领取、订阅前已接受的工作等到下一次续约才登记、断流后停止领取、
  未持有租约时打开流或领取。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 信号与流建立触发领取 | `tests/cloud.rs::signals_claim_without_waiting_for_the_claim_interval`（`claim_interval_ms` 为 1 小时，信号后 1 秒内登记，整段调用序列相等）与 `work_accepted_before_the_stream_opens_is_claimed_on_establishment` | Covered |
| 在线时只按信号与兜底领取 | `signals.rs::ticks_claim_by_stream_state` 与 `stream_events_follow_the_decision_table` 覆盖在线状态下领取节拍不领取、续约节拍兜底领取、`NodeAssignment` 与未知信号不领取；上一行两个测试的完整调用序列证明在线时没有周期领取；10 秒兜底没有端到端测试 | Partial |
| 断流回退与重开 | `tests/cloud.rs::a_broken_stream_falls_back_to_periodic_claims_and_reopens`：流以错误结束后，无信号的工作由周期领取登记，假 Cloud 重新接受后流重开并按信号登记 | Covered |
| 无租约不打开、不领取 | `signals.rs::opens_only_under_a_lease_without_a_live_stream` 与 `a_lost_lease_closes_only_a_live_stream`；`tests/cloud.rs::a_stale_stream_open_stops_claims_and_writes` 证明丢弃 epoch 后不再发出调用 | Covered |

决策依据：D1、D2、D6，不变量 2、3。

## Drain Pauses Claiming Until A Watch Is Reestablished

- **风险**：Cloud 关停期间持续发出注定失败的领取，排空时释放租约或打断 Node 会话，Cloud 恢复后 Controller
  不再领取，或 `Watch` 被明确拒绝时无限期停止领取。
- **前置状态**：云端模式 Controller 持有租约且流在线；假 Cloud 可发送 `Drain` 并正常结束流、以指定状态拒绝后续
  `Watch`、之后重新接受。
- **触发**：发送 `Drain` 后正常结束流；只正常结束流而不发送 `Drain`；排空期间以 `UNAVAILABLE` 拒绝重开；
  排空期间在队列中积压工作后重新接受 `Watch`；排空期间以 `UNIMPLEMENTED` 等其他错误拒绝重开；排空期间丢弃
  epoch。
- **必须成立**：收到 `Drain` 或正常结束后不领取、继续续约、不释放租约，Node 会话照常；`UNAVAILABLE`、超时与
  资格失效保持排空；重开成功后立即领取并登记积压；以其他错误拒绝时转入无流并恢复周期领取。
- **禁止结果**：排空期间发出 `ClaimWork` 或 `RecordDispatch`、排空时调用 `ReleaseLease`、Cloud 恢复后仍不领取、
  `Watch` 被明确拒绝后永久不领取。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 排空期间不领取、不释放租约 | `tests/cloud.rs::a_drain_pauses_claims_until_a_new_stream_opens`：`Drain` 后至少三次被拒的重开（每次对应一个节拍）之间没有其他调用，全程只在关停时释放一次租约；Node 会话照常只有代码路径保证，没有断言 | Partial |
| 恢复后立即领取积压 | 同一测试：假 Cloud 重新接受 `Watch` 后，排空期间入队的工作被登记 | Covered |
| 按打开结果决定去向 | `signals.rs::open_attempts_follow_the_decision_table` 与 `stream_events_follow_the_decision_table` 覆盖 D3 表每一行，以及 `Drain` 与正常结束进入同一状态；`fault.rs::opening_separates_unavailable_from_refusals` 与 `opening_without_headers_times_out_as_unavailable` 覆盖分类；`tests/cloud.rs::a_refused_stream_after_a_drain_resumes_periodic_claims` 覆盖排空后以 `UNIMPLEMENTED` 拒绝时恢复周期领取 | Covered |

决策依据：D3，不变量 3、4。

## One Claim Trigger Registers The Whole Backlog In Bounded Batches

- **风险**：信号被丢弃后积压只能按兜底节拍逐项排空，长队列推迟续约直到租约过期，或队头无法登记时反复告警。
- **前置状态**：假 Cloud 队列中有超过一批上限（16 项）的工作；另有一项按 Node 命令校验失败的工作。
- **触发**：只发送一个 `WorkAvailable`；让无法登记的工作位于队头。
- **必须成立**：一次触发连续领取并登记直到队列为空；满批后先处理关停与续约再继续；某一步失败即停止并等待下一次
  触发，同一队头只告警一次。
- **禁止结果**：每次触发只登记一项、满批时跳过续约、对同一队头重复告警或跳过它领取后续工作。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 连续登记到队列为空 | `tests/cloud.rs::one_signal_registers_a_backlog_beyond_one_batch`：入队 17 项、只发一个信号，全部按序登记 | Covered |
| 满批让出与失败即停 | 上一行的测试证明满批后继续领取，但不能区分"让出后继续"与"不分批"；队头失败即停且只告警一次没有直接测试 | Missing |

决策依据：D4，不变量 1、5。

## Watch Lives Only Under The Held Epoch

- **风险**：陈旧 Controller 在打开 `Watch` 被判资格失效后继续领取或写入，或关停时流悬挂、租约未释放。
- **前置状态**：云端模式 Controller；假 Cloud 可以带 `STALE_CONTROLLER` 的 `FAILED_PRECONDITION` 拒绝 `Watch`，
  并能观察流被取消。
- **触发**：打开 `Watch` 时被判陈旧；持有租约且流在线时关停 Controller。
- **必须成立**：打开时被判陈旧即丢弃 epoch、关闭流，之后在重新取得租约前不领取、不写入；关停时先关闭流再以当前
  epoch 释放租约。
- **禁止结果**：丢弃 epoch 后仍发出 `ClaimWork`、`RecordDispatch` 或其他写入；关停后假 Cloud 仍持有订阅；
  以旧 epoch 释放租约。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 资格失效后停止领取与写入 | `tests/cloud.rs::a_stale_stream_open_stops_claims_and_writes`：打开被判陈旧后，六个领取节拍内的完整调用序列只有获取租约与那次打开，没有登记；等待以节拍倍数表达，因为没有租约时循环不发出任何可等待的调用 | Covered |
| 关停关闭流并释放租约 | `tests/cloud.rs::shutdown_cancels_the_stream_and_releases_the_lease`：假 Cloud 观察到流被取消，最后一次调用是带正确 epoch 的 `ReleaseLease`；"先关流再释放"的先后没有断言 | Partial |

决策依据：D1，不变量 2；持久协调后续决策 D1。

## Idle Cloud Connections Stay Alive And Lost Ones Fail Over

- **风险**：经 NAT 或中间设备静默失联的连接让 Controller 一直以为流在线，或保活 PING 被 Cloud 以 `GOAWAY`
  断开整条连接。
- **前置状态**：Cloud 监听已放宽 PING 执行策略；Controller 的 Cloud 通道启用 30 秒 / 10 秒、空闲也发送的保活。
- **触发**：流在线且空闲超过 grpc-go 默认策略下的断开时间；对端停止回应 PING。
- **必须成立**：空闲连接不被 `GOAWAY`；对端不回应时连接关闭，流以错误结束并进入无流。
- **禁止结果**：保活触发 `too_many_pings`；失联连接上的流一直被当作在线。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空闲不被断开 | Cloud 侧由[监听保活用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md#controller-keepalive-pings-never-trigger-goaway)证明策略；Controller 侧只有 desktop#614 记录的手工验证（真实 Cloud 空闲约三分钟未断开），没有自动化测试 | Missing |
| 失联后回退 | 只有 desktop#614 记录的手工验证：冻结 Cloud 进程约 33 秒后连接被保活关闭；没有自动化测试。另观察到失联期间流错误要等续约与领取的超时结束后才被处理，状态暂时停留在在线 | Missing |

决策依据：D5，不变量 6。
