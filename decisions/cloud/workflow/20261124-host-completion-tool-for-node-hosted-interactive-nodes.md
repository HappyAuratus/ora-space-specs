---
status: implemented
---

# Cloud 经 Node 宿主 MCP 提供完成工具，交互节点的 `agent` 完成策略生效

Cloud 进程在本机回环上提供工作流完成工具，经 Node 的会话级宿主 MCP 交给 `completionSignal: "agent"` 的交互
节点，按[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md)（下称完成信号 ADR）执行
信号、确认与自动接受。从接入它、且锁定的 Node 声明 `session_host_mcp_servers` 的产品版本起，编辑器给新交互
节点默认写 `agent`，已保存的值不动；Node 不具备宿主能力时，运行在启动准入处以有名字的原因被拒绝，不降级为 `end_turn`。本决策是
[Controller 交付宿主 MCP](../../node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)
（下称宿主 MCP ADR）的第一个使用方，也兑现
[默认 `end_turn`](20261119-end-turn-default-until-host-completion-tool.md)（下称默认值 ADR）的 D4。

本决策处于 `implemented`：D1–D8 于 2026-11-24 经 owner 批准（[ora-space/specs#334](https://github.com/ora-space/specs/pull/334)），
已在 Cloud v2 实现，正文按最终实现同步，与批准时的差异见“实现与批准时的差异”。`implemented` 不表示每项义务都有直接证据，
验证义务、证据与缺口见
[完成工具用例](../../../test-cases/cloud/workflow/host-completion-tool.md)与
[Controller 交付宿主 MCP 用例](../../../test-cases/controller/session/host-mcp-servers.md)。

它适用于接受 `schemaVersion: 4`、经 Node 承载交互会话的 Cloud v2 工作流编辑器与 `v2-server`，以单机部署为前提（D1），
部署范围见[单进程的 Cloud](../lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（下称 lifecycle ADR）。本决策不改
Node 协议、desktop 或完成信号的门禁语义，没有数据迁移义务。Cloud v1 的发布门禁继续拒绝 `schemaVersion: 4`。

## 继承与修改

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| [工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md) 的历史决策：“正式 Cloud”的发布门禁拒绝 `schemaVersion: 4` 与非 null 的 `agentConfig.completion` | 继承，不变 | 这道门禁在 Cloud v1 的路径上，本决策不打开它；`v2-server` 的路径接受 `schemaVersion: 4`，是本决策的适用范围 |
| 默认值 ADR D4：接入完成工具、且锁定的 Node 声明 `session_host_mcp_servers` 的产品版本起，新节点默认 `agent` | 继承，不变 | 兑现本 ADR 的 D4：同时满足两项条件的产品版本才是 D6 的恢复版本 |
| 默认值 ADR D2：恢复版本之前 `agent` 按 `end_turn` 执行并标为暂不生效 | 到期 | 恢复版本起不再适用；D6 的例外只有升级前已启动的会话 |
| 默认值 ADR D3、不变量 4、5：导入和恢复都不改写 `completionSignal`，已保存的 `end_turn`、快照与冻结的图不动 | 继承，不变 | D6 |
| 默认值 ADR：升级前已启动的会话在结束前保持 `end_turn` | 继承，不变 | D6 |
| 默认值 ADR 风险：Agent 到不了 Controller 端点的部署，默认值留给使用方决策 | 承接 | 现在只有单机部署；出现多机、容器或沙箱部署时另立决策（D1） |
| 完成信号 ADR D1、D2：token 绑定会话与 attempt，`session/new`、`session/load` 时重新推导 | 按宿主 MCP ADR 已改的方式继承 | 宿主值在 start 时给定、整个 execution 内固定；token 仍按轮签发（D2、D3） |
| 完成信号 ADR D1：Agent 替换时签发新 token、旧 token 作废 | 修改，只限 Node 在同一 execution 内重启 Agent（插件刷新） | 沿用为该轮签发的 token，端点凭据随 execution 有效；换到新的 execution 时仍签发新 token（D3） |
| 完成信号 ADR D2–D6：失败、崩溃、超时、取消的一轮丢弃请求；两个信号 blocker、`signalId` 确认、自动接受、`requester` 由服务端推导 | 继承，不变 | D5；desktop 的实现在异常停靠时仍绑定请求，Cloud 按 ADR 文字丢弃 |
| 完成信号 ADR D7：不支持工具调用的 Agent 靠 `force` 兜底，不探测能力 | 按宿主 MCP ADR D4 的修改继承 | 不声明 HTTP MCP 的 Agent 以 `host_mcp_http_capability_missing` 失败（D4） |
| 宿主 MCP ADR D1：文法、上限与共用样例，发送前逐字符校验 | 继承，不变 | D1 |
| 宿主 MCP ADR D6：按能力派发、启动前准入 | 继承并扩展 | D4：“从失败处继续”与启动同样准入 |
| 宿主 MCP ADR D6：可达性由部署配置决定，每个 Node 使用哪种 URL 由 Controller 的部署配置决定 | 修改，只限单机部署 | Cloud v2 固定用本机回环的随机端口，不读部署配置；多机、容器或沙箱部署另立决策（D1） |
| 宿主 MCP ADR D6：先持久化再发送，或重启时结束所有受影响的 execution | 选择后者 | D2 |
| 宿主 MCP ADR D6：启动时检查可恢复性，不能恢复的走正常结束路径并记录有名字的原因 | 继承，升级前启动的 execution 例外 | D2：停靠会话保留，它的旧 execution 同样在提供服务前结算；升级前启动、没有记录的不补记 |
| 宿主 MCP ADR D6：凭据只在持久 execution 记录处于非终态时有效 | 当前实现有差异 | D3：凭据按 execution 在内存里校验，持久门禁检查运行、节点、会话与轮次，不读 execution 的结算记录，见“风险与落地顺序”的未决项 |
| 宿主 MCP ADR D6：token 随 prompt 进入 Node 持久的会话命令输入，由使用方规定是否接受 | 决定：接受 | D7 |
| 宿主 MCP ADR D2：“首次停住前没有收到 `initialize` / `tools/list`”由使用方处理 | 决定：停住时提示，不让节点失败 | D8 |
| lifecycle ADR D2、D3：Node 是本地子进程；重启时没跑完的执行标为中断，不续跑 | 继承，不变 | D1、D2 的可达性与凭据做法依赖它 |

## 问题与约束

- 完成工具只有在宿主提供它时才有意义，而 Cloud 的会话由 Node 启动，Cloud 不直接对 Agent 做 ACP setup。
- token 必须到达模型才能被 Agent 传回，因而会经过 Node 与 Agent；Cloud 自己的持久面没有这个必要。
- OpenCode 的 MCP 注册表是进程级的，工具可见不等于有权调用（宿主 MCP ADR D5）。
- 交互节点停住后可以跨 Cloud 重启等待；重启后它的旧 Agent 已经不在，凭据不能假装还有效。
- 完成策略来自冻结的图：实际执行与图不一致又没有记录，是宿主 MCP ADR D6 禁止的静默降级。

## D1：只有 `agent` 策略的根作用域交互节点得到完成工具，端点在本机回环

- Cloud 在 `127.0.0.1` 的随机端口上提供 HTTP MCP，只有一个工具 `ora_complete_workflow_step`，参数只有
  `attemptToken`。端点只回普通 JSON，不提供服务端事件流；单个请求有大小与时长上限。
- 会话启动时，只有根作用域、`interactive`、`completionSignal: "agent"` 的节点在
  `AgentSessionSpec.host_mcp_servers` 里带一项 `ora-workflow-completion`。`end_turn` 节点、非交互节点、Loop 与
  Iteration 的成员、普通对话和任务运行都不带。
- 发送前按宿主 MCP ADR D1 的文法校验，校验测试使用 Node 的共用样例；非法输入不发送。只向同时声明
  `session_host_mcp_servers` 与 `agent_session` 的 Node 发送。
- 宿主能力不进入插件 MCP 的选择、白名单或安装。

**为什么是明文回环**：Node 是 Cloud 的本地子进程，Agent 与 Cloud 在同一网络命名空间，宿主 MCP ADR D1 允许的正是
这种情形，随机端口也不需要部署配置。

**前提是单机部署**：现在只有单机部署。出现多机、容器或沙箱部署时，端点、可达性与这类部署的默认值另立决策，本决策不覆盖。

## D2：端点凭据在内存里，随 Node execution 有效；Cloud 重启即失效

- 端点凭据在 URL 路径里，是 128 位随机值，按 Node execution 签发，会话换成新的 execution 时旧凭据
  作废。Node 在同一 execution 内重启 Agent 时，新进程拿到原样的宿主列表，凭据继续有效。
- 凭据和 token 只在 Cloud 进程内存中。重启时没跑完的执行按 lifecycle ADR D3 标为中断；除跨重启等待的停靠会话外，都在宿主
  Server 监听之前结束。Cloud 进程被杀时 Node 可能残留；下次启动在恢复执行和提供宿主 Server 之前，先停掉本数据目录下
  的全部旧 Node，停不掉就不启动。所以重启必然换 Node，旧凭据没有一个仍有效。跨重启等待的节点保留完成策略与停住那一轮的信号；它的下一轮启动新 Agent，拿到新凭据。
  重建后的轮次与信号记录的 `turnKey` 不一致时，信号丢弃。
- 同一 execution 的 start 若重发，必须携带逐字节相同的宿主列表；不靠重发轮换凭据。
- 宿主 MCP ADR D6 的 execution 指 Node 协议的 execution。每个带宿主工具的 execution 有一条持久记录（身份、会话、开始、结束、
  原因），在发送 start 之前写入；写不进去就不发 start。没有宿主工具的 execution 不记。
- 每个 execution 只结算一次，先定下的原因算数：

  | 怎样结束 | 记录的原因 |
  | --- | --- |
  | Cloud 结束会话 | `session_ended`；因某个失败而结束的，记那个失败码 |
  | Agent 自己结束 | Node 报告的原因 |
  | Node 丢失，或整个 Node 被换掉 | `agent_transport_closed` |
  | Cloud 重启后的启动恢复 | `server_restarted` |

  结束写不进库时在后台重试；会话结束的同一事务把它还没结算的 execution 一并结算，已经定下、还没写进去的原因照写，其余记
  `session_ended`。会话不会带着运行中的 execution 结束。
- 停靠的 `agent` 节点跨重启保留会话；Cloud 在宿主 Server 开始服务之前，在启动恢复的同一事务里把所有还没结束的 execution 以
  `server_restarted` 结算。结算失败则整个恢复回滚，Cloud 不启动。下一轮是新的 execution。Node 之后补报的旧结束事件只确认收到，
  不改已记的原因。没有 execution 的停靠会话（Node 早已丢失，或上次重启后没再发消息）没有可结算的。
- 正常关停时停靠会话不收尾、不写结束，留给下次启动，所以和被杀掉记同一个原因；还在后台重试的原因也会被下次启动记成
  `server_restarted`。关停时正在执行的一轮由关停结束会话，记 `session_ended`。
- 记录里的开始是写入记录的时间，start 可能没有发出；`server_restarted` 的结束是下次启动结算的时间，不是旧 Agent 停止的时间。
- 有记录之前就启动的 execution 不补记：升级前已停靠的 `agent` 会话在第一次重启时没有结论和原因，凭据同样全部失效。

**为什么不持久化凭据**：宿主 MCP ADR D6 允许“重启时结束所有受影响的 execution”代替持久化；Cloud 重启本来就中断全部执行，
持久化只会多一处保存秘密。会话与 execution 分开处理，是为了保留跨重启等待，又不让旧 execution 没有结论。
代价是 Cloud 以后若让 execution 活过重启，本条必须重新决定。

**为什么要持久记录 execution**：是否仍在运行要有状态可查，也要能区分 Node 丢失与重启。先写记录再发 start，崩溃在两者之间时
留下的是一条会被启动恢复结算的记录，而不是没有记录的 execution。升级前的会话分不清 Agent 是否早已丢失，补记只能猜原因。
结算失败就不启动，是因为先服务后补会出现宿主 Server 已经服务、旧 execution 仍是运行态；代价是一条写不进去，整个服务起不来。

## D3：每次调用都按凭据、token 与持久状态三重授权

- attempt token 每一轮签发一个，随 prompt 之后单独的一块交给 Agent。轮次换到新的 execution 时签发新 token，
  旧 token 和用它提交的请求作废；轮次停住、节点完成、失败、取消后 token 失效。
- 插件刷新在当前 prompt 结束后、下一条 prompt 进入 Agent 前重启进程，execution 不变：已经排队的那一轮沿用为它签发的 token，
  不重签；端点凭据随 execution 有效。
- 一次调用被接受，必须同时满足：凭据有效；token 是同一个节点当前某一轮的 token；持久状态显示该节点正在自己
  没有结束的会话里执行一轮，且它所属的运行仍在运行。任一不满足都不留下任何信号，并回答稳定的原因：
  `unknown attempt token`、`stale_attempt`、`not_awaiting`；持久状态读不到时回答 `temporarily_unavailable`，不作废
  token。
- 同一轮重复调用返回同一个 `signalId`。其他节点的凭据或 token 一律拒绝，即使工具在它的 Agent 里可见。
- 请求只绑定发起它的那一轮；停住时取走，之后到达的请求不可能落到下一轮。

**为什么还要看持久状态**：内存里的记录可能落后于停止、取消与失败；以持久状态为准，持久状态已经不显示节点在自己的会话里执行
一轮时，不会因内存延迟而仍可调用。持久状态指运行、节点、会话与轮次；execution 的结算记录先于轮次结束时还有一个窗口，见未决项。

**为什么不重签**：Node 不通知 Controller 这种重启，重签需要改 Node 协议；token 仍只对这一轮有效，上一轮的 token 已经失效，
没有一个 token 因此跨轮可用。

## D4：启动准入先确认 Node 能力，缺了就拒绝，不降级

- 冻结图里有 `agent` 策略的根作用域交互节点时，启动先确认项目的 Node 声明宿主能力；Node 没有运行则先启动它。
  不具备时返回 `workflow_completion_tool_unavailable`，没有节点开始运行，运行状态不变。没有 Agent 运行时的部署同样以此拒绝：
  它不会自己恢复。Node 起不来之类的临时故障按执行服务不可用报告，不算缺少完成工具。
- “从失败处继续”与启动同样准入：缺能力即以同一原因拒绝，运行状态不变。
- 派发时再查一次：准入之后 Node 换成不具备能力的，节点以 `host_mcp_node_capability_missing` 失败。
- Agent 不声明 HTTP MCP 时，会话以 Node 的 `host_mcp_http_capability_missing` 结束，节点失败。
- 任何一种情况都不按 `end_turn` 执行。

**为什么先启动 Node**：能力只有握手后才知道。等到派发才发现，前面的节点已经用掉了模型额度。代价是启动一个运行
可能要先等 Node 就绪。

## D5：门禁、确认与自动接受沿用完成信号 ADR

- `agent_signal_pending`、`agent_confirmation_pending`、`confirmAgentCompletion{signalId}`、`requester` 与
  `confirmation` 的含义与完成信号 ADR D3–D6 相同；过期的 `signalId` 返回 `workflow_node_signal_changed`，`force`
  不能越过它。首轮的 `turn_unconfirmed` 必须由人确认。
- 只有正常结算的一轮才保留与该轮 attempt 匹配的上报。轮次被停止、被截断、被拒绝、Agent 丢失或结局不明时，丢弃该轮
  上报，强制完成记 `requester: human`，沿用完成信号 ADR D2、D3 的结算与停靠规则。
- 节点通过完成门禁并提交完成结果后，后继节点才可启动；满足完成信号 ADR D3 的 `force` 路径也可推进。
- 开启自动接受的节点接受同一信任域内的残余风险：伪造的一次上报能让人发起、正常结束的一轮直接完成节点，但改不了产出，
  同一系统用户的进程本来就在同一信任域里。
- 自动接受由服务端在一轮停住后走同一门禁，只在这一轮由人发起、正常结束、只剩 `agent_confirmation_pending` 时
  生效；首轮与异常停靠不自动接受，被门禁拒绝时留给发起人。

除自动接受的残余风险外，本条没有新的取舍。

## D6：恢复版本起新节点默认 `agent`，已保存的值不动

- 编辑器为没有完成策略的节点打开交互模式时，写 `completionSignal: "agent"` 与 `schemaVersion: 4`；节点已有的值
  （`end_turn` 或 `agent`）原样保留，没有该字段的旧交互节点读作 `end_turn`，不补写。关闭交互模式按完成信号 ADR
  的 version-4 边界清理策略字段。
- `completionSignal` 与 `autoAcceptAgentCompletion` 不再列为未应用的设置，编辑器与运行页不再显示“暂不生效”。
- 升级前已停靠、快照记录为 `end_turn` 的节点，在会话结束前继续按 `end_turn` 执行，不附加完成工具。
- 用户面的代价：新节点要等 Agent 上报，Agent 不上报时由人继续对话或“仍然完成”。

## D7：token 可以留在 Node 与 Agent 的存储里，不出现在 Cloud 自己的任何持久面

- 接受 attempt token 随 prompt 进入 Node 的会话文件与 ledger，以及 Agent 自己的存储：它们与 Cloud 在同一信任域
  （宿主 MCP ADR D5），token 只在一轮内有效，且受 D3 限制。
- Cloud 自己的日志、事件、数据库、会话历史、最终回答与 Node 事件诊断文件里不得出现凭据或 token。token 块从发布的
  历史里去掉，其余位置的值换成固定占位符；写入诊断文件之前替换，而不是事后清理。重启或 Node 重建后重放的事件属于本进程
  不认识的 execution，按结构脱敏；clone 与插件安装的结果除外，它们的 execution 里没有 Agent，也就没有凭据，原样保留用于诊断。
- 端点凭据（URL 路径）只接受留在两处：Agent 自己的存储，以及宿主插件日志里 Agent 自写的 stderr，它们与上一条属于同一
  信任域。Node 自身的持久输入、日志与诊断仍按宿主 MCP ADR D3 不得出现端点凭据。

**为什么接受**：不持久化的承载方式需要改 Node 协议，而它挡不住同一身份下读取当前 prompt 的进程。开启自动接受的
节点没有人工兜底，影响更大，仍接受（D5）。端点凭据必须到达 Agent 才能使用，Node 不过滤 Agent 自己写下的内容；凭据只随 execution 有效，
单有凭据没有当轮 token 也不产生信号（D3）。

## D8：完成工具没有连上时在节点上提示，不让节点失败

- Cloud 按端点凭据在内存里记录收到的 `initialize` 或 `tools/list` 请求，收到任意一个就算连上。通知（没有 `id` 或 `id` 为 `null`）和
  其他方法都不算；凭据换掉之后才到的请求，不算换上来的 execution 的。
- 节点停住时，当前 execution 的 start 已经发给 Node 而端点还没有收到，就在节点上提示完成工具没有连上。提示和停住时的其余摘要在
  同一次写入里，只存一个布尔值，跨重启保留，发起人和其他成员都看得到；节点完成后的摘要不再带它。
- 提示跟着 execution 走：同一 execution 之后连上，下一次停住时去掉；换成新的 execution，按它的首次停住重新判断，在那之前保留上一次的
  结论。页面文字因此写明是“上次停住时”的观察。
- 首轮被发起人停止或失败也算停住，只要 start 已经发给 Node。没有 execution 可说的不判断，保留上一次的结论：start 没有发出的一轮，
  以及会话已经结束而没有送达的消息，即使更早的 execution 之后才连上。
- 提示不改变节点与运行的状态，也不改变完成门禁：节点照常停住等待，blocker 不变，发起人可以继续对话或“仍然完成”。

接受的代价：提示可能出现后又消失；一轮停得太早、Agent 还没来得及连时也会提示。

**为什么不让节点失败**：Agent 可能丢掉连不上的 Server 而照常启动，产出仍然有效，发起人有 `force` 兜底；失败会丢掉已经
完成的工作。只停在 `agent_signal_pending` 而不提示，发起人分不清是 Agent 没有上报还是根本没有工具。

## 不变量

1. 只有根作用域、`completionSignal: "agent"` 的交互节点的会话带完成工具。
2. 端点凭据属于别的节点或已经换掉的 execution，或者 token 属于别的节点或已经停住的轮次时，调用不改变任何状态；
   同一 execution 内重启 Agent 不使该轮的 token 失效。
3. 持久状态不显示节点正在自己的会话里执行一轮时，调用不产生信号。
4. 冻结图需要完成工具而 Node 不具备能力时，启动与“从失败处继续”都被拒绝，没有节点开始运行；任何情况都不降级为 `end_turn`。
5. Cloud 重启之后，重启前签发的凭据与 token 都无效；停靠节点的会话、策略与有效信号仍在，它有记录的旧 execution 在宿主 Server
   开始服务前已经结算，重启时还在运行的原因是 `server_restarted`。
6. 首轮不自动接受；节点通过完成门禁并提交完成结果之前，后继不启动，`force` 只按完成信号 ADR D3 的条件推进。
7. Cloud 自己的日志、事件、数据库、会话历史与诊断文件里没有凭据或 token；Node 自身的持久面与诊断里没有端点凭据。
8. 恢复默认值不改写任何已保存的 `completionSignal`、快照或冻结的图。
9. 完成工具没有连上只产生提示，不改变节点、运行与完成门禁。
10. 带宿主工具的 execution 在 start 发出之前已有记录，只结算一次；会话结束时没有仍在运行的 execution。

## 实现与批准时的差异

| 位置 | 批准时 | 最终实现 | 原因 |
| --- | --- | --- | --- |
| D2、不变量 5 | 停靠会话的旧 execution 都在提供服务前结算 | 升级前已停靠、没有记录的不补记 | 分不清 Agent 是否早已丢失，不写猜出来的原因 |
| D2 | 没有规定记在哪里、失败怎样 | 每个带宿主工具的 execution 一条记录，先写后发 start；结算失败不启动 | 宿主 MCP ADR D6 要有状态可查，也要能区分 Node 丢失与重启 |
| D2 | 没有规定正常关停 | 停靠会话留给下次启动；正在执行的一轮记 `session_ended`；还在重试的原因会变成 `server_restarted` | 停靠会话不因关停结束；正在执行的一轮本身记为取消 |
| D3 | 宿主 MCP ADR D6 的“以持久 execution 记录判定”标为继承 | 批准时还没有 execution 记录；现在有了，但持久门禁仍读运行、节点、会话与轮次，不读它 | 没有随实现一起改；已决定把 execution 终态纳入门禁，见未决项 |
| D4 | 没有写没有 Agent 运行时的部署 | 按缺少完成工具拒绝 | 重试不会好，不能报成临时故障 |
| D7 | 未知 execution 的事件一律按结构脱敏 | clone 与插件安装的结果原样保留 | 里面没有凭据，脱敏会丢掉诊断要用的 commit 与版本 |
| D8 | 首次停住前没有收到就提示 | 另外规定怎样算收到、何时去掉、何时不判断，文字写明是上次停住时 | 批准时没有写明，逐项由 owner 决定 |

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 完成工具做成插件 MCP | 不改宿主接入路径，但依赖插件安装与市场，违背完成信号 ADR D1；包内 URL 写死且必须是 https |
| 解析回答里的完成标记 | 不需要任何宿主能力，但要剥离文本，模型会漏写或误写，完成信号 ADR 已否决 |
| 凭据持久化，重启后原样可用 | 在让 execution 活过重启的部署里更优；Cloud 重启即中断，持久化只多一处秘密（D2） |
| 只用端点凭据，不要 attempt token | 少一个秘密到达模型；但无法区分同一 Agent 的各轮，过期的请求会落到新的一轮 |
| Node 不具备能力时按 `end_turn` 执行 | 用户不必升级 Node；但实际执行与冻结的图不一致，是宿主 MCP ADR D6 禁止的静默降级 |
| 工具没有连上时让节点失败 | 尽早暴露配置问题；但产出仍然有效，发起人有 `force` 兜底，失败会丢掉已完成的工作（D8） |
| 插件刷新重启 Agent 时重签 token | 与完成信号 ADR D1 的文字一致；但要改 Node 协议来通知重启，而 token 本来只对一轮有效（D3） |
| 重启时连停靠会话一起结束 | 结算最简单；但放弃跨重启等待（D2） |
| 停靠会话的旧 execution 只靠换 Node 失效，不结算 | 不需要新的持久状态；但旧 execution 没有结论和原因，不满足宿主 MCP ADR D6（D2） |
| 不记 execution，启动时从会话事件推断 | 少一张表；但 Node 早已丢失的会话会被记成因重启结束（D2） |
| 结算失败时先启动、后台重试 | 一条写不进去不影响启动；但宿主 Server 已经服务、旧 execution 仍是运行态（D2） |
| 没有连上的提示只在内存里，或一直保留到节点结束 | 前者不用存；但重启后提示消失而节点还在等。后者最简单；但工具后来连上了仍说没有连上（D8） |
| 必须收到 `tools/list` 才算连上 | 更接近“工具可用”；但握手后推迟列出工具的 Agent 会被误提示（D8） |
| 运行中不显示上一次的提示 | 不会在已经连上之后仍显示；但成员看不到上一次的结论，页面与保留的状态不一致（D8） |
| “从失败处继续”只在派发时检查 | 少一次检查；但前面的节点可能先用掉模型额度（D4） |
| 避免 Agent 侧留存端点凭据 | 少一处留存；但要另外设计承载方式，同一信任域内的进程本来就能读到（D7） |
| 现在就为多机、容器或沙箱部署定端点或默认值 | 以后不必再决定；但没有这类部署，取舍没有依据（D1） |
| 继续默认 `end_turn` | 少一种要等上报的状态；但与 desktop 不一致，也不符合默认值 ADR D4 |

## 风险与落地顺序

- **提示不是当前状态**：提示是停住时的观察，Agent 之后可能已经连上，也可能只是没来得及连（D8）。没有连上不代表 Agent 不会再连。
- **升级前停靠的会话**：第一次重启时没有结论和原因（D2）。凭据与 token 仍全部失效。
- **首轮绑定之前的上报**：首轮在会话绑定到节点之前就上报，持久状态还不成立，会被拒绝（`not_awaiting`），Agent 要再调用一次。这是已知
  限制，不是本决策的规则，是否消除另行决定。
- **没有在真实链路上验证结算与提示**：D2 的结算与 D8 的提示用假的 Agent 与 Node 以及真实 PostgreSQL 验证，没有跑真实 Node 或 Agent。
- **端点凭据的留存只验证了一种 Agent**：一次真实 Node 与 OpenCode 的运行中，检查的 Node 会话文件与数据库、OpenCode 数据库与
  日志、Cloud 数据库里都没有端点凭据；Claude Code 还没有验证。
- **同一信任域**：见 D5、D7；D3 只能防误用和重放，挡不住同一系统用户下的恶意进程。
- **默认值改变用户习惯**：新节点要等上报，编辑器在完成判定旁说明这一点。
- **未决项：与宿主 MCP ADR D6 的一处差异**：D6 要求凭据只在持久 execution 记录处于非终态时有效。Agent 自己结束或 Node 丢失时，execution 记录先结算，
  轮次随后才结束；两者之间的调用仍会通过持久门禁。这一轮不是正常结束，它的上报随后被丢弃（D5），不会完成节点，但回答是“接受”。已决定
  把 execution 终态纳入持久门禁，与 D6 一致；落地之前这处差异仍在。
- **未解决**：多机、容器或沙箱部署的端点、转发与默认值（D1）；宿主值的会话内轮换；desktop 是否也检查 HTTP MCP。

落地已按三步完成：核心用例与 `approved`；D2 的结算与 D8 的提示分别实现；按最终实现同步本文与用例证据，缺口如实保留为
`Partial` / `Missing`。用 Claude Code 再检索一次端点凭据不在 `implemented` 的条件里，仍是缺口。

从恢复版本起改变的只有 `agent` 策略的交互节点和新节点的默认值；`end_turn` 节点、非交互节点、插件 MCP、Node 协议
与 desktop 的行为不变。
