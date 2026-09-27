---
status: implemented
---

# Controller 控制监听允许 HTTP/2 保活：放宽 PING 执行策略，更正 `Drain` 的持有者义务表述，并约定 `Watch` 只在排空时正常结束

Cloud 的 Controller gRPC 监听（`control.grpc_addr`）接受客户端的 HTTP/2 PING 保活：两次 PING 间隔
不短于 5 秒即不计违规，连接上没有活动流时也允许 PING。`signals.proto` 中 `Drain` 的注释与根决策
D5 一致：它表示发送它的 Cloud 实例即将停止，持有者停止对它领取直到新的 `Watch` 建立，不要求
释放租约，也不取消用户任务。`Watch` 以 OK 状态结束是契约的一部分：只在实例排空时发生，其他任何结束都以
错误状态报告，使持有者可以把正常结束当作排空证据。三项都不改变任何消息、状态码、事务或授权。

本决策处于 `implemented` 状态，由 ora-space/cloud#29 落地（主干合并提交 `34d8067`）；验证证据登记在
[Cloud 内部控制契约核心用例](../../../test-cases/cloud/controller-integration/cloud-owned-internal-grpc-contract.md#controller-keepalive-pings-never-trigger-goaway)。它是
[Cloud 拥有 Internal Control gRPC 契约](0-cloud-owned-internal-grpc-contract.md)的后续决策，修改其 D7
（监听）的传输参数，并澄清 D5 中 `Drain` 的持有者义务与 `Watch` 的结束语义。它只涉及
`internal/controlgrpc` 的服务端选项与 `proto/ora/cloud/internal/v1/signals.proto` 的注释；Controller 侧的
消费方式与保活参数见
[Controller 消费 Watch 信号流](../../controller/api-boundary/20260924-controller-consumes-watch-signals.md)。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：proto 是唯一契约源；`v1` 只接受非破坏性修改；生成物提交并检查漂移 | 继承；D2、D3 只修改注释，是非破坏性变更 |
| D5：`Watch` 至多一次、不持久化、缓冲有界；关停前 `Drain`，之后流正常结束，新订阅为 `UNAVAILABLE` | 继承并澄清：正常结束只在排空时发生，其他结束都是错误状态（D3） |
| D5：持有者收到 `Drain` 后不再领取，不取消用户任务、不销毁沙盒 | 继承并澄清：不再领取持续到新流建立；不要求释放租约（D2） |
| D7：独立地址监听、不经 Gateway、只能位于回环或私网；`x-ora-controller-id` 声明身份 | 继承，不变 |
| D7：监听使用 grpc-go 默认传输参数（未明示） | 修改：显式放宽 keepalive 执行策略（D1） |

## 决策

### D1：接受间隔不短于 5 秒的客户端 PING，允许无活动流时 PING

`controlgrpc.New` 构造的服务端使用执行策略 `MinTime = 5s`、`PermitWithoutStream = true`。

- grpc-go 默认策略为 `MinTime = 5min`、`PermitWithoutStream = false`：间隔更短的 PING，或无活动流时的
  PING，各计一次违规；超过两次即以 `GOAWAY(ENHANCE_YOUR_CALM, "too_many_pings")` 关闭整条连接，连接上
  的 unary 调用与 `Watch` 一并中断；服务端每次写出数据或响应头都会清零违规计数。Controller 的客户端
  只在连续 30 秒没有读到任何帧时发送 PING，而每 10 秒的续约回复让健康连接上从不出现 PING，所以默认
  策略在正常运行中不会断开 Controller（以本决策之前的 Cloud 为对端，空闲约三分钟未出现 `GOAWAY`）。
  PING 只在 Cloud 长时间不写任何帧时出现；此时默认策略会在第三次违规后断开连接，把一次可以恢复的
  停顿变成连接上全部调用的中断。放宽策略让这种停顿只由客户端的保活超时来判定，是纵深防御。
- 5 秒低于任何合理的客户端间隔（grpc-go 客户端的下限是 10 秒，Controller 使用 30 秒），为网络抖动留出
  余量，同时仍能拒绝真正的 PING 洪泛。
- 允许无活动流时 PING：Controller 在 `Watch` 断开后的回退期间同样需要保活，才能在下一个普通调用前
  发现失联连接。
- 监听仍只能位于回环或私网（D7 不变），放宽 PING 策略不扩大暴露面。

服务端不主动发起保活 PING：Go 的 TCP 监听默认对接受的连接启用 TCP keepalive，失联 Controller 的连接
最终会被关闭，其 `Watch` 上下文随之取消并退订；在此之前订阅者缓冲有界，失联订阅只会丢弃自己的信号，
不阻塞发布者，也不影响其他订阅者。

### D2：`Drain` 只表示发送它的实例即将停止

`signals.proto` 中 `Drain` 的注释是：发送它的 Cloud 实例即将停止；持有者停止对其领取，直到新的
`Watch` 建立；它不要求释放租约，从不取消用户任务或销毁沙盒。此前的注释要求持有者"排空在途协调，
然后释放租约"，与 D5 不一致。

- 根决策 D5 与 `cmd/server` 的关停顺序都把 `Drain` 定义为实例关停前的提示；此时释放租约的调用会被
  正在停止的服务端拒绝，而租约是 PostgreSQL 中的事实，Cloud 重启后在过期前仍然有效。
- 注释是契约的一部分，消费方据此实现；注释与决策不一致会迫使实现者二选一。
- 多实例下"让位"意义上的排空若有需要，由 scale 决策以新的信号或字段表达，不复用这条注释。

### D3：`Watch` 只在排空时以 OK 状态结束

`Watch` 以 OK 状态结束只有一种原因：发出它的实例正在排空，且已先向该订阅尝试发送 `Drain`。订阅被
拒绝、传输或内部失败、服务端被强制停止等其他结束方式都以错误状态报告；由调用方取消的流不受此约束，
调用方本身不会观察到它的结束状态。`signals.proto` 中 `rpc Watch` 的注释写明这一规则。

- Watch handler 只在排空关闭订阅通道时正常返回，订阅被拒绝返回 `UNAVAILABLE`，epoch 校验失败返回
  对应裁决。本条把这一既有行为从实现细节提升为契约，没有改变任何代码路径。
- 持有者依赖它区分"实例即将停止"与"流出错"：`Drain` 消息与其他信号一样可能因订阅缓冲满被丢弃，
  而正常结束不会丢失，是更可靠的排空证据
  （[Controller 消费 Watch 信号流](../../controller/api-boundary/20260924-controller-consumes-watch-signals.md) D3）。
  若服务端在排空之外正常结束 `Watch`（例如为轮换长流主动关闭），持有者会误入排空并暂停领取一个节拍。
- 今后若需要在排空之外主动结束流，必须以错误状态（例如 `UNAVAILABLE`）结束，或由新的决策定义新的
  结束语义，不能复用 OK。

## 不变量

1. 监听接受间隔不短于 5 秒的客户端 PING，包括连接上没有活动流时；按此间隔保活的 Controller 不会被
   `GOAWAY` 断开。
2. 保活参数不改变任何 RPC 的消息、状态码、事务或授权；监听仍只能位于回环或私网。
3. `Drain` 只表示发送它的实例即将停止；它不改变归属，不要求释放租约，不取消用户任务。
4. `Watch` 只在发出它的实例排空时以 OK 状态结束；其他结束对调用方都表现为错误状态。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保持默认策略，Controller 以 5 分钟以上的间隔保活 | Cloud 无需改动；但失联检测超过 5 分钟，且间隔贴近 `MinTime` 时网络抖动仍会累计违规 |
| Controller 只用 TCP keepalive | Cloud 无需改动；但探测的是最近的 TCP 对端，经过代理时不能证明 Cloud 仍在服务 |
| 服务端同时主动 PING 客户端 | 能更快清理失联订阅；但 TCP keepalive 已保证最终清理，而缓冲有界使失联订阅不影响他人 |
| `MinTime` 取 10 秒 | 更保守；但与 grpc-go 客户端下限相同，没有余量，一次抖动即可计入违规 |
| 不约定 `Watch` 的结束语义，持有者只以 `Drain` 消息判断排空 | Cloud 无需新增承诺；但 `Drain` 消息可能因缓冲满丢失，持有者会把实例关停误判为断流并反复领取失败 |
| 保留 `Drain` 注释中的"释放租约" | 为多实例预留"让位"语义；但与当前唯一的发送时机矛盾，让位应由 scale 决策显式定义 |

## 本决策未解决的问题

- TLS、Controller 认证与非私网监听，仍留给认证 ADR。
- 多实例下 `Drain` 的让位语义与 `NodeAssignment`，仍留给 scale 决策。

## 落地结果

- ora-space/cloud#29（主干合并提交 `34d8067`）：`internal/controlgrpc` 以常量 `keepaliveMinTime` 设置 D1
  的执行策略；`signals.proto` 的 `Drain` 与 `rpc Watch` 注释写明 D2、D3，`internal/controlpb` 重新生成且
  只有注释变化；`internal/core` 中 `SignalDrain` 的注释与 `internal/controlgrpc` 中英文 README 同步。
- 与提案一致，没有偏离。
- 部署顺序：提案阶段认为启用 30 秒保活的 Controller
  （[Controller 消费 Watch 信号流](../../controller/api-boundary/20260924-controller-consumes-watch-signals.md) D5）
  必须在本决策之后部署，否则空闲约两分钟即被 `GOAWAY`。以本决策之前的 Cloud（`7d5f227`）做的手工对照
  没有复现断开：客户端只在连续 30 秒无入站帧时发 PING，续约回复让健康连接上从不发 PING。本决策仍按
  纵深防御保留，但不构成部署顺序约束。
