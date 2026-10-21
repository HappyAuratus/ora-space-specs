---
status: proposed
---

# Scheduled Run 的 worktree 归属在任务提交事务内原子建立

本 ADR 处于 `proposed`，内容已获 owner 在 [ora-space/circus#376](https://github.com/ora-space/circus/issues/376) 逐项批准（其中存量孤儿的处理范围被缩小），待本 PR 评审通过后标记为 `approved`。它解决一个已核实的崩溃窗口：local Scheduled Run 在
`TaskApi::create`（provisioning）成功之后、`ScheduledRunTransition::Start` 落库之前进程死亡，会留下与运行行无链接的隔离
worktree 任务——启动恢复把运行结算为 `failed`，而保留清理清单要求链接非空，该 worktree 与分支从此不可见、不可回收。

最终方案：把 Scheduled Run 的 worktree 归属并入既有的任务提交事务——Git worktree 创建前先持久化
`WorktreeProvisioningLease`，scheduled 用例的内部提交适配器在同一个 SQLite immediate 事务里提交 task/workspace/worktree
与运行行的 `worktree_task_id + worktree_path`，并在**同一事务内**删除 lease。提交前崩溃由 lease 的既有补偿回收；提交后
崩溃运行行已带稳定关联，交给既有恢复与 janitor，且事务提交成功后不再存在 lease 残留。伴随两处修正：retention 对缺路径
运行的分类不得把未知当干净，且必须先处理「任务已不存在」的清链；session 恢复复用既有 `reconcile_running_sessions`，
不新增机制。涉及模块：desktop 的 scheduled_task runtime、task provisioning 接缝、scheduled-run repository 与 worktree
janitor。兼容性边界：普通任务与 manual run 的 provisioning 路径、`TaskApi::create` 公开契约、10 个保留上限与 100 条历史
规则不变；本 ADR 不承担存量孤儿的数据迁移义务（见 D5）。

## 继承与修改

前序决策：根决策 [0-local-and-cloud-execution-ownership.md](0-local-and-cloud-execution-ownership.md)（下称根决策）与
owner 在 ora-space/circus#254 中确认的「仍带保留 worktree 链接的运行不参与历史裁剪」例外。

| 约定 | 处置 |
| --- | --- |
| 根决策 D4：每次 local 运行使用独立 worktree；无改动直接删除、有改动保留并提供打开/丢弃；保留数量有界（10 个） | 继承，不变 |
| 根决策 D4：worktree 创建复用 Desktop 已有的初始化端口 | 继承，不变；lease 与提交适配器复用同一接缝，不另建第二套 provisioning |
| 根决策 D7：运行状态机 `accepted / running / …` 与 60 分钟超时 | 继承，不变；`Start` 仍是 `running` 的唯一写入者 |
| 根决策 D7 与 #254 例外：100 条历史裁剪豁免仍带保留 worktree 链接的运行，链接清除后恢复可裁剪 | 继承，不变；链接语义变化不改变豁免规则本身 |
| 根决策 D7：通知是交互辅助不是事实源，`RunStarted` 在运行实际开始后发布 | 继承，不变；归属提前建立不提前通知（D2） |
| 根决策 D4/D7 隐含：`worktree_task_id` 由 `Start` 写入，含义是「运行已在该 worktree 启动」 | **修改**：链接提前到任务提交事务内写入，含义改为「任务提交事务已为该运行建立 worktree 归属」；`Start` 改为校验并沿用原绑定（D2） |
| 根决策 D4 隐含：保留清理只处理带路径的已完成运行 | **修改**：retention 扩展到带链接但缺路径的运行，先清「任务已不存在」的陈旧链接，再解析真实路径分类；未知与失败一律保守保留（D3） |
| 根决策未涉及：session 的启动期状态收敛 | **新增**：复用既有 `reconcile_running_sessions` 并补装配顺序回归（D4） |

## 决策

### D1：归属随任务提交原子建立

scheduled 用例提供一个内部提交适配器：Git worktree 创建前先持久化 `WorktreeProvisioningLease`（write-ahead，既有接缝，
`application/src/task/handlers.rs`）；提交阶段在**同一个 SQLite immediate 事务**内写入 task、workspace、worktree 与
运行行的 `worktree_task_id + worktree_path`，并在**同一事务内**删除 provisioning lease。不给公开 `CreateTaskRequest` 加
scheduler 专用参数，runtime 不手写多段 SQL 拼事务。

承诺：

1. 提交前崩溃：lease 仍由既有补偿路径回收，不存在无主资源；提交后崩溃：运行行已带稳定关联，启动恢复把它结算为
   `failed` 后，retention 仍能看到并处理该 worktree。系统不存在「任务已提交但没有持久归属」的中间态。
2. 提交时校验运行行仍为 `accepted`、project 匹配且无冲突绑定；重放不创建第二份资源、不覆盖既有绑定；校验或写入失败
   时整个任务提交回滚，lease 留给既有补偿路径，本次运行按既有失败路径结算。
3. lease 的删除与任务行提交在同一事务内：事务提交成功后不存在 lease 残留——残留 lease 会被到期回收转为 cleanup job，
   回收已归属运行的资源；lease 删除失败时整个提交回滚，资源回到 lease 手中。
4. 普通任务与 manual run 继续走原提交路径；scheduled 语义由 scheduled 用例拥有，共用的 Git 创建、Effect 接入与删除
   级联由原所有者负责。

这样选的原因：崩溃窗口的本质是「资源创建」与「归属记录」分属两个提交。「create 返回后另写一次链接」仍是两个独立提交，
窗口只被缩短而非消除，不满足「崩溃后仍可证明归属」的不变量；任务提交事务是 provisioning 已有的原子接缝，把归属并
入其中是唯一不需要新增资源账本或状态机的完整封闭方式。代价是改动触及任务提交接缝，实现与测试面大于一次仓储写入。

### D2：`accepted` 运行可以拥有已 provision 的 worktree，`Start` 校验原绑定

链接写入提前后，`accepted` 状态的运行行即可持有 `worktree_task_id + worktree_path`；这不改变运行状态机——`Start` 仍是
`running` 的唯一写入者，且必须校验并沿用提交事务建立的原绑定，不得另建或换绑。通知边界不变：`RunStarted` 在 session
真正就绪后才发布，归属提前建立不产生任何提前通知；「每次定时运行开始时发出桌面通知」的承诺（根决策 D7）不受影响。

失败与取消不得清除尚未完成回收的关联：运行失败或被取消时，若 worktree 尚未丢弃，链接保留给 retention 与用户入口
（打开/丢弃）；只有任务删除成功、或清理路径确证任务已不存在时才清链（与 #254 例外中的恢复条件一致）。

边界与反例：`accepted` 运行带链接 ≠ 用户可见的「运行中」；运行历史与侧栏不得把带链接的 `accepted` 运行渲染为 running。
链接也不是丢弃凭证——丢弃仍走用户显式操作或 retention 分类，不因「链接存在」自动删除。

### D3：retention 扩展——先清链，再按真实路径分类，未知不当干净

retention 扫描扩展到带链接但缺路径的运行，顺序固定：

1. 任务行已不存在：按既有「陈旧链接」规则清链，让该运行恢复可裁剪（先于一切路径处理，避免空路径陈旧链接永久豁免历史
   裁剪）。
2. 缺路径但任务存在：从确证关联的 task/workspace 解析真实 worktree 路径，复用 baseline + Git status/diff 分类；解析
   失败视为分类失败，保守保留。
3. 分类结果：无改动照常丢弃；有改动保留；实际 in-use（有活跃 session 或 workflow）不自动删；10 个保留上限与删除顺序
   不变。

禁止把 `worktree_path = NULL` 当作无改动证据：NULL 只表示字段尚未登记，不证明重启前后无人修改文件。既有
`run_finished(None)` 把未启动运行按「全新无改动」直接丢弃，是因为该分支持有 provisioning 当场的确定性；重启后的恢复
不具备这一前提，不得沿用该假设。

### D4：session 恢复复用既有机制，只补装配顺序回归

Agent runtime 启动时的 `reconcile_running_sessions` 已把存储中 `Running` 的 session 收敛为 `Stopped`，且 bootstrap
顺序保证它先于 scheduled 恢复执行。本决策不新增 session 启动恢复机制，只要求补一条回归：真实 bootstrap 下，残留
`Running` session 先收敛、干净 worktree 随后可进入清理；用户重启后重新打开的会话仍正常阻止删除。同时明确边界：该恢复
是 SQLite 生命周期收敛，不宣称能证明所有外部进程已退出。

### D5：存量孤儿只做文档说明，不做自动认领

崩溃窗口窄、功能上线时间短，存量孤儿预期很少，且它们本就是任务列表可见、可手动删除的普通隔离任务。因此不做诊断与确
认流程，不写自动认领或删除代码；在 release note / 用户文档说明如何识别（标题形如 `Scheduled run {UUID}`、无关联运行
的隔离任务）与手动清理。标题可改可复制、历史可被裁剪，不构成可编程的归属证明。若实施中发现这类任务在 UI 不可见或删
不掉，另开 issue 处理。

## 不变量

1. 任一崩溃时点后，provision 出的 worktree 资源都存在可证明的 owner：提交前归 lease 补偿，提交后归运行行链接。
2. 归属写入、task/workspace/worktree 提交在同一事务内；不存在「任务已提交但没有持久归属」的中间态。
3. 重复恢复不产生第二个任务、第二次资源创建或重复副作用。
4. `Start` 是 `running` 的唯一写入者，且沿用并校验提交事务建立的原绑定；`RunStarted` 通知不因归属提前而提前。
5. 失败、取消不清除未完成回收的关联；只有任务删除成功或确证任务已不存在才清链。
6. `worktree_path` 缺失、解析失败、分类失败一律保守保留，不得当作已清理或无改动。
7. 普通任务与 manual run 的提交路径、公开 `TaskApi::create` 契约保持不变。
8. 100 条历史裁剪豁免与 10 个保留上限不变；清链条件与 #254 例外一致。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| `create` 返回后另写一次链接（原方案 A） | 实现最小，但仍是两个独立提交：任务提交后、链接提交前仍可被杀，链接本身也可能写失败；只能缩短窗口，不满足「崩溃后仍可证明归属」 |
| 启动时按标题模式认领孤儿（方案 B 常驻化） | 无需新持久化，但标题可修改、可复制、run 行可被历史裁剪，归属不可证明；误删风险不可归零，只适合做用户显式确认下的一次性诊断（D5 已进一步缩小为纯文档说明） |
| 新增 session 启动恢复机制 | 既有 `reconcile_running_sessions` 已覆盖该场景且先于 scheduled 恢复执行，属重复建设 |
| 把 `Start` 前移到 provisioning 之前、或拆成两个转换 | `Start` 携带 `session_id + worktree_path + worktree_task_id`，前移必然拆分状态机并引入「running 但无 worktree」的中间态，契约变化大于收益 |
| 为毫秒级窗口扩展通用 provisioning 状态机 | 窗口确实只有数秒，但泄漏是永久的且不可见；复用既有 lease + 事务接缝即可完整封闭，无需新状态机 |

## 本决策未解决的问题

- 存量孤儿在 UI 中不可见或无法手动删除的情形（owner 指定：实施中发现时另开 issue）。
- 点击系统通知直接导航等通知交互增强，仍属后续决策，与本 ADR 无关。
- Cloud Scheduled Task 的对应归属语义：Cloud 侧资源创建与归属记录的原子性由 Cloud 域 ADR 自行定义，本 ADR 不预置。

## 核心测试用例

本 ADR 的核心用例维护在
[`test-cases/desktop/core/scheduled-task/worktree-ownership-and-recovery.md`](../../../../test-cases/desktop/core/scheduled-task/worktree-ownership-and-recovery.md)，
覆盖：provisioning 各阶段故障注入后每次恢复仍有资源 owner 且重复恢复不产生第二份资源；归属提交校验失败整体回滚且
lease 补偿仍在；bootstrap 装配顺序下残留 session 先收敛、干净 worktree 随后可清理；retention 先清链再按真实路径分
类，未知与失败保守保留；普通任务、manual run、occurrence 幂等、100 条历史与 10 个保留上限不回退。既有用例
[execution-isolation.md](../../../../test-cases/desktop/core/scheduled-task/execution-isolation.md) 与
[history-notifications-and-residency.md](../../../../test-cases/desktop/core/scheduled-task/history-notifications-and-residency.md)
继续有效，其中保留链接豁免义务的证据随实现补充。

## 落地顺序

1. **specs**：本 ADR 评审通过标记 `approved`，同步建立核心测试用例（义务初始为 `Missing`）。
2. **desktop 实现**（[ora-space/circus#379](https://github.com/ora-space/circus/issues/379)）：提交适配器、`Start`
   校验、retention 修正扩展、装配顺序回归与故障注入测试；产品 PR 在 ADR `approved` 后交付，含存量孤儿的用户文档说明。
3. **落地验收**（[ora-space/circus#380](https://github.com/ora-space/circus/issues/380)）：合并后在 `main` 验收不变
   量，补证据状态，ADR 转 `implemented`。

行为变化边界：第 2 步合入前，现有行为（`Start` 才写链接、崩溃窗口产生孤儿）保持不变；合入后新运行不再产生孤儿，存量
孤儿按 D5 由用户手动处理。
