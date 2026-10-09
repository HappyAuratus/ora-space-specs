---
status: proposed
---

# 云端 Controller 派发 Agent 会话与 Revision 交付，按序中继 Thread 事件与命令，不自行决定运行阶段

云端持久模式的 Controller 从 Cloud 领取 IssueRun 放出的 Agent 会话与 Revision 交付工作项，派发给运行
Workspace 的 Node；把 Node 的 Thread 事件按序批量交给 Cloud 接管后再确认；把 Cloud 持久化的用户消息与
结束请求投递给 Node；为交付执行向 Cloud 申请上传授权并转交 Node。Controller 不判断会话何时结束、不决定
是否释放沙盒，这些都是 Cloud 的阶段转换；Controller 也不持有对象存储凭据。

本 ADR 处于 `proposed`，是 `controller/session` 的根决策（本目录范围：“Session 业务记录、生命周期和
Node 会话编排”；这里的 Session 指 Agent 会话，不是 Controller–Node 连接会话）。契约见
[gRPC 后续决策](../../cloud/controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)，
Node 侧消息见 [Node 协议后续决策](../../node/protocol/20260928-streamed-thread-events-and-session-commands.md)。
会话接线与 Revision 交付、授权转交均已形成待评审实现；本文仍保持 `proposed`，不以代码完成代替设计批准。

它复用 [Controller 驱动 Workspace 沙盒](../node-management/0-controller-drives-workspace-sandboxes.md) 维护的
“sandbox → Node 会话”映射与代报，不改变 operation 的推进方式。涉及 desktop `apps/ora-controller` 的云端
适配器；SQLite（Desktop 本机）模式不承担 Agent 会话，本文不涉及。

## 决策

### D1：会话与交付工作项沿用领取—登记—派发

- 领取循环在 `WorkAvailable` 信号或周期节拍上调用 `ClaimWork`，对会话与交付工作项：确认 `target` 的
  sandbox 在当前映射中且 Node 会话已握手（`node_id` 一致）→ `RecordDispatch` → 发送
  会话的 `ControlledStartAgentSession`（Revision 的 `DeliverRevision` 留待交付阶段）。会话要求 Node 声明 `agent_session`；
  派发前重新取得该执行的 runtime permit，登记事实不代替当前权限。Node-local operation ID 采用 Cloud 登记回包，
  不使用 IssueRun ID 猜测。登记与 quiesce 共用 sandbox gate，防止 idle 检查期间加入新责任。
  目标 Node 暂无会话时不登记，等待下一次领取；握手完成唤醒领取。登记后才发送，
  发送失败的执行由既有恢复流程（查询状态、重发原执行）处理。
- 插件执行在 operation 的 `plugin` 步骤中派发，方式与 clone 步骤相同（见
  [Controller plugin 根决策](../plugin/0-plugin-installs-as-node-executions.md)）。

### D2：Thread 事件按执行串行、按批接管

- 每个会话执行有一个中继队列：Node 送来的非终态事件按序号入队，Controller 以不超过 64 个事件或 100 ms
  为一批调用 `TakeOverThreadEvents`；编码载荷累计约 1 MiB 时提前提交以满足 gRPC 消息上限。
  同一执行同时至多一个在途批次，因此序号永远按顺序到达 Cloud。
- 批次成功后，对其中每个序号发送 `EventAck`；失败（`UNAVAILABLE`）时保留队列并退避重试，不确认；
  `CONFLICT` 时不确认并关闭连接，重连触发 Node 从最小未确认序号重放。仅清空内存队列不会触发 Node 重放，
  且可能让已满的窗口永久等待。回复丢失沿用同一 submission ID 有限重试；耗尽后也由重连重放恢复。
- 终态事件在该执行队列清空后经 `TakeOverNodeEvent` 接管，再确认。Completed 查询不携带事件序号，不能
  据此调用 `RecordQueriedResult` 提前结算会话；等待真实终态 envelope 及此前 Thread 事件重放。
- 不同执行的中继互不阻塞，也不被 operation 的串行推进阻塞：中继运行在各 Node 连接会话自己的任务中。

为什么不在 Controller 内存中缓存后先确认：协议根决策规定 Controller 的内存去重不构成确认依据；
Controller 崩溃时未提交的事件必须还能从 Node 重放。

### D3：命令由 Cloud 领取、Node 受理后登记

- 在 `ThreadCommandAvailable` 信号或周期节拍上调用 `ClaimThreadCommands`，把命令按运行内顺序发送给目标
  Node；收到与已发送命令身份一致的 `SessionCommandAccepted{command_id}` 或
  `SessionCommandRejected{reason: session_ended}` 后调用 `RecordThreadCommandDelivered`，与 Node 协议 D4 对齐。
  每个运行只投递队首，登记投递成功后才推进下一条；发送成功不作为投递证据。
- 命令目标必须匹配运行、执行、Node 和当前沙盒 runtime scope；先查询确认 Node 已知该执行，避免开始命令
  尚未落盘就发送用户命令。目标 Node 无会话、或会话执行尚未登记时跳过该运行的命令，保持顺序：同一运行的后续命令不得先于前面
  未投递的命令发送。
- 重连或重启后未登记的命令会再次被领取和发送；Node 按 `command_id` 去重。

### D4：交付执行的上传授权即取即用

交付的派发与会话相同：领取要求 Node 声明 `revision_delivery`，以 `RecordDispatch` 固定 Cloud 给出的输入
（Controller 只补上目标 `node_id`），再在拿到精确运行许可后发送 `ControlledDeliverRevision{binding, command}`。

上传授权只按 Node 的请求申请：

- 派发时**不**申请授权。此时 Node 尚未固定对象字节，申请到的授权无法绑定校验和；Node 在固定内容后、
  每个仍需上传的对象没有有效授权时发送 `UploadGrantNeeded{node_id, checksums}`。
- Controller 把 Node 给出的 `checksums` 原样交给 `GrantRevisionUpload`，并以 `UploadGrant` 把授权（请求头与
  过期时间逐字不变，只含所请求的对象键）发给 Node；从不在没有校验和时申请授权。
- 新连接首次看到未结束的交付时，若本进程记得该执行最近一次的校验和就重新申请；Controller 重启后不记得，
  由 Node 在每个新连接上重发 `UploadGrantNeeded`（Node 协议 D5）。校验和只保存在进程内存，不是凭据。
- `Unavailable` 或租约丢失时稍后重试；Cloud 回答 `Conflict`/`NotFound`（交付已有结果或运行已不在交付阶段）
  时不再申请，**不**把它变成交付失败。
- 授权只在内存中转交，不写日志、不写任何持久存储，也不以调试格式输出。

交付的终态只从 Node 的有序 `RevisionResult` 事件接管：Controller 先核对其基础提交、ref 与对象键和派发
输入一致，经 `TakeOverNodeEvent` 交给 Cloud，提交后才 `EventAck`。状态查询得到的 `Completed` 不登记：Cloud
要求交付结果带接管回执，Node 会重放事件直到确认。Controller 关闭、租约丢失或重启都不写 `RevisionFailed`；
已登记的交付经状态查询与重发继续。

### D5：释放由 Cloud 决定，Controller 只提供 idle 证据

Controller 不因会话结束或交付完成而终止沙盒。释放是 Cloud 在 Revision 登记后创建的 `delete_workspace`，
Controller 按既有步骤推进；其 quiesce 的 idle 证据来自派发账本，账本包含本文的会话、交付与插件执行——
任何一个尚无终态，idle 就是 `false`。

## 不变量

1. Controller 只派发 Cloud 放出、且已登记的会话与交付执行。
2. 同一执行的 Thread 事件按序号顺序提交给 Cloud，且每个确认晚于对应提交。
3. 同一运行的 Thread 命令按 Cloud 的创建顺序投递。
4. 上传授权不被 Controller 持久化或记录。
5. Controller 不决定运行阶段，也不在 idle 证据之外影响沙盒释放。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Controller 在会话结束后直接派发交付 | 少一次 Cloud 往返；但“何时交付”变成 Controller 的内存决策，重启后无法恢复，也绕过 Cloud 对取消与失败的判断 |
| Controller 按 Agent 的 `TurnEnded` 判断会话结束 | Controller 需要理解会话记录与空闲窗口；这是 Thread 的业务规则，由 Cloud 按数据库时间判断 |
| 所有中继并入 operation 串行循环 | 实现简单；一个慢 operation 会冻结所有 Thread |

## 本决策未解决的问题

- Desktop 本机（SQLite 持久）模式下的 Agent 会话。
- 多 Controller worker 下按运行分区中继。

## 落地顺序

1. 领取与派发会话/交付工作项；中继队列与批量接管；以假 Cloud 与假 Node 的 Rust 测试覆盖顺序、冲突不确认、
   终态晚于事件。
2. 命令领取与投递；上传授权转交。
3. 经 cluster 与 Cloud 联调。
