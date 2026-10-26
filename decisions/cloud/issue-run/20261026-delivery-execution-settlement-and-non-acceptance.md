---
status: proposed
---

# 交付截止由 Cloud 结算业务责任，未受理事实由 Node 的持久围栏证明

交付截止后，Cloud 在业务结算事务中关闭原交付执行的新派发资格，并独立记录停止等待的来源；
Controller 不能把这笔结算或一次 `Unknown` 当作 Node 空闲。解除未完成执行责任须有 Node 在原身份下
持久证明“没有受理且以后也不能受理”，或者既有受控处置取得真实终止和迟到创建围栏。

本文细化已经批准的 [A+B 职责分离](https://github.com/ora-space/circus/issues/670)，当前为 `proposed`：
具体持久化、证明协议和兼容边界尚待评审。覆盖 IssueRun 的 Revision 与 Scheduled Run 产物交付；
不改变 WorkflowRun 的会话/成果契约、交付窗口长度、失败暂停或隐藏 Workspace 的运维权限。

## 问题与约束

交付工作已经在 Cloud 登记，但 Node 可能尚未受理。绝对截止能够结束业务等待，却不能生成缺失的
执行事实。若 Controller 永久轮询原执行，删除的 quiesce 会持续发现未完成责任；若直接过滤 pending，
则在途派发或丢失回报的真实执行可能仍在写文件。

必须把两个事实分开：Cloud 不再等待这次交付；原执行是否已经受理及能否继续产生副作用。
登记响应丢失、Node 查无、断线、超时和 sequence=0 均不证明后一事实。

## 继承与修改

| 前序决策 | 本文关系 |
| --- | --- |
| [IssueRun 根 D4/D5/D8](0-agent-run-in-disposable-isolated-workspace.md) | 继承业务结果、成果保存和物理释放的分离及既有窗口；新增原交付执行的独立业务结算，不重写 Node result。 |
| [未登记撤回 D1](20261012-bounded-agent-startup-and-undispatched-withdrawal.md) | 继承登记与撤回互斥；本文处理已经登记的执行，不能套用“没有登记”的证明。 |
| [verdict/产物 D3](../scheduled-task/20261022-run-verdict-and-artifacts.md) | 继承运行级不可续期截止、截止后只接回执不登记对象；新增截止事务封闭派发及独立执行结算。 |
| [迟到事实 D1–D3](20261022-late-delivery-fact-takeover.md) | 继承原身份事实接管、receipt-only、后创建 Run 守卫及真实接管后 ACK；行政结算不关闭该入口。 |
| [历史 D8 两阶段处置](20261025-d8-residual-without-delete-operation.md) | 继承真实终止/围栏、终止后保留数据、独立确认后删除；本文不新增物理回收授权。 |
| [任务串行交接](20261018-serialize-task-agent-handoffs.md) | 继承失败暂停和显式恢复，不以新结算自动丢弃未保存成果。 |

## 决策

### D1：业务结算与封闭派发在同一 Cloud 事务提交

Cloud 在以下既有业务边界结算原执行：产物绝对截止；Revision 已按 D5 放弃；兼容迁移确认原 Run
已经结算但仍有未知交付执行。结算来源和时间使用独立持久字段，不能写入 Node 的 result、event
sequence、receipt 或 Revision。真实终态已经存在时保留它，不补一条伪造终态。

截止判断采用取得权威锁后的数据库时间。原执行结算、原 work 的新登记拒绝、原执行的新 permit
拒绝与业务截止结论原子生效。旧 claim 和已缓存输入不能绕过重新检查；提交失败则全部回滚。
产物截止只封闭产物交付，仍在窗口内的 Revision 不受影响；反向亦然。

已经发出的 permit 或已经受理的执行可能仍有外部责任。D1 不撤销其已发生的事实，也不作为 idle、
quiesce、terminated 或数据可删的证据。未登记 work 留存用于审计，但不再可领取/登记；不能让它
反复占据领取队头，也不能用新的 execution 身份重新试探原结果。

### D2：Node 原子检查受理 ledger，并永久封闭这个 execution 的输入

Controller 只对 D1 已结算、仍无真实终态的交付执行请求未受理证明。Cloud 提供精确原身份；请求
至少绑定 tenant、Workspace、sandbox/generation、Node/instance/incarnation、execution、work、
Cloud operation 和 Node operation，以及冻结的执行输入。Controller 不能自行换绑实例或代次。

Node 在与受理互斥的持久事务中检查完整执行身份 ledger，包含已完成和未完成记录。只有该 execution
从未受理，才能写入不可撤销的 execution 输入围栏与对应证明。围栏和证明一并提交后才答复。
同一 execution 的迟到旧 permit、重连和新控制 epoch 均不能越过围栏；不同输入或 operation 复用
该身份必须拒绝。围栏是特定 execution 的输入关闭，不取消同一 Workspace 其他仍有效的交付。

已有 Accepted/Running/Completed、恢复未知责任或无法完整读取 ledger 时不产生未受理证明。
真实执行继续由原事件/查询链对账。受理先提交，证明必须拒绝；证明先提交，受理必须拒绝，不能
二者都成功。持久化失败不能回复成功；磁盘升级、重启与控制接管均保留围栏。

Node 身份发生变化时不能用新实例的一张空表证明旧实例没有受理。已经持久化的原证明可按原身份
重放；不能取得原 ledger 或原身份证据的执行继续保留未知责任，转既有 D8 处置。

### D3：Cloud 接管专门的未受理事实，Controller 据此解除对应责任

Cloud 在当前 Controller 服务身份和租约下，逐项核对原冻结身份、D1 结算及 Node 证明，单独持久
保存未受理事实。此事实不是 Node 的 Completed/Failed 事件，不占用其事件 sequence，不产生普通
event ACK、artifact 登记或 Revision；不同证明冲突时拒绝，不覆盖原事实。

只在该证明真实提交后，Controller 才把对应 execution 从未完成集合中移除，并清除它自身的
unresolved 观察。quiesce 仍关闭 Workspace 的新派发、核对其余所有 execution 与执行票据，保持
当前 admission/控制代次围栏。仅有 D1 结算、一次查无、超时、旧缓存或证明上报失败都不能跳过。

如果原 Node 永久不可达，业务仍可按 D1 有界结束，未知执行仍承担物理对账责任。真实终止加
late-ensure 围栏可按既有受控处置结算物理责任；数据保留/删除的两阶段确认不被本文合并。

### D4：迟到真实事实和原失败历史分别保留

D1 后真实迟到终态仍按原 execution/input/receipt 接管。不得因行政结算直接 ACK、丢弃终态、改变
Run 的旧 deliveryState、补写 Revision 或延长对象登记窗口。旧 Revision 的首次迟到登记继续受
同 Issue 后创建 Run 守卫约束。无真实接管成功不得 ACK。

未受理证明与真实受理/终态矛盾时保持可审计冲突，不能静默覆盖任一事实或把矛盾当作 idle。
已经完成的普通回执幂等规则保持；未受理证明的重放只确认原证明，不创造 Node 终态事件。

### D5：显式迁移存量，混合版本保守等待

Cloud 只追加迁移。存量产物执行按持久产物截止/结算证据补独立来源；其他已结算 Run 的未知交付
按“存量已结算首次观测”记录迁移时刻，不能假称它是原截止。旧 result、sequence、receipt、Run
结论和 Revision 均不回写。未来登记时冻结原实例与执行输入身份；存量缺少可靠原实例证据时不
由当前实例推算 B 的资格，保留未知责任并走真实终止路径。

Node 增加显式 schema 升级；旧程序面对新围栏 schema 必须拒绝启动，不能降级后忽略围栏。协议
通过新 capability 协商；旧 Node 不接新请求，旧 Controller 不会把新结算当执行终态。Cloud 的
新字段不自动混入已有封闭响应，新增 wire 字段需在拥有者契约中显式定义并生成。

部署按已批准规格、数据库迁移、Cloud、Controller/Node 共同验证推进。混合版本尚无 B 能力时只
允许保守保留未知责任，不能通过兼容 fallback 宣告 idle。只有实现评审/合并和部署版本均明确后，
才对保留现场做获准恢复；部署、终止、数据删除继续分别受既有授权约束。

## 不变量

1. 行政结算和执行事实是两份独立持久事实，彼此不能伪造。
2. 截止事务与新登记/permit 的竞争只有一方获得资格；旧 permit 不证明没有在途副作用。
3. Node 的未受理证明和迟到受理不能同时提交成功，持久围栏跨重启和 epoch 生效。
4. 任意原身份不匹配、证据不足或持久化失败都保留未知责任，不产生成功证明或 ACK。
5. 业务截止本身不授权 quiesce、物理终止或数据删除。
6. 迟到真实事实不篡改已结算 Run，也不跨越同 Issue 后创建 Run 覆盖成果。
7. 产物与 Revision 的截止、结果、授权和持久化继续分别拥有。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 截止时给 result 写 failed | 能退出 pending，但伪造执行端终态，丢失已受理未知责任。 |
| 查询为 Unknown 就删除 pending | 简单，却不能排除丢失回报、迟到派发或新实例空 ledger。 |
| 所有业务等待都等 B | 证据直接，但原 Node 永久失联时无法有界结束业务。 |
| 只有 A，没有 Node 输入围栏 | 业务有界，但旧有效 permit 仍可能迟到受理，不能证明 quiesce 安全。 |
| 为每次重试复制 execution | 增加执行机会，却破坏原身份对账并可能重复外部副作用。 |
| 产物到期关闭整个 Workspace 输入 | 围栏简单，但会中断尚有效的 Revision；使用 execution 围栏并保留全局 quiesce。 |

## 验收与落地

规格获批后建立核心用例，分别检验：从未派发、登记响应未知、已受理但回报丢失、截止/登记与
受理/围栏竞争、旧 permit、错绑实例/代次/输入、重启/接管、存量升级、持久化失败、迟到终态和
quiesce。单个低层状态断言不能代替跨进程义务。

产品实现完成评审后，独立 specs 落地 PR 按最终实现同步并改为 implemented；先合并产品再合并
落地 PR。原任务恢复随后在原任务内验证真实 Claude 执行、两轮 Revision、最终字节/SHA-256、
后续消息及必要物理释放，不能用副本任务或 CI 替代。
