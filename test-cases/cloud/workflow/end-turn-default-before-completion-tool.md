# 完成工具送达前的默认完成方式

依据[经 Node 承载的交互节点默认 `end_turn`](../../../decisions/cloud/workflow/20261119-end-turn-default-until-host-completion-tool.md)
（D2、D3、D4）。这些用例覆盖经 Node 承载的 Controller（目前是 Cloud v2）在接入完成工具之前怎样写默认值、怎样
执行显式 `agent`，图在 desktop 与 Cloud v2 之间导入时怎样保留 `completionSignal`，以及恢复版本之后的行为。
完成工具怎样经 Node 送达不在这里，见[Controller 交付宿主 MCP](../../controller/session/host-mcp-servers.md)；
desktop 上 `end_turn` 节点与旧快照的 legacy 行为见
[Legacy End Turn Nodes And Old Snapshots Stay Unchanged](../../desktop/core/workflow/agent-completion-signal.md#legacy-end-turn-nodes-and-old-snapshots-stay-unchanged)。

ADR 为 `implemented`。证据基线是 cloud main 上 [cloud#403](https://github.com/ora-space/cloud/pull/403) 的合并提交
[`2b80c023`](https://github.com/ora-space/cloud/commit/2b80c023)：编辑器测试、v2-server 单元测试和 `frontend-v2/e2e` 的真实 Node
用例。只有测试失败会直接说明义务不成立时才登记为证据；没有测试跑过 `agent` 策略的交互节点，所以“按 `end_turn`
执行”一类义务如实保留为 `Missing`。恢复版本（D4）还不存在，最后一个用例的义务留待该版本验证。

## New Interactive Nodes Default To End Turn Before The Completion Tool Ships

- 风险：编辑器给新交互节点写 `agent`，用户看到的默认值和实际执行不一致，完成工具上线时所有按默认值建出的
  节点一起改变行为；或者编辑器在保存时改写节点已有的 `completionSignal`，覆盖用户的显式选择；或者给没有该
  字段的旧节点补写值，改变旧图的含义。
- 前置状态：Controller 的产品版本尚未接入完成工具（恢复版本之前）；编辑器打开一张图，图里可能有新建的
  交互节点、已有 `completionSignal: "agent"` 或 `"end_turn"` 的交互节点，以及没有该字段的旧交互节点。
- 触发：新建交互节点；为没有完成策略的 Agent 节点打开交互模式；保存图；关闭交互模式。
- 必须成立：新建交互节点和为没有策略的节点打开交互模式时，写入 `completionSignal: "end_turn"` 与
  `schemaVersion: 4`；已有的 `completionSignal` 原样保存；没有该字段的旧节点保存后仍没有该字段；关闭交互模式
  按完成信号 ADR 的 version-4 边界清理策略字段。
- 禁止结果：新建节点写出 `agent`；保存把已有的 `agent` 改成 `end_turn` 或反过来；给旧节点补写
  `completionSignal`。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 新建交互节点与打开交互模式时写 `end_turn` 和 `schemaVersion: 4` | [`workflow-agent-completion-fields.test.tsx`](https://github.com/ora-space/cloud/blob/2b80c023/frontend-v2/src/features/workflows/editor/workflow-agent-completion-fields.test.tsx) 的 “defaults a node turned interactive to end_turn and clears the policy when turned off”；[`workflow-agent-inactivity-field.test.tsx`](https://github.com/ora-space/cloud/blob/2b80c023/frontend-v2/src/features/workflows/editor/workflow-agent-inactivity-field.test.tsx) 断言打开交互模式只多出 `interactive`、`schemaVersion: 4` 与 `completionSignal: "end_turn"` 三项。v2 编辑器里新建交互节点就是打开交互模式 | `Covered` |
| 已有 `completionSignal` 原样保存，旧节点不补写 | [`workflow-agent-completion-fields.test.tsx`](https://github.com/ora-space/cloud/blob/2b80c023/frontend-v2/src/features/workflows/editor/workflow-agent-completion-fields.test.tsx) 的 “keeps version 4 when the last file is removed from a signal node” 与 “offers auto-accept only for the agent policy and writes it with version 4”：编辑其他字段后 `agent` 保持不变。缺口：没有测试保存一个没有该字段、且不打开交互模式的旧交互节点，也没有覆盖显式 `end_turn` | `Partial` |

- 决策依据：D2（写图），不变量 1、5。

## An Explicit Agent Policy Runs As End Turn And Is Marked Unapplied

- 风险：恢复版本之前，显式 `agent` 的节点按 `agent` 门禁执行，永远等不到完成信号；或者按 `end_turn` 执行却
  没有任何标记，成为宿主 MCP ADR D6 禁止的静默降级；或者 Controller 探测 Agent 与 Node 能力，同一产品版本
  在不同部署上执行不同的策略；`autoAcceptAgentCompletion` 在没有信号的情况下自动完成节点。
- 前置状态：恢复版本之前；冻结的图里有 `completionSignal: "agent"` 的交互节点，部分开启
  `autoAcceptAgentCompletion`。
- 触发：启动 WorkflowRun；节点首轮和后续轮次结束；发起人完成节点；在编辑器里查看该节点；查看运行页。
- 必须成立：节点按 `end_turn` 的完成门禁执行（首轮确认与 `force` 规则不变），不附加宿主 MCP、不签发
  attempt token；自动接受不生效，仍由发起人完成；运行记录把 `completionSignal` 与
  `autoAcceptAgentCompletion` 列为未应用的设置；编辑器在这两项旁显示“暂不生效”，并说明完成工具上线后会
  开始生效；结果不随 Agent 或 Node 的能力变化。
- 禁止结果：节点因缺少完成信号而无法普通完成；会话带着宿主 MCP 或 attempt token 启动；自动接受完成了节点；
  运行记录或编辑器没有“暂不生效”的标记；同一产品版本下，某些部署按 `agent` 执行。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| `agent` 节点按 `end_turn` 门禁完成，会话不带宿主 MCP 与 token | Controller 运行测试（真实 PostgreSQL）：`agent` 策略的交互节点正常结束后由发起人完成，检查启动参数没有宿主列表。现有的交互运行测试与 e2e 只用 `end_turn` 或没有策略的节点 | `Missing` |
| 自动接受不生效 | Controller 运行测试：开启 `autoAcceptAgentCompletion` 的节点在回合结束后保持等待，由发起人完成。没有直接测试 | `Missing` |
| 运行记录与编辑器把两项标为暂不生效 | [`TestAnalysisReportsUnappliedAgentSettings`](https://github.com/ora-space/cloud/blob/2b80c023/v2-server/internal/workflow/analysis_test.go) 断言 `agent` 策略节点的未应用设置正是 `completionSignal` 与 `autoAcceptAgentCompletion`，且与运行写入 payload 的 `Agent().Unapplied` 同源；[`workflow-agent-completion-fields.test.tsx`](https://github.com/ora-space/cloud/blob/2b80c023/frontend-v2/src/features/workflows/editor/workflow-agent-completion-fields.test.tsx) 的 “marks the agent policy and auto-accept as not applied where the server says so” 与 [`workflow-interactive.spec.ts`](https://github.com/ora-space/cloud/blob/2b80c023/frontend-v2/e2e/workflow-interactive.spec.ts) 的 “the Agent inspector edits the completion rule and deliverables, and marks the agent rule as not applied”。缺口：提示只说明现在按“回合正常结束即可”执行，没有说明完成工具上线后会开始生效；也没有运行级测试检查 payload 里的这两项 | `Partial` |
| 执行策略不取决于运行时探测 | Controller 测试：Node 声明或不声明 `session_host_mcp_servers` 时，同一节点的执行与记录一致。[`TestAnalysisReportsUnappliedAgentSettings`](https://github.com/ora-space/cloud/blob/2b80c023/v2-server/internal/workflow/analysis_test.go) 与 [`TestAgentReadsCompletionSettings`](https://github.com/ora-space/cloud/blob/2b80c023/v2-server/internal/workflow/completion_internal_test.go) 只说明结果由节点配置和位置决定，没有变化 Node 能力 | `Missing` |

- 决策依据：D2（显式选择、执行、判断依据），不变量 2、3；宿主 MCP ADR D6 的适用时点（继承与修改表）。

## Cross-Host Import Keeps The Completion Signal As Written

- 风险：图在 desktop 与 Cloud v2 之间导入时按目标端的默认值改写 `completionSignal`，用户在一端显式选的
  `end_turn` 或 `agent` 被覆盖；来回导入一次，图的含义就变了；没有该字段的旧图在某一端被补写。
- 前置状态：分别由 v2 编辑器（写 `end_turn`）与 desktop 编辑器（写 `agent`）保存的图，以及没有
  `completionSignal` 的旧图。
- 触发：把图从 Cloud v2 导入 desktop、从 desktop 导入 Cloud v2，再导回原端。
- 必须成立：每次导入后 `completionSignal` 与原图相同，缺失的字段仍缺失；v2 写的 `end_turn` 在 desktop 上是
  显式 `end_turn`，不注入完成工具；desktop 写的 `agent` 在恢复版本之前的 Cloud v2 上按 `end_turn` 执行并标为
  暂不生效。
- 禁止结果：导入改写或补写 `completionSignal`；来回导入后值不同；desktop 写的 `agent` 在 Cloud v2 上没有
  “暂不生效”的标记。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 两个方向的导入都不改写或补写 `completionSignal` | [`workflow-transfer.test.ts`](https://github.com/ora-space/cloud/blob/2b80c023/frontend-v2/src/lib/workflows/editor/workflow-transfer.test.ts) 的 “exports and imports the frozen snapshot policy … despite later draft edits”：v2 的导出与导入原样保留 `agentConfig`，缺失的字段仍缺失。缺口：用的是 `promptInactivity` 而不是 `completionSignal`，也没有 desktop 一侧的导入测试 | `Partial` |
| desktop 写的 `agent` 在 Cloud v2 上按 D2 执行并标为暂不生效 | [`TestAnalysisReportsUnappliedAgentSettings`](https://github.com/ora-space/cloud/blob/2b80c023/v2-server/internal/workflow/analysis_test.go) 覆盖“标为暂不生效”；导入的 `agent` 节点按 `end_turn` 完成没有运行测试，与上一个用例的执行义务同一缺口 | `Partial` |

- 决策依据：D3，不变量 4；desktop 侧显式 `end_turn` 的 legacy 执行沿用上文引用的完成信号用例。

## From The Restore Version Explicit Agent Policies Take Effect Without Rewriting Saved Values

- 风险：恢复版本把已保存的 `end_turn` 批量改回 `agent`，覆盖用户的选择或改写已发布快照；显式 `agent` 在恢复
  版本上仍按 `end_turn` 执行；能力不足时又悄悄降级为 `end_turn`；升级前已经启动、没有完成工具的会话被要求
  发出完成信号，永远无法普通完成。
- 前置状态：Controller 已升级到恢复版本（接入完成工具，锁定的 Node 声明 `session_host_mcp_servers`）；存在
  升级前保存的 `end_turn` 与 `agent` 节点、已发布的快照，以及升级前启动、仍在进行的节点会话。
- 触发：新建交互节点；启动冻结图里有 `agent` 节点的 WorkflowRun；目标 Node 不具备宿主能力，或 Agent 不支持
  HTTP MCP；升级前启动的会话继续轮次并完成。
- 必须成立：新交互节点写 `agent`；已保存的 `end_turn`、已发布的快照与冻结的运行不变；显式 `agent` 按完成信号
  ADR 执行，自动接受开始生效，“暂不生效”的标记消失；Node 不具备宿主能力时在启动准入时以有名字的原因拒绝，
  Agent 不支持 HTTP MCP 时以 `host_mcp_http_capability_missing` 失败；升级前启动的会话在结束前保持 `end_turn`。
- 禁止结果：任何已保存的值或快照被改写；`agent` 节点仍按 `end_turn` 执行；能力不足时按 `end_turn` 运行；
  升级前启动的会话被要求完成信号。
- 验证义务与证据（恢复版本尚不存在，待该版本验证）：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 新节点写 `agent`，已保存的值与快照不改写 | 编辑器测试与升级测试：升级前后比较已保存的图和快照 | `Missing` |
| 显式 `agent` 按完成信号 ADR 执行，自动接受生效 | Controller 运行测试：恢复版本下 `agent` 节点等待完成信号，开启自动接受的节点在信号后自动完成 | `Missing` |
| 能力不足时明确失败，不降级 | Controller 测试：Node 不声明能力时启动准入拒绝；Agent 不支持 HTTP MCP 时会话以 `host_mcp_http_capability_missing` 结束 | `Missing` |
| 升级前启动的会话保持 `end_turn` | 升级测试：升级前启动的交互会话在升级后由发起人按 `end_turn` 门禁完成 | `Missing` |

- 决策依据：D4，不变量 5、6；宿主 MCP ADR D4、D6。
