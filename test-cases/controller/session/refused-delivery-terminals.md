# Controller 局部隔离被拒绝的原交付终态

依据[终态拒绝决策](../../../decisions/controller/session/20261022-isolate-refused-delivery-terminals.md)
（下称“拒绝决策”）与[迟到事实接管](../../../decisions/cloud/issue-run/20261022-late-delivery-fact-takeover.md) D3。
本文件按每项义务分别登记证据，单仓 fake 的成功不自动覆盖真实 Cloud/Node 或物理资源释放。

固定证据基线为 [desktop#726](https://github.com/ora-space/desktop/pull/726) head
`13de5a3567d0f2166a4ab27d0f5c7fa05626bc9d` 和 [Cloud#142](https://github.com/ora-space/cloud/pull/142) head
`818226b49b8a97779f41d6191e69f6fc8518b0fb`。下文 desktop 测试名均位于
[`apps/ora-controller/tests/workspaces/deliveries/refusals.rs`](https://github.com/ora-space/desktop/blob/13de5a3567d0f2166a4ab27d0f5c7fa05626bc9d/apps/ora-controller/tests/workspaces/deliveries/refusals.rs)，
使用生产 Controller adapter、fake Cloud gRPC 与 WebSocket Node；错误分类单测位于
[`apps/ora-controller/src/cloud/fault.rs`](https://github.com/ora-space/desktop/blob/13de5a3567d0f2166a4ab27d0f5c7fa05626bc9d/apps/ora-controller/src/cloud/fault.rs)。

## Exact Refusal Of A Registered Delivery Is Isolated Without Closing The Node Session

- 风险：一个业务已退休的终态失败使整 Node 连接反复重建，其他执行及 quiesce 一起停滞。
- 前置状态：Node 已完成握手，Cloud 已登记 delivery；原身份、Node-local operation、producer 及冻结
  spec/result 均匹配，Node 保留原有序终态；同连接还有另一个正常执行。
- 触发：原 terminal takeover 返回 gRPC `Aborted`、typed `ErrorDetail` 分类 `CONFLICT`，且原 Cloud status
  message 精确为 `dispatch_conflict`。这不是新的 reason 或 disposition。
- 必须成立：只有该 execution unresolved，无 EventAck、无新的 delivery start/permit/grant，Node
  identity 和连接存续，heartbeat 与 Connected reports 正常，另一执行仍能接管和 ACK；原事件留在 outbox。
  首次日志保留原 code/message 和执行身份，不把它改成只有通用 Conflict 的警告。
- 禁止结果：清 Node identity、由此拒绝主动断线重连；吞其他错误；伪造 connected、receipt 或成功。
- 验证义务与证据：

| 义务 | 直接证据及边界 | 状态 |
| --- | --- | --- |
| 精确三元组分类 | `cloud::fault::tests::terminal_dispatch_refusal_requires_the_exact_cloud_verdict`：code、typed detail、message 必须全部匹配；不接受缺失/错误 detail 或相似 message | Covered |
| 精确拒绝仅影响原执行 | `deliveries::refusals::refused_terminal_isolated_on_live_session`：无 SessionEnded/disconnected、无 ACK/receipt、outbox 保留、仅一次启动；同连接另一执行取得 receipt/ACK | Covered |
| 心跳及连接报告仍真实前进 | 同测试跨越 1s I/O deadline，Connected report version 增加且 Node record id/identity 不变 | Covered |
| 原错误与执行身份保留，重复不刷警告 | 同测试断言唯一警告的 `grpc_code=Aborted`、原 detail、`cloud_message=dispatch_conflict`、operation/execution/Node/incarnation/workspace/sequence，以及本地 disposition；日志不含 history 路径 | Covered |

- 决策依据：拒绝决策 D1、D2、D4，不变量 1、2、5。精确 message 的跨仓依赖见拒绝决策“与批准文本的差异及原因”。

## Duplicate Pending And Completed Status Cannot Clear A Refused Terminal

- 风险：刚隔离的终态被下一次 Completed/Unknown 查询误清，或者每个 tick 重交/重派并制造警告。
- 前置状态：该 delivery 已因精确拒绝处于 unresolved，原 Node 会话仍存活。
- 触发：同连接内重复 terminal、重复 pending、Completed/Unknown status；其他执行成功 receipt；quiesce 查询。
- 必须成立：拒绝状态仍优先，不清 unresolved、不再次 takeover、不发 ACK、不重派或申请新授权；
  相同重复状态不每 tick 警告。该执行仍使 quiesce 非 idle，其他执行的成功不解除本执行。
- 禁止结果：查询结果被当作原事件回执；把拒绝 execution 仅塞进“已重派”集合后又走普通 Answered；
  以等待时间或 Run done 猜已解决。
- 验证义务与证据：

| 义务 | 直接证据及边界 | 状态 |
| --- | --- | --- |
| 重复事件/Completed/Unknown 不重交、不 ACK、不重派、不申请 grant | `deliveries::refusals::refused_terminal_isolated_on_live_session` 注入相同 terminal 与两个 status，断言 takeover 总数为 1、启动总数为 1、grant 数不变、无 GrantIssued/GrantRefused、无 ACK | Covered |
| pending 查询不能掩盖 unresolved；其他 receipt 不解除本执行 | 同测试让另一执行 ACK，再隐藏原执行 pending，完成 quiesce 轮次并断言 Idle(false)、原 outbox/拒绝状态仍保留 | Covered |
| 重复提示不持续制造同一警告 | 同测试直接断言拒绝警告仅 1 条 | Covered |

- 决策依据：拒绝决策 D2，不变量 2、3。

## Real Reconnect Replays The Original Fact And Only Its Success Receipt Clears Refusal

- 风险：拒绝状态永久失去恢复机会，或重连时用新执行/Completed 查询绕过原事件回执。
- 前置状态：原 delivery 因拒绝 unresolved，Node outbox 保留原 sequence/event；Cloud 后续具备真实接管能力。
- 触发：真实 transport 重连或 Controller 重启；原事件再被拒绝或成功接管；回复丢失后重放。
- 必须成立：重交同 execution/work/resource/sequence/event，不重派交付命令。仍被拒绝则隔离该
  execution 且不触发新一轮主动重连；成功响应只在原事件持久接管后产生，随后才清对应 unresolved、
  ACK 并排空原 outbox。登记 Revision 与 receipt-only 两种成功都成立，Run 的业务交付仍为 failed。
- 禁止结果：重启写 RevisionFailed；新 execution 代替旧事件；不相关 receipt、Completed 或内存接收
  清状态；把 ACK 解释成业务重新成功。
- 验证义务与证据：

| 义务 | 直接证据及边界 | 状态 |
| --- | --- | --- |
| transport 重连重交原事实，仍拒绝则再次局部隔离 | `deliveries::refusals::reconnect_replays_refused_terminal_before_ack` 比较完整原请求（仅允许新 submission ID），重连前 Unknown 不重派，第二次拒绝仍无 ACK/receipt且连接存活 | Covered |
| Controller 重启重新发现原执行，重放前仍 busy，再次拒绝保持隔离 | `deliveries::refusals::restart_replays_refused_terminal_before_ack`：暂停 terminal 后重启，pending 存在时 quiesce 为 busy，无新启动；重放同一原事实并再次拒绝 | Covered |
| 重启后首次重放直接成功并清除拒绝 | 上一测试实际为“重启→再次拒绝→之后 transport 重连成功”，没有直接断言重启后的首次成功分支；共用 helper 不能填补该缺口 | Missing |
| 成功回执后才 ACK/清 outbox，含 receipt-only 与回复丢失 | 上述两个测试分别运行 result+receipt、receipt-only；成功发生于后续 transport 重连，断言同 submission 丢回复重试、receipt 精确匹配、RevisionTaken 先于 ACK、outbox 清空及随后 Idle(true) | Covered |
| 接管等待期间不提前清除/ACK | `deliveries::refusals::blocked_replay_keeps_refused_terminal_busy_until_receipt` 隐去 pending 并阻塞 fake Cloud 提交，断言 busy、无 receipt/ACK、outbox 保留；解阻 receipt-only 成功后 ACK、清 outbox及 idle | Covered |
| 真实 Cloud 提交到 Controller ACK/outbox 的连续边界 | [Cloud `integration/late_delivery_fact_test.go`](https://github.com/ora-space/cloud/blob/818226b49b8a97779f41d6191e69f6fc8518b0fb/integration/late_delivery_fact_test.go) 的 `TestLateDeliveryFactAtomicRollbackOverGRPC` 直接证明服务端 result+receipt 与 later-Run receipt-only 写入失败回滚、无成功响应及恢复后重放；真实 Cloud 提交到 Controller ACK/outbox 的跨仓连续关系仍缺直接证据 | Partial |
| 重启所需原 pending 的跨仓条件 | desktop 重启测试使用 fake pending；[Cloud `pendingExecutions`](https://github.com/ora-space/cloud/blob/818226b49b8a97779f41d6191e69f6fc8518b0fb/internal/core/node_execution.go#L221-L230) 仍查询无 result/未 force-stop 原执行，不按 Run phase 过滤。这是固定源码依据，不是真 Cloud 重启演练 | Partial |
| 真实栈拒绝窗口及恢复 | 仍需真实 Cloud/Node 连续证明“精确拒绝后连接存活、无 ACK/重派→真实重连或 Controller 重启→原 receipt 成功后清除”。现有作者运行未进入拒绝隔离窗口，亦未重启 Controller | Missing |
| 真实 Node 新 incarnation 保持原事实恢复 | [reviewer 明确保留的缺口](https://github.com/ora-space/circus/issues/407#issuecomment-6342996936)；Cloud simulator 的新 incarnation 分支不能代替真实 Node/container 重启 | Missing |

作者 [wanglongan587 的真实栈记录](https://github.com/ora-space/circus/issues/407#issuecomment-6341728665)
使用上述两个固定 head：暂停 GitServer，约 90s 达到 D5，约 240s 解暂停，约 256s 到 done，报告恢复约 10s；
WorkspaceDeleted、container=0、volume=0，stop 对照约 102s done。证据 JSON 位于作者机器
`ora-space/cluster/.acceptance/407/407-late-terminal-818226b4.json` 与 `ora-space/cluster/.acceptance/407/407-stop-818226b4.json`；
此处引用作者评论及其产物位置，未在本机读取这些 JSON。该运行在首次迟到接管即成功，证明正常迟到恢复和释放，
不能证明本用例拒绝窗口；释放证据详见[Cloud 用例](../../cloud/issue-run/late-delivery-fact-takeover.md)。

- 决策依据：拒绝决策 D3，不变量 4、5；迟到事实接管 D3。

## Other Conflicts And Invalid Identities Keep Their Existing Fault Semantics

- 风险：为修复重连而全局吞 Conflict，接管越权执行或保留已经失效的协调资格。
- 前置状态：未知 execution、错误 Node/operation/local operation/spec/producer，普通 dispatch 注册冲突，
  其他真实 result/receipt/submission 冲突、Thread 批次冲突和 stale controller lease。
- 触发：相应生产接口返回冲突或不匹配；真实断线、协议/worker 错误及 Cloud 暂不可达。
- 必须成立：只有完整匹配原登记的 delivery terminal 的精确 dispatch_conflict 被局部隔离；其余继续
  原有拒绝、Thread 重放、连接失败、暂失败重试或租约恢复规则，不发伪造 ACK，不保留无效权限。
- 禁止结果：`Conflict => continue` 的全局处理；把 FailedPrecondition/stale_controller 当作原终态隔离；
  为维持 connected 隐藏真实 transport/协议故障。
- 验证义务与证据：

| 义务 | 直接证据及边界 | 状态 |
| --- | --- | --- |
| 非精确拒绝、其他 result/receipt Conflict、Unavailable 仍使 session 失败且保留原码/message | `deliveries::refusals::non_refusal_errors_still_close_session`：缺 detail、错误 code/detail、尾空格、result_conflict、receipt_conflict 等负例直接断言 SessionEnded、无 receipt/ACK与原错误日志 | Covered |
| stale_controller 不保留无效协调资格 | 同测试断言 FailedPrecondition 后 session 结束，runtime shutdown 不调用旧 epoch 的 ReleaseLease | Covered |
| 错误 Node/operation/spec/未知 execution 及拒绝后改内容/producer 不进入隔离 | 同测试断言 RPC 前拒绝或原 takeover 次数不变，无错误 ACK | Covered |
| local operation 与 submission 冲突的全部 Controller 负例 | 上述直接矩阵不逐项覆盖独立 local operation/submission 变体；Cloud 冲突测试只证明服务端围栏 | Partial |
| RecordDispatch 同名冲突继续排队/注册处理 | `deliveries::refusals::ordinary_dispatch_conflict_keeps_work_queued`：冲突期间无注册/接管，连接保留；解除故障后相同输入正常注册并 ACK | Covered |
| 控制队列故障不被新隔离吞掉 | `deliveries::refusals::control_queue_overflow_still_closes_session`：阻塞另一接管、超过既有队列界限，断言 overflow 原因、SessionEnded、无 receipt/ACK | Covered |
| Thread 批次冲突保持原重放与确认顺序 | [desktop `apps/ora-controller/tests/workspaces/agents.rs`](https://github.com/ora-space/desktop/blob/13de5a3567d0f2166a4ab27d0f5c7fa05626bc9d/apps/ora-controller/tests/workspaces/agents.rs) 的 `agents::batch_failures_and_lost_replies_preserve_exact_events`；既有中继义务见[会话中继用例](agent-session-relay.md) | Covered |

- 决策依据：拒绝决策 D1、D4，不变量 1、5。
