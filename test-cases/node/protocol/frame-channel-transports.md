# Controller–Node 传输：FrameChannel、本机 IPC 与云端 WebSocket

依据[已实现的传输决策](../../../decisions/node/protocol/20260924-frame-channel-with-ipc-and-websocket.md)。
这里登记的是跨传输一致性和 WebSocket 接入的验证义务。握手身份、单会话、持久接管和确认顺序
这些与传输无关的规则，继续以[本机归属核心用例](../../controller/persistence/local-controller-node-ownership.md)
为准；本文只要求它们在两种传输下都成立。证据来自 desktop `37759e91`；上行心跳、close code 与
`Mismatch`/`Silent` 分类依据[心跳与 close code 决策](../../../decisions/node/protocol/20260925-controller-heartbeat-and-websocket-close-codes.md)，
证据来自 desktop#615 与 desktop#617，两者合入前以其分支上的测试为准。经 cluster Sandbox Server
或平台路由的联调属于部署验证，不在本文登记。

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
| 帧体唯一定义与 IPC 线上兼容 | `ora-node-protocol` 的 `stream_envelope_is_length_prefix_around_frame_body` 证明流式线上字节恰为长度前缀加帧体，`rejects_invalid_frame_bodies` 证明帧体与流式读取共用长度、类型、JSON 和不变量校验；`ora-node-transport` 的 `frames_round_trip_with_length_prefix_wire` 与 `rejects_truncated_and_out_of_range_frames` 覆盖 IPC 通道的线上字节、半帧和越界长度 | Covered |
| WS 一消息一帧与非法输入拒绝 | `frames_round_trip_as_binary_messages` 覆盖双向一消息一帧；`close_codes_and_invalid_messages` 覆盖 text message 与超过 `MAX_FRAME_LENGTH` 的消息被拒绝 | Covered |

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
| IPC 取消安全 | `cancelled_receive_keeps_partial_frame` 以 3 字节分片慢速写入，并在 `select!` 中反复取消 `recv`，验证得到完整帧且其后正常结束 | Covered |
| WS 取消安全 | `cancelled_receive_keeps_fragmented_message` 以慢速分片（中间夹 ping）发送一条消息，并在 `select!` 中反复取消 `recv`，验证整条消息恰好交付一次，其后正常结束 | Covered |

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
| IPC 会话语义 | 本机归属核心用例中的 `local_ipc_enforces_owner_and_replays_original_clone_after_restart`、`independent_controller_replays_durable_takeover_after_lost_ack_and_kill` 与 `ora-controller/tests/session.rs` 在迁移后经 `FrameChannel` 运行，全部通过 | Covered |
| WS 会话语义 | `controller_session_over_websocket_takes_over_clone_and_keeps_single_session` 让生产 Controller 会话经 WebSocket 连接生产 Node，覆盖握手、clone 结果持久接管和身份不符拒绝；`websocket_replays_unacknowledged_result_across_disconnect_and_restart_until_controller_acks` 覆盖未确认结果在断线与 Node 重启后按原身份重放、生产 Controller 会话接管并确认重放事件，确认后状态查询仍返回 `Completed` 而事件不再重放 | Covered |

决策依据：D1、D3、D5，不变量 1、5、6。

## One Control Session Regardless Of Listener

- **风险**：抽出准入后，某种监听方式绕过了独占规则；WS 的拒绝经代理后被映射为 `502`，与 Node
  不可达无法区分。
- **前置状态**：Node 以 IPC 或 WS 中的一个入口监听，已有有效会话，并在执行长时间 clone。
- **触发**：第二条连接到达；旧会话关闭或超时撤销后重连。
- **必须成立**：两种入口都拒绝第二会话且不抢占；WS 先 upgrade 再以约定的 close code（`4409`）
  关闭，Controller 能归类为 `Busy`（路由原样转发 close code，这是 D4 选择该方式的前提）；撤销后旧连接未受理的消息失效。
- **禁止结果**：最后连接者获胜；WS 用 HTTP 状态拒绝；配置同时启用两个入口（`ControlListen`
  是二选一的 enum，该状态在类型上不可表示，不需要运行时证据）。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| IPC 独占 | `local_ipc_enforces_owner_and_replays_original_clone_after_restart`、`busy_clone_keeps_heartbeats_and_revokes_timed_out_queued_command` 在准入改为对 `Acceptor` 泛型后通过；`connect_classifies_missing_endpoint_and_busy_rejection_closes` 覆盖 IPC 拒绝即关闭 | Covered |
| WS 独占与 close code 可辨 | `busy_rejection_is_recognizable_close_code` 覆盖 upgrade 后以 `4409` 关闭且可识别；真实 Node 测试在会话存活时验证第二连接收到 `4409` | Covered |

决策依据：D4，不变量 4。

## Connection Failures Are Classified Without Losing Responsibility

- **风险**：所有连接失败混为一类，日志和后续 Provider 无法区分沙盒不存在、凭据错误与暂时不可达；
  或者某类失败触发了执行失败或重建。
- **前置状态**：Controller 有待派发或未确认的执行；路由以原始 HTTP 响应模拟。
- **触发**：Node 未启动、路由返回 `404`/`403`/`502`/`504`、会话被 `4409` 拒绝、握手身份或能力不符、
  收到 `4403`、本端 I/O 期限到期。
- **必须成立**：分别归类为 `Connect(Unreachable)`、`Connect(NotFound)`、`Connect(Rejected)`、
  `Busy`、`Mismatch`、`Silent`、`Protocol`、`Disconnected`；身份或能力不符（本端发现或收到 `4403`）
  归为 `Mismatch` 而不是 `Protocol`，本端 I/O 期限到期归为 `Silent` 而不是 `Disconnected`；IPC 无
  close code，会话占用归入 `Protocol`；各类都按 `reconnect_ms` 重试，原执行责任保留。
- **禁止结果**：因 `NotFound` 或其他失败标记执行失败、改换执行身份或销毁沙盒。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 失败分类 | `connect_failures_are_classified` 覆盖路由 `404`/`403`/`502`/`504`、错误路径、无监听和非法端点；Controller `connection_failures_are_classified_without_releasing_responsibility` 覆盖 IPC 与 WS 的 `Unreachable`；真实 Node 测试覆盖 `Protocol` 与 `Busy` | Covered |
| `Mismatch` 与 `Silent` | desktop#617：`node_identity_close_is_a_mismatch` 覆盖收到 `4403` 归为 `Mismatch`，`controller_closes_with_the_reason_code` 覆盖本端发现 Node 不符时归为 `Mismatch`；本端 I/O 期限到期归为 `Silent` 没有直接断言 | Partial |
| 失败后保留责任 | 同一 Controller 测试在两种连接失败后验证原派发仍待处理且无结果；本机归属核心用例覆盖 IPC 断连后保留原执行 | Covered |

决策依据：传输决策 D7，不变量 6；心跳与 close code 决策 D4，不变量 7。

## Transport Layer Stays Vendor Neutral

- **风险**：平台寻址或鉴权逻辑渗入传输和会话层，接入新厂商时必须修改会话代码；平台凭据泄露到
  Node。
- **前置状态**：WS 端点由部署配置提供 URL 与请求头（如 `ate-target-actor`、平台凭据）。
- **触发**：更换 atespace、base URL 或请求头；检查 Node 配置与会话代码。
- **必须成立**：切换路由只需修改端点配置；传输层把请求头原样发送，不解析其含义；Node 握手只校验
  `ControllerId` 归属。Node 的配置与镜像不含平台凭据是 D6 的部署前提，由部署方保证，不在本文登记
  测试证据。
- **禁止结果**：传输或会话代码中出现 `ate-target-actor` 等厂商字段的特殊处理；Node 校验平台凭据。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 端点配置驱动、请求头原样发送 | `configured_headers_are_sent_verbatim` 验证路由收到原样的路径、query 与 `ate-target-actor`；传输与会话代码不包含厂商字段 | Covered |

决策依据：D3、D6、D8，不变量 7。

## Idle Controller Keeps Its Session Through Heartbeats

- **风险**：Controller 空闲时没有上行帧，Node 的每帧读期限到期后反复断开并重建会话；或为了避免
  断开而取消 Node 读期限，使半开连接长期占住控制槽，代理恢复后首次重连收到 `4409`。
- **前置状态**：会话已建立，Controller 没有待确认的派发；Node 的 `frame_timeout_ms` 远小于空闲时长，
  `query_interval_ms` 小于其一半。
- **触发**：长时间空闲；空闲期间收到 `controller_id` 不属于本 Node 的上行 `Heartbeat`。
- **必须成立**：Controller 每个查询节拍恰好发送一个上行帧，无待查派发时为 `Heartbeat`；Node 在读循环
  内处理心跳、不进入 worker 受理队列，空闲会话持续存活并保有准入；`controller_id` 不符时会话以身份
  不符结束并释放准入。
- **禁止结果**：空闲会话因读期限到期而结束；上行心跳排在 Git 或 SQLite 之后；Node 取消读期限；
  其他 Controller 的心跳被接受。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 上行心跳线上形状 | desktop#615：`ora-node-protocol` 会话切片的 `controller_heartbeat` 线上 fixture 经 `round_trips_messages`、`preserves_wire_shapes`、`rejects_versions_directions_and_payloads` 与 `rejects_missing_and_empty_fields` 覆盖形状、方向、版本与空 `controller_id` 拒绝 | Covered |
| Controller 空闲节拍发送心跳 | desktop#615：`idle_session_sends_one_heartbeat_per_query_tick` 验证无待查派发时每个节拍恰好收到一个携带 `controller_id` 的 `Heartbeat`；`uncertain_execution_retransmits_at_most_once_per_connection` 验证有待查派发时只发查询 | Covered |
| Node 以心跳维持空闲会话并校验归属 | desktop#615：`controller_heartbeats_keep_idle_session_and_foreign_heartbeat_ends_it` 让真实 Node 以 1 s 读期限经 IPC 空闲 3 s 仍保有准入，其后他人心跳结束会话并释放准入；WS 由同一份会话代码处理，没有单独的 WS 空闲测试 | Covered |

决策依据：心跳与 close code 决策 D1、D3，不变量 1–4。

## Deliberate Session End Carries A Close Code

- **风险**：会话由一端主动结束时不发 close，中继一律代发 `1011 "<side> connection lost"`，日志无法区分
  正常停止、协议错误、身份不符、存活超时与真实断线；或关闭握手占用 Node 准入，使下一次重连收到 `4409`。
- **前置状态**：WS 会话已建立，可能经过原样转发 close code 的路由。
- **触发**：Node 或 Controller 正常停止；非法帧；身份或能力不符；I/O 期限到期；本端持久化或受理失败。
- **必须成立**：结束方按原因发送 `1001`/`1002`/`4403`/`4408`/`1011`，对端回复 close，双方不等待握手
  超时；关闭握手受 I/O 期限约束，Node 因失败关闭时不占用准入；任何 close code 都不改变执行责任。
- **禁止结果**：主动结束时不发 close；关闭后不离开的对端让下一次重连收到 `4409`；因 close code 判定
  执行失败或销毁沙盒。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 每种原因的 code 与关闭握手 | desktop#617：`deliberate_close_carries_reason_and_is_answered` 对每种原因验证 code 送达且对端回复 close；`close_connection_finishes_with_the_close_handshake` 覆盖排空至握手完成；IPC `deliberate_close_is_a_clean_end_of_stream` 覆盖 IPC 关闭即 EOF | Covered |
| 真实两端按原因关闭 | desktop#617：`node_closes_sessions_with_the_reason_code` 覆盖真实 Node 的 `4403`、`1002` 与 SIGTERM 下的 `1001`；`controller_closes_with_the_reason_code` 覆盖 Controller 的 `4403`、`1002` 与停止时 `1001`；两端以 `4408`、`1011` 关闭没有直接测试 | Partial |

决策依据：心跳与 close code 决策 D2，不变量 5、6。
