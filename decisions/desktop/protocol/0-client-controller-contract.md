---
status: superseded
---

# Client–Controller Contract：异步命令、查询与订阅

> 已被取代：本决策由 [Space API 与 Space 级失效事件流：REST 为权威，SSE 只做失效通知](../../cloud/api-boundary/0-space-api-and-event-stream.md) 与 [退役 Controller 过渡 JSON 接口与监听器：可执行入口在任何形态下都不开监听，本机受理只经内嵌句柄](../../controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md) 取代，原因是客户端与控制面边界已分别落到 Space API/事件流与 Controller 内嵌入口，本篇 proposed 契约不再构成当前系统承诺。当前系统承诺以取代者为准。

Client 与 Controller 之间使用统一的应用契约。Client 表达业务意图，Controller 持久化并接管
后续操作；任何可能触发 Node 或其他外部副作用的命令都采用异步接受语义。Desktop 使用本机
IPC 连接独立 Controller，Cloud 的 Web Client 通过 API Server 使用同一语义；传输和公开序列化
格式不改变应用行为。

本决策处于 `proposed` 状态，覆盖 Client–Controller 的最小 Worktree 闭环。它不决定
Controller–Node Protocol、API Server 的具体租户模型、UI 组件或大文件传输实现。

## 背景与约束

桌面是 `Tauri App + Controller + Node` 三进程，Web 场景是 Web Client 连接管理节点 API Server。
创建或删除远端 Worktree 可能跨进程、SSH、集群和网络，Client 请求不能绑定 Node 的执行时间。

当前单体契约中的 `CreateTask`、`DeleteTask`、`GetTask` 和 `GetTaskWorkspace` 已经表达了第一条
业务链路，但同步成功响应无法表示“已接管、执行中、结果未知”等状态，也无法在连接中断后安全
恢复展示。

## 决策

### D1：Command、Query、Subscription 三类操作

Contract 只定义三类应用操作：

| 类型 | 语义 | 最小示例 |
|---|---|---|
| Command | 请求改变业务状态，返回接受结果 | `CreateTask`、`DeleteTask` |
| Query | 读取带版本的当前快照 | `GetTask`、`GetTaskWorkspace` |
| Subscription | 从游标开始接收状态变化 | `SubscribeTask` |

Client 不直接发送 `EnsureWorktree` 或 `RemoveWorktree`；这些属于 Controller–Node Protocol。

### D2：所有副作用命令异步接受

命令接入方在权威存储中原子保存命令意图、`request_id` 对应的接受结果、`operation_id` 和可恢复的
待处理工作后，立即返回接受结果。Desktop 由 Controller 完成本地提交；Cloud 由 API Server
完成提交，不必等待 Controller 领取工作或 Node 执行：

```text
Accepted {
    request_id
    operation_id
    resource_id
    resource_version
}
```

`Accepted` 不表示 Node 已收到、已开始或已完成。Client 使用 Query 或 Subscription 观察
`Provisioning`、`Ready`、`Failed`、`Destroying` 和 `Destroyed` 等状态。

相同 `request_id` 的重传必须返回同一个接受结果；同一身份携带不同 payload 必须返回冲突。
接受响应丢失后，Client 使用原 `request_id` 重传即可找回原 `operation_id` 和资源身份；不能因为
尚未拿到资源身份就新建命令。`resource_version` 是接受时的资源版本，重传不将其替换为当前版本。
只读 Query 可以同步返回当前快照，但快照中的异步状态仍由 Controller 推进。

三份协议使用以下身份分工；这些是语义约定，不要求三条链路使用相同消息格式：

| 身份 | 含义 | 重传与恢复规则 |
|---|---|---|
| `request_id` | 一次逻辑 Client Command | Client 重传保留，映射到同一接受结果 |
| `operation_id` | 接受命令后由控制面接管的业务操作 | 跨协议保持关联，直到该操作结束 |
| `execution_id` | 业务操作中的一次 Node 执行尝试 | 派发重传、断线和进程恢复保留原身份 |

`execution_id` 是控制面与 Node 的执行身份，Client 不负责生成或选择。创建与删除同一个
worktree 是不同业务操作；资源身份不能替代操作身份。执行身份与恢复规则见
[Controller–Node Protocol](../../node/protocol/0-controller-node-protocol.md)。

### D3：第一版最小消息集合

第一版 Client–Controller Contract 只要求：

```text
Commands:
  CreateTask
  DeleteTask

Queries:
  GetTask
  GetTaskWorkspace

Subscriptions:
  SubscribeTask
```

`CreateTask` 至少包含 Project、标题和 base branch；Controller 根据 Project 的 Main Workspace
和 Node 选择结果创建 Task Workspace，并向 Node 发起 Worktree 操作。`DeleteTask` 触发 Task
Workspace 和 task worktree 的异步清理。Client 不传递或解释 Controller 本机、远端 Node 的绝对路径。

### D4：Query 快照和 Subscription 游标必须可衔接

Query 返回状态快照、`resource_version` 和对应订阅范围的 `snapshot_cursor`；Subscription 接受
`after_cursor`。资源版本用于标识资源状态，游标用于定位事件流，Client 不能将两者互换或自行换算。

快照与游标必须来自同一一致性边界：该订阅范围内截至游标的已提交变化已反映在快照中，从
`after_cursor = snapshot_cursor` 续订能收到快照之后的已提交变化，包括 Query 返回后、订阅建立前
发生的变化。服务端不能先读取旧快照，再独立读取较新的游标，否则会跳过两次读取之间的事件。
游标过期、范围不匹配或无法衔接时，服务端返回 `ResyncRequired`，Client 必须重新获取快照。

事件由 Controller（或 Cloud 场景的 API Server 适配器）发布，至少覆盖：

```text
task_created
worktree_provisioning
worktree_ready
worktree_failed
worktree_destroying
worktree_removed
```

Node 内部事件不直接暴露给 Client；ACP、日志和执行细节只有在被 Controller 汇总为应用事件后
才能进入 Client 订阅。上述持久业务事件只能在权威状态提交后发布，并须能从持久记录恢复发布；
Controller 收到尚未持久化的 Node 结果，不代表 Client 已可观察到相应终态。

### D5：传输适配不改变 Contract

Desktop Client 通过 Tauri 平台适配层连接本机 Controller；Web Client 通过 API Server 的 HTTPS
和事件流连接。两者可以使用不同序列化和传输，但必须映射到同一组 Command、Query、Subscription、
Accepted、状态和错误语义。

### D6：Cloud 场景的三层职责

当 Client 通过 Cloud 的 API Server 接入时，职责关系为：**API Server 是权威状态源，Controller
是 reconcile worker，Node 是执行环境。** API Server 负责租户边界和持久化，Controller 不直接访问
数据库，Node 不直接访问 API Server。Desktop 场景可以由本地 Controller 使用 SQLite 持久化适配器，
但仍保持 Controller 调度 Node、Node 执行并上报事实的职责划分。

## 不变量

1. Client Command 的接受结果由 `request_id` 幂等；重传不会创建第二个外部操作。
2. `Accepted` 表示命令意图、接受结果和待处理工作已在权威存储中提交，不表示 Node 已执行。
3. Client 不直接调用 Node，也不依赖 Node 的内部路径、进程句柄或协议帧。
4. Query 快照及其游标来自同一一致性边界；续订不遗漏之后的已提交变化，无法衔接时要求重新同步。
5. 连接断开不改变 Controller 已持久化的业务状态，也不直接把操作标记为失败。
6. API Server 的租户授权发生在公开 Client 边界；Controller Contract 不引入租户领域模型。
7. `request_id`、`operation_id` 和 `execution_id` 分别标识客户端命令、业务操作和执行尝试。

## 为什么不是其他方案

| 方案 | 否决理由 |
|---|---|
| Client 等待 Node 执行完成后同步返回 | 远程执行时间不可控；断线时无法区分超时、执行中和结果未知 |
| Client 直接调用 `EnsureWorktree` | 将 Node 执行细节暴露到产品 API，阻止 Controller 更换调度和执行位置 |
| 每种传输定义一套消息 | Desktop、Web 和未来 Client 无法保持状态和恢复语义一致 |
| 用事件推送代替 Query | 新 Client、断线 Client 和游标失效 Client 无法获得完整当前状态 |

## 本决策未解决的问题

- 完整资源类型、错误码、状态字段和版本演进规则；
- Web API 的具体 HTTP、WebSocket 或 SSE 选择；
- Desktop 本机 IPC 的具体 framing 和认证；
- API Server 与 Controller 内部双向协议；
- 大文件、终端流和产物的独立传输策略。
