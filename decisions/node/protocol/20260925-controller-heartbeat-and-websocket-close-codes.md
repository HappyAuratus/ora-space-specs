---
status: proposed
---

# Controller 上行心跳作为 Node 读期限的存活依据，WebSocket 会话以约定 close code 结束

Controller 会话在空闲时向 Node 发送应用层 `Heartbeat`，Node 保留每帧读期限 `frame_timeout_ms`，
把它作为“Controller 仍在、连接仍通”的依据。空闲会话不再每约 11 s 被 Node 断开并重建两段连接；
半开连接会在一个读期限内被 Node 发现并释放控制槽。同时固定 WebSocket 会话结束时两端发送的
close code，使中继与 Controller 能区分进程停止、协议错误、身份不符、存活超时与内部失败，而不是
一律看到中继代发的 `1011 "<side> connection lost"`。

本决策处于 `proposed` 状态，等待 Eric 在两种方向中确认：本文采用“Controller 上行心跳”，备选
“Node 只在有未完成请求时施加读超时”见替代方案。它继承
[帧通道传输决策](20260924-frame-channel-with-ipc-and-websocket.md)，修改其 D5（心跳与连接存活）
与 D7（会话失败分类）中与 close code 相关的行，其余不变。涉及 `ora-node-protocol` 的
Controller→Node 消息、Controller 会话、Node 会话读循环和 `ora-node-transport` 的 WS 关闭接口。
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
| D7：`Busy` 来自 `4409`，其余非 normal close 归为 `Disconnected` | 修改：`4403`、`1002` 归为 `Protocol`，其余不变（D2） |
| D7：所有分类都按 `reconnect_ms` 重试 | 继承，不变 |
| 根 D4：会话绑定 `ControllerId` 与 Node 身份 | 继承；上行心跳携带 `ControllerId`，不符即协议错误 |

## 决策

### D1：Controller 在每个查询节拍上恰好发送一个上行帧

Controller 会话已有每 `query_interval_ms` 一次的查询节拍。每个节拍发送恰好一个上行帧：

- 有待确认的派发时，照旧轮换发送一条 `GetExecutionStatus`；
- 没有时，发送一条 `Heartbeat { controller_id }`。

新增的 `Heartbeat` 是 `ControllerToNodeMessage` 的一个变体，线上 `message_type` 为 `heartbeat`，
负载只有 `controller_id`；它与 Node→Controller 的 `Heartbeat` 分属两个方向的消息枚举，互不混淆。

Node 收到上行 `Heartbeat` 时：

- 在会话读循环内直接处理，不进入 worker 受理队列，因此不会排在 Git 或 SQLite 之后；
- `controller_id` 必须等于本 Node 的归属 Controller，否则按协议错误结束会话；
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

WebSocket 会话结束时，结束方尽力发送一个 close frame。发送受该端现有 I/O 期限约束，失败不改变
会话结果、不影响执行责任，也不阻塞重连。IPC 没有 close frame，照旧直接关闭 socket。

| code | 含义 | 发送方与时机 | Controller 分类 |
|---|---|---|---|
| `1000` | 正常结束 | 本文不主动使用；收到时与无 code 的 close 一样视为对端正常结束 | `Disconnected` |
| `1001` | 进程停止 | Controller 或 Node 正常停止时 | `Disconnected` |
| `1002` | 协议错误 | 收到非法帧或消息、握手顺序错误（如首帧不是 `Hello`） | `Protocol` |
| `1011` | 内部失败 | 本端持久化失败、受理队列不可用等与对端无关的原因 | `Disconnected` |
| `4403` | 身份不符 | Node：`ControllerId` 不属于本 Node；Controller：`NodeId` 或能力不符 | `Protocol` |
| `4408` | 存活超时 | 期限内未收到对端任何帧（含心跳） | `Disconnected` |
| `4409` | 会话占用 | Node 已有有效会话（传输决策 D4，不变） | `Busy` |

规则：

- close code 只用于诊断和分类。任何 code 都不取消执行、不改变执行身份、不触发沙盒重建，所有
  分类仍按 `reconnect_ms` 重连（传输决策 D5、D7）。
- 私有 code 取 `4000–4999` 段，按 HTTP 状态码的后三位命名，便于在中继日志中直接辨认；中继原样
  转发 close code，这是传输决策 D4 已依赖的前提。
- 中继自身产生的 `1011 "<side> connection lost"` 继续表示另一段未发送 close 就断开，即真实断线或
  进程被强杀；两端开始发送 close 后，这一行只在真正异常时出现。
- `4403` 与 `1002` 归为 `Protocol`：它们表示配置或实现错误，重连不会自行恢复，运维需要与连接抖动
  区分；`4408` 与 `1001`、`1011` 归为 `Disconnected`：它们是暂时状态，重连可能恢复。

### D3：Node 不因 Controller 空闲而把会话判为失败

传输决策 D5 规定“应用层心跳与 I/O 期限是端到端存活的唯一依据”，但只有 Node 发送心跳，Controller
空闲时 Node 的读期限必然到期。D1 让两个方向都有心跳，使该规则对称成立：

- Controller 以 `io_timeout_ms` 为 Node 存活期限，依据是 Node 的 `Heartbeat`；
- Node 以 `frame_timeout_ms` 为 Controller 存活期限，依据是 Controller 的任意上行帧，空闲时即 `Heartbeat`。

两端都能在一个期限内独立发现对端消失或连接半开，不依赖对端先关闭连接。

## 不变量

1. 已建立的 Controller 会话在任意 `query_interval_ms` 节拍内恰好发送一个上行帧：待查派发的
   `GetExecutionStatus`，或 `Heartbeat`。
2. Node 处理上行 `Heartbeat` 不经过 worker 受理队列，也不产生回复。
3. 上行 `Heartbeat` 的 `controller_id` 与 Node 归属不符时，会话以协议错误结束。
4. 空闲但两端都存活、且 `query_interval_ms` 小于 `frame_timeout_ms` 的会话，不会因读期限到期而结束。
5. WebSocket 会话由本端结束时尽力发送 D2 表中对应的 close code；IPC 不受此约束。
6. 任何 close code 都不改变执行身份、不判定执行失败、不销毁沙盒。

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

## 落地顺序

1. **协议与 Controller**：`ora-node-protocol` 增加上行 `Heartbeat`；Controller 会话在空闲节拍发送。
   完成条件：Controller 会话单测证明空闲节拍发出 `Heartbeat`、有待查派发时只发查询。
2. **Node**：读循环就地处理上行心跳并校验 `controller_id`。完成条件：真实 Node 在读期限远小于空闲
   时长的配置下，经 IPC 与 WS 维持单一会话不重建；`controller_id` 不符时断开。
3. **close code**：`ora-node-transport` 增加带 code 关闭的接口，两端按 D2 发送，Controller 按 D2 分类。
   可与 1、2 并行实现。

步骤 1、2 必须同时发布（混合版本不支持）；步骤 3 独立。从步骤 2 起，空闲会话不再周期性重建；
执行、恢复、重放与确认语义保持不变。
