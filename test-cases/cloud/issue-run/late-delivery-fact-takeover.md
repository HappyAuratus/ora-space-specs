# 迟到交付事实、续作顺序与释放恢复

依据[迟到交付事实接管决策](../../../decisions/cloud/issue-run/20261022-late-delivery-fact-takeover.md)
（下称“迟到决策”）及[Controller 终态拒绝决策](../../../decisions/controller/session/20261022-isolate-refused-delivery-terminals.md)。
本文件拥有根 D5 无账本接收、后创建运行守卫以及接管后恢复释放的新增义务；原有
[运行编排](agent-run-orchestration.md)、[续作](resume-from-latest-revision.md)和
[残留回收](residual-resource-reclamation.md)的证据不自动覆盖它们。

证据固定为 Cloud PR #142 merge `b777496` 与 Desktop PR #726 merge
`2564b42`。下表链接固定合并提交的测试文件，并列出真实函数名。
Cloud 测试使用 PostgreSQL 与 HTTP/gRPC，起始 Run/Workspace 为存储夹具，部分 Controller/Node/Substrate 由 simulator
扮演；Desktop 测试使用实际 Controller、WebSocket fake Node 与 fake Cloud。单仓 `Covered` 只覆盖对应直接断言，
不表示跨仓真实链路完整覆盖。

源码与测试引用使用已合并的 Cloud `b777496` / Desktop `2564b42`。此前真栈复跑仍实际使用 PR head `818226b4` / `13de5a35`；[owner 合并说明](https://github.com/ora-space/circus/issues/407#issuecomment-6354830227)确认落后 main 期间的新增提交未触及相关交付、释放及 Controller 会话路径，无语义漂移。该说明不代替合并后 main 的复跑证据。

真实栈证据来自 `wanglongan587` 的[固定版本运行回执][real-stack]及[固定版本评审][review]。
作者机器产物为 `ora-space/cluster/.acceptance/407/407-late-terminal-818226b4.json` 与
`407-stop-818226b4.json`；本文件引用评论及产物名，不声称读取过这些本地文件。O4 拒绝隔离窗口已由[后续真栈复测][o4-stack]补齐；真实新 incarnation 仍缺外部证据。

[O4 真栈复测](https://github.com/ora-space/circus/issues/516#issuecomment-6345535566)：desktop `13de5a3567d0` + 旧 Cloud main `7a48f3403ae3`，精确拒绝告警仅一次（`Aborted` / `ERROR_CODE_CONFLICT` / `dispatch_conflict`、`unresolved/no_ack/no_redispatch`）；60 秒无 receipt、result=null、仅一个 deliver_revision，Node 同 incarnation 保持 connected、无重新注册，仅 heartbeat 前进。换 Cloud `818226b49b8a` 并只重启 Cloud 后，Cloud 不可用触发 Controller↔Node 会话重建，原终态在 14.6 秒内接收（receipt seq=1，`push_rejected` / `revision_failed`）；同一 delete 经 TERMINATE→CLEANUP→DONE，Run done、deliveryState=failed、revisionId=null、0 revisions，Workspace deleted、container/volume 均回收。随后才重启 Controller，状态不变。

O4 证据目前仅位于作者机器的 `ora-space/cluster/.acceptance/407/407-o4-isolation.json`，未提交，尚无已提供的长期访问位置；作者给出的 SHA-256 为 `7cb8f80b44a489dc09a962359b42713b1c068e0aba417449dce8a1298387079e`，此处原样引用，未在本机读取文件或重算摘要。边界：echo agent（受控模型），90s give-up 为测试 override；上游仓库绑定/快照由 `git:preset snapshot --synthetic 3` 预置，该验收工具直接写仓库卷及数据库，只代替上游同步；交付接管与释放使用真实组件。失败终态不覆盖 O2 迟到成功 Revision 登记或后创建 Run receipt-only 的真栈分支，它们仍引用 Cloud 集成测试。

## Root D5 Takes Over An Original Late Success Without A Residual Ledger

- 风险：Run 已放弃交付但仍拒绝原终态，Node 不能清除 outbox，或接收事实时重新结算业务。
- 前置状态：原 delivery 已经生产路径登记，Node 形成成功终态但 Cloud 尚无结果和 receipt；生产 D5
  放弃后 Run 为 `releasing`、`deliveryState=failed`、`revisionId=null`；Workspace 可以 deleting/admission closed，
  也可因失败 quiesce 恢复 running/ready、admission open，不以 desired=deleted 为资格；无 residual 行、无运维意图，
  同 Issue 没有后创建 Run，原 Run 可软删除。
- 触发：原 Node 恢复，按原 execution/work/resource 提交原有序事件；分别覆盖 delivered 与 unchanged。
- 必须成立：按原证据规则接收，证据有效时恰好登记原 Revision；结果、原始 receipt、去重响应原子提交后
  才返回成功。接管事务前后 Run 全部字段、业务活动、admission 不变；不新建执行、资源、delete 或回收责任。
- 原 Run 软删除不撤销 D1/D2 资格；有效证据且无后创建 Run 时仍登记 Revision。同 Node、sandbox、资源 generation
  下的新 incarnation 不等于换资源；新资源 generation 仍拒绝。
- 禁止结果：要求先建账本；把 Run 改成 saved、写回 revisionId；绕过原身份或通过扩大存活交付权限接收。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 无账本生产 D5 原成功接收 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactRootD5AndD8OverGRPC`：生产登记、真实 RPC 接收前取消、生产 give-up、delivered/unchanged、无账本/意图断言 | Covered |
| 接管与业务独立 | 同文件 `TestLateDeliveryFactRootD5AndD8OverGRPC` 完整 Run/Workspace/operations/活动/责任快照，禁止结算 hook，含重放 | Covered |
| 失败 quiesce 恢复后仍可接收 | [integration/late_delivery_release_retry_test.go][cloud-release] `TestLateDeliveryAfterRefusedQuiesceConvergesThroughReleaseRetryOverGRPC`：ClaimOperation/ReportNodeIdle(false) 后 running/ready/admission open，接管不改快照，独立 retry 到 done；含同/新 incarnation，资源层为 simulator | Covered |
| 软删除原 Run 仍按 D1/D2 接管/登记 | [integration/late_delivery_fact_deleted_test.go][cloud-deleted] `TestLateDeliveryFactSoftDeletedOriginalRunOverGRPC`：D5/D8 × 有/无后创建 Run，原/新 submission 重放 | Covered |
| 原身份及普通资格围栏不放宽 | [integration/late_delivery_fact_fence_test.go][cloud-fence] `TestLateDeliveryFactOriginalResourceFencesOverGRPC`、`TestLateDeliveryFactCurrentWorkspaceGenerationFenceOverGRPC`、`TestLateDeliveryFactDoesNotBroadenLiveDispatchOrCancellationOverGRPC`：原资源/新 generation、admission-only、取消、替换 Node、新派发负例 | Covered |
| 无效或暂不可达证据不伪造 Revision | [integration/late_delivery_fact_race_test.go][cloud-race] `TestLateDeliveryFactExternalFailureDoesNotAcknowledgeOverGRPC`；[integration/agent_run_delivery_test.go][cloud-delivery] `TestLateInvalidRevisionIsReceiptedWithoutChangingReleasedRun`：无效证据只接管声明，无 Revision；暂失败无成功接管 | Covered |

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
- 长期存量缺少唯一创建记录时持续 receipt-only，不按时间戳/UUID 补造顺序，不因原事件重放升级登记。
- 禁止结果：按 revisions.created_at 或随机 UUID 猜创建先后；只保护已有成功 Revision 的 N；残留路径
  绕过守卫；先返回 ACK 再尝试补登记；把 receipt-only 当成 verified delivery 或额外清理授权。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 所有接收路径使用同一后创建条件 | [integration/late_delivery_fact_order_test.go][cloud-order] `TestLateDeliveryFactRunCreationOrderOverGRPC`：root-D5/D8、reconciling/reclaiming/cleared/controlled × 有/无后创建 Run、反向时间戳、receipt-only 重放、责任及 residual 冲突审计；后创建者为另一 Agent | Covered |
| 后创建状态/软删除与缺失/歧义顺序证据 | 同文件 `TestLateDeliveryFactLaterRunStatesAndMissingOrderOverGRPC`：queued/running/terminal/soft-deleted、original-missing/original-ambiguous/later-missing；`TestLateDeliveryFactNoLaterRunResumesRescuedRevisionOverGRPC`：其他租户/Issue 更大 seq 不阻挡 | Covered |
| 缺创建证据的首次 receipt-only 在后续重放仍不升级 | MissingOrder 只直接断言首次接管；`TestLateDeliveryFactRunCreationOrderOverGRPC` 的 receipt-only 重放由后创建 Run 触发，未直接覆盖缺记录首次接管、后来恢复创建记录再重放 | Partial |
| 登记与新 Run 创建竞争的双向交错 | [integration/late_delivery_fact_race_test.go][cloud-race] `TestLateDeliveryFactLaterRunCommitsDuringExternalReadOverGRPC`：外部读取屏障期间新 Run 先提交；[integration/late_delivery_fact_order_test.go][cloud-order] `TestLateDeliveryFactNoLaterRunResumesRescuedRevisionOverGRPC` 覆盖先登记再创建，但该方向没有屏障控制并发交错 | Partial |
| 后续续作不回到迟到旧成果，无后创建时可救援续作 | [integration/late_delivery_fact_order_test.go][cloud-order] `TestLateDeliveryFactResumeKeepsLaterRevisionOverGRPC`、`TestLateDeliveryFactNoLaterRunResumesRescuedRevisionOverGRPC`：启动下一会话并直接检查冻结 priorRevision；起始/后续 Workspace 为存储夹具 | Covered |

- 决策依据：迟到决策 D2、不变量 3、4；Git 交付决策 D7 的既有候选规则。

## A Late Failure Is Receipted Without Retrying Or Resettling Delivery

- 风险：迟到失败结果再次生成交付重试，或未记收据使 Node 永久重放。
- 前置状态：原已登记 execution 终态未接收，D5 releasing 或 D8 done，原输入与资源仍可核验。
- 触发：提交原 `revision_failed` 有序事件。
- 必须成立：原失败声明及 receipt 同事务提交，可正常 ACK；无新 Revision、work、dispatch 或交付重试；
  Run 的结算字段和原失败 delete 保持不变。不同合法失败原因不能被 Cloud 改写成伪造成功。
- 禁止结果：因 Run 已终态拒收原事实；把迟到失败作为再次结算或重新尝试交付的触发。
- 验证义务与证据：

| 义务 | 直接证据与边界 | 状态 |
| --- | --- | --- |
| D5/D8 迟到失败不重试或重结算 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactRootD5AndD8OverGRPC` 的 failed 分支：原 result/receipt/submission、完整业务及 work/execution 计数快照，原/新 submission 重放 | Covered |
| 各合法失败原因首次接管均保真 | [O4 真栈复测][o4-stack]补充 push_rejected/revision_failed 首次迟到接管；上述测试覆盖 upload_failed；同文件 `TestLateDeliveryFactAcceptedConflictMatrixOverGRPC` 覆盖改为 push_failed 的冲突拒绝，但未直接覆盖每个合法原因首次接管 | Partial |
- 决策依据：迟到决策 D1、D3、不变量 2、5。

## Exact Replays Preserve The Original Decision And Conflicts Preserve Canonical Facts

- 风险：重放变成第二次登记或结算；receipt-only 在稍后重放中被升级；不同事件覆盖已接管原事实。
- 前置状态：分别已有正常登记与 receipt-only 两种迟到成功接管，以及一个迟到失败接管。
- 触发：同 submission 重放；换 submission 的同 execution/result/sequence/event；在首次接管后创建 N 再
  重放；同 execution 不同 result、同 sequence 不同 event、序号缺口、错误原资源和 stale lease。
- 必须成立：相同事实返回原接管结论，既有 Revision 不重复或删除，receipt-only 不补登记，不再次结算；
  真实冲突仍拒绝，无可用于 ACK 的成功答复，canonical 结果/receipt 不变；有账本时仍保留其冲突审计。
- 禁止结果：用业务退休理由吞掉身份/结果/收据冲突；对重放重新申请执行权；为冲突伪造 acceptance。
- 验证义务与证据：

| 义务 | 直接证据与边界 | 状态 |
| --- | --- | --- |
| root/residual 原事实幂等、后创建 receipt-only 不升级 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactRootD5AndD8OverGRPC`；[integration/late_delivery_fact_order_test.go][cloud-order] `TestLateDeliveryFactRunCreationOrderOverGRPC`；[integration/late_delivery_fact_deleted_test.go][cloud-deleted] `TestLateDeliveryFactSoftDeletedOriginalRunOverGRPC`。缺创建记录分支重放缺口见前表 | Covered |
| 冲突不改 canonical/业务，residual 留审计 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactAcceptedConflictMatrixOverGRPC`：verified/receipt-only/revision_failed × event/result/extra-sequence/submission-content；`TestLateDeliveryFactIdentitySequenceAndConflictFencesOverGRPC`：接管前序号缺口及原输入负例；[integration/late_delivery_fact_order_test.go][cloud-order] `TestLateDeliveryFactRunCreationOrderOverGRPC`：residual 各状态冲突持久审计、责任不变 | Covered |
| 重放仍校验当前 lease 与原身份 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactReplayLeaseAndOriginalResourceFencesOverGRPC`：三接管状态 × 原/新 submission，epoch/holder/expired/node/operation/execution 精确错误与无成功 submission | Covered |
- 决策依据：迟到决策 D1–D3、不变量 4–6。

## A Takeover Success Receipt Precedes Ack Without Changing The Business Outcome

- 风险：内存接收、Completed 查询或单独审计被误当接管成功，Node 过早丢弃唯一终态。
- 前置状态：Node outbox 保留原 sequence/event；原业务交付已经失败；分别设置可登记和 receipt-only 分支。
- 触发：阻塞/失败原结果、receipt、去重响应或必要证据提交；返回 Completed 查询；随后解除故障重交原事件。
- 必须成立：提交前无成功回执、无 EventAck、Node 保留事件；完整提交后现有 takeover OK 才允许 ACK 和
  outbox 排空，两分支都成立。业务 deliveryState/revisionId 不因 ACK 改变，回复丢失可以按原事件恢复。
- 禁止结果：收到 Conflict 也 ACK；以 GetDispatch.result 或 Completed 代替本次事件 receipt；新增 disposition
  以规避原提交要求；宣称 ACK 表示 Run 的业务交付改成成功。
- 验证义务与证据：

| 义务 | 直接证据与边界 | 状态 |
| --- | --- | --- |
| Cloud 原子提交后才成功答复 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactAtomicRollbackOverGRPC`：登记分支五表、receipt-only 三张实际写入表故障，全部无成功答复、无残留事实，恢复原事件重交成功且业务不变 | Covered |
| Controller 等成功 receipt 后才 ACK，含 receipt-only、丢响应及重连恢复 | [apps/ora-controller/tests/workspaces/deliveries/refusals.rs][desktop-refusals] `deliveries::refusals::blocked_replay_keeps_refused_terminal_busy_until_receipt`、`reconnect_replays_refused_terminal_before_ack`、`restart_replays_refused_terminal_before_ack`：Controller/WebSocket 与 fake Cloud 提交边界直接断言；重启测试实际先再拒绝、后 transport 重连成功，不包括重启后首次直接成功分支，后者见[Controller 缺口](../../controller/session/refused-delivery-terminals.md#real-reconnect-replays-the-original-fact-and-only-its-success-receipt-clears-refusal) | Covered |
| 真实 Cloud → Controller → Node 的逐事件提交/ACK/outbox 顺序 | [原真实栈回执][real-stack]及[O4 真栈复测][o4-stack]直接覆盖最终 receipt、释放及物理资源，但未逐事件展示 Controller ACK/Node outbox 清理顺序；两仓测试分别覆盖两侧，不能拼成该边界的连续真栈证据 | Partial |
- 决策依据：迟到决策 D3、不变量 5、6；Controller 终态拒绝决策。

## Reconnecting With A Late Terminal Completes The Original Release And Frees Resources

- 风险：原终态的 dispatch_conflict 导致连续重建连接，Run 永久 releasing，container/volume 未释放，
  后续消息永久 waiting；仅证明 Node 短暂 connected 却遗漏资源释放。
- 前置状态：真实 Cloud/Controller/Node/Substrate 链路，原 delivery 已登记；分别覆盖先形成终态后在接收前断线，
  以及 Git Server 阻塞交付、恢复后才形成迟到终态的变体。
  可将测试交付放弃窗口配置为 90 秒以稳定触发，生产默认和 D8 条件不因此改变。根 D5 后 Run releasing、
  Workspace 可在 deleted/deleting 与 running/ready 间变化，无 residual/运维意图；原 delete 尚未被 D8 终结失败。
- 触发：恢复原依赖/Node 网络，或 Cloud 不可用/重启引起 Controller↔Node 会话重建后，重放原终态并继续生产释放；不要求 Controller 重启；另设 execution 及后续 waiting 消息。
  #312 永久失败变体必须显式执行真实 transport 断线重连或 Controller 重启，再重交同一原 terminal，不能只等自然重试。
- 必须成立：Node 完成握手并保持连接，原事件获得真实接管回执后 ACK、outbox 排空；旧终态不再制造
  整会话循环，另一 execution 正常。随后同一原释放路径取得真实 quiesce、sandbox 终止和数据删除证据，
  delete succeeded、Run 到 done，原 container 与 volume 确认释放。waiting 消息不再因这一永久释放阻塞而
  悬挂，但交付失败引起的暂停/用户恢复授权仍遵循串行交接决策，不能为了用例通过自动越过。
- 禁止结果：只断言 Node connected 或 Run done，不核对物理资源；放宽 D8 把反复重连视作释放成功；
  手工写 done/插下游结果；偷偷先强停或新建回收意图；接管本身重写 Run 的历史结算。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| Git Server 阻塞变体后的真实释放与物理资源 | [真实栈回执][real-stack]固定两份 head：Git Server 暂停 → 90 秒 give-up，Run releasing、Workspace deleted/deleting/running-ready 间变化且 1 container/1 volume → 240 秒恢复 → 约 256 秒 done，作者报告恢复后约 10 秒，Workspace deleted、0 container/0 volume，无 Controller 重启；stop 对照约 102 秒 done，产物名见文件开头 | Covered |
| 先形成终态、接管前 transport 断线的真实栈完整时序 | [integration/late_delivery_fact_test.go][cloud-fact] `TestLateDeliveryFactRootD5AndD8OverGRPC` 与[Cloud 释放测试][cloud-release]覆盖对应 PG/gRPC 与 simulator 路径；作者真栈复跑使用 Git Server 阻塞，未直接证明该更窄故障注入时序 | Partial |
| O4 真实精确拒绝后的隔离窗口与显式重连/重启重交 | [O4 真栈复测][o4-stack]：旧 Cloud 拒绝后 60 秒同 incarnation connected、无 ACK/receipt、无第二次 delivery start；换候选 Cloud 并重启 Cloud 引起会话重建，14.6 秒内原失败终态 seq 1 被接受，同一 delete 到 DONE、Run done、Workspace deleted、容器/卷均回收。Controller 未重启即可恢复；版本、作者摘要和边界见文件开头 | Covered |
| 真实重连后原终态被接管并完成跨仓物理释放 | [O4 真栈复测][o4-stack]直接记录 Cloud 不可用导致会话重建、原失败终态 receipt seq 1、同一 delete TERMINATE→CLEANUP→DONE、Run done、Workspace deleted、0 container/0 volume；不要求 Controller 重启，不据此推断逐事件 ACK/outbox 顺序或 O2 成功分支 | Covered |
| 同稳定 Node/资源的新 incarnation 真实恢复 | [integration/late_delivery_release_retry_test.go][cloud-release] `TestLateDeliveryAfterRefusedQuiesceConvergesThroughReleaseRetryOverGRPC/new-incarnation`：生产 EndNode/RegisterNode/ReportNodeStatus 与 simulator 证明原事件接管及释放；[固定版本评审][review]明确真实新 incarnation 仍缺，待外部运行 | Partial |
| 合并后 main 的永久失败与 O4 真栈复跑 | 未在 desktop `2564b42` + Cloud `b777496` 上复跑。[owner 合并说明](https://github.com/ora-space/circus/issues/407#issuecomment-6354830227)确认合并前后相关路径没有变化，复跑不作为本次落地条件；既有 PR-head 真栈证据的 Covered/Partial 评级不变，不宣称 main 已复跑 | Missing |
| 后续输入保留授权边界 | 无直接产品测试/运行证据；[integration/late_delivery_release_retry_test.go][cloud-release] `TestLateDeliveryAfterRefusedQuiesceConvergesThroughReleaseRetryOverGRPC` 与[真实栈回执][real-stack]均未直接覆盖 waiting 输入及显式恢复 | Missing |

- 决策依据：迟到决策 D1、D3、不变量 1、2、5；IssueRun 根 D5/D8；串行交接 D1–D3。

## A Root D8 Late Receipt Does Not Restart The Historical Failed Delete

- 风险：把 #312 的正常释放要求错误套到已经 D8 done 的历史残留，复活失败 delete 或伪造资源已释放。
- 前置状态：生产 D8 已令 Run done、原 delete failed/node_unavailable，无 ledger、无运维意图；原终态未接管。
- 触发：原 Node 恢复并提交原事件，随后既有释放扫描运行。
- 必须成立：按迟到决策接收声明和 receipt，Revision 仍服从创建顺序守卫；原 Run 全行、failed delete
  及结算活动不变，admission 不重开，无新的回收责任或自动重启删除。后续真实资源处置只沿既有显式授权。
- 禁止结果：把历史 done 改回 releasing、原 failed delete 改成功，或因为 ACK 就声称资源释放。
- 验证义务与证据：**Covered** — [integration/late_delivery_fact_test.go][cloud-fact]
  `TestLateDeliveryFactRootD5AndD8OverGRPC` 的 done=true 三终态分支：生产 D8 后接管、重放，再运行
  RedeclareRunWorkspaceDeletesOnce/GiveUpStaleWorkspaceReleasesOnce，行级快照（run / workspace / operations / runtime / sandbox / activities）及计数级快照
  （work / executions / effects / residuals / intents）不变，
  无账本/意图；[integration/late_delivery_fact_order_test.go][cloud-order]
  `TestLateDeliveryFactRunCreationOrderOverGRPC/root-D8` 覆盖有/无后创建 Run 的登记资格。
- 决策依据：迟到决策 D1–D3；残留回收 D6。

## A Later Workflow Run Also Blocks An Older Agent Revision

- 风险：守卫只保护后创建 Agent Run，workflow executor 的后续工作仍被旧成果越过。
- 前置状态：同租户同 Issue 先有已放弃的 Agent Run R，再经生产入口创建 workflow Run N，具有真实较大创建 seq；
  N 分别尚未完成或已终态，可尚无 Revision。
- 触发：接 R 的有效迟到成功、原事件重放，再启动下一次可续作运行。
- 必须成立：N 的 executorType 不影响 D2；R 只保存原声明与 receipt，不登记 Revision/verified delivery、不改业务/责任；
  重放不升级，下一次会话不选择 R。
- 禁止结果：按 executorType=agent 缩小守卫；把 Agent 夹具手改字段当作真实 workflow 创建路径。
- 验证义务与证据：**Missing** — 没有直接覆盖 workflow 后创建路径的产品测试。
  [integration/late_delivery_fact_order_test.go][cloud-order] `TestLateDeliveryFactRunCreationOrderOverGRPC` 与
  `TestLateDeliveryFactLaterRunStatesAndMissingOrderOverGRPC` 的后创建者都由 Agent 创建 helper 产生，只能补充共同守卫证据。
- 决策依据：迟到决策 D2 对同 Issue 任意 Run 的范围。

[cloud-fact]: https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_fact_test.go
[cloud-order]: https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_fact_order_test.go
[cloud-race]: https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_fact_race_test.go
[cloud-fence]: https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_fact_fence_test.go
[cloud-deleted]: https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_fact_deleted_test.go
[cloud-release]: https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_release_retry_test.go
[cloud-delivery]: https://github.com/ora-space/cloud/blob/b777496/integration/agent_run_delivery_test.go
[desktop-refusals]: https://github.com/ora-space/desktop/blob/2564b42/apps/ora-controller/tests/workspaces/deliveries/refusals.rs
[real-stack]: https://github.com/ora-space/circus/issues/407#issuecomment-6341728665
[review]: https://github.com/ora-space/circus/issues/407#issuecomment-6342996936

[o4-stack]: https://github.com/ora-space/circus/issues/516#issuecomment-6345535566
