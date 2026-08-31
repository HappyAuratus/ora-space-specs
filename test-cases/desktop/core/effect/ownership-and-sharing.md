# Effect ownership 与共享 Resource

## Unowned Observed Items Must Never Be Adopted or Mutated

### 风险

外部 Resource 中可能存在用户或其他工具创建的内容。名称、路径、Source、内容摘要或 fingerprint
与 Desired 相似，都不能证明 Ora 拥有该内容。如果 planner 或 adapter 从 observation 推断 ownership，
reconcile 就可能覆盖或删除用户数据。

### 前置状态

- filesystem Resource 的 observation 中存在一个 item。
- 该 item 没有与 Managed ledger 和 adapter ownership evidence 精确匹配的 durable authority。
- Desired projection 为空，或要求占用与该 item 冲突的 native identity。

### 触发

planner 合并完整 Resource contribution 并生成 Resource projection；如计划允许，reconciler 随后尝试
应用该 projection。

### 必须成立

- 未精确匹配 ownership authority 的 item 被分类为 Preserved State，而不是 Managed State。
- 不冲突的 Preserved item 保留在 observation 中，且不会产生 Create、Update、Replace 或 Delete
  mutation。
- 与 Desired native identity 冲突且无法共存的 Preserved item 产生 blocking Condition。
- marker 缺失、marker identity 不匹配、ledger 不匹配或 fingerprint drift 均不能被静默修复、覆盖或
  删除。
- 只有精确 Managed ledger，或对应未终结 Operation/Artifact 的有限 authority，能够授权 mutation
  或清理。

### 禁止结果

- 因名称、路径、Source、内容或 fingerprint 相同而自动接管 Observed item。
- 为满足 Desired 而覆盖冲突的用户内容。
- 将 marker 本身当作脱离数据库 ledger 的完整 ownership authority。
- 把 Preserved item 写入 Managed ledger，以便后续删除。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 没有 ownership evidence 的无冲突 Observed item 被保留且不产生 mutation | `ora_effect_skill::tests::unowned_observed_item_is_preserved_and_never_becomes_a_mutation` | `Covered` |
| Operation 的 mutation label 必须与 exact previous/planned state 一致 | `ora_effect::tests::operation_intent_rejects_a_mutation_label_that_disagrees_with_exact_states` | `Covered` |
| Preserved item 与 Desired native identity 冲突时形成 blocking Condition，且不产生 mutation | 尚无直接证据 | `Missing` |
| filesystem marker 与 Managed ledger identity 不匹配时不覆盖、不删除外部内容 | 尚无直接证据 | `Missing` |
| 已管理内容发生 fingerprint drift 时不被静默替换或删除 | 尚无直接证据 | `Missing` |
| adapter 的实际 apply 路径拒绝任何缺少精确 ledger 或 Operation authority 的 mutation | 现有 planner 测试证明不会规划 mutation，但没有直接跨过 filesystem adapter 边界 | `Partial` |
| observation 不会把 Preserved item 写入 Managed ledger | 现有纯 planner 测试未检查 repository 提交后的完整 ledger | `Partial` |

### 决策依据

- ADR 不变量 5：Desired、Managed、Observed 与 Preserved State 含义独立。
- ADR 不变量 6：只有精确 Managed ledger 或 durable Operation/Artifact authority 允许 mutation。
- ADR 被否决边界：不能从路径、内容相似性或 observation 推断 ownership。

## Shared Resource Contributions Must Agree on Materialization Contract

### 风险

多个 Target 可以向同一物理 Resource 贡献 Desired item。如果 declaration 中的 materialization
contract 在 binding 或 projection 中丢失，planner 可能用自己的默认值覆盖声明；如果不同 contributor
的 contract 被静默合并，同一个 writer 会按不一致的外部语义生成 mutation 和 recovery authority。

### 前置状态

- 两个 active 或 retiring Target 绑定同一个 Resource。
- 每个 Target projection 都携带完整 `ResourceRequirement`。
- 两个 requirement 声明不同的 materialization contract。

### 触发

持有 Resource claim 的 worker 合并全部 contributor requirement，并尝试生成唯一
`ResourceProjection`。

### 必须成立

- contract 从 Consumer Resource template 原样进入 Target binding 和 requirement。
- merge 在选择 materialization input 或 mutation 前比较全部 contributor contract。
- 不同 contract 产生 Resource-owned blocking Condition，并保留未来重新评估的 obligation。
- 没有完整一致 contract 的计划不得准备 Attempt、Operation 或 Artifact authority。

### 禁止结果

- planner 用内建默认 contract 覆盖 Target declaration。
- 只采用第一个或最后一个 contributor 的 contract。
- contract 冲突时仍生成 Resource projection 或 filesystem mutation。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 同一 Resource 的不同 contributor contract 会产生 `materialization_contract_conflict` blocking Condition | `ora_effect_skill::tests::shared_resource_rejects_incompatible_materialization_contracts` | `Covered` |
| Target projection 中的 requirement contract 与 binding 完全相同 | `ora_effect_skill::tests::retiring_target_projects_no_desired_contribution_but_keeps_cleanup_binding` | `Covered` |
| contract 冲突的完整 reconcile 不会持久化 Attempt、Operation 或 Artifact | 当前纯 planner 测试证明不会生成 ResourcePlan，尚无 repository 级直接证据 | `Partial` |

### 决策依据

- ADR Projection 与共享贡献：materialization contract 必须从 template 贯穿到 requirement。
- ADR 不变量 4：Resource projection 必须合并全部 active 或 retiring contributor。
- 未知或冲突的版本化契约必须被拒绝，不能猜测执行。
