# Controller–Node 传输：FrameChannel、本机 IPC 与云端 WebSocket

依据[已批准的传输决策](../../../decisions/node/protocol/20260924-frame-channel-with-ipc-and-websocket.md)。
这里登记的是跨传输一致性和 WebSocket 接入的验证义务。握手身份、单会话、持久接管和确认顺序
这些与传输无关的规则，继续以[本机归属核心用例](../../controller/persistence/local-controller-node-ownership.md)
为准；本文只要求它们在两种传输下都成立。本决策已批准、尚未实现：现有证据只来自当前的 Unix socket 实现
和字节流 codec，没有任何 `FrameChannel` 或 WebSocket 证据。

## Both Transports Carry Identical Frame Bodies

- **风险**：两种传输各自编码，导致字段、校验或超限判断出现分叉；WS 与代理的限额不一致，
  超限消息被部分转发或静默截断。
- **前置状态**：同一组 Controller→Node 和 Node→Controller 消息，包括接近 `MAX_FRAME_LENGTH` 的帧。
- **触发**：分别经 IPC 与 WS 发送；WS 发送 text message、未知 type、无法解码的 JSON、超过上限的
  binary message；IPC 分片写入。
- **必须成立**：两种传输的帧体 `[type][JSON]` 逐字节相同，IPC 线上字节与决策前一致；WS 一条
  binary message 恰好一帧；上述非法输入在两种传输上都作为协议错误关闭连接。
- **禁止结果**：WS 帧体带长度前缀、一条消息承载多帧或半帧、超限帧被截断后仍被解码、传输各自
  维护一份校验逻辑。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 帧体唯一定义与 IPC 线上兼容 | `ora-node-protocol` 往返测试（如 `clone_request_has_independent_wire_and_fragmented_round_trip`）覆盖现有字节流 codec 与分片；尚缺拆出纯帧体函数后的 IPC 线上字节不变证据 | Partial |
| WS 一消息一帧与非法输入拒绝 | 无 | Missing |

决策依据：D1、D2，不变量 2、3。

## Receive Cancellation Never Loses Or Splits A Frame

- **风险**：会话在 `select!` 中因查询 tick 或心跳取消了一次 `recv`，丢掉已读到的部分帧，后续帧
  错位或事件丢失。
- **前置状态**：会话层在接收的同时周期发送查询与心跳；对端缓慢或分片发送较大帧。
- **触发**：在帧只到达一部分时反复触发其他分支。
- **必须成立**：每次成功 `recv` 恰好得到一个完整且已校验的帧；取消不丢帧、不截断；两种传输
  行为一致。
- **禁止结果**：依赖调用方 pin 住读 future 才能保证正确；取消后把剩余字节当作新帧解析。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| IPC 取消安全 | 当前 Controller 以 pin 整帧读取保证，没有针对 `FrameChannel` 接口的测试 | Missing |
| WS 取消安全 | 无 | Missing |

决策依据：D1，不变量 2。

## Session Behavior Is The Same Over IPC And WebSocket

- **风险**：WS 路径复制或绕过会话逻辑，导致身份校验、对账、接管后确认或重放在云端与本机不一致。
- **前置状态**：真实 Node 已持有未确认终态事件；Controller 使用同一个持久存储。
- **触发**：分别经 IPC 和 WS 建立会话；在结果交付中断开，重连后让 ACK 丢失；目标 `NodeId` 不符或
  缺少 clone 能力。
- **必须成立**：两种传输执行同一份会话代码；重连后重放原事件，持久接管后才确认，ACK 丢失后只
  重复确认；身份或能力不符时拒绝派发；断线不改变执行身份、不判定失败。
- **禁止结果**：WS 会话跳过身份校验，把路由选中的沙盒当作 Node 身份，或因断线新建执行。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| IPC 会话语义 | 本机归属核心用例中 `local_ipc_enforces_owner_and_replays_original_clone_after_restart`、`ora-controller/tests/session.rs` 等证据覆盖当前 Unix socket 实现；尚缺迁到 `FrameChannel` 后的同一组证据 | Partial |
| WS 会话语义 | 无；需要在进程内 WS 上重复同一组会话测试 | Missing |
| 经 Sandbox Server 的端到端恢复 | 无；需要在 cluster 中覆盖代理断连、重连、ACK 丢失重放和身份不符 | Missing |

决策依据：D1、D3、D5，不变量 1、5、6。

## One Control Session Regardless Of Listener

- **风险**：抽出准入后，某种监听方式绕过了独占规则；WS 的拒绝经代理后被映射为 `502`，与 Node
  不可达无法区分。
- **前置状态**：Node 以 IPC 或 WS 中的一个入口监听，已有有效会话，并在执行长时间 clone。
- **触发**：第二条连接直接到达，或经 Sandbox Server 到达；旧会话关闭或超时撤销后重连。
- **必须成立**：两种入口都拒绝第二会话且不抢占；WS 先 upgrade 再以约定的 close code（`4409`）
  关闭，经代理后 Controller 能归类为 `Busy`；撤销后旧连接未受理的消息失效。
- **禁止结果**：最后连接者获胜；WS 用 HTTP 状态拒绝；配置同时启用两个入口。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| IPC 独占 | `local_ipc_enforces_owner_and_replays_original_clone_after_restart`、`busy_clone_keeps_heartbeats_and_revokes_timed_out_queued_command` 覆盖当前 accept 循环实现；尚缺独立准入对象后的证据 | Partial |
| WS 独占与 close code 端到端可辨 | 无 | Missing |
| 监听配置只能二选一 | 无 | Missing |

决策依据：D4，不变量 4。

## Connection Failures Are Classified Without Losing Responsibility

- **风险**：所有连接失败混为一类，日志和后续 Provider 无法区分沙盒不存在、凭据错误与暂时不可达；
  或者某类失败触发了执行失败或重建。
- **前置状态**：Controller 有待派发或未确认的执行；Sandbox Server 按静态路由表转发。
- **触发**：Node 未启动、路由返回 `404`/`403`/`502`/`504`、会话被 `4409` 拒绝、握手身份不符。
- **必须成立**：分别归类为 `Unreachable`、`NotFound`、`Rejected`、`Busy`、`Protocol`；各类都按
  `reconnect_ms` 重试，原执行责任保留。
- **禁止结果**：因 `NotFound` 或其他失败标记执行失败、改换执行身份或销毁沙盒。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 失败分类 | 无 | Missing |
| 失败后保留责任 | 本机归属核心用例覆盖 IPC 断连后保留原执行；尚缺 WS 与各分类的证据 | Partial |

决策依据：D7，不变量 6。

## Transport Layer Stays Vendor Neutral

- **风险**：平台寻址或鉴权逻辑渗入传输和会话层，接入新厂商时必须修改会话代码；平台凭据泄露到
  Node。
- **前置状态**：WS 端点由部署配置提供 URL 与请求头（如 `ate-target-actor`、平台凭据）。
- **触发**：更换 atespace、base URL 或请求头；检查 Node 配置与会话代码。
- **必须成立**：只修改端点配置就能切换本地 Sandbox Server 与平台；传输层把请求头原样发送，不解析
  其含义；Node 的配置与镜像不含平台凭据；Node 握手只校验 `ControllerId` 归属。
- **禁止结果**：传输或会话代码中出现 `ate-target-actor` 等厂商字段的特殊处理；Node 校验平台凭据。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 端点配置驱动、请求头原样发送 | 无 | Missing |
| Node 不持有平台凭据 | 无；属于部署前提，需要在 cluster 配置评审或测试中核对 | Missing |

决策依据：D3、D6、D8，不变量 7。
