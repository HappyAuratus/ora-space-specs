---
status: implemented
---

# Controller 上行心跳作为 Node 读期限的存活依据，WebSocket 会话以约定 close code 结束

> 后续已实现决策：[运行时控制同步交付](../../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)约束运行时旧指令与交互输入，状态为 `implemented`，完整故障矩阵仍按核心用例验证。本文保留原状态；心跳或会话关闭不等同于用户撤权或执行结束。

Controller 会话在空闲时向 Node 发送应用层 `Heartbeat`，Node 保留每帧读期限 `frame_timeout_ms`，
把它作为“Controller 仍在、连接仍通”的依据。空闲会话不再每约 11 s 被 Node 断开并重建两段连接；
半开连接会在一个读期限内被 Node 发现并释放控制槽。同时固定 WebSocket 会话结束时两端发送的
close code，使中继与 Controller 能区分进程停止、协议错误、身份不符、存活超时与内部失败，而不是
一律看到中继代发的 `1011 "<side> connection lost"`。

本决策处于 `implemented` 状态：2026-09-26 经 Eric 批准采用“Controller 上行心跳”，备选“Node 只在
有未完成请求时施加读超时”的否决理由见替代方案。上行心跳由 desktop `a767bc7` 落地，close code 与
`Mismatch`/`Silent` 分类由 desktop `3d819e6` 落地，他人心跳以 `4403` 关闭在两者合并时（`46ce0e0`）
接通。验证义务登记在
[传输核心用例](../../../test-cases/node/protocol/frame-channel-transports.md)。它继承
[帧通道传输决策](20260924-frame-channel-with-ipc-and-websocket.md)，修改其 D5（心跳与连接存活）
与 D7（会话失败分类）中身份不符、I/O 期限到期和 close code 相关的行，其余不变。涉及 `ora-node-protocol` 的
Controller→Node 消息、Controller 会话与 `SessionError`、Node 会话读循环和 `ora-node-transport` 的关闭接口。
协议版本保持 `1`，不支持 Controller 与 Node 混合新旧版本（见 D1）。本文不处理 Node 读循环被
受理阻塞（问题 2a）和 Git 占用 worker（问题 2b），它们由各自的后续决策或实现负责。

## 背景

2026-09-25 经 cluster Sandbox Server 手动联调时发现两个问题：

1. **空闲会话每约 11 s 被断开**。Node 读循环对每帧施加 `frame_timeout_ms`（10 s）期限，而 Controller
   只在 `pending_dispatches` 非空时发送 `GetExecutionStatus`，空闲时没有任何上行帧。`docs/node/local-ipc.md`
   写有“空闲 Controller 应定期查询”，实现没有做到。功能不受损，但每 11 s 一条告警、两段连接重建；
   IPC 同样存在。
2. **会话结束时两端都不发送 close frame**（唯一例外是 `4409`）。中继把任一段未经 close 的断开转成
   对侧 `1011 "<side> connection lost"`，日志无法区分正常停止和真实断线。

## 继承与修改

| 前序约定（传输决策） | 本文处理 |
|---|---|
| D1–D4：帧通道、帧体、端点配置、单入口独占准入与 `4409` | 继承，不变 |
| D5：应用层 `Heartbeat` 与 I/O 期限是端到端存活的唯一依据 | 继承，不变 |
| D5：只有 Node 每 `heartbeat_ms` 发送应用层心跳 | 修改：Controller 也发送上行 `Heartbeat`，节奏见 D1 |
| D5：两端不主动发 WS ping，只自动回复 | 继承，不变；上行心跳让两个方向都持续有流量 |
| D5：连接关闭只表示连接不可用，不取消执行、不改身份、不销毁沙盒 | 继承，不变；close code 只用于诊断和分类 |
| D6：鉴权只在平台层 | 继承，不变 |
| D7：`Busy` 来自 `4409`，其余非 normal close 归为 `Disconnected` | 修改：`1002` 归为 `Protocol`，`4403` 归为新增的 `Mismatch`，其余不变（D2、D4） |
| D7：身份或能力不符归为 `Protocol` | 修改：归为新增的 `Mismatch`（D4） |
| D7：I/O 期限到期归为 `Disconnected` | 修改：归为新增的 `Silent`（D4） |
| D7：所有分类都按 `reconnect_ms` 重试 | 继承，不变 |
| 根 D4：会话绑定 `ControllerId` 与 Node 身份 | 继承；上行心跳携带 `ControllerId`，不符即身份不符（`4403`） |

## 决策

### D1：Controller 在每个查询节拍上恰好发送一个上行帧

Controller 会话已有每 `query_interval_ms` 一次的查询节拍。每个节拍发送恰好一个上行帧：

- 有待确认的派发时，照旧轮换发送一条 `GetExecutionStatus`；
- 没有时，发送一条 `Heartbeat { controller_id }`。

新增的 `Heartbeat` 是 `ControllerToNodeMessage` 的一个变体，线上 `message_type` 为 `heartbeat`，
负载只有 `controller_id`；它与 Node→Controller 的 `Heartbeat` 分属两个方向的消息枚举，互不混淆。

Node 收到上行 `Heartbeat` 时：

- 在会话读循环内直接处理，不进入 worker 受理队列，因此不会排在 Git 或 SQLite 之后；
- `controller_id` 必须等于本 Node 的归属 Controller，否则按身份不符结束会话（WS 上以 `4403` 关闭）；
- 不回复。Node 自己的 `Heartbeat` 仍按 `heartbeat_ms` 独立发送。

Node 的每帧读期限 `frame_timeout_ms` 不变，语义从“等待下一条命令”变为“Controller 存活期限”：
期限内收不到 Controller 的任何帧，说明 Controller 已失联或连接半开，Node 结束会话并释放控制槽。

为什么复用查询节拍而不是新增 `heartbeat_ms`：

- 查询节拍本来就是 Controller 唯一的周期性上行，空闲时补一个心跳帧即可让上行间隔恒为
  `query_interval_ms`，不需要第二个计时器，也不改变有待查派发时的行为；
- `SessionConfig` 使用 `deny_unknown_fields`，新增必填字段会让现有部署配置和 minicloud 启动器写出
  的配置无法启动，而这个取舍不值得一次配置迁移。

由此产生一条跨进程的部署约束：`query_interval_ms` 必须小于 Node 的 `frame_timeout_ms`，且留出
网络与调度余量（建议不超过其一半）。两个值在不同进程的配置中，无法在一端启动时校验；minicloud
默认 `1000` / `10000`，满足约束。部署文档须写明该约束。

协议版本保持 `1`。Controller 与 Node 由同一个 desktop 仓库同时发布，minicloud 与云端沙盒镜像都
使用同一次构建；Hello 协商目前只有一个版本，为一条新消息引入版本协商只会增加尚无使用者的兼容
路径。旧 Node 收到上行 `Heartbeat` 会以解码错误结束会话，因此混合新旧版本是不支持的部署。

### D2：WebSocket 会话以约定 close code 结束

主动结束一个已建立会话的一端，按结束原因发送 close code；中继原样转发，接收端回复 close，双方都
不必等待关闭握手超时。发送受该端现有 I/O 期限约束，失败不改变会话结果、不影响执行责任，也不阻塞
重连。因对端关闭而结束的一端（收到 close、读到 EOF、`Busy`）不再回发带原因的 close，只完成握手。
IPC 没有 close code：关闭只结束写方向，对端读到 EOF。

关闭握手有界且不占用准入：发出 close 的一端排空入站数据直到对端回复，最长等待该端的 I/O 期限；
Node 因失败关闭时在准入之外完成这一步，立即释放控制槽，关闭后不离开的对端不会让下一次重连收到
`4409`。Node 正常停止与 Controller 运行时停止则等待 close 发出后再退出，Controller 最多等待
2 × `io_timeout_ms`。

| 原因 | code | 发送方与时机 | 接收方 Controller 的分类 |
|---|---|---|---|
| 进程停止 | `1001` | Node 正常停止；Controller 运行时停止 | `Disconnected` |
| 协议错误 | `1002` | 帧解码失败、text message、握手顺序错误（缺少 `Hello` / `HelloAccepted`）、不支持的消息；Node worker 判定的校验失败、身份冲突或非法确认 | `Protocol` |
| 身份不符 | `4403` | Node：`ControllerId`（含上行 `Heartbeat` 中的）不属于本 Node；Controller：`node_id` 或能力不符 | `Mismatch` |
| 存活超时 | `4408` | 任一端的读或写 I/O 期限到期（含未在期限内收到心跳） | `Disconnected` |
| 内部失败 | `1011` | Controller 持久化失败；Node 受理队列不可用、受理期限到期、存储失败 | `Disconnected` |
| 会话占用 | `4409` | Node 已有有效会话时的新连接（传输决策 D4，不变） | `Busy` |

`1000` 不主动使用；收到 `1000` 或无 code 的 close 时视为对端正常结束。

规则：

- close code 只用于诊断和分类。任何 code 都不取消执行、不改变执行身份、不触发沙盒重建，所有
  分类仍按 `reconnect_ms` 重连（传输决策 D5、D7）。
- 私有 code 取 `4000–4999` 段，后三位沿用对应的 HTTP 状态码，便于在中继日志中直接辨认；中继原样
  转发 close code，这是传输决策 D4 已依赖的前提。
- 中继自身产生的 `1011 "<side> connection lost"` 继续表示另一段未发送 close 就断开，即真实断线或
  进程被强杀；两端开始发送 close 后，这一行只在真正异常时出现。
- 收到的 `4408`、`1001`、`1011` 都是暂时状态，重连可能恢复，归为 `Disconnected`；`1002` 与 `4403`
  表示实现或配置错误，重连不会自行恢复，分别归为 `Protocol` 与 `Mismatch`，便于运维与连接抖动区分。

### D3：Node 不因 Controller 空闲而把会话判为失败

传输决策 D5 规定“应用层心跳与 I/O 期限是端到端存活的唯一依据”，但只有 Node 发送心跳，Controller
空闲时 Node 的读期限必然到期。D1 让两个方向都有心跳，使该规则对称成立：

- Controller 以 `io_timeout_ms` 为 Node 存活期限，依据是 Node 的 `Heartbeat`；
- Node 以 `frame_timeout_ms` 为 Controller 存活期限，依据是 Controller 的任意上行帧，空闲时即 `Heartbeat`。

两端都能在一个期限内独立发现对端消失或连接半开，不依赖对端先关闭连接。

### D4：会话失败分类新增 `Mismatch` 与 `Silent`

传输决策 D7 的 `SessionError` 增加两类，其余不变，仍全部按 `reconnect_ms` 重连：

| 分类 | 来源 | 此前归类 |
|---|---|---|
| `Mismatch` | Controller 自己发现 `node_id` 或能力不符；收到对端的 `4403` | `Protocol` |
| `Silent` | Controller 自己的读或写 I/O 期限到期（此时向 Node 发送 `4408`） | `Disconnected` |

身份或能力不符是部署配置错误，与帧格式错误的处置人和排查路径都不同；I/O 期限到期说明对端或
中间链路沉默，与对端主动关闭或连接被重置不同。拆开后日志可以直接区分“配错了”“对端卡住了”和
“连接断了”，不需要再读错误文本。

## 不变量

1. 已建立的 Controller 会话在任意 `query_interval_ms` 节拍内恰好发送一个上行帧：待查派发的
   `GetExecutionStatus`，或 `Heartbeat`。
2. Node 处理上行 `Heartbeat` 不经过 worker 受理队列，也不产生回复。
3. 上行 `Heartbeat` 的 `controller_id` 与 Node 归属不符时，会话以身份不符结束。
4. 空闲但两端都存活、且 `query_interval_ms` 小于 `frame_timeout_ms` 的会话，不会因读期限到期而结束。
5. WebSocket 会话由本端主动结束时尽力发送 D2 表中对应的 close code；IPC 不受此约束。
6. 任何 close code 都不改变执行身份、不判定执行失败、不销毁沙盒。
7. Controller 把身份或能力不符归为 `Mismatch`，把自身 I/O 期限到期归为 `Silent`，两者都不归为 `Protocol` 或 `Disconnected`。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| Node 只在有未完成请求时施加读超时 | 改动只在 Node 一端，也能消除空闲断开；但 Node 失去发现半开连接的手段，中继或网络断开后控制槽要等 TCP 层超时才释放，代理恢复后首次重连仍会收到 `4409`，与 D3 的目标相反 |
| 依赖 WS ping/pong | ping 只证明单跳存活，中继在代理处终止 WS（传输决策 D5）；IPC 没有 ping；tungstenite 只在读取时回复 ping，读循环阻塞时反而误判 |
| Controller 空闲时发送一条 `GetExecutionStatus` 充当心跳 | 查询需要一个真实的执行身份，并经过 worker 受理队列，Git 占用 worker 时会排队，心跳语义被执行负载拖住 |
| 新增 `heartbeat_ms` 配置项 | 需要迁移所有 Controller 配置和 minicloud 启动器，而查询节拍已能提供同样的上行节奏（D1） |
| 升级协议版本并协商上行心跳 | Controller 与 Node 同时发布，没有独立升级的部署；协商分支没有使用者，只增加测试面 |
| close 只用 `1000`/`1011` 两种 code | 无法区分配置错误与暂时断线，运维仍需翻日志；私有段 code 的成本只是一张表 |

## 本决策未解决的问题

- Node 读循环在等待受理回复时停止读取（问题 2a）：上行心跳会在此期间积压在 socket 缓冲中，读循环
  恢复后立即读到，但若受理阻塞超过 `frame_timeout_ms`，会话仍会因受理期限而结束。由读写解耦处理。
- Git 占用 worker（问题 2b）：需要单独的设计决策。
- `query_interval_ms` 与 `frame_timeout_ms` 的跨进程约束无法自动校验；若将来由 `SandboxProvider`
  同时生成两端配置，再由 Provider 保证。

## 落地结果

- desktop `a767bc7`：`ora-node-protocol` 增加 `ControllerToNodeMessage::Heartbeat`（`ControllerHeartbeat`），
  Controller 会话在无待查派发的节拍发送，Node 在读循环内就地处理并校验 `controller_id`。
- desktop `3d819e6`：`ora-node-transport` 增加 `CloseReason` 与带原因关闭，两端按 D2 关闭，Controller
  增加 `Mismatch`、`Silent` 并在运行时停止时等待 `1001` 发出。
- desktop `46ce0e0`：合并两者，他人心跳改为以 `4403` 关闭。

与批准方案一致，没有偏离。执行、恢复、重放与确认语义不变；Controller 与 Node 必须同版本发布。
