# Workflow 核心测试用例

本目录是工作流编排领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/core/workflow) 镜像。

架构边界和不变量来自
[节点运行时编排决策](../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md)。

## 阅读与维护规则

- 核心用例按业务风险组织，不按单元测试、集成测试或 crate 组织。
- 一个核心用例可以由纯状态测试、repository 测试和跨 adapter 测试共同证明。
- 只有在测试失败会直接说明某项验证义务不成立时，才把该测试列为代表性证据。
- 文件路径与用例标题共同构成稳定引用；代表性测试可以在 doc comment 中引用它们。

## 当前状态

节点运行时重构、区域机制与第一个 foreach 迭代复合运行时均已实现并验证：引擎核心经运行时注册表
分发，轮次与账本由持久化事实重算，编辑器通过显式复合区域入口维护成员图，运行视图按轮投影。
证据状态按各文件中的验证义务持续维护。

| 文件 | 覆盖的风险面 |
|---|---|
| [engine-parity.md](engine-parity.md) | Stage 1 重构行为等价；调度层无节点类型字面量 |
| [region-and-loop.md](region-and-loop.md) | 区域黑盒与结构校验；DAG 不变量；迭代持久化；`max_iterations` 硬上限 |
| [stateless-recovery-and-hitl.md](stateless-recovery-and-hitl.md) | 崩溃后从持久化续算循环；循环体内 HITL 停靠与推进 |
| [runtime-forms.md](runtime-forms.md) | 快执行器锁内无 IO；session 归属；回报幂等 |
| [variable-pool-boundary.md](variable-pool-boundary.md) | 一切节点形态经声明 owner 与类型校验写入 |
| [invalidation-events.md](invalidation-events.md) | 失效事件不携带工作流状态，重查收敛 |
| [iteration-node.md](iteration-node.md) | foreach 轮次、显式区域编辑、容器几何、级联删除与 Desktop 真实执行 |

"未解决的问题"中的预留项（节点级错误策略、并行迭代、嵌套循环、Code 沙箱、远程子工作流细节）
不登记核心用例：它们尚未构成系统承诺，待各自后续 ADR 批准后随其义务登记。
