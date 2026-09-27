---
status: implemented
---

# Controller 消费 Watch 信号流：信号驱动领取、断流退回周期领取、`Drain` 暂停领取直到新流建立

> 待评审关联：[运行时控制同步交付提案](../../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)拟增加服务认证、派发前许可检查与旧执行对账；目前为 `proposed`，不改变本文 Watch 语义。

云端持久模式的 Controller 在持有 Cloud 租约期间维持一条 `ControlSignalService.Watch` 服务端流，并把它
作为领取的主要触发：流建立时与每个 `WorkAvailable` 到达时立即领取，另在每次租约续期后兜底领取一次；
流不可用时退回按 `claim_interval_ms` 周期领取；收到 `Drain` 或流正常结束时暂停领取，直到新流建立。
每次领取连续登记已接受的工作直到队列为空。Controller 到 Cloud 的连接启用 HTTP/2 保活，使失联的流被
及时发现并退回周期领取。信号仍只是加速器：工作归属继续由 `RecordDispatch` 事务决定，信号丢失只影响
延迟。

本决策处于 `implemented` 状态，由 ora-space/desktop#614（主干合并提交 `03d521b`）落地，Cloud 侧配套由
ora-space/cloud#29 落地；验证证据登记在
[Watch 信号流核心用例](../../../test-cases/controller/api-boundary/watch-signal-claiming.md)。它是
[Controller 契约方向后续决策](20260922-cloud-owned-contract-and-controller-dial-out.md)的后续决策，修改
其 D2 中"当前 Controller 不消费该流，只做周期领取"的约定，以及
[持久协调双适配器](../persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md) D2 中
`claim_interval_ms` 的含义。它只涉及 `ora-controller` Cloud 适配器的长驻任务（`CloudStore::serve`）与
Cloud 连接参数；契约消息、Cloud 的信号语义
（[Cloud 根决策](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md) D5）、本机
SQLite 模式、Node 会话与部署配置字段都不变。Cloud 监听端为保活所需的配套修改见
[Controller 控制监听允许 HTTP/2 保活](../../cloud/controller-integration/20260924-control-listener-keepalive.md)。
多 worker、`NodeAssignment` 与多实例下 `Drain` 的让位语义仍留给 scale 决策。

## 背景

此前 Controller 每 `claim_interval_ms`（minicloud 生成 500 ms）调用一次 `ClaimWork`：空闲时每秒两次
PostgreSQL 查询，派发延迟的下限就是这个间隔。Cloud 已按根决策 D5 提供 `Watch`：订阅登记后才发送
响应头；接受事务提交后发送带 `operation_id` 的 `WorkAvailable`；每个订阅者缓冲 16 条、满时丢弃；关停
前发送 `Drain` 并正常结束流，排空期间新的 `Watch` 为 `UNAVAILABLE`；打开时只校验一次 epoch。

前序决策把消费 `Watch` 与 `Drain` 的排空语义一起留给 scale 决策。本文的判断是二者可以拆开：单实例下
`Drain` 唯一的发送时机是 Cloud 实例关停（`cmd/server` 收到 SIGTERM 后先 `Drain` 再 `GracefulStop`），
Controller 对它的义务"不再领取、不取消用户任务、不销毁沙盒"已由 Cloud 根决策 D5 规定，只缺"不再领取
到何时为止"；这个结束条件不依赖多 worker 设计。多实例相关部分仍保留在 scale 决策。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| dial-out D1：契约唯一源在 Cloud 仓库，desktop 以 submodule 锁定 commit 生成客户端 | 继承，不变；本文不需要新的契约消息，`Watch` 客户端已在生成物中 |
| dial-out D2：Controller 只拨出，Cloud 部署不打开监听 | 继承，不变 |
| dial-out D2：每 10 秒续期 30 秒的租约，按 `claim_interval_ms` 周期领取 | 修改：续期不变；领取节拍改由流状态决定（D2），`claim_interval_ms` 只在无流时生效 |
| dial-out D2：当前 Controller 不消费 `Watch`，只做周期领取 | 修改：Controller 在持有租约期间维持 `Watch` 并按信号领取（D1–D4） |
| dial-out D2、不变量 3：信号只是加速器，不构成归属，丢失只影响延迟 | 继承，不变 |
| dial-out D3：Cloud 裁决只在 `cloud/fault` 一处映射为接口错误分类 | 继承；打开 `Watch` 的失败按读取归类（D1） |
| dial-out D4：Controller 只以 `x-ora-controller-id` metadata 声明身份，不出示凭据 | 继承，不变；`Watch` 同样携带 |
| persistence D1：`serve` 是适配器与权威的长驻协调，停止时释放租约 | 继承；`serve` 额外持有流，停止时先关闭流再释放租约 |
| persistence D2：`{ kind: cloud, endpoint, claim_interval_ms }` | 字段与校验不变；修改含义：`claim_interval_ms` 是无流时的回退领取间隔 |
| Cloud 根决策 D5：`Drain` 后 Controller 不再领取，不取消用户任务、不销毁沙盒 | 继承；本文 D3 给出单实例下"不再领取"的结束条件 |

## 决策

### D1：持有租约期间维持一条 Watch 流，收到响应头才算建立

- Controller 持有租约且没有在线的流时，在每个续约节拍与领取节拍上以当前 epoch 打开 `Watch`，请求
  携带 `x-ora-controller-id`。
- 收到响应头才算流已建立。Cloud 只在订阅登记之后发送响应头，此后发布的信号都会进入该流；建立后
  立即领取一次，覆盖订阅之前已被接受、其信号因此无人接收的工作。
- 流在持有租约期间一直保留，续期与 epoch 变化不重开：Cloud 只在打开时校验 epoch，即使陈旧持有者
  仍收到信号，它的所有写入也会被 fencing 拒绝。
- 任一调用被裁决为资格失效、Controller 丢弃 epoch 时，关闭在线的流；重新获取租约后按本条重新打开。
- 等待响应头的上限与读取相同（10 秒）。打开失败按读取归类：`FAILED_PRECONDITION` 为资格失效，丢弃
  epoch；其余失败不建立流，之后的状态按 D2 与 D3 决定。打开 `Watch` 不提交任何东西，超时不属于"未知"。

在两个节拍上都尝试打开：续约节拍保证即使 `claim_interval_ms` 很大，Controller 也在取得租约的同一轮
建立流；领取节拍让断流或 Cloud 重启后的重开延迟与回退领取一致，而不是等到下一次续期。

### D2：领取节拍由流状态决定

| 流状态 | 进入条件 | 领取节拍（`claim_interval_ms`） | 续约节拍（10 秒） | 信号 |
|---|---|---|---|---|
| 无流 | 启动；流以错误结束；在线时丢弃 epoch；排空时打开被明确拒绝（D3） | 持有租约时先尝试打开流，然后领取一次 | 续约；持有租约时尝试打开流 | — |
| 在线 | 打开成功（任何状态） | 不领取 | 续约后兜底领取一次 | `WorkAvailable` 立即领取；`NodeAssignment` 见 D6 |
| 排空 | 收到 `Drain`；流正常结束 | 持有租约时尝试打开流；打开成功才领取，失败时按 D3 保持排空或转入无流 | 续约；持有租约时尝试打开流 | — |

打开成功从任何状态进入"在线"并立即领取（D1）；未持有租约时任何状态都不领取。

- 在线时不做周期领取：Cloud 在每次接受后发信号，周期领取只会重复空查询。兜底领取覆盖 Cloud 因订阅者
  缓冲满而丢弃的信号，以及上一次领取因不可用而未完成的工作；它随续约进行，不引入新的配置或计时器。
- `claim_interval_ms` 的字段与校验不变，含义收窄为"无流时的领取间隔"；minicloud 生成的 500 ms 不需要
  修改。
- 空闲时 Cloud 的领取查询从每个 `claim_interval_ms` 一次降为每 10 秒一次。

| 情形 | 派发延迟上界 |
|---|---|
| 流在线，信号送达 | 一次 `ClaimWork` 与 `RecordDispatch` 往返 |
| 流在线，信号被 Cloud 丢弃 | 10 秒：下一次续约后的兜底领取，并按 D4 一次领完积压 |
| 无流 | `claim_interval_ms` |
| Cloud 关停后重启 | Cloud 恢复后的第一个领取节拍：重开流并立即领取 |

### D3：`Drain` 与正常结束暂停领取，直到新流建立

收到 `Drain`，或流在没有错误的情况下结束，Controller 进入"排空"：

- 不领取，不登记新的派发；
- 继续续约，不释放租约；
- Node 会话、事件接管、查询结果与恢复读取照常进行，不取消用户任务、不销毁沙盒；
- 在领取与续约节拍上尝试打开新流，按打开结果决定去向：

| 打开结果 | 含义 | 去向 |
|---|---|---|
| 成功 | 新的或恢复的 Cloud 实例已接受订阅 | 在线，立即领取 |
| `UNAVAILABLE` 或等待响应头超时 | 发出 `Drain` 的实例仍在停止，或 Cloud 尚不可达 | 保持排空 |
| `FAILED_PRECONDITION`（资格失效） | 丢弃 epoch，重新获取租约后再打开 | 保持排空 |
| 其他错误 | 有一个仍在服务的 Cloud 给出了明确裁决，但不接受 `Watch` | 无流，按周期领取并告警 |

流以错误状态结束（传输中断、Cloud 被强制停止、保活超时）时进入"无流"，按周期领取。

正常结束等同 `Drain`：Cloud 只在排空时正常结束 `Watch`（先向订阅者发送 `Drain` 再关闭其通道），这由
[Cloud 后续决策](../../cloud/controller-integration/20260924-control-listener-keepalive.md) D3 约定为契约；
而 `Drain` 消息与其他信号一样可能因缓冲满被丢弃；正常结束不会丢失，是比 `Drain` 消息更可靠的排空证据。
即使 Cloud 违反该约定，健康实例上的误判也只让领取暂停到下一个节拍的重开成功为止。

不释放租约：`Drain` 表示发送它的 Cloud 实例即将停止，而不是要求 Controller 让位。租约是 Cloud
PostgreSQL 中的事实，Cloud 重启后原租约在过期前仍然有效；此时向正在停止的实例发送 `ReleaseLease`
会被拒绝，即使成功也只会让 Controller 之后以新的 epoch 重新获取。`signals.proto` 中 `Drain` 的注释
已由 Cloud 后续决策改为同样的表述。

暂停而不是继续周期领取：排空后发往该实例的领取只会失败并反复告警；暂停遵守 Cloud 根决策 D5 的
"不再领取"，而持续的重开尝试保证 Cloud 恢复后第一时间继续。

打开被明确拒绝时退出排空：排空的前提是"打不开 `Watch` 意味着 Cloud 尚未恢复"，这只对 `UNAVAILABLE`
与超时成立。Cloud 已恢复、普通调用可用而 `Watch` 持续失败时（例如新实例未注册信号服务而返回
`UNIMPLEMENTED`，或订阅前的校验返回内部错误），若仍只以打开成功结束排空，Controller 会无限期停止
领取且没有可见的失败。排空中的实例以 `UNAVAILABLE` 拒绝订阅，给出其他裁决的通常是仍在服务的实例，
因此转入无流不违背 D5 对发送 `Drain` 的实例"不再领取"的要求；派发降级为周期领取，而不是停止。即使
误判（排空中的实例在登记订阅前的校验中返回了其他错误），代价也只是该实例停止前的几次失败领取与告警，
领取仍不产生归属。

### D4：每次领取连续登记，直到队列为空

一次领取触发（流建立、`WorkAvailable`、兜底或周期领取）重复"`ClaimWork` → 按 Node 命令校验 →
`RecordDispatch`"，直到 Cloud 返回无工作或某一步失败；单批最多 16 项，满批时先让关停与续约得到处理，
再立即继续下一批。

- 必须连续：在线时兜底每 10 秒一次，若一次只领一项，Cloud 丢弃 N 个信号造成的积压要 N×10 秒才能排空。
- 失败即停：队头无法登记时 `ClaimWork` 会一直返回同一项；停止并沿用"同一项只告警一次"的日志规则，
  等待下一次触发。
- 满批让出：长队列不应推迟续约直到租约过期。

`ClaimWork` 仍是纯读，`RecordDispatch` 仍决定归属；连续领取不改变任何归属或幂等规则。

### D5：Cloud 连接启用 HTTP/2 保活

Cloud 通道启用 HTTP/2 PING 保活：连续 30 秒没有读到对端的任何帧才发送 PING，等待回应 10 秒，没有进行
中的调用时也发送。未按时回应时连接被关闭，在线的流以错误结束（进入"无流"，周期领取并重开），后续
调用在新连接上进行。健康连接上每 10 秒的续约回复都会重置计时，因此 PING 实际只在对端停止发送任何帧
时出现，也就是需要探测失联的时候。

- `Watch` 是长时间可能没有消息的流；经过 NAT 或中间设备时，连接可能被静默丢弃而两端都收不到关闭。
  没有保活，Controller 会一直认为流在线，派发延迟退化为 10 秒兜底；同一条失效连接还会让每个普通
  调用等满 10 秒超时，直到内核放弃 TCP 重传（Linux 默认约 15 分钟）。
- 选 HTTP/2 PING 而不是 TCP keepalive：PING 由 gRPC 对端回应，经过终结 TCP 的代理时探测的仍是 Cloud
  本身，且检测时间不依赖各主机的内核参数。
- 与 Cloud 执行策略的关系：grpc-go 服务端默认要求客户端 PING 间隔不短于 5 分钟，服务端两次写帧之间
  累计三次违规即以 `GOAWAY` 关闭整条连接。由于只有连续 30 秒无入站帧才发 PING，健康连接上不会累计
  违规；只有 Cloud 长时间不写任何帧而连接仍在时，默认策略才会断开连接。Cloud 放宽该策略（Cloud 后续
  决策）是纵深防御，不是本条的部署前提。

### D6：`NodeAssignment` 只记录

第一版 Cloud 不产生 `NodeAssignment`；Controller 收到时只记录日志，不改变 Node 会话或派发目标。其语义
留给 scale 决策。

## 不变量

1. 信号不产生归属：Controller 只在 `RecordDispatch` 成功后派发，任何信号都不跳过登记。
2. 未持有租约时不打开流、不领取；流只以当前持有的 epoch 打开，丢弃 epoch 时关闭在线的流。
3. 流在线时，领取只由流建立、信号与每次续约后的兜底触发；无流时按 `claim_interval_ms` 周期领取；排空
   时不领取。
4. 排空不释放租约，不影响 Node 会话、事件接管与恢复读取；只有新流建立，或打开被 `UNAVAILABLE`、超时
   与资格失效之外的错误拒绝，才结束排空。
5. 每次领取触发都登记到队列为空或失败为止，单批有界。
6. Cloud 连接启用 HTTP/2 保活；失联的流以错误结束并退回周期领取。
7. 本机 SQLite 模式、部署配置字段与 Node 协议不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 维持现状，只做周期领取 | 最简单且已满足正确性；但延迟下限是 `claim_interval_ms`，空闲时持续查询 Cloud，已实现的 `Watch` 无人使用 |
| 接入 `Watch`，周期领取仍按 `claim_interval_ms` | 只降低延迟、不减负载；兜底频率由部署配置决定而与流是否健康无关 |
| 在线时的兜底使用新的配置项或独立计时器 | 可以单独调节；但多一个与 Cloud 缓冲策略耦合的部署参数，而续约节拍已是 Controller 与 Cloud 之间固定的最低频心跳 |
| 收到 `Drain` 释放租约（按现有 proto 注释） | 符合"让位"的直觉；但 `Drain` 来自正在停止的 Cloud 实例，释放调用无法到达，即使到达也只让 epoch 变化 |
| `Drain` 只当作断流，立即回退周期领取 | 状态更少；但违背 Cloud 根决策 D5 的"不再领取"，并在 Cloud 停止期间持续产生失败的领取与告警 |
| 排空只以新流建立结束 | 状态迁移最少；但 Cloud 恢复而 `Watch` 持续失败时，Controller 无限期停止领取且没有可见失败 |
| 排空超过固定时长后回退周期领取 | 能覆盖任何未预见的失败；但引入难以取值的时长，并在 Cloud 确实长时间停止时产生注定失败的领取与告警 |
| 每个信号只领一项 | 与信号一一对应、实现最小；但信号被丢弃后，积压只能按兜底节拍逐项排空 |
| 只用 TCP keepalive，不要求 Cloud 改动 | 没有跨仓库依赖；但探测的是最近的 TCP 对端而不是 gRPC 服务，默认内核参数下失联检测要十几分钟 |
| Cloud 经流推送工作并以推送为归属 | 少一次往返；Cloud 根决策已否决：至多一次投递使断线时的归属不可判定 |

## 本决策未解决的问题

- 多 worker 下 `Drain` 的让位与在途协调交接、`NodeAssignment` 与租约分区，仍属于 scale 决策。
- 流上的应用层心跳：当前依赖 HTTP/2 保活，契约没有心跳消息；需要穿越不转发 PING 的中间层时再议。
- Controller 认证与 TLS 不变，仍留给认证 ADR。

## 落地结果

- ora-space/cloud#29（Cloud 主干 `34d8067`）：监听放宽 PING 执行策略，更正 `Drain` 注释并约定 `Watch` 只在
  排空时正常结束。
- ora-space/desktop#614（desktop 主干 `03d521b`）：
  - `third_party/cloud` 移到 `34d8067`，重新生成只有注释变化。这也更正了前序决策中的一处事实：desktop#600
    已关闭而未合并，此前 gitlink 一直是 cloud#19 的 PR head `535ca76c`。
  - Cloud 适配器的长驻任务拆为 `cloud/coordinate.rs`（主循环）、`cloud/signals.rs`（三状态机，对流类型泛型的
    纯逻辑）、`cloud/claim.rs`（批量领取）与只管租约的 `cloud/lease.rs`；`cloud/fault.rs` 区分打开失败中的
    `UNAVAILABLE`/超时与其他拒绝，状态码仍只在该模块检查。
  - `ora-controller-proto` 以 `test-server` feature 生成服务端桩，生产构建不编译；Controller 测试据此托管
    内存假 Cloud。模块尺寸检查不再统计带 `@generated` 标记的生成文件，生成物由漂移检查约束。
  - `scripts/bash/init-cloud-proto.sh` 在全新 partial clone 后重建子模块索引：pin 恰为 Cloud 默认分支头时，
    原流程会把全部契约文件报告为已暂存删除。
  - `docs/controller/local-runtime` 与 `docs/protocols/controller-cloud-contract` 中英文同步。
- 与本决策一致，没有偏离。已观察到的边界：主循环在处理器内串行等待 RPC，Cloud 停止回应期间，续约与领取
  依次等满超时，保活已关闭连接后流的错误事件要等这些调用返回才被处理，其间状态停留在"在线"；领取仍受
  租约与 fencing 约束，不影响正确性。
- 以真实 Cloud 为对端的手工验证：信号到派发登记约 14 ms；空闲时不再按 `claim_interval_ms` 领取；空闲超过
  两分钟不断开；SIGTERM 后进入排空且不再产生领取失败，Cloud 重启后下一个节拍重开流；冻结 Cloud 约 33 秒后
  保活关闭连接，恢复后重新获取租约并重开流。以 cloud#29 之前的 Cloud 为对端，空闲约三分钟未出现 `GOAWAY`。
