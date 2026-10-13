---
status: implemented
---

# Controller–Node Protocol：统一消息帧与可替换 Transport

Controller 与 Node 共用一套与部署位置无关的协议语义：统一消息帧、身份、执行、事件、结果、
确认与恢复。协议层与传输层分离；当前已落地的传输是本机 IPC 与云端 WebSocket，由
[FrameChannel 传输决策](20260924-frame-channel-with-ipc-and-websocket.md) 固定。Controller
主动管理 Node 的创建与停止，并主动建立控制会话；Node 每次运行只监听一个入口。

在 Cloud 部署中，职责关系固定为：**API Server 是权威状态源，Controller 是 reconcile worker，
Node 是执行环境。** Node 不成为业务状态源，也不直接与 API Server 交换业务数据；Controller
根据 API Server 提供的已授权状态驱动 Node，并将 Node 事实回写给 API Server。

本决策处于 `implemented` 状态。相对原 approved 草案，实际采用的传输与连接方式以 FrameChannel
为准：云端不再以 Controller 对 Node 私有 endpoint 的端到端 mTLS 直连为当前承诺；SSH / stdio /
通用 NetworkTransport 未作为当前系统承诺落地。首条业务闭环是
[clone 指定仓库与分支](../repository/0-clone-selected-repository-branch.md)，不再以 Worktree
专用消息集合作为首条闭环。后续能力扩展见
[会话与 Thread 事件决策](20260928-streamed-thread-events-and-session-commands.md) 等后续 ADR。

## 背景与约束

Desktop 本机 Node 与 Cloud 沙盒 Node 需要同一套执行、事件、重试和恢复语义。按传输分别设计
业务协议会导致语义分裂；把 Node 绑死在单一字节流形态，则无法同时覆盖本机 IPC 与经平台路由的
云端连接。

跨进程、断线、重传和结果未知都必须可对账。业务输入、结果与能力协商按其独立 ADR 定义，不由
传输恢复自动发明新的执行身份。

## 决策

### D1：协议层与 Transport 层分离

```text
Controller–Node Protocol
        │
        └── FrameChannel（完整协议帧）
                │
                ├── IPC（本机 Unix socket / 长度前缀）
                └── WebSocket（云端，经平台路由；一消息一帧）
```

Protocol 只理解消息和会话；Transport 只负责可靠地收发完整协议帧。两种传输映射到相同的会话与
消息语义。帧通道的接口与线上表示由 FrameChannel ADR 细化；根决策只要求：传输可替换，会话逻辑
不随传输分叉。

### D2：统一消息帧与执行身份

所有传输使用明确 framing。一次成功接收恰好得到一条完整、已校验的帧；超过最大帧大小、版本
不兼容或无法解码时必须明确拒绝。

消息至少支持以下语义字段（按消息类型选用，不要求握手等无业务消息携带执行身份）：

```text
protocol_version
message_type
request_id
operation_id
execution_id
sequence
payload
```

`request_id` 关联上游客户端命令，`operation_id` 关联业务操作，`execution_id` 标识一次 Node
执行尝试。`operation_id` 与 `execution_id` 不是别名；执行事件按 `execution_id` 与该执行内的
单调序号识别。

Controller 在派发前将操作、执行、目标 Node 和输入的关联保存到权威存储：Desktop 使用本地
持久化，Cloud 使用 API Server。网络重传、重新领取协调工作及 Controller／Node 重启均保留原
执行身份。结果未知时先对账，不能新建操作或执行身份绕过去重；是否允许新的业务重试，不由
传输恢复自动决定。

### D3：Controller 主动拉起并建立会话；当前传输为 IPC 与 WebSocket

Controller 通过部署适配器申请或启动 Node（本机子进程或云端沙盒）。启动与发现产出与厂商无关的
`NodeEndpoint`；传输层不认识厂商。

**实际采用的连接方式（相对草案的偏离）：**

| 部署 | 连接方式 | 鉴权 |
|---|---|---|
| Desktop / 本机 | Controller 连接 Node 的本机 IPC endpoint | 本机路径权限与会话握手中的归属校验 |
| Cloud | Controller 经平台 WebSocket 路由连接 Node | 平台层凭据与可达性；Node 不做独立传输鉴权 |

Controller 始终主动发起控制会话；Node 不依赖 Controller 进程存活来解释协议角色。草案中的
“Controller 以 mTLS 直连 Node 私有服务 endpoint”、SSH 通道 / stdio 承载，以及 Node 出站回连
兼容连接器，均未作为当前系统承诺落地；若未来部署需要，由后续 ADR 新增 `NodeEndpoint` 变体或
连接器，而不改协议身份与恢复语义。

### D4：身份、能力和会话恢复

每个会话绑定 `ControllerId`、持久 `NodeId`、本次运行的 `NodeIncarnationId`、协议版本和能力
集合。Node 重启必须生成新的运行实例标识。

连接断开时 Controller 暂停向该 Node 派发新任务。重连后双方以 `execution_id` 和事件序号进行
恢复对账；Node 重启改变运行实例身份，但不改变已有执行身份或重放事件的身份。Transport 断开
不能直接把执行标记为失败。

Node 在启动外部操作前持久保存执行身份与输入，并保留未确认的事件和结果。Controller 只有在
权威存储持久接管事实、去重身份及可恢复的后续处理责任后，才能确认对应事件：

| 部署 | 允许向 Node 确认的时点 |
|---|---|
| Desktop | Controller 本地持久化成功后 |
| Cloud | API Server 持久化成功并经 Cloud 内部控制接口确认后 |

Controller 收到消息或在内存中完成去重都不满足确认条件。权威存储不可用或持久化结果未知时，
事件保持未确认；Node 收到确认后才可清理对应重放记录。清理事件不意味着删除执行去重记录。
确认丢失时，Node 重放同一事件，Controller 根据权威存储中的记录重新确认。Cloud 侧提交恢复遵循
[Cloud Internal Control gRPC 契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)。

持久执行记录不等于外部副作用恰好一次。若 Node 在外部副作用发生后、结果落盘前崩溃，必须结合
执行记录和实际资源对账；无法确定结果时保留结果未知状态，不盲目重跑。

#### 状态查询、事件重放与确认的职责

状态查询用于执行对账，回答当前状态及已保留的终态结果；它不携带原事件的 `sequence`，不承担
事件交付或确认职责。`EventAck` 始终确认原事件的精确 `(execution_id, sequence)`。

Node 在会话恢复后主动驱动未确认事件的重放，保持原事件身份与结果。状态查询不会确认事件、
停止重放或触发未确认记录清理。查询回复与重放事件可能以任意顺序到达；Controller 对同一执行
结果的处理必须幂等。

### D5：首条业务闭环与后续消息扩展

首条业务闭环的消息与契约由
[clone 指定仓库与分支](../repository/0-clone-selected-repository-branch.md) 定义，而不是本根决策
中的 Worktree 专用消息集合。根决策只固定：同一执行身份和参数一致时，已完成的执行返回原结果，
执行中的返回当前状态；参数不同则冲突；结果不明确时进入恢复状态，不能猜测成功或自动创建第二次
执行。

插件、Agent 会话与 Revision 等后续消息扩展由后续 ADR 在继承本根决策身份与恢复规则的前提下增量
引入。

### D6：逻辑通道与流量控制

同一物理连接可以复用控制、事件与大体积数据等逻辑优先级。控制和心跳必须优先于大量日志或产物；
缓冲达到上限时必须暂停可暂停生产者或明确失败，不能静默丢失终态。

## 不变量

1. 本机与云端复用同一 Controller–Node Protocol；已采用的传输行为一致。
2. 同一执行的重传复用原 `operation_id` 和 `execution_id`，不重复启动外部操作。
3. Node 身份由 `NodeId` 和 `NodeIncarnationId` 共同确认，不能使用进程 ID 或路径替代。
4. Node 断线、重启或响应丢失不会让 Controller 无证据地标记成功或再次启动执行。
5. Node 内部日志、进程句柄和绝对路径不成为协议的跨部署语义。
6. Node 只在自己的环境中解析和验证 Workspace、Repository 和路径边界。
7. Node 事件在权威存储持久接管后才能确认；Controller 内存去重不构成持久化确认。
8. 执行身份及输入先持久化再启动；运行实例变化和结果重放不改变已有执行与事件身份。
9. 状态查询与事件确认相互独立；未确认事件在会话恢复后重放，查询与重放乱序不重复触发业务处理。

## 为什么不是其他方案

| 方案 | 否决理由 |
|---|---|
| 本地与云端各自定义业务协议 | 重复执行、事件和恢复语义，无法保证跨部署一致 |
| 所有 Node 强制使用单一字节流形态 | 无法同时自然覆盖本机 IPC 与经平台路由的云端连接 |
| Controller 直接把本机路径传给 Node | 远端 Node 无法解释该路径，且越过 Node 的路径授权边界 |
| 只使用一次性请求响应 | 无法承载进度、心跳、确认、重放和取消协调 |
| 云端坚持端到端 mTLS 直连 Node | 与平台 WebSocket 代理拓扑冲突；实际采用平台层鉴权（见 FrameChannel D6） |

## 本决策未解决的问题

- 沙盒生命周期 API 与 `SandboxProvider` 接口；
- Ora 层 bootstrap credential（仅当平台可达性前提不成立时需要）；
- 大文件、终端和产物的独立传输实现；
- 尚未采用的 SSH / stdio / 出站回连等传输变体。
