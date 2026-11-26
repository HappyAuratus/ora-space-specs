# 工作流按 skill ID 解析 Skill 存储路径

决策依据：[Skill 按存储路径以会话清单交给 Agent](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`implemented`）D8，代表性证据固定于 [desktop#827](https://github.com/ora-space/desktop/pull/827) 合并提交
[`9f3ed63f`](https://github.com/ora-space/desktop/commit/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf)。

## Workflow Nodes Must Resolve Skills By ID At Dispatch

### 风险

工作流不再冻结 Skill 内容或路径。如果节点仍沿用创建运行时的路径，或升级前冻结的工作树相对 `packagePaths`，它会读到
已清理的副本或已删除的旧版本目录。如果 Skill 在运行中被卸载，节点退回到同名的其他 Skill，就会静默执行错误的说明。
如果编辑器传入的裸 Skill 名在 Catalog 里匹配多个来源却静默选一个，节点会默默用到错误的 Skill；损坏的冻结 payload
如果静默回退重解析，也会绕过运行冻结承诺。启动检查如果仍要求 Agent 声明 Skill 交付能力，所有 Agent 都会被误拒。

### 前置状态

- 一个工作流有两个顺序执行的 Agent 节点，各启用一个 Skill：节点 1 用本地 Skill，节点 2 用插件 Skill。
- 另有一个与插件 Skill 同名的本地 Skill。
- 另有一个升级前创建、冻结记录中带工作树相对 `packagePaths` 且无 `catalog_skill_id` 的运行，以及损坏的冻结 payload 样例。
- 执行节点的 Agent 从未声明 Skill 交付能力，且未声明已接入原生 Skill。

### 触发

1. `start` 运行；节点 1 完成后，把插件升级到新版本，再派发节点 2。
2. 再次 `start`；节点 1 完成后卸载节点 2 用的插件，再派发节点 2。
3. 在缺少某个已启用 Skill、裸名匹配多个 Catalog ID、或冻结 payload 损坏时，执行创建运行、`start`、`restart` 或切换快照的 `resume`。
4. 恢复升级前创建的运行，派发它的下一个节点。

### 必须成立

- 运行冻结的只有 `{ node_id, skill_id, invocation_name }`（带精确 `catalog_skill_id`）。
- 裸 Skill 名匹配多个活跃 Catalog ID（保守计入不可用候选）时，创建运行、启动与切换快照均以公共错误 `workflow_skill_ambiguous { nodeId, nodeTitle, skillRef, candidates }` 拒绝，提示指定具体 Skill；显式 catalog ID 照常解析。
- 启动、重启与恢复前的检查是：精确 `catalog_skill_id`（及历史无该字段的裸引用）能在 Catalog 中解析为 Available Skill，且其存储路径下的 `SKILL.md` 存在并可解析；不满足时返回 `workflow_skill_not_found`，带节点标题，运行保持原状态。损坏的冻结 payload 显式失败，不静默回退到未冻结解析。
- 每个节点派发时按冻结记录重新解析当前存储路径；`<required_skills>` 块呈现调用名和剥离 Windows `\\?\` 前缀后的原生 `SKILL.md` 绝对路径，说明要求读取并遵循它；仅当 Agent 插件声明已接入原生 Skill 时才在块前附加前导 `/调用名`（当前无插件声明，不附加前导 `/调用名`）。
- 插件升级后，节点 2 读到新版本目录下的 `SKILL.md`。
- 插件在运行中被卸载后，节点 2 记录 `MissingRequiredSkill` 节点失败（兼容既有记录的序列化值 `missing_skill_materialization`，不可续跑）；已冻结的精确 ID（`plugin:` 前缀或已删除 ID）绝不回退到同名 Skill。
- 升级前的运行照常读取冻结记录，忽略其中的工作树相对 `packagePaths`，按既有 skill ID 解析。
- 未声明 Skill 交付能力的 Agent 照常执行带 Skill 的节点。
- 同一运行中两个不同 Skill 解析出相同归一化调用名时，启动被拒绝；同一 Skill 在多处复用允许通过。

### 禁止结果

- 冻结 Skill 内容、版本或路径，或按 revision 保留存储目录。
- 裸名歧义时静默任选一个候选，或已冻结精确 ID 缺失后回退到同名 Skill。
- 损坏的冻结 payload 静默回退到未冻结图重解析。
- 未声明接入原生 Skill 的插件收到无意义的前导 `/调用名`。
- 因 Agent 没有声明 Skill 交付能力而拒绝启动。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 冻结记录只含节点 ID、skill ID 与调用名（不写 workspace） | [`freezes_catalog_bindings_for_local_and_plugin_skills_without_writing_workspace`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prerequisites/tests.rs#L74)、[`initialize_workspace_records_child_loop_skills_without_copying_packages`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prerequisites/tests.rs#L223)、[`switched_receipt_freezes_catalog_skill_bindings_without_workspace_paths`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L551) | `Covered` |
| 启动、重启、切换快照恢复按“Catalog 可解析且 `SKILL.md` 存在”检查，失败时用 `workflow_skill_not_found` | [`rejects_missing_deleted_or_unusable_catalog_skills`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prerequisites/tests.rs#L167)、[`a_removed_skill_plugin_blocks_start_and_restart`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L231)、[`a_removed_skill_plugin_blocks_a_pending_start`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L268)、[`a_removed_skill_plugin_blocks_a_resume`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L313)、[`target_dependencies_are_checked_before_checkpoint_rollback`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L483) | `Covered` |
| 派发时按冻结 catalog ID 解析，运行中升级后后续节点读到新路径 | [`required_skills_resolve_live_storage_paths_by_frozen_catalog_id`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/executor.rs#L930) 验证按冻结 ID 解析本地 Skill 及文件缺失时失败；[`skill_plugins_bound_to_agent_nodes_reach_the_prompt`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/e2e/tests/desktop/core/session/tests/workflow_scenarios.rs#L1567) 验证固定版本的插件路径进入 prompt；尚无直接证据覆盖运行中切换插件 `package_root` 后下一节点使用新路径 | `Partial` |
| 运行中卸载后节点以 `MissingRequiredSkill` 失败，绝不回退到同名 Skill | [`uninstalled_plugin_skill_never_falls_back_to_same_named_local_skill_at_dispatch`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/executor.rs#L997)、[`uninstalled_plugin_or_deleted_id_never_falls_back_to_same_named_skill`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prerequisites/tests.rs#L266)、[`removed_plugin_skill_does_not_fall_back_to_same_named_local_skill`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L591) | `Covered` |
| 升级前运行的工作树相对 `packagePaths` 被忽略，损坏的冻结 payload 显式失败 | [`workflow_run_payload_ignores_legacy_package_paths`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/application/src/workflow_run/engine/skill_delivery.rs#L229)、[`corrupt_frozen_payload_rejects_start_and_same_snapshot_resume`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L991) | `Covered` |
| 同名裸 Skill 名匹配多个 Catalog ID（含不可用候选）时，以 `workflow_skill_ambiguous` 拒绝并附带中英文候选提示 | [`rejects_ambiguous_bare_skill_name_while_allowing_exact_catalog_ids`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prerequisites/tests.rs#L446)、[`ambiguous_bare_skill_name_blocks_create_start_and_snapshot_switch`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/operations/precheck_tests.rs#L840)、[中英文错误文案与分隔符测试](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/packages/app-shell/src/i18n/contract-error.test.ts#L102)、[en-US 候选分隔测试](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/packages/app-shell/src/i18n/contract-error.test.ts#L163) | `Covered` |
| 不再要求 Agent 声明 Skill 交付能力；调用名冲突仍被拒绝（同一 Skill 复用允许）；未接入原生 Skill 时不加前导 `/调用名` | [`rejects_conflicting_invocation_names_across_distinct_skills_and_allows_reuse`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prerequisites/tests.rs#L362)、[`skill_plugins_bound_to_agent_nodes_reach_the_prompt`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/e2e/tests/desktop/core/session/tests/workflow_scenarios.rs#L1567)、[`assembly_keeps_skill_invocations_first_and_requires_every_skill`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/workflow/run/prompt.rs#L709) | `Covered` |

### 决策依据

- Skill 会话清单决策 D8；派发预检中的 Skill 依赖检查由此修改。
- 不变量 10。
