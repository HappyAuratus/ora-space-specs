---
status: implemented
---

# while-Loop：以持久轮次作用域隔离循环体，按条件、上限或显式早退结束

`loop` 执行一个独立 body DAG，每轮完成后求值 `until`；条件满足时成功，否则继续下一轮。
轮次、变量池、carried values 和退出清理意图持久化，默认达到硬上限即失败，可显式配置耗尽成功。
`loopExit` 提供不依赖下一轮 feedback 的提前退出。这里的 while 是后测循环，首轮总会执行。

本 ADR 按已合入实现记为 `implemented`，涉及 application 引擎、db scope repository、backend
清理、冻结图契约与编辑器/运行视图。证据见[while-Loop 核心用例](../../../../test-cases/desktop/core/workflow/while-loop.md)。
它不迁移成 foreach，不承诺嵌套、并行轮次或区域内 HITL。新图使用 schemaVersion 2；平图保持可读。

## 继承与修改

| 前序约定 | 处理 | 最终规则 |
| --- | --- | --- |
| [根决策](0-node-runtime-orchestration.md)的持久重算、DAG、变量 owner 与失效事件 | 继承，不变 | body 也是 DAG，状态从 repository 重算 |
| [迭代决策](20260915-iteration-composite-runtime.md)先实现 foreach | 继承，不变 | `iteration` 与 `loop` 是不同节点，不混用 item/index 与 carried values |
| [前序静默策略](20261008-prompt-inactivity-policy.md)冻结每个 Agent 节点的等待策略 | 继承，不变 | body 内 Agent 仍按冻结策略等待或安全失败，Loop 上限不取代 prompt 生命周期 |
| 根 D2：所有复合节点使用相同 advance 接口 | 修改 | Loop 注册为 ScopedLoop，以专用 scope 事务驱动；foreach 接口不变 |
| 根 D4：区域输出覆盖外层变量池 | 修改 | 每轮独立池，外层只能读 Loop 显式 outputs；feedback 同时赋值 |
| 根不变量 6：耗尽必然失败 | 修改 | 默认 fail，显式 succeed 只允许正常完成最后一轮后成功，不掩盖值/执行错误 |
| 原预留 `exit_when` | 修改 | 使用实际 `until` 配置，支持条件终止与独立 `loopExit` |

## 决策

### D1：容器归属显式，外层与 body 分别校验

`containerId` 是执行成员身份，UI 的 parentage/几何信息不代替它。每个 body 有 Start，内部无
Output 或嵌套复合节点，跨容器连边拒绝。Loop 是外层一个普通节点；body 行只在本轮作用域调度。
当前 while 图校验不拒绝 body 内 `interactive` Agent；这与 foreach 的显式拒绝不同。
解析接受不构成区域内停靠、人工完成及重启恢复的整链保证，相关边界见
[区域交互用例](../../../../test-cases/desktop/core/workflow/stateless-recovery-and-hitl.md#interactive-nodes-inside-a-region-park-and-resume-through-existing-paths)。

采用 ScopedLoop 而非把所有逻辑塞进 foreach runtime，是因为 while 需要拥有独立变量池、轮间
携带及早退清理事务。新增此显式形态修改了根决策“一种通用复合接口覆盖全部节点”的预期。

### D2：输入只初始化一次，feedback 在同一完成快照上同时求值

`variables` 声明类型、常量/外部 selector 初值和 feedback selector。初值在进入 Loop 时固定。
每轮只读可见祖先及全局输入，写入自己的 carried values 和 body 输出；不能写外层输入或读取
其它轮的私有输出。外层引用 body selector 在执行前拒绝，只能通过 Loop 的命名 outputs 导出。

正常轮结束时，全部 feedback 从同一已完成变量池读取并校验，再一次性决定继续；相互交换变量
不会受到逐条赋值顺序影响。缺值或类型冲突失败，不提交半组 feedback。分支选择也保存在本轮
scope 内，不能复用旧轮结果。

### D3：后测条件优先，上限策略只控制正常耗尽

`maxIterations` 必须为 `1..100`。正常轮完成并完成值校验后求值 `until`，条件满足优先成功；
包括最后一轮满足条件的情况。没有有效 until 比较视为未满足。

| 最后一轮结果 | maxIterationsBehavior | 结局 |
| --- | --- | --- |
| until 满足 | 任意 | 成功，原因 condition satisfied |
| until 未满足 | 省略或 fail | 失败，记录上限与有界观测摘要 |
| until 未满足 | succeed | 导出最后一轮 outputs，成功，原因 max iterations |
| 执行、feedback、until 或 outputs 出错 | 任意 | 失败 |

绝不发起第 N+1 轮。显式 succeed 支持固定预算迭代，但不是通用错误忽略开关；正常完成仍须通过
feedback 和 outputs 的实际校验，不能靠上限策略把坏数据包装成成功。

### D4：轮次结算与继续/父节点终态事务性提交

每轮为独立的 `workflow_execution_scopes` 行，保存父 Loop、从 1 开始的 round index、状态、
变量池与分支事实。body 的 node-run 绑定 scope，不借用 foreach 的零起 iteration。

旧轮结算和下一轮 scope/Start 创建，或父 Loop 终态与 outputs 发布，在同一 SQLite 事务提交。
重复回报不能创建第二个下一轮。普通调度波读取已提交 scope；不能只凭内存计数推测未完成副作用。
持久 scope 不意味着进程重启后自动接续当前轮；启动恢复的失败边界见 D6。

### D5：loopExit 持久化清理意图，完成后才发布退出

`loopExit` 只能是 body 内无出边的终端节点，只有被实际分支选中才触发。它从本轮快照解析
outputs，跳过下一轮 feedback；先持久化退出意图，再清理本轮仍在途的执行。清理成功后完成
Loop 并发布 outputs，清理失败则 fail closed。重启可重放 pending cleanup。

早退不取消外层并行工作，晚到 body 回报不能复活已结束 scope；若整个 run 已取消，迟到的
cleanup 不能重新提交成功。输出不可用同样失败。

### D6：显式 resume 从 Loop 第一轮重开，checkpoint 不因完成丢失

失败或取消关闭活动 scope；显式 resume 将受影响 Loop 从 round 1 重开，而不复用失败轮的
carried state。成功完成 Loop 时保留其进入前 checkpoint，供后续 resume 定位。外层兄弟失败
不把已独立提交的 Loop scope 任意改写；run 的 resume 边界由原有恢复 owner 处理。

进程重启时，boot sweep 不把 Running 的 `loop` 当作可吸收中断的 foreach composite：
普通执行轮使 run 以 `interrupted_by_restart` 失败，之后由用户显式 resume 从 round 1 重开。
已持久化 pending `loopExit` 的父 Loop 行被 sweep 豁免，交由清理恢复；该豁免不保护其它仍在途的外层工作。
普通调度波重算、启动 sweep 和用户 resume 是不同边界，不能以直接调用 engine.resume 的测试
证明自动重启续算，也不承诺已消失的 Agent 进程继续原副作用。

## 不变量

1. root/body 均为 DAG，执行归属不由 UI parentage 推测。
2. 外层只能经 Loop outputs 读取循环体产物。
3. 一轮只拥有一份持久变量池，feedback 从同一快照同时赋值。
4. 每轮结算与下一轮或父节点终态原子提交，重复回报不重复开轮。
5. 上限为 `1..100`，任何策略都不掩盖执行/值错误或创建超限轮次。
6. 早退先记清理意图，清理失败不成功，晚到回报不复活 scope/run。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| body 加回边 | 编辑简单，但破坏 DAG 和每轮所有权边界 |
| 沿用 foreach 的输出覆盖池 | 少一套存储，但轮间陈旧值可泄漏、feedback 赋值顺序不确定 |
| 到上限无条件成功 | 支持固定预算，却让未达到目标的循环默认静默成功；因此必须显式选择 succeed |
| early exit 立即宣告成功 | 响应快，但兄弟 body 执行仍可能写入；持久清理意图保证恢复及终态隔离 |

## 后果与验收

轮次作用域增加持久化和清理复杂度，换取可审计的隔离与恢复。核心验证覆盖图边界、同时赋值、
上限两策略、输出错误、重复回报、早退恢复/隔离和显式 resume；编辑器/真实 Desktop 的整链验收
缺口按核心用例如实登记，不以纯配置解析测试替代。
