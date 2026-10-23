# Agent Run 残留资源的对账、受控回收与账本闭环

依据[已实现的残留资源回收决策](../../../decisions/cloud/issue-run/20261018-residual-agent-run-reclamation.md)及[owner 的首版范围与运维授权决定](https://github.com/ora-space/circus/issues/251#issuecomment-6315318982)。下文的 D 编号与不变量编号都指这篇 ADR；“启动决策”指[有界启动与未派发撤回决策](../../../decisions/cloud/issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)，它的核心用例见[有界启动用例](bounded-agent-startup.md)。

本文覆盖 D4/D5 业务收尾之后残留责任的收尾：账本清理状态机、自动对账层的证据与系统清理授权、部署运维的受控回收意图、逐目标终止与 late ensure fencing 的完成条件、三级显式升级，以及存量 D8 删除残留的按身份接管。业务结算、启动与创建截止、迟到事实接管本身的义务仍由[有界启动用例](bounded-agent-startup.md)拥有，本文不重复登记。

回收实现的 PR 为 cloud#112、cloud#116、desktop#710 与 cluster#35；演练工具为 cluster#36。部署证据固定如下，后文“第一轮”“第二轮”“第三轮”均指对应候选，不指可变 PR head：

| 轮次 | 固定演练记录 | 候选与工具版本 | 场景及边界 |
|---|---|---|---|
| 第一轮 | [首轮结果](https://github.com/ora-space/circus/issues/388#issuecomment-6299342004)、[D8 身份接管补跑](https://github.com/ora-space/circus/issues/388#issuecomment-6300622765) | Cloud `59a2443`；desktop `7ea516b`；Sandbox Server 为 cluster#35 `590c444` 加 Node 内存选项（集成提交 `3d9f552`）。工具：S2/S3/S2b 为 `04e2bd8`，S1/原 S4 为 `da3903d`，D8 补跑为 `8640080`。 | S1 通过；S2 约 300 秒不可达后恢复通过，已有卷保留；S3 外部确认通过；S2b inconclusive；原 S4 因 Node 恢复失败未进入接管，D8 补跑仅身份接管通过，已有卷先保留再显式删除。 |
| 第二轮（post-#183） | [S1 重跑](https://github.com/ora-space/circus/issues/388#issuecomment-6308500014) | Cloud `118b759`；desktop `fbe37e24`；cluster main `3e07a4c`；工具 `8209d6b`。 | 只运行 S1，恢复后沿同一意图与两个目标取得双确认，已确认目标不重发，显式 `delete-data` 到 `released`。终止前 Run Workspace 没有数据卷，不能证明已有卷被保留再物理删除。 |
| 第三轮（#560） | [固定候选演练结果](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955) | Cloud `d73eca76`；desktop `e177e021`；cluster / Sandbox Server `ae1dd951`。工具 driver `b87d30b`，复用 proxy 镜像源码 `7e08dc6`。 | `p343-run01` 的 S2b inconclusive、S2 中断不计证据；`p343-run02` 复用同一批镜像，仅 S2/S3/D8，均通过。D5/D8 均使用真实默认 30m；D8 最后活动到事务 1807.9 秒，原身份接管、双确认 retained、显式 delete-data 到 released、零账本及历史不变。 |

前两轮都是单机 Docker 上的真实服务与 Node。S1 的第二个历史 sandbox 身份仅作为输入布置，后续计划、派发、终止与确认来自真实系统，不代表真实多代历史。第一轮 D8 补跑先确认交付已被 Cloud 接收，再经 `ReportNodeIdle` 屏障断线，避开未接收终态重连冲突（ora-space/circus#392）；约 9.5 秒即形成 D8，未遵守默认 30 分钟窗口（ora-space/circus#391，现已由 ora-space/cloud#118 修复，但前两轮均未验证修复后的窗口）。它不证明窗口条件或旧版本滚动升级。第二轮没有重跑 S2、S3、S4、S2b 或 D8 接管，第一轮结果只适用于第一轮候选，不能作为最终 PR head 的部署证据。

第三轮采用同一 pinned candidate 的真实默认窗口，[S2/S3/D8 均通过](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)；`last_seen_at` 是 Node 最后记录的活动，Run `updated_at` 是 D8 事务时间戳，1807.9 秒不是精确心跳发送到提交的耗时。它补齐重试与显式升级、D8 身份接管两项跨仓联合路径，不证明旧版本滚动升级或未测试的故障交错。

Cloud 直接测试与部署证据分开判断：复核基线为 cloud#116 `6257136bfc164fc39a6c0a0a87e73bd173425e23`，已合入 cloud#112 及后续 main；下文 Cloud 测试见该提交的 [core](https://github.com/ora-space/cloud/tree/6257136bfc164fc39a6c0a0a87e73bd173425e23/internal/core)、[integration](https://github.com/ora-space/cloud/tree/6257136bfc164fc39a6c0a0a87e73bd173425e23/integration) 与 [cloudctl](https://github.com/ora-space/cloud/tree/6257136bfc164fc39a6c0a0a87e73bd173425e23/cmd/cloudctl)。它们在第二轮 Cloud `118b759` 已存在，向前合并后保留相关断言；前六项的 Cloud 范围由这些直接测试支撑，不把第一轮物理结果升级为新版结果。物理成功在这些测试中由提交证据驱动，不是实际容器或卷的证明；跨进程与跨仓库的完整义务不能由 Cloud 单仓测试单独记为 `Covered`。沿用该提交[已有 CI 记录](https://github.com/ora-space/cloud/actions/runs/39556403820)中相关测试的通过结果；该 CI 整体因 integration 包超时失败，本次没有重跑测试。迁移编号按该基线为 residual `0057`、controlled `0058`。单机演练不覆盖网络与崩溃的全部交错、断电、多 Controller 并发或非 Docker 的生产 Substrate。

迟到成果救援也受[迟到交付事实与续作顺序](late-delivery-fact-takeover.md)的后创建 Run 守卫约束：创建先后依同 Issue 原子 `run.enqueued` 活动 `seq` 判断，已有后创建 Run，或原 Run 与同 Issue 其他 Run 的创建顺序证据缺失、有歧义时，仅保存声明与 receipt，不登记迟到 Revision；只有顺序证据唯一且没有后创建 Run 时救援行为才不变。receipt-only 不算已验证 Revision 或额外清理授权。下列既有 `Covered`/`Partial` 不自动覆盖该新增规则；Cloud #142 merge `b777496` 的直接测试已覆盖各接收路径的后创建 Run 守卫，缺创建记录后的重放、双向并发交错仍为 `Partial`，workflow 后创建路径仍为 `Missing`，逐项见该独立用例。

## Residual Ledger Reclaim State Advances Only With Per-Target Disposal Evidence

- 风险：账本被当作“业务终态之后的表单”提前清除，残留资源失去保护；或对账证据不落账，重启后无从判断谁还欠什么。
- 前置状态：`agent_run_residuals` 中有 D4 启动放弃与 D5 创建放弃两种来源的行，`cleared_at` 为空；另有一条只有部分沙盒取得终止证据的行。
- 触发：对账周期写入部分证据；全部目标闭环后写 `cleared_at`；推进者重启后再对同一行对账。
- 必须成立：每行有 `reclaim_state`（`reconciling → reclaiming → cleared`）与处置方式 `disposition`（`released` / `terminated_data_retained` / `externally_confirmed`）；证据逐条追加（类型、来源、时间），`reason` 与 `responsibility` 不被改写；`cleared` 只在全部目标取得闭环证据或显式处置记录时写入；重启后接管同一责任，不产生第二份推进责任。
- 禁止结果：证据不全即写 `cleared_at`；用业务 `done`/`failed` 或时间流逝代替处置证据；重复对账重建 Run、Workspace、ensure 或执行身份；改写 `reason` 与既有 `responsibility`。
- 验证义务与证据：**Covered** — cloud#112 的 `TestResidualOwnedDeleteRunsCloudTopologyAndPreservesBusinessTerminal`、`TestResidualCleanupRequiresEveryDualTerminationAndRemovedProof`、`TestResidualCompletionRollbackKeepsLedgerAndWorkspaceAtomic`、`TestResidualClaimRollbackAndRestartKeepOneBoundOperation`；cloud#116 的 `TestControlledResidualEveryPlannedSandbox`、`TestControlledResidualRetainedThenReleased`、`TestControlledResidualQueryAndResume` 覆盖另两种处置方式与首次 `cleared_at` 保留。第一轮 S1/S2/S3 观察到全部目标闭环前 `cleared_at` 为空，分别以 `released`、`terminated_data_retained`、`externally_confirmed` 闭环；第二轮只补证 S1 的账本与意图状态，不证明已有卷删除。
- 决策依据：D1，不变量 2。

## Reclaim State Migration Marks Existing Rows Reconciling Without Backfilling Evidence

- 风险：升级时按历史推断出“已清理”，把仍有物理资源的条目销账；或迁移失败让既有保护消失。
- 前置状态：迁移前已有的 D4/D5 账本行，无清理证据列；正常样本 `cleared_at` 为空，另有异常旧行已为非空。
- 触发：新库迁移、升级迁移、重复执行迁移；异常旧行使升级受阻后，运维按 ADR D1 排查并决定兼容处理。
- 必须成立：正常存量行统一置 `reconciling`，不回溯判定证据，保留空 `cleared_at`、原 `reason` 与 `responsibility`；表尚无 `reclaim_state` 而旧行 `cleared_at` 非空时，0057 明确报错并回滚本次迁移事务，不猜测处置、不记为已应用。运维以 D1 的旧列只读 SQL 定位，核对原身份及实际责任；只有证明误写后才允许单独审阅的定点数据库修正，真实历史闭环或无法查明者保持阻挡并另行设计兼容处理。`agent_run_residuals` 只增列，既有保护查询不变；重复迁移不重置进度。
- 禁止结果：迁移推断出处置证据或写 `cleared_at`；为升级批量清空旧时间戳、删行、跳过 guard 或伪造迁移记录；将 `cloudctl residual` 当作迁移前修复工具；改 `reason` CHECK 之外的来源；升级后残留保护失效。
- 验证义务与证据：**Covered** — cloud#112 的 `TestMigration0057ResidualReclamationFreshAndConstraints`、`TestMigration0057ResidualReclamationUpgradePreservesHistory`、`TestMigration0057ResidualReclamationReplayPreservesProgress`、`TestMigration0057ResidualReclamationRejectsLegacyClearedRows`（本应为空的旧 `cleared_at` 非空时迁移失败并回滚）；cloud#116 的 `TestControlledResidualMigrationFreshAndConstraints`、`TestControlledResidualMigrationUpgradePreserves0057HistoryAndDoesNotInventD8Ledger`；迁移后普通入口仍拒绝见 `TestResidualOrdinaryLifecycleGuardsAndUnrelatedD8StayProtected`。两轮候选均经正常迁移启动；这不代替存量行升级与迁移重放的直接测试。
- 决策依据：D1（存量行迁移），兼容性边界。

## Automatic Reconciliation Collects Evidence Only And Never Terminates

- 风险：自动层把“证据不足”解释成“已放弃”，对仍在执行的未知目标发起终止或数据删除；或把查询失败记成证据。
- 前置状态：`reconciling` 行，其中一条执行状态未知、Controller 暂时不可达，另一条已有迟到终态但交付未确认，第三条全部证据齐全。
- 触发：对账周期扫描既有事实；Controller 不可达或无法回报历史事实；已有事实缺失；交付验证未完成。
- 必须成立：自动层只读取经既有事实入口持久接管的原身份证据，成果救援经既有 Revision 核验路径，不新增取证查询 RPC 或 Controller 历史发现能力；证据齐才申请系统 owned delete 授权。交付资格仅来自原 `deliver_revision` 执行登记且按原身份核验完成的 Revision，其他交付执行也须各自核验完成；首版没有明确放弃交付的自动资格路径。Controller 无法回报历史事实时保持 `reconciling`，交运维依 D3 显式接管；不可达、租约失效或读取失败不算证据，下周期重读；“没有结果记录”不代替完整证明。
- 禁止结果：自动层排 `sandbox_terminate` 或针对未知执行的数据删除；把 `done`、超时或查询失败当作放弃证据；因救援重开会话、输入或续期窗口。
- 验证义务与证据：**Covered** — cloud#112 的 `TestResidualUnknownAndDeliveryFailureNeverAuthorizeDataDeletion`、`TestResidualPartialAndNullScopeTargetsRemainAUnion`、`TestResidualOwnedDeleteRunsCloudTopologyAndPreservesBusinessTerminal`；cloud#116 的 `TestControlledResidualDoesNotAuthorizeAutomaticEscalation`、`TestControlledResidualRaceAutomaticClaimBothCommitOrders`。证据只经既有事实入口到达，不新增查询 RPC；Controller 无法回报的历史事实使条目保持 `reconciling`。第一轮 S2 的 Substrate 约 300 秒不可达期间没有自动升级或数据删除，第一轮 D8 补跑的 Node 恢复后跨两个扫描周期没有新的删除；第二轮未重跑这两条路径。
- 决策依据：D2，不变量 1、3。

## System Cleanup Authorization Keeps Residual Protection Until Every Target Closes

- 风险：为执行 owned delete 而提前清账，解除保护并拒收创建侧迟到事实；或自动层与运维命令同时取得删除/终止权。
- 前置状态：证据齐全的 `reconciling` 行，其 Workspace 的普通 `deleteRunWorkspace` 入口当前以 `resource_in_use` 拒绝；另有并发到达的受控回收命令与创建侧迟到事实。
- 触发：自动层申请系统清理授权；运维受控回收命令同时到达；清理进行中普通入口、Project stop/delete 与创建侧事实入口再次被调用；清理事务失败。
- 必须成立：新增的 Cloud 系统清理入口只在同一互斥事务内重读原资源关联与完整证据、固定全部待处置目标并封闭后续新派发后授予清理权；授权只针对这次残留与原资源，不授予新会话、ensure 或换代；取得单一持久责任后置 `reclaiming`，`cleared_at` 保持空值；`reclaiming` 期间只有绑定该责任的系统动作越过残留阻挡，公开 API、普通删除及涉及残留的 Project stop/delete 继续受保护；事务失败全部回滚，不留半份责任；执行仍逐步核验 quiesce、终止与围栏、数据删除证据。
- 禁止结果：先写 `cleared_at` 再调用普通删除入口；清理未完成时普通入口或 Project 操作取得旁路；自动层与受控命令并行计划副作用，或自动层自行升级为强停；授权授予新会话、ensure 或换代权。
- 验证义务与证据：**Covered** — cloud#112 的 `TestResidualClaimRollbackAndRestartKeepOneBoundOperation`、`TestResidualOrdinaryLifecycleGuardsAndUnrelatedD8StayProtected`、`TestResidualCompletionRollbackKeepsLedgerAndWorkspaceAtomic`、`TestResidualRaceAuthorizationVersusLateFactsBothCommitOrders`、`TestResidualRaceAutomaticVersusControlledClaimBothCommitOrders`；cloud#116 的 `TestControlledResidualGuardsAndRollback`、`TestControlledResidualRaceAutomaticClaimBothCommitOrders`、`TestControlledResidualRaceNoTargetProofAndDispatchBothCommitOrders`。竞争测试用真实 PostgreSQL 的两个事务覆盖两种提交顺序；Project stop/delete 经生产共用的残留阻挡验证，没有单独的 HTTP 路由测试。第一轮 S2 的 `reclaiming` 条目在故障期间保持 `cleared_at` 为空；第二轮仅补证 S1 的恢复路径。
- 决策依据：D2（新增授权与保护转换），不变量 5、9。

## Late Facts Are Taken Over Across Reclaiming And Cleared Without Resurrecting Resources

- 风险：账本清除后拒收同身份真实事实，或迟到事实被用来复活已回收的资源与业务。
- 前置状态：一条已进入 `reclaiming` 的残留，另一条已 `cleared`；原 effect/execution 身份与租户、资源仍可核验；迟到事实与清理完成事务并发到达。
- 触发：Controller 以原身份提交迟到终态、交付结果或 effect 事实；同一事实重复提交；清理完成先提交、迟到事实后提交（以及相反顺序）。
- 必须成立：创建侧事实入口与原执行接管覆盖 `reconciling`、`reclaiming` 与 `cleared`；仍校验原租户、资源、effect/execution 身份及当前服务资格；先持久接管再 ACK，相同事实幂等重放只追加证据；不重开 admission、不再次结算、不重建已删除资源；后提交者看到先提交者的证据与责任，新发现的未闭环目标阻止清除。
- 禁止结果：因账本已清除或 Run `done` 拒收事实；迟到事实把 Run 改回运行态或复活 Workspace/ensure；事实与处置结论冲突时静默丢弃事实或伪造新的完成证明。
- 验证义务与证据：**Covered** — cloud#112 的 `TestResidualProvisioningFactsAcrossReclaimStatesOverGRPC`、`TestResidualSessionTerminalTakeoverOverGRPC`、`TestResidualVerifiedDeliveryTakeoverOverGRPC`、`TestResidualRaceCompletionVersusLateDeliveryBothCommitOrders`、`TestResidualRaceCompletionVersusNewUnclosedExecutionTargets`、`TestResidualRaceClearedReplayVersusLateFactBothCommitOrders`；cloud#116 的 `TestControlledResidualTerminalFactsAcrossRealStatesOverGRPC`、`TestControlledResidualProvisioningFactConflictCommitsBeforeGRPCResponse`、`TestControlledResidualRaceCompletionAndLateFactsBothCommitOrders`。冲突事实先提交审计与阻挡，再返回 `residual_fact_conflict`。
  新增迟到登记守卫的直接证据为 Cloud #142 merge `b777496` 的
  [integration/late_delivery_fact_order_test.go](https://github.com/ora-space/cloud/blob/b777496/integration/late_delivery_fact_order_test.go)
  `TestLateDeliveryFactRunCreationOrderOverGRPC`：reconciling/reclaiming/cleared/controlled 的交付终态接管、
  有/无后创建 Run 两侧、业务及责任不变、原事件幂等、receipt-only 不升级、冲突持久审计。
  该测试只覆盖交付事实子路径，不单独证明全部创建侧/effect 事实、清理完成事务的双向并发或冲突后后续处置被阻断；
  原有 `Covered` 仍由上列 residual/controlled 专项测试支撑，新增守卫的保留缺口见[独立用例](late-delivery-fact-takeover.md)。
- 决策依据：D2（事实接管贯穿清理前后），不变量 10。

## Controlled Reclamation Is An Explicit Deployment Intent With Confirmation And Audit

- 风险：受控回收变成租户侧可枚举隐藏 Workspace 的旁路，或终止成功即被当作删卷授权。
- 前置状态：证据长期收不齐的残留（含无账本行的存量 D8 残留）；隐藏运行 Workspace 对公开 API 恒 404；同租户另有运行中的 Workspace。
- 触发：部署运维凭部署数据库凭据运行本地 `cloudctl residual` 发起回收（带/不带显式放弃确认、重复幂等键、同键不同输入）；只终止不删数据；随后单独发起数据删除；租户管理员经公开 API 尝试同样操作。
- 必须成立：受控回收只经本地 `cloudctl residual` 发起，没有 HTTP 管理端点；授权来自受控部署主机及数据库凭据访问控制，认证主体为 PostgreSQL `session_user`，`--operator-label` 不能替代或覆盖它，租户角色不授予此权；输入含目标范围、非空 `reason` 与语义同强停 `impactConfirmed=true` 的显式放弃确认；意图独立持久化为 `registered → terminating → succeeded` 并留发起者、时间、reason 审计；确认语义同强停，终止与数据删除分两个命令；只终止不删数据是合法终态（`terminated_data_retained`），删卷必须另有显式命令（`released`）；同幂等键重放返回原意图，同键不同输入拒绝；租户侧与公开 API 仍 404，不获得入口。
- 禁止结果：租户管理员或普通成员发起受控回收；缺确认即终止；终止成功隐式排 `workspace_data_delete`；重复命令产生第二份责任；改写 Run 终态、重开 admission 或重写 `deliveryState`。
- 验证义务与证据：**Covered** — cloud#116 的 `TestControlledResidualCommands`、`TestControlledResidualRetainedThenReleased`、`TestControlledResidualRetainedThenReleasedOverGRPC`、`TestControlledResidualHiddenWorkspaceRemains404ForAdminAndMember`、`TestControlledResidualRaceIdempotencyBothCommitOrders`、`TestControlledResidualRaceDifferentKeysKeepOneRoot`、`TestCloudctlResidualManagement`、`TestCloudctlResidualRejectsUnsafeOrAmbiguousArguments`。部署运维入口是本地 `cloudctl residual` 命令，使用部署数据库凭据，操作主体为 PostgreSQL `session_user`，没有 HTTP 路由。第一轮 S1 与 D8 补跑用真实 `cloudctl residual` 走完 terminate → `terminated_data_retained` → 独立 `delete-data` → `released`，S3 验证非法请求拒绝、重放与冲突审计；已有卷保留再物理删除的证据来自第一轮 D8 补跑。第二轮 S1 只补证两阶段命令、独立删除 operation 与状态转换，终止前无卷，不能证明已有卷的保留和删除。本项 `Covered` 限上述 Cloud 授权、命令与审计契约；其测试注入的物理成功结果不是新版物理删卷证明。
- 决策依据：D3，不变量 3、5。

## Termination Completes Only With Per-Target Terminated And Late Ensure Fenced Evidence

- 风险：用 Substrate 查询 `404` 或 Node 不可达代替终止证据，或迟到 ensure 让已经回收的 sandbox 重新创建出可写实例。
- 前置状态：账本记录多个稳定沙盒计划 ID，包含已 ensure、仍在途和从未执行 ensure 的目标；旧请求可能携带回收前签发的许可。
- 触发：回收意图经当前 Controller lease 逐目标执行 `sandbox_terminate`；旧 ensure 迟到，或 Controller 许可失效 / Substrate 不可达；终止双确认后读回目标状态。
- 必须成立：Controller 只尝试有界的既有 Shutdown 控制关闭，不新增 EndSession 或 Cloud 直连 Substrate 路径，不以控制关闭响应替代双确认；仍以当前 lease epoch 和许可执行，许可失效不发起新动作，Cloud 只规划既有 `sandbox_terminate` / `workspace_data_delete` effects。生产 Substrate 适配器保持先墓碑后破坏、幂等、目标不存在也双确认的语义；目标包括从未执行过的 ensure 计划，逐 ID 取得 `terminated` 与 `lateEnsureFenced`。Cloud 只有在与创建计划 / 派发互斥的事务中证明从未形成计划、许可或在途责任并封闭后续入口时，才可记录“无目标待终止”。受控回收关闭 admission、提升 control epoch 后，旧 ensure 不得让已回收 sandbox 复活；墓碑对当前 epoch 的有效 ensure 保留纵深防御。
- 禁止结果：以 `404`、Node 不可达、许可过期或一条拒绝响应替代完整终止证据；遗漏未到达的计划 ID；终止确认前删卷；删除原 effect、假报 sandbox 缺失或换 generation 挂载同一可写卷；已回收 sandbox 被迟到 ensure 重建。
- 验证义务与证据：**Partial** — 按 [#573 owner 决定](https://github.com/ora-space/circus/issues/573#issuecomment-6365603852)，义务保护“迟到 ensure 不能让已回收 sandbox 复活”；下面三部分均有直接证据才为 `Covered`，不再要求生产路径必须出现墓碑 503。

**第 1 部分：固定候选的结构性论证。** Cloud `d73eca76`、desktop `e177e021`、cluster `ae1dd951`：

1. Cloud [`effectPermit` :96–128](https://github.com/ora-space/cloud/blob/d73eca76fe65746b39396403864e200cee55d6f5/internal/core/runtime_force_stop.go#L96) 每次实时核验责任后签发，期限不超过当前时间 +10 秒且不超过 Controller lease；不是可长期重放的许可。Controller 的 [MIN=2000ms 与 validate_permit :244–261](https://github.com/ora-space/desktop/blob/e177e0214ea235595d8198927e135bb6fff880d5/apps/ora-controller/src/cloud/substrate.rs#L244) 要求剩余 >2 秒，并在 [PUT 前 :177–185](https://github.com/ora-space/desktop/blob/e177e0214ea235595d8198927e135bb6fff880d5/apps/ora-controller/src/cloud/substrate.rs#L177) 再核验；Sandbox Server 的 [`authorization.valid` :24–34](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/authorization.go#L24) 同样要求剩余 >2 秒及最长 10 秒。因此原许可的可用送达时间少于约 8 秒。创建期残留在默认 ≥30 分钟后才有资格，不能把原许可一直扣留到回收后再当成仍有效。
2. Cloud [`registerControlledTermination` :118–137](https://github.com/ora-space/cloud/blob/d73eca76fe65746b39396403864e200cee55d6f5/internal/core/agent_run_controlled_reclaim.go#L118) 在既已关闭 admission 的资源上登记受控责任，再调用 [`closeAdmission` :745–750](https://github.com/ora-space/cloud/blob/d73eca76fe65746b39396403864e200cee55d6f5/internal/core/public.go#L745)，写 `admission_open=false` 并推进 admission epoch。普通 effect 许可入口先执行 [受控责任 / 残留冲突检查 :110–125](https://github.com/ora-space/cloud/blob/d73eca76fe65746b39396403864e200cee55d6f5/internal/core/runtime_force_stop.go#L110)，[`requireNoControlledReclamation` :133–145](https://github.com/ora-space/cloud/blob/d73eca76fe65746b39396403864e200cee55d6f5/internal/core/agent_run_controlled_reclaim_targets.go#L133) 保护未完成、retained 及 externally-confirmed 资源；不会给旧 ensure 责任继续签许可。许可是实时返回值，没有可用 effect 快照替代的签发账本。
3. 登记受控 terminate 在同一事务将 [runtime control epoch 从 E 提升至 E+1 :131–134](https://github.com/ora-space/cloud/blob/d73eca76fe65746b39396403864e200cee55d6f5/internal/core/agent_run_controlled_reclaim.go#L131)。Sandbox [先校验许可和旧 epoch :21–38](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/runtime_fence.go#L21)，再检查 [墓碑 :40–48](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/runtime_fence.go#L40)；有效期内的旧 E 先被拒为 `403 stale_runtime_effect_permission`。[`lifecycle.put` :198–218](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/lifecycle.go#L198) 在重放历史或执行前调用 authorizeEffect；新 epoch / closed generation 的 [持久写入 :55–66](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/runtime_fence.go#L55) 完成后才会 [执行 terminate :254–258](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/lifecycle.go#L254)，后者 [先写墓碑再 stop/remove 并确认容器消失 :112–140](https://github.com/ora-space/cluster/blob/ae1dd951950d0295849c3b2f5271b02a2bc9de7c/sandbox-server/internal/lifecycle/effects.go#L112)。所以回收后旧许可即使尚未超时也没有可用权威；超时更不能绕过检查。当前 epoch 的有效 ensure 命中墓碑则为 `503 sandbox_terminated`。

**第 2 部分：单仓直接测试。**

- 已合并 commit [`590c4449f1e3fd5c8c2ce01f8faad9fad8bcfc50`](https://github.com/ora-space/cluster/commit/590c4449f1e3fd5c8c2ce01f8faad9fad8bcfc50) 的 [`termination_test.go` / `TestPlannedEnsure404ThenTerminateRejectsLateAuthorizedEnsure`](https://github.com/ora-space/cluster/blob/590c4449f1e3fd5c8c2ce01f8faad9fad8bcfc50/sandbox-server/internal/lifecycle/termination_test.go#L237)：保持同一当前 epoch，明确校验原许可仍有效；terminate 双确认后 ensure 返回 503 `sandbox_terminated`，无 create/start、容器为空且不写入 ensure journal entry。
- [cluster#45](https://github.com/ora-space/cluster/pull/45) **待合并**，固定 head `0cb2bcd390b0ef7a91242c79a8726ddf58b01bd9`，[`old_control_epoch_test.go` / `TestValidOldControlEpochPermitCannotRecreateTerminatedSandbox`](https://github.com/ora-space/cluster/blob/0cb2bcd390b0ef7a91242c79a8726ddf58b01bd9/sandbox-server/internal/lifecycle/old_control_epoch_test.go#L15)：原 ensure E=2，terminate E=3；读回持久 epoch / closed generation，原许可仍有效且作用域不变时 HTTP 403 `stale_runtime_effect_permission`，无 create/start、容器仍不存在，墓碑与历史保持。只加测试；WSL 正常生命周期包测试通过（0.379s）。fake Engine + 真实临时 journal 属于单仓证据，不冒充真实 Docker 栈。
- 原逐目标与并发证据仍保留 Cloud `TestResidualCleanupRequiresEveryDualTerminationAndRemovedProof`、`TestResidualPartialAndNullScopeTargetsRemainAUnion`，以及 `TestControlledResidualEveryPlannedSandbox`、`TestControlledResidualRetainedThenReleasedOverGRPC`、`TestControlledResidualRaceNoTargetProofAndDispatchBothCommitOrders`；Controller 的 `destructive_put_requires_current_unexpired_permit`、`force_stop_preserves_all_cloud_effect_targets_and_evidence`、`force_stop_attempts_bounded_control_close_before_terminate`、`tombstoned_ensure_does_not_retry_or_change_generation`、`absence_and_unavailable_nodes_are_not_termination_confirmation` 仍按 desktop#710 保留（fake peers）；Sandbox 的 `TestTerminateOfUnknownSandboxSucceedsAndTombstones`、`TestTerminateTombstonePrecedesDestroyAndReplayIsIdempotent` 等原单仓证据也保留。

**第 3 部分：真实栈读回及缺口。** [#560 固定候选结果](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)，本地证据 `hb-management/circus-556/evidence/p343-run02/drill.json`（下列为 JSON pointer，日期 2026-10-23 UTC）：

- S2 `/scenarios/0`：原 ensure `d4c431da-8a9a-43b6-bdc0-92576eb21895` 的唯一 PUT received / forwarded / HTTP200 在 `/scenarios/0/faultEvents/3`、`/scenarios/0/faultEvents/4`、`/scenarios/0/faultEvents/5`（08:52:40.179731014Z 至 08:52:40.377403Z），早于受控回收；首 terminate PUT `/scenarios/0/faultEvents/11` 为 09:23:51.281338112Z。`/scenarios/0/observations/1` 至 `/scenarios/0/observations/7` 的 effect 快照保持原 ensure ID，admission 关闭；最后 `restored-retained`（`/scenarios/0/observations/7/databaseNow=09:29:03.495277Z`）Sandbox terminated、`terminated_at=09:29:01.509131Z`，ledger retained / cleared。`/scenarios/0/physicalEvidence/0` 为 terminate succeeded、tombstone / containerAbsent=true。proxy 的最后事件 `/scenarios/0/faultEvents/78` 在 09:29:01.502830144Z，早于 `cleared_at=09:29:01.514328Z`，不能宣称覆盖完整的双确认后观察窗口。
- D8 `/scenarios/2`：原 ensure `0b5639d8-0c77-4f01-b408-cb7812065608` 的唯一 PUT 在 `/scenarios/2/faultEvents/3`、`/scenarios/2/faultEvents/4`、`/scenarios/2/faultEvents/5`（10:02:30.959869516Z 至 HTTP200 10:02:31.153938968Z）；首 terminate PUT `/scenarios/2/faultEvents/1805` 为 10:37:12.322845153Z，成功响应 `/scenarios/2/faultEvents/1812` 为 10:37:22.616669317Z。最终 `/scenarios/2/observations/6` 为 released，`databaseNow=10:38:23.497986Z`、Sandbox terminated、`terminated_at=10:37:22.624795Z`，effect 列表只有原 ensure 加 data-delete，`/scenarios/2/physicalEvidence/0` 的 containerAbsent / tombstone=true。已记录的后续事件无 ensure PUT，但没有保存双确认完成时间，物理证据也无自身采样时间。
- 两场景的 `observations[*].effects` 只有 id / operation_id / kind / state / result / reconciled_epoch，没有创建时间或 permit 签发记录；workspace 有 admission_open / runtime_generation，却没有 runtime_controls 行与 epoch 事务时间。`serviceLogMetadata.cloud.structuredLines=0`、scopedEvents 为空。故只能说“快照中未见新增 ensure effect、已记录事件中无回收后 ensure PUT、最终读回未复活”，不能把这些或第 1 部分源码推论当成“Cloud 未签发新 permit”的直接运行 trace。

**结论与剩余缺口：** 第 2 部分新增测试已有直接结果，但位于 cluster#45 的待合并固定 head；第 3 部分还缺受控开始到双确认后完整观察窗口的许可签发 / 拒绝 trace、runtime-control / intent 时间与带时间的物理读回。#575 仅拟同候选最小补录计划，未运行；本义务保持 `Partial`，总数仍为 **8 Covered / 1 Partial**。原 held ensure 的响应仅作为运行记录（#560 实际为资格前 HTTP200）；包括过期许可 403 在内，均不作为本义务的围栏证据。
- 决策依据：D4，不变量 8；[#573 证据组成决定](https://github.com/ora-space/circus/issues/573#issuecomment-6365603852)。

## Reclamation Intents Retry And Escalate Only On Explicit Operator Decision

- 风险：未完成的意图因超时被判成功，或把未知写成已处置，账本带着虚假证据闭环。
- 前置状态：`terminating` 状态的意图，Controller 失联、Substrate 长期不可达；另一条账本长期停在 `reconciling` 且无新证据。
- 触发：Controller 恢复；Substrate 持续不可达；运维发起外部确认（有意图与无意图两种）。
- 必须成立：自动层按周期重试，已授权的 owned delete 由持久清理责任驱动，复用既有 quiesce/删除步骤与幂等规则，不依赖只扫描 `releasing` 的业务重试循环；受控意图保持未完成即持续对账，恢复后继续；任何超时都不把未确认判为成功；升级只能是三级显式路径（自动对账 → 受控回收 → 外部确认），自动层不发起受控回收也不申请外部确认；外部确认必须先有未完成的回收意图，非空确认说明留存审计，账本以 `externally_confirmed` 闭环。
- 禁止结果：超时自动成功或自动升级；无意图直接销账；外部确认文本写成 `terminated` 或伪造 `lateEnsureFenced`；告警缺失被当作已完成。
- 验证义务与证据：**Covered** — cloud#112 的 `TestResidualClaimRollbackAndRestartKeepOneBoundOperation`、`TestResidualOwnedCleanupRetriesSameFailedEffectAcrossControllerEpoch`、`TestResidualRaceRestartRebuildsTwoStoresWithoutNewResponsibility`；cloud#116 的 `TestControlledResidualQueryAndResume`、`TestControlledResidualExternalRequiresUnfinishedIntent`、`TestControlledResidualExternalDisposition`、`TestControlledResidualRaceExternalAndPhysicalCompletionBothCommitOrders`。第一轮 S1 中 Controller 停止 31 秒后重启，同一根意图与目标继续，已确认目标没有再次 PUT；第一轮 S2 约 300 秒内意图保持 `terminating`、无成功写入、无自动数据删除，恢复后凭真实双确认闭环；第一轮 S3 拒绝无意图、缺目标引用或缺责任转交的请求，有效确认留存主体、理由、逐目标外部引用与责任转交，没有写物理成功证据。第二轮只重跑 S1，确认同一意图恢复、不重发已确认目标；未补证新版候选的 S2/S3。Cloud 直接测试支撑持久责任、重试与外部审计规则，不能代替新版跨仓长期不可达及外部确认的联合路径，因此前两轮时完整义务为 `Partial`。第三轮 `p343-run02` 的 S2 + S3 在 Cloud `d73eca76`、desktop `e177e021`、cluster `ae1dd951`，driver `b87d30b`（proxy `7e08dc6`）上补齐联合路径，见[固定证据](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)：S2 连续 300 秒 404 / Node 丢失不完成、不自动删数据，恢复后真实双确认到 `terminated_data_retained`；S3 显式外部确认留审计并转为 `externally_confirmed`，Sandbox 仍 allocating、terminated_at=null，没有补造物理终止证据。结合上述直接测试，本义务改为 `Covered`。演练只证明所测故障时长，不证明无限期失联。
- 决策依据：D5，不变量 6。

## Legacy D8 Residuals Are Taken Over By Identity Without Rewriting History

- 风险：Node 恢复后既有重试自动重新声明删除，或为接管而回填账本、改写历史失败结果。
- 前置状态：存量 D8 残留：Run 已 `done`、`delete_workspace` 以 `node_unavailable` 终结失败、Workspace 行与运行绑定保留、无 `agent_run_residuals` 行；对应 Node 随后恢复可达。
- 触发：Node 恢复；释放扫描周期运行；部署运维按原资源身份发起受控回收；回收完成或外部确认。
- 必须成立：Node 恢复不会让 `done` Run 重新进入只扫描 `releasing` 的既有循环，本 ADR 不新增存量自动扫描；接管以资源身份（Workspace、失败的 delete operation、沙盒计划）为目标，不要求也不伪造账本行；无账本行时由回收意图承载保护与证据；终止仍逐目标双确认，数据删除仍需单独显式命令；完成后由新意图及处置记录引用原失败 operation，原 operation 的 `failed/node_unavailable` 与 Run 的 `status`/`done`/结算来源都不变。
- 禁止结果：回填账本行或改写 `reason` 来源；把历史 `done` Run 改回 `releasing` 借用普通扫描；把原 operation 改成成功；Node 恢复连接被当作清理授权。
- 验证义务与证据：**Covered** — cloud#116 的 `TestControlledResidualLegacyD8`、`TestControlledResidualLegacyD8RetainedThenReleased`、`TestControlledResidualRaceNoTargetProofAndDispatchBothCommitOrders`；cloud#112 的 `TestResidualOrdinaryLifecycleGuardsAndUnrelatedD8StayProtected`（无账本 D8 不被自动补账或重试）与既有 `integration/agent_run_release_giveup_test.go` 的 `TestG032_7_ADoneRunIsNeverRedeclaredOrRewritten`。第一轮 D8 身份接管补跑 `legacy-d8`（工具 `8640080`，`--legacy-fault idle-barrier`）：交付先被 Cloud 接收再断线，自然形成零账本 D8；同一 Node incarnation 恢复后跨两个扫描周期没有新的删除；随后按原 Run、Workspace 与失败 delete 身份经 `cloudctl residual terminate` 取得真实双确认到 `terminated_data_retained`，再独立 `delete-data` 到 `released`；重放仍是同一根意图，原失败 operation、Run 终态与版本始终不变，账本始终无行。边界：根 D8 默认 30 分钟窗口在演练中没有被等待（约 9.5 秒即形成 D8），是上游缺陷 ora-space/circus#391（现已由 cloud#118 修复，但前两轮未重跑窗口），该轮结果不证明窗口条件；也不是旧版本部署滚动升级的证明。Node 带未被 Cloud 接收终态重连时被拒（ora-space/circus#392），演练因此先确认交付已接收再断线。第二轮没有重跑 D8；Cloud 直接测试仅证明身份、幂等与历史保护规则，不能代替新版候选恢复后真实接管、保留卷再显式删除的联合路径，因此前两轮时完整义务为 `Partial`。第三轮 `p343-run02` 在上述 pinned candidate、driver `b87d30b`（proxy `7e08dc6`）上以真实 30m 默认窗口完成 `legacy-d8`，见[固定证据](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)：Node `last_seen_at=10:04:51.215898Z` 到 Run `updated_at=10:34:59.158819Z` 为 **1807.9 秒**；`last_seen_at` 是最后记录的活动，`updated_at` 是 D8 事务时间戳，不是精确心跳发送或事务提交时间，Run `completed_at` 也不是 D8 结束时间。首次观察 done 为 10:34:59.659359Z（最后活动起 1808.4 秒上界），真实 idle disconnect 起首次观察 done 为 1806.9 秒。随后原身份接管、双确认到 `terminated_data_retained`、显式 `delete-data` 到 `released` 均通过，ledger 始终为零，原 failed delete operation 和 Run 历史不改写。本义务据此改为 `Covered`，仍不证明旧版本滚动升级。
- 决策依据：D6，不变量 7。

## 跨仓库验证边界

产品实现及演练工具 PR（cloud#112/#116/#117/#127、desktop#710/#712、cluster#35/#36）均已合并；cluster#36 的合并提交为 `7adcb6e`、合入 head 为 `a11e025`，包含 S2b 响应等待修复但未重跑部署演练。上表保留实际运行的历史工具 head。以下 `Partial` 是固定版本的联合验收证据缺口，不表示产品实现仍未合并；不因合并状态升级覆盖结论。

- 第 1、2 步的账本、授权、意图与事实接管义务在 cloud 内可直接验证；它们的 `Covered` 判定不依赖 desktop/cluster。
- Node 侧停止证据未确认的命名原因、等待上限与到期转交由 [Node 恢复截止与停止确认用例](../../node/revision/restore-deadline-and-stop-confirmation.md)登记；本文承接其在残留账本上的命名证据（`stop_unconfirmed`）与后续回收义务，不重复登记。
- 第 3 步的终止完成条件跨 cloud、desktop、cluster：按 #573 的结构性论证、有效旧 epoch / 当前 epoch 单仓测试、真实栈受控回收后许可 trace 与未复活读回三部分取证；不要求生产路径必须命中墓碑 503。三部分任一缺直接证据即 `Partial`。
- 第 4 步的联合故障演练按上表固定评论与候选取证：第一轮包含 S1/S2/S3/原 S4/S2b 及 D8 补跑，第二轮仅 S1，第三轮在单一 pinned candidate 上补齐 S2/S3/D8（[证据](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)）。九项中 **8 项 `Covered`、1 项 `Partial`**：前六项的 Cloud 范围保持 `Covered`，重试与显式升级、D8 身份接管新增完整联合路径证据；逐目标终止与围栏按 #573 改为未复活义务，仍因第 3 部分运行 trace / 时间边界不足而为 `Partial`；cluster#45 的旧 epoch 补测待合并。不把旧候选部署结果自动带到其他版本。
