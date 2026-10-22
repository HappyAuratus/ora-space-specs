# Agent Run 残留资源的对账、受控回收与账本闭环

依据[已实现的残留资源回收决策](../../../decisions/cloud/issue-run/20261018-residual-agent-run-reclamation.md)及[owner 的首版范围与运维授权决定](https://github.com/ora-space/circus/issues/251#issuecomment-6315318982)。下文的 D 编号与不变量编号都指这篇 ADR；“启动决策”指[有界启动与未派发撤回决策](../../../decisions/cloud/issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)，它的核心用例见[有界启动用例](bounded-agent-startup.md)。

本文覆盖 D4/D5 业务收尾之后残留责任的收尾：账本清理状态机、自动对账层的证据与系统清理授权、部署运维的受控回收意图、逐目标终止与 late ensure fencing 的完成条件、三级显式升级，以及存量 D8 删除残留的按身份接管。业务结算、启动与创建截止、迟到事实接管本身的义务仍由[有界启动用例](bounded-agent-startup.md)拥有，本文不重复登记。

回收实现的 PR 为 cloud#112、cloud#116、desktop#710 与 cluster#35；演练工具为 cluster#36。部署证据固定如下，后文“第一轮”“第二轮”均指对应候选，不指可变 PR head：

| 轮次 | 固定演练记录 | 候选与工具版本 | 场景及边界 |
|---|---|---|---|
| 第一轮 | [首轮结果](https://github.com/ora-space/circus/issues/388#issuecomment-6299342004)、[D8 身份接管补跑](https://github.com/ora-space/circus/issues/388#issuecomment-6300622765) | Cloud `59a2443`；desktop `7ea516b`；Sandbox Server 为 cluster#35 `590c444` 加 Node 内存选项（集成提交 `3d9f552`）。工具：S2/S3/S2b 为 `04e2bd8`，S1/原 S4 为 `da3903d`，D8 补跑为 `8640080`。 | S1 通过；S2 约 300 秒不可达后恢复通过，已有卷保留；S3 外部确认通过；S2b inconclusive；原 S4 因 Node 恢复失败未进入接管，D8 补跑仅身份接管通过，已有卷先保留再显式删除。 |
| 第二轮（post-#183） | [S1 重跑](https://github.com/ora-space/circus/issues/388#issuecomment-6308500014) | Cloud `118b759`；desktop `fbe37e24`；cluster main `3e07a4c`；工具 `8209d6b`。 | 只运行 S1，恢复后沿同一意图与两个目标取得双确认，已确认目标不重发，显式 `delete-data` 到 `released`。终止前 Run Workspace 没有数据卷，不能证明已有卷被保留再物理删除。 |

两轮都是单机 Docker 上的真实服务与 Node。S1 的第二个历史 sandbox 身份仅作为输入布置，后续计划、派发、终止与确认来自真实系统，不代表真实多代历史。第一轮 D8 补跑先确认交付已被 Cloud 接收，再经 `ReportNodeIdle` 屏障断线，避开未接收终态重连冲突（ora-space/circus#392）；约 9.5 秒即形成 D8，未遵守默认 30 分钟窗口（ora-space/circus#391，现已由 ora-space/cloud#118 修复，但两轮均未验证修复后的窗口）。它不证明窗口条件或旧版本滚动升级。第二轮没有重跑 S2、S3、S4、S2b 或 D8 接管，第一轮结果只适用于第一轮候选，不能作为最终 PR head 的部署证据。

Cloud 直接测试与部署证据分开判断：复核基线为 cloud#116 `6257136bfc164fc39a6c0a0a87e73bd173425e23`，已合入 cloud#112 及后续 main；下文 Cloud 测试见该提交的 [core](https://github.com/ora-space/cloud/tree/6257136bfc164fc39a6c0a0a87e73bd173425e23/internal/core)、[integration](https://github.com/ora-space/cloud/tree/6257136bfc164fc39a6c0a0a87e73bd173425e23/integration) 与 [cloudctl](https://github.com/ora-space/cloud/tree/6257136bfc164fc39a6c0a0a87e73bd173425e23/cmd/cloudctl)。它们在第二轮 Cloud `118b759` 已存在，向前合并后保留相关断言；前六项的 Cloud 范围由这些直接测试支撑，不把第一轮物理结果升级为新版结果。物理成功在这些测试中由提交证据驱动，不是实际容器或卷的证明；跨进程与跨仓库的完整义务不能由 Cloud 单仓测试单独记为 `Covered`。沿用该提交[已有 CI 记录](https://github.com/ora-space/cloud/actions/runs/39556403820)中相关测试的通过结果；该 CI 整体因 integration 包超时失败，本次没有重跑测试。迁移编号按该基线为 residual `0057`、controlled `0058`。单机演练不覆盖网络与崩溃的全部交错、断电、多 Controller 并发或非 Docker 的生产 Substrate。

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

- 风险：用 Substrate 查询 `404` 或 Node 不可达代替终止证据，让仍在网络中的 ensure 之后创建出可写实例。
- 前置状态：账本 `responsibility` 记录多个沙盒计划 ID，其中 A 已 ensure 成功、B 已持久计划但请求仍在网络中（查询得到 `404`）、C 从未 ensure；B 的 ensure 持有效许可。
- 触发：回收意图经当前 Controller lease 计划 `sandbox_terminate`；对 A/B/C 逐一终止；terminate 双确认后再送达 B 的迟到 ensure；Controller 许可失效或 Substrate 不可达。
- 必须成立：Controller 在 terminate 前只尝试有界的既有 Shutdown 控制关闭，不为回收新增 EndSession 结束请求，也不以控制关闭响应作为终止前置或双确认替代；Cloud 只为既有 `sandbox_terminate` / `workspace_data_delete` 计划 effect，不新增直连 Substrate 路径；Controller 以当前 lease epoch 与许可执行，许可失效时不发起新动作；目标集合包含从未执行过的 ensure 计划，逐 ID 取得 `terminated` 与 `lateEnsureFenced` 双确认；生产 Substrate 适配器保持 Sandbox Server 已有语义（先墓碑后破坏、幂等、对不存在的目标也确认）；只有 Cloud 在与创建计划/派发互斥的事务中证明目标从未形成计划、许可或在途责任并封闭后续入口，才可记录“无目标待终止”；`404` 既不写墓碑也不排除在途 ensure。
- 禁止结果：以 effect 查询 `404`、Node 不可达或许可过期作为终止或围栏证据；跳过未到达的 ensure 计划 ID；删卷先于全部终止确认；删除原 effect 行、假报 sandbox 缺失或新建 generation 挂载同一可写卷；墓碑后迟到 ensure 仍创建出实例。
- 验证义务与证据：**Partial** — Cloud：cloud#112 的 `TestResidualCleanupRequiresEveryDualTerminationAndRemovedProof`、`TestResidualPartialAndNullScopeTargetsRemainAUnion`，cloud#116 的 `TestControlledResidualEveryPlannedSandbox`、`TestControlledResidualRetainedThenReleasedOverGRPC`、`TestControlledResidualRaceNoTargetProofAndDispatchBothCommitOrders`（从未计划证明与并发计划/派发只有一方取得责任）。Controller：desktop#710 的 `cloud::substrate::put_admission::destructive_put_requires_current_unexpired_permit`、`force_stop::force_stop_preserves_all_cloud_effect_targets_and_evidence`、`force_stop::force_stop_attempts_bounded_control_close_before_terminate`、`force_stop::tombstoned_ensure_does_not_retry_or_change_generation`、`force_stop::absence_and_unavailable_nodes_are_not_termination_confirmation`，对端是假 Cloud/Substrate/Node。Sandbox Server：`sandbox-server/internal/lifecycle` 既有的 `TestTerminateOfUnknownSandboxSucceedsAndTombstones`、`TestTerminateRemovesContainerAndBlocksLateEnsure`，以及 cluster#35 的 `TestTerminateTombstonePrecedesDestroyAndReplayIsIdempotent`、`TestTerminateOfExternallyRemovedContainerReturnsDualEvidence`、`TestEnsureTerminateInterleavingsPreserveTombstoneFence`、`TestPlannedEnsure404ThenTerminateRejectsLateAuthorizedEnsure`、`TestManagedTombstoneClassificationPreservesPermitChecksAndHistory`（fake Engine 与真实临时 journal）。部署演练：第一轮 S1、第二轮 S1 的两个目标逐一取得真实双确认、已确认目标没有再次 PUT；第一轮 S2 约 300 秒内真实 `404` 与 Node 失联都没有让意图完成，第二轮未重跑 S2。缺口：“terminate 双确认后送达的有效迟到 ensure 被墓碑拒绝”在第一轮 S2b `late-ensure-fenced` 中未能经生产路径触发（原 ensure 许可有效期内生产回收资格未成立，结论 inconclusive），目前只有 cluster#35 的单仓测试证明。
- 决策依据：D4，不变量 8。

## Reclamation Intents Retry And Escalate Only On Explicit Operator Decision

- 风险：未完成的意图因超时被判成功，或把未知写成已处置，账本带着虚假证据闭环。
- 前置状态：`terminating` 状态的意图，Controller 失联、Substrate 长期不可达；另一条账本长期停在 `reconciling` 且无新证据。
- 触发：Controller 恢复；Substrate 持续不可达；运维发起外部确认（有意图与无意图两种）。
- 必须成立：自动层按周期重试，已授权的 owned delete 由持久清理责任驱动，复用既有 quiesce/删除步骤与幂等规则，不依赖只扫描 `releasing` 的业务重试循环；受控意图保持未完成即持续对账，恢复后继续；任何超时都不把未确认判为成功；升级只能是三级显式路径（自动对账 → 受控回收 → 外部确认），自动层不发起受控回收也不申请外部确认；外部确认必须先有未完成的回收意图，非空确认说明留存审计，账本以 `externally_confirmed` 闭环。
- 禁止结果：超时自动成功或自动升级；无意图直接销账；外部确认文本写成 `terminated` 或伪造 `lateEnsureFenced`；告警缺失被当作已完成。
- 验证义务与证据：**Partial** — cloud#112 的 `TestResidualClaimRollbackAndRestartKeepOneBoundOperation`、`TestResidualOwnedCleanupRetriesSameFailedEffectAcrossControllerEpoch`、`TestResidualRaceRestartRebuildsTwoStoresWithoutNewResponsibility`；cloud#116 的 `TestControlledResidualQueryAndResume`、`TestControlledResidualExternalRequiresUnfinishedIntent`、`TestControlledResidualExternalDisposition`、`TestControlledResidualRaceExternalAndPhysicalCompletionBothCommitOrders`。第一轮 S1 中 Controller 停止 31 秒后重启，同一根意图与目标继续，已确认目标没有再次 PUT；第一轮 S2 约 300 秒内意图保持 `terminating`、无成功写入、无自动数据删除，恢复后凭真实双确认闭环；第一轮 S3 拒绝无意图、缺目标引用或缺责任转交的请求，有效确认留存主体、理由、逐目标外部引用与责任转交，没有写物理成功证据。第二轮只重跑 S1，确认同一意图恢复、不重发已确认目标；未补证新版候选的 S2/S3。Cloud 直接测试支撑持久责任、重试与外部审计规则，不能代替新版跨仓长期不可达及外部确认的联合路径，因此完整义务降为 `Partial`。演练只证明所测故障时长，不证明无限期失联。
- 决策依据：D5，不变量 6。

## Legacy D8 Residuals Are Taken Over By Identity Without Rewriting History

- 风险：Node 恢复后既有重试自动重新声明删除，或为接管而回填账本、改写历史失败结果。
- 前置状态：存量 D8 残留：Run 已 `done`、`delete_workspace` 以 `node_unavailable` 终结失败、Workspace 行与运行绑定保留、无 `agent_run_residuals` 行；对应 Node 随后恢复可达。
- 触发：Node 恢复；释放扫描周期运行；部署运维按原资源身份发起受控回收；回收完成或外部确认。
- 必须成立：Node 恢复不会让 `done` Run 重新进入只扫描 `releasing` 的既有循环，本 ADR 不新增存量自动扫描；接管以资源身份（Workspace、失败的 delete operation、沙盒计划）为目标，不要求也不伪造账本行；无账本行时由回收意图承载保护与证据；终止仍逐目标双确认，数据删除仍需单独显式命令；完成后由新意图及处置记录引用原失败 operation，原 operation 的 `failed/node_unavailable` 与 Run 的 `status`/`done`/结算来源都不变。
- 禁止结果：回填账本行或改写 `reason` 来源；把历史 `done` Run 改回 `releasing` 借用普通扫描；把原 operation 改成成功；Node 恢复连接被当作清理授权。
- 验证义务与证据：**Partial** — cloud#116 的 `TestControlledResidualLegacyD8`、`TestControlledResidualLegacyD8RetainedThenReleased`、`TestControlledResidualRaceNoTargetProofAndDispatchBothCommitOrders`；cloud#112 的 `TestResidualOrdinaryLifecycleGuardsAndUnrelatedD8StayProtected`（无账本 D8 不被自动补账或重试）与既有 `integration/agent_run_release_giveup_test.go` 的 `TestG032_7_ADoneRunIsNeverRedeclaredOrRewritten`。第一轮 D8 身份接管补跑 `legacy-d8`（工具 `8640080`，`--legacy-fault idle-barrier`）：交付先被 Cloud 接收再断线，自然形成零账本 D8；同一 Node incarnation 恢复后跨两个扫描周期没有新的删除；随后按原 Run、Workspace 与失败 delete 身份经 `cloudctl residual terminate` 取得真实双确认到 `terminated_data_retained`，再独立 `delete-data` 到 `released`；重放仍是同一根意图，原失败 operation、Run 终态与版本始终不变，账本始终无行。边界：根 D8 默认 30 分钟窗口在演练中没有被等待（约 9.5 秒即形成 D8），是上游缺陷 ora-space/circus#391（现已由 cloud#118 修复，但未重跑窗口），本条不证明窗口条件；也不是旧版本部署滚动升级的证明。Node 带未被 Cloud 接收终态重连时被拒（ora-space/circus#392），演练因此先确认交付已接收再断线。第二轮没有重跑 D8；Cloud 直接测试仅证明身份、幂等与历史保护规则，不能代替新版候选恢复后真实接管、保留卷再显式删除的联合路径，因此完整义务降为 `Partial`。
- 决策依据：D6，不变量 7。

## 跨仓库验证边界

产品实现及演练工具 PR（cloud#112/#116/#117/#127、desktop#710/#712、cluster#35/#36）均已合并；cluster#36 的合并提交为 `7adcb6e`、合入 head 为 `a11e025`，包含 S2b 响应等待修复但未重跑部署演练。上表保留实际运行的历史工具 head。以下 `Partial` 是固定版本的联合验收证据缺口，不表示产品实现仍未合并；不因合并状态升级覆盖结论。

- 第 1、2 步的账本、授权、意图与事实接管义务在 cloud 内可直接验证；它们的 `Covered` 判定不依赖 desktop/cluster。
- Node 侧停止证据未确认的命名原因、等待上限与到期转交由 [Node 恢复截止与停止确认用例](../../node/revision/restore-deadline-and-stop-confirmation.md)登记；本文承接其在残留账本上的命名证据（`stop_unconfirmed`）与后续回收义务，不重复登记。
- 第 3 步的终止完成条件跨 cloud、desktop、cluster：Cloud 的计划与“从未计划”证明、Controller 的许可与查询、Sandbox Server 的墓碑与迟到 ensure 拒绝必须同时取证，任何一侧缺失即 `Partial`。
- 第 4 步的联合故障演练按上表固定评论与候选取证：第一轮包含 S1/S2/S3/原 S4/S2b 及 D8 补跑，第二轮仅 S1。九项中前六项的 Cloud 范围保持 `Covered`；逐目标终止与围栏、重试与显式升级、D8 身份接管的完整跨仓义务为 `Partial`，不把旧候选部署结果自动带到最终版本。
