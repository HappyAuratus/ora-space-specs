# Effect 收敛

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
