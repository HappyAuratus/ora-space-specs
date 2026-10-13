# 无状态恢复与循环内人工介入

反应式调度从持久事实恢复或安全结束。本文件的自动轮次恢复用例针对 foreach；while 的显式
resume 从 round 1 重开，见[while-Loop 用例](while-loop.md)，不能将两者等同。区域内 HITL 尚非
当前交付承诺；第二个稳定标题保留历史验收目标，需后续 ADR 批准后再交付。

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

> 历史验收目标：原根决策的区域 HITL 目标已由 iteration ADR D5 收窄。当前 foreach 在执行前
> 拒绝 interactive 成员，while ADR 也不承诺完整区域 HITL。下列原目标不构成 implemented 的
> 系统承诺，Missing 表示未交付该扩展，不能据此开放区域交互。

### 风险

循环体内的交互节点若不能停靠 `Pending`，或人工完成后循环无法推进，HITL 与循环的组合就不可
用——而"reviewer 打回"恰恰是最核心的循环场景。

### 前置状态

- 循环体内的 Agent 节点声明 `interactive: true`。
- 该节点第一轮以 EndTurn 结束。

### 触发

第一轮结束；随后人工追问若干轮；最终人工完成该节点。

### 必须成立

- 节点停靠 `Pending`（AwaitingInput），迭代投影 `has_in_flight` 计为在途，外层不误判 run
  结束。
- 人工追问经既有 `begin_human_turn`/`end_human_turn` 路径，节点在 Running/Pending 间翻转。
- 人工完成的回报经统一 sink 进入引擎，触发调度波；调度波把 Running 的复合节点交还运行时
  （advance），循环自动推进决策或下一轮。
- 终态节点的会话只读，拒绝新 prompt。

### 禁止结果

- 循环体内交互节点被当作普通节点自动重试或跳过。
- 人工完成后需要专门的通知通道才能推进循环（决策明确禁止）。
- 终态节点的 session 仍可接收 prompt。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 循环体内交互节点停靠后人工完成，循环经调度波自动推进 | 尚无直接证据 | `Missing` |
| 终态节点会话拒绝新 prompt（既有行为在区域内保持） | 既有 `interactive` 路径测试不覆盖区域内场景 | `Missing` |

### 决策依据

同上决策 D5 与[迭代节点决策](../../../../decisions/desktop/core/workflow/20260915-iteration-composite-runtime.md)
D5：v1 结构校验禁止迭代区域内 interactive 节点。直接证据为
`ora_application::workflow_run::engine::iteration::tests::rejects_interactive_nodes_inside_the_region`
（拒绝义务 Covered）。[while-Loop ADR](../../../../decisions/desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md)
未认领该扩展，后续批准后才恢复上述 park/resume 验收目标。
