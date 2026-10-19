# Effect journal 与恢复

## Unfinished Operations Must Enter RecoveryRequired Without Guessing

### 风险

进程可能在外部 mutation 前后任意位置崩溃。Prepared 或 Applied journal 只能证明计划和已记录的
进度，不能仅凭路径、文件内容或相似 fingerprint 判断副作用是否完成。如果普通 reconcile 猜测结果
或重新执行 mutation，可能覆盖外部状态、丢失备份，或者重复应用一次非幂等操作。

### 前置状态

- Target 已持久化 ReconcileAttempt 和至少一个未 Finalized EffectOperation。
- Operation 固化 exact previous/planned state、adapter payload 和 Resource projection digest。
- worker lease 已过期；或者进程重启后已经没有 worker 对该 Operation 持有有效 authority。
- 对应 Artifact 可能仍然存在。

### 触发

startup recovery scan 或普通 claim 前的 recovery scan 发现未终结 Operation。

### 必须成立

- recovery scan 原子地把相关 Attempt、Operation、TargetStatus 和 ResourceStatus 隔离为
  `RecoveryRequired`。
- Target 和 Resource 都得到 blocking/manual Condition。
- 相关 Target 不再被普通 claim 查询选中。
- 普通 generation wakeup、lease expiry 或普通 retry 不能绕过 RecoveryRequired。
- 未终结 Operation 和 Artifact authority 被保留，不创建第二个 mutation journal。
- 仍有有效 worker lease 的 Operation 不被误隔离。
- 后续恢复只能通过能够验证 exact adapter state 的显式恢复路径完成。

### 禁止结果

- 根据目标路径存在、名称相同或内容看似匹配而把 Operation 猜成成功。
- 对 Prepared 或 Applied Operation 直接重新执行 mutation。
- 隔离时删除 staging、backup 或 compensation Artifact。
- 只隔离 Target 而让共享 Resource 继续被其他 claim 修改。
- 普通 wakeup 把 RecoveryRequired 改回 Pending。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| lease 仍有效时，Prepared Operation 不被 recovery scan 隔离 | `ora_db::effect_repository_tests::unfinished_operation_is_quarantined_with_target_and_resource_conditions` | `Covered` |
| lease 过期后，未终结 Operation 同时隔离 Target 与 Resource，并记录 Conditions | `ora_db::effect_repository_tests::unfinished_operation_is_quarantined_with_target_and_resource_conditions` | `Covered` |
| 隔离后的 Target 被普通 claim 查询排除 | 现有测试主要检查持久化隔离结果，未找到直接的后续 claim 断言 | `Partial` |
| 恢复关闭最后一个 journal 后，Target 重新被普通 claim 调度并收敛 | `ora_db::effect_repository_tests::recovery::recovery_replays_a_proven_applied_leftover_and_resumes_scheduling` | `Covered` |
| Applied 但未 Finalized 的 Operation 与 Prepared Operation 一样进入 RecoveryRequired，且不重放 mutation | `ora_db::effect_repository_tests::recovery::recovery_replays_a_proven_applied_leftover_and_resumes_scheduling`、`e2e desktop::quarantined_target_recovery_restores_the_send_gate` | `Covered` |
| 显式恢复只能通过 exact adapter state 证明关闭，漂移状态拒绝恢复并保留隔离 | `ora_db::effect_repository_tests::recovery::recovery_refuses_drifted_state_and_keeps_the_quarantine` | `Covered` |
| 证明 exact expected 基线的 Prepared journal 以 FinalizedWithoutEffect 关闭且不授予 ownership，随后普通收敛继续 | `ora_db::effect_repository_tests::recovery::recovery_closes_a_prepared_journal_without_effect_and_reconverges` | `Covered` |
| generation wakeup、lease expiry和显式普通 retry 均不能绕过 RecoveryRequired | 尚无直接证据 | `Missing` |
| recovery scan 保留所有 Artifact authority 和对应外部 staging/backup | 尚无直接证据 | `Missing` |
| Artifact cleanup 失败持久化为 `CleanupFailed`，且不能误报为已清理 | 尚无直接证据 | `Missing` |
| recovery 不根据 filesystem 路径、名称或相似内容猜测完成状态 | `ora_db::effect_repository_tests::recovery::recovery_refuses_drifted_state_and_keeps_the_quarantine`：跨真实 filesystem adapter 证明漂移内容既不匹配 planned 也不匹配 expected 时拒绝恢复 | `Covered` |
| 隔离中的 Target 被退休（插件卸载/替换）时不破坏 status CHECK，恢复后以新 Scope epoch 完成退役收敛 | `ora_db::effect_repository_tests::recovery::recovery_of_a_retiring_target_advances_the_epoch_and_finishes_retirement` | `Covered` |
| 对健康 Target 调用恢复是幂等 no-op，不改变任何持久化状态 | `ora_db::effect_repository_tests::recovery::recovery_of_a_healthy_target_is_an_idempotent_no_op` | `Covered` |

### 决策依据

- ADR 不变量 11：Prepared Attempt、Operation 和 Artifact authority 不因新 generation 改写。
- ADR 不变量 14：不明确的 mutation 必须进入 RecoveryRequired，不能猜测完成状态。
- Target claim 不授权共享 Resource mutation；恢复隔离必须同时保护相关 Target 与 Resource。

## Explicit Recovery Must Prove Exact Adapter State Before Closing

### 风险

隔离态是安全闸门：Prepared/Applied journal 的外部副作用可能已发生、可能未发生，也可能被外部修改。
如果恢复命令凭路径存在、名称相同或内容相似就判定成功，会覆盖用户内容、伪造 ownership 或把
半应用状态当成已完成；如果不提供任何出口，被隔离的 Workspace 永久无法发送（circus#177）。

### 前置状态

- Target、Attempt、Operation、TargetStatus、ResourceStatus 均处于 `RecoveryRequired`，request
  为 `blocked` 并携带 blocking/manual Condition。
- Operation 固化 exact previous/planned state；外部状态可能等于 planned、等于 expected，或两者都不等于。
- 共享 Resource 上可能还有其他 Target 的被隔离 Operation。

### 触发

用户从发送门控的恢复入口调用受控命令 `recoverEffectTarget`。

### 必须成立

- 每个被隔离 Operation 必须由 Resource adapter 证明：exact planned → journal Finalized 且
  ledger 保留该行；exact expected → journal `FinalizedWithoutEffect` 且不授予任何 ownership。
- 证明前无创建副作用地重新校验持久化 plan 的 Workspace / Resource 路径边界与精确根目录身份；
  文件、链接、不可访问路径与重定向不能冒充 Missing，安全缺失的 Resource 可以证明 Missing。
- 证明两者都不是时拒绝恢复、保留隔离与 Conditions，不重放 mutation、不猜测、不伪造历史。
- 关闭最后一个 journal 的事务同时：终结 attempt、按证据推进 ledger、清除 Target 的
  manual Conditions、Target/Resource status 离开隔离（水位不动）、request 回 pending。
- 共享 Resource 上仍有其他被隔离 Operation 时，Resource 保持隔离并把证据重新指向仍存在的
  Operation。
- retiring Target 恢复后推进 Scope epoch，空投影落到新的不可变 identity，不与活跃期投影冲突。
- 恢复后普通 reconcile 按当前 Desired State 重新收敛并证明 readiness，水位只能由新证据推进。
- staging/backup Artifact 在 journal 关闭后转入 cleanup 流程，失败持久化为 `CleanupFailed`。
- 对没有隔离 journal 的 Target 调用恢复是幂等 no-op。
- 终结事务把各 Operation 的原 `detected_at`、identity 和关闭结果保存到恢复 audit，再清空
  Operation 上仅属于 RecoveryRequired 阶段的字段；audit 失败时整个关闭回滚。
- 前端通过 effects 数据 owner 发起恢复，成功后刷新实际观察到该 Target 的查询并重新推导 readiness；
  不刷新其他 Workspace / Agent，失败继续阻止发送。

### 禁止结果

- 凭路径、名称或相似内容猜成成功；对被隔离 Operation 直接重放 mutation。
- 以 FinalizedWithoutEffect 关闭却推进 ownership ledger。
- 删除或改写不可变投影历史、journal 意图；清空阶段检测字段却未在同一事务的 audit 中保留原检测证据。
- 恢复路径绕过 worker 的 fence/claim 语义与 Artifact cleanup 权限。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 证明 planned 的 Applied 遗留终结并恢复调度，收敛后 send gate 判据恢复 | `e2e desktop::quarantined_target_recovery_restores_the_send_gate` | `Covered` |
| 证明 expected 基线的 Prepared journal 无效果关闭且 ledger 不变，随后普通收敛继续 | `ora_db::effect_repository_tests::recovery::recovery_closes_a_prepared_journal_without_effect_and_reconverges` | `Covered` |
| 漂移状态拒绝恢复并保留隔离 | `ora_db::effect_repository_tests::recovery::recovery_refuses_drifted_state_and_keeps_the_quarantine` | `Covered` |
| retiring Target 的 epoch 推进与退役完成 | `ora_db::effect_repository_tests::recovery::recovery_of_a_retiring_target_advances_the_epoch_and_finishes_retirement` | `Covered` |
| 健康 Target 的幂等 no-op | `ora_db::effect_repository_tests::recovery::recovery_of_a_healthy_target_is_an_idempotent_no_op` | `Covered` |
| Workspace / Resource 根或中间目录变成文件时，两种 Missing 证明与恢复都被拒绝且隔离保留 | `ora_db::effect_repository_tests::recovery::paths::recovery_refuses_file_roots_without_closing_the_journal` | `Covered` |
| 安全缺失的 Resource 可证明 Missing，验证与恢复不创建目录 | `ora_db::effect_repository_tests::recovery::paths::recovery_proves_missing_resource_without_creating_directories` | `Covered` |
| 根目录或条目链接不能参与精确证明，悬空链接不能冒充 Missing | `ora_db::effect_repository_tests::recovery::paths::recovery_refuses_link_roots_and_dangling_item_links`（Unix）；Windows 链接路径尚无直接测试 | `Partial` |
| 两种关闭结果均保留原检测时间及不可变意图，Operation 的阶段检测字段清空 | 本表 planned / expected 恢复测试通过 `recovery::evidence::assert_preserved` 比较 SQLite journal 与 audit | `Covered` |
| 检测证据无法写入 audit 时，journal、调度和隔离一并回滚 | `ora_db::effect_repository_tests::recovery::evidence::recovery_rolls_back_when_detection_evidence_cannot_be_saved` | `Covered` |
| 恢复成功刷新真实 readiness 并解除门控，其他 Workspace 与 Agent 不被刷新 | `AgentEffectRecoveryBanner > refreshes real readiness after recovery and isolates other workspaces and agents` | `Covered` |
| 恢复失败时真实 readiness 继续阻止发送 | `AgentEffectRecoveryBanner > keeps the real send gate failed when recovery is refused` | `Covered` |

### 决策依据

- ADR「调度、claim 与公平性」：恢复只能通过能够验证 exact adapter state 的显式受控命令完成。
- ADR 不变量 14：不明确的 mutation 必须进入 RecoveryRequired，不能猜测完成状态。
- ADR 不变量 11：Prepared Attempt、Operation 和 Artifact authority 不因新 generation 改写。
- [行审计时间与业务时间](../../../../decisions/desktop/core/effect/20260905-separate-audit-and-business-time.md)：阶段检测字段与历史恢复 audit 分别承载当前状态和检测证据。
