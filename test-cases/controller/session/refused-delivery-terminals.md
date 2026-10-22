# Controller 局部隔离被拒绝的原交付终态

依据[终态拒绝决策](../../../decisions/controller/session/20261022-isolate-refused-delivery-terminals.md)
（下称“拒绝决策”）与[迟到事实接管](../../../decisions/cloud/issue-run/20261022-late-delivery-fact-takeover.md) D3。
新增义务全为 **Missing**，由实现及落地运行补直接证据；
[既有中继用例](agent-session-relay.md)的 Covered 不自动覆盖这些新增拒绝行为。

## Exact Refusal Of A Registered Delivery Is Isolated Without Closing The Node Session

- 风险：一个业务已退休的终态失败使整 Node 连接反复重建，其他执行及 quiesce 一起停滞。
- 前置状态：Node 已完成握手，Cloud 已登记 delivery；原身份、Node-local operation、producer 及冻结
  spec/result 均匹配，Node 保留原有序终态；同连接还有另一个正常执行。
- 触发：原 terminal takeover 返回 Aborted/CONFLICT、原 message 为 dispatch_conflict。
- 必须成立：只有该 execution unresolved，无 EventAck、无新的 delivery start/permit/grant，Node
  identity 和连接存续，heartbeat 与 Connected reports 正常，另一执行仍能接管和 ACK；原事件留在 outbox。
  首次日志保留原 code/message 和执行身份，不把它改成只有通用 Conflict 的警告。
- 禁止结果：清 Node identity、由此拒绝主动断线重连；吞其他错误；伪造 connected、receipt 或成功。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 精确拒绝仅影响原执行 | 生产 Controller adapter 与 fake Cloud/Node 的有序终态拒绝，断言连接、identity、无 ACK/重派与另一执行完成 | Missing |
| 报告及原错误保留 | 同一场景的 heartbeat、Connected reports 和结构化日志断言，日志包含原 gRPC code、分类与 message | Missing |

- 决策依据：拒绝决策 D1、D2、D4，不变量 1、2、5。

## Duplicate Pending And Completed Status Cannot Clear A Refused Terminal

- 风险：刚隔离的终态被下一次 Completed/Unknown 查询误清，或者每个 tick 重交/重派并制造警告。
- 前置状态：该 delivery 已因精确拒绝处于 unresolved，原 Node 会话仍存活。
- 触发：同连接内重复 terminal、重复 pending、Completed/Unknown status；其他执行成功 receipt；quiesce 查询。
- 必须成立：拒绝状态仍优先，不清 unresolved、不再次 takeover、不发 ACK、不重派或申请新授权；
  相同重复状态不每 tick 警告。该执行仍使 quiesce 非 idle，其他执行的成功不解除本执行。
- 禁止结果：查询结果被当作原事件回执；把拒绝 execution 仅塞进“已重派”集合后又走普通 Answered；
  以等待时间或 Run done 猜已解决。
- 验证义务与证据：**Missing** — 需同连接重复事件/状态的 Controller 测试，断言精确调用次数、
  unresolved 保持、无 ACK/重派、新授权及 quiesce busy；同时证明另一执行可继续。
- 决策依据：拒绝决策 D2，不变量 2、3。

## Real Reconnect Replays The Original Fact And Only Its Success Receipt Clears Refusal

- 风险：拒绝状态永久失去恢复机会，或重连时用新执行/Completed 查询绕过原事件回执。
- 前置状态：原 delivery 因拒绝 unresolved，Node outbox 保留原 sequence/event；Cloud 后续具备真实接管能力。
- 触发：分别真实 transport 重连、Controller 进程重启；原事件再被拒绝或成功接管；回复丢失后重放。
- 必须成立：重交同 execution/work/resource/sequence/event，不重派交付命令。仍被拒绝则隔离该
  execution 且不触发新一轮主动重连；成功响应只在原事件持久接管后产生，随后才清对应 unresolved、
  ACK 并排空原 outbox。登记 Revision 与 receipt-only 两种成功都成立，Run 的业务交付仍为 failed。
- 禁止结果：重启写 RevisionFailed；新 execution 代替旧事件；不相关 receipt、Completed 或内存接收
  清状态；把 ACK 解释成业务重新成功。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 两类真实恢复入口 | Controller transport 重连与进程重启分别保留原事实身份，拒绝/成功分支都有直接断言 | Missing |
| 清除晚于成功接管 | 阻塞持久提交及丢失回复的测试，提交前无 ACK/清除，提交后原事件重放可恢复；含 receipt-only | Missing |
| 真实栈恢复边界 | 真 Cloud/Node 的原 outbox、receipt、ACK 和连接证据；另见 Cloud 的释放整链义务 | Missing |

- 决策依据：拒绝决策 D3，不变量 4、5；迟到事实接管 D3。

## Other Conflicts And Invalid Identities Keep Their Existing Fault Semantics

- 风险：为修复重连而全局吞 Conflict，接管越权执行或保留已经失效的协调资格。
- 前置状态：分别为未知 execution、错误 Node/operation/local operation/spec/producer，普通 dispatch
  注册冲突，以及其他真实 result/receipt/submission 冲突、Thread 批次冲突和 stale controller lease。
- 触发：在相应生产接口返回冲突或不匹配；另注入真实断线、协议/worker 错误及 Cloud 暂不可达。
- 必须成立：只有完整匹配原登记的 delivery terminal 的精确 dispatch_conflict 被局部隔离；其余继续
  原有拒绝、Thread 重放、连接失败、暂失败重试或租约恢复规则，不发伪造 ACK，不保留无效权限。
- 禁止结果：`Conflict => continue` 的全局处理；把 FailedPrecondition/stale_controller 当作原终态隔离；
  为维持 connected 隐藏真实 transport/协议故障。
- 验证义务与证据：**Missing** — 需错误分类与对应接口的 Controller 负例，保留原 gRPC code/message，
  并证明非目标错误没有进入新局部隔离分支；Thread 批次已有规则不弱化。
- 决策依据：拒绝决策 D1、D4，不变量 1、5。
