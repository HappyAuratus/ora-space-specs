---
status: implemented
---

# 放弃交付后仍接管原执行事实；迟到 Revision 不得越过同 Issue 后创建的运行

Cloud 已按交付上限放弃一个 Run 后，仍按原 execution、work 和资源身份接收它的迟到交付终态，
不要求先有残留账本或运维接管意图。接管成功允许确认原事件，不改变已结算的业务结果。只有同一 Issue
尚无后创建的 Run，且原交付证据验证成立，才登记迟到的原 Revision；否则仅保存声明与事件收据。

本文为 `implemented`：最终规则依据 [owner 决定](https://github.com/ora-space/circus/issues/407#issuecomment-6335015004)，
实现及证据固定于 [Cloud #142 head `818226b4`](https://github.com/ora-space/cloud/tree/818226b49b8a97779f41d6191e69f6fc8518b0fb)
与 [Desktop #726 head `13de5a35`](https://github.com/ora-space/desktop/tree/13de5a3567d0f2166a4ab27d0f5c7fa05626bc9d)。
不新增协议状态码或数据库结构，不改变 D5/D8 窗口、残留回收授权、普通交付资格或同任务交接的暂停与恢复授权。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [同任务串行交接](20261018-serialize-task-agent-handoffs.md) D1–D3 | 继承，不变；接管迟到事实不自行恢复 waiting 消息，不越过失败暂停或释放确认。 |
| [IssueRun 根决策](0-agent-run-in-disposable-isolated-workspace.md) D4/D5/D8 | 继承业务结论、放弃窗口和失败删除历史；修改放弃后的原交付事实入口，使其不再依赖存活的交付资格（D1）。 |
| [残留回收](20261018-residual-agent-run-reclamation.md) D2/D6 | 继承原身份事实接管、同事实幂等和回收保护；修改该 ADR D2 的成果救援：同 Issue 已有后创建 Run 时不登记迟到 Revision，具体条件见本文 D2；无账本接收不创建回收责任。 |
| [IssueRun 续接](20261010-resume-run-from-latest-issue-revision.md) D1 与 [Git 交付及续作](../revision/20261017-git-native-delivery-refs-and-resume.md) D7 | 继承已登记 Revision 的候选条件、按登记时间选择和启动时固定；修改迟到 Revision 的登记资格，防止旧运行取得新的登记时间后覆盖后续成果（D2）。 |
| [执行接管与收据](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md) D2/D6 及 [内部 gRPC](../controller-integration/0-cloud-owned-internal-grpc-contract.md) | 继承原事件收据、原子提交、重放及错误分类；现有 takeover 成功只确认事件事实接管，不重新宣告业务成功（D3）。 |

## 背景

Node 已形成交付终态但 Cloud 尚未接管时连接中断，Cloud 可能先按根 D5 放弃交付并开始删除 Workspace。
原终态重放若被当作新的存活交付而拒绝，又导致 Controller 拆掉整个连接，Node 就无法稳定参与 quiesce。
运行可长期停在 `releasing`；反复重连也不能被假定为满足根 D8 的“状态未知”条件。只增加超时或放宽
普通交付资格，均不能解决这个事实接管与连接存续问题。

此外，续作候选按 Revision 登记时间选取。若旧运行的迟到成果在同 Issue 较新运行之后登记，下一次
续作可能选回旧成果。因此接收原事实与允许新登记 Revision 必须分别判断。

## 决策

### D1：根 D5/D8 后无账本也接管原交付，不重新结算

- 原 Agent IssueRun 已因根 D5 以 `deliveryState=failed`、`revisionId=null` 进入 `releasing`，或随后
  因根 D8 进入 `done`，都允许接管原 `deliver_revision` 的迟到终态；Run 的 `status` 不要求是 failed。
  没有 `agent_run_residuals` 行、没有运维接管意图，不构成拒收理由。
- 原 Run 软删除只影响业务可见性，不撤销仍可核验的原事实身份；其迟到事实仍按 D1 接收，有效证据且
  同 Issue 没有后创建 Run 时仍可按 D2 登记原 Revision，不因软删除额外降为 receipt-only。
- 必须核验已登记 execution → 原 work → 原 Run/Workspace，以及原租户、Project、Node、sandbox 与资源
  generation 的关联；声明仍符合原冻结输入，事件序号及原内容满足既有收据规则。只关闭 admission、
  任意取消、换 Node、未登记执行或身份不符，不能借此取得接管资格。普通 `liveDelivery` 资格仍保护
  存活交付与新派发，不能被整体删除或放宽。
- 接收资格取决于上述 Run 与完整原身份，不要求 Workspace `desiredState=deleted`，也不依赖当下
  observed state、admission 或某一次 delete attempt。失败 quiesce 会恢复 `running/ready` 并重开 admission；
  原 Run 仍已放弃交付，原终态必须能接管，随后由独立释放重试到达 `done`，避免“拒绝接管 → quiesce busy
  → 恢复 Workspace → 再拒绝接管”的循环。同稳定 Node、sandbox 与资源 generation 下的新 incarnation
  不等于换资源；推进 Workspace 资源 generation 则仍受原身份围栏拒绝。
- 接收原声明、精确事件收据和必要的后续责任在同一权威事务提交。迟到失败结果同样接管，不因它是
  `revision_failed` 再放出交付重试。成功声明的证据和 Revision 登记按 D2 分别处理。
- 接管事务不改变 Run 的 phase、status、result、failure_reason 或业务版本，不重写结算活动，不重新
  打开 admission，不创建执行、ensure、资源或回收责任，也不改原已失败的 delete operation。
- 这不阻止另一个依据真实证据的释放事务继续原 owned delete：仍在 `releasing` 的 Run，在 Node 正常
  参与 quiesce、沙盒终止与数据删除确认后，应按既有流程到达 `done`。接管时业务行不变，不等于禁止
  后续正常释放。根 D8 已 done 的历史失败删除则不会因重连自动重启，仍遵循残留回收 D6。

### D2：迟到 Revision 只在同 Issue 没有后创建运行时登记，残留路径同样适用

原交付的身份、声明和证据规则不降级。只有证据有效且满足下述创建顺序条件，才登记原 execution
的 Revision；Run 仍保持 `deliveryState=failed`、`revisionId=null`。存在后创建运行时，只持久接收
原声明与收据，不登记 Revision，也不把这份声明当作已验证的 Revision 或已完成交付证据。

**创建顺序的精确定义：**同租户、同 Issue 的每个 Run，以创建 Run 的同一事务写入的
`run.enqueued` 活动作为创建记录；活动的 `details.runId` 对应该 Run，`seq` 是该 Issue 共享 Timeline
中的严格递增序号。创建事件的 seq 不必连续，评论与其他活动可以占据中间序号。对原 Run `R`，
如果存在同租户同 Issue 的另一个 Run `N`，且 `N` 的创建事件 `seq > R` 的创建事件 `seq`，
则禁止为 R 首次登记迟到 Revision。N 的执行者、状态、阶段、是否已软删除、是否已有 Revision
均不改变这个判断；后创建的 workflow executor 同样阻止登记，不能只检查后续成功运行、Agent Run 或同 Project 的运行。

这使用已有创建事件的提交顺序，不用 `revisions.created_at` 判断 Run 的创建先后，也不用
`issue_runs.created_at` 与随机 UUID 推测真实先后。创建事件只作为顺序证据，不从 Timeline 重建
Run 的业务状态。只有能唯一定位原 Run 及同 Issue 其他 Run 的创建记录、并确认不存在后创建运行，
才满足登记资格；顺序证据缺失或有歧义时按 receipt-only 处理：保存原声明与收据，不登记 Revision。
新创建的 Run 在同一事务写入 `run.enqueued`。长期存量数据可能缺少这些创建证据；此时 receipt-only
是持续适用的保守结果，不推断或补造顺序，不用时间戳/UUID 猜测，也不通过重放升级登记。不需要新增序号列或迁移。

判断在最终登记事务中进行，与同 Issue 的 Run 创建互斥：若新 Run 先提交，迟到接收只能保存声明和
收据；若迟到 Revision 先提交，新 Run 后创建，则按既有续作规则选取已提交候选。事务外证据读取前
看到“没有后创建运行”，不能替代最终事务的判断。

本规则也约束残留回收 D2 的成果救援：有 ledger 的 `reconciling`、`reclaiming`、`cleared` 路径，
以及 D8 已显式受控接管的原身份路径，都不能绕过限制。它们同样可能登记一个时间更晚的旧 Revision，
因此没有豁免理由。同 Issue 没有后创建 Run 时，残留成果救援的行为不变。账本仍按既有规则追加真实证据与冲突审计；receipt-only 不伪造 verified delivery，
不自动产生清理授权，不改变回收所需的完整证据或明确放弃依据。

限制只针对首次迟到登记，不回溯删除已经提交的 Revision，不改变已有候选排序或已经冻结的
`priorRevision`。首次已按 receipt-only 接管的原事件，幂等重放也不能被提升成一次新的登记机会。

### D3：ACK 只表示原事件事实已持久接管

- 复用现有 `TakeOverNodeEvent` 成功响应，严格绑定本次原 execution、sequence 和原事件。允许
  ACK 的前提是原声明、精确 receipt 与必要证据、去重和可恢复责任已一起提交；只收到请求、写一条
  日志、发现 Run 已终态或查询到 Completed 都不够。
- ACK 表示“这个原事件的事实已经由 Cloud 接管”，不表示 Run 再次业务成功。receipt-only 分支同样
  可以得到真实成功回执；没有 Revision 不等于事实未接收。
- 相同 submission/原事件重放返回原接管结论，不重复登记 Revision、结算或产生副作用；换 submission
  的同 execution/result/sequence/event 仍是同一事实。不同结果、同序号不同事件及序号缺口继续按
  既有冲突规则拒绝，不覆盖原事实、不发可用于 ACK 的成功答复。已有 ledger 的冲突仍按原规则留审计。
- 任何持久化失败都不能产生成功回执；暂不可达不伪装为已保存。不新增 disposition、错误码或
  “失败响应也可以 ACK”的旁路。Controller 对精确终态拒绝的局部隔离由
  [终态拒绝决策](../../controller/session/20261022-isolate-refused-delivery-terminals.md)规定。

## 不变量

1. 原执行的迟到事实接收不依赖新建残留回收责任，也不授予新的执行或资源权限。
2. 接管事务保留 Run 已结算字段和原失败删除；正常释放在独立证据事务中继续推进。
3. 同 Issue 已有后创建 Run 时，旧运行的迟到事实不会产生一个新的 Revision 候选。
4. 根 D5/D8 与残留救援对同一创建顺序限制没有不同解释。
5. 只有本次原事件真实持久接管成功才可 ACK；成功 ACK 不重写业务结论。
6. 重放不新增业务转换，冲突不覆盖 canonical 事实，Completed 查询不清除未解决终态。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 为接收迟到结果先创建 residual 行或运维意图 | 复用入口方便，但混淆事实接收与回收授权，也覆盖不了普通 D5 删除中的重连。 |
| 删除 `liveDelivery` 限制 | 可以接到结果，却同时放过无效派发、换代和普通身份冲突。 |
| 任何已验证迟到成果都登记 Revision | 保留成果更多，但登记时间可能使旧运行覆盖同 Issue 后续工作。 |
| 用创建时间和随机 UUID 当真实创建顺序 | 无需新列，但事务时间和 UUID 不能证明创建先后；已有原子创建事件 seq 可以明确表达条件。 |
| 改成选择所有 Run 中创建最晚的 Revision | 改变整个续作候选语义；本次只限制迟到登记，不重排已有历史。 |
| 新建终态 disposition，或收到 Conflict 就 ACK | 前者扩大协议，后者伪造接收；现有成功回执已经能明确表达事实接管。 |

## 与 approved 文本的差异与澄清

| 项目 | 最终规则与原因 |
| --- | --- |
| Workspace 删除形态 | D1 明确没有 `desiredState=deleted` 前提。真实 quiesce 失败会恢复 running/ready；把动态删除形态当接管资格会使未确认终态与释放互相阻塞。完整原身份和 generation 围栏保留。 |
| 原 Run 软删除 | D1/D2 不增加可见性过滤。软删除不改变历史事实的归属，既不能拒收，也不能独立剥夺无后创建 Run 时的 Revision 登记资格。 |
| 旧数据缺创建事件 | 删除“仅作防御性处理”的假设。顺序证据缺失/歧义的历史数据长期保持 receipt-only；当前写入惯例不能证明旧数据齐全。 |
| 后创建运行的执行者 | D2 的“任一 Run”包括 workflow executor；创建顺序属于 Issue，不能只枚举 Agent Run。该分支的直接产品测试仍缺失，见核心用例。 |

以上说明最终规则及其原因，不扩大普通交付或回收授权；协议和数据结构保持既有契约。

## 验证与保留边界

直接测试与逐项状态见[迟到交付事实与续作保护](../../../test-cases/cloud/issue-run/late-delivery-fact-takeover.md)。
[固定版本真实栈回执](https://github.com/ora-space/circus/issues/407#issuecomment-6341728665)证明 Git Server
阻塞变体在恢复后完成迟到接管、正常释放到 `done` 和物理资源回收；未进入 O4 拒绝隔离窗口。
[后续 O4 真栈复测](https://github.com/ora-space/circus/issues/516#issuecomment-6345535566)补齐旧 Cloud 拒绝后的隔离窗口，以及换候选 Cloud 并重启 Cloud 引起 Controller↔Node 会话重建后重交、正常释放的连续证据；不要求 Controller 重启。真实新 incarnation 的证据缺口仍保留。workflow 后创建 Run、顺序竞争双向交错等未完整覆盖的义务也在核心用例中保留
`Missing` / `Partial`，不由 ADR 状态自动提升。
