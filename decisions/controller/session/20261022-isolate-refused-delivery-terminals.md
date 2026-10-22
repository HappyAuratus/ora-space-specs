---
status: approved
---

# 原交付终态遭精确 dispatch_conflict 拒绝时，只隔离该执行并保留 Node 连接

已登记交付的原身份与输入均匹配，但其终态接管收到精确 `dispatch_conflict` 时，Controller 只把该
execution 标为 unresolved：不 ACK、不重派，保留已握手 Node 连接和连接报告，并保留 Cloud 原错误码。
真实重连或重启可以重新提交原事实，只有成功的原事件接管回执才能解除。其他 Conflict 的处理不变。

本文为 `approved`，依据 [owner 决定](https://github.com/ora-space/circus/issues/407#issuecomment-6335015004)；新拒绝隔离行为尚待产品实现与直接验证，不把现有中继测试当作已覆盖。本决策不改
Node 协议、Cloud 业务阶段或释放授权，不新增本地持久化状态机、超时成功或新的 disposition。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [Controller session 根决策](0-controller-relays-agent-sessions.md) D1/D4：原身份登记与许可派发，终态经 takeover 后确认 | 继承原身份登记与许可派发；修改根 D4 的终态接管，增加精确拒绝的 execution 内局部隔离，不把拒绝升级为整连接故障（D1）。 |
| 根 D2：Thread 批次 Conflict 关闭连接以触发重放 | 继承，不变；本决策仅处理已经匹配原登记信息的 delivery terminal，不推广到 Thread 批次。 |
| 根 D4/D5：Completed 查询不能代替终态 receipt，未结束执行使 quiesce busy | 继承并明确：查询与重复状态也不能清除被拒绝的 unresolved（D2）。 |
| [迟到事实接管](../../cloud/issue-run/20261022-late-delivery-fact-takeover.md) D1–D3 | 继承原事件持久接管及 ACK 与业务结论的区别；成功 receipt 处理与局部拒绝处理分开。 |

## 决策

### D1：只识别已登记原交付终态的精确拒绝

必须先确认该 `deliver_revision` 已在 Cloud 登记，原 execution、operation、Node、Node-local operation
与冻结输入均匹配，终态生产者与声明也匹配；随后只有这次 `TakeOverNodeEvent` 返回
`gRPC Aborted`、`ErrorDetail.CONFLICT` 且原 status message 为 `dispatch_conflict`，才能进入局部隔离。
仅看到通用 Conflict 分类不能证明属于此情形。

隔离只影响这个 execution，不能因该拒绝退出中继任务、清除已握手 Node identity 或主动重建连接。
同 Node 的其他执行、Thread 中继、heartbeat 与正常连接报告继续。不能为了保持 connected 伪造报告，
真实 transport 状态仍按既有机制上报。

身份或原输入不匹配、未知执行、RecordDispatch 冲突、其他 result/receipt/submission 冲突、Thread
批次冲突、非法协议和协调资格失效，继续按各自既有规则处理。尤其不能全局吞掉 Conflict，也不能
把 stale controller lease 当作原终态拒绝而保留无效协调资格。

### D2：拒绝保持 unresolved，不确认、不重派、不被查询清除

对这个 execution 保留明确的“终态接管被拒绝”状态和原错误信息。当前连接内的重复终态、pending
列表、Completed/Unknown 查询都不能清掉它，不能反复重新 takeover，也不能进入 Unknown 的命令重派
或新的上传授权路径。Node 继续持有原未确认终态；Controller 不伪造接收回执或 RevisionFailed。

没有原事件成功回执时，该执行仍属于 unresolved，quiesce 不能把它当作 idle。单纯的状态查询、计时器、
轮询成功或 Run 已终态都不能解除拒绝状态，也不能把其他 execution 的 receipt 用于本执行。

被隔离的 execution 仍使 quiesce busy；只有 Cloud 按迟到事实接管决策 D1 接收原迟到事实后，释放流程
才能继续完成。只部署 Controller 一侧的改动会停止重连循环，但本故障中的 Run 仍停在 `releasing`。

### D3：真实重连或重启重交原事实，成功回执才解除

真实 transport 重连、Controller 重启后允许重新提交原 execution 的原有序事件。重新提交的是原事实，
不是再次发送交付命令，也不生成新 execution 或再次执行交付。仍遭精确拒绝时再次保持局部 unresolved，
不能由这个拒绝自身制造下一轮连接重建。

只有本次原 execution/sequence/event 已持久接管的成功回执，才能完成确认、清除对应 unresolved，
允许 Node 按协议移除原 outbox 事件。无论 Cloud 登记原 Revision，还是因后创建 Run 只接收声明和
receipt，都使用相同成功回执。ACK 只表示事实已接管，不改变 Run 的失败交付结论。

### D4：保留 Cloud 原错误，其他 session 故障仍可见

首次隔离日志保留原 operation/execution/Node 身份、sequence、gRPC code、ErrorDetail 分类及原
status message，并说明本地采取 unresolved/no ACK/no redispatch。当前 ErrorDetail 的分类与原
message 不混写成另一个通用原因；同连接内重复状态不持续制造相同警告。不记录事件正文或凭据。

真实断线、协议错误、握手身份/能力不符、其他真实冲突、队列故障或 worker 异常仍保持既有 session
错误路径；日志保留原因，不能为了避免重连而掩盖它们。Cloud 临时不可达、重试与租约恢复继续各自的
既有语义，本决策不扩大到任意接管失败。

## 不变量

1. 局部隔离仅由已匹配原登记信息的交付终态精确拒绝触发。
2. 拒绝不产生 ACK、重新派发或整会话重建，其他执行不被它阻塞。
3. Completed/Unknown 和重复 pending 不能清除被拒绝的 unresolved。
4. 真实重连/重启仅重交原事实，原事件成功 receipt 是解除与 ACK 的唯一依据。
5. ACK 不重新结算业务；其他 Conflict、身份错误及租约失效不被吞掉。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 所有 Conflict 都继续连接 | 能停止部分重连，却掩盖原身份、输入和收据冲突。 |
| 拒绝后先 ACK 让 Node 清队列 | 直接丢失尚未被 Cloud 接管的原事实。 |
| 查询到 Completed 就清 unresolved | 查询不能证明本次 sequence/event 已接管。 |
| 拒绝后重新派发交付 | 把事实恢复变成重复执行，且可能绕过已关闭资格。 |
| 无条件断线重试 | 一个退休执行可持续破坏整 Node 连接，quiesce 及其他执行也受影响。 |

## 落地与验证

新增行为由[终态拒绝核心用例](../../../test-cases/controller/session/refused-delivery-terminals.md)登记，
当前证据全部为 `Missing`，由产品实现与落地运行补齐。真实释放至 done 和 container/volume 回收由
[迟到交付与释放恢复](../../../test-cases/cloud/issue-run/late-delivery-fact-takeover.md)的跨仓用例承担，
不以单仓 fake 的连接存续结果替代。
