# 交互式节点完成与停靠

本文件是[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md)的核心用例。契约改变 `completeWorkflowNode` 的公开语义、park 快照与交互式节点的停靠范围；实现随 `ora-space/desktop` PR #729（`Refs ora-space/circus#500`）落地，以下证据按该实现与 CI 结果标注。

e2e 覆盖的是「重启 → 停止 → 完成」路径，与记录 2 的「停止 → 重启」顺序不同，但落到同一判定（`turn_stopped` 不调度下一节点、继续后可正常完成），证据表按等价路径记录。

证据表里仍标 `Partial` / `Missing` 的项是已知缺口：真实 agent 的重启恢复证据与人类回合步骤边界注入证据由 #567 承接，其余非阻断改进（e2e 停止时机、act card 截断展示、文档措辞、格式）由 #565 承接。

另一条已知限制：完成按已存快照的 blockers 判定，`prepare` 复算新出现而快照未记录的 blocker 不会阻断本次完成（ADR 风险段已登记）。

## Only An EndTurn Turn May Complete A Parked Node

### 风险

停止生成、token 截断、拒答或无法界定终态的回合被当成完成，用半截产出推进到下一节点。

### 前置状态

- 一个交互式 Agent 节点已 park 在 `Pending`，最后一轮的 stop reason 为 `Cancelled` / `MaxTokens` / `MaxTurnRequests` / `Refusal` / 未知之一。

### 触发

用户点击完成；或在 `force` 下点击完成。

### 必须成立

- 无 `force` 时返回 `WorkflowNodeIncomplete`，blockers 对应 `turn_stopped` / `turn_truncated` / `turn_refused` / `turn_unknown`，不调度下一节点。
- `force` 二次确认后可以完成，payload 记录 `completion_forced` 与 blockers 快照；结构化失败时不写 `structured_output`。
- 只有 `EndTurn` 的正常路径与现有行为一致。

### 禁止结果

- 无 `force` 时用半截 assistant 文本完成；或 force 没有审计记录。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 五种非 `EndTurn` 终态各自阻塞完成 | `ora_backend::workflow::run::completion_snapshot::tests::non_end_turns_are_blocked_by_kind`；`workflow::run::interactive::completion_tests::blocked_turn_returns_every_blocker_with_its_detail` | `Covered` |
| force 只覆盖 blockers，并记录审计 | `workflow::run::interactive::completion_tests::forcing_a_stopped_turn_records_the_turn_window`、`forcing_an_unsettled_turn_records_the_last_assistant_message`；`workflow::run::operations::tests::a_forced_completion_records_its_audit_in_the_completed_payload` | `Covered` |
| `EndTurn` 正常路径不回归 | `workflow::run::completion_snapshot::tests::completed_turn_is_ready`；`workflow::run::interactive::completion_tests::an_existing_ready_snapshot_is_not_rewritten` | `Covered` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D1。

## Empty Output And Integrity Gaps Follow The Declared Contract

### 风险

把 `None` 契约的空输出误判为失败，或把 `Text` / `Structured` 的空输出当成可完成；历史完整性缺口被静默当成 ready。

### 前置状态

- `None` 契约节点只改文件、最终回复为空；
- `Text` 契约节点空输出；
- `Structured` 契约节点没有合法 JSON；
- `HistoryIntegrity` 非 `Complete`，可能仍有一条 `TurnEnded`。

### 触发

park 计算快照；完成。

### 必须成立

- `None` 空输出只记非阻塞 warning，仍可完成。
- `Text` 空输出产生 `output_missing` blocker；`Structured` 失败产生 `structured_invalid` blocker。
- `HistoryIntegrity` 非 `Complete` 判 `turn_unknown` 并加 `history_integrity_gap` warning；能找到 `TurnEnded` 时字段可填，但 blocker 不变，只能 `force`。

### 禁止结果

- `None` 空输出被阻塞；`Text` / `Structured` 失败被放行；完整性缺口自动 ready。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| `None` 空输出仍可完成，只显示 warning | `workflow::run::completion_snapshot::tests::none_contract_empty_output_is_a_warning` | `Covered` |
| `Text` / `Structured` 失败产生对应 blocker | `workflow::run::completion_snapshot::tests::structured_failure_is_a_blocker`；`Text` 的 `output_missing` 仅由快照映射代码路径覆盖，无单独用例 | `Partial` |
| 完整性缺口判 `turn_unknown` 且字段形状正确 | `workflow::run::completion_snapshot::tests::damaged_history_stays_unknown` | `Covered` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D1。

## A Park Commit Writes The Snapshot And Pending Status In One Transaction

### 风险

park 分两次写快照与状态，观察者看到「`Pending` 但没有快照」，或投影为了补救而在轮询里读历史。

### 前置状态

- 首轮自动回合或人类回合刚结束，节点仍为 `Running`。

### 触发

park 事务提交。

### 必须成立

- 同一个 `Immediate` guarded `UPDATE` 同时写 `payload.completion` 和 `Running → Pending`，只允许 `status = 'running'`。
- payload 用 JSON merge，保留 `error_detail` / `ai_diagnosis` / `file_changes` 等既有 key。
- guard 拒绝时什么都不写；新 run 不会出现 `Pending` 无快照。
- 提交后发布一次 run invalidation。
- 读取投影只解析快照，不读历史、不跑 schema、不做修复。

### 禁止结果

- 两次写之间出现 `Pending` 无快照；或整体覆盖既有 payload key。
- `get` 路径读历史或写 payload。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 首轮与人类回合 park 都走同一个单事务 | `workflow::run::transitions::tests::park_node_run_commits_snapshot_and_status`；e2e `a_stopped_step_blocks_completion_and_continues_after_a_restart` 断言首轮 `payload.completion` | `Covered` |
| guard 拒绝不产生孤儿快照 | `workflow::run::transitions::tests::rejected_park_writes_nothing` | `Covered` |
| 投影不读历史 | 代码走查：读取投影只解析 `payload.completion`；无专门回归测试 | `Partial` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D3、D4。

## Turn Identity And Window Text Come From History

### 风险

park 读取的「最后一轮」是上一轮的 `TurnEnded` 或旧 assistant 文本；没有收尾消息时被早先的进度说明掩盖。

### 前置状态

- 同一会话存在多轮历史；最后一轮可能没有 `TurnEnded`，或经历过 prompt 重试。

### 触发

park 与 `prepare` 读取历史。

### 必须成立

- turn key 为 `{userSeq, endedSeq}`：最后一条用户消息的 `HistoryLine.seq` 与它之后那条 `TurnEnded` 的 `seq`；不一致时 `complete` 返回 `WorkflowNodeTurnChanged`。
- 没有 `TurnEnded` 时不生成 turn key，判 `turn_unknown`，不读取上一轮输出。
- 最终文本取 `(userSeq, endedSeq]` 窗口内按 `seq` 的最后一条 assistant 消息，可能为空，不取最后一条非空。
- 重试时被放弃尝试的文本可能进入窗口，这是已登记的已知限制；后续独立 attempt boundary record 修复。

### 禁止结果

- 用上一轮的 `TurnEnded`/assistant 文本完成；或用最后一条非空文本掩盖没有收尾消息。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| turn key 从历史推导并在 complete 比对 | `workflow::run::interactive::completion_tests::a_changed_turn_wins_over_the_blockers_gate`、`forcing_a_changed_turn_still_reports_the_conflict`、`changed_output_text_reports_the_two_digests` | `Covered` |
| 空收尾消息不被旧文本掩盖 | `workflow::run::completion_snapshot::tests::final_text_is_the_last_assistant_message_even_when_empty`、`an_unsettled_turn_does_not_report_output_contract_problems` | `Covered` |
| 重试已知限制有直接测试与说明 | 无直接用例；按 ADR D2 登记为已知限制 | `Missing` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D2、D5。

## Human Turn Cleanup Waits And Repairs Provisional Snapshots

### 风险

流在 actor 写入 `TurnEnded` 前被 drop，cleanup 立即读历史会把实际 `EndTurn` 的一轮写成 `turn_unknown`，且之后没有修复路径。

### 前置状态

- 人类回合的流被 drop；actor 尚未收敛。

### 触发

cleanup；之后的 `prepare`。

### 必须成立

- cleanup 先有界等待 actor 收敛，再 park。
- 超时仍无 `TurnEnded` 时写 `provisional: true` 的 `turn_unknown`，快照保留 `userSeq`。
- 只有 `prepare` 在同一个 `userSeq` 之后看到 `TurnEnded` 时替换 provisional 快照；`get` 不做修复、不读历史。
- cleanup 幂等，不覆盖已提交的非 provisional 快照。

### 禁止结果

- 在 `get` 轮询里反复读历史；或 provisional 快照永远无法修复。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| cleanup 的有界等待与 provisional 写入 | `workflow::run::interactive::cleanup_tests::cleanup_polls_until_the_turn_ended_is_durable`、`cleanup_parks_provisionally_when_the_turn_never_ends` | `Covered` |
| `prepare` 按 `userSeq` 修复且 `get` 不读历史 | `workflow::run::interactive::completion_tests::a_provisional_snapshot_is_repaired_for_the_same_user_message`、`a_provisional_snapshot_of_another_turn_is_left_alone`；`get` 路径无专门测试 | `Partial` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D3、D4。

## Human Turns Carry A Step Boundary Without Rewriting The Prompt

### 风险

后端改写或吞掉用户消息，或边界块承诺运行时阻止后续节点；DAG 多后继与末节点的文案错误。

### 前置状态

- workflow 人类回合；当前节点有多个直接后继，或没有后继。

### 触发

发送人类回合 prompt。

### 必须成立

- 后端从冻结图渲染 `<workflow_step_boundary>` 并前置到 prompt。
- `record_prompt` 保留用户原文。
- 文案写直接后续节点（可能多个）或「下一步」；末节点不出现空的「下一步」。
- 不承诺运行时阻止后续节点；不写死 Desktop 按钮名。
- 首轮自动 handoff 保持不变。

### 禁止结果

- 拦截或改写用户消息；写死单个下一节点；末节点出现空下一步。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 人类回合 prompt 带步骤边界，`record_prompt` 保持原文 | `workflow::run::interactive::turns::tests::boundary_precedes_the_prompt_and_records_the_original_turn`、`boundary_preserves_an_existing_recorded_turn`、`boundary_does_not_scope_an_empty_prompt`；渲染文案仍由 prompt 测试覆盖 | `Covered` |
| 多后继 / 末节点文案正确 | prompt 测试覆盖多后继与不承诺运行时阻止；末节点空「下一步」由步骤卡片前端测试覆盖 | `Partial` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D6。

## Force Is Bounded By State And Claim And Audited

### 风险

`force` 越过已取消、已完成或正在完成的节点，重复提交终态；或把结构化失败静默写成成功。

### 前置状态

- 节点不在 `Pending`、run 不在 `Running`、completion claim 不持有，或快照 blockers 非空。

### 触发

带 `force` 的完成请求。

### 必须成立

- 状态/claim 不满足 → `workflow_node_not_awaiting_input`；`force` 不能越过。
- turn key 不一致 → `WorkflowNodeTurnChanged`；digest 不一致 → `WorkflowNodeOutputChanged`。
- 只有 blockers 可被 `force` 覆盖；`completion_forced` 记录 blockers 快照。
- 结构化失败时 force 不写 `structured_output`，raw output 仍保存。

### 禁止结果

- force 让非 `Pending` 节点完成；或 force 记录缺失。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 四级错误优先级 | `workflow::run::interactive::completion_tests::a_node_that_left_the_awaiting_state_is_rejected_before_its_snapshot`、`a_changed_turn_wins_over_the_blockers_gate`、`changed_output_text_reports_the_two_digests`、`blocked_turn_returns_every_blocker_with_its_detail` | `Covered` |
| force 审计与结构化失败行为 | `workflow::run::interactive::completion_tests::forcing_a_structured_failure_keeps_raw_output_without_structured_output`、`forcing_a_changed_turn_still_reports_the_conflict`、`forcing_a_rewritten_output_still_reports_the_conflict`；`workflow::run::operations::tests::a_forced_completion_records_its_audit_in_the_completed_payload` | `Covered` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D5。

## Restart Preserves Pending And Continue Uses Runtime Restore Or Rebuild

### 风险

重启后 `Pending` 丢失或被自动续算；「继续当前步骤」假设恢复成功；无法继续时静默丢回合。

### 前置状态

- 一个交互式节点停在 `Pending`，进程重启；
- 另一个 run 有 `Running` 节点。

### 触发

重启；点击「继续当前步骤」。

### 必须成立

- `Pending` 跨重启保留。
- 重启后 `session/load` 成功、失败→`rebuild_provider_session` 两条路径，「继续当前步骤」都能得到一轮回复（真实 agent 集成测试）。
- continue 的 prompt 失败时：界面明确报错，节点保持 `Pending`，「仍然完成」仍可用，不静默丢回合。
- 在途 `Running` 仍按 `interrupted_by_restart` 失败，不自动续算。
- 工作流层不预设「重新运行本步骤」的降级路径。

### 禁止结果

- 重启后自动继续或丢失 `Pending`；把 restore 失败当成功。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| `Pending` 跨重启保留 | e2e `a_stopped_step_blocks_completion_and_continues_after_a_restart`；`workflow::run::operations::tests::recovery_preserves_awaiting_nodes_and_prunes_orphaned_baselines` | `Covered` |
| `session/load` 成功与 rebuild 两条路径都能继续 | 同一 e2e 用例的 `restart_scenario(refuse_load)` 两个分支使用 fake plugin；真实 agent 集成证据缺失，由 #567 承接 | `Partial` |
| continue 失败保持 `Pending` 且仍可完成 | `workflow::run::interactive::cleanup_tests::a_failed_human_turn_restores_the_previous_snapshot_and_still_forces`；`workflow::run::operations::tests::failed_prompt_start_restores_the_awaiting_node_before_returning` | `Covered` |
| `Running` 仍按既有失败边界处理 | 现有 boot sweep 测试部分覆盖 | `Partial` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D8、不变量 16。

## Interactive Refusal And Unknown Turns Park Instead Of Failing

### 风险

契约把 `Refusal` / 未知列为 blocker，但首轮仍按 `SessionEndedWithoutStopReason` 或其他失败路径结束，导致 blocker 不可达；或为了可达而改变非交互节点的行为。

### 前置状态

- 交互式 Agent 节点首轮或人类回合以 `Refusal` / 未知 stop reason 结束；另有一个非交互节点同样结束。

### 触发

节点停靠或失败。

### 必须成立

- 交互式节点对所有 terminal stop reason 停靠并写对应 blocker。
- 非交互节点保持现有失败语义，不进入停靠路径。
- 首轮缺失 stop reason 仍由 `SessionEndedWithoutStopReason` 失败，不伪造 `turn_unknown` 停靠。

### 禁止结果

- 交互式 `turn_refused` / `turn_unknown` 不可达；或非交互节点被误改成停靠。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 交互式 Refusal/未知停靠 | 快照层由 `workflow::run::completion_snapshot::tests::non_end_turns_are_blocked_by_kind` 覆盖；首轮 executor 的停靠路径无直接用例 | `Partial` |
| 非交互节点行为不变 | 现有 workflow run 回归测试部分覆盖 | `Partial` |
| 首轮缺失 stop reason 仍失败 | 现有 `SessionEndedWithoutStopReason` 路径无新增用例 | `Partial` |

### 决策依据

[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D1。

## Region Interactive Behaviour Remains Out Of Scope

### 风险

把本契约套用到 Loop/Iteration 区域，误以为区域内 interactive 的停靠、完成与恢复已被覆盖。

### 前置状态

- foreach 或 while 区域内含 `interactive: true` Agent。

### 触发

图校验与区域运行。

### 必须成立

- foreach 继续在执行前拒绝 interactive 成员。
- while 的解析接受不构成本契约的整链保证；区域行为不变。
- 本契约只覆盖外层平图。

### 禁止结果

- 用外层平图的完成测试替代区域内的整链证据。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| foreach 拒绝 interactive 成员 | `ora_application::workflow_run::engine::iteration::tests::rejects_interactive_nodes_inside_the_region` | `Covered` |
| while 区域内的停靠/完成/恢复整链 | 现有[区域 HITL 用例](stateless-recovery-and-hitl.md#interactive-nodes-inside-a-region-park-and-resume-through-existing-paths)仍标记 Missing | `Missing` |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md) D5 末段；
[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)的 D5 继承行与 D7；
[while-Loop 决策](../../../../decisions/desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md) D1。
