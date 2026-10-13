# while-Loop 的轮次隔离、退出与恢复

决策依据：[while-Loop ADR](../../../../decisions/desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md)。
源码路径相对于 desktop；表中简写测试路径均位于 `crates/application/src/workflow_run/engine/` 或
`crates/backend/src/workflow/run/`。Covered 只证明列出的义务，不代表全部 UI/真实故障已验收。

## Loop Scopes Must Reject Cross Boundary Edges and Variable Leaks

- **风险：** body 越界写入或旧轮产物泄漏。
- **前置状态：** schemaVersion 2 图包含跨边、嵌套、越界 selector 或坏配置。
- **触发：** 解析图及构造轮池。
- **必须成立：** root/body 为独立 DAG，归属显式；外层只能经 Loop outputs 读取 body，祖先值只读。
- **禁止结果：** 跨作用域边、UI parentage 充当执行归属或子输出泄漏。
- **验证义务与证据：** Covered：`loop_graph/tests.rs` 的 `rejects_cycles_in_each_scope`、
  `rejects_invalid_ownership_and_cross_scope_edges`、`enforces_binding_visibility`、
  `rejects_missing_and_unknown_config_fields`；`loop_round/tests.rs` 的
  `round_pools_isolate_child_outputs_and_inherited_writers`。
- **决策依据：** D1、D2。

## Feedback Must Read One Completed Snapshot Before Advancing

- **风险：** 逐项赋值影响交换变量，失败留下半组状态。
- **前置状态：** 多项 carried values、常量/selector 初值、缺值和类型错误。
- **触发：** 初始化或轮次完成。
- **必须成立：** 初值一次固定，feedback 同时读取同一完成快照，失败不修改原池。
- **禁止结果：** 一项 feedback 读取另一项刚写入的值，坏值仍推进。
- **验证义务与证据：** Covered：`loop_round/tests.rs` 的 `initializes_literals_and_references`、
  `applies_feedback_simultaneously_without_mutation`、`missing_feedback_leaves_completed_pool_unchanged`、
  `rejects_incompatible_initial_and_feedback_values`。
- **决策依据：** D2。

## An Explicit Success Limit Must Never Mask Execution or Value Errors

- **风险：** 上限成功吞掉执行/值错误，重复回报开启超限轮次。
- **前置状态：** 上限 1 或 2，until 假或末轮真，fail/succeed 及坏输出。
- **触发：** 末轮结算、重算或重复回报。
- **必须成立：** 条件优先；默认 fail；显式 succeed 只改变正常耗尽结果，导出末轮值并记录原因。
- **禁止结果：** 第 N+1 轮、坏输出成功或重复完成。
- **验证义务与证据：**

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 精确上限、条件和取值错误 | `loop_round/tests.rs`：`enforces_exact_round_limit`、`terminates_on_first_or_last_round`、`successful_limit_policy_preserves_value_validation` | Covered |
| 持久引擎、重复回报与错误隔离 | backend `loop_limit_tests.rs`：`loop_limit_policy_controls_only_exhaustion`、`loop_limit_success_recovers_and_ignores_duplicate_callbacks`、`loop_limit_success_does_not_mask_execution_or_output_errors` | Covered |
| 提交各窗口的真实进程 kill 恢复 | 尚无逐窗口演练；持久状态重算不能替代 | Missing |

- **决策依据：** D3、D4。

## Loop Exit Must Persist Cleanup Before Publishing Outputs

- **风险：** 早退后旧 body 仍写入，迟到 cleanup 复活已取消 run。
- **前置状态：** 被选中 loopExit、并行 body、pending cleanup、坏输出或整 run 取消。
- **触发：** 早退、恢复及迟到回报。
- **必须成立：** 先持久意图再清理，成功后发布 outputs；不依赖下一轮 feedback，不取消外层并行工作。
- **禁止结果：** 未选分支触发、清理失败却成功或旧回报复活终态。
- **验证义务与证据：** Covered：backend `loop_exit_tests.rs` 的
  `loop_exit_only_fires_on_the_taken_branch`、`loop_exit_requires_a_terminal_loop_member`、
  `loop_exit_publishes_outputs_without_feedback`、`loop_exit_rejects_unavailable_result`、
  `loop_exit_recovers_pending_cleanup_and_rejects_late_completion`、
  `loop_exit_cleanup_cannot_revive_a_cancelled_run`、`loop_exit_cleanup_failure_fails_closed`、
  `loop_exit_preserves_outer_parallel_work`。实际 bound session 接线为 Partial：backend `operations.rs` 的
  `loop_exit_recovery_finishes_bound_session_cleanup`；未覆盖不响应停止的真实外部 Agent。
- **决策依据：** D5。

## Explicit Resume Must Reopen Round One and Preserve the Loop Checkpoint

- **风险：** resume 复用失败轮池，成功完成抹掉 checkpoint。
- **前置状态：** 第二轮失败/取消，或外层兄弟失败。
- **触发：** 显式 resume 和 Loop 完成。
- **必须成立：** 原活动 scope 关闭，受影响 Loop 重开 round 1；完成保留进入前 checkpoint，外层失败不任意改写 Loop scope。
- **禁止结果：** 两个活动轮、失败轮 feedback 复用、checkpoint 丢失。
- **验证义务与证据：** Covered：backend `loop_resume_tests.rs` 的
  `resume_after_a_second_round_failure_restarts_the_loop_from_round_one`、
  `cancel_during_a_round_closes_the_scope_and_resume_opens_a_fresh_round_one`、
  `loop_completion_keeps_the_checkpoint_keys_on_the_loop_row`、
  `a_root_sibling_failure_leaves_the_loop_round_alone_and_the_run_resumable`。
- **决策依据：** D6。

## The Editor Must Preserve the Frozen Loop Configuration Through Execution

- **风险：** UI 保存改变循环含义，运行视图不能解释轮次和完成原因。
- **前置状态：** 用户配置 variables、until、outputs 与上限策略，包含旧平图。
- **触发：** 编辑、保存、真实 Desktop 执行及查看。
- **必须成立：** 冻结图与配置一致，旧平图可读，可区分条件、早退和显式上限成功。
- **禁止结果：** 纯解析测试被当成真实 UI/宿主整链验收。
- **验证义务与证据：** Partial：配置解析、旧平图和 backend 调度有直接测试；尚未登记真实 Desktop
  从编辑到执行、轮次视图的完整证据。
- **决策依据：** D1、D3 及兼容边界。
