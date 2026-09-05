# Effect 时间戳约束问题

从 9c2f26444a37d148f3d01dd13b994b6e38ba068a 中 crates/db/src/repository/effect/declaration.rs 的改动记录：

Effect 的 SQLite schema 中有多种时间顺序约束，例如：

```sql
CHECK (updated_at >= requested_at)
CHECK (updated_at >= created_at)
CHECK (updated_at >= prepared_at)
```

这些约束本身是合理的，但写入方不能假设自己持有的时间戳仍然是最新时间。

## Bug 原因

Worker 会在一次 reconcile 开始时取得时间 `t0`，并在后续状态提交中继续使用它。reconcile 期间，另一个事务可能产生更新的请求，并把同一行写成：

```text
requested_at = t1
updated_at   = t1
t1 > t0
```

旧 reconcile 随后提交。如果它直接执行 `updated_at = t0`，该行就会变成：

```text
requested_at = t1
updated_at   = t0
```

这违反 `updated_at >= requested_at`。SQLite 会回滚整个提交事务，而此前已经提交的 claim 仍然存在。结果是 reconcile 没有完成，请求也可能继续停留在 `claimed` 状态。

增加 timeout 无法解决这个问题，因为失败不是处理速度慢，而是每次提交都会触发确定性的约束错误。

## 核心结论

墙上时钟只表示采样时间，不表示事务的因果顺序。一个较晚提交的事务完全可能携带较早采样的时间戳。跨事务或外部调用保存下来的 `now` 都应视为可能过期。

对不拥有基准字段的更新，应保留数据库中已经存在的较新时间：

```sql
updated_at = MAX(updated_at, ?candidate)
```

不能通过删除约束来规避错误；那只会把非法时间顺序写入数据库。generation、version 和 fencing token 应负责并发正确性，时间戳主要用于审计和调度。

## 后续审计

检查 Effect 中所有带时间顺序约束的表及其 `UPDATE` 语句，尤其是跨越 adapter、callback、文件系统操作或其他事务的路径。任何无条件 `SET updated_at = ?` 都需要确认调用方的时间戳不可能早于该行当前的基准时间或 `updated_at`；否则应使用数据库侧的单调更新，并增加一个“旧操作提交期间发生新写入”的确定性并发测试。


## 本轮改进

本问题已按 [时间语义 ADR](decisions/desktop/core/effect/20260905-separate-audit-and-business-time.md)
实施：Effect repository 在取得写事务后采样注入时钟，各行用自己的下界保护审计时间；Skill 历史时间
只保留版本语义，不再传播到 Effect 行。Target/Resource 领域状态不携带可写入的审计时间，读取时以
独立 metadata 返回。reconciler 在业务阶段重新采样，Operation 的恢复 detected_at 独立持久化。

迁移 0010 保留原有身份、ownership、request 和 journal，支持降级后再升级；Workspace Scope 初始化
改由 Project/TaskWorkspace repository 与业务行共同提交。独立 migration CLI 仍保留初始化日志之前
可用的系统时钟，Effect 生产默认使用 Ora 本地时钟。

回归证据见 [时间语义核心用例](test-cases/desktop/core/effect/time.md)，包括旧 Skill 回填、交错请求、
时钟回拨、Condition 重放、Scope 原子初始化，以及带 ownership/artifact/claim 的迁移往返。

验证完成：`task format` 和完整 `task test` 均通过，覆盖前端、Rust workspace、Tauri 与 E2E。
时间语义 ADR 已标记为 `implemented`。
