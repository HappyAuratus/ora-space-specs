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
| Applied 但未 Finalized 的 Operation 与 Prepared Operation 一样进入 RecoveryRequired，且不重放 mutation | 尚无直接证据 | `Missing` |
| generation wakeup、lease expiry和显式普通 retry 均不能绕过 RecoveryRequired | 尚无直接证据 | `Missing` |
| recovery scan 保留所有 Artifact authority 和对应外部 staging/backup | 尚无直接证据 | `Missing` |
| Artifact cleanup 失败持久化为 `CleanupFailed`，且不能误报为已清理 | 尚无直接证据 | `Missing` |
| recovery 不根据 filesystem 路径、名称或相似内容猜测完成状态 | repository 隔离测试证明状态进入 RecoveryRequired，但未跨 filesystem adapter 验证“不猜测”行为 | `Partial` |

### 决策依据

- ADR 不变量 11：Prepared Attempt、Operation 和 Artifact authority 不因新 generation 改写。
- ADR 不变量 14：不明确的 mutation 必须进入 RecoveryRequired，不能猜测完成状态。
- Target claim 不授权共享 Resource mutation；恢复隔离必须同时保护相关 Target 与 Resource。
