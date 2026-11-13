# agent 显式完成信号

本文件是[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md)的核心用例。实现已由 [ora-space/desktop#767](https://github.com/ora-space/desktop/pull/767) 落地，证据按直接测试标注。

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
| 无 token / 旧 token / 跨 session 被拒绝，同 attempt 幂等 | `crates/backend/src/workflow/run/interactive/completion_server.rs` 的 `an_unknown_token_is_a_tool_error_and_a_wrong_path_is_unauthorized`、`a_tool_call_accepts_the_attempt_token_and_binds_to_its_turn`；`agent_signal.rs` 的 `unknown_and_revoked_tokens_are_rejected_with_the_adr_reason`（stale_attempt / not_awaiting）；e2e `the_agent_signal_binds_to_the_turn_and_the_human_confirmation_completes_it` | `Covered` |
| park 只绑定结算 attempt，旧请求丢弃 | e2e `the_agent_signal_binds_to_the_turn_and_the_human_confirmation_completes_it`、`a_new_turn_without_a_signal_clears_the_previous_one` | `Covered` |

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
| `completionPolicy: agent` 且缺信号时阻塞、普通完成被拒 | 单测 `agent_policy_materializes_the_signal_gate`；e2e `the_missing_signal_first_turn_uses_the_documented_force_exception` | `Covered` |
| 非首轮 force 覆盖 `agent_signal_pending` 并写审计 | e2e `a_new_turn_without_a_signal_clears_the_previous_one`；单测 `forcing_a_stopped_turn_records_the_turn_window`、`forcing_an_unsettled_turn_records_the_last_assistant_message` | `Covered` |

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
| 确认必须匹配 `signalId` 与当前 turn | e2e `the_agent_signal_binds_to_the_turn_and_the_human_confirmation_completes_it`（stale id 返回 `WorkflowNodeSignalChanged`）；单测 `confirming_an_agent_signal_does_not_confirm_the_opening_turn` | `Covered` |
| 确认完成不写 `completion_forced` | e2e `the_agent_signal_binds_to_the_turn_and_the_human_confirmation_completes_it`（最终 payload 无强制审计） | `Covered` |

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

- 场景 A：主操作继续；次操作「仍然完成」走二次确认，并同时发送 `confirmInitialTurn: true` + `force: true`；完成时 `requester: human`，`completion_forced` 记录 `agent_signal_pending`、`prepare` 复算出的 blockers 与 `stop_cause`，排除由 confirm 解除的 `turn_unconfirmed`；
- 场景 A 只带 `force` 是合法请求形状，但返回 `WorkflowNodeIncomplete` 并列出未解除的 `turn_unconfirmed`，不推进、不写审计；`invalid_request` 只用于例外之外的组合请求（没有 `turn_unconfirmed`，或没有可被 force 覆盖的 blocker）；
- 场景 B：确认请求必须同时带 `confirmInitialTurn` 与 `confirmAgentCompletion`，只带一个被另一个 blocker 拒绝；
- 两种场景都不改变「首轮必须人工确认」的语义。

### 禁止结果

- 首轮无信号时只有继续、没有人工推进出口；只带 `force` 完成首轮；只带一个确认字段完成。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 单独 `force` 返回 `WorkflowNodeIncomplete`（含 `turn_unconfirmed`），不推进、不写审计 | e2e `the_missing_signal_first_turn_uses_the_documented_force_exception` | `Covered` |
| 场景 A 卡片次操作二次确认并同时发送 `confirmInitialTurn` + `force` | `packages/app-shell/src/features/workflow-run/run-node-session-chat.test.tsx` 的 `offers continue and the documented force exit for a first turn without a signal` | `Covered` |
| 组合请求审计：`requester: human`、`confirmation: null`、`stop_cause`，`blockers` 恰含 `agent_signal_pending` 与 `prepare` 复算的文件 blocker，排除 `turn_unconfirmed` | e2e `the_missing_signal_first_turn_uses_the_documented_force_exception` 覆盖 `requester`、`confirmation`、`agent_signal_pending` 与 `stop_cause`；`prepare` 复算的输出契约/文件 blocker 由 `forcing_a_structured_failure_keeps_raw_output_without_structured_output` 等 force 单测覆盖，但尚未在同一请求上组合断言 | `Partial` |
| 例外之外（没有 `turn_unconfirmed`，或没有可被 force 覆盖的 blocker）返回 `invalid_request` | 单测 `conflicting_completion_flags_are_rejected_before_claiming` | `Covered` |
| 首轮有信号时两个确认字段缺一不可 | e2e `the_agent_signal_binds_to_the_turn_and_the_human_confirmation_completes_it`；单测 `confirming_an_agent_signal_does_not_confirm_the_opening_turn` | `Covered` |

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
| 首轮用户停止的停靠分类与 blocker（`turn_stopped` + `user_stopped`，无 `turn_unconfirmed`） | e2e `a_stopped_opening_turn_waits_for_the_user`；单测 `forcing_a_stopped_turn_records_the_turn_window` | `Covered` |
| 卡片「继续」动作可用 | `run-node-session-chat.test.tsx` 的 `keeps a user-stopped turn awaiting input instead of reading as an abnormal stop` | `Covered` |
| 卡片 force 二次确认与发送 | 同一组件用例断言 `force-complete-node` 可用；`offers continue and the documented force exit for a first turn without a signal` 断言确认后发送 `force` + `confirmInitialTurn` | `Covered` |
| force 审计：`blockers` 含 `turn_stopped`、`stop_cause = user_stopped` | 单测 `forcing_a_stopped_turn_records_the_turn_window` | `Covered` |

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
| 自动接受只在结算后、非首轮、非异常时走同一门禁 | e2e `auto_accept_completes_a_settled_human_turn_without_the_card`；单测 `a_headless_auto_accept_records_auto_confirmation`、`a_manual_confirmation_on_an_auto_accept_node_is_recorded_as_human`；前端 `shows the abnormal exit instead of a confirmation for a precise stop with a signal` | `Covered` |

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
| 缺 `completionPolicy` 走 legacy；`agent` 缺信号仍阻塞 | 单测 `completion_meta_defaults_to_end_turn_and_round_trips_agent_fields`、`agent_policy_materializes_the_signal_gate`、`a_legacy_node_is_materialized_and_blocked_by_an_unsettled_turn`、`prepare_materializes_the_frozen_completion_policy`；e2e `a_legacy_end_turn_node_never_receives_the_completion_tool` | `Covered` |

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
- 只有交互式 agent-policy 节点签发 attempt token 并注入完成指令；`end_turn` 或非交互节点不注入完成工具，也不签发 token；
- 恢复时按节点绑定与冻结策略重新推导；绑定缺失或损坏时不注入；
- 节点离开 `Pending`/`Running` 后 token 失效，越权调用被拒绝。

### 禁止结果

- 空 `Explicit` 因为宿主工具被放宽；`end_turn` 或非交互节点获得完成工具或 token；撤销后旧 token 仍可用。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 宿主能力与插件 MCP 白名单隔离、撤销后 token 失效 | e2e `workflow_mcp_allowlists_survive_restore_rebuild_and_refresh`、`mcp_plugins_bound_to_agent_nodes_reach_only_the_frozen_allowlist`、`a_legacy_end_turn_node_never_receives_the_completion_tool`、`a_non_interactive_agent_policy_node_never_receives_the_completion_tool`、`a_restarted_backend_restores_the_completion_tool`；单测 `an_oversized_request_is_rejected_before_reading_it` | `Covered` |

### 决策依据

[agent 显式完成信号决策](../../../../decisions/desktop/core/workflow/20261110-agent-completion-signal.md) D1、D7；[工作流显式选择 MCP](../../../../decisions/desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) D1、D4。
