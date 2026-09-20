---
status: proposed
---

# API Server–Controller Internal Control Protocol：双向控制与状态回写

Cloud 场景中，管理节点 API Server 负责租户隔离、认证授权和业务数据的权威持久化；Rust
Controller 作为子进程负责无租户的调度与 reconcile。两者通过双向 Internal Control API 通信：
API Server 向 Controller 提供已授权的工作和资源快照，Controller 向 API Server 提交接受、状态、
事件和执行结果。Controller 不直接访问 Go 数据库。

本决策处于 `proposed` 状态，覆盖 Controller 子进程与管理节点 API Server 的最小 Worktree 闭环。
它不定义公开 Web API、Controller–Node Protocol 或具体数据库 schema。

Cloud 场景的职责原则是：**API Server 是权威状态源，Controller 是 reconcile worker，Node 是
执行环境。** API Server 保存声明和状态，Controller 根据状态协调工作，Node 只在自身环境中执行
并上报事实。

## 背景与约束

为了复用 Rust 的 `ora-effect`、调度和恢复逻辑，Cloud 不在 Go API Server 中重写 Controller
核心。API Server 必须成为业务状态唯一写入者，而 Controller 需要获取待处理工作、读取一致快照、
领取协调租约并提交结果。

Controller 没有 Tenant 领域概念。API Server 在公开请求和内部接口边界完成租户授权，只提供已经
限定范围的资源和 opaque work item。Controller 的缓存和内存 reconcile 状态可以丢弃并从 API Server
恢复。

## 决策

### D1：Internal Control API 是双向应用协议

API Server 与 Controller 之间使用双向长连接或等价的请求／事件通道，双方都可以发送消息：

```text
API Server → Controller:
  WorkAvailable
  ResourceSnapshot
  NodeAssignment
  Shutdown / Drain

Controller → API Server:
  WorkClaimed
  StatusUpdate
  DomainEvent
  ExecutionRequest
  ReconcileCompleted / ReconcileFailed
```

API Server 不把数据库连接、SQL、租户表结构或 Go 内存对象暴露给 Controller。Controller 也不绕过
Internal Control API 直接写数据库。

### D2：API Server 是持久化权威

API Server 持久化 Project、Workspace、Task、Session、Desired State、Status、Effect 状态、
协调队列、outbox、lease 和 fencing 信息。Controller 提交状态时必须携带资源版本、操作身份和
必要的 fencing token；API Server 以事务校验并原子提交。

Controller 可以维护运行时缓存、连接表和临时调度状态，但这些数据不是恢复依据。API Server 重启
或 Controller 重启后，未完成工作必须能从持久化队列、Desired State 和 Execution 状态重新发现。

API Server 接受 Client Command 时，必须原子保存命令意图、接受结果、`operation_id` 和待处理工作，
再向 Client 返回 `Accepted`，不以 Controller 的内存领取结果作为接受依据。该边界与
[Client–Controller Contract](../../desktop/protocol/0-client-controller-contract.md) 一致。

Controller 向 Node 派发前，必须先让 API Server 持久保存 `operation_id`、`execution_id`、目标
Node 和执行输入的关联。派发响应丢失或 Controller 重启后，从该记录恢复同一次执行；重新领取
协调工作不产生新的执行尝试。API Server 不可用时，不能派发尚未持久记录的新执行。

### D3：Controller 只处理租户无关的已授权对象

Internal Control API 向 Controller 提供全局唯一或 opaque 的资源身份及已经授权的快照。Controller
不得执行租户筛选、跨租户查询或根据客户端身份自行放宽访问范围。

如果 API Server 需要让 Controller 访问某个资源，API Server 必须在返回 work item 或 snapshot 时
已经完成授权和范围限制。Controller 返回的状态和事件由 API Server 重新归属到原资源及其租户。

### D4：消息必须支持领取、幂等和恢复

消息按语义携带工作身份、资源身份与版本、业务操作／执行关联以及事件序号或游标。身份沿用
[Client Contract 的分工](../../desktop/protocol/0-client-controller-contract.md)：`request_id` 关联
原 Client Command，`operation_id` 关联业务操作，`execution_id` 关联 Node 执行尝试。

每次逻辑提交另有稳定的提交身份，以下称为 `submission_id`；名称仅用于说明语义，不固定消息字段。
同一 operation 的领取、进度更新、结果提交是不同提交，不能仅用 `operation_id` 或资源版本去重。
传输重发保留原提交身份和内容。

API Server 验证调用方身份及资源访问范围后，按以下规则处理：

| 提交情况 | 行为 |
|---|---|
| 已处理的提交身份，内容一致 | 返回原提交结果，不重新应用状态变更 |
| 已处理的提交身份，内容不同 | 返回身份冲突，不覆盖原提交 |
| 尚未处理的新提交 | 校验资源版本和所需 fencing token，再原子保存状态变化及提交结果 |
| 新提交的版本或 fencing 校验失败 | 拒绝写入，要求重新读取快照或取得有效协调资格 |

例如，提交基于版本 7 成功写入版本 8，但响应丢失；同一提交重传应返回原成功结果，不能因当前版本
已经是 8 而报版本冲突。返回历史结果不授予新的写入权限；重复领取得到的历史租约也不因此续期。
读取新快照后改变输入再提交，属于新的逻辑提交，但不能因此改变原有业务操作或执行身份。

Controller 与 API Server 任一方断线时，未确认消息必须可以重新获取或重放，不得依赖内存中的
单次回调。不同提交若携带同一 Node 执行事件，还须按该事件的执行身份与序号去重，避免恢复后
重复形成业务事件。

### D5：Controller 的 Effect 复用方式

Cloud 中的 Controller 使用 Rust `ora-effect` planner／reconciler。它通过 API Server 适配的
`EffectRepository` 读取 Desired State、Target、Resource 和 Status，并通过 Controller–Node Protocol
要求 Node 观测或执行。API Server 只负责持久化证据和事务，不复制 Effect 算法。

### D6：Node 结果确认以 API Server 持久化为边界

Controller 收到 Node 事件或结果后，先提交 API Server。API Server 持久保存事实及其去重身份，
并保证后续状态推进和业务事件发布可恢复后，Controller 才能向 Node 确认该事件。状态投影可以
后续推进，但不能只保存在 Controller 内存中。该顺序与
[Controller–Node Protocol](../../node/protocol/0-controller-node-protocol.md) 的确认语义一致。

API Server 不可用、提交被拒绝或提交结果尚未确认时，Node 事件保持未确认。API Server 已提交
但响应丢失时，Controller 重放原提交以取得持久化确认；Controller 在向 Node 确认前崩溃时，新的
Controller 从 Node 重放及 API Server 去重记录恢复。不能先确认 Node，再依赖内存回调补写状态，
否则 Node 清理重放记录后会失去唯一的结果证据。

## 不变量

1. API Server 是 Cloud 场景业务数据的唯一权威写入者。
2. Controller 不直接访问 Go 数据库，不包含 Tenant 领域模型。
3. API Server 返回给 Controller 的对象已经经过租户授权和资源范围限制。
4. 状态提交必须经过版本和 fencing 校验；过期 Controller 不能覆盖新状态。
5. 断线、进程重启和重复消息不会丢失持久化工作，也不会产生重复外部操作。
6. Controller 的内存缓存丢失后，可以仅依赖 API Server 持久状态恢复。
7. 同一逻辑提交的重传返回原结果；同身份不同内容冲突，新的状态变更必须通过版本和 fencing 校验。
8. Node 事件只有在 API Server 持久接管其事实和后续处理责任后才能被确认。
9. 派发前持久保存执行身份和输入；协调重领、重传与进程恢复不创建新的执行尝试。

## 为什么不是其他方案

| 方案 | 否决理由 |
|---|---|
| Controller 直接连接 Go 数据库 | 引入跨语言 schema 耦合，绕过租户边界和 API Server 的事务控制 |
| Go API Server 重写 Effect／调度逻辑 | 产生第二套复杂实现，恢复和 ownership 语义会分叉 |
| API Server 只推送一次性事件 | Controller 重启或断线后无法重建完整状态和欠账工作 |
| Controller 自己携带 Tenant 模型 | 将产品租户概念渗入可复用控制核心，扩大授权和测试边界 |

## 本决策未解决的问题

- 双向通道使用本机 IPC、stdio、Unix Socket 还是其他 Transport；
- work queue、outbox、lease 和 snapshot 的具体消息字段；
- 一个 API Server 运行一个还是多个 Controller 子进程；
- Controller 版本升级和协议兼容策略；
- Cloud API Server 的具体数据库和租户分区实现。
