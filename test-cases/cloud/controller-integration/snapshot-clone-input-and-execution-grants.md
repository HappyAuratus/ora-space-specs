# 快照 clone 输入与执行授权

依据 [快照 clone 输入与执行授权决策](../../../decisions/cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)（`implemented`）。
下文的 D 编号与不变量编号都指这篇 ADR；“契约”指
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)，“Cloud 交付决策”指
[Cloud Revision Git 交付决策](../../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)。

[Revision 恢复契约](revision-restore-contract.md) 与 [Agent 运行执行与上传授权](agent-run-executions-and-thread.md) 中关于 bundle 字段与
下载授权的义务，以本文 `Delivery And Session Inputs Describe Only Git Locations` 为准。证据按 cloud `a9c5f20`、desktop `ff9de8f`
（Controller，路径相对 `apps/ora-controller/`）的直接测试，以及 cluster `b0186a6` 的内部线验收脚本 `scripts/workspace-git-acceptance.py`
（下称“验收脚本”，列出的是它的检查项）登记；没有直接证据的部分如实标为 Partial 或 Missing。

## The Clone Contract Carries Only The Snapshot Input

- 风险：`CloneSpec` 仍能表达外部仓库与分支；`snapshot_ref` 与 `base_commit` 不对应，Node clone 出的不是 Cloud 冻结的那份代码。
- 前置状态：重新生成的 `executions.proto` 与 `internal/controlgrpc`。
- 触发：解码字段 1、2 有值的旧形状；`snapshot_ref` 末段不等于 `base_commit`；`repository_url` 带用户信息、查询、片段或不是 `https`；`workspace_ref` 不在 `refs/heads/workspaces/<id>/` 下或短名为 `HEAD`。
- 必须成立：只有四个字段都合法的输入被接受，落库为 `{kind: "clone", repositoryUrl, snapshotRef, baseCommit, workspaceRef}` 且往返不变；`BRANCH_NOT_FOUND` 不再是合法枚举值，`SNAPSHOT_UNAVAILABLE`、`BASE_COMMIT_MISMATCH` 计入三次尝试。
- 禁止结果：旧形状被接受；`buf breaking` 的豁免在下一次修改 `proto/buf.yaml` 时仍然保留。
- 验证义务与证据：
  - Cloud 解码的形状校验与往返：**Covered** — `internal/controlgrpc/git_codec_test.go` 的 `TestCloneSpecRoundTripsTheSnapshotInput`、
    `TestCloneSpecRejectsMalformedInput`（空地址、`http`、用户信息、查询、片段、快照指向另一提交、快照不在 upstreams 下、
    大写提交、`workspace_ref` 越界、短名 `HEAD`、缺少 Workspace 段、空 `workspace_ref` 均为 `INVALID_ARGUMENT`）。字段 1、2 已
    `reserved`，旧形状在线上解码为空字段，落在“空地址”一项；没有单独构造旧形状的测试。
  - Controller 同样只接受快照输入：**Covered** — `src/cloud/mapping.rs` 的 `frozen_clone_inputs_are_forwarded_verbatim`、
    `clone_inputs_outside_the_contract_are_refused`。
  - 失败码：**Partial** — `src/cloud/mapping.rs` 的 `results_project_onto_the_contract_outcome` 证明 `SNAPSHOT_UNAVAILABLE`、
    `BASE_COMMIT_MISMATCH` 往返不变、`BRANCH_NOT_FOUND` 没有契约值；`integration/clone_snapshot_test.go` 的
    `TestFrozenCloneInputOutlivesSnapshotUpdates` 证明已知失败后同一输入再派发。仍缺：没有测试证明这两种新失败按三次尝试计数。
  - `buf breaking` 豁免只用一次：**Missing** — 这是变更流程上的约束；CI 的 `buf breaking` 只能证明本次豁免生效，无法证明下一次
    修改时移除。
- 决策依据：D1。

## The Controller Forwards The Frozen Clone Input Verbatim

- 风险：Controller 用 Project 与 Workspace 字段拼输入，或在 Cloud 冻结前替 Cloud 判断等待；外部仓库与凭据引用继续流向 Controller。
- 前置状态：operation 冻结了 `request.clone`；另有一个缺少 `request.clone` 或输入不满足 Node 协议的 clone 步骤（缺陷注入）。
- 触发：Controller 领取并派发；Controller 改写输入中的一个字段后 `RecordDispatch`；等待快照的 operation 排在队列里。
- 必须成立：`OperationSnapshot.clone_input` 由 `request.clone` 投影，只在 clone 步骤设置，Controller 原样放进 `RecordDispatch` 与 Node 命令；`OperationProject` 只有 `id`；逐字不同的输入以 `ABORTED`/`CONFLICT` 拒绝；没有输入或输入不合法的 clone 步骤不派发，以 `blocked`、`external_failure` 停下；等待快照的 operation 不交给 Controller。
- 禁止结果：Controller 读取 Project 的仓库字段或 `requested_ref`；契约出现等待快照的 defer 原因；Controller 访问 Git Server 或持有任何 Git 凭据。
- 验证义务与证据：
  - Cloud 投影 `clone_input`，`OperationProject` 只有 `id`，`RecordDispatch` 逐字核对：**Covered** —
    `integration/workspace_operations_grpc_test.go` 的 `TestGRPCControllerDrivesWorkspaceThroughNodeAndClone`（sandbox 步骤的快照没有
    `clone_input`，clone 步骤的 `clone_input` 等于冻结输入；改动字段或 Node 为 `ABORTED`/`CONFLICT`）。
  - Controller 原样转交给 Cloud 与 Node：**Covered** — `tests/workspaces.rs` 的 `the_frozen_clone_input_reaches_cloud_and_the_node_verbatim`；
    验收脚本检查项“快照登记后推进”（`clone_executions.input` 与 `request.clone` 逐字相同）。
  - 没有输入或输入不合法时不派发、停下：**Covered** — `tests/workspaces.rs` 的
    `a_clone_step_without_a_frozen_input_blocks_without_a_dispatch`、
    `an_invalid_frozen_clone_input_blocks_the_clone_step_without_a_dispatch`。
  - 等待快照不经过 Controller：**Covered** — `integration/clone_snapshot_test.go` 的 `TestClaimHandsOutTheNextOperationPastAWaiter`；
    验收脚本检查项“等待不经过 Controller：等待期间 Controller 日志中没有该 operation 的 clone 派发”。
  - Controller 不访问 Git Server、不持有 Git 凭据：**Missing** — 只由 Controller 没有 Git Server 配置项、契约中没有凭据字段这一
    结构保证，没有测试直接证明。
- 决策依据：D2，不变量 1、2、9。

## Only A Clone Result Matching The Base Commit Is Taken Over

- 风险：`CloneReady.commit` 与输入不一致却被接管，交付与续作此后都基于错误的基线提交。
- 前置状态：派发输入的 `base_commit` 为 `C`。
- 触发：Node 报告 `commit = C'`；Controller 重启后经 `RecordQueriedResult` 报告同一结果。
- 必须成立：Controller 不接管、不确认该事件；Cloud 的 `TakeOverNodeEvent` 与 `RecordQueriedResult` 以 `INVALID_ARGUMENT` 拒绝且不写入结果；`advanceClone` 再核对一次。
- 禁止结果：`workspaces.base_commit_id` 被写为 `C'`；结果覆盖已有事实。
- 验证义务与证据：
  - Cloud 两条结果路径拒绝：**Covered** — `integration/workspace_operations_grpc_test.go` 的
    `TestGRPCControllerDrivesWorkspaceThroughNodeAndClone`（两条路径均为 `INVALID_ARGUMENT`，执行仍没有结果）。
  - Controller 不接管：**Covered** — `tests/workspaces.rs` 的 `a_ready_clone_at_another_commit_is_not_taken_over`（Cloud 没有结果，
    Workspace 没有基线、准入关闭）；`src/cloud/mapping.rs` 的 `ready_results_must_sit_at_the_dispatched_base_commit`。
  - `advanceClone` 的再次核对：**Missing** — 两条结果路径已经拒绝不符的结果，没有测试绕过它们直接检验 `advanceClone`。
- 决策依据：D3，不变量 3。

## There Is No Clone Path Outside Workspace Operations

- 风险：`ClaimWork` 的 clone 分支或 Controller 的本机受理入口残留，Node 仍能被要求以部署凭据 clone 外部地址。
- 前置状态：新 Controller 与 Cloud。
- 触发：`ClaimWork`；调用 Controller 的本机受理接口；`RecordDispatch` 指向不是 Workspace operation 的 `operation_id`。
- 必须成立：`ClaimWork` 只返回带 `target` 的 Agent 会话与交付工作；`CloneIntake` 与 `ControllerHandle` 的 `accept_clone`、`operations`、`operation` 不存在；`RecordDispatch` 以 `NOT_FOUND` 拒绝该 `operation_id`。
- 禁止结果：`Watch` 因 clone 请求发出 `WorkAvailable`；Controller 把 clone 输入的工作项派发出去。
- 验证义务与证据：
  - Cloud 侧：**Covered** — `integration/workspace_operations_grpc_test.go` 的 `TestGRPCControllerDrivesWorkspaceThroughNodeAndClone`
    （`ClaimWork` 不提供 clone 工作；指向随机 ID 的 `RecordDispatch` 为 `NOT_FOUND`）；验收脚本检查项“租户级 clone 不存在：
    clone_executions 中没有不属于 Workspace operation 的执行”。
  - Controller 侧：**Partial** — `CloneIntake` 与 `ControllerHandle` 的受理方法已从代码中删除，`src/cloud/claim.rs` 把 clone 与插件
    输入的工作项归为不可派发；没有测试向 Controller 投递 clone 输入或无 `target` 的工作项并断言它留在 Cloud。
  - `Watch` 不为 clone 发出 `WorkAvailable`：**Missing** — 只由租户级 clone 的存储与入口已删除保证，没有直接测试。
- 决策依据：D4，不变量 8。

## Each Sandbox Execution Writes Only What Its Kind Allows

- 风险：clone 执行能写 ref；Agent 会话能写别的 Workspace 的分支或非快进改写；交付执行能写交付 ref 与本 Workspace 分支以外的 ref；结束的执行仍能访问。
- 前置状态：同一 Project 的 Workspace A、B，各有 clone、会话与交付执行；真实 `git` 客户端经 Git Server 访问。
- 触发：clone 执行在 clone 步骤内读与推送，执行有结果或 operation 推进后再读；A 的会话推送 `workspaces/<A>/x`、`workspaces/<B>/main`、upstream ref、非快进；会话结束后推送；A 的交付新建自己的交付 ref、快进 `workspaceRef`、推送别的分支；插件执行访问；旧代次证书配上其余条件都成立的会话执行。
- 必须成立：通过判定的执行都能读整个 Project 仓库；clone 执行只读且只在 operation 处于 clone 步骤、执行无结果时有效（Project 可为 `provisioning`）；会话只能新建或快进 `refs/heads/workspaces/<证书中的 workspaceId>/` 下的分支；交付只能新建 `refs/ora/deliveries/<工作项 ID>` 并快进本 Workspace 的 `workspaceRef`；其余一律以同一个 `403` 拒绝。
- 禁止结果：任何沙盒执行写 `refs/ora/upstreams/**`、别的 Workspace 的分支或表外 ref；旧代次证书获授权。
- 验证义务与证据：
  - clone 执行：**Covered** — `integration/git_server_grants_test.go` 的 `TestGitServerGrantsACloneExecutionReadOnlyDuringItsStep`
    （create_project 期间可读、推送被拒且 ref 不变，执行有结果或 operation 离开 clone 步骤后被拒）、
    `TestGitServerDeniesACloneExecutionItsRuntimeNoLongerAllows`（发起人失去使用权、强停、输入指向别的仓库时被拒）；验收脚本检查项
    “clone 执行的授权”。
  - Agent 会话：**Covered** — `integration/git_server_grants_test.go` 的 `TestGitServerGrantsASessionItsOwnWorkspaceBranches`（可读整个
    仓库、可新建多个本 Workspace 分支；别的 Workspace 分支、upstream ref、交付 ref、`refs/heads/main` 与非快进被拒；旧代次证书与
    会话结束后被拒）；`internal/core/git_access_db_test.go` 的 `TestGitAccessDeniesWhenAnyCurrentFactIsWithdrawn`、
    `TestGitAccessSessionAccessEndsWithTheSession`；验收脚本检查项“会话可读整个仓库”“5b / 写别人的分支”“5f / 执行结束后再推”
    “5e / 旧代次”。
  - 交付执行：**Covered** — `integration/git_server_test.go` 的 `TestGitServerAuthorizesDeliveryPushFromCloudState`（旧代次证书、别的
    交付 ref 被拒；可新建并快进本 Workspace 分支，非快进与其他分支被拒）；`internal/core/git_access_db_test.go` 的
    `TestGitAccessGrantsSessionBranchesAndDeliveryRefsFromCurrentState`、`TestGitAccessNeverWidensBeyondTheCertificateWorkspace`；验收脚本
    检查项“场景 4 / 交付更新 Workspace 分支”。
  - 插件执行：**Covered** — `internal/core/git_access_db_test.go` 的 `TestGitAccessDeniesPluginExecutions`。
- 决策依据：D5、D6，不变量 4–7；契约 D3、D4。

## WorkflowRun Agent Sessions Are Not Granted Git Access

- 风险：只按 IssueRun 写的有效性条件被误套到 WorkflowRun，WorkflowRun 的会话得到未经决策的写权限；或者它的 Workspace 混进目录，让 Agent 误以为可以协作。
- 前置状态：一个执行到 agent 节点的 WorkflowRun，它的 Workspace 与 `agent_session` 执行已登记。
- 触发：该会话执行经 Git Server 读取与推送；同一 Project 的 IssueRun 会话放出时生成 Workspace 目录。
- 必须成立：Git Server 对 WorkflowRun 的会话执行一律 `403`；IssueRun 会话的 Workspace 目录不含 WorkflowRun 的 Workspace。
- 禁止结果：WorkflowRun 的会话读到或写入任何 ref；目录出现没有定义的 `kind`。
- 验证义务与证据：**Covered** — `integration/git_server_grants_test.go` 的 `TestGitServerDeniesAWorkflowRunSession`（同一执行从 IssueRun
  移到 WorkflowRun 后读与推送都被拒，ref 未被创建）；`internal/core/agent_run_session_directory_db_test.go` 的
  `TestSessionWorkspaceDirectoryListsTheProjectNewestFirst`（WorkflowRun 的 Workspace 即使最新也不列入，`kind` 只有 `main`、
  `isolated`、`run`）。
- 决策依据：D5、D7，“本决策未解决的问题”；Cloud 交付决策 D5。

## Delivery And Session Inputs Describe Only Git Locations

- 风险：bundle 字段或下载授权残留，读契约的人以为 bundle 路径仍在；交付输入缺少 Git 位置，Node 不知道推到哪里。
- 前置状态：重新生成的 `agent_executions.proto` 与 `agent_runs.proto`。
- 触发：放出交付与会话工作项；解码缺少 `git_repository_url`、`delivery_ref`、`workspace_ref` 或 `history_key` 的交付输入；交付输入中的前序带 Git 字段；会话前序只带部分 Git 字段；`GrantRevisionUpload` 的 `checksums` 含 JSONL 以外的键；结果带 `BUNDLE_FAILED`。
- 必须成立：交付输入带仓库地址、`refs/ora/deliveries/<工作项 ID>` 与取自 clone 输入的 `workspace_ref`；会话前序三个 Git 字段同时出现；上传授权只签发 `history_key`，其余以 `invalid_upload_checksum` 拒绝；`GrantRevisionDownload` 与 `DownloadGrant` 不存在；`BUNDLE_FAILED` 与 `BRANCH_NOT_FOUND` 都已 `reserved`。
- 禁止结果：Controller 改写或补全仓库地址与 ref；Controller 访问 Git Server。
- 验证义务与证据：
  - 交付输入的 Git 位置与校验：**Covered** — `internal/controlgrpc/git_codec_test.go` 的 `TestDeliverySpecCarriesGitLocations`（缺少或越界的
    仓库地址、交付 ref、`workspace_ref`、`history_key` 均被拒，前序只有两项）；`integration/agent_run_delivery_test.go` 的
    `TestDeliveryExecutionClaimCarriesTheFixedSpecAndMovesNothing`（`workspaceRef` 取自 run Workspace 的 clone 输入）。
  - 结果与枚举中没有 bundle：**Covered** — `internal/controlgrpc/git_codec_test.go` 的 `TestRevisionDeliveredCarriesOnlyTheHistory`；
    `BUNDLE_FAILED`、`BRANCH_NOT_FOUND` 的 `reserved` 与删除的方法由 CI 的 `buf lint` / `buf breaking`（带一次性豁免）约束。
  - 会话前序三个 Git 字段同时出现、交付前序不带 Git 字段：**Partial** — Controller 侧 `src/cloud/agents/mapping.rs` 的
    `prior_revisions_outside_the_git_server_contract_are_refused`、`resumed_sessions_carry_the_prior_revision_and_the_directory_verbatim`，
    `src/cloud/deliveries/mapping.rs` 的 `delivery_inputs_naming_a_prior_fetch_location_are_refused`；Cloud 侧
    `internal/controlgrpc/session_models_restore_test.go` 的 `TestSessionWirePreservesModelBindingAndPriorRevision` 证明三个字段往返不变。
    仍缺：Cloud 解码不单独拒绝部分 Git 字段（见 ADR“与批准版本的偏离”），没有测试证明这样的输入在 `RecordDispatch` 被拒。
  - 上传授权只针对 JSONL：**Covered** — `integration/agent_run_delivery_test.go` 的 `TestGrantRevisionUploadSignsOnlyTheHistoryKey`；
    `tests/workspaces/deliveries.rs` 的 `upload_grants_are_requested_only_for_the_history_key`、`refused_grants_never_fail_the_delivery`。
  - Controller 原样转交并按派发输入核对交付终态：**Covered** — `src/cloud/deliveries/mapping.rs` 的
    `records_rebuild_the_delivery_for_the_target_node`、`terminal_results_must_match_the_dispatched_delivery`。
  - Controller 不访问 Git Server：**Missing** — 与 `The Controller Forwards The Frozen Clone Input Verbatim` 相同，只由结构保证。
- 决策依据：D6、D9，不变量 12–14。

## The Workspace Directory Is Fixed At Release And Never Grants Access

- 风险：目录在会话期间被改写或来自 Node 的请求；目录被当作授权依据，Agent 借目录访问它本不能写的分支。
- 前置状态：同一 Project 有超过 100 个 Workspace，其中有已删除的 run Workspace。
- 触发：放出 IssueRun 会话工作项；Controller 改动目录后 `RecordDispatch`；会话期间新建 Workspace。
- 必须成立：目录按创建时间倒序取最多 100 个，含已删除的 run Workspace、不含已删除的 main 与 isolated Workspace，随固定输入逐字核对；会话期间新建的 Workspace 不出现在目录中。
- 禁止结果：目录影响 Git Server 的任何判定；目录包含其他 Project 或租户的信息。
- 验证义务与证据：
  - 目录的内容、顺序与上限：**Covered** — `internal/core/agent_run_session_directory_db_test.go` 的
    `TestSessionWorkspaceDirectoryListsTheProjectNewestFirst`（另一租户的 Project 不列入，超过 100 个时只留最新的 100 个）；验收脚本
    检查项“Workspace 目录”（含已完成的 run Workspace 及其 Issue 标题与 run ID，`ora-git list` 带上这些信息）。
  - 目录随固定输入原样转交：**Covered** — `src/cloud/agents/mapping.rs` 的
    `resumed_sessions_carry_the_prior_revision_and_the_directory_verbatim`；`internal/controlgrpc/session_models_restore_test.go` 的
    `TestSessionWirePreservesModelBindingAndPriorRevision`。
  - 改动目录后 `RecordDispatch` 被拒、目录不参与授权：**Missing** — 会话输入的逐字核对与授权判定不读目录都没有针对目录的直接测试。
- 决策依据：D7，不变量 11。

## A Resumed Run Combines The Snapshot Clone With The Prior Delivery Ref

- 风险：续作把基线改成前序的最终提交，交付与续作的判定要分情况；两份输入指向不同仓库，恢复拉取了别处的提交。
- 前置状态：同一 Issue 已有一个有交付 ref 与新提交的 Revision；当前快照为 `S`。
- 触发：再次运行该 Issue；checkout 所用 clone 输入的仓库地址与当前配置拼出的不同。
- 必须成立：新 Workspace 的基线与交付输入的 `baseCommit` 都是 `S`；会话前序的仓库地址等于 clone 输入的 `repositoryUrl`，不同时不带 `priorRevision`、记 `repository_changed`；交付输入的前序只有 `revisionId` 与 `finalCommit`。
- 禁止结果：快照输入被替换为前序最终提交；前序的交付 ref 被改写。
- 验证义务与证据：
  - 前序带上 Git 位置且仓库地址等于 clone 输入：**Covered** — `integration/agent_run_resume_test.go` 的
    `TestFollowUpRunResumesTheIssueLatestRevision`；验收脚本检查项“场景 4 / 续作的组合”（新 Workspace 的 `base_commit_id` 是快照提交，
    Agent 启动前 checkout 指向前序最终提交，`priorRevision.gitRepositoryUrl` 与 `request.clone.repositoryUrl` 相同）。
  - 仓库地址不同时从头开始：**Partial** — `integration/agent_run_resume_test.go` 的 `TestRunClonedFromAnotherRepositoryStartsFresh`
    覆盖选取前序时的核对；放出会话工作项时的第二次核对没有直接测试。
  - 交付输入的前序只有两项：**Covered** — `internal/controlgrpc/git_codec_test.go` 的 `TestDeliverySpecCarriesGitLocations`；
    `src/cloud/deliveries/mapping.rs` 的 `delivery_inputs_naming_a_prior_fetch_location_are_refused`。
- 决策依据：D8，不变量 10。
