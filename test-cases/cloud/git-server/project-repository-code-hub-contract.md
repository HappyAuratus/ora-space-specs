# Project 仓库代码中心：授权、ref 规则与两条线交接

依据已批准的 [Git Server 根决策](../../../decisions/cloud/git-server/0-project-code-hub-git-server-over-https.md)与
[代码中心契约决策](../../../decisions/cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)。
下文“决策依据”中的“根决策”指前者，其余 D 编号与不变量编号指契约决策。
本文登记契约本身的义务：Git Server 一侧的授权与 ref 规则、Cloud 的 Binding/快照/Operation 存储与生命周期。
clone 消费快照、Revision 交付、执行器同步与发布的端到端义务，由内部线和外部线的后续决策各自登记。

底座的直接证据在 [cloud#75](https://github.com/ora-space/cloud/pull/75)（落地顺序第 1 步）和
[cluster#26](https://github.com/ora-space/cluster/pull/26)，下文的测试路径都指 cloud 仓库。底座只提供契约和预置工具，
真实同步、真实 clone 尚未接入，相应义务如实标为 `Partial` 或 `Missing`。

## Every Git Request Is Authorized From Current Cloud State On Its Own Listener

- 风险：证书本身被当作授权，或请求借用另一个监听的上下文头，让沙盒摸到执行器的写权限；Cloud 不可用时放行。
- 前置状态：三个监听（仓库、服务仓库、ref 查询）各接一个网络；沙盒持 git-access 证书，Repo Executor 持独立 CA 的服务证书。
- 触发：带错误证书、缺少或多带上下文头、带另一监听的上下文头访问；推送过程中 Cloud 状态改变；Cloud 数据库不可用。
- 必须成立：每个请求恰好一个与监听匹配的上下文头；请求时和 `pre-receive` 时都按 Cloud 当前状态判定；判定不可用返回 `503`。
- 禁止结果：仅凭证书放行、同时接受两种上下文头、`pre-receive` 沿用请求开始时的判定、Cloud 故障时降级放行。
- 验证义务与证据：**Partial** — `internal/gitserver/service_listener_test.go` 的 `TestListenersRequireTheirOwnIdentityAndContext`、
  `TestServiceListenerRechecksOperationInPreReceive`；`internal/gitserver/service_test.go` 的 `TestPreReceiveRechecksCurrentState` 与
  `TestRepositoryBoundaryResponses`（数据库故障返回 `503`）。
- 仍缺：沙盒网络访问不到服务仓库监听和 ref 查询监听，目前只有 cluster#26 起栈时的手工核验，没有可重跑的自动化证据。
- 决策依据：D2、D3，不变量 1、2、10。

## Writes Follow The Per-Namespace Rule Set

- 风险：沙盒改写快照或其他 Workspace 的分支，执行器写到 Workspace 分支，ref 被删除或回退。
- 前置状态：仓库中已有快照 ref、跟踪 ref、多个 Workspace 分支和交付 ref。
- 触发：沙盒与执行器分别推送新建、快进、非快进、删除、重名快照、跟踪 ref 指向无快照的提交、跟踪 ref 的 expected old 不符。
- 必须成立：快照 ref 和交付 ref 只能新建，快照名必须等于目标提交；Workspace 分支只能新建或快进；跟踪 ref 只能 CAS，新值必须有对应快照；写入只限规则集列出的 ref。
- 禁止结果：任何 ref 被删除；规则集外的 ref 被写入；非快进改写 Workspace 分支；跟踪 ref 领先于快照。
- 验证义务与证据：**Partial** — `internal/gitserver/rules_test.go` 的 `TestCreateOnlyRuleOnSnapshots`、`TestFastForwardRuleOnWorkspaceBranches`、
  `TestTrackingCASRule`；`internal/gitserver/service_test.go` 的 `TestPushRulesRejectEveryOtherUpdate`、
  `TestDeliveryPushCreatesOnlyItsRefAndReplayIsUpToDate`。
- 仍缺：执行授权目前只产出交付规则，Workspace 分支规则由内部线接入；跟踪 ref 的 expected old 由外部线在进入 `running` 前记录，接入后才有端到端证据。
- 决策依据：D3、D4，不变量 3、5、6、7、8。

## Repository Operation Access Follows Operation State And Project Lifecycle

- 风险：已结束、被取消或租约过期的 Operation 仍能写 Git Server；旧执行器借新租约以同一个 ID 写入；删除中的 Project 无法收尾已在运行的同步。
- 前置状态：Project 处于 `provisioning`、`active`、`deleting`；同步与发布 Operation 处于各个状态。
- 触发：执行器以各 Operation ID 访问服务仓库监听；Operation 在推送过程中结束；租约过期后重试。
- 必须成立：只有 `running` 且租约未过期、Project 与 Binding 匹配、Project 未删除的 Operation 获得授权；同步只能写本 Binding 的快照和跟踪 ref，发布在 Git Server 上没有写权限；一个 Operation 至多进入一次 `running`。
- 禁止结果：要求 Project 处于 `active` 而让首次同步或删除中的收尾失败；已结束的 Operation 仍被放行；发布写入任何 ref。
- 验证义务与证据：**Partial** — `integration/git_operation_access_test.go` 的 `TestGitOperationAccessFollowsOperationAndProjectLifecycle`、
  `TestGitServerAuthorizesExecutorSyncFromCloudState`。
- 仍缺：租约字段由外部线添加，"租约过期后同一个 Operation ID 返回 403"和"至多进入一次 `running`"在外部线验收；目前为 **Missing**。
- 决策依据：D5 第 3 项，不变量 4、14。

## Repository Operation Storage Fixes States And Allows One Unfinished Sync

- 风险：两条线各自定义状态；同一 Binding 并发出现两个未结束的同步；终态没有结束时间，删除永远等待。
- 前置状态：新建的 Project 和它的 Binding。
- 触发：建 Project；写入各类型、各状态的 Operation；为同一 Binding 再写一个未结束的同步；写入快照记录。
- 必须成立：建 Project 的同一事务建立 Binding 和一行 `pending`、`project_created` 的同步；状态与终态由 CHECK 约束固定，终态与 `finished_at` 同时出现；每个 Binding 至多一个未结束的同步；快照 ref 名与提交一致且非零。
- 禁止结果：被拒绝的建 Project 留下 Binding 或 Operation；同步出现发布专有状态；第二个未结束的同步写入成功。
- 验证义务与证据：**Covered** — `integration/repository_binding_test.go` 的 `TestCreateProjectRecordsBindingAndInitialSync`、
  `TestRepositoryOperationStorageConstraints`。
- 决策依据：D5 第 2、3 项。

## Project Deletion Cancels Queued Repository Operations And An Aborted Deletion Restores The Owed Sync

- 风险：删除开始后又有新的 Git Server 写入；删除在 Operation 未结束时清理仓库；删除被回滚后，被取消的同步和补同步请求永久丢失。
- 前置状态：Project 有 `pending`/`leased`/`awaiting_approval`/`running`/`uncertain` 的 Operation，Binding 可能带补同步标记。
- 触发：发起删除；Operation 收尾；Node 拒绝 quiesce，Project 回到 `active`。
- 必须成立：进入 `deleting` 的事务取消尚未 `running` 的 Operation 并清除补同步标记；delete_project 等到没有未结束的 Operation；回滚的同一事务补回欠下的同步（新建 `deletion_aborted` 的 `pending` 同步，或在仍有同步未结束时重新置上补同步标记）；create_project 失败后 Project 为 `active`。
- 禁止结果：已在 `running` 的 Operation 被取消；回滚后既没有同步也没有补同步标记；补同步撞上"一个未结束同步"的索引让回滚失败；被取消的发布自动恢复。
- 验证义务与证据：**Covered** — `integration/repository_binding_test.go` 的
  `TestDeletingProjectCancelsQueuedRepositoryOperationsAndWaitsForRunning`、`TestAbortedProjectDeletionRestoresTheOwedSync`、
  `TestFailedProjectCreationLeavesProjectActive`。
- 决策依据：D5 第 3 项。

## The Current Snapshot Is Recorded Only After Verification And Never Leads The Git Server

- 风险：Cloud 登记了 Git Server 上不存在的快照，Workspace clone 失败或拿到错误代码；Cloud 直接写 Git Server。
- 前置状态：Binding 有若干快照 ref 和跟踪 ref；同步 Operation 报告结果。
- 触发：执行器报告同步结果；外部 force-push 回旧提交；登记前发生故障。
- 必须成立：Cloud 经 ref 查询确认快照 ref 和跟踪 ref 都指向报告的提交后才登记；当前快照总是最新一次核验通过的结果，可能落后、永不领先；Cloud 不写 Git Server。
- 禁止结果：未经核验就更新当前快照；当前快照指向不存在或不一致的 ref；Cloud 经任何接口写入 ref。
- 验证义务与证据：**Partial** — 契约断言 `internal/gitpreset/preset_test.go` 的 `TestSnapshotProducesAContractConformingProject`、
  `TestCheckReportsViolations`；ref 查询 `internal/gitserver/rules_test.go` 的 `TestRefQueryCoversEveryNamespaceAndListsByPrefix`、
  `internal/gitserver/service_test.go` 的 `TestRefQueryAcceptsOnlyTheCloudIdentity`；外部 force-push 后旧快照保留
  `internal/gitserver/service_listener_test.go` 的 `TestServiceListenerFollowsExternalForcePush`。
- 仍缺：真实同步的核验与登记由外部线实现，目前只有预置工具写入的数据。
- 决策依据：D5 第 2 项、D6，不变量 12、13。

## Project Repositories Stay Isolated Across Tenants And Projects

- 风险：同一 Git Server 上，一个 Project 的身份读到另一个 Project 或另一个租户的对象。
- 前置状态：两个租户、多个 Project 的仓库在同一仓库卷。
- 触发：以合法身份访问另一个 Project 或租户的仓库路径，或借对象 ID 取对象。
- 必须成立：授权绑定到 `/{tenantId}/{projectId}.git`；跨租户、跨 Project 的请求被拒绝，对象不可达。
- 禁止结果：仓库间共享对象库或 alternates 让对象可达；拒绝响应泄露其他仓库是否存在。
- 验证义务与证据：**Partial** — `internal/gitserver/service_test.go` 的 `TestRepositoryBoundaryResponses`（其他租户的路径返回 `403`）。
- 仍缺：服务仓库监听上的跨 Project 访问，以及按对象 ID 取对象的直接测试。
- 决策依据：D1、D3，不变量 9。

## Repo Executor Egress Stays Within The Deployment Allowlist And Resolver

- 风险：执行器按 Binding 里的地址直连任意目标，被用来探测 cluster、宿主或企业内网；域名在检查之后改指内部地址（DNS rebinding）；
  `git` 自行解析域名绕过检查。
- 前置状态：部署使用 `deployment-egress.json`，分别处于公网模式与企业模式；Binding 的外部仓库地址分别命中、不命中白名单，
  或解析到私网、cluster、宿主、回环地址；resolver 超时或返回畸形响应。
- 触发：同步与发布 Operation 访问外部仓库；连接复用或重新拨号之间 DNS 结果改变；更换出网配置。
- 必须成立：企业模式下只连白名单中精确的 `host:port`；全部解析地址通过检查后，只向已验证的数字地址拨号，TLS 按原 hostname
  校验；只用部署方 resolver 或静态映射，失败不回退；配置在启动时加载，按共享出网决策的重启流程生效。
- 禁止结果：未命中白名单的目标被查询或连接；任一地址落在 cluster、宿主、回环、链路本地仍被连接；回退系统 DNS、环境代理或跟随
  重定向；出网许可被当作仓库或凭据权限。
- 验证义务与证据：**Missing** — 执行器与它的出网接入由外部线实现，需要真实 DNS 与 Git 托管 fixture 的白名单、地址类别、
  rebinding 与 resolver 故障测试。
- 决策依据：契约 D2“执行器出网”，不变量 15；[部署方共享出网决策](../../../decisions/cloud/model-access/20261017-deployment-owned-egress-and-explicit-resolution.md) D1–D6。
