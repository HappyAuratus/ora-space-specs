---
status: proposed
---

# Cloud 经 Node 宿主 MCP 提供完成工具，交互节点的 `agent` 完成策略生效

Cloud 进程在本机回环上提供工作流完成工具，经 Node 的会话级宿主 MCP 交给 `completionSignal: "agent"` 的交互
节点，按[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md)（下称完成信号 ADR）执行
信号、确认与自动接受。从接入它、且锁定的 Node 声明 `session_host_mcp_servers` 的产品版本起，编辑器给新交互
节点默认写 `agent`，已保存的值不动；Node 不具备宿主能力时，运行在启动准入处以有名字的原因被拒绝，不降级为 `end_turn`。本决策是
[Controller 交付宿主 MCP](../../node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)
（下称宿主 MCP ADR）的第一个使用方，也兑现
[默认 `end_turn`](20261119-end-turn-default-until-host-completion-tool.md)（下称默认值 ADR）的 D4。

本决策处于 `proposed`：D1–D7 是提案，末节“待 owner 决定”列出尚未定的取舍，定之前不构成承诺。它适用于接受
`schemaVersion: 4`、经 Node 承载交互会话的 Cloud v2 工作流编辑器与 `v2-server`，部署范围见
[单进程的 Cloud](../lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（下称 lifecycle ADR）。本决策不改
Node 协议、desktop 或完成信号的门禁语义，没有数据迁移义务。Cloud v1 的发布门禁继续拒绝 `schemaVersion: 4`。

## 继承与修改

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| [工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md) 的历史决策：“正式 Cloud”的发布门禁拒绝 `schemaVersion: 4` 与非 null 的 `agentConfig.completion` | 继承，不变 | 这道门禁在 Cloud v1 的路径上，本决策不打开它；`v2-server` 的路径接受 `schemaVersion: 4`，是本决策的适用范围 |
| 默认值 ADR D4：接入完成工具、且锁定的 Node 声明 `session_host_mcp_servers` 的产品版本起，新节点默认 `agent` | 继承，不变 | 兑现本 ADR 的 D4：同时满足两项条件的产品版本才是 D6 的恢复版本 |
| 默认值 ADR D2：恢复版本之前 `agent` 按 `end_turn` 执行并标为暂不生效 | 到期 | 恢复版本起不再适用；D6 的例外只有升级前已启动的会话 |
| 默认值 ADR D3、不变量 4、5：导入和恢复都不改写 `completionSignal`，已保存的 `end_turn`、快照与冻结的图不动 | 继承，不变 | D6 |
| 默认值 ADR：升级前已启动的会话在结束前保持 `end_turn` | 继承，不变 | D6 |
| 默认值 ADR 风险：Agent 到不了 Controller 端点的部署，默认值留给使用方决策 | 承接 | 待 owner 决定 O3 |
| 完成信号 ADR D1、D2：token 绑定会话与 attempt，`session/new`、`session/load` 时重新推导 | 按宿主 MCP ADR 已改的方式继承 | 宿主值在 start 时给定、整个 execution 内固定；token 仍按轮签发（D2、D3） |
| 完成信号 ADR D2–D6：失败、崩溃、超时、取消的一轮丢弃请求；两个信号 blocker、`signalId` 确认、自动接受、`requester` 由服务端推导 | 继承，不变 | D5；desktop 的实现在异常停靠时仍绑定请求，Cloud 按 ADR 文字丢弃 |
| 完成信号 ADR D7：不支持工具调用的 Agent 靠 `force` 兜底，不探测能力 | 按宿主 MCP ADR D4 的修改继承 | 不声明 HTTP MCP 的 Agent 以 `host_mcp_http_capability_missing` 失败（D4） |
| 宿主 MCP ADR D1：文法、上限与共用样例，发送前逐字符校验 | 继承，不变 | D1 |
| 宿主 MCP ADR D6：按能力派发、启动前准入 | 继承并扩展 | D4；“从失败处继续”是否同样准入见 O4 |
| 宿主 MCP ADR D6：先持久化再发送，或重启时结束所有受影响的 execution | 选择后者 | D2；停靠会话的旧 execution 怎样结算、是否记录原因见 O8 |
| 宿主 MCP ADR D6：凭据只在持久 execution 记录处于非终态时有效 | 继承，不变 | D3 |
| 宿主 MCP ADR D6：token 随 prompt 进入 Node 持久的会话命令输入，由使用方规定是否接受 | 决定：接受 | D7 |
| 宿主 MCP ADR D2：“首次停住前没有收到 `initialize` / `tools/list`”由使用方处理 | 未定 | O1 |
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
这种情形。随机端口不需要部署配置；多机、容器或沙箱部署必须先另行决定端点（O3）。

## D2：端点凭据在内存里，随 Node execution 有效；Cloud 重启即失效

- 端点凭据在 URL 路径里，是 128 位随机值，按 Node execution 签发，会话换成新的 execution 时旧凭据
  作废。Node 在同一 execution 内重启 Agent 时，新进程拿到原样的宿主列表，凭据继续有效。
- 凭据和 token 只在 Cloud 进程内存中。重启时没跑完的执行按 lifecycle ADR D3 标为中断；除跨重启等待的停靠会话外，都在宿主
  Server 监听之前结束。Cloud 进程被杀时 Node 可能残留；下次启动在恢复执行和提供宿主 Server 之前，先停掉本数据目录下
  的全部旧 Node，停不掉就不启动。所以重启必然换 Node，旧凭据没有一个仍有效。跨重启等待的节点保留完成策略与停住那一轮的信号；它的下一轮启动新 Agent，拿到新凭据。
  重建后的轮次与信号记录的 `turnKey` 不一致时，信号丢弃。
- 同一 execution 的 start 若重发，必须携带逐字节相同的宿主列表；不靠重发轮换凭据。
- 宿主 MCP ADR D6 的 execution 指 Node 协议的 execution：停靠的 `agent` 节点跨重启保留持久会话记录，重启必然换 Node、
  旧凭据全部失效，下一轮是新的 execution，因此满足“重启时结束所有受影响的 execution”。旧 execution 的结算与原因记录见 O8。

**为什么不持久化凭据**：宿主 MCP ADR D6 允许“重启时结束所有受影响的 execution”代替持久化；Cloud 重启本来就中断
全部执行，持久化只会多一处保存秘密。代价是 Cloud 以后若让 execution 活过重启，本条必须重新决定。

## D3：每次调用都按凭据、token 与持久状态三重授权

- attempt token 每一轮签发一个，随 prompt 之后单独的一块交给 Agent。轮次换到新的 execution 时签发新 token，
  旧 token 和用它提交的请求作废；Node 在同一 execution 内重启 Agent 时的做法见 O6；轮次停住、节点完成、失败、取消后 token 失效。
- 一次调用被接受，必须同时满足：凭据有效；token 是同一个节点当前某一轮的 token；持久状态显示该节点正在自己
  没有结束的会话里执行一轮，且它所属的运行仍在运行。任一不满足都不留下任何信号，并回答稳定的原因：
  `unknown attempt token`、`stale_attempt`、`not_awaiting`；持久状态读不到时回答 `temporarily_unavailable`，不作废
  token。
- 同一轮重复调用返回同一个 `signalId`。其他节点的凭据或 token 一律拒绝，即使工具在它的 Agent 里可见。
- 请求只绑定发起它的那一轮；停住时取走，之后到达的请求不可能落到下一轮。

**为什么还要看持久状态**：内存里的记录可能落后于停止、取消与失败；以持久状态为准，已结束的执行不会因内存延迟
而仍可调用（宿主 MCP ADR D6）。

## D4：启动准入先确认 Node 能力，缺了就拒绝，不降级

- 冻结图里有 `agent` 策略的根作用域交互节点时，启动先确认项目的 Node 声明宿主能力；Node 没有运行则先启动它。
  不具备时返回 `workflow_completion_tool_unavailable`，没有节点开始运行，运行状态不变。Node 起不来之类的临时故障按执行服务
  不可用报告，不算缺少完成工具。
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
  不认识的 execution，按结构脱敏。

**为什么接受**：不持久化的承载方式需要改 Node 协议，而它挡不住同一身份下读取当前 prompt 的进程。开启自动接受的
节点没有人工兜底，影响更大，仍接受（D5）。本条只接受 token，端点凭据见 O7。

## 不变量

1. 只有根作用域、`completionSignal: "agent"` 的交互节点的会话带完成工具。
2. 端点凭据属于别的节点或已经换掉的 execution，或者 token 属于别的节点或已经停住的轮次时，调用不改变任何状态。
3. 持久状态不显示节点正在自己的会话里执行一轮时，调用不产生信号。
4. 冻结图需要完成工具而 Node 不具备能力时，没有节点开始运行；任何情况都不降级为 `end_turn`。
5. Cloud 重启之后，重启前签发的凭据与 token 都无效；停靠节点的策略与有效信号仍在。
6. 首轮不自动接受；节点通过完成门禁并提交完成结果之前，后继不启动，`force` 只按完成信号 ADR D3 的条件推进。
7. Cloud 自己的日志、事件、数据库、会话历史与诊断文件里没有凭据或 token。
8. 恢复默认值不改写任何已保存的 `completionSignal`、快照或冻结的图。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 完成工具做成插件 MCP | 不改宿主接入路径，但依赖插件安装与市场，违背完成信号 ADR D1；包内 URL 写死且必须是 https |
| 解析回答里的完成标记 | 不需要任何宿主能力，但要剥离文本，模型会漏写或误写，完成信号 ADR 已否决 |
| 凭据持久化，重启后原样可用 | 在让 execution 活过重启的部署里更优；Cloud 重启即中断，持久化只多一处秘密（D2） |
| 只用端点凭据，不要 attempt token | 少一个秘密到达模型；但无法区分同一 Agent 的各轮，过期的请求会落到新的一轮 |
| Node 不具备能力时按 `end_turn` 执行 | 用户不必升级 Node；但实际执行与冻结的图不一致，是宿主 MCP ADR D6 禁止的静默降级 |
| 继续默认 `end_turn` | 少一种要等上报的状态；但与 desktop 不一致，也不符合默认值 ADR D4 |

## 待 owner 决定

下列取舍没有结论，本文不替 owner 选择；编号沿用评审过程，O2、O5 的结论已写进 D2、D5。每项写明当前实现的做法，便于评审，它不等于决定。

| # | 问题 | 选项 | 当前实现 |
| --- | --- | --- | --- |
| O1 | 宿主 MCP ADR D2 建议把“首次停住前没有收到 `initialize` / `tools/list`”当作宿主能力缺失的信号，由使用方处理 | A：不单独识别，节点停在 `agent_signal_pending`，由人继续或“仍然完成”；B：识别后在节点上显示工具没有连上；C：识别后让节点失败 | A |
| O3 | Agent 到不了 Cloud 端点的部署（多机、容器、沙箱） | A：不提供宿主能力，编辑器仍默认 `agent`，显式 `agent` 在启动准入被拒绝；B：这类部署的编辑器默认写 `end_turn`；C：出现这类部署时另立决策 | 只有单机部署，没有实现 |
| O4 | “从失败处继续”是否与启动同样准入 | A：同样准入，缺能力即拒绝；B：只在派发时失败 | A |
| O6 | 插件刷新在当前 prompt 结束后、下一条 prompt 获准进入 Agent 前重启进程。已结束轮次的 token 按 D3 失效。如果下一轮已排队并已签发 token，是否要在它交给新 Agent 前重签？完成信号 ADR D1 要求 Agent 替换时签发新 token，Node 目前不通知 Controller 这次重启 | A：沿用为该轮签发的 token，端点凭据随 execution 有效；B：重签该轮 token，需要 Node 协议通知重启 | A |
| O7 | 是否接受 Agent 自己的存储或宿主插件日志中的 Agent 自写 stderr 留存端点凭据（URL 路径）？Node 自身的持久输入与诊断继续遵守宿主 MCP ADR D3 | A：接受这两处的留存风险，与 D7 同一信任域；B：要求避免这两处留存，另行决定承载方式 | 一次真实 Node 与 OpenCode 的运行中，在检查的 Node 会话文件与数据库、OpenCode 数据库与日志、Cloud 数据库中未发现凭据；未验证 Claude Code |
| O8 | 停靠会话跨重启时，旧 execution 是否要由 Cloud 在宿主 Server 开始服务前经正常结束路径结算，并记录有名字的原因（宿主 MCP ADR D6“启动时检查可恢复性”与 Controller 用例） | A：不要求，换 Node、凭据失效即可，Node 下次启动时自己把旧 execution 结算为 `interrupted`；要在继承表写明修改 D6 的这一条；B：保留会话，另由 Cloud 在提供服务前结算旧 execution 并记录重启原因；C：重启时连会话一起结束，放弃跨重启等待 | 非停靠会话在提供服务前结束并记录重启原因；停靠会话只保留，Cloud 不持久化 Node execution，不结算、不记录原因；Node 之后上报的结束事件没有对应会话，Cloud 只确认收到 |

## 风险与落地顺序

- **工具没有接上却没有提示**：Agent 可能丢掉连不上的 Server 而照常启动，节点只表现为一直没有上报（O1）。
- **同一信任域**：见 D5、D7；D3 只能防误用和重放，挡不住同一系统用户下的恶意进程。
- **默认值改变用户习惯**：新节点要等上报，编辑器在完成判定旁说明这一点。
- **未解决**：多机或隔离部署的端点与转发（O3）；宿主值的会话内轮换；desktop 是否也检查 HTTP MCP。

落地顺序：

1. **本 ADR 评审**：owner 定 O1、O3、O4、O6–O8，按结论改正文后转为 `approved`。同一变更在默认值 ADR 标题下加
   `> 历史决策：` 指向本文，并依据最终方案补核心用例：工作流侧放在 `test-cases/cloud/workflow/`，Controller 侧
   的义务继续登记在[Controller 交付宿主 MCP 用例](../../../test-cases/controller/session/host-mcp-servers.md)。
2. **`implemented`**：实现合入、证据齐备后，按最终实现同步本文与用例证据，缺口如实保留为 `Partial` / `Missing`。

从恢复版本起改变的只有 `agent` 策略的交互节点和新节点的默认值；`end_turn` 节点、非交互节点、插件 MCP、Node 协议
与 desktop 的行为不变。
