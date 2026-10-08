# 工作流 prompt 静默策略

本文件守住持续等待、冻结配置、会话隔离与取消清理；依据
[静默策略决策](../../../../decisions/desktop/core/workflow/20261008-prompt-inactivity-policy.md)。

## A Silent Waiting Prompt Remains Cancellable And Session Local

**风险**：隐藏子代理无父会话更新时被取消重发；关闭静默计时又误关取消或让其他会话获得豁免。

**前置状态**：同一 provider connection 上，一个 prompt 使用 `wait`，另一个使用默认策略；两者无
有效进展更新。

**触发**：跨越全部默认窗口，再向等待 prompt 返回响应、取消或断开 provider。

**必须成立**：等待 prompt 不自动取消或重发，恢复输出后可完成；默认 prompt 仍沿原规则超时；手动
取消、owning stream 释放、工作流停止与宿主关闭均收敛并清理路由。

**禁止结果**：跨会话保活、静默触发等待节点重试、停止后仍有持有资源的等待操作。

| 验证义务                                                           | 代表性证据     | 状态    |
| ------------------------------------------------------------------ | -------------- | ------- |
| 真实 ACP routing 与 actor 在静默后仍可完成、取消并保持默认会话隔离 | `ora_agent_runtime::connection::prompt_policy_tests::{waiting_prompt_survives_silence_while_default_session_retries, waiting_prompt_remains_cancellable_and_reusable, child_updates_do_not_rearm_the_parent_prompt}`；暂停时钟推进两小时，经过生产 plugin transport、ACP peer、路由与 actor | Covered |
| owning stream、provider 与最后宿主/stream owner 释放等待操作 | `ora_agent_runtime::connection::prompt_policy_tests::{dropping_a_waiting_prompt_cancels_its_owner, provider_disconnect_ends_a_waiting_prompt, releasing_the_host_and_stream_ends_a_silent_waiting_actor}` | Covered |
| 工作流停止与真实失败重试正常收敛 | Desktop `core::session::tests::lifecycle::{workflow_cancellation_stops_the_live_node_session, confirmed_and_dropped_workflow_follow_ups_restore_awaiting_status}` 同时验证两种策略；`ora_backend::workflow::run::executor::inactivity_tests::wait_policy_preserves_real_session_failures_and_frozen_policy_across_retries` 使用实际 runtime 错误与 SQLite 验证预算耗尽 | Covered |

**决策依据**：D1、D3、D4。

诊断义务由 `ora_agent_runtime::connection::prompt_policy_tests::inactivity_diagnostics_record_the_window_tools_and_cancellation_result`
完整比较结构化 WARN 字段证明（`Covered`）：策略、窗口、尝试、静默时长、当前工具数量与取消结果可观察，
新诊断不包含 prompt 或工具参数。`unconfirmed_cancellation_fences_never_resend_the_prompt` 验证正常完成、
请求错误与缺失取消响应均不会触发新的 provider turn，保留响应栅栏规则（`Covered`）。

## Every Workflow Turn Uses The Frozen Inactivity Policy

**风险**：首轮可持续等待，交互追问、重试或复合轮次却恢复默认；草稿编辑改变活动运行策略。

**前置状态**：冻结节点配置为 `wait`，随后编辑草稿；存在首次、交互追问、重试或复合轮次的发送。

**触发**：通过生产工作流入口发送各次 prompt，并重新加载运行详情。

**必须成立**：每次 prompt 使用冻结策略；普通会话仍默认；配置解析失败不把交互节点留在 Running；
运行详情展示冻结值。旧图缺失或 null 使用默认且不会自动补写，显式配置经保存发布及导入导出保留。

**禁止结果**：读取当前草稿决定活动策略、只覆盖首轮、配置归一化重写所有旧图、非法策略静默降级。

| 验证义务                                         | 代表性证据     | 状态    |
| ------------------------------------------------ | -------------- | ------- |
| 冻结图解析、非法值与交互后续 turn 的策略 | `ora_application::workflow_run::engine::agent_config::tests::{prompt_inactivity_defaults_and_explicit_policies_survive_graph_parsing, rejects_invalid_prompt_inactivity_with_node_and_field_context}`；`ora_backend::workflow::run::interactive::inactivity_tests::{every_human_turn_uses_the_frozen_policy_after_a_draft_edit, invalid_frozen_policy_rejects_a_human_turn_without_changing_node_state}` | Covered |
| 首轮、追问和标题获取期间的 admission 确实传递等待策略 | Desktop `core::session::tests::lifecycle::workflow_first_and_follow_up_turns_keep_waiting_past_the_default_window` 通过外部 fake ACP 验证静默 50 秒后无重发且可取消；`ora_agent_runtime::connection::prompt_policy_tests::title_polling_admission_preserves_a_waiting_prompt_policy` 验证预抢标题请求的 turn 仍可静默两小时 | Covered |
| 重试与 Loop/Iteration 每轮保留冻结节点策略 | `ora_backend::workflow::run::executor::inactivity_tests::wait_policy_preserves_real_session_failures_and_frozen_policy_across_retries`；`ora_backend::workflow::run::mixed_scope_tests::loop_and_iteration_dispatch_each_round_with_the_committed_pool` 比较每轮生产 dispatch 的策略与变量池 | Covered |
| 编辑配置、旧图不改写、保存发布导入导出与运行展示 | app-shell `workflow-agent-inactivity-field.test.tsx`、`run-act-inspector.test.tsx`；`workflow-transfer.test.ts` 的 `exports and imports the frozen snapshot policy %s despite later draft edits`；`workflow-editor.test.tsx` 的 `previews imported plugin dependencies before creating and publishing`；workflow-runtime `graph-codec.test.ts` 的 `preserves the inactivity policy %s in saved and published graph snapshots` | Covered |

**决策依据**：D1、D2。
