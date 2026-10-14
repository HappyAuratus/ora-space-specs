---
status: implemented
---

# 工作流编排：以注册运行时驱动节点，以持久事实重算调度

工作流保持反应式调度：从 SQLite 的冻结图、node-run、变量和作用域事实重算每个调度波。
节点执行策略通过运行时注册表分发；区域在外层表现为黑盒，进度以提交后的失效事件通知观察者
重查。首个复合运行时是 foreach 迭代，while-Loop 随后采用独立轮次作用域落地。

本决策已实现于 application 引擎、backend adapter、db repository、契约和编辑器/运行视图。
对应证据见[Workflow 核心用例](../../../../test-cases/desktop/core/workflow/README.md)。
既有 start/agent/condition/output 平图继续执行；未知节点经图解析拒绝，不产生半解析运行。
本决策不承诺未来 Code、HTTP、远程节点或嵌套循环已经交付。

## 后续决策的继承与修改

| 原约定 | 后续决策 | 最终规则 |
| --- | --- | --- |
| D4：while-Loop 是首个复合运行时 | [foreach 迭代](20260915-iteration-composite-runtime.md) | 首先遍历冻结数组，零起 iteration 与账本是持久事实 |
| D5：区域内 HITL 直接沿用 | foreach 迭代 D5 | 首版 foreach 区域禁止 interactive，避免未定义的恢复交互 |
| D4：仅用 iteration 行推导全部循环进度 | [while-Loop](20261013-while-loop-with-durable-round-scopes.md) | while 使用显式 round scope、独立变量池和 carried values |
| 不变量 6：Loop 达上限一律失败 | while-Loop D3 | 默认 fail，可显式 succeed；两者均禁止第 N+1 轮，执行/值错误仍失败 |
| D2：复合节点统一使用 advance plan | while-Loop D1 | foreach 保持 composite advance，while 由注册的 ScopedLoop 驱动专用作用域调度 |

## 决策

### D1：主调度波按运行时注册表分发节点策略

`run_schedule` 经注册表取得节点执行形态，保留重算投影、启动 ready、分发、回报与收尾。
Start、Condition、Output、Aggregator 和 LoopExit 的同步行为由 swift runtime 负责；Agent
由异步 runtime 负责；foreach 使用 composite runtime。注册元数据也决定输出优先级和重试策略。

实际实现并未把所有持久化动作归入同一个通用 sink：Condition 分支、Output 暴露和 Loop scope
启动仍由拥有相应事务的引擎/repository 协作完成。while 的 `ScopedLoop` 是显式形态扩展，
不能把“新增任意复合节点只注册一行”继续写成普遍承诺；选择它是为了完整拥有轮次事务和隔离。

### D2：执行形态与事务边界明确

| 形态 | 最终边界 |
| --- | --- |
| swift | 调度门内纯计算，无 IO、等待或无界工作 |
| async | 分发后由后台 owner 驱动，完成/失败重入 per-run 串行门 |
| composite | 从持久事实生成续算计划，由 repository 提交 |
| ScopedLoop | 以独立持久 round scope 驱动 body，作用域事务决定推进和退出 |

同一 run 的状态变化在 per-run 排他门及对应 SQLite 事务内提交。等待 Agent、网络或人工输入
不跨越该门；只有 Agent 执行创建并绑定 session。晚到和重复回报依据持久终态幂等拒绝。

### D3：区域是外层黑盒，每个作用域仍是 DAG

foreach 根据冻结 parent/entry 关系形成 region；while 使用 `containerId` 明确成员并将外层与
body 分为独立 DAG。区域成员不进入外层 ready/current_nodes，区域内输出不会自动泄漏外层。

结构校验在执行前拒绝：无有效入口/不可达成员、body 内 Output、嵌套复合、跨边界连边或指回
body 的外层边，以及非法轮数配置。foreach 的安全上限至少为 1；while 为 `1..100`，body 必须
有自己的 Start。每种区域的精确图形与变量规则由其后续 ADR 固定，不把 UI parentage 当执行归属。

### D4：foreach 与 while 的轮次均为持久事实，编码分别拥有

foreach 的轮变量、按轮 node-run、分支决策及账本事务性提交；当前轮只读取本轮行，不能将上轮
陈旧输出冒充本轮收集结果。while 的 round scope 保存一基轮次、独立变量池、分支决定和 carried
values；旧轮结算与新轮启动或父 Loop 完成在同一事务提交。

重算不依赖内存循环计数。终止、上限、早退及 resume 语义以
[while-Loop 决策](20261013-while-loop-with-durable-round-scopes.md)为准；foreach 的数组长度
与启动上限语义以[迭代决策](20260915-iteration-composite-runtime.md)为准。

### D5：回报推进作用域，失败按明确的传播边界处理

区域内完成回报触发新的调度波；在途行阻止外层提前结束。foreach 的 `continue` 仅吸收该区域
内失败并记账，复合节点自身失败仍传播 run。while 的执行、取值或 cleanup 失败传播失败；取消
关闭活动作用域，不能被晚到回报复活。foreach v1 禁止区域内 HITL，不把普通 Agent 的交互能力
推导为所有复合区域的交互保证。

### D6：变量、输出和并发由各自 owner 保证

变量池以声明 owner 和类型验证写入，区域不能绕过该边界。Aggregator 是 swift runtime：候选
输入只能有一个已赋值值，多值冲突失败。并发来自 DAG ready 集合与扇入规则，不需要并行容器。
未来节点的 IO、沙箱、超时和失败恢复须由相应 ADR 决定；不能在注册表扩展名义下跨越调度锁等待。

### D7：状态提交后发失效事件，观察者重查持久状态

事件只标识受影响 run，不携带另一份工作流状态。前端重查读取接口，丢失、重复或乱序事件由后续
重查收敛。Agent 的实时 ACP 流仍由 session 通道负责，不能用失效事件代替输出流。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 通用有环图 / token 调度器 | 表达力更大，但重写 DAG、血缘和恢复基础，并失去结构化轮次边界 |
| 请求期内存队列编排 | 可降低派发开销，但桌面进程中断后必须另建持久恢复协议 |
| 全部节点塞进 Agent executor | 可复用入口，却把同步与复合语义绑在 session 生命周期上 |
| 所有循环使用同一 iteration 字段 | 少一套存储，但不能直接表达 while 的隔离池、携带变量和早退清理 |

## 不变量

1. 外层及每个 body 的冻结图均为 DAG。
2. 调度从持久事实重算；恢复可续算或安全结束，不依赖内存轮计数。
3. 同一 run 的状态变化串行并事务性提交；swift 计算无 IO/等待。
4. 变量写入经过声明 owner 与类型校验。
5. 区域内部节点不进入外层 ready/current_nodes。
6. foreach 在启动边界验证数组长度；while 的最大轮数为 `1..100`，默认耗尽失败，显式 succeed
   只改变耗尽结果；任何策略都不创建超限下一轮或掩盖实际执行/取值错误。
7. body 无 Output 或嵌套复合，连边不能跨作用域。
8. 只有 Agent 节点创建并绑定 session。
9. 既有平图兼容，未知类型在执行前拒绝。
10. 失效事件不携带工作流状态，SQLite 是唯一事实源。

## 后果与后续边界

显式运行时和持久作用域提高可恢复性，也增加了轮次事务与失败吸收的验证义务。嵌套、并行迭代、
复合区域 HITL、Code/远程执行及跨机器 run ownership 分别需要后续决策；不能将这些预留写成
已实现的系统承诺。节点级错误策略向任意节点的泛化、循环体失败重试也仍需后续 ADR；
foreach 的 `FailurePropagation` 只覆盖当前区域失败吸收，不能代替这两项预留。
相关核心用例保留实际的 Covered、Partial、Missing 证据状态。
