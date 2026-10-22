# 迟到交付事实、续作顺序与释放恢复

依据[迟到交付事实接管决策](../../../decisions/cloud/issue-run/20261022-late-delivery-fact-takeover.md)
（下称“迟到决策”）及[Controller 终态拒绝决策](../../../decisions/controller/session/20261022-isolate-refused-delivery-terminals.md)。
本文件拥有根 D5 无账本接收、后创建运行守卫以及接管后恢复释放的新增义务；原有
[运行编排](agent-run-orchestration.md)、[续作](resume-from-latest-revision.md)和
[残留回收](residual-resource-reclamation.md)的证据不自动覆盖它们。新增义务都为 **Missing**，
由实现与落地运行填入已合并版本的直接证据；本 ADR 运行不把未来测试名或在审代码列成已完成证据。

## Root D5 Takes Over An Original Late Success Without A Residual Ledger

- 风险：Run 已放弃交付但仍拒绝原终态，Node 不能清除 outbox，或接收事实时重新结算业务。
- 前置状态：原 delivery 已经生产路径登记，Node 形成成功终态但 Cloud 尚无结果和 receipt；生产 D5
  放弃后 Run 为 `releasing`、`deliveryState=failed`、`revisionId=null`，Workspace deleting、admission
  关闭；无 residual 行、无运维意图，同 Issue 没有后创建 Run。
- 触发：原 Node 恢复，按原 execution/work/resource 提交原有序事件；分别覆盖 delivered 与 unchanged。
- 必须成立：按原证据规则接收，证据有效时恰好登记原 Revision；结果、原始 receipt、去重响应原子提交后
  才返回成功。接管事务前后 Run 全部字段、业务活动、admission 不变；不新建执行、资源、delete 或回收责任。
- 禁止结果：要求先建账本；把 Run 改成 saved、写回 revisionId；绕过原身份或通过扩大存活交付权限接收。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 无账本的生产 D5 路径确实能接收 | PostgreSQL/gRPC 连续驱动原 dispatch、接收前断线、生产 give-up 和原事件恢复；不得先插 residual 行代替 | Missing |
| 接管与业务独立 | 同一次接管前后完整 Run/业务活动及原资源身份对比；仅原 execution、receipt 和合格 Revision 变化 | Missing |
| 无效或暂不可达证据不伪造 Revision | 原声明校验/证据负例，无错误的 Revision 或成功业务结论；暂失败不产生接管成功 | Missing |

- 决策依据：迟到决策 D1–D3、不变量 1、2、5。

## A Later Created Run Prevents Late Revision Registration On Every Intake Path

- 风险：旧 Run 的迟到 Revision 因登记时间更晚成为最新候选，下一运行越过已发生的后续工作。
- 前置状态：同 Issue 的 R 后已创建 N；创建事件 `seq(N)>seq(R)`，故意让时间戳相同或与 UUID 排序
  相反；N 分别处于 queued、执行中、终态、软删除，且可尚无 Revision。另设不同 Issue/租户的运行。
  接收路径包括根 D5 无账本、根 D8 done 无意图、残留账本 reconciling/reclaiming/cleared 和已显式接管的 D8。
- 触发：对 R 首次提交有效迟到成功终态；另一组让 N 在事务外证据读取之后、最终登记之前创建。
- 必须成立：所有路径都保存原声明与精确 receipt 并返回接管成功，但不登记 R 的 Revision；同 Issue
  任一后创建 Run 都阻止登记，不按 Agent、status、Project 或是否已有 Revision 缩小条件。不同 Issue/
  租户不能错误阻挡。最终事务与创建互斥，两种提交顺序各自符合迟到决策 D2；顺序证据不充分时，
  原声明与 receipt 存在、Revision 不存在。
- 禁止结果：按 revisions.created_at 或随机 UUID 猜创建先后；只保护已有成功 Revision 的 N；残留路径
  绕过守卫；先返回 ACK 再尝试补登记；把 receipt-only 当成 verified delivery 或额外清理授权。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 所有接收路径使用同一后创建条件 | 真实创建事件和各接收入口的 PG/gRPC 测试，精确断言结果/receipt 存在、Revision 不存在、业务及责任不变 | Missing |
| 登记与新 Run 创建竞争 | 两个真实事务控制两种提交顺序，含外部读取等待，不以最终偶然结果代替交错证据 | Missing |
| 后续续作不回到迟到旧成果 | N 已有较新成果时接 R，再启动下一 Run，冻结的 priorRevision 仍为既有最新合格成果 | Missing |

- 决策依据：迟到决策 D2、不变量 3、4；Git 交付决策 D7 的既有候选规则。

## A Late Failure Is Receipted Without Retrying Or Resettling Delivery

- 风险：迟到失败结果再次生成交付重试，或未记收据使 Node 永久重放。
- 前置状态：原已登记 execution 终态未接收，D5 releasing 或 D8 done，原输入与资源仍可核验。
- 触发：提交原 `revision_failed` 有序事件。
- 必须成立：原失败声明及 receipt 同事务提交，可正常 ACK；无新 Revision、work、dispatch 或交付重试；
  Run 的结算字段和原失败 delete 保持不变。不同合法失败原因不能被 Cloud 改写成伪造成功。
- 禁止结果：因 Run 已终态拒收原事实；把迟到失败作为再次结算或重新尝试交付的触发。
- 验证义务与证据：**Missing** — 需根 D5/D8 的 PG/gRPC 直接证据，比较接管前后业务及执行计数，并断言
  原失败 result、事件收据与成功答复的提交边界。
- 决策依据：迟到决策 D1、D3、不变量 2、5。

## Exact Replays Preserve The Original Decision And Conflicts Preserve Canonical Facts

- 风险：重放变成第二次登记或结算；receipt-only 在稍后重放中被升级；不同事件覆盖已接管原事实。
- 前置状态：分别已有正常登记与 receipt-only 两种迟到成功接管，以及一个迟到失败接管。
- 触发：同 submission 重放；换 submission 的同 execution/result/sequence/event；在首次接管后创建 N 再
  重放；同 execution 不同 result、同 sequence 不同 event、序号缺口、错误原资源和 stale lease。
- 必须成立：相同事实返回原接管结论，既有 Revision 不重复或删除，receipt-only 不补登记，不再次结算；
  真实冲突仍拒绝，无可用于 ACK 的成功答复，canonical 结果/receipt 不变；有账本时仍保留其冲突审计。
- 禁止结果：用业务退休理由吞掉身份/结果/收据冲突；对重放重新申请执行权；为冲突伪造 acceptance。
- 验证义务与证据：**Missing** — 需真实数据库重放及冲突矩阵，覆盖 root 与 residual 入口、当前服务资格，
  断言 canonical 事实、原业务行和副作用计数；新增规则的证据由落地运行补充。
- 决策依据：迟到决策 D1–D3、不变量 4–6。

## A Takeover Success Receipt Precedes Ack Without Changing The Business Outcome

- 风险：内存接收、Completed 查询或单独审计被误当接管成功，Node 过早丢弃唯一终态。
- 前置状态：Node outbox 保留原 sequence/event；原业务交付已经失败；分别设置可登记和 receipt-only 分支。
- 触发：阻塞/失败原结果、receipt、去重响应或必要证据提交；返回 Completed 查询；随后解除故障重交原事件。
- 必须成立：提交前无成功回执、无 EventAck、Node 保留事件；完整提交后现有 takeover OK 才允许 ACK 和
  outbox 排空，两分支都成立。业务 deliveryState/revisionId 不因 ACK 改变，回复丢失可以按原事件恢复。
- 禁止结果：收到 Conflict 也 ACK；以 GetDispatch.result 或 Completed 代替本次事件 receipt；新增 disposition
  以规避原提交要求；宣称 ACK 表示 Run 的业务交付改成成功。
- 验证义务与证据：**Missing** — 需 PG 原子失败注入及真实 Controller/Node 接管与 outbox 证据，单仓 fake
  只能覆盖对应边界，不能替代跨进程 ACK 顺序。
- 决策依据：迟到决策 D3、不变量 5、6；Controller 终态拒绝决策。

## Reconnecting With A Late Terminal Completes The Original Release And Frees Resources

- 风险：原终态的 dispatch_conflict 导致连续重建连接，Run 永久 releasing，container/volume 未释放，
  后续消息永久 waiting；仅证明 Node 短暂 connected 却遗漏资源释放。
- 前置状态：真实 Cloud/Controller/Node/Substrate 链路，原 delivery 已登记并产生未接管终态；接收前断线。
  可将测试交付放弃窗口配置为 90 秒以稳定触发，生产默认和 D8 条件不因此改变。根 D5 后 Run releasing、
  Workspace desired=deleted/observed=deleting，无 residual/运维意图；原 delete 尚未被 D8 终结失败。
- 触发：在该状态恢复原 Node 网络，重放原终态，继续生产释放流程；另有 execution 及后续 waiting 消息。
- 必须成立：Node 完成握手并保持连接，原事件获得真实接管回执后 ACK、outbox 排空；旧终态不再制造
  整会话循环，另一 execution 正常。随后同一原释放路径取得真实 quiesce、sandbox 终止和数据删除证据，
  delete succeeded、Run 到 done，原 container 与 volume 确认释放。waiting 消息不再因这一永久释放阻塞而
  悬挂，但交付失败引起的暂停/用户恢复授权仍遵循串行交接决策，不能为了用例通过自动越过。
- 禁止结果：只断言 Node connected 或 Run done，不核对物理资源；放宽 D8 把反复重连视作释放成功；
  手工写 done/插下游结果；偷偷先强停或新建回收意图；接管本身重写 Run 的历史结算。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 原故障时序后的真实释放 | 一次连续真实栈运行，保留同一 Run/Workspace/sandbox/Node/execution 和 receipt 身份，从接收前断线经 D5、重连、接管到 quiesce/terminate/data delete/done；并有 container/volume 不存在的物理证据 | Missing |
| 连接与执行隔离 | 同一运行中的连接/reports、ACK/outbox 与另一 execution 完成证据；仅 fake Cloud 单测不算整链已覆盖 | Missing |
| 后续输入保留授权边界 | 释放阻塞解除后等待原因可核对；需要显式恢复的输入保持该要求，不以自动交接掩盖失败暂停 | Missing |

- 决策依据：迟到决策 D1、D3、不变量 1、2、5；IssueRun 根 D5/D8；串行交接 D1–D3。

## A Root D8 Late Receipt Does Not Restart The Historical Failed Delete

- 风险：把 #312 的正常释放要求错误套到已经 D8 done 的历史残留，复活失败 delete 或伪造资源已释放。
- 前置状态：生产 D8 已令 Run done、原 delete failed/node_unavailable，无 ledger、无运维意图；原终态未接管。
- 触发：原 Node 恢复并提交原事件，随后既有释放扫描运行。
- 必须成立：按迟到决策接收声明和 receipt，Revision 仍服从创建顺序守卫；原 Run 全行、failed delete
  及结算活动不变，admission 不重开，无新的回收责任或自动重启删除。后续真实资源处置只沿既有显式授权。
- 禁止结果：把历史 done 改回 releasing、原 failed delete 改成功，或因为 ACK 就声称资源释放。
- 验证义务与证据：**Missing** — 需真实 D8 生产路径加迟到 gRPC 接收与随后扫描，断言原行及责任不变；
  不以预插 residual 行替代无账本场景。
- 决策依据：迟到决策 D1–D3；残留回收 D6。
