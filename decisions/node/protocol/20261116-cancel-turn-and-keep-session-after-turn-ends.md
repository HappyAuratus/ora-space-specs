---
status: proposed
---

# Node 协议增加"取消一轮"命令；一轮以取消或失败结束、且上一轮已确认停下时，会话保留

Controller 新增会话命令 `CancelTurn{command_id, turn_id}`：Node 持久受理后，对正在执行的这一轮向 Agent 发
ACP `session/cancel`，这一轮以 `TurnEnded{cancelled}` 结束，会话保留，排在后面的 `SubmitUserTurn` 照常执行。
同时，选择了新行为的会话在一轮**失败**（ACP prompt 报错、无进展看门狗放弃）后也不再结束，而是只结束这一轮。
Agent 进程退出、ACP 连接断开、上一轮无法确认已经停下、Node 账本或记录不可用等情况仍然结束会话，清单见 D3。
新行为由 Node 声明能力 `turn_cancel`、Controller 在 `StartAgentSession` 中逐会话选择；没有选择的会话和旧对端
保持原行为。

本 ADR 处于 `proposed`，供评审纠偏，尚无实现，也不维护核心测试用例。影响 desktop 的 `crates/node-protocol`
（消息、能力、字段）、`crates/node-db`（受理新命令）与 `apps/ora-node/src/session/`（会话执行）；共享
`crates/agent-runtime` 已有取消当前一轮、取消宽限期与续用 provider 会话的能力，本决策不改变它的规则。
Controller 侧（停止按钮改用新命令、展示一轮的结束）由各 Controller 自己接入，不在本 ADR 内。兼容性边界：
已有消息、已有会话输入和未选择新行为的会话完全不变；不恢复 Node 重启前的会话（协议 D6 不变）；失败原因的
上报不在本 ADR 内（见 D6）。

## 继承与修改

继续[实时预览](20261018-live-thread-previews.md)，会话命令与终态的规则来自
[会话执行与命令](20260928-streamed-thread-events-and-session-commands.md)，结束规则来自
[agent-runtime 根决策](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D6。

| 前序约定 | 本文处理 |
| --- | --- |
| 协议根：能力在握手中协商，执行输入先持久化再启动，同输入重发幂等 | 继承；新增能力 `turn_cancel`，新字段属于执行输入（D5） |
| 会话命令 D4：命令先持久受理再回复 `SessionCommandAccepted`；同一 `command_id` 至多执行一次；用户轮次按受理顺序执行 | 继承；`CancelTurn` 沿用同一受理、幂等与回复规则（D1） |
| 会话命令 D4：`EndSession` 取消当前轮次、丢弃排队轮次并结束会话 | 继承，不变；`EndSession` 优先于 `CancelTurn`（D4） |
| 会话命令 D4：会话已终态时命令回复 `SessionCommandRejected{session_ended}` | 继承；`CancelTurn` 同样适用。新增拒绝原因 `turn_cancel_unsupported`（D1） |
| 会话执行 D2：Thread 事件持久化后才发送，确认与重放规则 | 继承，不变；一轮的结束只以带 `turn_id` 的 `TurnEnded` Thread 事件为准（D4） |
| 协议 D6：Node 重启后会话以 `interrupted` 结束，不恢复 Agent | 继承，不变 |
| 实时预览：旁路不改变权威状态，查看与断开不触发取消 | 继承，不变 |
| agent-runtime D6："轮次失败、超时、连接丢失：记录 `TurnEnded`，会话保持" | 修改。当前实现为"只有成功的 ACP 完成才接受下一轮，失败以 `agent_failed` 结束会话"，与 D6 原文不符。改为：连接丢失结束会话；失败与超时只在选择了新行为、且上一轮已确认停下时保留会话（D2、D3）；未选择的会话保持当前实现的行为 |

## 问题与约束

Node 协议的会话命令只有 `SubmitUserTurn` 与 `EndSession`，没有"只取消当前一轮"。Controller 想停下一轮，只能
`EndSession`，会话随之结束，用户的对话只能新开、之前的上下文全部丢失。

一轮失败同样结束会话：模型服务报错或限流、key 失效、要求重新认证、模型长时间没有输出被看门狗放弃，这些情况下
Agent 进程通常还活着，只是这一轮没能完成，但 Node 以 `agent_failed` 结束整个会话。这样做的原因是：此前失败的
一轮被当作正常完成，调用方看不出这一轮失败，会话停在空闲直到空闲超时，同时一直占着插件租约和模型授权。所以
任何放宽都必须满足：

- 依赖"失败即终态"的调用方（例如以会话终态结算一次运行的 Controller）行为不变；
- 一轮的结束对调用方可观察，不能只是会话静默空闲；
- 上一轮没有确认停下时，不能在同一工作区里开始下一轮，否则两轮可能并发修改同一 checkout；
- 用户的停止与 Agent 自己结束这一轮存在竞争，取消不能误伤用户刚提交的下一轮。

共享运行时已经具备所需的底层能力：取消当前一轮时先以 `cancelled` 应答挂起的权限请求，再发 `session/cancel`，
在宽限期内等待 Agent 对这次 prompt 的回应；回应到达时 provider 会话可以续用，回应没有到达时运行时隔离该
provider 会话，下一轮通过 `session/load`（Agent 支持时）或带 Ora 记录交接的新 provider 会话继续。本决策只
规定 Node 协议与会话执行如何使用这些能力。

## 决策

### D1：`CancelTurn` 命令

`CancelTurn{node_id, command_id, turn_id}`，Controller → Node，与 `SubmitUserTurn`、`EndSession` 同属一个会话执行：

| 目标轮次 `turn_id` 的状态 | Node 行为 | 这一轮的结束 |
| --- | --- | --- |
| 正在执行 | 以 `cancelled` 应答挂起的权限请求，向 Agent 发 ACP `session/cancel`，在宽限期内等待 Agent 回应这次 prompt | `TurnEnded{cancelled}`；宽限期内没有回应时按 D3 结束会话 |
| 已受理、还在排队 | 轮到它时不发给 Agent：照常写入这一轮的用户消息记录，随即写 `TurnEnded{cancelled}` | `TurnEnded{cancelled}` |
| 已经结束，或没有这个 `turn_id` | 不做任何事 | 不产生新记录 |

- 持久化后回复 `SessionCommandAccepted{command_id}`；同一 `command_id` 重复发送只回复受理，内容不同以协议错误
  拒绝，与其他会话命令相同。
- `CancelTurn` 只作用于 `turn_id` 那一轮：不丢弃其他排队轮次，不结束会话；这一轮结束后按受理顺序执行下一轮。
- Controller 只在目标 `SubmitUserTurn` 已被受理之后发送 `CancelTurn`；Node 不为尚未受理的 `turn_id` 保留取消。
- 会话已终态时回复 `SessionCommandRejected{session_ended}`。会话没有选择新行为（D5）时回复
  `SessionCommandRejected{turn_cancel_unsupported}`，不执行，也不改变会话。

为什么带 `turn_id`：用户点停止时，这一轮可能恰好自然结束、下一轮随即开始。"取消当前轮"会落在用户刚提交的
下一轮上；按 `turn_id` 定位则只会让停止变成空操作。为什么排队中的轮次仍写用户消息：调用方已经把这一轮登记为
用户发出的消息，记录里保留它和它的 `TurnEnded`，Thread 才与调用方一致，"每个受理的轮次都有一个结束"也才成立
（不变量 1）。

### D2：一轮以取消或失败结束后会话保留

对选择了新行为的会话，下列结束方式只结束这一轮，会话回到"等待下一轮或结束命令"：

| 这一轮怎么结束 | 记录 |
| --- | --- |
| `CancelTurn`，Agent 在宽限期内回应了这次 prompt | `TurnEnded{cancelled}` |
| ACP `session/prompt` 返回错误（模型请求失败、限流、key 失效、要求重新认证、Agent 内部错误） | `TurnEnded{cancelled}` |
| 无进展看门狗取消后 Agent 回应了这次 prompt，但不再重发（回应不是 `cancelled`，或重发次数用尽） | `TurnEnded{cancelled}` |

- 下一轮在同一个会话执行、同一个 Ora 会话记录上进行。provider 会话仍可用时直接续用；运行时已隔离它时，
  按运行时既有规则通过 `session/load` 恢复，或开新的 provider 会话并交接 Ora 的记录。会话的模型授权、插件租约
  和工作区在会话结束前一直保留，与一轮成功完成后相同。
- ACP 错误回应不等于 provider 会话仍然可用：Agent 可能在报错的同时关闭了这个 provider 会话（例如之后的
  prompt 回"会话已结束"或"会话不存在"）。Node 把这类回应视为 provider 会话已失效，下一轮先按运行时既有规则
  `session/load` 恢复或交接记录重建，而不是把下一轮发到已失效的会话上；恢复和重建都失败时按 D3 结束会话。
- 失败的一轮与取消的一轮在 Thread 里都记为 `TurnEnded{cancelled}`，本决策不区分两者（见 D6）。

为什么要求"Agent 回应了这次 prompt"：回应是 Agent 停止处理这一轮的唯一证据。ACP 的更新不带请求身份，
没有回应时，Node 无法区分迟到的更新属于上一轮还是下一轮，也无法保证上一轮的工具不再修改 checkout。

**取消宽限期**：`CancelTurn` 等待 Agent 回应的上限记为 G，是 Node 对 Controller 的承诺：G 到期仍没有回应时，
Node 按 D3 结束会话。G 覆盖所支持 Agent 正常取消的最坏耗时，而不是 Agent 卡死时的兜底：

| Agent | 正常取消的最坏耗时 | 卡死时的兜底 |
| --- | --- | --- |
| OpenCode | 中断后自己最多等 5 秒，shell 工具再过 3 秒强制终止，约 8 秒 | 同左 |
| Claude Code | 中断通常很快 | SDK 无响应时 30 秒后强制回 `cancelled`，此时 provider 会话状态本身可疑 |

第一版暂定 G 为 15 秒，在实现阶段用真实 Agent 实测后定值，写入本 ADR 的 `approved` 版本；当前共享运行时的
5 秒短于 OpenCode 的正常取消耗时，不能沿用。G 只是上限，Agent 回应后立即结束等待，用户不会因此等满 G
（D4 允许在等待期间提交下一轮）。看门狗取消的宽限期不随之加长，仍按运行时既有规则，以免模型无输出时的重发
被拖慢；看门狗取消没有得到回应时同样按 D3 结束会话。

### D3：必须结束会话的情况

**Node 结束会话**，发 `AgentSessionEnded`（选择了新行为的会话同样适用）：

| 情况 | 终态 |
| --- | --- |
| Agent 进程退出、崩溃，或 Node 与 Agent 之间的 ACP 连接断开 | `agent_failed`，detail 为既有的 `agent_unavailable` 或 `agent_stream_closed` |
| 取消或看门狗取消后，宽限期内 Agent 没有回应这次 prompt | `agent_failed`，detail `turn_stop_unconfirmed` |
| 运行时不再受理新的一轮（会话记录已降级、轮次未被受理） | `agent_failed`，detail 为既有代码 |
| Thread 镜像不可用 | `agent_failed`，detail `thread_unavailable` |
| Node 账本读写失败，包括命令结算失败 | `agent_failed`，detail `ledger_unavailable` |
| 模型授权被撤销 | `agent_failed`，detail `model_access_revoked` |
| `EndSession` | 命令给出的原因 |
| Node 重启 | `interrupted`（协议 D6） |

**Controller 结束会话**：Controller 一侧的故障由 Controller 判断，需要时发 `EndSession`，或按 Controller 自己的
决策把会话当作结束，本决策不改变这些规则。包括：与 Node 的连接断开且不再恢复、Thread 事件在 Controller 一侧
持久化失败、Node 回复 `SessionCommandRejected`、Node 回复的身份与会话对不上。

没有选择新行为的会话，D2 中的失败仍以 `agent_failed` 结束会话，与本决策之前一致。

**为什么没有回应时结束会话是安全的**：Node 结束会话时，先停止这个会话的 Agent 插件及其整棵进程树，再写
`AgentSessionEnded`。所以终态到达 Controller 时，Agent 已经不可能再修改 checkout，结束会话本身就是"上一轮
已停下"的兜底证据。停下之前 Agent 已经做出的修改留在 checkout 中，由会话结束后的 Revision 交付照常保存，
不会丢失。不确定的只是这一轮里没有收到终态的工具调用：它们在记录中保持 `pending` 或 `in_progress`，由
`TurnEnded{cancelled}` 说明这一轮被打断，不被记成成功。代价是用户的对话结束，而不是工作区与记录不一致。

### D4：与 `EndSession`、工具调用和事件回执的先后关系

- **`EndSession` 优先**：队列中任意位置的 `EndSession` 仍优先于其前面的命令；与 `CancelTurn` 同时存在时按
  `EndSession` 结束会话，`CancelTurn` 随之结算，不另产生记录。`EndSession` 之后到达的 `CancelTurn` 回复
  `session_ended`。已持久受理的 `EndSession` 与一轮失败同时发生时，会话以 `EndSession` 的原因结束（沿用当前
  实现）。
- **先到者为准**：`CancelTurn` 到达时这一轮已经以 Agent 自己的停止原因结束，`TurnEnded` 保持那个原因，取消
  不改写它。同一 `turn_id` 的多个 `CancelTurn` 只有第一个可能生效。
- **工具调用**：取消前先以 `cancelled` 应答这一轮挂起的权限请求。Agent 回应之前发出的更新都记入这一轮；回应
  之后到达的迟到更新不记入任何一轮，也不送进下一轮。这一轮结束时仍处于 `pending` 或 `in_progress` 的工具调用
  保持原状态，由 `TurnEnded{cancelled}` 说明这一轮是被打断的，不推断为成功或失败。
- **事件回执**：`SessionCommandAccepted` 只表示命令已持久受理，不表示这一轮已经停下。这一轮的结束只以带
  `turn_id` 的 `TurnEnded` Thread 事件为准，它按既有规则持久化后发送、确认前保留、重连后重放。Controller 可以
  在收到 `TurnEnded` 之前提交下一轮，下一轮按受理顺序排在被取消的一轮之后执行。
- **Controller 的等待上限**：Controller 发出 `CancelTurn` 后等待结论的上限 T 必须大于 G 加上 Node 停止 Agent
  进程树、写入并发送终态所需的时间。这样 Controller 总能收到两种确定结论之一：`TurnEnded`（这一轮已停下，
  会话保留）或 `AgentSessionEnded`（Agent 已停止，会话结束）。T 更短时，Controller 会在 Node 仍在正常停止
  Agent 时超时，只能自行猜测：当作"已停下"可能让下一轮或新会话与仍在运行的旧 Agent 同时修改同一 checkout；
  当作"停止未确认"则把一次可以确认的停止变成故障。只有 Node 本身失联时才会等满 T，此时按 Controller 的连接
  断开规则处理。

### D5：能力协商与逐会话选择

- Node 在握手的 `capabilities` 中声明 `turn_cancel`，表示它支持 `CancelTurn` 和 D2 的会话保留。
- `AgentSessionSpec` 增加可选字段 `keep_session_on_turn_end: bool`，省略即 `false`。Controller 只在当前连接的
  握手声明了 `turn_cancel` 时设置为 `true`；它和其他输入一样先持久化，同一执行重发时参与输入比较。
- 字段为 `true` 的会话：接受 `CancelTurn`，按 D2、D3 处理一轮的结束。字段省略或为 `false` 的会话：拒绝
  `CancelTurn`（`turn_cancel_unsupported`），一轮失败仍结束会话。
- 旧 Node 不声明 `turn_cancel`，且 `StartAgentSession` 使用 `deny_unknown_fields`：Controller 不能向它发送新字段
  或 `CancelTurn`，停止仍用 `EndSession`。旧 Controller 不设置新字段：Node 对它的会话保持原行为。
- 新命令写入 Node 账本的会话命令表；需要时账本 schema 前向迁移，旧程序拒绝打开新库（沿用 Node 升级规则）。

为什么按会话选择，而不是在握手里打开（类似实时预览的 `Hello.thread_previews`）：会话比连接活得久，重连可能
连到另一个 Controller 版本。选择写进执行输入后，同一会话从开始到结束只有一套规则，取消、失败和恢复时都不需要
再看当前连接是谁。

### D6：与失败原因上报的边界

本决策只规定一轮以取消或失败结束时会话是否保留，以及调用方如何观察这一轮结束。它不区分"取消"和"失败"，也不
携带失败原因：两者在 Thread 中都是 `TurnEnded{cancelled}`，用户在调用方看到的是"这一轮已结束"。Controller 可以
用"自己是否为这一轮发过 `CancelTurn`"判断这是不是用户请求的取消，但不能据此认定其余的结束都是失败，也拿不到
原因。区分取消与失败、在协议和历史记录中携带失败原因、看门狗取消后 Agent 回 `end_turn` 时是重发还是以"无进展"
失败，都由后续决策承担；它们建立在本决策的会话保留之上。

## 不变量

1. 一个被受理的 `SubmitUserTurn` 在会话结束前恰好产生一个带其 `turn_id` 的 `TurnEnded`；会话先结束时，它按
   `EndSession` 的既有规则被丢弃。
2. `CancelTurn` 只影响 `turn_id` 那一轮，不丢弃其他轮次，不结束会话。
3. `EndSession` 优先于 `CancelTurn`；会话已终态后命令不复活会话。
4. 会话在一轮取消或失败后保留，当且仅当会话选择了新行为、Agent 连接仍在、且 Agent 回应了这一轮的 prompt。
5. 上一轮被确认停下之前，Node 不向 Agent 发送下一轮。
6. 没有选择新行为的会话，行为与本决策之前完全一致。
7. `SessionCommandAccepted` 不代表这一轮已停下；一轮的结束只以 `TurnEnded` Thread 事件为准。
8. Node 收到 `CancelTurn` 后，在 G 加停止 Agent 所需的时间内，一定给出 `TurnEnded` 或 `AgentSessionEnded`
   之一；发出 `AgentSessionEnded` 时，Agent 的进程树已经停止。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| `EndSession` 增加"只取消一轮"的原因 | 不新增消息；但 `EndSession` 的规则是丢弃排队轮次并产生终态，一个命令两种语义会让"终态之后没有事件"和命令幂等的判断都依赖原因字段 |
| `CancelTurn` 不带 `turn_id`，取消"当前"一轮 | Controller 更简单；但停止与这一轮自然结束、下一轮开始存在竞争，会取消用户刚提交的下一轮（D1） |
| 失败后一律保留会话，不协商 | 不改输入；但依赖"失败即终态"的调用方会看到会话停在空闲直到超时，并一直占用租约和模型授权，这正是当前实现改为失败即结束的原因 |
| 在握手中打开新行为 | 与实时预览一致；但会话跨连接存在，重连到不同版本的 Controller 会让同一会话前后规则不同（D5） |
| 取消没有得到 Agent 回应时也保留会话 | 用户少遇到一次对话结束；但 Agent 可能仍在执行上一轮，下一轮与它并发修改同一 checkout，迟到的更新也会混进下一轮 |
| 取消没有得到回应时只重启 Agent 进程、保留会话：停止进程树后，下一轮经 `session/load` 或交接记录重建 provider 会话 | 卡死时对话也能保住，停止进程树同样证明上一轮已停下，共享运行时也已有"进程被替换后下一轮重新接上"的逻辑；但与"Agent 进程退出即结束会话"的规则相反，要另外处理重启后的模型授权、Git 访问和插件租约，重建出的上下文也不再是原来的 provider 会话。第一版不采用，可作为后续决策 |
| 宽限期取 30 秒，覆盖 Claude Code 卡死时的强制回应 | 卡死的 Claude Code 也能保留会话；但为一个 provider 会话本身已可疑的情况让用户多等，并迫使每个 Controller 把等待上限（D4）放宽到 30 秒以上，卡死时用户要等更久才知道对话已结束 |
| 失败后结束会话，由 Controller 新开会话并重放历史 | Node 不变；但同一 Agent 进程仍可用时要重建上下文，成本高，并把会话连续性的责任推给每个 Controller |
| Agent 进程退出后由运行时重启进程、恢复会话 | 用户体验更好；依赖 Agent 支持 `session/load`，并且要证明旧进程里的这一轮没有留下副作用，另立决策 |

## 已知风险与待实测

按 OpenCode（`@opencode/cli` 2.0.32）与 Claude Code（`@agentclientprotocol/claude-agent-acp` 0.107.0）的 ACP
实现核对：取消、模型请求错误、要求重新认证、看门狗取消后回 `end_turn`，以及取消时有进行中的工具调用或挂起的
权限请求，这些情况之后两者都能在同一个 ACP 会话上继续，上下文保留。下列风险只靠源码不能排除，在实现阶段用
真实 Agent 实测，结论写入 `approved` 版本：

| 风险 | 影响 | 处理 |
| --- | --- | --- |
| `cancelled` 回应只证明主轮次停下：OpenCode 的子代理、Claude Code 的后台任务可能在回应之后继续运行，并发出轮次之外的更新和权限请求 | 不变量 5 只对主轮次成立；后台工作可能在下一轮开始后仍修改 checkout | 迟到的更新不记录、权限请求以 `cancelled` 应答（D4）；实测后台工作是否随取消停止，必要时在 D2 中收紧保留条件 |
| OpenCode 内部异常时，ACP 层返回错误，但 server 端的执行可能仍在进行，下一轮可能被并进去 | 下一轮混入上一轮未停的工作 | 实测确认；成立时把这类错误归入 D3"无法确认已停下" |
| Claude Code 内部或传输错误后关闭 provider 会话，Agent 进程仍在 | 下一轮发到已失效的会话 | 按 D2 识别后 `session/load` 恢复 |
| G 短于 Agent 正常取消的耗时 | 本可续用的会话被结束 | 按 D2 实测 OpenCode 取消运行中 shell 工具的耗时后定 G |

## 本决策未解决的问题

- 区分取消与失败、上报失败原因，以及看门狗取消后 Agent 回 `end_turn` 的处理（D6）。
- Agent 进程退出后的会话恢复，以及 Node 重启后的会话恢复（协议 D6）。
- 各 Controller 如何展示被取消或失败的一轮、何时自动发 `EndSession`。

## 落地顺序

1. **协议与账本**：`crates/node-protocol` 增加 `CancelTurn` 消息、`turn_cancel` 能力、`keep_session_on_turn_end`
   字段和拒绝原因 `turn_cancel_unsupported`，补校验与往返测试；`crates/node-db` 受理并持久化新命令。完成条件：
   旧形状的消息与输入编解码不变，新字段省略时输入比较与之前相同。
2. **Node 会话执行**：`apps/ora-node/src/session/` 按 D1–D4 执行 `CancelTurn`，并按 D2、D3 区分一轮失败与必须
   结束会话的情况。完成条件：取消后与失败后同一会话可以接着 `SubmitUserTurn`；取消时进行中的工具调用与权限
   请求按 D4 收尾；D3 中每一种情况仍结束会话；未选择新行为的会话与当前行为一致；“已知风险与待实测”中的
   每一项都有真实 Agent 的实测结论。
3. **Controller 接入**：各 Controller 在看到 `turn_cancel` 后为新会话选择新行为，停止改用 `CancelTurn`，并按 D4
   设置大于 G 加停止耗时的等待上限。

保持不变：已有消息、已有会话、未选择新行为的会话，以及 Node 重启后的 `interrupted`。从第 2 步开始改变：选择了
新行为的会话在一轮取消或失败后保留。
