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
