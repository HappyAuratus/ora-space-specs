# Scheduled Task 计划与 occurrence

## Structured Schedules Must Reject Unbounded Or Sub-Hour Input

- 风险：接受 cron 或小于一小时的周期会绕过首版负载边界；无效日历时间会让不同运行时产生不一致触发。
- 前置状态：用户创建或编辑一个 local Scheduled Task。
- 触发：提交每 N 小时、每日时刻或每周日期及时刻之一，或提交 cron、零小时间隔、空星期集合和非法时刻。
- 必须成立：三种结构化形状完整往返；`interval >= 1`；小时、分钟和每周非空集合被校验；契约不包含 cron 字段。
- 禁止结果：静默修正无效值、存储 cron，或由 UI 与 backend 分别解释计划。
- 决策依据：根决策 D1 与不变量 3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| domain 接受三种结构化规则并拒绝零间隔、非法时刻和空星期集合 | `ora_domain::tests::validates_scheduled_task_definition` | `Covered` |
| weekly 的 tagged JSON 形状和 weekday 值稳定 | `ora_contracts::scheduled_task::tests::weekly_schedule_uses_the_public_tagged_shape` | `Covered` |
| Desktop 表单不产生 cron 且在提交前显示同一校验结果 | 尚无直接证据 | `Missing` |

## One Scheduled Time Must Admit At Most One Run

- 风险：重复 timer 投递、进程恢复或系统时间回拨为同一计划点创建多个有费用的会话。
- 前置状态：任务已启用，同一 `(taskId, scheduledAt)` 被投递一次或多次。
- 触发：scheduler 接纳 occurrence，随后重放同一身份。
- 必须成立：持久化唯一性以 `taskId + scheduledAt` 为准；首次接纳至多创建一个 Scheduled Run；重放返回既有事实。
- 禁止结果：把 revision 加入幂等身份、重放新建 run/session，或只用内存去重。
- 决策依据：根决策 D2 与不变量 2。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| occurrence repository 具有 `(task_id, scheduled_at)` 唯一约束 | `ora_db::repository::scheduled_run::tests::replay_returns_the_first_run_and_its_frozen_snapshot`、`missed_outcome_prevents_a_later_run_for_the_same_occurrence`（真实 SQLite 事务与冲突重放） | `Covered` |
| application admission 重放返回同一 run 且不二次启动目标 | `ora_application::scheduled_task::admission_tests::replay_returns_the_original_application_admission_after_definition_changes` 直接证明返回原 run；尚缺 runtime 不二次启动目标的直接断言 | `Partial` |
| Desktop 重启后重放仍不重复 | 尚缺关闭并重开数据库/进程后的调度重放测试；同一 pool 内重建 repository 不等同重启 | `Missing` |

## Late And Manual Occurrences Must Remain Distinct

- 风险：唤醒后的过期 timer 执行失去时效的任务，或“立即运行”伪装成原计划点绕过 missed 事实。
- 前置状态：存在一个到期计划点；实际观察可能在 10 分钟内、超过 10 分钟、Desktop 离线后恢复，或来自手动运行。
- 触发：scheduler 恢复并评估到期点，或用户点击“立即运行”。
- 必须成立：10 分钟内的迟到照常接纳；超过 10 分钟记 `late_trigger` missed 且不执行；离线区间以
  `app_not_running` 记录 missed 且不逐条补跑；manual 使用独立身份。
- 禁止结果：自动 catch-up、把 missed 写成 succeeded/failed run、或用 manual 覆盖原 occurrence。
- 决策依据：根决策 D4、不变量 4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 10 分钟边界两侧得到 run 与 `late_trigger` missed | `ora_application::scheduled_task::admission_tests::ten_minute_boundary_admits_but_the_next_millisecond_records_missed` | `Covered` |
| Desktop 离线恢复只记录有界 `app_not_running` missed 且不执行 | `ora_application::scheduled_task::offline_recovery_tests::offline_interval_records_only_its_latest_expired_point`、`a_point_inside_its_lateness_window_is_left_for_live_admission`；尚缺完整 runtime 恢复不执行过期点的直接测试 | `Partial` |
| manual occurrence 不占用计划 occurrence 的幂等身份 | 尚无公开 manual command/API/UI 与独立身份实现，不能用计划 occurrence 重放测试替代 | `Missing` |
| DST 跳过不补跑，回拨只接纳第一个 UTC 点，时区变化不改写已落盘 occurrence | 已有系统时区 calendar 求值代码，但尚无这些边界的直接测试 | `Missing` |

