# 回合交付：每轮的交付记录、相对起点的快照、只新建的交付 ref、交付后才进待验收

依据 [回合交付](../../../decisions/cloud/project-git/20261112-single-machine-turn-delivery.md)（`implemented`）。下文的 D 编号与
不变量编号都指这篇决策。

证据读取的是 cloud `7fb7c972`（PR #410 的 head）的 `v2-server/`，路径相对该目录；前端证据以 `frontend-v2/` 开头。
需要 PostgreSQL 的测试在设置 `TEST_DATABASE_URL` 时运行。7 项汇总为 7 Covered / 0 Partial / 0 Missing。

## Every Turn End Produces Exactly One Delivery Record

- 风险：某些回合没有交付，改动随工作目录一起丢失；同一轮被重复交付，出现两条互相矛盾的记录。
- 前置状态：一个对话、一个任务运行和一个带两个 Agent 节点的工作流运行，都在同一个项目里。
- 触发：三种来源各结束一轮，分别以成功、失败、被停止结束；工作流的两个节点依次结束，其中一个是交互节点；交付过程中
  重复收到同一轮的结束通知；一个节点尝试在提示被接收之前就失败。
- 必须成立：每个被接收的回合结束恰好一条回合交付记录，结果是 `delivered`、`empty` 或 `failed`；工作流每个节点尝试
  一条，交互节点每个被接收的提示一条、完成节点不另交付，交付的是运行共用的工作目录；同一工作目录的两次交付按顺序
  进行；提示被接收之前就失败的节点尝试没有交付。
- 禁止结果：失败或被停止的回合没有交付记录；同一轮有两条记录。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 三种来源、三种结局各一条记录，重复通知不产生第二条 | `internal/executionruntime/delivery_test.go` 的 `TestTurnDeliveryOnceForEachSourceAndOutcome`：对话成功、任务失败、工作流被停止各交付一次，交付期间回合终态不可见、对话追加被拒，重复暂存终态不改变记录，镜像里恰好 3 个交付 ref | Covered |
| 工作流按节点尝试和被接收的提示交付，交付共享目录 | `internal/workflowhost/delivery_test.go` 的 `TestSharedWorkspaceAttemptsAndInteractiveTurnsDeliver`：交互节点两个提示各交付一次、完成节点不另交付、后继节点再交付一次，共 3 个 ref；`TestDeliveryFailurePreservesNodeAndRunOutcome` 断言第二个节点的交付同时包含两个节点写的文件 | Covered |
| 同一工作目录的交付按顺序进行 | `internal/executionruntime/delivery_liveness_test.go` 的 `TestPublishDeliverySerializesThroughRecordCommit`：前一次的记录写入被挡住时，后一次不进入 Git；`internal/workspacefs/delivery_test.go` 的 `TestDeliveryWorkspaceSerialization` 断言经符号链接指向同一目录的两次加锁互斥 | Covered |
| 提示被接收之前就失败的节点尝试不交付 | `internal/workflowhost/delivery_test.go` 的 `TestPreAdmissionFailureProducesNoDelivery`（子测试 `unassigned template`、`unknown agent`）：没有交付记录、没有交付义务、镜像里没有交付 ref，节点照旧以原来的错误失败 | Covered |

- 决策依据：D1、D3，不变量 1。

## A Delivery Is A Full Snapshot Against The Session Base And Leaves The Agent's Repository Untouched

- 风险：交付漏掉新文件、带进 Ora 物化的技能；父提交依赖上一次交付，中间一次失败后后面的交付无法单独使用；交付过程改动了
  Agent 的索引、分支或工作目录，或执行了 Agent 配置的 Git 外部程序。
- 前置状态：会话起点提交为 B；Agent 改了已跟踪文件、新建了未跟踪文件、自己提交过一次又 `reset` 回去；工作目录里有 Ora
  物化的技能目录和被 `.gitignore` 忽略的文件；`.git` 分别是目录、gitfile、linked worktree、符号链接，或已被删除。
- 触发：这一轮结束，做交付；随后再做一轮，第一轮的交付之后有一次交付失败。
- 必须成立：交付提交的父提交是 B，树包含所有未被忽略的改动和新文件，不含物化的技能和被忽略的文件；Agent 的索引、`HEAD`、
  分支和工作目录与交付前完全一致，Agent 配置的 filter、hooks、fsmonitor、textconv 不执行；排除文件无法安全读取时交付
  失败；没有 `.git` 时照常交付，技能目录有改动时拒绝交付；树与 B 相同时记为 `empty` 且不写 ref；超过大小上限时记为
  `failed` 并写明“改动过大”，镜像不留下对象；任何一次交付都能单独在 B 上打出完整 patch。
- 禁止结果：交付提交以上一次交付或 Agent 的 `HEAD` 为父；物化的技能进入交付；交付改变 Agent 的 Git 状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 父提交是起点，树完整且排除忽略文件与技能，Agent 状态不变 | `internal/workspacefs/delivery_test.go` 的 `TestDeliveryUsesBaseAndLeavesAgentRepositoryIntact`：Agent 提交后 `reset`、暂存区与工作区不同、`.gitignore` 反向收回技能目录，断言父提交与 Ora 身份、树的内容、Agent 的索引、`HEAD`、ref 和文件不变；`delivery_isolation_test.go` 的 `TestDeliveryTrackedFileNewlyIgnored` | Covered |
| 不执行 Agent 配置的 Git 外部程序 | `internal/workspacefs/delivery_isolation_test.go` 的 `TestDeliveryNoForeignGitCodeRuns`（filter、fsmonitor、hooks、textconv、全局与环境注入配置，stash 与 reflog 不变） | Covered |
| 各种检出布局下技能都被剔除，排除文件不安全时失败 | `internal/workspacefs/delivery_isolation_test.go` 的 `TestDeliveryGitFileExcludesMaterializedSkill`、`TestDeliveryLinkedWorktreeUsesBothExcludeFiles`、`TestDeliverySymlinkedExcludeKeepsMaterializedSkillPrivate`、`TestDeliveryExcludeSymlinkOutside`；`delivery_layout_test.go` 的 `TestDeliveryThroughGitSymlink`；`internal/executionruntime/delivery_liveness_test.go` 的 `TestDeliveryEscapingExcludeIsUnavailable` 断言记为 `delivery_unavailable` 且不写 ref；物化一侧见 `internal/agentruntime/git_exclude_test.go` 的 `TestMaterializerRecordsPrivateSkillInLinkedWorktree`、`TestMaterializerRecordsSkillThroughGitSymlink` | Covered |
| 没有 `.git` 时照常交付，技能目录有改动时拒绝 | `internal/workspacefs/delivery_layout_test.go` 的 `TestDeliveryWithoutGitDirectory`（`.opencode`、`.claude` 两处技能目录各一例，拒绝时不写 ref） | Covered |
| 无改动时记为 `empty` 且不写 ref | `internal/executionruntime/delivery_test.go` 的 `TestUnchangedTurnRecordsEmptyWithoutRef`：真实执行运行时下没有改动的回合记为 `empty`、交付提交为空、镜像里没有交付 ref，终态照常公开；快照层见 `internal/workspacefs/delivery_test.go` 的 `TestEmptyDeliveryWritesNoRef` | Covered |
| 超过大小上限时失败、写明原因、镜像不留对象 | `internal/workspacefs/delivery_test.go` 的 `TestDeliveryObjectCap`；`delivery_isolation_test.go` 的 `TestDeliveryRejectsSparseFileBeforeWritingObjects`、`TestDeliveryRejectsAggregateInputBeforeWritingObjects`、`TestDeliveryKeepsLogicalObjectLimitAfterPreflight`；`internal/executionruntime/delivery_test.go` 的 `TestChatDeliveryFailureKeepsSessionOpen` 断言记录原因为“改动过大” | Covered |
| 每次交付都能单独在起点上应用 | `internal/workspacefs/delivery_test.go` 的 `TestDeliveryPatchAppliesOnBase`；`internal/server/workspace_delivery_test.go` 的 `TestWorkspacePatchAppliesOnBaseOverHTTP`（两次交付各自应用，后一次不混进前一次）；前一次失败后的交付仍包含全部改动见 `internal/workflowhost/delivery_test.go` 的 `TestDeliveryFailurePreservesNodeAndRunOutcome` | Covered |

- gitlink 的记录方式和共享目录取收集时刻的状态沿用 Git 的行为，
  不单列义务。
- 决策依据：D2，不变量 3、4、8。

## A Delivery Ref Is Written Before Its Record And Never Rewritten

- 风险：记录说已交付但 ref 不存在；交付 ref 被改写或删除，验收看到的不是当时的成果；同步时把交付 ref 清掉或覆盖。
- 前置状态：项目镜像里已有快照 ref 和若干交付 ref。
- 触发：写 ref 之后、写记录之前让 Cloud 崩溃；对同一个交付 ID 再写一次；对项目做一次手动同步，包括外部分支被强推的情况。
- 必须成立：结果为 `delivered` 的记录对应的 ref 一定存在并指向记录里的提交；崩溃只留下没有记录引用的 ref，它不出现在
  任何视图里；已存在的交付 ref 不能被改写；同步前后所有交付 ref 不变。
- 禁止结果：有记录无 ref；交付 ref 被改写、删除，或被同步动过。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 先写 ref 再写记录，崩溃留下的 ref 不进视图，重启只补一次 | `internal/executionruntime/delivery_test.go` 的 `TestDeliveryRefBeforeRecordCrashAndRestart`：写 ref 后立即停止服务，视图里没有结果、镜像里有 ref；两次启动补交付只产生一条 `delivered` 记录，原来的孤儿 ref 保留，已确认的回合结果不丢；工作目录登记被替换时已写的 ref 成为孤儿、记录为失败见 `delivery_liveness_test.go` 的 `TestDeliveryRegistryReplacementOrphansRef` | Covered |
| 交付 ref 只新建 | `internal/workspacefs/delivery_test.go` 的 `TestDeliveryRefIsCreateOnly` | Covered |
| 同步不动交付 ref，含外部分支被强推 | `internal/projectrepos/sync_test.go` 的 `TestSyncAdvancesCurrentSnapshotOnlyOnSuccess`、`TestSyncFollowsForcePushedBranchAndKeepsOldSnapshot`：同步前埋两个交付 ref，同步后名字和 OID 都不变 | Covered |
| 记录与 ref 不一致时不向用户提供错误内容 | `internal/server/workspace_delivery_test.go` 的 `TestBrokenDeliveryRefAnswersWorkspaceUnavailable`：ref 与记录不符时所有读取返回 `503 workspace_unavailable`，不泄露 Git 诊断和路径 | Covered |

- 决策依据：D3，不变量 2、3、5。

## A Task Reaches Review Only After Its Turn Is Delivered

- 风险：任务进了待验收，验收的人却看不到改动；交付失败的任务停在进行中或误进待验收；交付期间有人手动推进任务；
  交付期间的停止请求结束了健康的会话。
- 前置状态：一个指派给 Agent 的任务，处于进行中。
- 触发：这一轮成功结束、交付成功；成功结束但交付失败；没有改动（`empty`）；这一轮被停止；交付进行中有人手动推进任务
  或停止执行；交付记录写不进。
- 必须成立：交付为 `delivered` 或 `empty` 后任务才进入待验收；交付失败时任务回到待处理并记下原因；被停止时照常交付、
  任务回到待处理；交付完成前人工推进返回 `409 run_active`；收尾失败时任务不停在进行中；交付中的停止请求不结束会话。
  对话交付失败时会话不结束；工作流节点交付失败时节点和运行结果不变，运行页显示失败原因和节点。
- 禁止结果：交付完成之前任务进入待验收；交付失败改变工作流节点的结果。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 按交付结果结算任务，交付前人工推进返回 409 | `internal/server/task_runs_test.go` 的 `TestTaskRunDeliveryGatesReviewAndManualStatus`：`delivered`、`empty`、执行失败进入待验收，交付失败、被停止、交付后才到的停止意图回到待处理，交付前结算返回 `ErrRunActive`、HTTP 推进返回 `409 run_active`，重复结算无效；`internal/executionruntime/delivery_test.go` 的 `TestTaskHostUsesDeliveryOutcome` 以真实执行运行时验证交付失败后任务回到待处理 | Covered |
| 收尾失败时回合结束、任务不被卡住 | `internal/executionruntime/delivery_liveness_test.go` 的 `TestDeliveryFinalizationErrorDoesNotStrandTask`（只有交付记录写不进、失败记录也写不进两种情况，后者终态为 `failed / delivery_unavailable` 并保留交付义务）、`TestDeliveryRetriesTransientRecordWrite` | Covered |
| 交付中的停止请求不结束会话 | `internal/executionruntime/delivery_liveness_test.go` 的 `TestCancelDuringStagedDeliveryKeepsSessionOpen`、`TestCancelStageRaceAfterPreRead`；`cancel_finish_test.go` 的 `TestCancelWhileFinishingReturnedTurnKeepsSession` | Covered |
| 对话交付失败时会话不结束 | `internal/executionruntime/delivery_test.go` 的 `TestChatDeliveryFailureKeepsSessionOpen`：回合结果不变、记录“改动过大”，同一会话能接下一轮 | Covered |
| 工作流节点交付失败不改变结果，运行页显示失败的节点 | `internal/workflowhost/delivery_test.go` 的 `TestDeliveryFailurePreservesNodeAndRunOutcome`：节点和运行成功，`deliveryState` 的最近失败指向第一个节点、最近成功指向第二个节点；`frontend-v2/src/components/workspace-panel/workspace-panel.test.tsx` 的 `always reports a workflow run node failure, naming the node, after other nodes delivered` | Covered |

- 决策依据：D5，不变量 6。

## The Change View And Patch Read The Delivery, Not The Working Directory

- 风险：工作目录被清理后改动就看不到了；patch 无法在起点上应用；同团队的人读到别人的对话交付；大 patch 耗尽内存。
- 前置状态：一个任务运行和一个对话各有两次交付，之后工作目录被 `-cleanup` 删除；一个回合正在进行；另有一次交付失败；
  一个本文落地之前开始的旧会话。
- 触发：读取变更视图、单文件 diff 和整份 patch；浏览文件；由对话主人以外的成员读取对话的交付；超过下载上限的 patch；
  并发下载；删除对话后再读取它的交付；读取旧会话的变更视图。
- 必须成立：变更视图显示最近一次 `delivered` 或 `empty` 交付相对起点的改动，回合进行中显示上一次交付并标明本轮进行中；
  工作目录被清理后变更视图和 patch 仍可用，浏览文件返回 `410 workspace_removed`；patch 包含新文件和二进制文件，能在
  起点提交上 `git apply`，空交付返回 `204`；对话的交付只有主人能读，任务和工作流运行的交付团队成员都能读；超过下载
  上限时返回 `413 patch_too_large` 且交付保留；同时最多生成 4 份 patch；删除来源后它的交付不可读；对话和任务只在最近
  一次结果是失败时显示失败；旧会话照旧读工作目录，登记交付之后不再读。
- 禁止结果：登记过交付的会话的变更视图读工作目录；工作目录被清理后变更视图失效；越权读到对话的交付。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 变更视图读最近一次 `delivered`/`empty` 交付，进行中显示上一次并标明 | `internal/server/workspace_delivery_test.go` 的 `TestDeliveryWorkspaceContractAndAuthorization`（首次交付前为空、进行中保留上一次且 `inProgress`、失败后仍读上一次成功、空交付取代前一次、指定 `deliveryId` 的 diff）、`TestEndedDeliveryWorkDoesNotShowLiveChanges`；`frontend-v2/src/components/workspace-panel/workspace-panel.test.tsx` 的 `says a turn is running and pins diffs to the previous delivery it shows` | Covered |
| 工作目录被清理后仍可读，浏览文件返回 410 | `internal/server/workspace_delivery_test.go` 的 `TestDeliveryViewAfterCleanup` | Covered |
| patch 完整并能在起点上应用，空交付 204 | `internal/server/workspace_delivery_test.go` 的 `TestWorkspacePatchAppliesOnBaseOverHTTP`（修改、新增、二进制、可执行、删除）；`TestDeliveryWorkspaceContractAndAuthorization` 的空交付 `204` | Covered |
| 读取权限与删除来源后不可读 | `internal/server/workspace_delivery_test.go` 的 `TestDeliveryWorkspaceContractAndAuthorization`：成员能读任务和工作流运行、读不到别人的对话（与不存在的对话同样拒绝），交付 ID 不跨来源，删除来源后不可读且交付 ref 不变 | Covered |
| 下载上限与并发上限 | `internal/server/workspace_delivery_test.go` 的 `TestWorkspacePatchDownloadCapRetainsDelivery`、`TestConcurrentPatchBuildsAreBounded`、`TestStalledPatchReadersDoNotHoldEveryBuildSlot`；`delivery_limits_test.go` 的 `TestDeliveryPatchLimitHasACeiling` | Covered |
| 失败的显示规则与旧会话 | `frontend-v2/src/components/workspace-panel/workspace-panel.test.tsx` 的 `reports a conversation failure only while it is the latest outcome, in the server order`、`keeps a legacy working-directory view without delivery notes or download`；`internal/server/workspace_delivery_test.go` 的 `TestDeliveryWorkspaceContractAndAuthorization` 的旧会话部分 | Covered |

- 决策依据：D4、D6，不变量 7。

## Cloud Restart Delivers Interrupted Turns Before Serving

- 风险：重启时正在跑的回合的改动随后被清理掉；补交付变成重做执行；补交付还没完成就开始服务，状态互相覆盖；一个回合
  的补交付失败让整个 Cloud 起不来。
- 前置状态：Cloud 重启前有一个进行中的任务运行、一个进行中的工作流节点和一个进行中的对话回合，其中两个的工作目录还在；
  另一个被中断回合的工作目录已经不在；数据目录里留有上次交付的临时目录。
- 触发：Cloud 重启。
- 必须成立：启动时把未结束的执行标为中断（A1 D3），再为有交付义务的被中断回合各做一次交付，然后才恢复工作流、接手拆解
  任务和开始服务请求；工作目录不在的记为 `failed`；任务回到待处理；不重新执行任何回合；交付的临时目录被清理；一个
  回合写不进记录时继续处理其余回合。
- 禁止结果：重启后重新执行被中断的回合；开始服务之后才补交付；补交付把任务推进到待验收。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 补交付先于工作流恢复和服务请求 | `delivery_startup_test.go` 的 `TestInterruptedTurnsDeliveredBeforeListener`：真实启动流程处理第一个请求时交付已经写入、临时目录已清理；`internal/boot/boot_test.go` 的 `TestRecoverDeliversBeforeWorkflowResumeAndAdmission` 固定启动各步的顺序 | Covered |
| 三类来源各补一次，工作目录不在的记为失败，任务回到待处理，不重做 | `internal/executionruntime/delivery_test.go` 的 `TestInterruptedTurnsAreDeliveredBeforeAdmission`：回合结果保持 `interrupted`，工作目录不在的记 `workspace_removed`，任务回到待处理，再次补交付不产生新 ref；`delivery_liveness_test.go` 的 `TestBootDeliveryWithoutRegisteredPath` | Covered |
| 单个回合写不进记录时继续 | `internal/executionruntime/delivery_liveness_test.go` 的 `TestBootDeliveryContinuesAfterPersistentItemFailure` | Covered |

- 决策依据：D5，不变量 1、6。

## Nodes And Agents Cannot Write Refs Through Any Cloud Entry Point

- 风险：Agent 经 Cloud 的 Git 入口推送，伪造或覆盖交付 ref 和快照 ref。
- 前置状态：一个项目的 Node 持有有效的本机服务凭据；三类来源各有一次交付。
- 触发：Node 经本机服务发起 `git-receive-pack`；对交付相关接口发起写方法请求。
- 必须成立：本机服务只放行只读的 upload-pack，推送被拒绝；交付相关接口只接受读取，任何请求都不改变交付 ref 和交付记录。
  第一版不隔离（A1 D2），Agent 在磁盘上直接改动镜像不在这条承诺之内。
- 禁止结果：经 Cloud 的入口推送成功或改动交付 ref。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 本机服务拒绝推送 | `internal/nodeservices/service_test.go` 的 `TestServicesRequireIdentityAdmissionAndLiveGrant` 检查 `git-receive-pack` 返回 403 | Covered |
| 交付接口只读 | `internal/server/workspace_delivery_test.go` 的 `TestDeliveryEndpointsAreReadOnly`：三类来源的 `changes`、`diff`、`patch` 对 POST、PUT、PATCH、DELETE 都拒绝，GET、HEAD 前后交付 ref 和记录不变 | Covered |

- 决策依据：D3，不变量 9。
