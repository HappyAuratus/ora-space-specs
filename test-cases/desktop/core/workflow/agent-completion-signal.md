# agent 显式完成信号

本文件是[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md)的核心用例。实现尚未开始，证据状态按现状标为 `Missing` / `Partial`；实现落地后按直接测试更新。

## Agent Completion Signal Binds To The Signaled Attempt

### 风险

工具调用只靠 session 定位，旧轮次的迟到请求被当成新一轮的完成信号；或一个 attempt 的信号被复用到下一轮。

### 前置状态

- 交互式 Agent 节点，冻结策略 `completionSignal: agent`；
- 节点 `Running`，当前 prompt attempt 已获得 host 签发的 attempt token。

### 触发

agent 调用 `ora_complete_workflow_step({ attemptToken })`。

### 必须成立

- token 绑定发起时的 `(session, attemptId)`，无 token、旧 token、其他 session 的 token 都被拒绝；
- 接受的请求按 attempt 保存，park 只接受与结算 attempt 一致的请求；
- 同一 attempt 重复调用幂等，返回同一个 `signalId`；
- attempt 结束后到达的调用返回 `stale_attempt`，不改变任何状态。

### 禁止结果

- 不携带 token 也能记录信号；旧 attempt 的请求被算到当前 turn；跨 session 调用成功。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 无 token / 旧 token / 跨 session 被拒绝，同 attempt 幂等 | 无实现，暂无直接测试 | `Missing` |
| park 只绑定结算 attempt，旧请求丢弃 | 无实现，暂无直接测试 | `Missing` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D1、D2。

## A Missing Signal Keeps The Node Awaiting

### 风险

agent 用 `EndTurn` 结束一个提问回合，但没有调用完成工具；界面把提问当成完成。

### 前置状态

- 节点 `Pending`，快照 `completionPolicy: agent`；
- 当前 turn 正常结束、没有精确 stop blocker，但没有有效信号。

### 触发

读取投影，或发起普通完成。

### 必须成立

- 快照含 `agent_signal_pending`，`ready = false`；
- 普通完成返回 `WorkflowNodeIncomplete` 并列出该 blocker；
- 卡片显示「等待参与」，主操作是继续；
- 非首轮允许 force 覆盖该 blocker 并按 D6 写审计。

### 禁止结果

- 缺信号时投影成 ready 或按 legacy 处理；普通完成成功。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| `completionPolicy: agent` 且缺信号时阻塞、普通完成被拒 | 无实现，暂无直接测试 | `Missing` |
| 非首轮 force 覆盖 `agent_signal_pending` 并写审计 | 无实现，暂无直接测试 | `Missing` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D2、D3、D6。

## Confirmation Binds To The Signaled Turn

### 风险

用户看到的确认被应用到下一轮的信号上；或普通完成绕过人工确认。

### 前置状态

- 当前 turn 有有效信号，`agent_confirmation_pending` 存在；
- 前端投影已暴露 `signalId` 与 `turnKey`。

### 触发

发起普通完成，或带 `confirmAgentCompletion: { signalId }` 的完成。

### 必须成立

- 普通完成返回 `agent_confirmation_pending`；
- 只有携带匹配 `signalId` 且绑定当前 turn 的确认才被接受；
- `signalId` 或 turn 不匹配返回 `WorkflowNodeSignalChanged`，旧确认不落到新信号上；
- 确认后是普通完成，不写 `completion_forced`。

### 禁止结果

- 只有 bool 的确认被接受；旧 `signalId` 确认成功；确认写入强制审计。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 确认必须匹配 `signalId` 与当前 turn | 无实现，暂无直接测试 | `Missing` |
| 确认完成不写 `completion_forced` | 无实现，暂无直接测试 | `Missing` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D3、D6。

## First Turn Combines Initial Confirmation With The Signal Gate

### 风险

首轮无信号时没有人工出口；或有信号时用一个确认字段绕过另一个 gate。

### 前置状态

- 首轮正常结束，`turn_unconfirmed` 存在；
- 场景 A：没有 agent 信号（`agent_signal_pending`）；场景 B：有信号但未确认（`agent_confirmation_pending`）。

### 触发

发起完成请求。

### 必须成立

- 场景 A：主操作继续；同一请求带 `confirmInitialTurn: true` + `force: true` 时完成，`requester: human`，`completion_forced` 记录 `agent_signal_pending` 与 `stop_cause`；单独 `force` 仍返回 `invalid_request`；
- 场景 B：确认请求必须同时带 `confirmInitialTurn` 与 `confirmAgentCompletion`，只带一个被另一个 blocker 拒绝；
- 两种场景都不改变「首轮必须人工确认」的语义。

### 禁止结果

- 首轮无信号时只有继续、没有人工推进出口；只带 `force` 完成首轮；只带一个确认字段完成。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 首轮无信号时 `confirmInitialTurn` + `force` 可完成，单独 force 被拒 | 无实现，暂无直接测试 | `Missing` |
| 首轮有信号时两个确认字段缺一不可 | 无实现，暂无直接测试 | `Missing` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D3、D5；[首轮确认与异常停靠](first-turn-confirmation.md)。

## First Turn User Stop Keeps Continue Or Force

### 风险

首轮用户主动停止被信号门禁误判成正常首轮，或卡片隐藏了继承契约允许的 force。

### 前置状态

- 首轮用户主动停止：`turn_stopped` + `stop_cause = user_stopped`，`parkKind = awaiting_user`；
- 没有 `turn_unconfirmed`，也没有信号 blocker。

### 触发

读取投影，或发起完成。

### 必须成立

- 卡片提供「继续」与「仍然完成（force，二次确认）」；
- force 覆盖 `turn_stopped` 并写审计；
- 不要求 agent 信号，也不叠加 `agent_signal_pending` / `agent_confirmation_pending`。

### 禁止结果

- 首轮用户停止被显示成异常停靠；强制完成被信号门禁拦截。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 首轮用户停止不叠加信号 blocker，继续/force 可用 | 现有 e2e `a_stopped_opening_turn_waits_for_the_user` 覆盖 legacy 语义，未覆盖 agent 策略 | `Partial` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D3、D5；[首轮确认与异常停靠](first-turn-confirmation.md) D1、D5。

## Auto Accept Waits For Settlement And Uses The Same Gate

### 风险

自动接受在 turn 进行中直接完成，绕过结算、首轮确认或异常门禁。

### 前置状态

- 节点 `autoAcceptAgentCompletion: true`；
- 本轮已结算，非首轮，无 `turn_unconfirmed`，无异常 blocker。

### 触发

park 完成后由 host 触发内部完成请求。

### 必须成立

- 不在 `Running` 期间完成；
- 只在本轮正常结算后执行，首轮与异常停靠不自动接受；
- 走与人工确认相同的门禁，`confirmation.by = "auto"`；
- 失败只记日志并保持 `Pending`，不重试、不写 `completion_forced`。

### 禁止结果

- turn 中途完成；首轮或异常停靠自动完成；自动接受写强制审计。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 自动接受只在结算后、非首轮、非异常时走同一门禁 | 无实现，暂无直接测试 | `Missing` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D4。

## Legacy End Turn Nodes And Old Snapshots Stay Unchanged

### 风险

新策略的快照缺 `agentSignal` 时被误判为 legacy，或旧快照被误升级成新语义。

### 前置状态

- 旧快照没有 `completionPolicy`；或节点策略为 `end_turn`。

### 触发

读取投影，或 `prepare` 修复快照。

### 必须成立

- 缺 `completionPolicy` 的旧快照按 legacy 读取；
- `end_turn` 节点不注入完成工具，不写信号 blocker；
- `agent` 策略的新快照即使缺 `agentSignal` 也按 `agent_signal_pending` 处理；
- `prepare` 按节点冻结策略重新物化并写入 `completionPolicy`。

### 禁止结果

- 缺 `agentSignal` 一律回退 legacy；旧快照被追溯改成新语义。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 缺 `completionPolicy` 走 legacy；`agent` 缺信号仍阻塞 | 现有 legacy 快照测试覆盖旧行为，未覆盖 `completionPolicy` | `Partial` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D3、D7。

## Host Completion Capability Does Not Change The Plugin MCP Whitelist

### 风险

宿主完成工具被当作插件 MCP，破坏「空 `Explicit` 不注入任何插件 MCP」与「安装不等于授权」。

### 前置状态

- 节点 `mcps` 为空或全部禁用；
- 节点策略 `completionSignal: agent`，或 legacy `end_turn`。

### 触发

创建、恢复、刷新或撤销节点 Session。

### 必须成立

- 空 `Explicit` 仍不注入任何插件 MCP；
- host 完成工具是独立宿主能力，不进入 `SessionMcpSelection` 与 `Effective MCP Set`；
- `end_turn` 节点不注入完成工具；
- 恢复时按节点绑定与冻结策略重新推导；绑定缺失或损坏时不注入；
- 节点离开 `Pending`/`Running` 后 token 失效，越权调用被拒绝。

### 禁止结果

- 空 `Explicit` 因为宿主工具被放宽；`end_turn` 节点获得完成工具；撤销后旧 token 仍可用。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 宿主能力与插件 MCP 白名单隔离、撤销后 token 失效 | 现有 MCP 白名单测试覆盖 `Explicit`，未覆盖宿主能力 | `Partial` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D1、D7；[工作流显式选择 MCP](../../../../decisions/desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) D1、D4。
