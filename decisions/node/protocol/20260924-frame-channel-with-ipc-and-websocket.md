---
status: implemented
---

# Controller–Node 传输：以帧为单位的 FrameChannel，本机 IPC 与云端 WebSocket 共用一套会话

Controller 与 Node 的会话逻辑（握手、身份与能力校验、执行对账、事件接管与 `EventAck`、事件
重放、心跳）只写一份，建立在以“完整协议帧”为单位收发的帧通道（`FrameReceiver` /
`FrameSender`）之上。传输有两种：本机 IPC（Unix socket 上的长度前缀帧，线上字节与此前一致）和
云端 WebSocket（一条 binary message 承载一帧，经平台沙盒路由转发）。Controller 按部署配置中的
`NodeEndpoint` 变体选择连接方式；Node 每次运行只监听一个入口。沙盒的创建、就绪与销毁留给后续的
`SandboxProvider`，本文只固定它与传输之间的边界：Provider 产出与厂商无关的 `NodeEndpoint`，
传输层不认识任何厂商。

本决策处于 `implemented` 状态，由 desktop `0d046a57` 落地。它继承[根决策](0-controller-node-protocol.md)
的协议与传输分离、身份与恢复语义，修改 D3 中“Controller 通过 mTLS 主动拨号 Node”的网络连接方式：
云端经平台 WebSocket 路由连接，鉴权只由平台层承担。本机 IPC 的归属、单一有效会话与 endpoint 权限
规则继承[本机 Controller–Node 归属决策](../../controller/persistence/0-local-controller-node-ownership.md)
的 D1、D2，不变。验证证据登记在
[传输核心用例](../../../test-cases/node/protocol/frame-channel-transports.md)。本文不设计沙盒生命周期
API、Node 出站回连、SSH/stdio 传输、TLS 证书管理，也不改变任何协议消息类型；经真实路由（cluster
Sandbox Server 或平台）的联调属于部署验证，不是本决策的验收条件。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D1：Protocol 与 Transport 分离，各传输映射到相同消息语义 | 继承并细化：分离点落在“帧”而不是“字节流”（D1） |
| 根 D2：明确 framing，拒绝超限、不兼容或无法解码的帧 | 继承，不变；帧体 `[type][JSON]` 与校验只实现一次，IPC 外加 4 字节长度前缀，WS 以消息边界代替（D2） |
| 根 D3：Controller 主动发起连接，Node 不依赖 Controller 进程 | 继承，不变 |
| 根 D3：普通容器服务由 Controller 以 mTLS 直连 Node 私有 endpoint | 修改：云端经平台 WS 路由连接 Node；路由在代理处终止 WS，端到端 mTLS 不适用，鉴权只在平台层（D6） |
| 根 D4：会话绑定身份、断线不判失败、原身份恢复、持久接管后确认 | 继承，不变；会话层与传输无关，两种传输行为一致 |
| 根 D6：控制与心跳优先，缓冲有界，不静默丢终态 | 继承，不变；WS 写入同样有界并可反压 |
| 归属 D1：Node 固定绑定单一 `ControllerId`，`ControllerId` 不是秘密 | 继承，不变；云端同样不把它当作凭据 |
| 归属 D2：单一有效会话，不抢占仍有效的连接 | 继承，规则跨传输一致；拒绝方式见 D4 |
| 归属 D2：本机 Unix socket endpoint，由本机权限保护 | 继承，不变，只作为 IPC 变体的规则 |

## 决策

### D1：抽象在“帧”这一层，不在字节流层

帧通道以一个完整协议帧为单位收发，拆为读写两半：`FrameReceiver::recv` 与 `FrameSender::send`。
Node 侧另有 `Acceptor`，负责接收入站连接、完成传输握手或以“会话占用”拒绝。会话层只依赖这些
接口，Controller 会话与 Node 会话各一份。

接口承诺：

- `recv` 在 `select!` 中被取消时不丢失、不截断帧，会话无需自己 pin 住读取；
- `recv` 以 `None` 表示对端正常结束，其余中断是错误；
- 写入有界：对端慢时阻塞，由调用方的期限判定失败，不无界缓冲。

不把 WebSocket 包装成字节流再复用长度前缀 codec：平台约定一条 binary message 就是一帧，路由按
消息转发与限额；字节流适配会让帧边界与消息边界脱钩，产生双重分帧，并让“一条消息超限”与“一帧
超限”不再是同一个判断。

两端使用静态分发：会话函数对帧通道和 `Acceptor` 泛型，Controller 按端点变体、Node 按监听入口
选择单态化的会话，不使用 trait object。

### D2：帧体只定义一次，两种传输只差边界表达

`node-protocol` 以纯函数编码和解码帧体 `[type][JSON]`，流式读写与帧体共用长度、类型、JSON 和
消息不变量校验。

| 传输 | 线上表示 | 帧边界 | 大小上限 |
|---|---|---|---|
| IPC | `[u32 长度][type][JSON]` | 长度前缀 | 帧体 ≤ `MAX_FRAME_LENGTH` |
| WebSocket | 一条 binary message = `[type][JSON]` | WS 消息边界 | 消息与单个 WS 帧均 ≤ `MAX_FRAME_LENGTH` |

`MAX_FRAME_LENGTH`（16 MiB，含 type 字节）与路由默认的单条消息上限一致，超限在两端和代理处是同一
个判断。WebSocket 收到 text message、超限消息、不支持的 type 或无法解码的帧体时，与 IPC 同样作为
协议错误结束连接。

### D3：端点是部署配置里的 enum，传输层不认识厂商

Controller 配置 `nodes` 的每一项是 `NodeTarget`：`node_id` 加一个 `NodeEndpoint`。`NodeEndpoint`
是 `ipc`（本机 socket 路径）或 `websocket`（`WsEndpoint`：`url` 与请求头映射）。厂商寻址（如
`ate-target-actor`）和平台凭据都只是请求头，传输层原样发送、不解析。

- 同一个 Controller 可以同时管理 IPC Node 与 WS Node。
- 端点校验归各自变体，在打开任何状态前完成：`ipc` 要求绝对路径，其父目录参与与 Controller home、
  受保护目录的重叠检查；`websocket` 要求 `ws://` 或 `wss://` URL 与合法的请求头，否则拒绝启动，
  而不是无限重连。
- `--single-node` 托管只接受 `ipc` 端点：托管的 Node 是本机进程。
- 若未来厂商无法以 WS 暴露 Node，就新增 `NodeEndpoint` 变体及对应的帧通道实现，会话层不变。

握手仍然校验预期 `NodeId`：路由把连接送到哪个沙盒，不能代替 Node 身份。

### D4：Node 只监听一个入口，独占准入与监听方式无关

Node 配置的 `control` 段包含归属、心跳、帧期限和唯一的监听入口 `listen`：`ipc`（`path` 直接位于
Node home 内）或 `websocket`（`bind` 地址与以 `/` 开头的 `path`，其他路径的 upgrade 以 `404` 拒绝）。
两者在类型上二选一，不支持同时监听。

单一有效会话由对 `Acceptor` 泛型的准入循环和可撤销的会话门禁共同负责。持有准入期间（包括受
`frame_timeout_ms` 约束的传输握手窗口）新连接一律被拒绝，旧会话不被抢占；拒绝在会话之外的独立
任务中执行，同样受 `frame_timeout_ms` 约束，不阻塞现有会话的心跳与受理。会话结束或超时撤销后
释放准入，撤销后旧连接未受理的消息不再派发或确认。

拒绝方式因传输而异：

| 传输 | 拒绝方式 | Controller 看到的结果 |
|---|---|---|
| WebSocket | 先完成 upgrade，再以 close code `4409`（`control session busy`）关闭 | 路由原样转发 close code，归类为 `Busy` |
| IPC | 关闭 socket | 握手阶段读到 EOF，归类为 `Protocol` |

WS 不在 upgrade 前返回 HTTP 状态，因为路由会把 Node 的 HTTP 拒绝统一映射为网关错误，与“Node
不可达”无法区分。IPC 没有 close code，Controller 无法区分“会话占用”与 Node 因归属不符而关闭连接；
两者都只是按 `reconnect_ms` 重连，区分它们对本机部署的诊断价值有限，不值得为此扩展协议消息。

### D5：心跳与连接存活

- 应用层 `Heartbeat` 与 I/O 期限是端到端存活的唯一依据。路由两段各自 ping/pong，WS ping 只能
  证明单跳存活，不能证明 Node 仍在响应。
- 两端都不主动发送 WS ping，只自动回复对端的 ping。原批准的方案是两端各自发送 ping 以维持中间
  设施（NAT、负载均衡）的连接；实现未采用，因为发送端由会话独占，定期 ping 需要改为由后台任务
  持有发送端或为其加锁，而 Node 每 `heartbeat_ms` 发送的应用层心跳已让连接持续有流量，路由两段也
  各自发送 ping。若部署中出现只按单一方向流量判断空闲的中间设施，再补充主动 ping；它仍不参与
  会话超时判定。
- 连接关闭一律只表示连接不可用，沿用根 D4：不取消执行，不改换执行身份，不销毁沙盒。

### D6：云端鉴权只由平台层承担，Node 不做鉴权

Controller 使用 Provider 或部署配置提供的平台凭据（以请求头形式放在 `WsEndpoint` 中）连接路由，
由平台决定是否放行。Node 的 WS 监听不校验任何凭据，握手继续只校验 `ControllerId` 归属，而
`ControllerId` 不是秘密。

安全性依赖“Node 的 WS 入口只能经平台路由到达”这一部署前提。部署须满足：

- Node 监听地址不暴露到沙盒网络以外，只能被平台路由（本地为 Sandbox Server）访问；
- 平台凭据不写入 Node 配置或沙盒镜像。

无法满足该前提的部署不能使用本方案的 WS 传输，须先补充 Ora 层 bootstrap credential 的后续决策。
相比根 D3 的 mTLS 直连，本方案失去了 Node 侧的独立验证，换来的是与平台代理拓扑一致、不需要在
沙盒内分发证书。

### D7：会话失败分类

Controller 会话以 `SessionError` 结束，失败归为有限几类，供日志和将来的 Provider 使用。无法建立
连接时由 `ConnectFailure` 细分：

| 分类 | IPC 来源 | WS 来源 |
|---|---|---|
| `Connect(Unreachable)` | socket 不存在、拒绝连接、连接超时 | 网络错误、连接超时，路由 `502`/`503`/`504` |
| `Connect(NotFound)` | — | 路由 `404`（未知沙盒；Node 路径不符也表现为 `404`） |
| `Connect(Rejected)` | — | 其他 HTTP 拒绝（如 `400`/`401`/`403`）、TLS 失败、非法 URL 或请求头 |
| `Busy` | — | close code `4409` |
| `Protocol` | 握手未被接受（含会话占用）、身份或能力不匹配、帧错误 | 同左，另含 text message |
| `Disconnected` | 已建立连接后的 I/O 失败、对端关闭、I/O 期限到期 | 同左，另含非 normal close code |
| `Configuration` | 会话参数非法 | 同左 |
| `Store` | 持久化拒绝或结果未知，此时不发送确认 | 同左 |

原批准的分类只覆盖建立连接与握手；实现增加了 `Disconnected`、`Configuration` 与 `Store`，使已建立
连接的断开、配置错误和持久化失败不再混入连接失败。所有分类都按 `reconnect_ms` 重试，并保留原执行
责任；本文不据此触发沙盒重建，`NotFound` 是否意味着沙盒已被销毁由后续 Provider 决策判断。

### D8：SandboxProvider 只定边界，不定 trait

沙盒生命周期（创建、就绪/状态查询、可重试销毁）由后续的 `SandboxProvider` 负责。本文只固定：

- Provider 的输出是 `NodeId` 与 `NodeEndpoint`；传输与会话层不调用 Provider，也不知道厂商名称；
- 端点目前来自静态部署配置。

目前只有 Substrate 一种平台形态，本地 Sandbox Server 也是对它的模拟。只依据这一种形态定义 Provider
trait，接口只能凭猜测，因此等接入生命周期时，再用 Docker（cluster）与平台两个真实实现提炼。

## 代码归属

| 位置 | 内容 |
|---|---|
| `crates/node-protocol` | 消息定义、帧体编解码与校验；长度前缀的流读写 |
| `crates/node-transport`（`ora-node-transport`） | `FrameReceiver`、`FrameSender`、`Acceptor`、传输与连接错误分类；IPC 实现；WS 客户端、服务端与 `WsEndpoint`（`websocket` feature，`tokio-tungstenite`，`wss://` 使用 rustls） |
| `apps/ora-controller` | `NodeTarget`、`NodeEndpoint` 与校验；`SessionError`；会话层 |
| `apps/ora-node` | `ControlConfig` 与 `ControlListen`；准入循环；会话层 |

## 配置文件兼容

Node 的旧 `ipc` 配置段与 Controller 中以字符串表示的 `endpoint` 不再被接受。配置文件是部署输入，
按“架构整洁优先于兼容”直接改为新形态，手写的部署配置需要手动迁移。minicloud 启动器是例外：它在
自己的状态目录中写过旧配置，且只在文件缺失时写入默认值，升级后会因旧字段无法启动，因此启动器在
初始化时就地迁移。迁移只改写旧字段，保留其他手工修改，经临时文件替换以免留下截断文件，已迁移的
文件不再改写。

## 不变量

1. 握手、对账、接管、确认、重放与心跳的会话逻辑在两种传输下是同一份代码，行为一致。
2. 一次成功的 `recv` 恰好得到一个完整、已校验的帧；取消 `recv` 不会丢失或截断帧。
3. WebSocket 上一条 binary message 恰好承载一帧，帧体与 IPC 的帧体逐字节相同。
4. 任一时刻一个 Node 至多有一个有效控制会话，与监听方式无关；新连接不抢占有效会话。
5. 经路由到达的连接仍须通过预期 `NodeId` 与能力校验后才能派发。
6. 连接断开或失败不改变执行身份、不判定执行失败、不销毁沙盒。
7. 传输层与会话层不包含任何厂商特定的寻址或鉴权逻辑。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 把 WS 适配为字节流，复用长度前缀流 codec | 帧边界与消息边界脱钩、双重分帧、超限判断不一致（D1、D2）；它的优势是改动最小，但与平台“一消息一帧”约定冲突 |
| 会话层使用 trait object | 端点变体在部署时已知，没有运行时多态需求；静态分发更符合仓库约定 |
| Node 同时监听 IPC 与 WS | 目前没有同一 Node 需要两个入口的部署；准入与监听方式无关（D4），以后需要时再加 |
| 现在定义 `SandboxProvider` trait | 只有一种真实平台形态，接口会是猜测（D8） |
| 云端第一版就加入 Ora 层 bootstrap credential | 平台层已控制可达性，不引入凭据分发与轮换；代价与前提见 D6 |
| 重复 WS 会话用 HTTP 409 拒绝 | 经路由后变成网关错误，无法与不可达区分（D4） |
| 为 IPC 增加“会话占用”协议消息 | 两种结果都只是重连，诊断收益不足以改变协议（D4） |

## 本决策未解决的问题

- 沙盒生命周期 API 与 `SandboxProvider` 接口，以及 `NotFound` 与沙盒重建的关系：需要平台与
  Docker 两个真实实现后再决策。
- Ora 层 bootstrap credential 与 Node 侧鉴权：只在 D6 的部署前提不成立时需要。
- WS 主动保活 ping：只在出现 D5 所述的中间设施时需要。
- Node 出站回连、SSH/stdio 传输、TLS 证书管理：没有当前部署需求。
- 平台凭据的来源与轮换：属于 Provider 或部署配置决策。

## 落地结果

desktop `0d046a57` 按“帧体拆分 → 帧通道与 IPC 迁移 → WebSocket 接入”三步落地。IPC 的线上字节与
会话行为不变，原有 IPC 测试全部经新接口通过；WebSocket 在进程内由生产 Controller 会话与生产 Node
验证握手、结果接管、断线与 Node 重启后的重放、确认后停止重放、`4409` 拒绝与身份不符拒绝。与批准
方案不同的最终决定：不主动发送 WS ping（D5）、IPC 下会话占用归为 `Protocol`（D4、D7）、失败分类
增加三类（D7），以及没有单独的准入类型（D4）。
