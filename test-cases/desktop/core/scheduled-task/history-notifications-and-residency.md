# Scheduled Task 历史、通知与数据驻留

## History Must Preserve Durable Outcomes Across Disable And Delete

- 风险：UI 从当前时间推断状态，或禁用/删除 definition 时级联删除审计事实。
- 前置状态：task 已产生 run、skip 与 missed 历史。
- 触发：禁用、软删除或重启 Desktop 后重新读取列表。
- 必须成立：运行状态、failure reason、missed reason、revision、occurrence 和 session/worktree 引用来自持久事实；定义删除保留墓碑与历史；
  每个 task 只保留最新 100 条记录并从最旧删除；仍带保留 worktree 链接的运行例外，继续保留（数量受根决策 D4 的
  worktree 保留上限约束），链接清除后（用户丢弃、或清理路径确认任务已删除）该运行恢复可裁剪。
- 禁止结果：硬删除历史、按“现在已过 nextRunAt”临时生成 missed，或清理最新 100 条内的记录。
- 决策依据：根决策 D7 与不变量 3、8；保留 worktree 链接例外经 owner 结论（方案 A）确认。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| definition CRUD 和软删除从列表隐藏但保留数据库行语义 | `ora_db::repository::scheduled_task::tests::definitions_round_trip_update_and_soft_delete` | `Partial` |
| run/missed 事实在禁用、删除和重启后保持 | `ora_db::repository::scheduled_run::tests::history_reads_newest_first_after_definition_delete_and_repository_restart` 证明软删除后重新构造 repository 可读取原 run/missed；测试复用同一 pool，未证明真实数据库重开或禁用路径 | `Partial` |
| 第 101 条历史写入后只清理最旧记录，保留最新 100 条 | `ora_db::repository::scheduled_run::tests::one_hundred_and_first_history_entry_prunes_only_the_oldest_fact` | `Covered` |
| 仍带保留 worktree 链接的运行不被裁剪；链接清除后恢复可裁剪 | `ora_db::repository::scheduled_run::tests::history_pruning_keeps_runs_that_still_reference_a_retained_worktree`、`ora_backend::scheduled_task::worktree::tests::retention_sweep_unlinks_a_run_whose_worktree_task_was_deleted` | `Partial` |

## Notifications Must Reflect Persisted Run Facts

- 风险：通知先于事务提交导致点击后没有历史，或漏报 failure/timeout/missed。
- 前置状态：run 开始或终止，或 scheduler 持久化 missed。
- 触发：对应事实成功提交。
- 必须成立：run start、failed、timeout、missed 各发一条桌面通知；导航入口的落地形态是应用内
  toast——开始通知的入口进入新会话，终态通知的入口定位对应历史；窗口不可见时发送系统级通知
  （点击仅聚焦窗口，桌面系统通知插件不提供点击回调），且该事实的导航入口在窗口重新可见后仍以
  应用内 toast 呈现，不因发通知时窗口不可见而丢失；通知失败不改写运行事实。
- 升级采纳例外：安装后首次运行（无已初始化的 seen 标记）把既有历史一次性标记为已见并把基线推进到
  最新事实，不逐条补报；代价是升级前离线期间的失败/`missed` 不再提示。这是对「补报离线事实」的
  有意例外；该标记只建立一次，其后产生的事实照常通知，事实本身仍完整保留在历史中。
- 禁止结果：通知充当状态源、事务回滚后仍发送成功通知，或同一事实重复通知。
- 决策依据：根决策 D7；系统通知与 toast 的分工、隐藏窗口期间保留导航入口的要求，以及升级采纳例外，均经 owner 结论确认并显式登记。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| start 通知在 run/session 持久化后发送并可导航 | `app_shell::state.data.scheduled-task-events.test`（toast 导航进入对应会话） | `Partial` |
| failed、timeout、missed 各自通知且绑定唯一事实 | `app_shell::state.data.scheduled-task-events.test`（重复投递同一事件只提示一次）、`app_shell::state.data.scheduled-task-startup-surface.test`（交错场景各恰好一次） | `Partial` |
| 通知发送失败不回滚或伪造运行状态 | `app_shell::state.data.scheduled-task-events.test`（host 调用失败仅保留历史入口） | `Partial` |
| 窗口不可见期间的通知在窗口重新可见后仍保留导航入口 | `app_shell::state.data.scheduled-task-events.test`、`app_shell::state.data.scheduled-task-startup-surface.test`（重获焦点后渲染 toast） | `Partial` |
| 升级后首次启动采纳现状，不逐条补报；其后的事实照常通知 | `app_shell::state.data.scheduled-task-startup-surface.test`（首启采纳 + 后续 pass 不回放） | `Partial` |

## Local Data Must Not Require Login Or Leave The Device

- 风险：本机路径、Prompt、run 结果或凭据引用被意外加入 Cloud 同步，或未登录用户无法使用 local task。
- 前置状态：Desktop 未登录或 Cloud 不可用，用户创建并运行 local task。
- 触发：definition/run 持久化、列表读取、应用重启及网络不可用。
- 必须成立：所有 local Scheduled Task 数据写本地 SQLite/本地运行资产；CRUD 与执行不需要账号；没有上传、跨设备同步或 Cloud fallback。
- 禁止结果：把本地 task 放入账号数据 API、因未登录拒绝运行，或将 Prompt/路径写入 Cloud telemetry payload。
- 决策依据：根决策 D1、D5、D8 与不变量 11。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| definition 通过 local SQLite repository 往返 | `ora_db::repository::scheduled_task::tests::definitions_round_trip_update_and_soft_delete` | `Partial` |
| 未登录、离线环境可完成 local CRUD 和执行 | 已有本机 SQLite 和真实 executor，实现不接收账号；尚缺未登录/断网贯穿 CRUD 与执行的直接测试 | `Missing` |
| contracts/backend 不调用 Cloud 同步或上传接口 | 尚无跨边界直接证据 | `Missing` |

