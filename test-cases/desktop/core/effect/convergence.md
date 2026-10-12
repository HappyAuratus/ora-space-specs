# Effect 收敛

## Older Completion Preserves Newer Target Intent

- 风险：旧 snapshot 整行覆盖 Target Status，回退 desired_generation 或 status_version，并在仍有新工作时显示 Current。
- 前置状态：worker 持有有效 claim，已规划 generation N；readiness 返回前发布更高 generation M。
- 触发：旧 reconcile 提交无 mutation 的 projection，或完成已执行 mutation 的 finalization。
- 必须成立：desired 保留 M，observed/applied/ready 只推进到已有证据的 N，phase 为 Pending；status_version 不丢失并发更新；request 可再次领取，下一轮可收敛到 M。
- 禁止结果：仅保住 updated_at、用旧 snapshot 覆盖新 intent、错误报告 Current、丢失新 request，或因新 generation 拒绝已有外部副作用的正常收尾。
- 决策依据：[Effect 系统基础](../../../../decisions/desktop/core/effect/0-effect-system-foundation.md) 中的状态水位、claim 与 journal 收尾规则。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 无 mutation 的旧 generation 0 完成后，状态保留 desired=1、ready=0、Pending，且新 request 与审计时间保留 | `ora_db::effect_repository_tests::newer_wakeup_during_projection_commit_preserves_request_timestamp` | `Covered` |
| mutation 的旧 generation 1 完成后保留 desired=6、ready=1、Pending 与较新的状态版本；外部内容已写入且没有未完成 Operation | `ora_db::effect_repository_tests::stale_status::older_mutation_finalizes_and_newer_intent_converges_on_the_next_pass` | `Covered` |
| 保留的 request 可再次领取，下一轮全部水位到达 6、状态 Current、请求完成，外部内容保持正确 | 同上 | `Covered` |
| 同一状态和 Condition 重放不制造状态版本变化或破坏 observation 历史 | `ora_db::effect_repository_tests::time_queue::scheduling_and_condition_history_use_independent_time` | `Covered` |

## Replaying a Converged Generation Must Have No Side Effects

### 风险

Effect 是 level-triggered 系统，同一 obligation 可能因通知、周期扫描、进程重启或显式 retry 被重复
评估。如果 Current Target 的重放再次创建 Operation、改写 ownership 或推进没有证据支持的水位，
重复唤醒就会从安全的 no-op 变成外部副作用来源。

### 前置状态

- 一个 Workspace Scope 包含一条可用的 Skill Desired Effect。
- 一个 active Target 绑定一个 filesystem Resource，且 Consumer Revision 支持所需能力。
- Target 和 Resource 尚未收敛到当前 generation。
- Consumer 能返回与本次 Target projection 匹配的 readiness receipt。

### 触发

worker 首次 reconcile 完整 generation，随后在 Desired、Consumer Revision、Target declaration 和
外部 Resource 均未发生语义变化时再次请求并执行 reconcile。

### 必须成立

- 首次 reconcile 物化预期内容，并原子提交 projection、Managed ledger、Operation 终态和状态水位。
- Resource 的 applied generation 只能在完整 Resource projection 被验证后推进。
- Target 的 ready generation 只能在完整 Target projection 被应用且 readiness receipt 精确匹配后
  推进。
- 首次收敛结束后 Target 为 `Current`，其水位满足
  `ready <= applied <= observed <= desired`。
- 第二次 reconcile 返回 Current/no-op，不创建新的 Attempt 或 Operation，也不改变外部内容和
  ownership identity。

### 禁止结果

- 仅因 wakeup 或观察结果相同就跳过必要的首次验证。
- readiness receipt 未匹配 Target、generation、Consumer Revision 或 projection digest 时推进 ready。
- 对已经 Current 的相同 projection 再次执行 filesystem mutation。
- 无 mutation 路径为制造进度证据而创建空 Attempt 或 Operation journal。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 一次完整 reconcile 会物化 Skill、提交 Managed ledger、Finalize Operation 并令 Target 到达 Current | `ora_db::effect_repository_tests::reconciler_materializes_and_finalizes_one_complete_target_generation` | `Covered` |
| 相同 generation 重放返回 Current，且 Operation 数量不增加 | `ora_db::effect_repository_tests::reconciler_materializes_and_finalizes_one_complete_target_generation` | `Covered` |
| 非法 Target 水位组合不能被恢复为有效状态 | `ora_effect::tests::target_watermarks_reject_unproven_readiness` | `Covered` |
| readiness receipt 的 Target、generation、Consumer Revision 和 projection digest 任一不匹配时均不能推进 ready | 尚无直接证据 | `Missing` |
| 无 mutation 的 reconcile 路径不创建 Attempt 或 Operation | 当前重放测试证明已有物化内容的 no-op，但未直接覆盖首次即无 mutation 的路径 | `Partial` |
| 重放保持外部内容、完整状态水位和 ownership identity 均不变 | 当前测试断言 Operation 数量不增加，但未比较完整持久化状态和外部快照 | `Partial` |

### 决策依据

- ADR 不变量 10：水位只由完整 projection、observation、verification 和 receipt 证据推进。
- ADR 不变量 14：mutation 必须可精确验证、可安全重试或进入 RecoveryRequired。
- durable request 表示仍欠一次重新评估；wakeup payload 不是 correctness 来源。
- 无 mutation 路径不应创建 Attempt 或 Operation journal。

## Target Retirement Must Finish on a Fresh Epoch

### 风险

Target lifecycle 从 active 进入 retiring 会把该 Target 的 Desired 选择清空。如果退役事务不推进
Scope generation，退役 Target 的空 projection 会落在它与 active 期间完全相同的不可变身份
`(target_id, generation, consumer_revision_id)` 上，与已持久化的非空 projection 同身份不同内容，
持久化必然失败并无限重试；接替 Consumer 把退役 Target 作为共享 Resource contributor 一并投影时
命中同一冲突，新旧 Workspace 全部无法收敛，readiness gate 永久阻止发送（desktop issue #6）。

### 前置状态

- 一个 Workspace Scope 内，一个纯 Consumer 型插件（自身不发布 Skill Source）的 Target 已在
  当前 generation 收敛为 Current，并在该 generation 持久化非空 projection。
- Skill catalog 由另一个 Skill 插件提供，卸载 Consumer 型插件不改变 Desired State。

### 触发

卸载该插件（`retire_consumer`），随后在同一 worker pass 之前安装同能力的 Marketplace 构建
（`declare_consumer`），再由（可能重启后的）worker 领取全部到期 Target。

### 必须成立

- 退役事务推进受影响 Scope 的 generation，并唤醒 Scope 内全部 active 与 retiring Target。
- 退役 Target 的空 projection 在新纪元持久化成功，收敛后 Target 行删除；共享 Resource 与
  Managed ledger 只在被最后一个贡献者放弃时清理。
- 接替 Target 在同一共享 Resource 上收敛为 Current/ready，既有 Workspace 的发送 gate 解除。
- 新建 Worktree 使用同一接替 Consumer 仍独立收敛。
- 声明替换（同一 Consumer 的新 Revision）同样推进纪元：旧 Target 在新纪元完成退役，新 Target
  收敛。
- 退役 Target 遇到已删除的 Workspace 根目录时，若共享 Resource 上没有存活贡献者仍 desire
  内容，则完成退役并清理其 ledger。
- 同一缺失根目录上若仍有接替贡献者 desire 内容，两个 Target 以 blocking、retry 为 `manual` 的
  Condition `workspace_root_missing` 进入 Blocked，不自动重试，也不重建该根目录。目录恢复后的
  显式 retry 重新收敛：接替 Target ready，退役 Target 删除，技能内容写回。

### 禁止结果

- 退役或声明替换在未推进 generation 的事务中唤醒 Target。
- 以"digest 冲突"失败后无限重试，或通过强删 projection/绑定"修复"。
- 把双重绑定（retiring 与 active Target 共享一个 Resource）当成非法状态强拆。
- 对已删除 Workspace 根目录无限重试 `Resource adapter operation failed`，或在仍有 desire
  内容时重建用户删除的根目录。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 卸载纯 Consumer 插件并安装 Marketplace 构建后，旧 Target 完成退役、新 Target 在共享 Resource 上 Current/ready、新 Worktree 独立收敛、重启后无剩余请求 | `ora_db::effect_repository_tests::consumer_handover::retiring_local_consumer_handover_to_marketplace_consumer_converges` | `Covered` |
| 同一 Consumer 声明变化替换 Target 时，旧 Target 在新纪元完成退役，新 Target Current/ready | `ora_db::effect_repository_tests::consumer_handover::replacing_a_consumer_declaration_retires_the_old_target_at_a_new_epoch` | `Covered` |
| Workspace 根目录已删除且无存活接替贡献者时，退役 Target 完成退役并清理全部自有行 | `ora_db::effect_repository_tests::consumer_handover::retiring_target_on_a_deleted_worktree_finishes_retiring` | `Covered` |
| Workspace 根目录已删除且存在存活接替贡献者仍 desire 内容时，两个 Target 以 manual `workspace_root_missing` 进入 Blocked；目录恢复后的显式 retry 重新收敛，退役 Target 删除，技能内容写回 | `ora_db::effect_repository_tests::consumer_handover::deleted_worktree_with_a_surviving_successor_blocks_until_the_directory_returns` | `Covered` |
| 已收敛的接替 Target 在桌面端删除 Workspace 目录并 retry 后出现该 Condition；目录恢复并再次 retry 后重新 ready，技能文件回来 | `ora-desktop-tests` `core::effect::tests::uninstall_and_reinstall_from_another_source_hands_over_the_shared_skill_directory` | `Covered` |
| 退役 Source 重新发布（同 revision）恢复全部 active Scope 的 Desired intent | `ora_db::effect_repository_tests::skill_republish::reinstalling_the_same_plugin_revision_restores_desired_effects` | `Covered` |

### 决策依据

- [Target 生命周期转换推进 Scope 收敛纪元](../../../../decisions/desktop/core/effect/20261010-retirement-epoch.md)
  D1、D2、D3 与不变量 1–5。
- [Effect 系统基础](../../../../decisions/desktop/core/effect/0-effect-system-foundation.md)：
  退役 Target 以空贡献参与共享 Resource 合并，只有空 projection 收敛后才删除 Target。Blocked
  由显式 retry 恢复，retry 不修改 Desired。

## Absent Effect Consumers Must Not Block Send

### 风险

发送门禁若把“已安装 Agent 且 status 为空”一律当成技能仍在同步，则没有 declared Consumer 的身份
会永远等不到 Target。升级后所选插件 ID 与已注册 Consumer 不一致、以及卸载完成后的旧插件 ID，
都会永久阻止发送（desktop issue #6 的另一条触发路径）。反过来，已声明 Consumer 的新建 Workspace
在 Target 行出现之前放行，会让第一条消息抢在物化之前发出。

### 前置状态

- 一个 Workspace Scope 已经存在。
- 至少一个 Agent Consumer 已经 `declared`，并至少在另一个 Workspace 上有 Target。
- 另有一个从未声明 Effect 的 Agent identity。

### 触发

按 Workspace 加 Agent identity 读取 Effect status，并把结果交给发送门禁。对失败态调用显式 retry。

### 必须成立

- 没有 declared Consumer 时，`obligation` 为 `not_required`，status 为空，发送门禁放行。
- 已声明 Consumer 在尚无 active Target 的 Workspace 上，`obligation` 仍为 `required`，status 为空，
  发送门禁继续等待。
- 已有 active Target 时，`obligation` 为 `required`，并返回该 Target 的 status。
- 插件卸载完成后，旧 Agent identity 变为 `not_required`；接替身份在声明后为 `required`。
- blocking 且 retry 为 `manual` 的 Condition 使门禁进入失败态，并调用显式 retry；retry 只携带
  Target identity。

### 禁止结果

- 用超时把 required Target 放行。
- 把 `not_required` 显示成“正在同步项目技能”。
- 在 required 且尚无 Target 时放行发送。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 未声明 identity 为 `not_required` 且无 status；已声明 Consumer 有 Target 时为 `required`；已声明但新 Workspace 尚无 Target 时仍为 `required` | `ora_db::effect_repository_tests::consumer_handover::effect_obligation_follows_the_declared_consumer_not_only_an_active_target` | `Covered` |
| 卸载完成后的旧 Agent 为 `not_required`，从未注册的 identity 为 `not_required`，仍声明的接替 Agent 为 `required` | `ora-desktop-tests` `core::effect::tests::uninstall_and_reinstall_from_another_source_hands_over_the_shared_skill_directory` | `Covered` |
| `not_required` 不关发送门；`required` 且 status 为空继续等待 | `@ora/app-shell` `use-agent-effect-readiness.test.tsx`：“does not gate an installed agent whose backend obligation is not required”，“keeps gating an installed agent until its required Target exists” | `Covered` |
| manual Condition 使门禁失败，并调用 `retryTarget` | `@ora/app-shell` `use-agent-effect-readiness.test.tsx`：“retries a failed Target through the Effect retry operation” | `Covered` |

### 决策依据

- [Target 生命周期转换推进 Scope 收敛纪元](../../../../decisions/desktop/core/effect/20261010-retirement-epoch.md)
  D4 与不变量 6。
- [Effect 系统基础](../../../../decisions/desktop/core/effect/0-effect-system-foundation.md)：
  显式 retry 只唤醒 reconcile，不改变 Desired。
