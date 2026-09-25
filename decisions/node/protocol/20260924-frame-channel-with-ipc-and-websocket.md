---
status: approved
---

# Controller–Node 传输：以帧为单位的 FrameChannel，本机 IPC 与云端 WebSocket 共用一套会话

Controller 与 Node 的会话逻辑（握手、身份与能力校验、执行对账、事件接管与 `EventAck`、事件
重放、心跳）只写一份，建立在以“完整协议帧”为单位收发的 `FrameChannel` 之上。第一版有两个
实现：本机 IPC（Unix socket 上的长度前缀帧，行为不变）和云端 WebSocket（一条 binary message
承载一帧，经平台沙盒路由转发）。Controller 按部署配置中的 `NodeEndpoint` 变体选择连接方式；
Node 每次运行只监听一个入口。沙盒的创建、就绪与销毁留给后续的 `SandboxProvider`，本文只固定
它与传输之间的边界：Provider 产出与厂商无关的 `NodeEndpoint`，传输层不认识任何厂商。

本决策处于 `approved` 状态（2026-09-25 评审通过，尚未实现）。验证义务登记在
[传输核心用例](../../../test-cases/node/protocol/frame-channel-transports.md)。它继承[根决策](0-controller-node-protocol.md)的协议与传输分离、
身份与恢复语义，修改 D3 中“Controller 通过 mTLS 主动拨号 Node”的网络连接方式：云端第一版经
平台 WebSocket 路由连接，鉴权只由平台层承担。本机 IPC 的归属、单一有效会话与 endpoint 权限规则
继承[本机 Controller–Node 归属决策](../../controller/persistence/0-local-controller-node-ownership.md)
的 D1、D2，不变。涉及 `crates/node-protocol`、新增的 `crates/node-transport`、
`apps/ora-controller` 的会话与部署配置、`apps/ora-node` 的服务监听与配置。本文不设计沙盒生命周期
API、Node 出站回连、SSH/stdio 传输、TLS 证书管理，也不改变任何协议消息类型。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D1：Protocol 与 Transport 分离，各传输映射到相同消息语义 | 继承并细化：分离点落在“帧”而不是“字节流”（D1） |
| 根 D2：明确 framing，拒绝超限、不兼容或无法解码的帧 | 继承，不变；帧体 `[type][JSON]` 与校验只实现一次，IPC 外加 4 字节长度前缀，WS 以消息边界代替（D2） |
| 根 D3：Controller 主动发起连接，Node 不依赖 Controller 进程 | 继承，不变 |
| 根 D3：普通容器服务由 Controller 以 mTLS 直连 Node 私有 endpoint | 修改：云端第一版经平台 WS 路由连接 Node；路由在代理处终止 WS，端到端 mTLS 不适用，鉴权只在平台层（D6） |
| 根 D4：会话绑定身份、断线不判失败、原身份恢复、持久接管后确认 | 继承，不变；会话层与传输无关，两种传输行为一致 |
| 根 D6：控制与心跳优先，缓冲有界，不静默丢终态 | 继承，不变；WS 写入同样有界并可反压 |
| 归属 D1：Node 固定绑定单一 `ControllerId`，`ControllerId` 不是秘密 | 继承，不变；云端同样不把它当作凭据 |
| 归属 D2：单一有效会话，不抢占仍有效的连接 | 继承，规则跨传输一致；WS 下的拒绝方式见 D4 |
| 归属 D2：本机 Unix socket endpoint，由本机权限保护 | 继承，不变，只作为 IPC 变体的规则 |

## 背景

平台沙盒方案与 cluster 的本地 Sandbox Server 都采用双段 WebSocket：

```text
Controller ── WS ── Sandbox Server / 平台路由 ── WS ── Ora Node
```

路由按 `ate-target-actor: <atespace>/<sandbox-id>` 请求头选择沙盒，路径与 query 原样转给 Node，
逐条消息整条转发，单条消息上限默认 16 MiB，close code 跨段传播，两段各自 ping/pong。

当前实现与 Unix socket 耦合：Controller `run_session` 直接 `UnixStream::connect`，
`NodeEndpoint.endpoint` 是 `PathBuf`；Node `session::listen` 直接使用 `UnixListener`，独占规则写在
`run_exclusive` 的 accept 循环里；配置只有 `IpcConfig`。协议帧 codec 已基于泛型
`AsyncRead/AsyncWrite`，会话语义本身与 socket 无关。

## 决策

### D1：抽象在“帧”这一层，不在字节流层

`FrameChannel` 以一个完整协议帧为单位收发，拆为读写两半，与两端现有的 `into_split` 用法一致。
会话层只依赖该接口，对两端各写一份：Controller 会话与 Node 会话。

接口承诺：

- `recv` 在 `select!` 中被取消时不丢失、不截断帧。当前 Controller 以 pin 住整帧读取来保证这一点；
  抽象后这是接口义务，不再由调用方记住。
- `recv` 以 `None` 表示对端正常结束，其余中断是错误。
- 写入有界：对端慢时阻塞或超时失败，不无界缓冲。

不把 WebSocket 包装成 `AsyncRead/AsyncWrite` 再复用长度前缀 codec，理由是：平台约定一条
binary message 就是一帧，路由按消息转发与限额；字节流适配会让帧边界与消息边界脱钩，产生双重
分帧，并让“一条消息超限”与“一帧超限”不再是同一个判断。

两端使用静态分发：会话函数对 `FrameChannel` 泛型，Controller 先 `match` 端点变体再进入对应的
单态化会话，不使用 trait object。

### D2：帧体只定义一次，两种传输只差边界表达

`node-protocol` 提供纯函数，把消息编码为帧体 `[type][JSON]`，并从帧体解码和校验消息，不涉及 I/O。
两种传输只在如何划分帧的边界上不同：

| 传输 | 线上表示 | 帧边界 | 大小上限 |
|---|---|---|---|
| IPC | `[u32 长度][type][JSON]` | 长度前缀（现状，不变） | 帧体 ≤ `MAX_FRAME_LENGTH` |
| WebSocket | 一条 binary message = `[type][JSON]` | WS 消息边界 | 消息 ≤ `MAX_FRAME_LENGTH` |

`MAX_FRAME_LENGTH`（16 MiB，含 type 字节）与路由默认的单条消息上限一致，因此超限在两端和代理
处是同一个判断。WebSocket 收到 text message、type 不支持或帧体无法解码时，与 IPC 同样作为协议
错误关闭连接。

### D3：端点是部署配置里的 enum，传输层不认识厂商

```rust
enum NodeEndpoint {
    Ipc { path: PathBuf },
    WebSocket(WsEndpoint),
}

struct WsEndpoint {
    url: Url,                       // ws:// 或 wss://，含路径，如 /ora-node/v1
    headers: Vec<(HeaderName, HeaderValue)>, // 如 ate-target-actor
    // 平台层凭据同样以请求头表达（D6）
}
```

这里的代码只说明语义，不构成实现契约。

- Controller 的 `nodes` 中每项为 `node_id` 加一个 `NodeEndpoint`，同一个 Controller 可以同时管理
  IPC Node 与 WS Node。
- 端点校验归各自变体：绝对路径、与 Controller home 及受保护目录不重叠只约束 `Ipc`；`WebSocket`
  只校验 URL scheme 与请求头格式。
- 厂商差异（请求头寻址、子域名、路径、临时 token）只由 Provider 翻译为 `WsEndpoint`。若未来厂商
  无法以 WS 暴露 Node，就新增 `NodeEndpoint` 变体及对应的 `FrameChannel`，会话层不变。

握手仍然校验预期 `NodeId`：路由把连接送到哪个沙盒，不能代替 Node 身份。

### D4：Node 只监听一个入口，独占准入独立于监听方式

Node 配置的监听入口是二选一的 enum：`Ipc { endpoint, … }` 或 `WebSocket { bind, path, … }`。
第一版不支持同时监听。

单一有效会话由独立的准入对象负责，监听器只负责接收连接并向它申请准入。持有准入期间（包括握手
窗口）新连接一律被拒绝，旧会话不被抢占；会话结束或超时撤销后释放准入，撤销后旧连接未受理的
消息不再派发或确认。这样规则不随传输变化，将来需要多入口时也不必改写准入。

WS 下拒绝重复会话的方式是：先完成 upgrade，再以应用自定义 close code 关闭。

| 方式 | 经路由后 Controller 看到的结果 | 采用 |
|---|---|---|
| upgrade 前返回 HTTP 409 | 路由把 Node 拒绝统一映射为 `502`，与“Node 不可达”无法区分 | 否 |
| upgrade 后以 `4409 control session busy` 关闭 | 路由原样传播 close code，Controller 能识别“已有会话” | 是 |

具体 close code 取值在实现前固定，并在两端共用常量。

### D5：心跳与连接存活

- 应用层 `Heartbeat` 与 `io_timeout` 是端到端存活的唯一依据。路由两段各自 ping/pong，WS ping
  只能证明单跳存活，不能证明 Node 仍在响应。
- WS 客户端和服务端各自发送 ping，只用于维持中间设施（NAT、负载均衡）的连接，不参与会话超时
  判定。
- 连接关闭一律只表示连接不可用，沿用根 D4：不取消执行，不改换执行身份，不销毁沙盒。

### D6：云端第一版鉴权只由平台层承担，Node 不做鉴权

Controller 使用 Provider 或部署配置提供的平台凭据（以请求头形式放在 `WsEndpoint` 中）连接路由，
由平台决定是否放行。Node 的 WS 监听不校验任何凭据，握手继续只校验 `ControllerId` 归属，而
`ControllerId` 不是秘密。

这是有意接受的边界：安全性依赖于“Node 的 WS 入口只能经平台路由到达”这一部署前提。部署须满足：

- Node 监听地址不暴露到沙盒网络以外，只能被平台路由（本地为 Sandbox Server）访问；
- 平台凭据不写入 Node 配置或沙盒镜像。

若某个部署无法满足该前提，就不能使用本方案的 WS 传输，须先补充 Ora 层 bootstrap credential
的后续决策。相比根 D3 的 mTLS 直连，本方案失去了 Node 侧的独立验证；换来的是与平台代理拓扑
一致、不需要在沙盒内分发证书。

### D7：连接失败分类

Controller 把连接失败归为有限几类，供日志和将来的 Provider 使用，而不是全部折叠为同一种错误：

| 分类 | IPC 来源 | WS 来源 |
|---|---|---|
| `Unreachable` | socket 不存在、拒绝连接、超时 | 网络错误，路由 `502`/`504` |
| `NotFound` | — | 路由 `404`（未知沙盒） |
| `Rejected` | — | 路由 `400`/`401`/`403`/`426` 等配置或凭据问题 |
| `Busy` | 已有会话时被关闭 | close code `4409` |
| `Protocol` | 握手失败、身份或能力不匹配、帧错误 | 同左 |

所有分类的处理都是按 `reconnect_ms` 重试，并保留原执行责任。本文不据此触发沙盒重建：
`NotFound` 是否意味着沙盒已被销毁，由后续 Provider 决策判断。

### D8：SandboxProvider 只定边界，不定 trait

沙盒生命周期（创建、就绪/状态查询、可重试销毁）由后续的 `SandboxProvider` 负责。本文只固定：

- Provider 的输出是 `NodeId` 与 `NodeEndpoint`；传输与会话层不调用 Provider，也不知道厂商名称。
- 第一版端点来自静态部署配置。这与 cluster Sandbox Server 用静态路由表过渡的做法对应。

目前只有 Substrate 一种平台形态，本地 Sandbox Server 也是对它的模拟。只依据这一种形态就定义
Provider trait，接口只能凭猜测，因此等接入生命周期时，再用 Docker（cluster）与平台两个真实实现
提炼。

## 代码归属

| 位置 | 内容 |
|---|---|
| `crates/node-protocol` | 消息定义、帧体编解码与校验（纯函数）；长度前缀的流读写保留供 IPC 使用 |
| `crates/node-transport`（新增，`ora-node-transport`） | `FrameChannel`、IPC 实现、WS 客户端与服务端实现（WS 依赖以 Cargo feature 控制）、连接失败分类 |
| `apps/ora-controller` | `NodeEndpoint` enum 与校验；会话层改为对 `FrameChannel` 泛型 |
| `apps/ora-node` | 监听入口 enum；准入对象；会话层改为对 `FrameChannel` 泛型 |

WS 库倾向 `tokio-tungstenite`；工作区当前没有 WS 依赖，选型在实现评审时确认。

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
| 把 WS 适配为 `AsyncRead/AsyncWrite`，复用长度前缀流 codec | 帧边界与消息边界脱钩、双重分帧、超限判断不一致（D1、D2）；它的优势是改动最小，但与平台“一消息一帧”约定冲突 |
| 会话层使用 `Box<dyn FrameChannel>` | 端点变体在部署时已知，没有运行时多态需求；静态分发更符合仓库约定 |
| Node 同时监听 IPC 与 WS | 目前没有同一 Node 需要两个入口的部署；准入已经独立（D4），以后需要时再加，不必改写规则 |
| 现在定义 `SandboxProvider` trait | 只有一种真实平台形态，接口会是猜测（D8） |
| 云端第一版就加入 Ora 层 bootstrap credential | 平台层已控制可达性，第一版不引入凭据分发与轮换；代价与前提见 D6 |
| 重复会话用 HTTP 409 拒绝 | 经路由后变成 `502`，无法与不可达区分（D4） |

## 本决策未解决的问题

- 沙盒生命周期 API 与 `SandboxProvider` 接口，以及 `NotFound` 与沙盒重建的关系：需要平台与
  Docker 两个真实实现后再决策。
- Ora 层 bootstrap credential 与 Node 侧鉴权：只在 D6 的部署前提不成立时需要。
- Node 出站回连、SSH/stdio 传输、TLS 证书管理：没有当前部署需求。
- 平台凭据的来源与轮换：属于 Provider 或部署配置决策。

## 落地顺序

1. **帧体拆分**：`node-protocol` 提供纯帧体编解码，IPC 流读写基于它实现。完成条件是现有协议
   测试不变地通过；线上字节不变。
2. **FrameChannel 与 IPC 实现**：新增 `ora-node-transport`，两端会话层改为对 `FrameChannel` 泛型，
   Node 准入从 accept 循环中独立出来。完成条件是本机 IPC 的握手、重复会话拒绝、断线重放和确认
   丢失重放测试全部经新接口通过；用户可见行为不变。
3. **WebSocket 实现与配置**：加入 `NodeEndpoint::WebSocket` 与 Node 的 WS 监听入口，以及 D4 的
   close code 与 D7 的失败分类。完成条件是在进程内 WS 上重复第 2 步的会话测试，并覆盖超限消息、
   text message、`4409` 拒绝与断线后原身份恢复。
4. **经 cluster Sandbox Server 联调**：用静态路由表跑通 clone 闭环，覆盖代理断连、重连、ACK 丢失
   重放、身份不匹配与重复连接拒绝。

第 1、2 步不改变任何对外行为，可以独立评审；从第 3 步开始新增云端连接方式；本机 IPC 行为在
所有步骤中保持不变。
