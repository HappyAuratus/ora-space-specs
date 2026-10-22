# Revision 经 Git Server 交付与续作

依据 `implemented` 的 [Cloud Revision Git 交付决策](../../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)。下文的
D 编号与不变量编号都指这篇 ADR；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)。

证据路径相对各产品仓库根目录：cloud 的 `integration/` 测试使用真实 PostgreSQL，ref 查询由 Git Server 替身应答
（`integration/git_native_support_test.go` 的 `gitRefs`），只有 `TestGitServerAuthorizesDeliveryPushFromCloudState` 使用真实
Git Server、真实 git 客户端与真实 ref 查询监听。cluster 的检查项是验收脚本写入证据的检查名开头。端到端验收中的 Agent
是 echo 替身或使用栈内受控 model-fixture 的 OpenCode，真实模型供应商下的交付与续作不在这些证据之内。

## The Delivery Input Fixes The Repository, Delivery Ref And Workspace Ref

- 风险：交付输入由 Controller 或 Node 拼地址，重传时变化；交付 ref 不是本工作项自己的，两次尝试写同一个 ref；`workspaceRef` 与 clone 时冻结的分支不同。
- 前置状态：clone 成功、进入 `delivering` 的运行；配置齐全的 `git_server`。
- 触发：放出交付工作项；Controller 改写输入中的地址后 `RecordDispatch`；交付失败后重试；放出后修改 Git Server 地址并重试交付。
- 必须成立：输入含 `repository_base_url` 加 `/<tenantId>/<projectId>.git` 拼成的地址、`refs/ora/deliveries/<该工作项 ID>`、逐字取自 checkout clone 输入的 `workspaceRef`，`baseCommit` 等于 Workspace 基线；改写的输入被拒；重试的新工作项有新的交付 ref 与 JSONL 键，其余位置不变；地址改变后的新工作项使用新地址。
- 禁止结果：输入含 `bundleKey` 或任何凭据；`revisionRef` 出现在 Git Server 上。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 输入的地址、交付 ref、`workspaceRef`、`baseCommit`、`revisionRef` 与 JSONL 键按 D1 固定，没有 `bundleKey` | cloud `integration/agent_run_delivery_test.go` `TestDeliveryExecutionClaimCarriesTheFixedSpecAndMovesNothing`；`internal/controlgrpc/git_codec_test.go` `TestDeliverySpecCarriesGitLocations` | Covered |
| 重试的新工作项有自己的交付 ref 与 JSONL 键，其余位置不变 | cloud `integration/agent_run_delivery_test.go` `TestFailedDeliveryKeepsDeliveringAndReleasesOneBackoffRetry` | Covered |
| `RecordDispatch` 拒绝改写过的交付输入 | 逐字比较的规则只在会话执行上有直接测试（cloud `integration/agent_run_control_test.go` 中改写会话输入的用例）；没有改写交付输入地址或 ref 的测试 | Partial |
| 修改 Git Server 地址后重试的工作项使用新地址 | 无直接测试 | Missing |
| Git Server 上没有 `refs/ora/revisions/*` | cluster `scripts/cross-line-acceptance.py` 检查项“内部线交付：Revision 已登记…Git Server 上没有 refs/ora/revisions/” | Covered |

### 决策依据

D1，不变量 1、9、11。

## Delivery Is Skipped Without Object Store Or Git Server Configuration

- 风险：缺少配置时仍放出交付工作项，或把未保存伪装成已保存。
- 前置状态：运行进入 `delivering`；分别去掉 `object_store`、`git_server`、两者都去掉。
- 触发：运行推进；重复放出；`git_server` 地址或证书形状非法时启动 Cloud。
- 必须成立：不放出交付工作项，同一事务以 `skipped` 结算并释放，原因依次为 `object_store_unconfigured`、`git_server_unconfigured`，每个运行只记一次，重复放出仍为 `skipped`；形状非法时拒绝启动，启动时不访问网络。
- 禁止结果：登记 Revision；同一运行记录两次跳过。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 没有 Git Server 时以 `git_server_unconfigured` 跳过一次、释放运行、不登记、重复放出仍跳过 | cloud `integration/agent_run_delivery_test.go` `TestDeliveryWithoutAGitServerIsSkipped` | Covered |
| 没有对象存储时跳过一次，跳过与业务钩子同事务回滚与重放 | cloud `integration/agent_hook_atomicity_test.go` `TestUnconfiguredRevisionStorageSkipsOnceAndRollsBackWithBusiness`；未断言原因为 `object_store_unconfigured` | Partial |
| 两者都缺时先记 `object_store_unconfigured` | 无直接测试 | Missing |
| `git_server` 形状非法或材料不能加载时拒绝启动，启动时不访问网络 | 无直接测试（`internal/config/git_server.go` 没有对应测试） | Missing |

### 决策依据

D2，不变量 10。

## A Revision Is Registered Only After The Delivery Ref Is Verified

- 风险：Node 声明的最终提交未经核验就登记；Git Server 短暂不可用被判为核验失败，白白消耗重试；ref 不存在仍登记，下一次续作才失败。
- 前置状态：Node 报告 `delivered` 与 `unchanged`；Git Server 与对象存储可以注入故障。
- 触发：ref 查询返回一致的提交、`404 ref_not_found`、不同的提交、其他 `404`、`5xx`、超时；JSONL 大小不符；结果带 bundle、`bundle_failed` 或 `verification_failed`；`unchanged` 的最终提交既不是基础提交也不是前序最终提交。
- 必须成立：比对在接管前完成且不访问外部；一致时在同一事务登记 `revision_verifications`、带 `delivery_ref` 的 `revisions` 与收据并调用 `deliverySettled`；不存在或不符时接管原事件、记 `verification_failed`、不登记并按放弃窗口重试；不可用时返回 `UNAVAILABLE`，不接管、不确认，重放同一事件后成功。
- 禁止结果：未核验就登记；`workspaceRef` 的快进结果参与核验；不可用被记为 `verification_failed`。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| `delivered` 与 `unchanged` 都只核验本尝试的交付 ref 与 JSONL，一致时同事务登记带 `delivery_ref` 的 Revision 并释放 | cloud `integration/agent_run_delivery_test.go` `TestRevisionDeliveredIsVerifiedRegisteredAndReleasesTheRun`；`integration/git_server_test.go` `TestGitServerAuthorizesDeliveryPushFromCloudState`（真实 Git Server 与 ref 查询）；cluster `scripts/revision-acceptance.py` 检查项“有改动的运行”“无改动的运行” | Covered |
| ref 不存在、指向其他提交或 JSONL 不符时记 Cloud 自己的 `verification_failed`，不登记，重试一次，原 Node 声明保留 | cloud `integration/agent_run_delivery_test.go` `TestRevisionVerificationFailureIsCloudsOwnVerdict`、`TestRevisionVerificationFailsOnAnyDisagreeingEvidence` | Covered |
| Git Server 或对象存储不可用时 `UNAVAILABLE`、不写任何记录，恢复后同一提交成功 | cloud `integration/agent_run_delivery_test.go` `TestDeliveryWithAnUnreachableGitServerIsRefusedTransiently`、`TestDeliveryWithAnUnreachableObjectStoreIsRefusedTransiently` | Covered |
| ref 查询只把 `404 ref_not_found` 当作否定结论，其他应答都不证明任何事 | cloud `internal/gitrefquery/client_test.go` `TestResolveGitRefDistinguishesMissingFromUnavailable` | Covered |
| 与输入矛盾的声明在 ref 查询与对象存储之前被拒，不留记录；越界失败原因（含 `bundle_failed`、`verification_failed`）被拒；结果不带 bundle | cloud `integration/agent_run_delivery_test.go` `TestRevisionResultContradictingItsInputNeverReachesTheObjectStore`、`TestNodeMayNotReportCloudsOwnVerificationVerdict`、`TestNodeMayNotReportTheReservedBundleFailure`；`internal/controlgrpc/git_codec_test.go` `TestRevisionDeliveredCarriesOnlyTheHistory` | Covered |
| 外部核验后再次检查租约与运行状态 | cloud `integration/revision_test.go` `TestRevisionLeaseAndStateAreRecheckedAfterExternalVerification` | Covered |
| `workspaceRef` 的快进结果不参与核验 | 无直接测试：没有在交付 ref 一致而 Workspace 分支未前进时断言照常登记的用例 | Missing |

### 决策依据

D3，不变量 2、3。

## Every Git Revision Has Its Own Delivery Ref And Old Rows Stay Untouched

- 风险：`unchanged` 的 Revision 没有自己的 ref，续作要一跳引用；两次尝试登记同一个 ref；迁移改写旧的 bundle 行。
- 前置状态：迁移前已有三种形状的 bundle Revision 行。
- 触发：执行迁移；登记一个 `delivered` 与一个 `unchanged`；试图写入 bundle 列非空的 Git 行、借用 bundle 的 Git 行、`delivery_ref` 形状非法或重复的行。
- 必须成立：旧行原样通过 `revisions_shape`；新行 `delivery_ref` 非空、形状合法、唯一，bundle 列全空，`bundle_revision_id` 不写入；`unchanged` 也有交付 ref。
- 禁止结果：迁移删除或改写旧行；违反形状的行被写入。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 新登记的 `delivered` 与 `unchanged` 都有自己的 `delivery_ref`，bundle 列全空 | cloud `integration/agent_run_delivery_test.go` `TestRevisionDeliveredIsVerifiedRegisteredAndReleasesTheRun`；`integration/agent_run_resume_test.go` `TestResumedRunWithoutNewCommitsHasItsOwnDeliveryRef`；cluster `scripts/revision-acceptance.py` 检查项“无改动的运行”（交付 ref 形状，未断言 bundle 列） | Covered |
| 数据库拒绝带 bundle 或借用 bundle 的 Git 行、形状非法与重复的交付 ref | cloud `integration/agent_run_resume_test.go` `TestRevisionShapeKeepsGitAndBundleRowsApart` | Covered |
| 迁移后 `revisions_shape` 取代旧约束，重复执行不变 | cloud `integration/migration_upgrade_path_test.go` `TestMigration0033RevisionResumeAppliesFreshAndUpgrades` | Covered |
| 升级前已有的 bundle 行原样保留并通过新约束 | 没有在 0054 之前写入三种 bundle 形状再迁移的测试；`integration/revision_upgrade_test.go` `TestRevisionUpgradePreservesRegisteredInputAndRawNodeEvidence` 只证明升级不改执行输入与 Node 原始证据、不推断核验结论 | Partial |

### 决策依据

D4、D9，不变量 4、5。

## Git Access Follows The IssueRun Phase

- 风险：交付执行在 `delivering` 之外仍能写交付 ref；会话在取消或 Thread 结束后仍能改变 Git Server；WorkflowRun 的会话套用 IssueRun 的条件获授权。
- 前置状态：IssueRun 的会话与交付执行；一个执行到 agent 节点的 WorkflowRun。
- 触发：会话在 `starting`、`running`、请求取消后、Thread `ending` 时访问；交付执行在 `delivering` 与之后访问；交付输入的 `deliveryRef` 不是本工作项的、`workspaceRef` 属于别的 Workspace；WorkflowRun 的会话访问。
- 必须成立：会话只在 `phase ∈ {starting, running}`、无取消请求、Thread 未在结束时获授权，不要求带 `priorRevision`；交付执行只在 `delivering` 时获授权，只能新建本工作项的交付 ref、只能快进本 Workspace 的分支；WorkflowRun 的会话一律 `403`；拒绝的应答一致，数据库故障为不可用。
- 禁止结果：已结束或已取消的会话推送成功；WorkflowRun 的会话读到任何 ref。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 交付执行只在 `delivering` 时获授权，只写本工作项的交付 ref 与本 Workspace 的分支 | cloud `internal/core/git_access_db_test.go` `TestGitAccessGrantsSessionBranchesAndDeliveryRefsFromCurrentState`、`TestGitAccessNeverWidensBeyondTheCertificateWorkspace`、`TestGitAccessDeniesWhenAnyCurrentFactIsWithdrawn`；`integration/git_server_test.go` `TestGitServerAuthorizesDeliveryPushFromCloudState` | Covered |
| 会话在请求取消或 Thread `ending` 后失去访问 | cloud `internal/core/git_access_db_test.go` `TestGitAccessSessionAccessEndsWithTheSession`；cluster `scripts/workspace-git-acceptance.py` 检查项“5f / 执行结束后再推” | Covered |
| 会话在 `starting` 与 `running` 都获授权，不要求 `priorRevision` | cloud `integration/git_server_grants_test.go` `TestGitServerGrantsASessionItsOwnWorkspaceBranches`（`running`、不带前序）；`starting` 阶段没有直接测试 | Partial |
| WorkflowRun 的会话一律 `403` | cloud `integration/git_server_grants_test.go` `TestGitServerDeniesAWorkflowRunSession` | Covered |
| 数据库故障不是拒绝 | cloud `internal/core/git_access_db_test.go` `TestGitAccessDatabaseFailureIsNotADenial` | Covered |

传输中途不复查授权是已知取舍，不作为义务。

### 决策依据

D5，不变量 7。

## Resume Selects Only Revisions With A Delivery Ref And New Commits

- 风险：续作选中只含快照提交的 Revision，把本地分支拨回更旧的提交；地址变化后仍续作，恢复必然失败；一次取消让 Revision 永久不可续作。
- 前置状态：同一 Issue 有旧 bundle 行、最终提交等于基础提交的 Git 行、有新提交的 Git 行、续接后无新提交的 Git 行。
- 触发：再次运行该 Issue；clone 自另一个 Git Server 地址后运行；前序恢复以 `prior_revision_base_unavailable` 或其他原因结束，分别在运行已请求取消与未取消时结算。
- 必须成立：候选只含有交付 ref 且有新提交的最新 Revision，续接后无新提交的 Revision 仍是候选；地址不等于本次 clone 输入时记 `repository_changed` 并从头开始；会话前序带 `{revisionId, finalCommit, baseCommit, gitRepositoryUrl, deliveryRef}`，交付前序只带 `{revisionId, finalCommit}`；`prior_revision_base_unavailable` 且未取消时标记不可续接，已取消或其他原因时不标记。
- 禁止结果：跳过后回退到更早的候选；旧 bundle 行或停在基础提交的行成为候选；查 `bundle_revision_id`。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 会话前序带候选自己的交付 ref、基础提交与等于 clone 输入的仓库地址，首条提问带 `resume` | cloud `integration/agent_run_resume_test.go` `TestFollowUpRunResumesTheIssueLatestRevision`；`internal/controlgrpc/session_models_restore_test.go` `TestSessionWirePreservesModelBindingAndPriorRevision`；cluster `scripts/workspace-git-acceptance.py` 检查项“场景 4 / 续作的组合”（断言 `deliveryRef` 与 `gitRepositoryUrl`，未断言 `baseCommit`） | Covered |
| 交付前序只带 `{revisionId, finalCommit}`；续接后无新提交的 Revision 有自己的交付 ref 并成为下一次的候选 | cloud `integration/agent_run_resume_test.go` `TestResumedRunWithoutNewCommitsHasItsOwnDeliveryRef`、`TestResumedRunWithNewCommitsDeliversItsOwnRef`；cluster `scripts/revision-resume-acceptance.py` 检查项“第二次运行（只续接）”“第三次运行（续接后再 [write]）” | Covered |
| 停在基础提交的 Revision 不是候选 | cloud `integration/agent_run_resume_test.go` `TestRevisionsWithoutNewCommitsOrRefsAreNotResumed` | Covered |
| 没有交付 ref 的旧 bundle 行不是候选 | 无直接测试：上一项测试只构造了停在基础提交的 Git 行，没有写入旧 bundle 行 | Missing |
| clone 自另一个仓库地址时记 `repository_changed` 并从头开始；Project 或外部仓库变化时记 `project_changed` | cloud `integration/agent_run_resume_test.go` `TestRunClonedFromAnotherRepositoryStartsFresh`、`TestRunOfAMovedIssueStartsFresh` | Covered |
| 只有未取消的 `prior_revision_base_unavailable` 标记不可续接，之后的运行以 `resume_refused` 从头开始 | cloud `integration/agent_run_resume_test.go` `TestRestoreFailuresAndTheNextRunSelection` | Covered |
| 跳过时不回退到更早的候选 | 上一项与 `TestRunClonedFromAnotherRepositoryStartsFresh` 只有一个候选，没有构造更早的可选 Revision | Partial |

### 决策依据

D7，不变量 6。

## Public Revision Reads Keep Their Shape

- 风险：Git Revision 的 `changed` 按 bundle 判断而恒为 `false`；仓库地址或交付 ref 进入公开 API；Issue 页仍显示 bundle 大小。
- 前置状态：旧 bundle 行、Git `delivered` 行、Git `unchanged` 行。
- 触发：读取运行与 Issue 页的 Revision。
- 必须成立：投影形状不变；Git 行的 `changed` 取自核验结论，`bundleSize` 为 `null`；旧行照旧；Issue 页不显示大小。
- 禁止结果：公开 API 出现仓库地址、交付 ref、`workspaceRef` 或对象键。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| Git 行的 `changed` 取自核验结论、`bundleSize` 为 `null`，公开读取不泄露私有位置 | cloud `integration/agent_run_delivery_test.go` `TestRevisionDeliveredIsVerifiedRegisteredAndReleasesTheRun`；`integration/agent_run_resume_test.go` `TestPublicRunProjectsResume`；cluster `scripts/revision-acceptance.py` 检查项“有改动的运行”“无改动的运行”（`changed` 与 `deliveryState`） | Covered |
| 没有核验结论的旧行按自有 bundle 判断 `changed` 并返回大小 | 无直接测试 | Missing |
| Issue 页显示短最终提交、不显示 bundle 大小，并区分无改动与续接后无新提交 | cloud `frontend/src/features/issues/thread/run-delivery.test.tsx` 的 “names the short final commit of a Git-delivered changed Revision without any size”、“never shows a bundle size, even for a legacy Revision that reports one”、“says an unchanged Revision saved no file changes”、“says a resumed run without new commits reused the resumed Revision” | Covered |

### 决策依据

D8。

## Upload Grants Cover Only The Session History

- 风险：上传授权仍能签发 bundle 或其他对象的 URL；下载授权残留，旧 Node 继续走 bundle 路径。
- 前置状态：已登记、尚无结果的交付执行。
- 触发：申请上传授权，分别不带校验和、带 JSONL 键的校验和、带其他键的校验和；对象存储缺失或执行已结束时申请。
- 必须成立：只签发 JSONL 键的一条 PUT；其他键的校验和以 `invalid_upload_checksum` 拒绝；契约中不存在下载授权。
- 禁止结果：签发 JSONL 以外的对象；授权写入数据库。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 只签发 JSONL 键，其他键被拒，拒绝的错误码有序 | cloud `integration/agent_run_delivery_test.go` `TestGrantRevisionUploadSignsOnlyTheHistoryKey` | Covered |
| 契约中没有 `GrantRevisionDownload` 与 `DownloadGrant`，删除的字段号与名称保留 | cloud `proto/ora/cloud/internal/v1/agent_executions.proto` 的保留声明由生成代码与 `buf breaking` 豁免约束，没有行为测试 | Partial |
| 交付后对象存储中只有会话 JSONL，没有 bundle 对象 | cluster 验收只读取 `session.jsonl`，没有列举或否定 bundle 对象的检查项 | Missing |

### 决策依据

D6，不变量 8。
