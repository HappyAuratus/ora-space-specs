---
status: implemented
---

# clone 输入替换为 Git Server 快照输入并由 Controller 原样转交；Git Server 按执行类型给出写入规则：clone 只读，Agent 会话写自己的 Workspace 分支，交付另写 `workspaceRef`；会话输入带上 Workspace 目录；交付与会话输入去掉 bundle，删除下载授权

Internal Control 契约中 Workspace clone 的输入由 `CloneSpec{repository, branch}` 直接替换为
[Git Server 代码中心决策](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约 ADR”）D5 第 1 项
的快照输入 `{repository_url, snapshot_ref, base_commit, workspace_ref}`，旧字段删除。Cloud 在 clone 步骤交给 Controller 之前冻结
这份输入，Controller 原样转交，不再用 Project 与 Workspace 的字段拼输入；`CloneReady` 的形状不变，但 `commit` 必须等于
`base_commit`。Git Server 判定沙盒请求时，获授权的执行都能读整个 Project 仓库（契约 ADR D3），本文只规定各执行类型能写什么：
clone 执行只读；Agent 会话可按快进规则写 `refs/heads/workspaces/<自己的 workspaceId>/` 下的分支；交付执行照旧只能新建自己的
交付 ref，另外可以把最终提交快进到本 Workspace 的 `workspaceRef`，使 run 的成果也出现在 Workspace 分支上。会话输入增加一份
Cloud 固定的 Workspace 目录，供 `ora-git list` 把分支与 Issue、run 对上。租户级 clone 与 Controller 本地的 clone 入口删除。
交付与会话输入改为只描述 Git 位置：`DeliverRevisionSpec` 删除 `bundle_key`，增加仓库地址、交付 ref 与 `workspace_ref`；会话的
`PriorRevision` 删除 `bundle`，增加仓库地址、交付 ref 与基础提交；`RevisionDelivered` 删除 `bundle`；`GrantRevisionDownload` 与
`DownloadGrant` 删除，`GrantRevisionUpload` 只签发会话 JSONL 的授权（D9）。

本 ADR 处于 `implemented`：契约在 cloud `a9c5f20` 的 `proto/ora/cloud/internal/v1/{executions,operations,agent_executions,agent_runs}.proto`，
Cloud 一侧在 `internal/controlgrpc`（`execution_codec.go`）与 `internal/core`（`clone.go`、`workspace_clone.go`、`execution_work.go`、
`git_access.go`、`agent_run_session_start.go`、`agent_run_resume.go`、`revision_grants.go`），Controller 一侧在 desktop `ff9de8f` 的
`apps/ora-controller/src/cloud`（`mapping.rs`、`claim.rs`、`operations/steps.rs`、`agents/mapping.rs`、`deliveries/mapping.rs`）；核心测试用例见
[内部线用例](../../../test-cases/cloud/controller-integration/snapshot-clone-input-and-execution-grants.md)。它继承
[Revision 恢复契约决策](20261010-revision-restore-contract.md)、
[Agent 运行执行与上传授权决策](20260928-agent-run-executions-thread-and-upload-grants.md)、
[Workspace 操作与 Controller 代报决策](20260926-workspace-operations-and-controller-reported-nodes.md) 与
[Internal Control 根决策](0-cloud-owned-internal-grpc-contract.md)，承担契约 ADR D3 中“沙盒执行的授权映射”与 D5 第 1 项中
“续作时快照输入与前序交付 ref 怎样组合”，并承担 Revision 改走 Git 后的 Internal Control 契约形状（D9）。它取代
[交付输入去 bundle 决策](20261012-git-delivery-inputs-without-bundle-grants.md)：那份修改说明写在 Git Server 只存交付 ref 的设计上，
随 Git Server 成为 Project 代码中心（[Git Server 根决策](../git-server/0-project-code-hub-git-server-over-https.md)）整体作废，它的契约修改按新设计并入本文 D9。Revision 的业务语义（交付输入的内容、
核验登记、续作选取、Git 授权的运行阶段条件）见
[Cloud Revision Git 交付决策](../revision/20261017-git-native-delivery-refs-and-resume.md)（下称“Cloud 交付决策”）。Cloud 一侧的步骤规则见
[clone 步骤消费快照决策](../operation/20261016-clone-step-consumes-binding-snapshot.md)，Node 一侧见
[Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)。最终实现与批准版本的差异见文末
“与批准版本的偏离”，正文已按最终实现表述。

兼容性边界：**破坏性切换**（契约 ADR 范围边界）。契约直接替换、不保留旧字段，与 D9 的 bundle 删除在同一次变更中完成；
proto `v1` 不再承诺只增不改。Cloud、Controller 与 Node 同时发布，desktop 按 commit 锁定 Cloud 的契约。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：proto 是唯一契约源 | 继承，不变 |
| 20260928 不变量 1：`v1` 已发布的消息与方法不变，`buf breaking` 必须通过 | 修改：本组决策是破坏性切换，`CloneSpec`、交付与会话输入直接替换，`buf breaking` 以一次性 `ignore_only` 豁免（D1、D9） |
| 20260928 D1：`DeliverRevision{session_execution_id, checkout_execution_id, base_commit, revision_ref, bundle_key, history_key}`；`RevisionDelivered` 带 bundle 与 history；`RevisionFailed.reason` 的封闭集合 | 修改：删除 `bundle_key` 与 `RevisionDelivered.bundle`，增加 `git_repository_url`、`delivery_ref`（D9）与 `workspace_ref`（D6）；失败原因增加 `PUSH_REJECTED`、`PUSH_FAILED`，删除 `BUNDLE_FAILED`（D9） |
| 20260928 D4：`GrantRevisionUpload` 为交付输入中的两个对象键签发 PUT | 修改：只为 `history_key` 签发（D9） |
| 20261010 D1：`PriorRevision{revision_id, final_commit, bundle}`，`bundle` 只在会话输入中 | 修改：删除 `bundle`，会话输入中改带 `git_repository_url`、`delivery_ref`、`base_commit`（D9） |
| 20261010 D2：`GrantRevisionDownload` 签发前序 bundle 的预签名 GET | 删除（D9） |
| 20261010 D3、D4：恢复失败以 `AGENT_FAILED` 加 `detail` 报告；`RevisionUnchanged` 可以等于前序最终提交 | 继承，不变；`detail` 的判定见 [Node 交付推送与 Git 恢复决策](../../node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md) D6 |
| 20261010 D5：Controller 中继下载授权；带前序的会话只派给声明 `revision_restore` 的 Node | 修改：删除下载授权中继；能力检查继承（D9） |
| [Controller 会话根决策](../../controller/session/0-controller-relays-agent-sessions.md) D4：交付终态先核对基础提交、ref 与对象键再交给 Cloud | 修改：核对基础提交、`revision_ref` 与 JSONL 键（D9） |
| 根 D4：输入与结果以固定 JSON 形状落库，输入 `{kind, repositoryUrl, branch}` | 修改：Workspace clone 的输入形状为 `{kind, repositoryUrl, snapshotRef, baseCommit, workspaceRef}`（D1） |
| 根 D4：每个请求至多一个执行；结果冲突不覆盖原事实；登记、接管、提交记录同一事务 | 继承，不变 |
| 根 D5：`WorkAvailable` 在 clone 请求接受后发出 | 修改：租户级 clone 删除后只为 Agent 会话与交付工作发出（D4） |
| 根 D8：租户级 clone 的公开入口 | 修改：随 [clone 步骤消费快照决策](../operation/20261016-clone-step-consumes-binding-snapshot.md) D5 删除 |
| [退役过渡 JSON 接口决策](../../controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md) D2、不变量 3：本机受理只经内嵌 `ControllerHandle`／`CloneIntake` | 修改：删除 `CloneIntake` 与 `ControllerHandle` 的受理、列表和查询方法，Controller 不再有本机受理 clone 的入口（D4） |
| 20260926 D3：Workspace clone 的输入必须等于 Project 的 `repositoryUrl` 与 Workspace 的 `requestedRef` | 修改：必须逐字等于 operation 冻结的 `request.clone`（D1） |
| 20260926 D3：结果写入与阶段推进是两次提交 | 继承，不变 |
| 20260926 D3：`ClaimWork` 只返回租户级 clone；Workspace 的 clone 由 `ClaimOperation` 领取 | 修改：`ClaimWork` 不再返回 clone 工作（D4）；Workspace 的 clone 继承 |
| Cloud 交付决策 D1、D7：交付输入带仓库地址、交付 ref 与 `workspaceRef`；会话的 `priorRevision` 带仓库地址、交付 ref 与基础提交 | 继承；本文给出线上形状（D6、D9），会话输入另增加 Workspace 目录（D7），与快照输入的组合见 D8 |
| 20260928 D1、20261010 D1：Controller 原样转交 Cloud 固定的输入 | 继承；Controller 不签发、不持有、不转交任何 Git 访问凭据，并扩展到 clone 输入（D2、D9） |
| [Git Server 根决策](../git-server/0-project-code-hub-git-server-over-https.md) D3：每个请求按身份、声明的上下文与 Cloud 当前状态判定出一份授权，规则集形状由契约 ADR 规定 | 继承；沙盒执行类型到契约 ADR D3 写入规则集的映射由本文给出（D5） |
| Cloud 交付决策 D5：交付执行与 Agent 会话执行的运行阶段条件；所有 Agent 会话都能访问，不要求带前序 | 继承；本文给出各执行类型的写入规则集，clone 执行另有自己的有效性条件（D5） |
| 契约 ADR D4 “沙盒”一列：只能写本 Workspace 分支和当前执行的交付 ref | 继承；D5 的映射不超出这一列 |

## 决策

### D1：`CloneSpec` 直接替换为快照输入

| 消息 | 形状 |
| --- | --- |
| `CloneSpec` | 字段 1、2 与名称 `repository`、`branch` 保留（`reserved`）；`repository_url = 3`（`https://<git-server>/<tenantId>/<projectId>.git`）、`snapshot_ref = 4`（`refs/ora/upstreams/<bindingId>/snapshots/<commitOid>`）、`base_commit = 5`（`snapshot_ref` 指向的提交）、`workspace_ref = 6`（`refs/heads/workspaces/<workspaceId>/<branch>`，`<branch>` 也是 checkout 的本地分支） |
| `CloneReady` | 形状不变；`commit` 必须等于输入的 `base_commit` |
| `CloneFailureReason` | 值 2 与名称 `CLONE_FAILURE_REASON_BRANCH_NOT_FOUND` 保留；0、1、3–5 不变；新增 `SNAPSHOT_UNAVAILABLE = 6`（Git Server 回答快照 ref 不存在或不可读，两者同一回答）、`BASE_COMMIT_MISMATCH = 7`（取得的快照 ref 不指向 `base_commit`） |

- `internal/controlgrpc` 解码 clone 输入时要求四个字段都合法：`repository_url` 只允许 `https`、有主机、不带用户信息、查询与
  片段、以 `.git` 结尾；`snapshot_ref` 在 `refs/ora/upstreams/<id>/snapshots/` 下且末段等于 `base_commit`（完整的小写提交 ID）；
  `workspace_ref` 在 `refs/heads/workspaces/<id>/` 下，分支短名不是 `HEAD`。不合法时为 `INVALID_ARGUMENT`（`INVALID_INPUT`）。
  落库形状为 `{kind: "clone", repositoryUrl, snapshotRef, baseCommit, workspaceRef}`，编解码往返不改变任何字段。
- `RecordDispatch` 对 Workspace clone 要求输入与 operation 的 `request.clone` 逐字相同，`node_id` 是该 Workspace 当前
  generation 的 Node；否则 `ABORTED`/`CONFLICT`。operation 还没有 `request.clone` 时同样冲突。
- `CLONE_FAILURE_REASON_SOURCE_UNAVAILABLE` 的含义改为“Git Server 不可达，或 `5xx` 重试耗尽”。两种新失败与其他已知失败一样计入
  三次尝试（根决策 D2 的规则不变）。
- 与 D9 的删除在同一次契约变更中完成，`buf breaking` 的一次性豁免见 D9。

为什么直接删除旧字段：破坏性切换下没有旧 Controller 或旧 Node 需要兼容；保留一组必须为空的字段，只会让读契约的人以为外部
clone 还在。为什么不新增一个 oneof 成员（例如 `SnapshotCloneSpec`）：clone 只剩一种形状，并存两个成员没有意义。

### D2：Controller 原样转交 clone 输入，不参与等待快照

Internal Control 的 `OperationSnapshot` 增加 `optional CloneSpec clone_input = 10`，由 Cloud 从持久化的 `request.clone` 投影，只在
operation 处于 clone 步骤时设置。`OperationProject` 只保留 `id`，删除 `repository_url`、`default_branch`、`credential_ref` 并保留其
字段编号与名称，外部仓库与凭据引用不再流向 Controller。

- Controller 从 `ClaimOperation` 返回的 operation 中读取 `clone_input`，原样作为 `CloneSpec` 转交给 `RecordDispatch`，再原样
  放进 Node 的 clone 命令，只在旁边附上目标 Node。它不读取 Project 的仓库字段或 Workspace 的 `requested_ref`。
- Controller 不访问 Git Server、不确认快照是否存在、不签发也不转交任何 Git 访问凭据；Node 凭自己的 git-access 证书访问
  Git Server，授权由 Git Server 判定（D5）。这与交付输入的规则相同。
- `ClaimOperation` 只返回已冻结 `request.clone` 的 clone 步骤：输入不能冻结时，Cloud 在领取事务中让 operation 等待、停下或失败，
  并在同一次调用中继续选下一个可领取的 operation（clone 步骤消费快照决策 D1、D2）。`ClaimOperation` 的对外语义（返回最早的
  可领取 operation，没有时返回空）不变，只是等待中的 operation 不算可领取。所以 Controller 不判断等待条件，契约也不增加
  defer 原因。
- 万一领到没有 `clone_input` 的 clone 步骤，或 `clone_input` 不满足 Node 协议（例如分支短名为 `HEAD`、`snapshot_ref` 末段不等于
  `base_commit`），Controller 不登记、不派发，以 `blocked`、`external_failure` 停下：这只会因缺陷发生，重试不会让它变得合法。

为什么由 Cloud 冻结、Controller 只转交：输入来自 Binding 当前快照，这是 Cloud 的业务事实；Controller 只有 operation 的
快照，看不到 Binding。由 Cloud 固定之后，`RecordDispatch` 的逐字校验让 Controller 无法改写输入，重传同一执行时输入不变。

### D3：接管 clone 结果时核对 `base_commit`

- Controller 在 `TakeOverNodeEvent` 与 `RecordQueriedResult` 之前核对 `CloneReady.commit` 等于派发输入的 `base_commit`；不等时
  不接管、不确认该事件，记录错误并按矛盾声明处理（与交付终态的核对同理，D9）。Node 在报告成功前已经校验过
  （Node 从快照 clone 决策），所以这只会因缺陷发生。
- Cloud 的 `TakeOverNodeEvent` 与 `RecordQueriedResult` 同样核对，不等时以 `INVALID_ARGUMENT`（`INVALID_INPUT`）拒绝，不写入结果；
  `advanceClone` 再核对一次（clone 步骤消费快照决策 D3）。
- 为什么三处都核对：基线提交会成为交付的 `baseCommit` 与续作判定的依据，一旦写错，后续每一步都基于错的提交。Node 的
  校验依赖 Node 自己读到的提交，Controller 与 Cloud 的核对只比较固定输入与结果两个字符串，几乎没有成本。

### D4：删除租户级 clone 与 Controller 本地的 clone 入口

- `ClaimWork` 不再返回 clone 工作；剩下的工作项（Agent 会话、交付）都带 `target`，`WorkItem.target` 总是设置。
  `RecordDispatch` 的 `operation_id` 不是 Workspace operation 时为 `NOT_FOUND`；`Watch` 只为 Agent 会话与交付工作发出
  `WorkAvailable`。
- Controller 删除 `ClaimWork` 的 clone 分支：工作项按输入分为会话与交付两类，clone 或插件输入不构成可派发的工作项；没有
  `target` 的工作项留在 Cloud 的队列里。
- Controller 删除本机受理入口：`CloneIntake`（`accept_request`、`operations`、`operation`）及 `ControllerHandle` 上对应的
  `accept_clone`、`operations`、`operation`。它是不接 Cloud 的部署向固定 Node 提交“clone 外部地址的某个分支”的入口。Node 只接受
  Git Server 快照输入以后，本地部署没有 Git Server，这个入口构造不出合法输入；保留它就要在 Node 里保留以部署凭据访问外部
  地址的 clone 路径，违背契约 ADR 需求 3。Desktop 本机模式的用户代码就在本地磁盘上，不需要 Node 代为 clone；它将来若需要从
  远端获取代码，由那时的决策另行设计入口。SQLite 持久适配器的其余职责不变。

### D5：执行类型到写入规则的映射

Git Server 解析沙盒请求时，先确认证书目标 `(tenantId, workspaceId, generation)` 是该 Workspace 当前存活的代次、请求路径中的
Project 是该 Workspace 的 Project，再按声明的执行查出类型。判定在只读的一致快照中进行，不取控制面的锁；任何一道条件不满足
都返回同一个 `403 git_access_denied`，沙盒无法借此得知仓库、执行或 Node 是否存在；数据库故障返回不可用，不当作拒绝。通过判定
的请求可以读整个 Project 仓库（契约 ADR D3），本文只给出契约 ADR D3 的 `Grant.writable`：

| 执行 | 有效性（任一不满足即 `403`） | `writable` |
| --- | --- | --- |
| clone（Workspace operation 的 clone 执行） | `clone_executions` 中属于该 Workspace 的执行，尚无结果，未被强停；所属 operation 处于 `running` 且 `step = clone`；执行的目标 Node 是该 Workspace 当前 Node；operation 的发起人仍能使用该运行时，该 Workspace 没有待处理的强停；Workspace 与 Project 未删除，Project `lifecycle ∈ {provisioning, active}`；输入的 `repositoryUrl` 指向请求的仓库 | 空（只读） |
| IssueRun 的 Agent 会话 | [Cloud 交付决策](../revision/20261017-git-native-delivery-refs-and-resume.md) D5 的共同条件与会话条件：已登记、尚无结果、未被强停；运行 `phase ∈ {starting, running}`，没有取消请求，`thread_state` 不是 `ending` 或 `ended`；维护绑定已确认。不要求输入带 `priorRevision` | `refs/heads/workspaces/<证书中的 workspaceId>/`（前缀），fast-forward |
| 交付 | Cloud 交付决策 D5：运行 `phase = delivering`，`deliveryRef` 等于 `refs/ora/deliveries/<该工作项 ID>` | 输入中的 `deliveryRef`，create-only；输入中的 `workspaceRef`，fast-forward，且只在它属于证书所属 Workspace 的分支前缀时授予（D6） |
| 其他（WorkflowRun 的 Agent 会话、插件执行） | — | 拒绝访问 |

- Agent 会话写前缀里的 `workspaceId` 取自证书，而不是请求或输入；证书目标已经与执行所属的 Workspace 核对过。一个 Agent
  会话可以在自己的前缀下新建多个分支，每个分支只能新建或快进（契约 ADR D4）。
- 交付输入的 `workspaceRef` 不属于证书所属 Workspace 时，交付仍可写自己的交付 ref，只是不获得任何 Workspace 分支的写权限。
- WorkflowRun 的 Agent 会话第一阶段不授权：它没有 IssueRun 的 `phase` 与 `thread_state`，本文也不为它另立有效性条件，Git
  Server 一律 `403`。它的 Agent 仍在本地 checkout 中工作，只是不能经 `ora-git` 推送或拉取。为什么不在本文补上：WorkflowRun
  不交付 Revision，它的 Agent 改了代码要不要推、推到哪个分支、同一 Workspace 上先后多个节点谁能写，都是 workflow 模块的语义；
  在内部线里补上等于替它做决定。`ora-git` 是本组决策新增的能力，不授权不会让 WorkflowRun 现有的行为倒退。
- 读取不区分执行类型与 Workspace：同一 Project 内的 Workspace 分支与交付 ref 对获授权的沙盒执行共享，已删除 Workspace 的
  分支也可读。契约 ADR 已在 Git 层面修改了运行时使用权限决策；界面上谁能看代码与分支详情不变。
- clone 执行的有效性不要求 Workspace 已就绪或准入已开放：clone 正是在准入开放之前发生（根决策 D1），main Workspace 的 clone
  发生在 create_project 期间，此时 Project 还是 `provisioning`。它改为要求 operation 正处于 clone 步骤，这同样随 operation 的推进、
  失败、取消，以及执行有了结果而立即失效。clone 执行按 `clone_executions` 单独判定，会话与交付执行按 `node_executions` 判定。
- 映射不超出契约 ADR D4 的“沙盒”一列：沙盒永远不能写 `refs/ora/upstreams/**`、别的 Workspace 的分支，也不能写表外的 ref。

为什么 Agent 会话的写权限是前缀，而不是只有 `workspaceRef` 一个 ref：Agent 可能需要多个工作分支（例如一个尝试性的
分支）。前缀已经把写入限制在本 Workspace 内，契约 ADR D4 的 namespace 也按前缀定义。代价是本 Workspace 的分支可能多于一个。

为什么所有 Agent 会话都能访问，而不只是带前序的会话：会话需要随时 fetch upstream 与其他 Workspace 的分支来协作。只让带前序的
会话读，前提是“其余会话从用户上游 clone，没有理由访问 Git Server”；Workspace 改为只从 Git Server 快照 clone 之后，这个前提已经
不成立（Cloud 交付决策 D5）。

### D6：交付输入增加 `workspace_ref`，交付执行可把最终提交快进到 Workspace 分支

- `DeliverRevisionSpec` 增加 `workspace_ref = 10`：run Workspace 的分支。Cloud 在放出交付工作项时，从该运行 checkout 所用、已成功
  的 clone 执行输入中取 `workspaceRef` 写入固定输入；`RecordDispatch` 照旧逐字核对。
- 交付执行的写入规则加上这个 ref 的 fast-forward（D5）。Node 先推交付 ref，再尝试快进 `workspaceRef`，规则见
  [Node 交付推送与 Git 恢复决策](../../node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md) D2：Node 不做本地
  祖先检查，非快进由 Git Server 拒绝。快进被拒（Agent 让本地分支偏离了已推送的分支）、旧值不符或网络失败，都不影响交付结果；
  交付的成败与核验只看交付 ref（Cloud 交付决策 D3）。
- 交付结果不增加字段：Workspace 分支是否更新，需要时由 Cloud 经 ref 查询读取（契约 ADR D6）。

为什么交付时更新 Workspace 分支：Agent 不一定记得推送；交付时快进一次，run 的成果就总出现在 Workspace 分支上，其他 Workspace
可以 fetch，发布也可以选它为源。为什么不和交付 ref 原子地一起推：快进可能因为 Agent 改写了本地历史而被拒，原子推送会让这种
情况拖垮整个交付。

### D7：会话输入增加 Workspace 目录，供 `ora-git list` 使用

`AgentSessionSpec` 增加 `repeated WorkspaceListing workspace_directory = 8`。`WorkspaceListing` 的字段为 `workspace_id`、`kind`
（`main`、`isolated` 或 `run`）、`issue_id`、`issue_title`、`issue_run_id` 与 `run_status`；后四项只对 run Workspace 设置，`run_status`
是放出工作项时的状态。

- Cloud 在放出会话工作项时，按创建时间倒序取本 Project 最多 100 个 Workspace 写入固定输入：未删除的 main 与 isolated Workspace，
  以及所有 run Workspace（包括已删除的，它们的分支仍留在 Git Server 上）；WorkflowRun 的 Workspace 与其他 Project、其他租户的
  Workspace 不写入。`RecordDispatch` 照旧逐字核对，Controller 原样转交。
- Node 在 `ora-git list` 中把 Git Server 上的 `refs/heads/workspaces/<workspaceId>/...` 与目录按 `workspaceId` 对上，附带 Issue 与
  run 的信息；不在目录里的 Workspace 只显示 ID。
- 目录是放出工作项那一刻的快照：会话期间新建的 Workspace 没有元数据，run 的状态也可能已经变化。它只用于帮助 Agent 找到
  分支，不参与任何授权；能读哪些分支只由 Git Server 决定（D5）。
- 目录只含同一 Project、同一租户的信息，触发运行的成员本来就能看到这些 Issue。
- WorkflowRun 的 Workspace 不写入目录：它的会话不授权（D5），目录里也没有它的字段。它的分支若存在，`list` 中只显示 ID。

为什么固定在会话输入里，而不是让 Node 运行时向 Cloud 查询：Node 不与 Cloud 通信（运行时 Workspace 根决策 D4）；实时查询要
增加一对 Controller–Node 消息和一个 Cloud 方法，Node 的 `ora-git list` 还要等待 Controller 往返。固定的目录与其他会话输入一样受
`RecordDispatch` 保护，代价是信息可能过时。

### D8：续作时快照输入与前序交付 ref 的组合

续作的 run Workspace 有两份输入，各自固定、各自授权：

| 输入 | 固定时机 | 内容 | Git Server 授权 |
| --- | --- | --- | --- |
| clone 步骤的快照输入 | clone 步骤交给 Controller 之前 | 当时的当前快照 `S` 与新 Workspace 的 `workspaceRef` | clone 执行，只读 |
| 会话输入的 `priorRevision` | 放出会话工作项 | 前序 Revision 的 `{revisionId, finalCommit F, baseCommit B, gitRepositoryUrl, deliveryRef}` | 会话执行，可读整个 Project 仓库 |

- 两者的仓库地址必须相同。Cloud 选取前序时已经要求同一 Project（[IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md)
  D1 第 2 条），两份地址由同一配置与同一 Project 拼成；Cloud 选取前序时核对按当前配置拼出的地址等于 checkout 所用 clone 输入的
  `repositoryUrl`，不同则从头开始、记 `repository_changed`（Cloud 交付决策 D7）；放出会话工作项时再核对一次，不同则不带
  `priorRevision`。
- 顺序：clone 得到 checkout，本地分支指向 `S`；会话开始时 Node 从 `deliveryRef` 恢复 `F`，把本地分支移到 `F`；然后
  Agent 启动。恢复的 Node 一侧规则见 [Node 交付推送与 Git 恢复决策](../../node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md) D5。
- Workspace 的基线提交与交付输入的 `baseCommit` 都是 `S`，不是 `B`；交付结果的判定不变：最终提交等于 `S` 或 `F` 时为
  `unchanged`。交付输入中的 `priorRevision` 仍只有 `{revisionId, finalCommit}`。
- `B` 只用来判断前序是否基于已被外部改写的历史：`B` 不是 `S` 的祖先时，Node 在首条提问末尾提示。`B` 是 `S` 的祖先
  （外部仓库只是前进了）时不提示，Agent 需要的话可以自己 fetch upstream 合并。
- 新 Workspace 的 `workspaceRef` 在续作时并不存在。Agent 推送或交付快进时，Workspace 分支从 `F` 的历史开始新建；前序 Revision
  的交付 ref 不会被改写。

为什么不让快照输入直接改成前序的 `F`：快照输入的语义是“外部仓库同步得到、Cloud 核验过的代码”（契约 ADR D5 第 1 项），
`F` 是平台内部的成果；混在一起会让 `baseCommit` 时而指外部代码、时而指 Agent 的提交，交付与续作的判定都要分情况。
两份输入分开，基线始终是快照。

### D9：交付与会话输入只描述 Git 位置；删除 bundle 字段与下载授权；`buf breaking` 一次性豁免

本节取代交付输入去 bundle 决策，给出 Revision 改走 Git 后 `agent_executions.proto` 与 `agent_runs.proto` 的形状。字段的业务
含义与取值见 Cloud 交付决策 D1、D7。

| 消息 | 形状 |
| --- | --- |
| `PriorRevision` | 字段 3 与名称 `bundle` 保留；`revision_id = 1`、`final_commit = 2`；只在会话输入中设置的 `git_repository_url = 4`、`delivery_ref = 5`、`base_commit = 6` |
| `DeliverRevisionSpec` | 字段 5 与名称 `bundle_key` 保留；`session_execution_id = 1`、`checkout_execution_id = 2`、`base_commit = 3`、`revision_ref = 4`（Node 本地的 `refs/ora/revisions/<runId>`，从不推送）、`history_key = 6`、`prior_revision = 7`（只有 `revision_id` 与 `final_commit`）、`git_repository_url = 8`、`delivery_ref = 9`（`refs/ora/deliveries/<deliveryWorkId>`）、`workspace_ref = 10`（D6） |
| `RevisionDelivered` | 字段 4 与名称 `bundle` 保留；`final_commit = 1`、`base_commit = 2`、`revision_ref = 3`、`history = 5` |
| `RevisionFailureReason` | 新增 `PUSH_REJECTED = 8`、`PUSH_FAILED = 9`；值 4 与名称 `REVISION_FAILURE_REASON_BUNDLE_FAILED` 保留；`VERIFICATION_FAILED` 仍只由 Cloud 记录 |
| `AgentRunService` | 删除 `GrantRevisionDownload` 及其请求、响应与 `DownloadGrant`；`GrantRevisionUpload` 形状不变 |

- 破坏性切换下没有会报 `BUNDLE_FAILED` 的 Node，旧值也不迁移。
- `GrantRevisionUpload` 只签发交付输入中 `history_key` 的授权；`checksums` 含其他键时以 `INVALID_ARGUMENT`
  （`invalid_upload_checksum`）拒绝。
- `internal/controlgrpc` 解码交付输入时要求 `session_execution_id`、`history_key` 非空，`git_repository_url` 是合法的 Git Server
  仓库地址，`delivery_ref` 在 `refs/ora/deliveries/` 下，`workspace_ref` 在 `refs/heads/workspaces/<id>/` 下，不再要求 `bundle_key`。
  会话输入中的 `PriorRevision` 只要带任一 Git 字段，三个 Git 字段就一并落库；它们与 Cloud 固定输入的一致性由 `RecordDispatch` 的
  逐字核对保证（见“与批准版本的偏离”）。
- **Controller**：原样转交交付与会话输入，不改写、不补全仓库地址与 ref，也不需要 Git Server 的任何配置；不签发、不申请、不
  持有、不转交任何 Git 访问凭据，删除下载授权中继。会话输入中的 `prior_revision` 三个 Git 字段缺一或不合法、交付输入中的
  `prior_revision` 带任何 Git 字段时，Controller 拒绝构造 Node 命令。上传授权只为 `history_key` 申请；Node 请求的键含其他对象时不向 Cloud 申请，交付照常进行。
  交付终态在 `TakeOverNodeEvent` 之前核对 `base_commit`、`revision_ref` 与 `history.key` 等于派发输入，`unchanged` 的最终提交等于
  基础提交或输入中前序的最终提交，`delivered` 二者都不等；交付 ref 与最终提交的对应关系只由 Cloud 经 ref 查询核验，Controller
  不访问 Git Server。带 `prior_revision` 的会话仍只派给声明了 `revision_restore` 的 Node。
- **`buf breaking`**：`FILE` 规则下，即使写了 `reserved`，删除字段、消息、方法与枚举值仍被拒绝。`proto/buf.yaml` 按文件一次性
  豁免：`FIELD_NO_DELETE`（`executions.proto`、`operations.proto`、`agent_executions.proto`）、`MESSAGE_NO_DELETE`
  （`agent_executions.proto`、`agent_runs.proto`）、`RPC_NO_DELETE`（`agent_runs.proto`）与 `ENUM_VALUE_NO_DELETE`（`executions.proto`
  的 `BRANCH_NOT_FOUND`、`agent_executions.proto` 的 `BUNDLE_FAILED`）。豁免只针对本次删除，下一次修改 `proto/buf.yaml` 的变更移除
  豁免，此后比较基准已包含本次删除。`reserved` 照样写上，防止字段号与名字以后被赋予别的含义。

为什么不开 `v2`：`buf.yaml` 约定破坏性修改以并列的 `v2` 表达，是为了让旧使用方继续工作。这里的前提不成立：desktop 的
`ora-controller` 是唯一使用方，按 commit 锁定契约、与 Cloud 同时发布；旧 Controller 拿不到仓库地址与交付 ref，无论 `v1` 是否
保留都无法驱动 Git 交付。为什么也删 `RevisionDelivered.bundle` 与 `DownloadGrant`：只保留 Git 交付后它们没有任何合法取值；
留着会让读契约的人以为还存在 bundle 路径。

## 不变量

1. Workspace clone 在 `RecordDispatch` 登记的输入与 operation 冻结的 `request.clone` 逐字相同。
2. Controller 不构造、不补全 clone、交付与会话输入，不访问 Git Server，不持有、不转交任何 Git 访问凭据。
3. 只有 `commit` 等于输入 `base_commit` 的 `CloneReady` 会被接管。
4. clone 执行不能写任何 ref；operation 离开 clone 步骤或执行有了结果，访问立即被拒绝。
5. Agent 会话只能写 `refs/heads/workspaces/<证书中的 workspaceId>/` 下的分支，且只能新建或快进。
6. 交付执行只能新建自己的交付 ref，并快进本 Workspace 的 `workspaceRef`；`workspaceRef` 的更新失败不改变交付结果。
7. 任何沙盒执行的写入都不超出契约 ADR D4 的“沙盒”一列。
8. `ClaimWork` 不返回 clone 工作；Controller 没有本机受理 clone 的入口；clone 只经 Workspace operation 派发。
9. Controller 领到的 clone 步骤总是带有冻结的 `request.clone`；等待快照不经过 Controller。
10. 续作的会话输入与 clone 输入指向同一个仓库；Workspace 的基线提交是快照提交。
11. Workspace 目录不参与授权，只来自放出会话工作项时的 Cloud 记录。
12. 交付与会话输入不含 bundle 的对象键或对象描述；交付输入中的前序只有 `revision_id` 与 `final_commit`。
13. `AgentRunService` 没有下载授权方法；上传授权只针对会话 JSONL 的键。
14. Controller 只按派发输入核对交付终态，从不访问 Git Server。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| Controller 按 operation 快照中的 Project 与 Workspace 字段拼输入（旧做法） | Cloud 不需要冻结；但 Controller 要读 Binding 与当前快照才能拼出来，等于把 Cloud 的业务事实搬进 Controller（D2） |
| 保留 `CloneSpec` 字段 1、2 并标为废弃 | `buf breaking` 不需要豁免；但破坏性切换下没有需要兼容的旧调用方，留下没有合法取值的字段只会误导（D1） |
| 新增 `SnapshotCloneSpec` 作为新的 oneof 成员 | 不需要删除字段；但 `ExecutionInput` 并存两种 clone，所有 clone 分支都要区分（D1） |
| clone 步骤没有输入时交给 Controller，由它以新的 defer 原因 `snapshot_not_ready` 推迟 | 与现有 defer 的写法一致，领取也不需要跳过等待者；但 Controller 只是把 Cloud 已经知道的结论再报回去，契约多一个枚举值，每次等待还多一次领取与 defer 的往返（D2） |
| `CloneReady` 增加 `snapshot_ref` 等字段回显 | 结果自描述；但这些值已在固定输入里，回显只多一处需要核对的冗余（D3） |
| Agent 会话只能写 `workspaceRef` 这一个 ref | 授权最窄；但 Agent 不能开第二个工作分支，契约 ADR D4 也按前缀定义 namespace（D5） |
| Agent 会话可以写本 Project 任意 Workspace 的分支 | 协作更直接；但违背契约 ADR D4“只能写本 Workspace 的分支”，一个 Agent 能推进别人的分支 |
| 交付时不更新 Workspace 分支，只靠 Agent 自己推送 | 交付只写一个 ref；但 Agent 没推送时 run 的成果只在交付 ref 上，协作与发布都要另找（D6） |
| 交付 ref 与 `workspaceRef` 一次原子推送 | 两者总是一致；但 `workspaceRef` 被拒时整个交付失败（D6） |
| Node 在 `ora-git list` 时经 Controller 实时向 Cloud 查询目录 | 信息总是最新；但要新增 Controller–Node 消息与 Cloud 方法，`list` 还要等 Controller 往返（D7） |
| 续作时把快照输入换成前序的最终提交 | 少一步恢复；但基线时而是外部代码时而是 Agent 成果，交付与续作的判定都要分情况（D8） |
| 保留 `GrantRevisionDownload` 与 `RevisionDelivered.bundle` 并标为废弃 | `buf breaking` 不需要豁免；但留下永远拒绝或无人调用的契约，读者以为 bundle 路径仍在（D9） |
| 新开 `v2` 包承载新形状 | 遵循 `buf.yaml` 的惯例，不需要豁免；但唯一使用方同步发布，旧 `v1` 不会再被调用，双包只增加注册与编解码的维护（D9） |
| Controller 按自己的配置注入 Git Server 地址 | Cloud 少一项配置；但地址脱离固定输入，`RecordDispatch` 无法核对，Controller 还要知道 Project 才能拼出仓库路径（D9） |

## 与批准版本的偏离

对照 cloud `a9c5f20` 与 desktop `ff9de8f`，最终实现与本文批准时相比有以下差异，正文已按最终实现表述：

| 批准时的决策 | 最终实现 | 原因 |
| --- | --- | --- |
| `internal/controlgrpc` 解码会话输入时要求 `PriorRevision` 的三个 Git 字段同时出现；交付输入中的前序带 Git 字段时拒绝（列在 Controller 一侧） | Cloud 解码不单独校验：会话前序只要带任一 Git 字段就三项一并落库，交付前序同样按原样解码。缺字段或多字段的输入在 `RecordDispatch` 与 Cloud 固定输入逐字比较时冲突；Controller 构造 Node 命令前拒绝缺字段、不合法的会话前序与带 Git 字段的交付前序（D9） | 固定输入由 Cloud 生成，任何偏离都已被逐字核对挡住；校验集中在 Controller 的映射与 Cloud 的逐字核对两处 |
| Controller 只在领到没有 `request.clone` 的 clone 步骤时以 `blocked`、`external_failure` 停下 | 输入存在但不满足 Node 协议（如分支短名为 `HEAD`、快照 ref 与基础提交不对应）时同样不登记、不派发并停下（D2） | 两者都只会因缺陷发生，重试不会让输入变得合法 |
| `RecordDispatch` 不再接受指向租户级请求的 `operation_id`（未写明错误） | 返回 `NOT_FOUND`（D4） | 租户级请求的存储已删除，这样的 ID 不指向任何可登记的对象 |
| Workspace 目录“按创建时间倒序取本 Project 最多 100 个 Workspace（包括已删除的 run Workspace）” | 明确只列未删除的 main 与 isolated Workspace 和所有 run Workspace；已删除的 main 与 isolated Workspace 不列（D7） | 已删除的 run Workspace 的分支仍是交付与续作的来源，值得列出；其余已删除的 Workspace 对协作没有意义 |
| 交付执行的 `workspaceRef` 写权限“必须在证书所属 Workspace 的分支前缀下” | 输入的 `workspaceRef` 不在该前缀下时，交付照常获授权写自己的交付 ref，只是不获得 Workspace 分支的写权限（D5） | 交付的成败只看交付 ref；Workspace 分支的更新本来就允许失败（D6） |

## 本决策未解决的问题

- **Publish 的源 ref**：Agent 会话能写多个分支后，发布选哪个分支由外部线的发布 ADR 决定；本文只保证它们都在
  `refs/heads/workspaces/<workspaceId>/` 下且只能快进。
- **WorkflowRun 的 Agent 会话的 Git 授权**：cloud 已经为 WorkflowRun 建 Workspace 并派发 `agent_session`，但第一阶段不授权
  （D5），它的会话用 `ora-git` 访问 Git Server 的动词得到 `denied`，Workspace 目录也不含它。要开放时由 workflow 模块的后续 ADR
  给出它的有效性条件（WorkflowRun 与节点的状态）、可写的 ref 与目录字段；写入规则沿用本文的会话映射即可。
- **Workspace 目录的规模**：第一阶段固定取最近 100 个；Project 的 Workspace 很多时怎样检索，留给后续决策。

## 验收

内部线端到端验收不依赖外部线，与 [clone 步骤消费快照决策](../operation/20261016-clone-step-consumes-binding-snapshot.md)、
[Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md) 共用一个栈与一个脚本
（cluster `scripts/workspace-git-acceptance.py`）。下面“以 Node 身份直接访问”指验收脚本进入沙盒、以 Node 用户、用管理 tmpfs
里的 git-access 证书运行 `git`，并在 `Ora-Execution-Id` 中填入从数据库查到的执行 ID，以便在 Node 正常路径之外直接检验 Git Server
的规则。本文承担契约与授权的场景：

| 场景 | 断言 |
| --- | --- |
| 输入原样转交 | `clone_executions.input` 与 `operations.request.clone` 逐字相同；`result.commit` 等于 `baseCommit` |
| 等待不经过 Controller | operation `retry_wait`、`snapshot_not_ready`；`clone_executions` 没有该 operation 的行；等待期间 Controller 日志中没有该 operation 的 clone 派发；快照登记后的执行是第 1 次尝试 |
| clone 执行的授权 | clone 完成后以 Node 身份用该 clone 执行的 ID 对仓库 `ls-remote` 得到 `403`。clone 进行中的授权由 Cloud 的集成测试覆盖 |
| 会话可读整个仓库 | 会话进行中以 Node 身份用会话执行 ID `ls-remote`，列出 `refs/ora/upstreams/**`、`refs/heads/workspaces/**` 与已有的 `refs/ora/deliveries/**` |
| 写别人的分支 | 以 Node 身份用会话执行 ID 推送到另一个 Workspace 的分支被拒，该 ref 不变 |
| 执行结束后再推 | 会话结束后以 Node 身份用会话执行 ID 推送自己的 Workspace 分支得到 `403`，该 ref 不变 |
| 旧代次 | 重启 Workspace 后，旧代次的证书 `ls-remote` 得到 `403` |
| 交付更新 Workspace 分支 | Agent 没有推送的运行完成后，`revisions.delivery_ref` 存在并指向最终提交，`workspaces/<run Workspace>/main` 也指向最终提交 |
| Workspace 目录 | 会话执行输入的 `workspaceDirectory` 含已完成的 run Workspace 及其 Issue 标题与 run ID；`ora-git list` 的输出带上这些信息 |
| 续作的组合 | 新 run Workspace 的 `base_commit_id` 等于它冻结的快照提交；会话输入的 `priorRevision.gitRepositoryUrl` 与 `request.clone.repositoryUrl` 相同；checkout 的本地分支在 Agent 启动前指向前序 `finalCommit` |
| 租户级 clone 不存在 | `clone_executions` 中没有不属于 Workspace operation 的执行 |

## 核心测试用例

本 ADR 的核心测试用例见[内部线用例](../../../test-cases/cloud/controller-integration/snapshot-clone-input-and-execution-grants.md)，
按义务登记 cloud `a9c5f20`、desktop `ff9de8f` 与 cluster `b0186a6` 上的直接证据，缺口如实标为 Partial 或 Missing。
