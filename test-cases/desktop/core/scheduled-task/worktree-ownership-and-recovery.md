# Scheduled Task worktree 归属与恢复

本用例服务后续 ADR
[20261020-worktree-ownership-atomic-commit.md](../../../../decisions/desktop/core/scheduled-task/20261020-worktree-ownership-atomic-commit.md)
（决策单 ora-space/circus#376 批准的 A′ 原子归属提交），覆盖归属建立的故障注入、恢复与 retention 分类边界。

## Crash Around Provisioning Must Always Leave A Durable Owner

- 风险：provisioning 与归属记录分属两个提交时，进程死亡留下无主隔离 worktree，清理路径不可见，资源永久泄漏。
- 前置状态：local 任务已接纳 occurrence，运行处于 `accepted`；lease、Git worktree、任务提交事务与 session 启动可能
  在任意交错点被注入崩溃。
- 触发：在 lease 提交后、Git worktree 创建后、task/workspace/worktree 与归属事务前后、session 持久化后 `Start` 前
  分别注入进程死亡并重启。
- 必须成立：每个时点后资源仍有可证明的 owner——提交前崩溃由 lease 既有补偿回收，提交后崩溃由运行行链接承载并进入
  既有恢复与 retention；重复恢复不产生第二个任务、第二次资源创建或重复副作用；带链接的 `accepted` 运行不渲染为
  running，也不提前发 `RunStarted`。崩溃验证通过重新打开数据库核对落盘状态，且至少一组注入跳过正常 Drop/补偿路径，
  避免只测到正常错误返回的补偿。
- 禁止结果：任务已提交但没有持久归属的中间态；恢复路径绕过 lease 或运行行自行创建第二份资源；归属提前触发通知。
- 决策依据：后续 ADR D1、D2 与不变量 1–4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| lease 提交后、Git 创建后崩溃：lease 补偿回收资源，无残留任务行 | 尚未实现 | `Missing` |
| 归属事务提交后崩溃：运行行带链接，恢复后由 retention 可见并分类 | 尚未实现 | `Missing` |
| session 持久化后 `Start` 前崩溃：归属仍完整，`Start` 缺失不产生 `running` 或 `RunStarted` | 尚未实现 | `Missing` |
| 重复恢复（多次重启）不产生第二个任务或重复副作用 | 尚未实现 | `Missing` |
| 带链接的 `accepted` 运行在历史与侧栏不显示为 running | 尚未实现 | `Missing` |
| 成功提交后重新打开数据库：task/run 绑定存在且 provisioning lease 已不存在（成功后无 lease 残留） | 尚未实现 | `Missing` |

## Failed Ownership Commit Must Roll Back And Keep Compensation

- 风险：归属校验或写入失败时部分提交，产生悬空资源或覆盖既有绑定。
- 前置状态：scheduled 提交适配器执行中；运行行可能已被启动恢复结算为终态、project 可能被并发删除、绑定可能冲突。
- 触发：运行行非 `accepted`、project 不匹配、绑定冲突、写入失败时提交。
- 必须成立：整个任务提交回滚；lease 留给既有补偿路径；重放不创建第二份资源、不覆盖既有绑定；本次运行按既有失败路径
  结算；在删 lease 处注入失败时同样整体回滚且不启动 session。
- 禁止结果：部分提交、绕过校验覆盖绑定、回滚后 lease 丢失或资源无主、吞掉 lease 删除失败并保留任务行。
- 决策依据：后续 ADR D1 与不变量 2、3、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 运行行非 `accepted` / project 不匹配 / 绑定冲突时提交整体回滚 | 尚未实现 | `Missing` |
| 回滚后 lease 由既有补偿路径回收 | 尚未实现 | `Missing` |
| 重放不创建第二份资源、不覆盖既有绑定 | 尚未实现 | `Missing` |
| 删 lease 处注入失败：task/workspace/worktree 与 run 绑定整体回滚，lease 保留补偿责任且不启动 session | 尚未实现 | `Missing` |

## Session Start Failure Must Not Loose The Ownership Binding

- 风险：归属提交后 session 启动失败或 `Start` 被拒绝时清掉关联或伪造开始通知，运行历史失去打开/丢弃已 provision worktree
  的入口。
- 前置状态：归属提交事务已成功（运行行带 `worktree_task_id + worktree_path`），session 启动失败或 `Start` 校验原绑定被拒。
- 触发：运行经既有失败路径结算，或 runtime 收到 `InvalidState`。
- 必须成立：失败与取消不清除归属绑定（清链仅发生在任务删除成功或确证任务已不存在）；不发布 `RunStarted` 或任何虚假开始
  通知；worktree 交给 janitor/retention 按运行终态分类处理。
- 禁止结果：失败路径删除绑定、把 `accepted` 带绑定渲染为 running、或发出未经 session 就绪的开始通知。
- 决策依据：后续 ADR D2 与不变量 4、5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| session 启动失败：绑定保留，运行按 `TargetUnavailable` 结算且不发 `RunStarted` | 尚未实现 | `Missing` |
| `Start` 校验原绑定被拒：不换绑、不丢失关联，runtime 走停会话 + janitor 丢弃分支 | 尚未实现 | `Missing` |

## Link Clearing Must Keep Durable Cleanup Responsible For Disk State

- 风险：任务删除后、清链前再次崩溃，或 durable Git cleanup 失败重试期间清链——磁盘上的 worktree/分支失去最终回收者。
- 前置状态：带链接运行的 worktree 任务已被删除（历史操作或 retention 丢弃），清链与磁盘回收分属两步。
- 触发：清链提交后、durable cleanup job 完成前进程死亡；或 cleanup job 失败重试。
- 必须成立：下次恢复仍识别「任务已不存在」并清链，清链后运行恢复可裁剪；Git worktree/分支的最终磁盘回收始终由持久
  cleanup job 承担，失败可重试，不因清链或重启遗失；任务行删除或清链不能单独证明磁盘资源已回收。
- 禁止结果：清链后磁盘资源无人认领、cleanup job 被取消或去重掉、或以「清链成功」宣称磁盘已回收。
- 决策依据：后续 ADR D2、D3 与不变量 5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 任务删除后、清链前崩溃：下次恢复清链，运行恢复可裁剪 | 尚未实现 | `Missing` |
| Git cleanup 失败重试仍负责磁盘回收，清链不取消 cleanup job | 尚未实现 | `Missing` |

## Startup Session Recovery Must Converge Before Scheduled Recovery

- 风险：残留 `Running` session 阻止 worktree 清理，或恢复顺序颠倒导致清理误判。
- 前置状态：上个进程崩溃时留下 `Running` session 行与可能的 scheduled worktree。
- 触发：真实 bootstrap 重启。
- 必须成立：`reconcile_running_sessions` 先把残留 `Running` session 收敛为 `Stopped`，先于 scheduled 恢复执行；干净
  worktree 随后可进入清理；用户新启动的会话仍正常阻止删除。
- 禁止结果：残留 `Running` session 永久阻止回收；新增第二套 session 恢复机制；宣称该恢复能证明外部进程已退出。
- 决策依据：后续 ADR D4 与不变量 1。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 真实 bootstrap 下残留 `Running` session 先收敛为 `Stopped`，先于 scheduled 恢复 | 尚未实现 | `Missing` |
| 收敛后干净 worktree 进入清理；用户新启动会话仍阻止删除 | 尚未实现 | `Missing` |

## Retention Must Classify Without Treating Unknown As Clean

- 风险：把缺路径或分类失败当作无改动，静默删除可能带结果的 worktree；或陈旧链接永久豁免历史裁剪。
- 前置状态：带链接的已完成运行，`worktree_path` 可能为 NULL；任务行可能已删除；worktree 可能 dirty、untracked、
  committed 或 in-use。
- 触发：retention 扫描处理带链接但缺路径的运行。
- 必须成立：任务行已不存在时先清链并恢复可裁剪；缺路径从确证关联的 task/workspace 解析真实路径再按 baseline + Git
  分类；dirty/untracked/committed 保留；解析或分类失败保守保留；in-use 不自动删；10 个上限与删除顺序不变；清理经既有
  durable Git cleanup 收敛，再次恢复行为一致。
- 禁止结果：把 `worktree_path = NULL` 当无改动证据、把分类失败当「已清理」、跳过清链直接处理空路径、或改写保留上限。
- 决策依据：后续 ADR D3 与不变量 5、6、8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 任务行已删除时先清链，运行恢复可裁剪 | 尚未实现 | `Missing` |
| 缺路径运行从 task/workspace 解析真实路径并正确分类 dirty/untracked/committed | 尚未实现 | `Missing` |
| 解析或分类失败保守保留，不当作已清理 | 尚未实现 | `Missing` |
| in-use 不自动删；10 个上限与最旧优先顺序不变 | 尚未实现 | `Missing` |
| 清理经 durable Git cleanup 收敛，再次恢复行为一致 | 尚未实现 | `Missing` |

## Regular Task Provisioning And Manual Runs Must Stay Unchanged

- 风险：scheduled 归属语义外溢，改变普通任务或 manual run 的提交路径与公开契约。
- 前置状态：普通任务创建、manual run、occurrence 幂等、100 条历史与保留链接豁免均为既有行为。
- 触发：scheduled 提交适配器合入后的回归。
- 必须成立：普通任务与 manual run 继续走原提交路径；`TaskApi::create` 公开契约不变；occurrence 幂等、100 条历史、
  保留链接豁免与 10 个上限不回退。
- 禁止结果：给公开 `CreateTaskRequest` 加 scheduler 专用参数、manual run 被卷入 scheduled 清理、历史裁剪规则被改写。
- 决策依据：后续 ADR D1、D5 与不变量 7、8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 普通任务与 manual run 提交路径不回退 | 尚未实现 | `Missing` |
| occurrence 幂等、100 条历史与保留链接豁免不回退 | `ora_db::repository::scheduled_run::tests::history_pruning_keeps_runs_that_still_reference_a_retained_worktree`、`ora_db::repository::scheduled_run::tests::one_hundred_and_first_history_entry_prunes_only_the_oldest_fact` | `Partial` |
