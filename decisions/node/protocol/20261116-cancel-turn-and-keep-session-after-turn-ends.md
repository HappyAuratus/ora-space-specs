---
status: proposed
---

# Node 协议增加"取消一轮"命令；一轮以取消或失败结束、且 Agent 已静止时，会话保留

Controller 新增会话命令 `CancelTurn{command_id, turn_id}`。Node 持久受理后，在有界时间内给出这次取消的确定
结论：目标轮次正在执行时，向 Agent 发 ACP `session/cancel`，这一轮以 `TurnEnded{cancelled}` 结束；目标轮次
还在排队时直接丢弃；目标轮次已经结束或不存在时不做任何事。每种情况都以新的非终态事件
`TurnCancelSettled` 或会话终态告知 Controller。选择了新行为的会话，在一轮以取消或失败（ACP prompt 报错、
无进展看门狗放弃）结束后，只有 Agent 满足可验证的静止条件（D2）时才保留会话、接着执行下一轮；条件不满足，
或属于 D3 清单中的情况时，Node 停止 Agent 进程树并结束会话。新行为由 Node 声明能力 `turn_cancel`、Controller
在 `StartAgentSession` 中逐会话选择；没有选择的会话和旧对端保持原行为。

本 ADR 处于 `proposed`，供评审纠偏，尚无实现，也不维护核心测试用例；评审需要覆盖的验证义务列在
"验证义务"一节，`approved` 时转为核心用例。影响 desktop 的 `crates/node-protocol`（消息、事件、能力、字段）、
`crates/node-db`（受理新命令、持久化新事件）与 `apps/ora-node/src/session/`（会话执行）；共享
`crates/agent-runtime` 已有取消当前一轮、取消宽限期与续用 provider 会话的能力，本决策只调整宽限期取值。
Controller 侧（停止按钮改用新命令、展示一轮的结束）由各 Controller 自己接入，不在本 ADR 内。兼容性边界：
已有消息、已有会话输入和未选择新行为的会话完全不变；不恢复 Node 重启前的会话（协议 D6 不变）；失败原因的
上报不在本 ADR 内（见 D6）。

## 继承与修改

继续[实时预览](20261018-live-thread-previews.md)，会话命令、事件与终态的规则来自
[会话执行与命令](20260928-streamed-thread-events-and-session-commands.md)，结束规则来自
[agent-runtime 根决策](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D6，子代理投影的
终态规则来自[子代理投影、历史与诊断](../agent-runtime/20261020-child-projection-history-and-diagnostics.md)。

| 前序约定 | 本文处理 |
| --- | --- |
| 协议根：能力在握手中协商，执行输入先持久化再启动，同输入重发幂等 | 继承；新增能力 `turn_cancel`，新字段属于执行输入（D5） |
| 会话命令 D4：命令先持久受理再回复 `SessionCommandAccepted`；同一 `command_id` 至多执行一次；用户轮次按受理顺序执行 | 继承；`CancelTurn` 沿用同一受理、幂等与回复规则（D1） |
| 会话命令 D4：只有 `EndSession` 丢弃排队轮次 | 修改：`CancelTurn` 也可以丢弃它指定的那一个排队轮次，并以 `TurnCancelSettled{not_started}` 告知（D1） |
| 会话命令 D4：`EndSession` 取消当前轮次、丢弃排队轮次并结束会话 | 继承，不变；`EndSession` 优先于 `CancelTurn`（D4） |
| 会话命令 D4：会话已终态时命令回复 `SessionCommandRejected{session_ended}` | 继承；`CancelTurn` 同样适用。新增拒绝原因 `turn_cancel_unsupported`（D1） |
| 会话执行 D2：非终态事件只有 Thread 事件，与终态共享序号空间，持久化后才发送，确认前保留，重连后重放 | 修改：新增非终态事件 `TurnCancelSettled`，沿用同一序号、持久化、确认与重放规则；它不是 Thread 记录，不写入会话 JSONL（D1） |
| 协议 D6：Node 重启后会话以 `interrupted` 结束，不恢复 Agent | 继承，不变 |
| 实时预览：旁路不改变权威状态，查看与断开不触发取消 | 继承，不变 |
| 子代理投影：不从静默、父工具完成或父 prompt 结束推断子代理终态 | 继承；未终态的子代理投影使 D2 的静止条件不成立 |
| agent-runtime D6："轮次失败、超时、连接丢失：记录 `TurnEnded`，会话保持" | 修改。当前实现为"只有成功的 ACP 完成才接受下一轮，失败以 `agent_failed` 结束会话"，与 D6 原文不符。改为：连接丢失结束会话；失败与超时只在选择了新行为、且 Agent 满足 D2 的静止条件时保留会话（D2、D3）；未选择的会话保持当前实现的行为 |

## 问题与约束

Node 协议的会话命令只有 `SubmitUserTurn` 与 `EndSession`，没有"只取消当前一轮"。Controller 想停下一轮，只能
`EndSession`，会话随之结束，用户的对话只能新开、之前的上下文全部丢失。

一轮失败同样结束会话：模型服务报错或限流、key 失效、要求重新认证、模型长时间没有输出被看门狗放弃，这些情况下
Agent 进程通常还活着，只是这一轮没能完成，但 Node 以 `agent_failed` 结束整个会话。这样做的原因是：此前失败的
一轮被当作正常完成，调用方看不出这一轮失败，会话停在空闲直到空闲超时，同时一直占着插件租约和模型授权。所以
任何放宽都必须满足：

- 依赖"失败即终态"的调用方（例如以会话终态结算一次运行的 Controller）行为不变；
- 每一次取消都在有界时间内得到确定结论，不能让 Controller 在某个分支上无限等待；
- 同一会话的下一轮开始前，上一轮的工作必须可验证地静止：ACP 的更新不带请求身份，上一轮的迟到更新一旦跨进
  下一轮，Node 就无法把它们分开，上一轮仍在运行的工具或子代理也可能继续修改 checkout；
- 用户的停止与 Agent 自己结束这一轮存在竞争，取消不能误伤用户刚提交的下一轮。

这里的"并发修改"只指同一会话内的两轮。同一 checkout 可能同时承载多个会话，不同会话之间的并发修改由
checkout 共享的规则约束，不属于本决策。

共享运行时已经具备所需的底层能力：取消当前一轮时先以 `cancelled` 应答挂起的权限请求，再发 `session/cancel`，
在宽限期内等待 Agent 对这次 prompt 的回应；回应到达时 provider 会话可以续用，回应没有到达时运行时隔离该
provider 会话，下一轮通过 `session/load`（Agent 支持时）或带 Ora 记录交接的新 provider 会话继续。本决策
规定 Node 协议与会话执行如何使用这些能力，以及何时不能使用。

## 决策

### D1：`CancelTurn` 命令及其结论

`CancelTurn{node_id, command_id, turn_id}`，Controller → Node，与 `SubmitUserTurn`、`EndSession` 同属一个会话执行。
持久化后回复 `SessionCommandAccepted{command_id}`；同一 `command_id` 重复发送只回复受理，内容不同以协议错误
拒绝，与其他会话命令相同。

`SessionCommandAccepted` 只表示命令已持久受理。取消的结论由 Node 发出的非终态事件
`TurnCancelSettled{command_id, turn_id, outcome}` 给出，或在会话因此结束时由终态 `AgentSessionEnded` 给出：

| 受理时目标轮次 `turn_id` 的状态 | Node 行为 | 结论 | 结论最晚何时发出（自受理起） |
| --- | --- | --- | --- |
| 正在执行 | 以 `cancelled` 应答挂起的权限请求，向 Agent 发 ACP `session/cancel`，在 G 内等待 Agent 回应这次 prompt，再按 D2 检查静止条件 | 回应到达且静止：先写 `TurnEnded{cancelled}`，再发 `TurnCancelSettled{stopped}`，会话保留。否则：停止 Agent 进程树，发 `AgentSessionEnded{agent_failed}`（D3） | G + Q + K（D2、D4） |
| 已受理、还在排队 | 从队列中丢弃，永不发给 Agent，不写它的用户消息和 `TurnEnded` | `TurnCancelSettled{not_started}` | 立即（只受账本写入约束） |
| 已经以自己的原因结束 | 不做任何事 | `TurnCancelSettled{already_ended}` | 立即 |
| 没有这个 `turn_id` | 不做任何事 | `TurnCancelSettled{unknown}` | 立即 |

- `CancelTurn` 只作用于 `turn_id` 那一轮：不丢弃其他排队轮次，不结束会话；被取消的一轮结束后按受理顺序
  执行下一轮。排在目标轮次前面的轮次照常执行，取消排队轮次的结论不等它们结束，所以前一轮运行多久都不影响
  这次取消的结论时间。
- `TurnCancelSettled` 与 Thread 事件共享执行内的序号空间：持久化后才发送，确认前保留，重连后按原序号重放，
  `EventAck` 确认精确的 `(execution_id, sequence)`。它不是 Thread 记录，不写入会话 JSONL，也不进入实时预览。
- Controller 只在目标 `SubmitUserTurn` 已被受理之后发送 `CancelTurn`；目标尚未受理时结论是 `unknown`，
  Node 不为它保留取消。
- 会话已终态时回复 `SessionCommandRejected{session_ended}`。会话没有选择新行为（D5）时回复
  `SessionCommandRejected{turn_cancel_unsupported}`，不执行，也不改变会话。

为什么带 `turn_id`：用户点停止时，这一轮可能恰好自然结束、下一轮随即开始。"取消当前轮"会落在用户刚提交的
下一轮上；按 `turn_id` 定位，这种竞争只会得到 `already_ended`。为什么每个分支都有显式结论：Controller 不能从
"没有收到 `TurnEnded`"推断目标是在排队、已结束还是不存在；没有显式结论，排队、已结束和未知三个分支都会让
Controller 一直等到自己的超时。为什么排队轮次直接丢弃而不是轮到时再记一个 `TurnEnded`：后者让结论时间取决于
前一轮跑多久，可能超过 Controller 的等待上限；丢弃与 `EndSession` 丢弃排队轮次的语义一致，Controller 以
`not_started` 把这一轮标为丢弃。

### D2：一轮以取消或失败结束后会话保留的条件

对选择了新行为的会话，一轮以下列方式结束时，只要**同时**满足下面三条静止条件，会话就保留，回到"等待下一轮或
结束命令"：

| 这一轮怎么结束 | 记录 |
| --- | --- |
| `CancelTurn`，Agent 在 G 内回应了这次 prompt | `TurnEnded{cancelled}` |
| ACP `session/prompt` 返回错误（模型请求失败、限流、key 失效、要求重新认证、Agent 内部错误） | `TurnEnded{cancelled}` |
| 无进展看门狗取消后 Agent 回应了这次 prompt，但不再重发（回应不是 `cancelled`，或重发次数用尽） | `TurnEnded{cancelled}` |

**静止条件**，Node 在发出下一轮之前逐条检查，任何一条不满足就按 D3 结束会话（detail `turn_not_quiescent`）：

1. **回应栅栏**：Agent 回应了这次 prompt（结果或错误）。ACP 要求 Agent 在中止进行中的操作、发完待发的更新
   之后才回应取消，回应是这一轮主流程结束的栅栏。
2. **没有未终态的工作**：回应时，这一轮的每个工具调用都已有终态（`completed` 或 `failed`），且没有仍未终态的
   子代理投影。仍处于 `pending`、`in_progress` 的工具调用或子代理，说明这一轮的工作可能还在进行。
3. **静默窗口**：回应之后的 Q 时间内，这个会话没有收到任何 Agent 发来的更新、权限请求或其他消息。

为什么要这三条：ACP 的更新不带请求身份。上一轮 A 回应后立即发出下一轮 B，之后再到达的 A 的工具更新与 B 的
更新无法区分，只能在 B 开始之前把 A 的残余流量挡在外面。条件 1 依据协议栅栏，条件 2 依据 Agent 自己报告的
工作状态，条件 3 检查栅栏之后确实没有残余流量；三条都可以由 Node 自己观察、在测试中构造。静默窗口里到达的
更新不记入任何一轮，也不送进下一轮，同时让会话走兜底分支：这是"迟到更新跨进下一轮"唯一可检验的拦截点。
条件不满足时的兜底是停止 Agent 的整棵进程树（D3），这是在 Node 一侧唯一能确定上一轮工作已经停下的办法。

静止条件之外的残余风险：Agent 回应、所有已报告的工作都已终态、静默窗口也过去之后，仍在运行却从未报告的后台
工作，Node 无法观察。本决策不把它当作已解决：实测（见"已知风险与待实测"）证明某个 Agent 存在这类工作时，
`approved` 版本把该 Agent 的取消与失败一律归入兜底分支，而不是放宽条件。

**时间参数**：

- **G（取消宽限期）**：`CancelTurn` 等待 Agent 回应的上限，自 Node 受理命令起算。G 覆盖所支持 Agent 正常取消的
  最坏耗时，而不是 Agent 卡死时的兜底：

  | Agent | 正常取消的最坏耗时 | 卡死时的兜底 |
  | --- | --- | --- |
  | OpenCode | 中断后自己最多等 5 秒，shell 工具再过 3 秒强制终止，约 8 秒 | 同左 |
  | Claude Code | 中断通常很快 | SDK 无响应时 30 秒后强制回 `cancelled`，此时 provider 会话状态本身可疑 |

  第一版暂定 G 为 15 秒，实测后定值，写入 `approved` 版本；当前共享运行时的 5 秒短于 OpenCode 的正常取消耗时，
  不能沿用。G 只是上限，Agent 回应后立即结束等待。看门狗取消的宽限期不随之加长，仍按运行时既有规则，以免
  模型无输出时的重发被拖慢；看门狗取消没有得到回应时同样按 D3 结束会话。
- **Q（静默窗口）**：第一版暂定 2 秒，实测后定值。只在一轮以取消或失败结束后检查，一轮正常完成后不等待，
  不影响正常对话的延迟。
- **K（停止耗时）**：Node 停止 Agent 进程树、写入并发送终态所需的时间上限，由 Node 的插件停止规则约束。

**续用方式**：满足静止条件后，下一轮在同一个会话执行、同一个 Ora 会话记录上进行。provider 会话仍可用时直接
续用；运行时已隔离它时，按运行时既有规则 `session/load` 恢复，或开新的 provider 会话并交接 Ora 的记录。ACP 错误
回应不等于 provider 会话仍然可用：Agent 可能在报错的同时关闭了这个 provider 会话（之后的 prompt 回"会话已结束"
或"会话不存在"），Node 把这类回应视为 provider 会话已失效，下一轮先恢复或重建，恢复和重建都失败时按 D3 结束
会话。会话的模型授权、插件租约和工作区在会话结束前一直保留，与一轮成功完成后相同。

失败的一轮与取消的一轮在 Thread 里都记为 `TurnEnded{cancelled}`，本决策不区分两者（见 D6）。

### D3：必须结束会话的情况

**Node 结束会话**，先停止这个会话的 Agent 插件及其整棵进程树，再发 `AgentSessionEnded`（选择了新行为的会话
同样适用）：

| 情况 | 终态 |
| --- | --- |
| Agent 进程退出、崩溃，或 Node 与 Agent 之间的 ACP 连接断开 | `agent_failed`，detail 为既有的 `agent_unavailable` 或 `agent_stream_closed` |
| 取消或看门狗取消后，宽限期内 Agent 没有回应这次 prompt | `agent_failed`，detail `turn_stop_unconfirmed` |
| 一轮以取消或失败结束后，D2 的静止条件不成立 | `agent_failed`，detail `turn_not_quiescent` |
| 运行时不再受理新的一轮（会话记录已降级、轮次未被受理、provider 会话恢复与重建都失败） | `agent_failed`，detail 为既有代码 |
| Thread 镜像不可用 | `agent_failed`，detail `thread_unavailable` |
| Node 账本读写失败，包括命令结算失败 | `agent_failed`，detail `ledger_unavailable` |
| 模型授权被撤销 | `agent_failed`，detail `model_access_revoked` |
| `EndSession` | 命令给出的原因 |
| Node 重启 | `interrupted`（协议 D6） |

**Controller 结束会话**：Controller 一侧的故障由 Controller 判断，需要时发 `EndSession`，或按 Controller 自己的
决策把会话当作结束，本决策不改变这些规则。包括：与 Node 的连接断开且不再恢复、事件在 Controller 一侧持久化
失败、Node 回复 `SessionCommandRejected`、Node 回复的身份与会话对不上。

没有选择新行为的会话，D2 中的失败仍以 `agent_failed` 结束会话，与本决策之前一致。

**为什么兜底是安全的**：终态到达 Controller 时 Agent 的进程树已经停止，不可能再修改 checkout，结束会话本身就是
"上一轮已停下"的证据。停下之前 Agent 已经做出的修改留在 checkout 中，由会话结束后的 Revision 交付照常保存，
不会丢失。不确定的只是这一轮里没有收到终态的工具调用：它们在记录中保持 `pending` 或 `in_progress`，由
`TurnEnded{cancelled}` 说明这一轮被打断，不被记成成功。代价是用户的对话结束，而不是工作区与记录不一致。

### D4：与 `EndSession`、工具调用和 Controller 等待上限的先后关系

- **`EndSession` 优先**：队列中任意位置的 `EndSession` 仍优先于其前面的命令；与 `CancelTurn` 同时存在时按
  `EndSession` 结束会话，未发出结论的 `CancelTurn` 以终态作为结论，不再单独发 `TurnCancelSettled`。
  `EndSession` 之后到达的 `CancelTurn` 回复 `session_ended`。已持久受理的 `EndSession` 与一轮失败同时发生时，
  会话以 `EndSession` 的原因结束（沿用当前实现）。
- **先到者为准**：`CancelTurn` 生效前这一轮已经以 Agent 自己的停止原因结束，`TurnEnded` 保持那个原因，结论为
  `already_ended`。同一 `turn_id` 的多个 `CancelTurn` 只有第一个可能得到 `stopped` 或 `not_started`，其余为
  `already_ended`。
- **工具调用与权限请求**：取消前先以 `cancelled` 应答这一轮挂起的权限请求。Agent 回应之前发出的更新都记入这
  一轮；回应之后、静默窗口内到达的流量按 D2 处理；这一轮结束时仍未终态的工具调用保持原状态，并使静止条件
  不成立。
- **事件顺序**：正在执行的一轮被取消并保留会话时，执行内的事件顺序为：这一轮的 Thread 事件 → `TurnEnded` →
  `TurnCancelSettled{stopped}` → 下一轮的 Thread 事件。会话因此结束时，`TurnEnded` 之后是终态，终态之后没有事件。
- **Controller 的等待上限**：Controller 自收到 `SessionCommandAccepted` 起等待结论的上限 T 必须大于
  G + Q + K。这样 Controller 总能收到确定结论：`TurnCancelSettled`（四种之一）或 `AgentSessionEnded`。T 更短时，
  Controller 会在 Node 仍在正常停止 Agent 时超时，只能自行猜测：当作"已停下"可能让下一轮或新会话与仍在运行的
  旧 Agent 同时修改同一 checkout；当作"停止未确认"则把一次可以确认的停止变成故障。只有 Node 本身失联时才会
  等满 T，此时按 Controller 的连接断开规则处理。Controller 可以在收到结论之前提交下一轮，下一轮按受理顺序排在
  被取消的一轮之后执行。

### D5：能力协商与逐会话选择

- Node 在握手的 `capabilities` 中声明 `turn_cancel`，表示它支持 `CancelTurn`、`TurnCancelSettled` 和 D2 的会话保留。
- `AgentSessionSpec` 增加可选字段 `keep_session_on_turn_end: bool`，省略即 `false`。Controller 只在当前连接的
  握手声明了 `turn_cancel` 时设置为 `true`；它和其他输入一样先持久化，同一执行重发时参与输入比较。
- 字段为 `true` 的会话：接受 `CancelTurn`，按 D1–D4 处理一轮的结束。字段省略或为 `false` 的会话：拒绝
  `CancelTurn`（`turn_cancel_unsupported`），一轮失败仍结束会话，也不会出现 `TurnCancelSettled`。
- 旧 Node 不声明 `turn_cancel`，且 `StartAgentSession` 使用 `deny_unknown_fields`：Controller 不能向它发送新字段
  或 `CancelTurn`，停止仍用 `EndSession`。旧 Controller 不设置新字段：Node 对它的会话保持原行为，不会收到它不认识
  的事件。
- 新命令与新事件写入 Node 账本；需要时账本 schema 前向迁移，旧程序拒绝打开新库（沿用 Node 升级规则）。

为什么按会话选择，而不是在握手里打开（类似实时预览的 `Hello.thread_previews`）：会话比连接活得久，重连可能
连到另一个 Controller 版本。选择写进执行输入后，同一会话从开始到结束只有一套规则，取消、失败和恢复时都不需要
再看当前连接是谁。

### D6：与失败原因上报的边界

本决策只规定一轮以取消或失败结束时会话是否保留，以及调用方如何观察这一轮和这次取消的结束。它不区分"取消"
和"失败"，也不携带失败原因：两者在 Thread 中都是 `TurnEnded{cancelled}`。Controller 可以用"这一轮是否收到了
`TurnCancelSettled{stopped}`"判断这是不是自己请求的取消，但不能据此认定其余的结束都是失败，也拿不到原因。
区分取消与失败、在协议和历史记录中携带失败原因、看门狗取消后 Agent 回 `end_turn` 时是重发还是以"无进展"
失败，都由后续决策承担；它们建立在本决策的会话保留之上。

## 不变量

1. 一个被受理的 `SubmitUserTurn` 恰好以下列之一结束：带其 `turn_id` 的一个 `TurnEnded`；被 `CancelTurn` 丢弃并有
   `TurnCancelSettled{not_started}`；会话先结束时按 `EndSession` 的既有规则被丢弃。
2. 每个被受理的 `CancelTurn`，在受理后 G + Q + K 内一定有结论：一条 `TurnCancelSettled`，或会话终态。
3. `CancelTurn` 只影响 `turn_id` 那一轮，不丢弃其他轮次；它导致会话结束只经由 D3 的兜底分支。
4. `EndSession` 优先于 `CancelTurn`；会话已终态后命令不复活会话。
5. 会话在一轮取消或失败后保留，当且仅当会话选择了新行为、Agent 连接仍在、且 D2 的三条静止条件都成立。
6. 同一会话内，上一轮满足静止条件之前，Node 不向 Agent 发送下一轮；回应之后到达的上一轮流量不记入下一轮。
7. 发出 `AgentSessionEnded` 时，Agent 的进程树已经停止。
8. 没有选择新行为的会话，行为与本决策之前完全一致。
9. `SessionCommandAccepted` 不代表取消已有结论；结论只以 `TurnCancelSettled` 或终态为准。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| `EndSession` 增加"只取消一轮"的原因 | 不新增消息；但 `EndSession` 的规则是丢弃排队轮次并产生终态，一个命令两种语义会让"终态之后没有事件"和命令幂等的判断都依赖原因字段 |
| `CancelTurn` 不带 `turn_id`，取消"当前"一轮 | Controller 更简单；但停止与这一轮自然结束、下一轮开始存在竞争，会取消用户刚提交的下一轮（D1） |
| 不新增结论事件，只用 `TurnEnded` 表示取消完成 | 少一种事件；但排队、已结束和未知三个分支没有 `TurnEnded` 或要等前一轮结束才有，Controller 只能等到超时（D1） |
| 排队轮次轮到时再记 `TurnEnded{cancelled}` | Thread 保留这条用户消息；但结论时间取决于前一轮跑多久，可能超过 Controller 的等待上限 |
| 回应后立即发下一轮，不检查静止条件 | 下一轮更早开始；但上一轮的迟到更新会与下一轮的更新混在一起，仍在运行的工具或子代理可能继续修改 checkout，且 Node 无法发现（D2） |
| 每次取消或失败后都重建 provider 会话，靠会话 ID 区分迟到更新 | 迟到更新可以按 provider 会话 ID 丢弃；但每次都失去原 provider 上下文，且无法阻止上一轮的工具继续修改 checkout |
| 失败后一律保留会话，不协商 | 不改输入；但依赖"失败即终态"的调用方会看到会话停在空闲直到超时，并一直占用租约和模型授权，这正是当前实现改为失败即结束的原因 |
| 在握手中打开新行为 | 与实时预览一致；但会话跨连接存在，重连到不同版本的 Controller 会让同一会话前后规则不同（D5） |
| 取消没有得到 Agent 回应、或静止条件不成立时也保留会话 | 用户少遇到一次对话结束；但上一轮可能仍在执行，下一轮与它并发修改同一 checkout，迟到的更新也会混进下一轮 |
| 兜底时只重启 Agent 进程、保留会话：停止进程树后，下一轮经 `session/load` 或交接记录重建 provider 会话 | 卡死时对话也能保住，停止进程树同样证明上一轮已停下，共享运行时也已有"进程被替换后下一轮重新接上"的逻辑；但与"Agent 进程退出即结束会话"的规则相反，要另外处理重启后的模型授权、Git 访问和插件租约，重建出的上下文也不再是原来的 provider 会话。第一版不采用，可作为后续决策 |
| 宽限期取 30 秒，覆盖 Claude Code 卡死时的强制回应 | 卡死的 Claude Code 也能保留会话；但为一个 provider 会话本身已可疑的情况让用户多等，并迫使每个 Controller 把等待上限放宽到 30 秒以上，卡死时用户要等更久才知道对话已结束 |
| 失败后结束会话，由 Controller 新开会话并重放历史 | Node 不变；但同一 Agent 进程仍可用时要重建上下文，成本高，并把会话连续性的责任推给每个 Controller |

## 已知风险与待实测

按 OpenCode（`@opencode/cli` 2.0.32）与 Claude Code（`@agentclientprotocol/claude-agent-acp` 0.107.0）的 ACP
实现核对：取消、模型请求错误、要求重新认证、看门狗取消后回 `end_turn`，以及取消时有进行中的工具调用或挂起的
权限请求，这些情况之后两者都能在同一个 ACP 会话上继续，上下文保留。下列风险只靠源码不能排除，在实现阶段用
真实 Agent 实测，结论写入 `approved` 版本：

| 风险 | 影响 | 处理 |
| --- | --- | --- |
| OpenCode 的子代理、Claude Code 的后台任务可能在回应之后继续运行，并发出轮次之外的更新和权限请求 | 已报告的子代理或工具由静止条件 2、3 拦截；从未报告的后台工作 Node 无法观察 | 实测后台工作是否随取消停止、是否总有可观察的痕迹；存在不可观察的后台工作时，该 Agent 的取消与失败一律走兜底分支（D2） |
| OpenCode 内部异常时，ACP 层返回错误，但 server 端的执行可能仍在进行 | 下一轮混入上一轮未停的工作 | 实测确认；成立时把这类错误归入兜底分支 |
| Claude Code 内部或传输错误后关闭 provider 会话，Agent 进程仍在 | 下一轮发到已失效的会话 | 按 D2 识别后 `session/load` 恢复 |
| G 短于 Agent 正常取消的耗时，或 Q 短于 Agent 回应后残余流量的间隔 | 本可续用的会话被结束，或残余流量落在窗口之外 | 实测 OpenCode 取消运行中 shell 工具的耗时、两者回应后残余流量的时间分布，再定 G 与 Q |
| 一轮失败后保留会话，会话存在的时间变长 | 同一 checkout 承载多个会话时，会更久地占用 checkout 的会话名额 | 由 Controller 的空闲超时与 `EndSession` 释放；checkout 名额的规则不在本决策内 |

## 验证义务

`approved` 时转为核心测试用例，每项都要有直接证据：

1. 取消正在执行的一轮后，同一会话可以接着 `SubmitUserTurn`，上下文保留；事件顺序为 `TurnEnded` →
   `TurnCancelSettled{stopped}` → 下一轮。
2. 取消排队中的一轮时，前一轮运行超过 Controller 的等待上限 T，`TurnCancelSettled{not_started}` 仍在受理后立即
   发出，被取消的轮次从未发给 Agent，前一轮不受影响。
3. 取消已结束和不存在的轮次，分别立即得到 `already_ended` 与 `unknown`。
4. 取消与这一轮自然结束竞争时，`TurnEnded` 保留 Agent 自己的停止原因，结论为 `already_ended`，用户随后提交的下一轮
   不被取消。
5. Agent 回应后在静默窗口内仍发来上一轮的工具更新：这些更新不记入任何一轮，会话以 `turn_not_quiescent` 结束，
   下一轮从未发给 Agent。
6. 回应时仍有未终态的工具调用或子代理投影：会话以 `turn_not_quiescent` 结束。
7. G 内没有回应：Agent 进程树停止后才发出 `AgentSessionEnded{turn_stop_unconfirmed}`，总耗时不超过 G + K。
8. ACP prompt 报错、看门狗放弃后满足静止条件时，会话保留；报错后 provider 会话已失效时，下一轮经恢复或重建继续。
9. D3 中每一种情况仍结束会话。
10. 未选择新行为的会话：`CancelTurn` 被拒绝，一轮失败仍以 `agent_failed` 结束，行为与之前一致；旧 Node 与旧
    Controller 的消息和输入编解码不变。
11. `EndSession` 与 `CancelTurn` 同时排队时按 `EndSession` 结束，`CancelTurn` 不再单独产生结论。

## 本决策未解决的问题

- 区分取消与失败、上报失败原因，以及看门狗取消后 Agent 回 `end_turn` 的处理（D6）。
- Agent 进程退出、兜底停止进程树之后的会话恢复，以及 Node 重启后的会话恢复（协议 D6）。
- 各 Controller 如何展示被取消或失败的一轮、何时自动发 `EndSession`。

## 落地顺序

1. **协议与账本**：`crates/node-protocol` 增加 `CancelTurn` 消息、`TurnCancelSettled` 事件、`turn_cancel` 能力、
   `keep_session_on_turn_end` 字段和拒绝原因 `turn_cancel_unsupported`，补校验与往返测试；`crates/node-db` 受理
   并持久化新命令与新事件。完成条件：旧形状的消息与输入编解码不变，新字段省略时输入比较与之前相同。
2. **Node 会话执行**：`apps/ora-node/src/session/` 按 D1–D4 执行 `CancelTurn`，按 D2 检查静止条件，按 D3 区分
   一轮失败与必须结束会话的情况。完成条件："验证义务"中的每一项都有直接测试；"已知风险与待实测"中的每一项都
   有真实 Agent 的实测结论，并据此确定 G 与 Q。
3. **Controller 接入**：各 Controller 在看到 `turn_cancel` 后为新会话选择新行为，停止改用 `CancelTurn`，并按 D4
   设置大于 G + Q + K 的等待上限。

保持不变：已有消息、已有会话、未选择新行为的会话，以及 Node 重启后的 `interrupted`。从第 2 步开始改变：选择了
新行为的会话在一轮取消或失败、且 Agent 已静止后保留。
