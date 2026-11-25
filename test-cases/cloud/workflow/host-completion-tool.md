# 经 Node 宿主 MCP 的完成工具

依据[Cloud 经 Node 宿主 MCP 提供完成工具](../../../decisions/cloud/workflow/20261124-host-completion-tool-for-node-hosted-interactive-nodes.md)
（D1–D8）。这些用例覆盖 Cloud v2 把完成工具交给哪些会话、怎样授权一次上报、能力不足时怎样拒绝、停靠节点怎样跨重启，
以及凭据与 token 的留存边界。提供宿主 Server 一方的通用义务（文法、按能力派发、凭据有效期）在
[Controller 交付宿主 MCP](../../controller/session/host-mcp-servers.md)；默认值恢复为 `agent` 见
[From The Restore Version Explicit Agent Policies Take Effect Without Rewriting Saved Values](end-turn-default-before-completion-tool.md#from-the-restore-version-explicit-agent-policies-take-effect-without-rewriting-saved-values)；
desktop 上完成信号的门禁见[Agent 完成信号用例](../../desktop/core/workflow/agent-completion-signal.md)。

ADR 为 `implemented`。证据基线是 cloud main 的合并提交
[`601a560f`](https://github.com/ora-space/cloud/commit/601a560f)：v2-server 的单元测试与真实 PostgreSQL 运行测试、前端测试。
除非另外写明，运行测试用假的 Agent 与 Node；没有一项义务由真实 Node 或 Agent 的自动测试证明。

## Only Root Agent Policy Nodes Get The Completion Tool

- 风险：完成工具进入 `end_turn`、非交互节点、容器成员、普通对话或任务运行的会话，Agent 能完成不归它的节点；`agent`
  节点没有拿到工具，永远停在等待上报；工具混入插件 MCP 的选择；端点在本机回环之外可达。
- 前置状态：单机部署；图里有根作用域的 `agent` 与 `end_turn` 交互节点、非交互节点、Loop 或 Iteration 的成员。
- 触发：启动各节点的会话并发送后续轮次；启动普通对话和任务运行；向端点发送各种请求。
- 必须成立：只有根作用域、`interactive`、`completionSignal: "agent"` 的节点的会话带 `ora-workflow-completion`；列表只随
  start 发送一次，后续轮次不带；端点在 `127.0.0.1`，只有 `ora_complete_workflow_step` 一个工具，只回普通 JSON，限制请求
  大小；插件 MCP 的选择不变。
- 禁止结果：其他会话带有宿主列表或 token；后续轮次再次发送宿主列表；端点列出其他工具或提供事件流。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 只有根作用域的 `agent` 交互节点带完成工具 | [`TestOnlyAnAgentPolicySessionGetsTheCompletionTool`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：`end_turn` 节点的首轮与后续轮次、非交互节点都不带，`agent` 节点带；[`TestHostToolsNeedANodeThatTakesThem`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go) 断言没有宿主工具的会话不带字段。缺口：没有 Loop 与 Iteration 成员、普通对话与任务运行的会话测试 | `Partial` |
| 列表只随 start 发送，token 按轮附在 prompt 之后 | [`TestHostToolsReachTheAgentAndNothingElse`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go)：start 带一项 Server，后续轮次不带，每轮末尾是各自的 token 块 | `Covered` |
| 端点只有一个工具、只回普通 JSON、限制大小 | [`TestTheEndpointServesTheOneTool`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：`tools/list` 只有一项，其他工具与方法报错，`GET` 为 405，超长请求为 413。缺口：没有测试请求时长上限，也没有直接断言只绑定 `127.0.0.1` | `Partial` |
| 不进入插件 MCP 的选择 | 会话启动测试：同时有插件 MCP 选择与宿主列表时，插件选择与宿主列表互不影响 | `Missing` |

- 决策依据：D1，不变量 1；完成信号 ADR D1。

## A Report Counts Only For Its Own Live Turn

- 风险：别的节点的凭据或 token 完成本节点；上一轮的 token 落到下一轮；持久状态已经不在执行，内存里的记录仍接受调用；
  被停止、被截断或被拒绝的一轮留下有效信号；execution 已经结束而轮次还没有结束时，调用仍被接受，之后才到的握手被记为已连上；
  结算写不进库时凭据一直可用；换成新的 execution 之后，新凭据因旧 execution 结束而失效；插件刷新重启 Agent 之后，已排队的一轮的 token 失效，Agent 无法上报。
- 前置状态：两个 `agent` 节点各在执行一轮；另有停靠、已完成、已取消的节点。
- 触发：用自己的、别的节点的、上一轮的、编造的 token 调用；重复调用；停止、截断、拒绝、丢失 Agent 后再停靠；持久状态读
  不到时调用；Node 在同一 execution 内重启 Agent，下一轮的 prompt 已经排队；execution 以各种方式结束
  而轮次还没有结束时调用、握手；读持久状态与回答之间 execution 结束；读取迟迟不返回；会话换成新的 execution。
- 必须成立：只有凭据、token 与持久状态三者都成立才接受；重复调用返回同一个 `signalId`；拒绝给出稳定原因，
  `temporarily_unavailable` 不作废 token；凭据只对签发它的 execution 有效，没有记录、已经结算或本进程已定下原因时，工具调用回答
  `not_awaiting`，其余带 `id` 的请求回 401，读不到时分别是 `temporarily_unavailable` 与 503，都不留信号、不记连接；持久状态只读一次，接受
  之后不再核对；只有正常结算的一轮保留上报；同一 execution 内重启 Agent 后，端点凭据与已排队那一轮
  的 token 继续有效，上一轮的 token 已经失效。
- 禁止结果：被拒绝的调用留下信号；execution 结算之后的调用被接受；上一轮的上报算进下一轮；不是正常结束的一轮带着信号停靠。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 跨节点的凭据或 token 被拒绝 | [`TestAnotherNodesCredentialOrTokenIsRefused`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)、[`TestAReportComesOnlyFromTheNodesOwnLiveTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：两个方向都拒绝，停靠后没有信号 | `Covered` |
| 一轮一个 token，重复调用幂等，停住后失效 | [`TestACallIsAcceptedOnceAndOnlyForItsOwnTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)、[`TestAReportDoesNotOutliveItsTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：重复调用返回同一信号，上一轮的 token 为 `stale_attempt`，旧信号的确认被拒绝 | `Covered` |
| 以持久状态判定一轮是否在执行 | [`TestWorkflowNodeTurnLive`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/workflow_completion_tool_test.go)：未绑定、停靠、轮次已结束、运行已取消，以及 execution 没有记录、属于别的会话、已经结算各自拒绝；[`TestStoredStateDecidesACall`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：读不到时 `temporarily_unavailable` 且 token 不作废；[`TestTheToolFollowsStoredExecutionState`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_execution_test.go)：真实执行服务与 PostgreSQL 上，执行中接受、停靠后不再 live。已知缺口：首轮在会话绑定前立即上报会被拒，没有测试覆盖这个窗口 | `Partial` |
| execution 结束后凭据失效，工具调用与握手都拒绝 | [`TestAnExecutionThatStoppedRunningVoidsItsCredential`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)：真实 PostgreSQL、driver 与端点，轮次在库里一直没有结束；决定结束、发出 `EndSession`、Node 报告结束、Node 丢失、`EndSession` 发不出去与恢复之后，只凭持久记录，工具调用为 `not_awaiting`，`initialize`、`tools/list`、`ping` 为 401，不留信号、不记连接；[`TestARunningExecutionMayUseItsTool`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)：execution 运行时都接受；[`TestAnEndedExecutionsCredentialIsRefused`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：不认识的方法同样拒绝，读不到时为 503 与 `temporarily_unavailable`，别的 execution 的握手照常；[`TestACredentialStandsForTheExecutionItWasIssuedFor`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：新凭据只看新的 execution；[`TestAHostExecutionRunsUntilItIsSettled`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/host_executions_test.go)：真实 PostgreSQL 上，没有记录与已结算的都不算运行 | `Covered` |
| 本进程定下原因而没写进库的也拒绝 | [`TestAnEndIsWrittenBeforeNodeIsAskedForIt`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)：正在写与写不进去时，工具调用与握手都拒绝；[`TestOnlyADecidedEndIsEnding`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)、[`TestAnUnwrittenEndOutlivesItsSession`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)：只有定下原因的算，会话被替换或已不在时仍算；[`TestRuntimeAsksItsDriverWhatIsEnding`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/executionruntime/runtime_held_test.go)：执行服务转给 driver；[`TestTheCompletionToolKnowsWhatThisProcessDecided`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/completion_tool_order_test.go)：只断言 `main.go` 源码里的接线 | `Partial` |
| 持久状态只读一次并限时 | [`TestAnEndBetweenTheReadAndTheAnswer`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：读取之后才写进库的结算不影响这一次，本进程定下的拒绝，别的 execution 的不影响；[`TestAStateThatIsNotReadInTimeIsUnavailable`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：读取迟迟不返回时，握手为 503，工具调用为 `temporarily_unavailable`，token 不作废。读取限时与读请求体叠加时的连接错误是接受的差异，没有测试 | `Covered` |
| 不是正常结束的一轮丢弃上报 | [`TestAReportOfATurnThatDidNotEndNormallyIsDropped`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：停止、截断、拒绝、未知停止与 Agent 丢失都不留信号，强制完成记 `requester: human`；[`TestAReportDoesNotCoverATurnThatDidNotEndNormally`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：拒绝的一轮不被自动接受 | `Covered` |
| 同一 execution 内重启 Agent 后沿用凭据与该轮 token | Controller 与 Node 测试：插件刷新重启 Agent 后，已排队的一轮用原 URL 与原 token 上报成功，上一轮的 token 被拒绝 | `Missing` |

- 决策依据：D2（凭据按 execution）、D3、D5，不变量 2、3；完成信号 ADR D1（本决策修改的一处）、D2、D3。

## Missing Host Capability Refuses Start And Resume Without Downgrade

- 风险：前面的节点用掉模型额度之后才发现没有完成工具；能力不足时悄悄按 `end_turn` 执行；Node 的临时故障被报成缺少完成工具；
  “从失败处继续”绕过启动时的检查。
- 前置状态：冻结图含根作用域的 `agent` 节点；Node 分别声明与不声明 `session_host_mcp_servers`，Agent 分别支持与不支持
  HTTP MCP；另有一个失败的运行可以从失败处继续。
- 触发：启动运行（Node 已运行或没有运行）；从失败处继续；准入之后换成不具备能力的 Node；启动不声明 HTTP MCP 的 Agent。
- 必须成立：Node 没有运行时先启动它再判断；启动与“从失败处继续”缺能力时以 `workflow_completion_tool_unavailable` 拒绝，没有节点开始运行，运行状态不变；
  没有 Agent 运行时的部署同样以此拒绝，Node 的临时故障按执行服务不可用报告；派发时缺能力以 `host_mcp_node_capability_missing` 失败，Agent 不声明 HTTP MCP 以
  `host_mcp_http_capability_missing` 失败；不含 `agent` 节点的图不受影响。
- 禁止结果：`agent` 节点按 `end_turn` 执行；被拒绝的启动或继续留下运行中的节点。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 准入只看根作用域、图里已连入的 `agent` 交互节点 | [`TestUsesAgentCompletionCountsOnlyParkedNodes`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflow/completion_internal_test.go)：非交互节点、图里未连入的节点（`spare`）与 Loop 成员不触发准入检查。缺口：没有 Iteration 成员 | `Partial` |
| Node 没有运行时先启动它再判断 | [`TestHostToolsNeedANodeThatTakesThem`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go) 与 [`TestCheckHostToolsAcceptsACapableNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go)：driver 没有 Node 时先启动再判断，不具备与具备能力各得到拒绝与通过。缺口：只在 driver 层，没有从启动一个运行开始的测试 | `Partial` |
| 启动准入拒绝，临时故障不算缺少能力 | [`TestARunIsRefusedAtItsStartWithoutTheTool`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：缺能力时运行仍为 pending、没有会话启动；Node 不可用与请求取消返回原错误；`end_turn` 图照常启动；[`TestARunIsRefusedWithoutAnAgentRuntime`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_execution_test.go)：没有 Agent 运行时以缺少完成工具拒绝，Node 起不来仍是执行服务不可用，运行仍为 pending、没有节点 | `Covered` |
| “从失败处继续”同样准入 | [`TestAResumeIsRefusedWithoutTheTool`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：缺能力、Node 不可用与请求取消各返回自己的原因，运行与节点行逐字段不变，没有会话启动，具备能力后继续并带上工具；[`TestWorkflowRunResumeRefusals`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/server/workflow_run_resume_test.go)：`/resume` 映射为 `400 workflow_completion_tool_unavailable` 与 `503` | `Covered` |
| 派发时再查，Agent 不支持 HTTP MCP 时明确失败 | [`TestHostToolsNeedANodeThatTakesThem`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go)：Node 不声明能力时不发送宿主列表，以 `host_mcp_node_capability_missing` 结束；[`TestAnAgentWithoutHTTPMCPKeepsNodesDetail`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go) 与 [`TestAnAgentWithoutHTTPMCPFailsTheNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：Node 给出的原因原样到达节点，节点与运行失败 | `Covered` |

- 决策依据：D4，不变量 4；宿主 MCP ADR D4、D6。

## Agent Completion Waits For Confirmation Or A Starter Turn

- 风险：Agent 的一次上报不经确认就推进后继；自动接受在首轮或异常停靠时完成节点；自动接受绕过交付物检查；确认了过期的
  信号；`force` 越过过期检查；审计把人的决定记成 Agent 的。
- 前置状态：`agent` 节点，部分开启 `autoAcceptAgentCompletion`，部分声明交付文件。
- 触发：首轮与后续轮次分别带或不带上报；普通完成、确认首轮、确认信号、强制完成以及它们的组合。
- 必须成立：首轮需要确认首轮；有上报时还要确认该信号；过期信号返回 `workflow_node_signal_changed`，`force` 不能越过；没有
  上报时按完成信号 ADR D3 强制完成；自动接受只作用于人发起、正常结束、只剩 `agent_confirmation_pending` 的一轮，走同一
  门禁；`requester` 与 `confirmation` 由服务端推导；节点提交完成结果之后后继才启动。
- 禁止结果：被拒绝的完成推进了运行；首轮被自动接受；自动接受越过缺失的交付物。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 首轮的两个确认缺一不可，后继在完成之后启动 | [`TestAReportedFirstTurnNeedsBothConfirmations`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：五种不完整的请求各自被拒绝且不推进运行，完成后记 `requester: agent`、`confirmation.by: human` | `Covered` |
| 没有上报时由人确认并强制完成 | [`TestAnUnreportedFirstTurnTakesConfirmationAndForceTogether`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：只确认或只强制都被拒绝，合起来才完成，记 `requester: human` 与被越过的 blocker | `Covered` |
| 过期信号被拒绝，`force` 不越过 | [`TestAReportDoesNotOutliveItsTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：带或不带 `force` 都返回信号已变；[`TestACompletionOfAnotherReportIsStale`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：五种存储与现场的组合逐一判断 | `Covered` |
| 强制越过未确认的信号有审计 | [`TestForcingPastAReportRecordsThatNobodyConfirmedIt`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：保留信号，没有 `confirmation`，记下被越过的 blocker | `Covered` |
| 自动接受只在允许时生效，走同一门禁 | [`TestAutoAcceptTakesAReportedHumanTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：首轮与没有上报的一轮不接受，人发起且上报的一轮完成，记 `confirmation.by` 为自动；[`TestAutoAcceptRefusedByTheGateLeavesTheNodeToTheStarter`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：交付物缺失时留给发起人 | `Covered` |

- 决策依据：D5，不变量 6；完成信号 ADR D3–D6。

## A Parked Node Survives A Restart While Its Old Execution Is Settled

- 风险：重启后旧凭据仍被接受；停靠会话的旧 execution 一直没有结论，也不知道为什么结束；宿主 Server 在旧 execution 结算之前
  就开始服务；为了结算把停靠会话一起结束；重建的轮次与信号对不上却仍然算数；start 已发出而没有记录；同一 execution 记了
  两个原因；关停或失败记成别的原因；结算失败仍然提供服务。
- 前置状态：`agent` 节点停靠等待，有或没有有效信号；另有没有停靠、正在执行的会话。
- 触发：停止 Cloud 进程（正常退出或被杀）后启动；用旧凭据与旧 token 调用；在停靠节点继续对话并完成；Cloud 结束会话、Agent 自己
  结束、Node 丢失或被换掉，包括发生在写入记录期间；记录或结算写不进库；Node 补报旧的结束事件。
- 必须成立：旧凭据与 token 全部失效；停靠节点的会话、完成策略与同一轮的信号仍在，可以继续对话并完成；在宿主 Server 开始
  服务之前，停靠会话的旧 execution 经正常结束路径结算，并记录有名字的重启原因；下一轮是新的 execution，拿到新的凭据与
  token；重建后与 `turnKey` 不一致的信号丢弃；升级前按 `end_turn` 停靠的节点不附加完成工具；带宿主工具的 execution 先有记录再发 start，
  写不进就不发；Cloud 决定结束时先写结算再发 `EndSession`（正常关停时的停靠会话除外），写不进库也照样发；每个 execution 只结算一次，原因是最先定下的那个；启动结算失败则整个恢复回滚、不启动；正常关停时停靠会话不结算，
  正在执行的一轮记 `session_ended`；没有 execution 或没有记录的停靠会话不补记。
- 禁止结果：重启前的凭据被接受；旧 execution 没有结算就开始服务；停靠会话因重启结束；会话带着运行中的 execution 结束；补报的事件改写
  原因。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 停靠会话、策略与信号保留，凭据与 token 换新 | [`TestAReportSurvivesARestartButItsCredentialsDoNot`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：同一信号与 blocker 仍在，旧 token 被拒绝，新的 token 可以上报；[`TestRecoveryKeepsTheSessionOfAParkedInteractiveNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/interactive_recovery_test.go)：停靠会话仍打开，正在执行的会话结束。缺口：没有用重启前的端点 URL 再调用 | `Partial` |
| 停靠会话的旧 execution 在服务前结算，并有有名字的原因 | [`TestARestartSettlesTheExecutionOfAParkedNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_execution_test.go)：真实 PostgreSQL 与执行服务上，按 `main.go` 的步骤重启，提供完成工具之前旧 execution 已以 `server_restarted` 结算，会话仍打开，信号与 blocker 仍在，下一轮是新的 execution，可以继续并完成；[`TestRecoverySettlesRunningHostExecutions`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/host_executions_test.go)：停靠与正在执行的都结算，停靠会话仍打开、正在执行的会话结束。缺口：`main.go` 的先后由下一项的源码顺序测试固定，没有在真实启动里核对宿主 Server 接受第一个请求之前的状态 | `Partial` |
| start 之前已有记录，写不进就不发 | [`TestAHostExecutionIsRecordedBeforeItsStart`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：Node 收到 start 时记录已在，没有宿主工具的会话不记，后续轮次不再记；[`TestAStartIsNotSentWithoutItsRecord`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：写入失败、没有写入或结算能力时都不发 start；[`TestARecordWaitsForTheEndOfItsSession`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/host_executions_test.go)：真实 PostgreSQL 上，会话正在结束时写入等待并被拒绝 | `Covered` |
| 每条结束路径记各自的原因，只结算一次 | [`TestEndSettlesAHostExecution`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)、[`TestAnEndedAgentSettlesItsHostExecution`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)（停靠中与一轮中）、[`TestALostNodeSettlesItsHostExecutions`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)、[`TestARetiredNodeSettlesItsOtherHostExecutionsAsLost`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)、[`TestAFailureThatEndsTheSessionIsTheReason`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)（事件写不进去、历史读取失败、MCP 选择变化、MCP 授权被撤）：各记自己的原因；[`TestWhatEndsAnExecutionWhileItIsRecordedSettlesIt`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：写入记录期间的 End 与 Node 丢失同样结算；[`TestAHostExecutionIsSettledOnce`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/host_executions_test.go)：后来的原因不改写，会话结束时仍在运行的记 `session_ended`；[`TestSettlingRacesKeepOneReason`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：并发结算只留一个原因 | `Covered` |
| 决定结束时先写结算，再发 `EndSession` | [`TestAnEndIsWrittenBeforeNodeIsAskedForIt`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)：写完之前没有发 `EndSession`，写不进去时照样发、之后写入；[`TestAnExecutionThatStoppedRunningVoidsItsCredential`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_gate_test.go)：`EndSession` 发不出去时已经结算，之后的失败不改原因 | `Covered` |
| 已定的原因写不进库时不丢，会话结束时一并写入 | [`TestAHostExecutionsEndIsWrittenOnceTheStoreTakesIt`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)、[`TestAnUnwrittenEndIsKnownUntilItIsWritten`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)、[`TestNothingComesBetweenSettlingAnExecutionAndListingIt`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：后台重试到写入为止，其间始终列得出原因；[`TestEndingASessionKeepsADecidedReason`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/host_executions_test.go)、[`TestRuntimeEndsASessionWithItsDriversReason`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/executionruntime/runtime_held_test.go)：会话收尾用已定的原因，重试的写入不再改变它 | `Covered` |
| 启动结算失败则不启动 | [`TestAStartThatCannotSettleHostExecutionsIsRefused`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/host_execution_startup_test.go)：结算被数据库拒绝时，启动返回错误、没有调用主 HTTP 的监听回调，记录与会话保持原样；能结算时，进入主 HTTP 监听回调之前已是 `server_restarted`。它不守宿主 Server 的监听，先后见“宿主 Server 在旧 Node 停止与恢复之后才监听”一项 | `Covered` |
| 没有可结算的不补记，Node 补报不改写 | [`TestRecoverySettlesRunningHostExecutions`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/store/host_executions_test.go)：Node 早已丢失的保留原来的原因与时间，`end_turn` 与升级前的会话没有记录，再次重启不变，补报不改写；[`TestATerminalOfAnEarlierProcessSettlesNothing`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：不认识的 execution 的结束事件只确认 | `Covered` |
| 正常关停不结算停靠会话，正在执行的一轮记 `session_ended` | [`TestAShutdownLeavesHostExecutionsToTheNextStart`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：关停 driver 时停靠会话没有结算。缺口：没有测试正在执行的一轮，也没有测试还在重试的原因被下次启动记成 `server_restarted` | `Partial` |
| 宿主 Server 在旧 Node 停止与恢复之后才监听 | [`TestTheCompletionToolIsServedAfterEarlierNodesStopped`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/completion_tool_order_test.go)：只断言 `main.go` 源码里的顺序，没有运行重启 | `Partial` |
| 对不上的信号丢弃 | [`TestARestartDropsAReportOfAnotherTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：多出一轮时信号不再算数 | `Covered` |
| 升级前的停靠节点保持 `end_turn` | [`TestANodeParkedBeforeTheToolStaysEndTurn`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_test.go)：摘要记录为 `end_turn` 的节点在重启后不带工具，由发起人普通完成 | `Covered` |

- 决策依据：D2、D6，不变量 5、8、10；宿主 MCP ADR D6“启动时检查可恢复性”。

## An Unconnected Completion Tool Is Shown Without Failing The Node

- 风险：Agent 丢掉连不上的 Server 照常启动，发起人分不清是没有上报还是没有工具；识别之后把节点判为失败，丢掉已完成的产出；把别的
  execution 的连接算进来，或者在没有 execution 可说时改掉上一次的结论。
- 前置状态：`agent` 节点的 execution 停住之前，端点分别收到与没有收到 `initialize` / `tools/list`；另有换过 execution、没能启动
  execution、会话已经结束的节点。
- 触发：首轮正常停住、被停止或失败；之后连上再停住；Node 丢失或 Cloud 重启后换成新的 execution；start 没有发出；消息因会话已结束被
  拒绝；凭据换掉之后旧请求才到；发起人查看节点、继续对话、强制完成。
- 必须成立：没有收到时节点上提示完成工具没有连上，文字写明是上次停住时；收到任意一个就没有提示；节点与运行的状态、完成门禁不变，可以
  继续对话或“仍然完成”；同一 execution 之后连上，下一次停住时去掉；换成新的 execution 时按它的首次停住重新判断，在那之前（包括重启
  之后）保留上一次的结论；没有 execution 可说时保留上一次的结论；提示不含凭据或 token。
- 禁止结果：节点因此失败；提示改变 blocker；`end_turn` 节点出现提示；换掉的 execution 的连接算作新 execution 的。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 按凭据记录，停住前没有收到时提示 | [`TestAnAgentThatDidNotConnectIsSaidOnTheNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：什么都没收到与只收到 `ping` 有提示，收到 `initialize` 或 `tools/list` 没有；[`TestAConnectionIsRecordedForTheStartedAgent`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：其他方法、没有 `id` 的通知、别的节点与换掉的凭据都不算；[`TestALateConnectionIsNotTheNewAgents`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：授权之后、读到请求之前换掉凭据，不算新 execution 的；[`TestAnEndedExecutionsCredentialIsRefused`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/completiontool/server_test.go)：凭据失效或持久状态读不到而被拒绝的握手不算 | `Covered` |
| 提示跟着 execution 走 | [`TestAConnectionOfTheSameExecutionEndsTheNotice`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：执行中仍在，连上后的下一次停住去掉，不再出现；[`TestANewExecutionIsJudgedAtItsFirstPark`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：新 execution 首次停住时重新判断，在那之前保留；[`TestTheNoticeSurvivesARestart`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：真实 PostgreSQL 上重启后保留，下一轮的 execution 单独判断；[`TestARefusedMessageParksWithWhatIsKnownByThen`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：消息被拒而 execution 仍在时按最新记录 | `Covered` |
| 首轮异常停住也判断 | [`TestAFirstTurnThatEndedAbnormallyIsJudgedToo`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：失败、拒绝、取消与发起人停止，收到与没有收到各一遍 | `Covered` |
| 没有 execution 可说时保留上一次的结论 | [`TestAnExecutionThatWasNotStartedIsNotJudged`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：签发凭据之前与之后失败，首轮没有提示，后来的一轮保留结论，包括旧 execution 之后才连上；[`TestARefusedMessageParksWithWhatIsKnownByThen`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：会话已结束而被拒的消息；[`TestAStartThatCouldNotBeSentIsNotSaidToBe`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)、[`TestAStartIsNotSentWithoutItsRecord`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_execution_test.go)：start 写给 Node 失败或没有发出时，driver 不报告已发出 | `Covered` |
| 提示不改变状态与门禁 | [`TestAnAgentThatDidNotConnectIsSaidOnTheNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：有无提示时 blocker、停靠种类与运行状态相同，继续对话、普通完成被拒与强制完成相同，完成后的摘要没有提示；[`TestAnEndTurnNodeSaysNothingOfTheTool`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：`end_turn` 节点没有提示；[`run-node-session-actions.test.tsx`](https://github.com/ora-space/cloud/blob/601a560f/frontend-v2/src/features/chat/workflow-run/run-node-session-actions.test.tsx) 的 “offers the same actions with the completion tool connected or not”：页面的操作相同 | `Covered` |
| 发起人与其他成员看得到提示 | [`run-node-session-actions.test.tsx`](https://github.com/ora-space/cloud/blob/601a560f/frontend-v2/src/features/chat/workflow-run/run-node-session-actions.test.tsx) 的 “tells the starter and everyone else that the completion tool was not connected when the node last parked”：中英文原文，等待与执行中，发起人与其他成员；[`server-run-adapter.test.ts`](https://github.com/ora-space/cloud/blob/601a560f/frontend-v2/src/features/chat/workflow-run/server-run-adapter.test.ts) 的 “reads that the Agent of a parked node did not connect to its completion tool”：只有布尔 `true` 算 | `Covered` |
| 提示的存储与显示没有凭据或 token | [`TestAnAgentThatDidNotConnectIsSaidOnTheNode`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/completion_tool_unconnected_test.go)：节点 payload 里没有凭据与 token；上一项的前端测试比较页面文字的全文。缺口：没有检索接口的回答 | `Partial` |

- 决策依据：D8，不变量 9；宿主 MCP ADR D2。

## Completion Credentials And Tokens Stay Out Of Cloud Surfaces

- 风险：端点凭据或 token 进入 Cloud 的日志、事件、数据库、会话历史、最终回答或 Node 事件诊断文件；重放的旧事件没有脱敏；
  Node 自己的持久输入或诊断留下端点凭据。
- 前置状态：`agent` 节点的 Agent 在回答与工具调用里回显 token 与端点 URL；Node 重放本进程不认识的 execution 的事件。
- 触发：正常结束一轮；Node 重建后重放事件；格式化宿主列表；检索各处存储。
- 必须成立：Cloud 的各个持久面都没有凭据或 token，token 块从发布的历史里去掉；未知 execution 的 Agent 会话消息按结构脱敏，
  clone 与插件安装的结果除外；
  Node 自身的持久输入、日志与诊断没有端点凭据。允许留存的只有：token 在 Node 的会话文件与 ledger 以及 Agent 自己的存储里；
  端点凭据在 Agent 自己的存储与宿主插件日志里 Agent 自写的 stderr 里。
- 禁止结果：上述允许之外的任何位置出现凭据或 token。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 事件、回答与诊断文件没有凭据或 token | [`TestHostToolsReachTheAgentAndNothingElse`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go)：回答与诊断文件里换成占位符，发布的事件里没有 token 块；[`TestTheJournalHasNoCredentialOfAnUnknownExecution`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go)：未知 execution 的四类会话消息在诊断文件里都没有 token 与凭据；[`TestTheJournalKeepsThePreparationResults`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/agentruntime/host_tools_test.go)：只有 clone 与插件安装的结果原样保留，未知 execution 的其他消息仍去掉 token | `Covered` |
| 节点、轮次与 prompt 的存储没有 token | [`TestTheToolFollowsStoredExecutionState`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/workflowhost/agent_completion_execution_test.go)：真实 PostgreSQL 上节点与轮次（含 prompt）都不含 token。缺口：没有断言端点凭据，没有检索整个数据库与进程日志 | `Partial` |
| 宿主列表不会被格式化出来 | [`TestAHostMCPServerNeverFormatsItsCredentials`](https://github.com/ora-space/cloud/blob/601a560f/v2-server/internal/execution/hostmcp_test.go) | `Covered` |
| 真实 Agent 的留存在允许范围内 | 真实 Node 与 OpenCode、Claude Code 各运行一次，检索 Node、Agent 与 Cloud 的存储、日志和进程参数。OpenCode 在 Node `447a59de` 上人工检索过一次，没有命中端点凭据，没有自动测试固定；Claude Code 没有验证 | `Missing` |

- 决策依据：D7，不变量 7；宿主 MCP ADR D3、D6。
