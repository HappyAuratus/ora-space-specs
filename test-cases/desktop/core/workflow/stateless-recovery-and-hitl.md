# 无状态恢复与循环内人工介入

反应式调度从持久事实恢复或安全结束。本文件的自动轮次恢复用例针对 foreach；while 的显式
resume 从 round 1 重开，见[while-Loop 用例](while-loop.md)，不能将两者等同。区域内 HITL 尚非
当前整链交付承诺；第二个用例明确 foreach 拒绝与 while 尚无整链证据的边界。

## Loop State Is Recovered From Persistence After A Crash

### 风险

复合运行时若在崩溃后无法从持久化行重建"当前第几轮、区域进行到哪"，迭代要么从头重跑
（浪费已完成的轮次），要么卡死在 Running。

### 前置状态

- 一个 Running 的 run，其迭代节点行 Running，第 k 轮区域行部分完成。
- 进程在任意时刻被杀（含区域节点 dispatch 后、回调前）。

### 触发

进程重启，boot sweep 扫描到该 run。

### 必须成立

- boot sweep 把 Running 的复合行交还其运行时续算（不按 `interrupted_by_restart` 一律标
  失败，仅对复合行生效）。
- 运行时从持久化行与账本重算轮次进度（v1 串行下从区域行的 `max(iteration)` 推导当前轮）；
  被打断的当轮按失败轮结算后继续，不重跑已 Succeeded 的轮与轮内节点。
- 区域内的非交互 `Pending` 行按既有 fail-closed 规则处理，不误判为可恢复。

### 禁止结果

- foreach 自动恢复无理由重跑已完成轮；不涵盖用户显式 resume while 的 round 1 重开。
- run 永久滞留 Running 无 in-flight 节点。
- 恢复路径依赖崩溃前的内存状态或日志文件。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 第 k 轮中途崩溃后恢复到第 k 轮续算，已完成轮内节点不重跑 | `ora_backend::workflow::run::recovery::tests::a_crashed_round_inside_a_running_iteration_is_absorbed_not_failed`（真实 boot sweep + reconcile：轮 0 已完成后崩溃于轮 1，轮 0 仅一行不重跑，轮 1 按失败结算后节点完成） | `Covered` |
| 既有"Running 无在途节点则 resume"语义在含 Loop 的 run 上成立 | `ora_backend::workflow::run::recovery::tests::a_crashed_round_inside_a_running_iteration_is_absorbed_not_failed`（sweep 保留 Running 复合行后 `reconcile_running_workflow_runs` 经 resume 驱动 advance 续算至终态） | `Covered` |

### 决策依据

[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)
不变量 2 与 D4 末段；迭代节点的落地语义见
[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D2，被打断轮的结算细节见[迭代节点核心用例](iteration-node.md)。

## Interactive Nodes Inside A Region Park And Resume Through Existing Paths

### 风险

把图解析接受误认为循环内 HITL 已完整交付，或把 foreach 的拒绝规则误套到 while。

### 前置状态

foreach 或 while 的区域内含 `interactive: true` Agent。

### 触发

图校验，以及 while 中 Agent 停靠、人工完成和进程重启。

### 必须成立

foreach 在执行前拒绝 interactive 成员。while 当前解析器不作同样拒绝；接受配置不能证明
停靠、人工完成后推进、终态会话只读及启动恢复的联合行为已验证。
while 普通 Running 轮的启动 sweep 仍遵守 while ADR D6 的失败边界。

### 禁止结果

把普通平图 interactive 测试作为区域内整链证据，或因 while 解析通过便承诺自动重启续跑。

### 验证义务与证据

| 验证义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| foreach 执行前拒绝 interactive 成员 | `ora_application::workflow_run::engine::iteration::tests::rejects_interactive_nodes_inside_the_region` | Covered |
| while 接受 interactive 配置后的停靠、人工完成、终态只读与重启组合 | 尚无对应区域内直接整链证据；普通 interactive 测试不能替代 | Missing |

### 决策依据

[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md) D5；
[while-Loop ADR](../../../../decisions/desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md) D1、D6。
完整区域 HITL 的产品保证需后续 ADR 承接，不由解析器是否接受配置推导。
