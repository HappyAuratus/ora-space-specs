# Effect 时间语义与升级

决策依据：[行审计时间与业务时间](../../../../decisions/desktop/core/effect/20260905-separate-audit-and-business-time.md)。
以下用例通过真实 SQLite repository 接口验证，不修改进程时钟或依赖等待。

## Historical publication uses the write transaction clock

- 风险：旧 Skill 回填时把历史时间传播到较新 Scope/Target，导致 CHECK 失败或审计倒退。
- 前置状态：Skill 版本时间 10、Scope 创建时间 100、Target 创建时间 200。
- 触发：在写时钟 300 发布已有 Skill，随后时钟回拨到 50，重复发布并退役 Source。
- 必须成立：Skill 版本仍为 10；新 Effect 行使用 300；既有行保留创建时间；重复发布不推进 generation；退役合法推进 generation，审计不倒退。
- 禁止结果：新建无意义 revision、回填事务失败、跨行使用 Scope 时间覆盖 Target 的下界。
- 验证义务与证据：**Covered** — `ora-db::effect_repository_tests::time::historical_publication_uses_the_write_transaction_clock` 直接比较版本、generation 和各行完整时间结果。

## Business phases and audit time survive clock rollback

- 风险：整轮 reconcile 复用开始时间，或回拨使阶段顺序违反约束，导致外部副作用已完成但数据库提交失败。
- 前置状态：已发布 Skill、有效 Target/Resource authority，使用真实 filesystem adapter。
- 触发：观察结束于 40，准备工作完成于 50，apply 完成于 200，验证期间回拨至 75；readiness 分别在 300 或 75 返回。
- 必须成立：prepared_at 为 40、applied_at 为 200、finalized_at 分别为 300 或 200；Operation 审计保留自身下界，Target 审计反映本行写时钟；提交后 request 完成。
- 禁止结果：所有阶段记录为批次开始时间，Operation 阶段倒退，或用 Operation 时间冒充 Target 行更新时间。
- 验证义务与证据：**Covered** — `ora-db::effect_repository_tests::time::business_phases_and_audit_time_survive_clock_rollback` 通过真实 mutation、readiness、finalization 与 cleanup 路径验证。
- 并发补充义务：**Covered** — `ora-db::effect_repository_tests::newer_wakeup_during_projection_commit_preserves_request_timestamp` 在 readiness 回调内发布更高 generation，验证新请求与较新时间得到保留。

## Scheduling and condition history use independent time

- 风险：审计时间被误用于到期判断，或重放 Condition 破坏 first/last observation 的顺序。
- 前置状态：行审计时间为 300，retry 的业务截止时间为 30。
- 触发：分别在业务时间 25、30 领取，随后在写时钟 300、200 重复阻塞相同 Condition；时钟回拨到 100 后修改并退役 Consumer。
- 必须成立：25 不可领取、30 可以领取；重复 Condition 的 identity、first/last observation 保持；旧 Target 退役后时间不倒退。
- 禁止结果：审计时钟提前解除 retry，或回拨阻止合法声明/退役。
- 验证义务与证据：**Covered** — `ora-db::effect_repository_tests::time_queue::scheduling_and_condition_history_use_independent_time`。

## Workspace and Effect initialization commit atomically

- 风险：移除 Scope trigger 后漏建 Scope，或 seed 失败留下半初始化 Workspace。
- 前置状态：存在可发布 Source，新增 Workspace 的业务时间为 10、写事务时钟为 300。
- 触发：分别通过 Project 和 TaskWorkspace repository 新建 Workspace；先注入 seed 失败，再解除失败重试。
- 必须成立：失败时业务行、Scope、Desired 均回滚，worktree lease 保留；重试成功后 Workspace 保留业务时间，Scope/Desired 使用 300 且 generation 为 1。
- 禁止结果：孤立 Workspace、遗留 Scope、丢失 lease、用 Workspace 历史时间初始化 Effect 审计。
- 验证义务与证据：**Covered** — `ora-db::effect_repository_tests::scope_initialization::workspace_and_effect_initialization_commit_atomically`。

## Migration round trips preserve recovery evidence

- 风险：新增 detected_at 或降级时丢失恢复检测时刻、ownership、artifact authority 或尚未完成的请求。
- 前置状态：有 RecoveryRequired Operation，其检测时间为 101、行审计为 200；存在 Managed Item、Artifact 和另一个仍 Claimed 的 Target。
- 触发：连续两次降级至 0009、升级至 0010 并重新读取。
- 必须成立：检测时间始终为 101；所有 Effect 表的业务内容保持，外键完整；恢复状态和检测字段关系受数据库校验保护。
- 禁止结果：将审计时间 200 当作检测时间，重建/删除业务身份，丢失 ownership 或 claimed request。
- 验证义务与证据：**Covered** — `ora-db::effect_repository_tests::time_migration::migration_round_trips_preserve_recovery_evidence` 比较全部 Effect 表业务列，并通过重新打开的 repository 还原 Operation。
