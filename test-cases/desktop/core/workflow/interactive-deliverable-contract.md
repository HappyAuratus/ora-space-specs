# 交互式节点产出物声明

本文件是[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md)的核心用例。决策已 `implemented`：desktop 实现见 [desktop#765](https://github.com/ora-space/desktop/pull/765)，Cloud 发布门禁见 [cloud#169](https://github.com/ora-space/cloud/pull/169)（合并提交 [`e35b23c1`](https://github.com/ora-space/cloud/commit/e35b23c1aa18d3abc27ba13e6a65b204718fb732)）。下列证据固定到 desktop [`fa824d9d`](https://github.com/ora-space/desktop/commit/fa824d9def94a4c9ffa34b03d2031d71777367bf) 与 cloud [`0ca0a9a0`](https://github.com/ora-space/cloud/commit/0ca0a9a0e0ab61ddaec3407f091181cc2a99778b)；`Covered` 表示该义务有直接失败即说明义务不成立的代表性证据，`Partial` 如实保留覆盖缺口。

文件门禁与[首轮确认决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md)同时生效：首轮停靠的节点仍需 `confirmInitialTurn`，它只解除 `turn_unconfirmed`，不解除文件 blockers；`force` 只覆盖非确认类 blocker，不能替代首轮确认。异常停靠与最新 main 的跨 crate e2e 证据由[首轮确认用例](first-turn-confirmation.md#abnormal-parking-is-distinct-from-awaiting-user)维护，本文件不重复登记。

## A Declared File That Is Missing Or Unchanged Blocks Completion

### 风险

节点声明了应产出的文件，agent 没有写出或没有改动就被完成，下游节点拿到空结果；或者用户补齐文件后仍被陈旧判定挡住。另一种错误相反：基线只观测了声明文件，展示层却把未观测的普通文件伪造成新增，或无关大文件让可判定的声明失效。

### 前置状态

- 一个交互式 Agent 节点已 park 在 `Pending`，本轮 `EndTurn`、输出契约满足；节点可能停在自动首轮，也可能是人类回合；
- 冻结图里声明了 `completion.files`，含 `mustChange: false` 与 `mustChange: true` 的条目；节点启动时的 worktree 基线存在；
- 另有声明被 Git 忽略或大于内联缓存预算的文件，以及一个与声明无关的超大文件。

### 触发

用户点击完成；随后补齐或改动文件后再次点击完成；或在 `force` 下点击完成；首轮停靠时同时发送确认。

### 必须成立

- 没有任何匹配路径存在时返回 `WorkflowNodeIncomplete`，blocker `file_missing`，`params.path` 是原始声明；
- `mustChange: true` 而没有匹配路径在相对基线的变更集合内时，blocker `file_unchanged`，`params.path` 同样是原始声明；
- 文件 blockers 在 `prepare` 完成时按现场 workspace 判定，不写入 `payload.completion`；补齐文件后重试即可完成，不需要重新 park；
- 文件 blockers 与回合、输出契约 blockers 汇入同一列表，错误优先级仍是状态 → turn key → digest → blockers，文件 blockers 排在输出契约 blockers 之后；
- `force` 二次确认后可覆盖文件 blockers，并在 `completion_forced` 审计里记录；首轮停靠还需 `confirmInitialTurn`，确认不解除文件 blockers，`force` 不能替代确认；
- 无声明节点的完成结果不因本能力多出文件 blocker；自动节点即使图上残留声明，也不会改变自己的普通文件差异；
- 节点启动基线覆盖显式声明的 ignored 文件与大文件；`mustChange` 在 ignored 文件上仍可判定；
- 无关超大文件或仅声明基线不使已有声明变成缺失，不伪造完整 `file_changes`，也不削弱可用的 `mustChange`；
- 文本缓存与摘要表示同一份字节时不算内容变更。

### 禁止结果

- 文件缺失或未变更时无 `force` 完成；
- 把文件判定写进快照，使补齐文件后仍被陈旧 blocker 挡住；
- `force` 覆盖文件 blockers 而不留审计，或用 `force` 代替首轮确认；
- 因 unrelated 大文件、ignored 文件或缓存表示形式把存在且未变的文件判成缺失/变更，或反过来把未观测的普通文件展示成新增。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 缺失与未变更分别产生 `file_missing` / `file_unchanged`，params 带原始声明 | [`completion_files_block_missing_and_unchanged_then_allow_a_retry`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L192) 断言两种 kind 与 `reports/new.md` / `reports/keep.md` 原值；判定实现见 [`completion_files.rs`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/interactive/completion_files.rs#L36) | `Covered` |
| 补齐文件后重试可完成，快照不含文件 blockers | 同一用例先断言拒绝后 `payload.completion.blockers == []`，补 `new.md`、改 `keep.md` 后同一入口成功；实现见 [`completion.rs`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/interactive/completion.rs#L300) | `Covered` |
| `force` 覆盖文件 blockers 并审计 | [`forcing_a_completion_records_file_blockers_in_its_audit`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L290) 断言 `completion_forced.blockers` 含两个文件 blocker，快照仍为空 | `Covered` |
| 无声明节点的完成行为不变 | [`an_existing_ready_snapshot_is_not_rewritten`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/interactive/completion_tests.rs#L726) 用无 `completion` 的图直接走 `prepare_completion` 并成功；基础完成语义由[交互式节点完成用例](interactive-node-completion.md#only-an-endturn-turn-may-complete-a-parked-node)维护 | `Covered` |
| 首轮确认只解除 `turn_unconfirmed`，不解除文件 blockers；修复后再次确认成功且无 forced 审计 | [`confirming_the_opening_turn_still_requires_declared_files`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L562) 经真实 `complete_node`：`confirmInitialTurn` 仍返回 `file_missing` / `file_unchanged`，快照保留 `turn_unconfirmed` 而不固化文件 blockers；补齐/改动文件后同一确认成功，payload 无 `completion_forced` | `Covered` |
| `force` 不能替代首轮确认 | [`force_cannot_stand_in_for_confirmation`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/interactive/completion_tests.rs#L885) 断言仅有 `turn_unconfirmed` 时 force 仍返回 `WorkflowNodeIncomplete` | `Covered` |
| ignored 文件在启动基线与完成判定中被覆盖 | [`must_change_stays_enforced_for_a_declared_gitignored_file`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1880) 先拒绝未变更、改动后成功；[`an_existing_gitignored_deliverable_satisfies_its_existence_declaration`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1454) 证明存在性不被 ignore 隐藏；定向捕获见 [`worktree.rs`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L228) | `Covered` |
| 无关大文件与仅声明基线不伪造判定或 `file_changes`，不削弱 `mustChange` | [`an_unrelated_oversized_file_does_not_hide_an_existing_deliverable`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1431)、[`must_change_stays_enforced_when_an_unrelated_file_exceeds_the_snapshot_budget`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1838)、[`partial_baseline_does_not_report_files_as_added_after_the_budget_recovers`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1511)；仅声明基线不产出完整 diff 由 [`worktree.rs`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L382) 的范围检查保证，[`declared_only_baseline_produces_no_full_diff`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L814) 与 [`unscoped_version_1_baseline_keeps_no_full_diff`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L698) 直接验证 | `Covered` |
| 文本与摘要表示相同内容时不算变更 | [`text_and_digest_representations_compare_by_content`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L911)；相同文本合并不丢展示由 [`merge_declared_capture_preserves_identical_existing_text`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L944) 覆盖 | `Covered` |
| 自动节点忽略残留声明，保持普通完成差异 | [`automatic_node_ignores_stale_completion_for_its_file_changes`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/executor/completion_baseline_tests.rs#L48) 断言自动节点仍取完整基线且回调收到空差异 | `Covered` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D1、D2、D4、D6、D7；[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D1；首轮确认与 force 边界见[首轮确认决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D2。

## Paths Stay Inside The Workspace And Glob Semantics Stay Minimal

### 风险

声明路径逃出运行 workspace，读取或判定了不属于本次运行的文件；glob 语义在不同平台上得出不同结论。

### 前置状态

- 冻结图里的 `completion.files` 含绝对路径、`..`、空段、`.` 段、反斜杠、控制字符、平台保留名或盘符前缀，或含 `{}`、`[]`、取反、错误位置的 `**`；
- workspace 内有指向 workspace 外的符号链接、目录和 gitlink；文件名大小写与声明不同，或使用不同 Unicode 组合形式。

### 触发

加载或校验该图；完成该节点。

### 必须成立

- 绝对路径、`..`、空段/`.` 段、反斜杠、控制字符、平台保留名、盘符前缀、过深目录与超长段在图校验时被拒并给出可读原因，字面量部分经 `StrictRelativePath` 校验；
- 只支持 `*`（不跨 `/`）、`**`（完整段，跨零个或多个目录）、`?`（一个字符）；`[]`、`{}`、前导 `!` 与嵌在段内的 `**` 在图校验时被拒；
- 匹配大小写敏感，不做 Unicode 归一化；逃出 workspace 的符号链接不算匹配，目录与 gitlink 也不算普通文件；
- `files` 为 1–32 条、`path` 不超过 512 UTF-8 字节，越界被拒。

### 禁止结果

- 跟随符号链接读取 workspace 外的文件并据此判定存在；
- 把不支持的 glob 语法当字面量或静默忽略；
- 把目录或 gitlink 当作已存在的交付文件。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 非法路径与不支持的 glob 在图校验时被拒 | [`rejects_out_of_bounds_completion_paths_and_file_counts`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/application/src/workflow_run/engine/agent_config.rs#L624) 覆盖空路径、`..`、空段、反斜杠、`[]`/`{}`、前导 `!`、错误 `**`、保留名、盘符、深度与 512 字节边界；[`rejects_unsupported_completion_subfields`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/application/src/workflow_run/engine/agent_config.rs#L605) 覆盖未知子字段；底层解析见 [`glob.rs`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/utils/src/path/glob.rs#L43) 与 [`rejects_unsafe_paths_and_unsupported_syntax`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/utils/src/path/glob.rs#L185) | `Covered` |
| `*` / `**` / `?` 的匹配语义 | [`wildcards_keep_their_segment_boundaries`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/utils/src/path/glob.rs#L211) 与 [`leading_globstar_matches_zero_or_more_directories`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/utils/src/path/glob.rs#L231)；真实 workspace 的 [`completion_globs_match_files_in_the_run_workspace`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L472) 同时验证三种通配符 | `Covered` |
| 匹配大小写敏感且不做 Unicode 归一化 | [`matching_is_case_sensitive_without_unicode_normalization`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/utils/src/path/glob.rs#L241) 覆盖大小写与组合/分解形式；真实完成路径的 [`completion_globs_are_case_sensitive`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L493) 断言大小写不同目录得到 `file_missing` | `Covered` |
| 逃出 workspace 的符号链接不算匹配 | [`completion_files_do_not_follow_a_symlink_out_of_the_workspace`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L526)（`cfg(any(unix, windows))`，两平台都真实创建临时文件链接，创建失败直接失败不 skip）；[09a 的 Windows job](https://github.com/ora-space/desktop/actions/runs/43171209830/job/131024721990) 明确列出该测试 `ok`；捕获层 [`capture_worktree_snapshot_excludes_an_escaping_symlink`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L976) 仍是 Unix 用例 | `Covered`：Linux 实跑通过，09a 的 Windows CI 实际执行通过；测试定义与 fa 相同，fa 的新 CI 由主会话补齐，不冒充已跑 |
| 目录与 gitlink 不算文件 | [`completion_files_do_not_accept_a_gitlink_or_directory`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L712) 对 `vendor/module` 与 `reports` 都返回 `file_missing` | `Covered` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D1、D3，不变量 4。

## A Missing Baseline Degrades mustChange To A Visible Warning

### 风险

升级前节点或基线写入失败时，`mustChange` 无法判定：当成阻塞会让节点永远无法完成，静默放行又让作者以为已检查。旧缓存中确实无法证明的内容也可能被伪造成新增或误判为未变更。

### 前置状态

- 节点声明了 `mustChange: true` 的文件，但节点启动时的基线不存在、损坏，或旧缓存对声明文件的旧内容无法证明（例如二进制存成 `null`）；
- 基线可能在 park 之后才被损坏。

### 触发

节点 park；用户点击完成。

### 必须成立

- 存在性只来自当前 workspace 的观测，缺失时仍是 `file_missing`；基线只参与变更比较；
- 确实无法判定时，`mustChange` 不产生 blocker，记录 warning `file_check_unavailable`，不作为完成阻塞；可读的旧文本仍按内容正常比较；
- park 时已知的基线缺失/损坏随 park 快照与状态迁移同一次原子提交写入 `payload.completion.warnings`，投影可见；park 后才损坏或完成前才发现不可解析的，`prepare` 使用既有 guarded `Pending` 快照修复改善提前投影；该修复是 best-effort，失败只记录日志，不影响最终完成；
- 准备结果在存在文件 warning 或 force 审计时携带对应 payload；最终完成事务将其与完成状态一起提交，因此完成后的 warning 不依赖 `prepare` 修复是否成功；最终事务写入失败时按既有错误返回，不假报成功，文件 blockers 仍不进入快照；
- 旧基线无法证明的二进制内容按同一 warning 降级，展示差异不把未知内容伪造成新增；
- warning 在步骤卡片中本地化显示。

### 禁止结果

- 因基线缺失或损坏阻塞完成；
- 基线缺失时不留任何可见痕迹；
- 把旧基线中不可证明的二进制内容伪造成变更或伪造成未变更；
- 把完成后的 warning 可见性建立在 `prepare` 补写成功上，或最终事务写入失败仍报告完成成功。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 基线缺失时存在性仍判定、`mustChange` 不阻塞 | [`a_missing_baseline_still_checks_declared_file_existence`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L371) 在移除基线后仍返回 `file_missing`，补文件后可完成；[`a_missing_baseline_degrades_must_change_to_a_visible_warning`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L329) 断言 `mustChange` 不阻塞 | `Covered` |
| `file_check_unavailable` 在 park 快照里且投影可见 | [`missing_baseline_warning_is_visible_after_the_first_turn_park`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/executor/first_turn_park.rs#L108) 经生产自动首轮 park 入口 + 真实 SQLite/history/transition/`GetWorkflowRunHandler` 投影：物理基线不存在，warning 恰好一次，`Pending`、`turnOrigin=workflow`、`turn_unconfirmed` 保留；不是纯 `with_file_check_warning` 单测，也没有外部 ACP 进程。人类回合 park 由 [`a_missing_baseline_warns_in_the_park_snapshot`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/interactive/cleanup_tests.rs#L399) 与 [`a_corrupt_baseline_warns_in_the_park_snapshot`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/interactive/cleanup_tests.rs#L447) 覆盖 | `Covered` |
| 最终完成事务恢复早期丢失的 warning，并与 force 审计一起提交 | [`completion_commit_restores_a_file_warning_lost_after_prepare`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1661) 在 `prepare` 后把已存 warning 改成空数组模拟早期修复丢失，经生产 `complete_node_with_payload_patch` 后 warning 仍可见、无 `completion_forced`、blockers 为空；[`completion_commit_keeps_forced_audit_and_file_warning_together`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1751) 在同一故障下同时保留 warning 与 `completion_forced`。测试不是制造真实 SQLite I/O 错误，而是直接验证最终提交不依赖早期补写成功 | `Covered` |
| 旧二进制基线的未知内容降级为 warning，且不伪造变更 | [`a_legacy_binary_baseline_degrades_with_a_visible_warning`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1474) 断言完成成功且 warnings 为 `["file_check_unavailable"]`；[`legacy_binary_null_is_not_reported_as_an_addition`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/operations/tests.rs#L1565) 断言 `file_changes` 为空；展示层 [`legacy_null_binary_is_omitted_from_the_display_diff`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/backend/src/workflow/run/worktree.rs#L833) | `Covered` |
| warning 在步骤卡片中可见且本地化 | [`localizes the file baseline warning in the step card`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-run/run-node-session-chat.test.tsx#L818) 渲染中文提示；[`run-node-completion-translations.test.ts`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-run/run-node-completion-translations.test.ts#L268) 覆盖中英文案 | `Covered` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D4、D7。

## Unsupported Declarations And Version Mismatches Are Refused, Not Ignored

### 风险

拼写错误、未定义的 `criteria` / 清单字段或版本错配被静默忽略，节点看似声明了验收却不经检查就完成；带声明的图在 Cloud 上不经检查运行。

### 前置状态

- 图里出现 `completion.criteria`、`checklist` 或未知子字段；
- `agentConfig.schemaVersion: 3` 同时带 `completion`，或版本大于 `4`、小于 `3`、不是整数；
- 同一份带 `completion` 的图被发布到 Cloud，或经资产导入冻结。

### 触发

desktop 加载、保存、导入或启动该图；Cloud 发布或资产导入该图；inspector 编辑声明。

### 必须成立

- desktop 对 `criteria`、`checklist` 与未知子字段报图校验错误并给出可读原因；
- 声明 `completion` 的图必须是 `schemaVersion: 4`；`3 + completion` 与不在 3/4 范围的版本被拒；缺省版本的旧图仍按 legacy 解析，版本 3 且无声明行为不变；
- TypeScript 规范化显式处理 `completion` 与版本，非法原值保留到导入校验；`workflow-mock`、`workflow-transfer`、graph codec 对 3/4 两个版本往返一致；
- inspector 在添加第一条文件时写版本 4，删除到空时移除声明并回到版本 3；非法路径就地提示且不丢失已声明行；
- Cloud 发布与资产导入都拒绝任何非 null `completion`，以及所有出现的非 3 版本；问题点名节点与字段，不产生快照；缺省/null 版本的旧文档保持原行为；
- Cloud 门禁先于提供该字段的编辑器合并/上线，合并顺序可证明。

### 禁止结果

- 静默丢弃不支持的声明字段后照常完成；
- 接受 `3 + completion` 或未知的更高版本；
- 在规范化、导入或 transfer 中丢掉非法原值，使校验器看不到它；
- desktop 编辑器先于 Cloud 门禁开放 `completion`。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| `criteria` / `checklist` / 未知子字段被拒 | [`rejects_unsupported_completion_subfields`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/application/src/workflow_run/engine/agent_config.rs#L605)；TypeScript 侧 [`refuses unsupported fields instead of dropping them`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/workflow-mock/tests/agent-completion.test.ts#L55) 与 [`does not throw when a malformed completion is normalized for display`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/workflow-mock/tests/agent-completion.test.ts#L193) | `Covered` |
| 版本 3/4 的合法与非法组合按 D7 判定 | [`rejects_completion_without_schema_version_4`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/application/src/workflow_run/engine/agent_config.rs#L552) 与 [`rejects_unsupported_agent_config_versions`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/crates/application/src/workflow_run/engine/agent_config.rs#L582)；TypeScript 版本矩阵 [`validates the version 3/4 matrix on imported graphs`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/workflow-mock/tests/agent-completion.test.ts#L137)；导入拒绝 [`refuses $name instead of dropping it`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L294) | `Covered` |
| mock、transfer、codec 对新字段与版本往返一致 | [`preserves version 4 completion declarations across graph round trips`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/workflow-runtime/src/graph-codec.test.ts#L496)；[`exports and imports the version and every declared file`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L234)；[`normalizes completion explicitly and keeps the declared version`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/workflow-mock/tests/agent-completion.test.ts#L168) | `Covered` |
| inspector 保存声明与步骤卡片展示拒绝 | Inspector 的 [`adds paths and writes version 4 with the first declaration`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-editor/workflow-agent-completion-fields.test.tsx#L62)（写版本 4、路径与 `mustChange`）；步骤卡片 [`renders file blockers from a rejected completion in the step card`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-run/run-node-session-chat.test.tsx#L779) | `Covered` |
| 非法路径与畸形文件行在编辑器/导入链路中保留到校验或修正 | Inspector 的 [`shows a localized path error inline and clears it after keyboard correction`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-editor/workflow-agent-completion-fields.test.tsx#L90) 与 [`keeps a malformed file row visible and lets the author correct it`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-editor/workflow-agent-completion-fields.test.tsx#L156)；transfer 的 [`invalid completion declarations survive normalization`](https://github.com/ora-space/desktop/blob/fa824d9def94a4c9ffa34b03d2031d71777367bf/packages/app-shell/src/features/workflow-editor/workflow-transfer.test.ts#L180) 断言非法原值到达导入校验 | `Covered` |
| Cloud 拒绝非 null `completion` 与非 3 版本，点名节点/字段且不产生快照 | [`TestWorkflowPublishRefusesInteractiveCompletionDeclaration`](https://github.com/ora-space/cloud/blob/0ca0a9a0e0ab61ddaec3407f091181cc2a99778b/integration/workflow_admission_test.go#L223) 覆盖发布两种拒绝、资产导入同一拒绝、`workflow_snapshots` 为 0，以及版本 3 无声明照常发布；单元矩阵 [`TestValidateWorkflowGraph`](https://github.com/ora-space/cloud/blob/0ca0a9a0e0ab61ddaec3407f091181cc2a99778b/internal/core/workflow_graph_test.go#L125) 含 4、null、字符串、1、4+completion 组合；实现见 [`validateAgentSchemaVersion`](https://github.com/ora-space/cloud/blob/0ca0a9a0e0ab61ddaec3407f091181cc2a99778b/internal/core/workflow_graph.go#L520) 与字段清单 [`workflowUnsupportedAgentConfigFields`](https://github.com/ora-space/cloud/blob/0ca0a9a0e0ab61ddaec3407f091181cc2a99778b/internal/core/workflow_graph.go#L56) | `Covered` |
| Cloud 门禁先于编辑器发布上线 | [cloud#169](https://github.com/ora-space/cloud/pull/169) 的合并提交 [`e35b23c1`](https://github.com/ora-space/cloud/commit/e35b23c1aa18d3abc27ba13e6a65b204718fb732) 已把门禁合入 Cloud main，编辑器由后续 [desktop#765](https://github.com/ora-space/desktop/pull/765) 提供；按 PR 合并顺序证明，不登记部署记录 | `Covered` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D1、D5、D7、D8；Cloud 门禁义务见[发布与启动门禁用例](../../../cloud/workflow/publish-and-start-admission.md#publishing-refuses-an-unsupported-node-or-configuration-and-names-it)。

## 边界

- v1 只定义 `files`；`criteria`、judge、`checklist` 与运行时验收清单不在范围内，图校验直接拒绝，不登记其运行时证据。
- `agent` 自报完成不属于本决策；`Loop`/`Iteration` 区域内的 interactive 行为保持既有边界。
- 最新 main 的三类异常停靠跨 crate e2e 属于[首轮确认用例](first-turn-confirmation.md#abnormal-parking-is-distinct-from-awaiting-user)；本文件的文件门禁不改变异常停靠、首轮确认或 `force` 审计规则。
