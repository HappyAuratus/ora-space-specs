---
status: implemented
---

# Node 协议增加插件、Agent 会话与 Revision 交付执行：会话执行发出有序的非终态 Thread 事件，并接受会话命令

Controller–Node 协议以新增能力与消息的方式承载第二个最小闭环。三类新执行沿用既有的执行身份、派发前
持久化、事件序号、`EventAck` 与重放规则；其中 Agent 会话执行在终态之前持续发出**非终态 Thread 事件**，
这些事件与终态事件共享同一执行内的序号空间、同样持久保留直到确认。会话执行在运行期间接受
`SubmitUserTurn`、`EndSession` 两种命令，Node 持久受理后回复确认。上传授权是唯一不持久化的消息。

本 ADR 处于 `implemented`，消息与校验在 desktop 的 `crates/node-protocol`，账本在 `crates/node-db`，会话处理在
`apps/ora-node/src/service/`；继承 [协议根决策](0-controller-node-protocol.md)、
[FrameChannel 传输](20260924-frame-channel-with-ipc-and-websocket.md) 与
[Controller 心跳与 close code](20260925-controller-heartbeat-and-websocket-close-codes.md)。执行的业务语义由
[plugin-runtime](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md)、
[agent-runtime](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md)、
[revision](../revision/0-internal-snapshot-and-incremental-bundle-upload.md) 三个根决策承担；Controller 侧见
[Controller session 根决策](../../controller/session/0-controller-relays-agent-sessions.md)。兼容性边界：已有
消息与 clone 行为不变；Node 账本 schema 前向迁移，旧程序拒绝打开新库（沿用 Node 升级规则）。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D2：`operation_id`/`execution_id`/`sequence` 身份；执行输入先持久化再启动 | 继承，扩展到新执行 |
| 根 D4：事件持久接管后才确认，未确认事件在重连后按原身份重放；查询与确认独立 | 继承；非终态事件适用同样规则（D2） |
| 根 D5 与 clone 契约：执行只有一个终态事件 | 修改：会话执行在终态前有任意多个非终态事件（D2） |
| 根 D6：控制与心跳优先于大量事件，不能静默丢失 | 继承；非终态事件有在途窗口与磁盘保留（D3） |
| 根不变量 5、6：Controller 不传 Node 本地路径 | 继承；checkout 以 clone 执行 ID 引用（D1） |
| 传输：一帧 16 MiB 上限 | 继承；单个 Thread 记录另限 256 KiB（D2） |
| 握手能力 `repository_clone`、`worktree_execution` | 修改：新增 `plugin_install`、`agent_session`、`revision_delivery`（D1）；`runtime_control`、`model_proxy`、`revision_restore` 由各自的后续决策新增 |

## 决策

### D1：新执行与能力

| 能力 | Controller → Node | Node → Controller 终态事件 |
|---|---|---|
| `plugin_install` | `InstallPlugins`、`RemovePlugins` | `PluginsResult`：`plugins_completed{items[]}` 或 `plugins_failed{plugin_root_unavailable \| interrupted}` |
| `agent_session` | `StartAgentSession{agent_plugin_id, agent_plugin_version, checkout_execution_id, git_identity, initial_turn}` | `AgentSessionEnded{reason, detail}` |
| `revision_delivery` | `DeliverRevision{session_execution_id, checkout_execution_id, base_commit, revision_ref, bundle_key, history_key}` | `RevisionResult{delivered \| unchanged \| failed}` |

- 输入字段与 Cloud 契约一一对应；执行输入在 Node 账本中持久化后才开始执行，同一执行同输入重发是幂等的，
  不同输入冲突，与 clone 相同。
- `checkout_execution_id` 必须引用本 Node 上一个成功的 clone 执行；Node 从账本解析 checkout 路径，
  找不到时以 `checkout_unavailable` 失败，不猜测路径。
- `AgentSessionEnded.reason`：`user_ended | idle_timeout | cancelled | agent_failed | interrupted`。

生产接线沿用已有 `RuntimeControl` 的受控执行入口：`ControlledStartAgentSession{binding, command}`
绑定已登记的会话输入及精确的 Node、incarnation、operation/execution 身份。受控部署拒绝裸
`StartAgentSession`；Node 在持久化与实际启动前检查许可，重复输入不重新启动 Agent。
交付同样以 `ControlledDeliverRevision{binding, command}` 进入，许可必须精确对应交付的 execution、node
operation 与 Node。
`SubmitUserTurn` / `EndSession` 使用既有消息形状，由当前绑定验证原执行的 control scope 和 incarnation；
同一 control epoch 可刷新许可，关闭、过期或 scope 改变后不能受理新命令。已终态的命令仍返回
`session_ended`，不复活执行。Controller 身份与执行归属检查先于上述入口。

### D2：会话执行的非终态 Thread 事件

`ThreadEvent{operation_id, execution_id, sequence, turn_id?, record}`：

- `record` 是 `ora-history` 一条定型记录的 JSON；`turn_id` 在该记录属于某个用户轮次时给出。
- 序号在执行内从 1 单调递增、无洞，Thread 事件与终态事件共享序号空间；终态事件是最后一个，之后 Node
  不再发出该执行的任何事件。
- 每个事件在写入 Node 账本后才发送；未确认前保留，重连后按序号从最小未确认者开始重放，内容与身份不变。
  `EventAck` 仍确认精确的 `(execution_id, sequence)`。
- 单条 `record` 超过 256 KiB 时，Node 在事件中以截断版本替代并标注 `truncated: true`；会话 JSONL 保留原文。
  这样一条巨大的工具输出不会挤占控制帧，也不会超过 Cloud 的条目上限。

### D3：事件在途窗口与磁盘保留

Node 同一执行最多有 256 个已发送未确认的事件；达到窗口时暂停发送（不暂停记录），事件留在账本中，收到
确认后继续。Agent 不因窗口满而被暂停。第一版不设账本的磁盘上限：会话事件在确认后即可清理，只有长时间
无法与 Controller 通信时才会积累。

为什么不丢弃：Thread 是用户可见的对话记录，静默丢失会让对话缺段；根 D6 已规定不能静默丢失终态，Thread
事件同理，只是允许慢。

### D4：会话命令

| 命令 | 规则 |
|---|---|
| `SubmitUserTurn{command_id, turn_id, content}` | 持久化后回复 `SessionCommandAccepted{command_id}`；按受理顺序排队，当前轮次结束后执行；执行时写入的用户消息记录带该 `turn_id` |
| `EndSession{command_id, reason}` | 持久化后回复受理；取消当前轮次（若有），丢弃排队的用户轮次，停止 Agent，完成会话记录后发终态 `AgentSessionEnded{reason}` |

- 同一 `command_id` 重复发送只回复受理，不重复执行；同一 `command_id` 不同内容以协议错误拒绝。
- 会话执行已有终态时，命令回复 `SessionCommandRejected{command_id, reason: session_ended}`，Controller
  照常登记投递（Cloud 以会话终态为准，排队未执行的轮次在 Cloud 标为 `discarded`）。
- `SessionCommandAccepted`/`Rejected` 是回复，不是事件，不带序号、不需要确认；丢失后 Controller 重发命令即可
  重新得到回复。

### D5：上传授权是非持久消息

- `UploadGrant{execution_id, grants[{object_key, url, method, headers, expires_at}]}`：Node 只保存在内存中，
  不写账本、不写日志。
- Node 需要上传而没有有效授权（未收到、过期、Node 重启后丢失）时发送 `UploadGrantNeeded{execution_id}`，
  并等待新授权；等待期间交付执行保持 `Running`。
- `UploadGrantNeeded` 携带 `checksums`，列出每个仍需上传的对象键及 Node 在首个 PUT 前
  固定的 SHA-256，Controller 据此申请绑定校验和的授权（Cloud gRPC 决策 D4 的 `checksums`）。Controller 不在
  派发时主动申请，Node 在等待授权期间于每个新连接上重发该消息，并按有界间隔重复。
- `UploadGrant` 与 `UploadGrantNeeded` 都不携带序号、不需要确认。

### D6：Node 重启中断会话

Node 重启后，没有终态的会话执行不会恢复 Agent 进程：provider 会话与 Agent 进程状态随进程终止丢失，
恢复出的会话无法保证与记录一致。恢复时 Node 关闭该执行的进程 scope、封口会话记录，然后发出
`AgentSessionEnded{reason: interrupted}`。会话 JSONL 与工作区修改都保留，后续交付照常保存它们。

## 不变量

1. 新执行沿用执行身份、派发前持久化、确认与重放规则。
2. 一个执行内事件序号连续；终态事件之后没有事件。
3. 任何事件在 Node 账本持久化后才发送，确认前不删除。
4. 同一 `command_id` 至多执行一次；用户轮次按受理顺序执行。
5. 上传授权只存在于 Node 内存。
6. Controller 不向 Node 传递 Node 本地路径。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Thread 事件至多一次、不持久 | 实现简单、无账本增长；但断线就丢对话记录，而用户要依靠 Thread 对话 |
| 每条 Thread 事件单独成一个执行 | 复用“一执行一终态”；执行数量随消息线性增长，派发前登记的成本不可接受 |
| 命令作为事件（带序号、需确认）的反方向 | 与事件对称；Controller 方向已有 Cloud 的持久命令表与投递登记，再加一层序号是重复的持久化 |
| 上传授权放入 `DeliverRevision` 输入 | 少一个消息；授权过期后重发同一执行即成冲突（Cloud Revision D3） |
| Node 重启后恢复 Agent 会话 | 用户体验更好；依赖 provider 支持 `session/load` 且需持久化进程与轮次状态，另立决策 |

## 本决策未解决的问题

- token 级增量的非持久旁路。
- 账本磁盘上限与长时间失联时的处理。
- Node 重启后恢复 Agent 会话。

## 落地

`crates/node-protocol` 增加消息与校验，Node 账本前向迁移（执行种类、事件表、命令表），Controller 与 Node 的
会话处理接线。在途窗口与截断上限分别是 `apps/ora-node/src/service/delivery.rs` 的 `WINDOW` 与
`MAX_THREAD_RECORD_BYTES`。协议消息往返测试不构成恢复保证的证据，恢复由 Node/Controller 测试承担。保持不变：
已有消息与 clone 行为。开始改变：Node 可以执行插件安装、Agent 会话与 Revision 交付。
