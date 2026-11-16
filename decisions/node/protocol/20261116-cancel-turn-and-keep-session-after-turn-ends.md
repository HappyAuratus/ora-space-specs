---
status: proposed
---

# Node 协议增加"取消一轮"命令；一轮以取消或失败结束、且 Agent 已静止时，会话保留

Controller 新增会话命令 `CancelTurn{command_id, turn_id}`。Node 持久受理后，在有界时间内在本地确定这次取消的
结论：目标轮次正在执行时，向 Agent 发 ACP `session/cancel`，这一轮以 `TurnEnded{cancelled}` 结束；目标轮次
已开始、但 prompt 还没有发给 Agent 时不再发出；目标轮次还在排队时直接丢弃；目标轮次已经在停止中时，并入那一次停止的结论；目标轮次已经结束或不存在时不做任何事。
结论以新的非终态事件 `TurnCancelSettled` 或会话终态，按既有的有序事件通道交给 Controller。选择了新行为的
会话，在一轮以取消或失败（ACP prompt 报错、无进展看门狗放弃）结束后，只有 Agent 具备可依赖的轮次边界、且
满足可验证的静止条件（D2）时才保留会话、接着执行下一轮；否则，或属于 D3 清单中的情况时，Node 停止 Agent
进程树并结束会话。新行为由 Node 与 Controller 在握手中共同协商 `turn_cancel`、Controller 在 `StartAgentSession`
中逐会话选择；没有选择的会话和旧对端保持原行为，不具备该能力的 Controller 不能接管已选择新行为的会话。

本 ADR 处于 `proposed`，供评审纠偏，尚无实现，也不维护核心测试用例；评审需要覆盖的验证义务列在
"验证义务"一节，`approved` 时转为核心用例。影响 desktop 的 `crates/node-protocol`（消息、事件、握手字段、能力、
会话输入字段）、`crates/node-db`（受理新命令、持久化新事件）与 `apps/ora-node/src/session/`（会话执行）；共享
`crates/agent-runtime` 已有取消当前一轮、取消宽限期、丢弃空闲期流量与续用 provider 会话的能力；本决策要它
区分用户取消与看门狗取消的宽限期、在 prompt 发出前和看门狗重发前认取消、识别已失效的 provider 会话，并向
Node 报告 D2 判定所需的事实。
Controller 侧（停止按钮改用新命令、展示一轮的结束）由各 Controller 自己接入，不在本 ADR 内。兼容性边界：
已有消息、已有会话输入和未选择新行为的会话完全不变；不恢复 Node 重启前的会话（协议 D6 不变）；失败原因的
上报不在本 ADR 内（见 D6）。

## 继承与修改

继续[实时预览](20261018-live-thread-previews.md)，会话命令、事件与终态的规则来自
[会话执行与命令](20260928-streamed-thread-events-and-session-commands.md)，连接关闭的分类来自
[Controller 心跳与 close code](20260925-controller-heartbeat-and-websocket-close-codes.md)，结束规则来自
[agent-runtime 根决策](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D6，子代理投影的
终态规则来自[子代理投影、历史与诊断](../agent-runtime/20261020-child-projection-history-and-diagnostics.md)。

| 前序约定 | 本文处理 |
| --- | --- |
| 协议根：能力在握手中协商，执行输入先持久化再启动，同输入重发幂等 | 继承；新增能力 `turn_cancel`，新字段属于执行输入（D5） |
| 实时预览：Controller 在 `Hello` 中声明可选能力，双方都支持时 `HelloAccepted` 才确认 | 继承同一形式；新增 `Hello.turn_cancel` 与 `HelloAccepted.turn_cancel`（D5） |
| close code：身份或能力不符时以 `4403` 结束会话，归为 `Mismatch` | 继承；Node 拒绝不具备 `turn_cancel` 的 Controller 接管已选择新行为的会话时使用它（D5） |
| 会话命令 D4：命令先持久受理再回复 `SessionCommandAccepted`；同一 `command_id` 至多执行一次；用户轮次按受理顺序执行 | 继承；`CancelTurn` 沿用同一受理、幂等与回复规则（D1） |
| 会话命令 D4：只有 `EndSession` 丢弃排队轮次 | 修改：`CancelTurn` 也可以丢弃它指定的那一个排队轮次，并以 `TurnCancelSettled{not_started}` 告知（D1） |
| 会话命令 D4：`EndSession` 取消当前轮次、丢弃排队轮次并结束会话 | 继承，不变；`EndSession` 优先于 `CancelTurn`（D4） |
| 会话命令 D4：会话已终态时命令回复 `SessionCommandRejected{session_ended}` | 继承；`CancelTurn` 同样适用。新增拒绝原因 `turn_cancel_unsupported`（D1） |
| 会话执行 D2：非终态事件只有 Thread 事件，与终态共享序号空间，持久化后才发送，确认前保留，重连后重放 | 修改：新增非终态事件 `TurnCancelSettled`，沿用同一序号、持久化、确认与重放规则；它不是 Thread 记录，不写入会话 JSONL（D1） |
| 会话执行 D3：每个执行最多 256 个已发送未确认的事件，满时暂停发送、不暂停记录 | 继承，不变；结论同样受窗口约束，因此本文区分本地结算期限与投递，Controller 按投递进展等待（D4） |
| 协议 D6：Node 重启后会话以 `interrupted` 结束，不恢复 Agent | 继承，不变 |
| 实时预览：旁路不改变权威状态，查看与断开不触发取消 | 继承，不变 |
| 子代理投影：不从静默、父工具完成或父 prompt 结束推断子代理终态 | 继承；启动过子代理投影的轮次不满足 D2 的静止条件 |
| agent-runtime D6："轮次失败、超时、连接丢失：记录 `TurnEnded`，会话保持" | 修改。当前实现为"只有成功的 ACP 完成才接受下一轮，失败以 `agent_failed` 结束会话"，与 D6 原文不符。改为：连接丢失结束会话；失败与超时只在选择了新行为、且满足 D2 时保留会话（D2、D3）；未选择的会话保持当前实现的行为 |

## 问题与约束

Node 协议的会话命令只有 `SubmitUserTurn` 与 `EndSession`，没有"只取消当前一轮"。Controller 想停下一轮，只能
`EndSession`，会话随之结束，用户的对话只能新开、之前的上下文全部丢失。

一轮失败同样结束会话：模型服务报错或限流、key 失效、要求重新认证、模型长时间没有输出被看门狗放弃，这些情况下
Agent 进程通常还活着，只是这一轮没能完成，但 Node 以 `agent_failed` 结束整个会话。这样做的原因是：此前失败的
一轮被当作正常完成，调用方看不出这一轮失败，会话停在空闲直到空闲超时，同时一直占着插件租约和模型授权。所以
任何放宽都必须满足：

- 依赖"失败即终态"的调用方（例如以会话终态结算一次运行的 Controller）行为不变；
- 每一次取消都在有界时间内在 Node 本地得到确定结论，并按序送达 Controller，不能让 Controller 在某个分支上
  无限等待，也不能让投递背压被误判为 Node 未能停止；
- 同一会话的下一轮开始前，上一轮的工作必须可验证地静止：ACP 的更新不带请求身份。上一轮回应之后、下一轮
  发出之前到达的更新，运行时已经丢弃；下一轮的 prompt 发出之后才到达的上一轮更新，Node 无法把它和下一轮
  的更新分开，上一轮仍在运行的工具或子代理也可能继续修改 checkout；
- 用户的停止与 Agent 自己结束这一轮存在竞争，取消不能误伤用户刚提交的下一轮，重复的停止请求也不能得到与
  实际不符的结论；
- 会话跨连接存在，任何时刻接管它的 Controller 都必须理解它会收到的事件。

这里的"并发修改"只指同一会话内的两轮。同一 checkout 可能同时承载多个会话，不同会话之间的并发修改由
checkout 共享的规则约束，不属于本决策。

共享运行时已经具备所需的底层能力：取消当前一轮时先以 `cancelled` 应答挂起的权限请求，再发 `session/cancel`，
在宽限期内等待 Agent 对这次 prompt 的回应；回应到达时 provider 会话可以续用，回应没有到达时运行时隔离该
provider 会话，下一轮通过 `session/load`（Agent 支持时）或带 Ora 记录交接的新 provider 会话继续。运行时在
本机会话上已经这样取消一轮并续用会话：它以回应作为唯一的栅栏，在两次 prompt 之间丢弃空闲期到达的更新、
以 `cancelled` 应答空闲期的权限请求，并保留标题更新。本决策规定 Node 协议与会话执行如何使用这些能力，
以及何时不能使用。

## 决策

### D1：`CancelTurn` 命令及其结论

`CancelTurn{node_id, command_id, turn_id}`，Controller → Node，与 `SubmitUserTurn`、`EndSession` 同属一个会话执行。
持久化后回复 `SessionCommandAccepted{command_id}`；同一 `command_id` 重复发送只回复受理，内容不同以协议错误
拒绝，与其他会话命令相同。

`SessionCommandAccepted` 只表示命令已持久受理。取消的结论由 Node 写入的非终态事件
`TurnCancelSettled{command_id, turn_id, outcome}` 给出，或在会话因此结束时由终态 `AgentSessionEnded` 给出。
一轮从被取消或失败开始、到 D2 判定会话保留或结束为止，处于"停止中"。按受理时目标轮次 `turn_id` 的状态：

| 状态 | Node 行为 | 结论 | 本地结算期限（自受理起） |
| --- | --- | --- | --- |
| 已开始，prompt 还没有发给 Agent（会话接入或 MCP 准备中） | 不向 Agent 发出这次 prompt；准备完成或中止后结束这一轮 | 先写 `TurnEnded{cancelled}`，再写 `TurnCancelSettled{stopped}`，会话保留；Agent 没有收到这一轮，不需要 D2 判定。准备在 G 内既没有完成也无法中止时，停止 Agent 进程树，写 `AgentSessionEnded{agent_failed}`（D3） | G + K |
| 正在执行（prompt 已发给 Agent） | 以 `cancelled` 应答挂起的权限请求，向 Agent 发 ACP `session/cancel`，在 G 内等待回应，再按 D2 判定 | 会话保留：先写 `TurnEnded{cancelled}`，再写 `TurnCancelSettled{stopped}`。否则停止 Agent 进程树，写 `AgentSessionEnded{agent_failed}`（D3） | G + Q + K |
| 停止中（已有一个 `CancelTurn`，或看门狗正在取消这次 prompt，或这一轮已失败，正在等待回应或静默窗口） | 不再发 `session/cancel`，并入正在进行的那一次停止；看门狗正在取消时，这一轮不再重发 | 与那一次停止同时结算：会话保留时写本命令的 `TurnCancelSettled`，结果为 `stopped`（这一轮因取消而停，包括看门狗取消期间受理的 `CancelTurn`）或 `already_ended`（这一轮因失败而停）；会话结束时以终态为结论 | 不晚于那一次停止的期限，因此也不超过自本命令受理起的 G + Q + K |
| 已受理、还在排队 | 从队列中丢弃，永不发给 Agent，不写它的用户消息和 `TurnEnded` | `TurnCancelSettled{not_started}` | 立即（只受账本写入约束） |
| 已经结束：正常完成，或以取消、失败结束且 D2 已判定会话保留 | 不做任何事 | `TurnCancelSettled{already_ended}` | 立即 |
| 没有这个 `turn_id` | 不做任何事 | `TurnCancelSettled{unknown}` | 立即 |

- 同一次停止并入的多个 `CancelTurn`，各自的 `TurnCancelSettled` 按受理顺序紧接在这一轮的 `TurnEnded` 之后写入。
- `TurnEnded` 在 Agent 回应这次 prompt、或宽限期到期时写入，早于 D2 的判定。所以"停止中"包含 `TurnEnded`
  已经写入之后的静默窗口，Controller 可能先收到 `TurnEnded`，之后才收到 `TurnCancelSettled` 或终态。
- 看门狗取消期间受理 `CancelTurn` 时，看门狗已经发出的 `session/cancel` 就是这次停止的取消，等待回应的上限改为
  自本命令受理起的 G；看门狗在两次发送之间时，不再发出下一次。
- `CancelTurn` 只作用于 `turn_id` 那一轮：不丢弃其他排队轮次，不结束会话；被取消的一轮结束后按受理顺序
  执行下一轮。排在目标轮次前面的轮次照常执行，取消排队轮次的结论不等它们结束，所以前一轮运行多久都不影响
  这次取消的本地结算时间。
- `TurnCancelSettled` 与 Thread 事件共享执行内的序号空间：持久化后才发送，确认前保留，重连后按原序号重放，
  `EventAck` 确认精确的 `(execution_id, sequence)`，同样受在途窗口约束（D4）。它不是 Thread 记录，不写入会话
  JSONL，也不进入实时预览。
- Controller 只在目标 `SubmitUserTurn` 已被受理之后发送 `CancelTurn`；目标尚未受理时结论是 `unknown`，
  Node 不为它保留取消。
- 会话已终态时回复 `SessionCommandRejected{session_ended}`。会话没有选择新行为（D5）时回复
  `SessionCommandRejected{turn_cancel_unsupported}`，不执行，也不改变会话。

为什么 prompt 发出前的取消不等 prompt 发出：等它发出再取消，Agent 会白白开始一次模型请求，结论时间还取决于
会话准备多久；不发出时 Agent 没有这一轮的任何工作，会话可以直接保留。为什么带 `turn_id`：用户点停止时，这一轮可能恰好自然结束、下一轮随即开始。"取消当前轮"会落在用户刚提交的
下一轮上；按 `turn_id` 定位，这种竞争只会得到 `already_ended`。为什么每个分支都有显式结论：Controller 不能从
"没有收到 `TurnEnded`"推断目标是在排队、已结束还是不存在；没有显式结论，这些分支都会让 Controller 一直等到
自己的超时。为什么停止中的重复取消要并入：此时这一轮还没有确定结果，回 `already_ended` 会先于第一个取消宣称
"已结束"，而两者最终可能一起以会话终态收场。为什么排队轮次直接丢弃：轮到时再记 `TurnEnded` 会让结论时间取决于
前一轮跑多久；丢弃与 `EndSession` 丢弃排队轮次的语义一致，Controller 以 `not_started` 把这一轮标为丢弃。

### D2：一轮以取消或失败结束后会话保留的条件

对选择了新行为的会话，一轮以下列方式结束时，只有同时满足"可依赖的轮次边界"和三条静止条件，会话才保留，
回到"等待下一轮或结束命令"。prompt 从未发给 Agent 的轮次不在此列，按 D1 直接保留：

| 这一轮怎么结束 | 记录 |
| --- | --- |
| `CancelTurn`，Agent 在 G 内回应了这次 prompt | `TurnEnded{cancelled}` |
| ACP `session/prompt` 返回错误（模型请求失败、限流、key 失效、要求重新认证、Agent 内部错误） | `TurnEnded{cancelled}` |
| 无进展看门狗取消后 Agent 回应了这次 prompt，但不再重发（回应不是 `cancelled`，或重发次数用尽） | `TurnEnded{cancelled}` |

**轮次流量与会话级通知**：本节的条件只针对轮次流量，即会写入一轮内容的消息：正文、思考、用户消息片段、
工具调用及其更新、计划、权限请求，以及带子代理来源的更新；Node 不认识的更新类型按轮次流量处理。会话级通知
（`session_info_update`、`available_commands_update`、`current_mode_update`、`config_option_update`、
`usage_update`）和 Agent 对 Node 自己发出的请求的回应不属于任何一轮：Agent 会在两轮之间正常发送它们（例如
标题和用量），运行时按既有规则处理，它们不影响轮次边界、静止条件和跨轮次检测。

**可依赖的轮次边界**：静止条件只能观察 Agent 发来的流量，挡不住静默窗口之后才到达、又没有身份的更新（例如
正文片段）。所以 Node 只对已验证具备轮次边界的 Agent 保留会话：Agent 回应一次 prompt 之后，不再为这一轮发出
任何轮次流量，也没有在回应后继续运行、却不报告的工作。这是上游 Agent 实现的属性，清单按能唯一确定它的版本
登记（插件锁定上游版本时，即插件的精确版本），由该版本的一致性测试证明（"验证义务"第 12 条），Node 只对
清单中的版本启用会话保留。不在清单中的 Agent，这一轮的取消或失败一律走兜底分支（D3，detail
`turn_boundary_unverified`），即使会话选择了新行为。

按 OpenCode（`@opencode/cli` 2.0.32）与 Claude Code（`@agentclientprotocol/claude-agent-acp` 0.107.0）的源码，
两者的主轮次都具备这一边界：OpenCode 每一轮只转发自己投递之后的事件，Claude Code 丢弃被中断工作的后续输出。
两者都不保证子代理和后台任务的边界：OpenCode 的子代理在取消后继续运行，Claude Code 的后台子代理在取消后
继续运行并可能发出权限请求。所以静止条件 2 排除启动过子代理的轮次。两者能否进入清单，以一致性测试为准，
`approved` 版本写明结论。

**静止条件**，Node 在发出下一轮之前逐条检查，任何一条不满足就按 D3 结束会话（detail `turn_not_quiescent`）：

1. **回应栅栏**：Agent 回应了这次 prompt（结果或错误）。ACP 要求 Agent 在中止进行中的操作、发完待发的更新
   之后才回应取消，回应是这一轮主流程结束的栅栏。
2. **没有未终态或无法收尾的工作**：回应时，这一轮的每个工具调用都已有终态（`completed` 或 `failed`），且这一轮
   没有启动过子代理投影。子代理可能在父轮次结束后继续运行，其终态不能由父 prompt 结束推断，首期子代理投影也
   没有子代理终态的来源。这是第一版的已知限制：用过子代理的轮次（长任务中常见）取消或失败后，会话一律结束。
3. **静默窗口**：回应之后的 Q 时间内，这个会话没有收到任何轮次流量。会话级通知不计入。

**跨轮次检测**：一轮以取消或失败结束并保留会话后，在这个会话其后的所有轮次中，若收到引用这一轮工具调用
身份或子代理来源的更新，Node 认定这一轮的流量跨进了后面的轮次：该更新不记入任何一轮，按 D3 结束会话
（detail `turn_leak_detected`）。静止条件 2 已经保证这些工具调用在回应时都有终态，所以之后任何引用它们的更新
都是违约。这是对具备身份的更新的兜底检测；没有身份的更新只能依靠轮次边界清单排除。一轮正常完成之后的迟到
更新不在本决策内，沿用运行时既有规则：空闲期到达的更新被丢弃，不结束会话。

为什么这样分层：ACP 的更新不带请求身份。上一轮 A 回应后、下一轮 B 发出前到达的更新，运行时会丢弃；B 的
prompt 发出之后才到达的 A 的无身份更新，与 B 的更新在协议上无法区分，任何静默窗口都只能降低概率，不能兑现
隔离。可依赖的轮次边界把"回应之后不再有 A 的流量"
变成对插件版本的可测试承诺；静止条件在每一次结束时检查 Agent 是否真的遵守了它；跨轮次检测拦截有身份的违约。
三者都不成立的情况一律停止 Agent 进程树，这是在 Node 一侧唯一能确定上一轮工作已经停下的办法。

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
- **Q（静默窗口）**：第一版暂定 2 秒，按实测的残余流量分布定值。它只用于发现违约，不是隔离的依据。只在一轮
  以取消或失败结束后检查，一轮正常完成后不等待，不影响正常对话的延迟。取消后立刻提交下一轮是最常见的用法，
  Q 直接加在这次等待上，所以取能发现残余轮次流量的最小值；实测没有残余轮次流量时可以取 0。
- **K（停止耗时）**：Node 停止 Agent 进程树、写入终态所需的时间上限，由 Node 的插件停止规则约束。

**续用方式**：满足上述条件后，下一轮在同一个会话执行、同一个 Ora 会话记录上进行。provider 会话仍可用时直接
续用；运行时已隔离它时，按运行时既有规则 `session/load` 恢复，或开新的 provider 会话并交接 Ora 的记录。ACP 错误
回应不等于 provider 会话仍然可用：Agent 可能在报错的同时关闭了这个 provider 会话（之后的 prompt 回"会话已结束"
或"会话不存在"），Node 把这类回应视为 provider 会话已失效，下一轮先恢复或重建，恢复和重建都失败时按 D3 结束
会话。会话的模型授权、插件租约和工作区在会话结束前一直保留，与一轮成功完成后相同。

失败的一轮与取消的一轮在 Thread 里都记为 `TurnEnded{cancelled}`，本决策不区分两者（见 D6）。

### D3：必须结束会话的情况

**Node 结束会话**，先停止这个会话的 Agent 插件及其整棵进程树，再写 `AgentSessionEnded`（选择了新行为的会话
同样适用）：

| 情况 | 终态 |
| --- | --- |
| Agent 进程退出、崩溃，或 Node 与 Agent 之间的 ACP 连接断开 | `agent_failed`，detail 为既有的 `agent_unavailable` 或 `agent_stream_closed` |
| 取消或看门狗取消后，宽限期内 Agent 没有回应这次 prompt；或取消时仍在进行的会话准备在 G 内既没有完成也无法中止 | `agent_failed`，detail `turn_stop_unconfirmed` |
| 一轮以取消或失败结束，而 Agent 插件版本不在轮次边界清单中 | `agent_failed`，detail `turn_boundary_unverified` |
| 一轮以取消或失败结束后，D2 的静止条件不成立 | `agent_failed`，detail `turn_not_quiescent` |
| 其后的轮次中收到可归属某个被取消或失败轮次的更新 | `agent_failed`，detail `turn_leak_detected` |
| 运行时不再受理新的一轮（会话记录已降级、轮次未被受理、provider 会话恢复与重建都失败） | `agent_failed`，detail 为既有代码 |
| Thread 镜像不可用 | `agent_failed`，detail `thread_unavailable` |
| Node 账本读写失败，包括命令结算失败 | `agent_failed`，detail `ledger_unavailable` |
| 模型授权被撤销 | `agent_failed`，detail `model_access_revoked` |
| `EndSession` | 命令给出的原因 |
| Node 重启 | `interrupted`（协议 D6） |

**Controller 结束会话**：Controller 一侧的故障由 Controller 判断，需要时发 `EndSession`，或按 Controller 自己的
决策把会话当作结束，本决策不改变这些规则。包括：与 Node 的连接断开且不再恢复、事件在 Controller 一侧持久化
失败、Node 回复 `SessionCommandRejected`、Node 回复的身份与会话对不上、以及 D4 中 Node 未在期限内结算。

没有选择新行为的会话，D2 中的失败仍以 `agent_failed` 结束会话，与本决策之前一致。

**为什么兜底是安全的**：终态写入时 Agent 的进程树已经停止，不可能再修改 checkout，结束会话本身就是"上一轮已
停下"的证据。停下之前 Agent 已经做出的修改留在 checkout 中，由会话结束后的 Revision 交付照常保存，不会丢失。
不确定的只是这一轮里没有收到终态的工具调用：它们在记录中保持 `pending` 或 `in_progress`，由
`TurnEnded{cancelled}` 说明这一轮被打断，不被记成成功。代价是用户的对话结束，而不是工作区与记录不一致。

### D4：先后关系、本地结算与投递

- **`EndSession` 优先**：队列中任意位置的 `EndSession` 仍优先于其前面的命令；与 `CancelTurn` 同时存在时按
  `EndSession` 结束会话，尚未结算的 `CancelTurn` 以终态作为结论，不再单独写 `TurnCancelSettled`。
  `EndSession` 之后到达的 `CancelTurn` 回复 `session_ended`。已持久受理的 `EndSession` 与一轮失败同时发生时，
  会话以 `EndSession` 的原因结束（沿用当前实现）。
- **先到者为准**：`CancelTurn` 受理前这一轮已经以 Agent 自己的停止原因结束并完成 D2 判定，`TurnEnded` 保持那个
  原因，结论为 `already_ended`。受理时这一轮仍在停止中，按 D1 并入那一次停止。
- **工具调用与权限请求**：取消前先以 `cancelled` 应答这一轮挂起的权限请求。Agent 回应之前发出的更新都记入这
  一轮；回应之后到达的轮次流量不记入任何一轮，并按 D2 让会话走兜底分支，会话级通知照常处理；这一轮结束时
  仍未终态的工具调用保持原状态，并使静止条件不成立。
- **事件顺序**：正在执行的一轮被取消并保留会话时，执行内的事件顺序为：这一轮的 Thread 事件 → `TurnEnded` →
  并入这次停止的各个 `TurnCancelSettled` → 下一轮的 Thread 事件。会话因此结束时，`TurnEnded` 之后是终态，终态
  之后没有事件。
- **本地结算期限**：Node 自受理 `CancelTurn` 起，在 G + Q + K 内把它的结论（`TurnCancelSettled` 或终态）持久化到
  账本。这一期限只取决于 Node 与 Agent，与连接和在途窗口无关。
- **投递**：结论与其他事件一样按序号投递，受在途窗口约束：前序事件未被确认、窗口已满时，结论在账本中等待，
  确认释放窗口后才发送；断线时在重连后按序重放。所以 Controller 收到结论的时间等于本地结算时间加上它确认
  前序事件所用的时间，后者由 Controller 自己的确认进度决定。
- **Controller 等待规则**：Controller 按投递进展而不是固定时长等待结论。等待期间，只要连接仍在、并且持续收到
  这个执行的事件，就继续确认并等待，不判为超时。只有连接仍在、Controller 已确认它收到的全部事件、且自受理与
  最后一次确认两者中较晚的时刻起超过 T 仍没有收到任何新事件时，才判定 Node 未在期限内结算，按 Controller 自己的
  故障规则处理（例如发 `EndSession`，并把这次停止视为未确认）。T 必须大于 G + Q + K 加上传输余量；T 更短时，
  Controller 会在 Node 仍在正常停止 Agent 时下结论：当作"已停下"可能让下一轮或新会话与仍在运行的旧 Agent 同时
  修改同一 checkout，当作"停止未确认"则把一次可以确认的停止变成故障。连接断开时按 Controller 的连接断开规则
  处理，重连后从重放的事件重新开始计算。Controller 可以在收到结论之前提交下一轮，下一轮按受理顺序排在被取消的
  一轮之后执行。

### D5：能力协商、逐会话选择与接管

- Node 在 `HelloAccepted.capabilities` 中声明 `turn_cancel`。Controller 在 `Hello` 中以 `turn_cancel: true` 声明它理解
  `CancelTurn`、`TurnCancelSettled`、新的拒绝原因与终态 detail；旧 Controller 省略该字段。双方都支持时，Node 在
  `HelloAccepted` 中回 `turn_cancel: true`，这次连接才算协商成功，与实时预览的协商形式相同。
- `AgentSessionSpec` 增加可选字段 `keep_session_on_turn_end: bool`，省略即 `false`。Controller 只在当前连接协商
  成功时设置为 `true`；它和其他输入一样先持久化，同一执行重发时参与输入比较。
- 字段为 `true` 的会话：接受 `CancelTurn`，按 D1–D4 处理一轮的结束。字段省略或为 `false` 的会话：拒绝
  `CancelTurn`（`turn_cancel_unsupported`），一轮失败仍结束会话，也不会产生 `TurnCancelSettled` 或新的 detail。
- **接管边界**：只要 Node 上还有字段为 `true`、且尚未终态或仍有未确认事件的会话执行，Node 就拒绝没有协商
  `turn_cancel` 的连接：在 `HelloAccepted` 之前以 `4403`（能力不符）结束这次连接，不投递、不重放任何事件，也不
  受理任何命令。这些执行在 Node 上照常进行并持久化事件，等到具备能力的 Controller 连接后，从最小的未确认序号
  起按序重放。所以已选择新行为的会话不会被不理解其事件的 Controller 接管。降级 Controller 之前，应先结束这些
  会话或等它们终态并确认完毕。
- **拒绝的持续时间**：close code 决策把 `4403` 归为重连不会自行恢复的配置错误，旧 Controller 仍按 `reconnect_ms`
  重连，在拒绝解除前每次都被拒。拒绝在这些会话终态、且事件全部被确认后解除；事件只能由具备能力的 Controller
  确认，所以只剩旧 Controller 时拒绝不会自行解除（见"本决策未解决的问题"）。
- **打开选择的前提**：Controller 只有在可能连接这个 Node 的所有 Controller 实例都具备 `turn_cancel` 之后，才为
  新会话选择新行为。滚动升级期间新旧实例并存时不选择，否则旧实例重连就会被拒绝。
- 旧 Node 不声明 `turn_cancel`，且 `StartAgentSession` 使用 `deny_unknown_fields`：Controller 不能向它发送新字段
  或 `CancelTurn`，停止仍用 `EndSession`。旧 Controller 不设置新字段：Node 对它启动的会话保持原行为。
- 新命令与新事件写入 Node 账本；需要时账本 schema 前向迁移，旧程序拒绝打开新库（沿用 Node 升级规则）。

为什么按会话选择，而不是只在握手里打开：会话比连接活得久，重连可能连到另一个 Controller 版本。选择写进执行输入
后，同一会话从开始到结束只有一套规则；握手协商只决定当前连接能否发出选择、能否接管已选择的会话。为什么在连接
层拒绝而不是只扣住这些执行的事件：扣住事件时，旧 Controller 仍会为这些会话发命令，却永远等不到结果，也无法区分
"Node 扣住了"与"Node 出故障了"；以 `4403` 拒绝连接，归类明确，且不改变任何执行的状态。

### D6：与失败原因上报的边界

本决策只规定一轮以取消或失败结束时会话是否保留，以及调用方如何观察这一轮和这次取消的结束。它不区分"取消"
和"失败"，也不携带失败原因：两者在 Thread 中都是 `TurnEnded{cancelled}`。Controller 可以用"这一轮是否收到了
`TurnCancelSettled{stopped}`"判断这是不是自己请求的取消，但不能据此认定其余的结束都是失败，也拿不到原因。
区分取消与失败、在协议和历史记录中携带失败原因、看门狗取消后 Agent 回 `end_turn` 时是重发还是以"无进展"
失败，都由后续决策承担；它们建立在本决策的会话保留之上。

后续决策落地之前，选择了新行为的会话有一个缺口。没有选择的会话，一轮失败时以 `agent_failed` 和它的 detail
（例如 `agent_authentication_required`、`agent_turn_failed`）结束，调用方能看到这次失败。选择了新行为并保留
会话后，这一轮只有 `TurnEnded{cancelled}`：调用方看不出这是失败，也拿不到"需要重新认证"。
`keep_session_on_turn_end` 同时打开取消后保留与失败后保留，所以只想要"停止"的 Controller 也会遇到这个缺口。
怎样消除它，见"本决策未解决的问题"。

## 不变量

1. 一个被受理的 `SubmitUserTurn` 恰好以下列之一结束：带其 `turn_id` 的一个 `TurnEnded`；被 `CancelTurn` 丢弃并有
   `TurnCancelSettled{not_started}`；会话先结束时按 `EndSession` 的既有规则被丢弃。
2. 每个被受理的 `CancelTurn`，在受理后 G + Q + K 内，Node 已把它的结论持久化：一条 `TurnCancelSettled`，或会话终态。
3. 结论按执行内的序号投递，在前序事件被确认、窗口释放后送达；连接仍在时，只要 Controller 持续确认，结论一定到达。
4. 并入同一次停止的多个 `CancelTurn` 得到与这次停止一致的结论；在这一轮停止中时，没有 `CancelTurn` 得到 `already_ended`。
5. `CancelTurn` 只影响 `turn_id` 那一轮，不丢弃其他轮次；它导致会话结束只经由 D3 的兜底分支。
6. `EndSession` 优先于 `CancelTurn`；会话已终态后命令不复活会话。
7. 会话在一轮取消或失败后保留，当且仅当会话选择了新行为，并且：这一轮的 prompt 从未发给 Agent；或者 Agent
   版本在轮次边界清单中、Agent 连接仍在、D2 的三条静止条件都成立。
8. 同一会话内，上一轮满足 D2 之前，Node 不向 Agent 发送下一轮；回应之后到达的上一轮轮次流量不记入任何一轮，
   其后的轮次中收到可归属被取消或失败轮次的更新时会话结束。
9. 写入 `AgentSessionEnded` 时，Agent 的进程树已经停止。
10. 已选择新行为、尚未终态或仍有未确认事件的会话，不会把事件投递给没有协商 `turn_cancel` 的 Controller。
11. 没有选择新行为的会话，行为与本决策之前完全一致。
12. `SessionCommandAccepted` 不代表取消已有结论；结论只以 `TurnCancelSettled` 或终态为准。
13. 会话级通知不改变任何一轮的结论，也不导致会话结束。
14. 被取消的一轮，在受理 `CancelTurn` 之后不再有新的 `session/prompt` 发给 Agent：prompt 尚未发出时不发出，
    看门狗不再重发。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| `EndSession` 增加"只取消一轮"的原因 | 不新增消息；但 `EndSession` 的规则是丢弃排队轮次并产生终态，一个命令两种语义会让"终态之后没有事件"和命令幂等的判断都依赖原因字段 |
| `CancelTurn` 不带 `turn_id`，取消"当前"一轮 | Controller 更简单；但停止与这一轮自然结束、下一轮开始存在竞争，会取消用户刚提交的下一轮（D1） |
| 不新增结论事件，只用 `TurnEnded` 表示取消完成 | 少一种事件；但排队、已结束和未知三个分支没有 `TurnEnded` 或要等前一轮结束才有，Controller 只能等到超时（D1） |
| 排队轮次轮到时再记 `TurnEnded{cancelled}` | Thread 保留这条用户消息；但结论时间取决于前一轮跑多久，可能超过 Controller 的等待上限 |
| 停止中的重复取消立即回 `already_ended` | 实现简单；但这一轮还没有确定结果，两次请求最终可能一起以会话终态收场，第二次的结论与事实不符（D1） |
| 结论绕过在途窗口优先发送 | Controller 更早知道结论；但破坏执行内序号连续、结论必须在 `TurnEnded` 之后的顺序，重放时还要处理越序事件 |
| Controller 按固定时长 T 等待 | 规则简单；但窗口背压下 Node 已按时结算，Controller 仍会等满 T，把投递延迟误判为停止失败（D4） |
| 只以 Agent 的回应为准，不设清单、静默窗口与跨轮次检测（共享运行时在本机会话上的现有做法） | 规则最简单，且已在使用；但它发现不了回应之后才到达的轮次流量。本决策在回应之上加了三层检查，代价是规则更复杂、更多会话在取消后被结束。三层是否都需要，取决于实测的残余流量，结论写入 `approved` 版本 |
| 静默窗口内收到任何消息都算违约 | 不需要区分消息类型；但 Agent 在两轮之间正常发送标题、用量等会话级通知，本可续用的会话会被结束（D2） |
| prompt 发出前受理的取消，等 prompt 发出后再取消 | 运行时不用改；但 Agent 白白开始一次模型请求，结论时间还取决于会话准备多久（D1） |
| 只靠静默窗口隔离迟到更新，加长 Q | 不依赖 Agent 的承诺；但静默窗口之后到达的无身份更新仍无法区分，Q 只能降低概率，并拖慢每一次取消后的下一轮 |
| 每次取消或失败后都重建 provider 会话，靠会话 ID 区分迟到更新 | 迟到更新可以按 provider 会话 ID 丢弃；但每次都失去原 provider 上下文，且无法阻止上一轮的工具继续修改 checkout |
| 失败后一律保留会话，不协商 | 不改输入；但依赖"失败即终态"的调用方会看到会话停在空闲直到超时，并一直占用租约和模型授权，这正是当前实现改为失败即结束的原因 |
| 只在握手中打开新行为，不写入会话输入 | 与实时预览一致；但会话跨连接存在，重连到不同版本的 Controller 会让同一会话前后规则不同（D5） |
| 旧 Controller 接管时，只扣住已选择新行为的执行的事件 | 不影响同一 Node 上的其他执行；但旧 Controller 仍会为这些会话发命令、永远等不到结果，也无法与 Node 故障区分（D5） |
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
| 主轮次回应之后仍有无身份的更新到达 | 静默窗口之后到达时会混进下一轮 | 由轮次边界一致性测试判定；测试不通过的插件版本不进入清单，取消与失败一律走兜底（D2） |
| OpenCode 的子代理、Claude Code 的后台任务可能在回应之后继续运行，并发出轮次之外的更新和权限请求 | 启动过子代理的轮次由静止条件 2 排除；不报告的后台任务 Node 无法观察 | 一致性测试覆盖后台任务；存在不可观察的后台工作时，该插件版本不进入清单 |
| OpenCode 内部异常时，ACP 层返回错误，但 server 端的执行可能仍在进行 | 下一轮混入上一轮未停的工作 | 实测确认；成立时把这类错误归入兜底分支 |
| Claude Code 内部或传输错误后关闭 provider 会话，Agent 进程仍在 | 下一轮发到已失效的会话 | 按 D2 识别后 `session/load` 恢复 |
| G 短于 Agent 正常取消的耗时，或 Q 短于 Agent 回应后残余流量的间隔 | 本可续用的会话被结束，或违约未被静默窗口发现 | 实测 OpenCode 取消运行中 shell 工具的耗时、两者回应后残余流量的时间分布，再定 G 与 Q |
| 一轮失败后保留会话，会话存在的时间变长 | 同一 checkout 承载多个会话时，会更久地占用 checkout 的会话名额 | 由 Controller 的空闲超时与 `EndSession` 释放；checkout 名额的规则不在本决策内 |
| 降级 Controller 时仍有已选择新行为的会话 | 该 Node 拒绝旧 Controller 的所有连接，同一 Node 上的其他执行也无人接管；这些会话终态后仍有未确认事件时，拒绝不会自行解除 | 全部 Controller 实例具备能力后才打开选择（D5）；降级前先结束这些会话并确认完毕；`4403` 让拒绝原因可诊断；解除办法待定 |
| 用过子代理的轮次取消或失败后会话一律结束 | 长任务常用子代理，这些轮次得不到会话保留，功能的实际覆盖面可能明显小于预期 | 实现阶段统计被取消轮次中用过子代理的比例；放宽需要子代理终态的来源，由后续决策承担 |
| 一轮失败后保留会话，调用方没有失败信号 | 选择了新行为的会话失败时，用户看不到失败，也看不到需要重新认证（D6） | 待定：见"本决策未解决的问题" |

## 验证义务

`approved` 时转为核心测试用例，每项都要有直接证据：

1. 取消正在执行的一轮后，同一会话可以接着 `SubmitUserTurn`，上下文保留；事件顺序为 `TurnEnded` →
   `TurnCancelSettled{stopped}` → 下一轮。
2. 取消排队中的一轮时，前一轮运行超过 Controller 的等待上限 T，`TurnCancelSettled{not_started}` 仍在受理后立即
   写入，被取消的轮次从未发给 Agent，前一轮不受影响。
3. 取消正常完成的轮次、取消已结束且已判定保留会话的轮次、取消不存在的轮次，分别立即得到 `already_ended`、
   `already_ended` 与 `unknown`。
4. 取消与这一轮自然结束竞争时，`TurnEnded` 保留 Agent 自己的停止原因，结论为 `already_ended`，用户随后提交的下一轮
   不被取消。
5. 同一轮的两个 `CancelTurn`：第二个分别在 G 内（仍在等待回应）和 Q 内（仍在静默窗口）受理时，两者得到相同的结论；
   这次停止以兜底结束时，两者都以同一个终态为结论；没有一个得到 `already_ended`。
6. 一轮失败后、静默窗口内受理的 `CancelTurn`，在判定后得到 `already_ended`，或随兜底以终态为结论。
7. Agent 回应后在静默窗口内仍发来上一轮的工具更新：这些更新不记入任何一轮，会话以 `turn_not_quiescent` 结束，
   下一轮从未发给 Agent。
8. 静默窗口已过、下一轮已开始后，才到达引用上一轮工具调用身份的更新：它不记入下一轮，会话以 `turn_leak_detected`
   结束。
9. 回应时仍有未终态的工具调用，或这一轮启动过子代理投影：会话以 `turn_not_quiescent` 结束。
10. 插件版本不在轮次边界清单中时，一轮取消或失败后会话以 `turn_boundary_unverified` 结束。
11. G 内没有回应：Agent 进程树停止后才写入 `AgentSessionEnded{turn_stop_unconfirmed}`，本地结算不超过 G + K。
12. 轮次边界一致性测试（按清单登记的版本）：对真实 Agent 构造取消运行中的 shell 工具、取消带子代理和后台任务的
    轮次、模型请求错误，记录 prompt 回应之后到达的全部流量；没有任何轮次流量才进入清单。
13. 在途窗口已满（256 个未确认事件）时受理 `CancelTurn`：结论在本地期限内写入账本；Controller 释放确认后按序收到
    结论；Controller 持续收到并确认事件期间不判为超时。
14. Controller 已确认全部事件、超过 T 仍无新事件时，判定 Node 未在期限内结算。
15. 新 Controller 启动选择新行为的会话，在确认 `TurnCancelSettled` 前断开；旧 Controller 重连时被以 `4403` 拒绝，
    没有收到任何事件；具备能力的 Controller 随后连接，从最小未确认序号起按序收到全部事件。
16. ACP prompt 报错、看门狗放弃后满足 D2 时，会话保留；报错后 provider 会话已失效时，下一轮经恢复或重建继续。
17. D3 中每一种情况仍结束会话。
18. 未选择新行为的会话：`CancelTurn` 被拒绝，一轮失败仍以 `agent_failed` 结束，行为与之前一致；旧 Node 与旧
    Controller 的消息、握手与输入编解码不变。
19. `EndSession` 与 `CancelTurn` 同时排队时按 `EndSession` 结束，`CancelTurn` 不再单独产生结论。
20. 静默窗口内只收到会话级通知（标题、用量、可用命令）时会话保留，这些通知不记入任何一轮。
21. prompt 发出前受理 `CancelTurn`：Agent 从未收到这次 prompt，事件为 `TurnEnded{cancelled}` →
    `TurnCancelSettled{stopped}`，会话保留，与轮次边界清单无关；准备在 G 内没有结束时以 `turn_stop_unconfirmed` 结束。
22. 看门狗取消期间受理 `CancelTurn`：这一轮不再重发；Agent 回应且满足 D2 时结论为 `stopped`，会话保留。
23. 被取消的一轮之后隔了一轮以上，才到达引用它的工具调用身份的更新：会话仍以 `turn_leak_detected` 结束。一轮
    正常完成后，空闲期到达的更新不结束会话。

## 本决策未解决的问题

- 区分取消与失败、上报失败原因，以及看门狗取消后 Agent 回 `end_turn` 的处理（D6）。
- Agent 进程退出、兜底停止进程树之后的会话恢复，以及 Node 重启后的会话恢复（协议 D6）。
- 轮次边界清单由插件自己声明（例如写入插件的 Agent 声明）还是由 Node 维护；第一版由 Node 维护，随一致性测试更新。
- 一轮失败后保留会话时的失败信号缺口（D6）：把选择拆成"取消后保留"与"失败后保留"两个，还是失败后保留等失败
  原因上报落地后再启用。
- 接管拒绝的解除（D5）：已选择新行为的会话终态后仍有未确认事件、而只剩不具备能力的 Controller 时，怎样让它
  重新接管这个 Node。
- 用过子代理的轮次取消或失败后保留会话，需要子代理终态的来源。
- 清单、静默窗口与跨轮次检测是否都需要，还是可以收敛为运行时现有的回应栅栏，取决于实测的残余流量。
- 各 Controller 如何展示被取消或失败的一轮、何时自动发 `EndSession`。

## 落地顺序

1. **协议与账本**：`crates/node-protocol` 增加 `CancelTurn` 消息、`TurnCancelSettled` 事件、`Hello`／`HelloAccepted`
   的 `turn_cancel`、能力 `turn_cancel`、`keep_session_on_turn_end` 字段、拒绝原因 `turn_cancel_unsupported` 与新的
   终态 detail，补校验与往返测试；`crates/node-db` 受理并持久化新命令与新事件。完成条件：旧形状的消息、握手与
   输入编解码不变，新字段省略时输入比较与之前相同。
2. **共享运行时、Node 会话执行与接管**：`crates/agent-runtime` 区分用户取消与看门狗取消的宽限期，在 prompt 发出前
   和看门狗重发前认取消，识别已失效的 provider 会话；`apps/ora-node/src/session/` 按 D1–D4 执行 `CancelTurn`，
   按 D2 检查轮次边界清单、静止条件与跨轮次泄漏，按 D3 区分一轮失败与必须结束会话的情况；连接层按 D5 拒绝
   不具备能力的 Controller。
   完成条件："验证义务"中的每一项都有直接测试；"已知风险与待实测"中的每一项都有真实 Agent 的实测结论，据此
   确定 G、Q 与轮次边界清单。
3. **Controller 接入**：各 Controller 在握手中声明 `turn_cancel`；可能连接 Node 的全部 Controller 实例都具备能力
   后，才为新会话选择新行为，停止改用 `CancelTurn`，并按 D4 的进展规则等待结论。

保持不变：已有消息、已有会话、未选择新行为的会话，以及 Node 重启后的 `interrupted`。从第 2 步开始改变：选择了
新行为的会话在一轮取消或失败、且满足 D2 后保留。
