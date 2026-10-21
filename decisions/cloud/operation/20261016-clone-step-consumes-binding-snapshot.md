---
status: implemented
---

# Workspace 的 clone 步骤改为消费 Binding 当前快照：没有快照时在领取时复核、随同步等待（`snapshot_not_ready`）或失败，Workspace 分支短名取 `Binding.branch`；删除租户级 clone

Workspace 的 `clone` 步骤不再让 Node 从 Project 的外部仓库地址 clone。clone 步骤交给 Controller 之前，Cloud 从该 Project 的
Binding 当前快照生成 [Git Server 代码中心决策](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称
“契约 ADR”）D5 第 1 项的快照输入 `{repositoryUrl, snapshotRef, baseCommit, workspaceRef}`，写入 operation 并冻结；Controller
原样转交，Node 只从 Git Server 读取。Binding 还没有当前快照时，Cloud 在每次领取时复核：最近的同步还没有结束，就以
`retry_wait`、错误码 `snapshot_not_ready` 等待，不登记 clone 执行、不消耗尝试次数；已经结束而仍没有快照，就以
`repository_sync_failed` 失败；任何情况下都不回退为外部 clone。Workspace 的分支短名取 `Binding.branch`，基线提交取快照输入的
`baseCommit`。建 run Workspace 时不隐式同步，直接使用当前快照。删除 Project 时，quiesce 要等该 Project 的同步与发布 Operation
结束。租户级 clone 请求（`clone_requests`）连同其公开接口删除。

本 ADR 处于 `implemented`：实现在 cloud `a9c5f20`（`internal/core/workspace_clone.go`、`internal/core/control.go`、
`internal/core/public.go`、`internal/core/agent_run_preparation.go` 与迁移 `internal/core/migrations/0053_retire_tenant_level_clones.sql`），
与 desktop `ff9de8f`、cluster `b0186a6` 一同落地；核心测试用例见[内部线用例](../../../test-cases/cloud/operation/clone-step-consumes-binding-snapshot.md)。
它继承 [运行时 Workspace 生命周期根决策](0-workspace-runtime-follows-desktop-node.md)、
[具体默认分支决策](20260926-require-concrete-default-branch.md) 与
[plugin 步骤决策](20260928-plugin-step-and-run-workspace-release.md)，承担契约 ADR 中“Workspace 怎样拿到快照输入”的
Cloud 一侧。涉及 Cloud 的 `internal/core`（clone 步骤、领取、Workspace 创建、删除 Project、租户级 clone）、公开 API 的
clones 接口与前端 clones 页。clone 输入在 Internal Control 契约中的形状与 Git Server 的授权映射见
[快照 clone 输入与执行授权决策](../controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)，Node 怎样
clone 见 [Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)。最终实现与批准版本
的差异见文末“与批准版本的偏离”，正文已按最终实现表述。

范围边界：

- Binding、当前快照记录、同步与发布 Operation 由外部线写入。本文只读三项存储：Binding、当前快照，以及同步与发布 Operation
  的最小存储（D2 用它判断等待还是失败，D6 用它判断删除能否继续）。建 Project 的事务同时建立 Binding 和一行 `pending` 的同步
  Operation（契约 ADR）。何时触发同步由[同步决策](../project/20261016-repository-binding-and-snapshot-sync.md)规定，本文不写。
- **破坏性切换**（契约 ADR 范围边界）：不设过渡开关，不做数据迁移。已有的租户级 clone 记录在升级时删除（D5），已有的开发环境
  数据（Project、Workspace）在切换后重建，不承诺可用。
- 公开 API 有意改变：租户级 clones 接口删除（D5），isolated Workspace 的 `baseRef` 增加一条拒绝规则（D3）。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1：create_project / create_workspace 为 sandbox → node → clone → …；start 不 clone；clone 之后（plugin 之后）才开放准入 | 继承，不变。步骤表不变，只改变 clone 步骤的输入来源 |
| 根 D1：delete_project 为 quiesce → terminate → cleanup → done；quiesce 等待各 Workspace 空闲 | 修改：该 Project 还有未结束的同步或发布 Operation 时，Cloud 不把它交给 Controller（D6） |
| 根 D2：clone 阶段派发一次 clone，仓库为 Project 的 `repositoryUrl`，分支为 Workspace 的 `requestedRef` | 修改：输入改为 Binding 当前快照生成的快照输入，交给 Controller 前冻结（D1）；没有快照时在领取时复核，等待或失败（D2） |
| 根 D2：clone 成功的 commit 写入 Workspace 的基线提交 | 修改：基线提交取冻结输入的 `baseCommit`；成功结果的 commit 必须等于它（D3） |
| 根 D2：失败最多三次，前两次 `retry_wait` 5/10 秒；未知结果保持 `blocked`；2026-10-11 检视修订的次数与退避规则 | 继承，不变。`snapshot_not_ready` 不是一次尝试，不计入次数（D2） |
| 根 D2：仓库凭据仍按现有规则，由基础设施注入，Cloud 只提供 credential reference | 修改：Workspace 的 clone 不使用任何外部仓库凭据；Node 凭 git-access 证书访问 Git Server（D1） |
| 根 D2：租户级 clone 请求保持现状，是否退役留给后续决策 | 修改：删除（D5） |
| 根 D6：已有数据与在途操作的迁移规则 | 不适用：本组决策是破坏性切换，不做迁移 |
| 根“未解决的问题”：租户级 clone 请求是否退役、前端 clones 页 | 本文回答（D5） |
| 根“未解决的问题”：isolated Workspace 的工作分支 | 部分回答：每个 Workspace 在 Git Server 上有自己的分支 `refs/heads/workspaces/<workspaceId>/<requested_ref>`（D3）；本地分支的创建见 Node 从快照 clone 决策 |
| 默认分支 D1：Project 必须给出具体默认分支，不能是 `HEAD` | 修改：分支由 Binding 承载，“具体分支、不能是 `HEAD`”的规则落在 `Binding.branch` 上，并要求它是合法的 Git 分支名；Workspace 的分支短名取它（D3） |
| 默认分支 D2：isolated Workspace 的 `baseRef: HEAD` 指 Project 默认分支 | 修改：`HEAD` 指 `Binding.branch`；`baseRef` 的含义改为 Workspace 分支短名，第一阶段只能等于 `Binding.branch`（D3） |
| plugin 步骤 D1：create_project / create_workspace 为 sandbox → node → clone → plugin → done | 继承，不变 |
| plugin 步骤 D4：run Workspace 由 Cloud 以系统身份建删 | 继承；建 run Workspace 不触发同步（D4） |
| 契约 ADR D5 第 1 项：快照输入的四个字段与语义 | 继承，本文生成它（D1） |
| 契约 ADR D5 第 2 项：没有当前快照时 clone 步骤不能继续，也不回退外部 clone | 继承并细化：Workspace 记录照常创建，clone 步骤按 D2 等待或失败 |
| 契约 ADR：clone 步骤在领取时复核快照与最近一次同步 | 继承，本文给出规则（D2） |

## 决策

### D1：clone 步骤交给 Controller 之前，从 Binding 当前快照冻结快照输入

Cloud 在自己的写事务中把快照输入写入 operation 的 `request.clone`，形状为
`{kind: "clone", repositoryUrl, snapshotRef, baseCommit, workspaceRef}`：

| 字段 | 来源 |
| --- | --- |
| `repositoryUrl` | `git_server.repository_base_url`（[Cloud Revision Git 交付决策](../revision/20261017-git-native-delivery-refs-and-resume.md) D2 的同一配置）加 Workspace 的 `tenantId`、`projectId`，即 `<base>/<tenantId>/<projectId>.git` |
| `snapshotRef`、`baseCommit` | 该 Project 的 Binding 当前快照记录（契约 ADR D5 第 2 项）的 `snapshotRef` 与 `commitOid` |
| `workspaceRef` | `refs/heads/workspaces/<workspaceId>/<requested_ref>`（D3） |

- 冻结发生在两个事务里：`node` 步骤推进到 `clone` 的事务；以及处于 `clone` 步骤、尚未冻结的 operation 被领取
  （`ClaimOperation`）的事务。已冻结时不重算。
- **Controller 只会领到已冻结输入的 clone 步骤。** 不能冻结时，Cloud 在领取事务中按 D2 让 operation 等待或失败，或按下文停下，
  不把它交给 Controller，并在同一次领取中继续选下一个可领取的 operation（D2）。领取返回的 operation 快照只在 clone 步骤带上
  冻结的输入。
- **每个 operation 至多冻结一次。** 冻结之后，同一 operation 的每次 clone 执行（包括失败后的新执行）都使用这一份
  输入；之后 Binding 的当前快照更新，不影响已经冻结的 operation。
- 以下情形无法生成输入，operation 以 `blocked` 停下、不自动重试，因为等待不会让它们消失：
  - `git_server` 没有配置：`git_server_unconfigured`，属于部署错误；
  - Project 没有 Binding：`binding_missing`。建 Project 的事务总会建立 Binding，这只可能是缺陷；
  - `workspaceRef` 组不成合法的 Workspace 分支 ref：`invalid_workspace_ref`。建 Project 已拒绝这样的分支（D3），只可能来自
    规则之前写入的数据。
- Workspace 的 clone 不读取 Binding 的外部仓库地址与凭据引用，也不把它们传给 Controller 或 Node。冻结的输入是 Cloud 与
  Controller 之间的数据，公开 API 的 operation 视图不包含 `request.clone`。

为什么在交给 Controller 之前冻结，而不是创建 Workspace 时：

- 创建 Workspace 与进入 clone 步骤之间隔着沙盒启动与 Node 连接，这时读取，得到的是离 clone 最近的快照。
- create_project 在创建 Project 的同一事务里创建 main Workspace，此时第一次同步刚刚建立，还没有快照可以冻结（D2）。

为什么只冻结一次：

- `RecordDispatch` 要求 Controller 转交的输入与 Cloud 固定的逐字相同；输入在 operation 内不变，重试与重传都能用同一条规则核对。
- 快照不可变（契约 ADR D4），失败后用同一快照重试仍然有效；每次重试换快照，会让“第几次失败对应哪份代码”无法回答。

为什么无法生成输入时停下而不是失败：这些情形都不会随时间改变，重试只会重复同一个结论；停下保留人工处理后重试的余地，也避免
一个缺陷让每次领取都失败。

### D2：没有当前快照时，领取时复核：最近的同步没结束就等待，结束了仍无快照就失败

clone 步骤需要冻结输入、而 Binding 没有当前快照时，Cloud 读取该 Binding 最近一次（按创建时间）的同步 Operation：

| 情况 | clone 步骤 |
| --- | --- |
| 有当前快照 | 按 D1 冻结输入，交给 Controller |
| 没有当前快照，最近的同步未结束（`pending`、`leased`、`running`） | 等待：`retry_wait`，`error_code = snapshot_not_ready`，`retry_at` 为 10 秒后 |
| 没有当前快照，最近的同步已经结束（`failed`、`superseded`、`cancelled`；成功却没有快照，或根本没有同步，只可能是缺陷，按同样方式处理） | operation 以 `failed`、`repository_sync_failed` 结束，`result` 只记录该同步 Operation 的 ID（`syncOperationId`），不另存错误码 |

- 判断发生在 D1 的两个事务里：进入 clone 步骤时，以及到期后每次领取时。所以同步失败后，等待者最迟在下一次领取（约 10 秒）
  时结束，外部线不需要在同步收尾时通知内部线。
- 当前快照与最近一次同步在同一条语句里读取。外部线在同一事务里登记成功的同步与它的快照，且不取领取持有的锁；分两条语句
  读取，可能先看到“没有快照”、再看到“同步已成功”，把一个有快照可 clone 的步骤判为失败。一条语句只会看到两者都在或都不在。
- 外部线的自动重放在同步收尾的同一事务里创建新的同步 Operation，“最近一次”随之指向新的 Operation，等待者继续等待。
- 不计入根 D2 的三次尝试（尝试次数按已登记的 clone 执行计数），不设时间上限；唯一没有上限的情形是同步 worker 整体不可用、
  同步停在 `pending` 或 `leased`，与其他 `retry_wait` 的 operation 一样按基础设施故障等待恢复。
- 等待期间不冻结输入，不登记 clone 执行，不派发任何 Node 工作；Workspace 保持 `admission_open = false`。
- 运行公开的 `preparation`（根 D2 的 2026-10-11 修订）把 `snapshot_not_ready` 与 `git_server_unconfigured` 作为安全错误码原样
  展示；clone 步骤以 `repository_sync_failed` 结束时，`preparation.errorCode` 为 `repository_sync_failed`，`preparation.syncErrorCode`
  取所记录同步 Operation 自己的 `error_code`（只在同一 Project 内查找；该同步没有错误码时为空）。两者都不暴露 Binding 或快照
  身份，公开的 operation 视图也不含 `syncOperationId`。
- 以 `repository_sync_failed` 结束的 operation 按 clone 步骤已有的失败路径处理（与第三次已知失败相同）：Workspace 置为
  `unavailable`、准入保持关闭，运行时维护控制归还；run Workspace 走原失败释放路径，项目主 Workspace 留存。
- create_project 以 `repository_sync_failed` 结束时，Project 按契约 ADR D5 第 3 项从 `provisioning` 置为 `active`。用户可以换
  凭据、手动同步，再新建 Workspace，也可以删除 Project；没有快照的 main Workspace 保持准入关闭，不能接受任务。

**领取不因等待者落空。** 领取按创建时间取最早的可领取 operation（`queued`、到期的 `retry_wait`，以及 `running`）。领取事务把
一个 operation 推后（本节、D6）或停下（D1）之后，在同一事务中继续选下一个，直到交出一个或确实没有可领取的；被推后或停下的
operation 离开可领取集合，所以循环一定结束。否则一个排在最前的等待者每 10 秒就让所有其他 operation 空等一轮。

**不回退为外部 clone。** 没有快照时，Node 不会访问外部仓库；Cloud 也不在这里触发同步。

为什么同步未结束时等待，而不是直接失败：新建 Project 时第一次同步与 create_project 同时开始，main Workspace 走到 clone 步骤时
快照往往还没登记。直接失败会让绝大多数新 Project 的第一次创建失败。

为什么按最近一次同步的终态判断，而不是固定的等待时长：首次同步的耗时从几秒到十几分钟不等，固定时长要么让大仓库失败，要么让
凭据错误这类立刻能确定的失败也要等满。同步本身的终态才是“快照会不会出现”的准确答案。

为什么在领取时复核，而不是由同步收尾时结束等待者：复核只读外部线写下的存储，两条线不需要互相调用；代价是同步失败到等待者
结束之间最多多出一个 10 秒的间隔。

为什么由 Cloud 等待，而不是交给 Controller 再 defer：Controller 只能把 Cloud 已经知道的结论再报回去，还要为此在契约里增加一个
defer 原因（快照 clone 输入决策 D2）。

为什么不计入尝试次数：三次预算衡量的是“Node 执行 clone 失败了几次”；等待期间没有任何执行，计入会让一次慢同步耗尽预算。

为什么只记录同步的 ID：错误码属于同步 Operation 本身，复制一份就有两处来源；运行的 `preparation` 按 ID 读回即可。

### D3：Workspace 分支短名取 `Binding.branch`；基线提交取 `baseCommit`

- `workspaces.requested_ref` 取 Project 的 `Binding.branch`（具体分支名，不能是 `HEAD`，第一阶段不允许修改）。建 Project 时该
  分支还必须是合法的 Git 分支名，否则 `400 default_branch_required`，保证 `workspaceRef` 总能组成合法的 ref。
- isolated Workspace 的 `baseRef` 缺省、为空（含仅空白）或为 `HEAD` 时取 `Binding.branch`；等于 `Binding.branch` 时照常；其他值
  返回 `400 base_ref_not_synced`，不写任何记录。
- 含义改变：`requested_ref` 不再是外部仓库中要 clone 的分支，而是本 Workspace 在 Git Server 上的分支短名，
  `workspaceRef = refs/heads/workspaces/<workspaceId>/<requested_ref>`。Node 在 checkout 里以同名本地分支检出快照提交。
- clone 结果只有在 commit 等于该执行输入的 `baseCommit` 时才会被登记（快照 clone 输入决策 D3）；`advanceClone` 再要求最近一次
  执行成功、结果 commit 等于冻结输入的 `baseCommit`、执行输入与冻结输入逐字相同、执行的 Node 仍是该 Workspace 的当前 Node，
  才把它写入 `workspaces.base_commit_id`。任一不符都不推进（`409 clone_incomplete`）。

为什么拒绝别的 `baseRef`：第一阶段每个 Project 只有一个 Binding、同步一个分支（契约 ADR D5 第 2 项），快照只有这个分支的
内容。接受别的名字等于给用户一个名为 `feature` 却装着 `main` 内容的 Workspace，违背
[clone 根决策](../../node/repository/0-clone-selected-repository-branch.md) 不变量 1“不静默用另一个分支替代”。

为什么短名取 `Binding.branch`，而不是 Cloud 另起一个名字（比如 `ora/<workspaceId>`）：`workspaceRef` 已经用
`workspaces/<workspaceId>/` 隔开了不同 Workspace，短名不需要再承担唯一性；与同步的分支同名，Agent 在 checkout 里看到的是熟悉的
`main`，交付、续作与提示里的“本地分支”也指同一个分支。

### D4：建 run Workspace 时不隐式同步，直接使用当前快照

建 run Workspace 与 IssueRun 的派发事务不创建同步 Operation，也不等待同步；run Workspace 的 clone 步骤按 D1 冻结
那一刻的当前快照，按 D2 等待尚不存在的快照。

- 为什么：同步要访问外部仓库，耗时与成败都取决于外部服务和凭据。run 的启动只依赖平台内部的 Git Server，外部仓库不可用时
  run 仍能从最近一次核验过的快照开始；快照的新鲜度由外部线的同步策略负责，而不是由每次 run 承担。
- 代价：run 可能基于稍旧的代码开始。交付的 `baseCommit` 就是这份快照，旧与新的差距在 Publish 时由外部线处理；
  续作时前序 Revision 基于更旧的快照，按 [Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)
  对恢复的修改给出提示。

### D5：删除租户级 clone 请求

- Cloud 没有租户级 clone 的入口与存储：`POST`/`GET /api/v1/tenants/{tid}/clones` 不在路由与 OpenAPI 中（请求得到 `404`），生成的
  前端客户端与前端 clones 页（侧边栏的仓库入口）删除，`clone_requests` 表删除。
- `clone_executions` 只服务 Workspace 的 clone 步骤：每一行都属于一个 Workspace operation，`workspace_id` 不可为空，不再有
  `clone_request_id` 列。
- `ClaimWork` 不返回 clone 工作；`RecordDispatch` 的 `operation_id` 不是 Workspace operation 时为 `NOT_FOUND`（快照 clone 输入
  决策 D4）。
- 升级不迁移已有的租户级 clone：迁移删除不属于 Workspace 的 clone 执行及其回执，再删除 `clone_requests` 表；Workspace 的
  clone 执行与回执保留。

为什么删除而不是改为从 Git Server clone：

- 租户级 clone 的语义是“Node 拿部署凭据访问任意外部地址”，正是契约 ADR 需求 3、4 要去掉的路径。
- 改成从 Git Server clone 需要一个“没有 Workspace 的执行”的授权规则；Git Server 的沙盒身份按 Workspace 与代次签发
  （[Git Server 根决策](../git-server/0-project-code-hub-git-server-over-https.md) D2），固定 Node 没有这个身份。
- 根 D2 保留它，是为了没有 Workspace 的部署和前端 clones 页。Cloud 部署都已改为每个 Workspace 一个沙盒，clones 页
  也没有业务调用方。

### D6：删除 Project 时，等同步与发布 Operation 结束再交给 Controller

delete_project 的 quiesce 步骤在原有条件（各 Workspace 没有活动、Node 报告空闲，根 D5）之外，还要求该 Project 没有未结束的
同步或发布 Operation。这一条由 Cloud 在交给 Controller 之前判断：领取处于 quiesce 步骤的 delete_project 时发现仍有未结束的
Operation，不把它交给 Controller，而是置为 `retry_wait`、错误码 `repository_operation_running`，`retry_at` 为 10 秒后，并按 D2 的
“领取不因等待者落空”继续选下一个；不计入任何尝试次数，也不失败该 operation。

- 进入删除流程（Project 变为 `deleting`）的事务里，尚未 `running` 的同步与发布 Operation 以 `cancelled` 结束；`deleting` 的
  Project 不能新建任何同步与发布（契约 ADR D5 第 3 项）。一个 Operation 至多进入一次 `running`，租约过期时同步以失败结束、
  发布进入次数有上限的 `uncertain` 核验。所以 quiesce 只需要等已经 `running` 的 Operation 收尾，等待有界。
- 删除可能被回滚：Node 拒绝 quiesce 时 Project 从 `deleting` 回到 `active`。进入 `deleting` 时被取消的同步，在恢复准入的
  同一事务里补回，被取消的发布不恢复（契约 ADR D5 第 3 项）；本文不另加规则。
- 为什么等待而不是直接继续删除：`running` 的同步或发布可能正在向 Git Server 写 ref，或正在向外部仓库推送。删除流程先确认
  它们结束，cleanup 才不会与仍在进行的写入竞争；外部推送的结果也能被登记，而不是随 Project 一起丢失。
- 为什么在领取时判断，而不是在 quiesce 推进时拒绝：Controller 在 quiesce 中先关闭各 Node 的派发、再报告空闲；Cloud 拒绝空闲
  时，按根 D5 的规则会失败该 operation 并恢复准入。同步与发布还在进行不是“Node 不空闲”，不应让删除失败；在领取时挡住，
  Controller 根本不会开始 quiesce，也不需要理解这个条件。
- 已结束的 Operation 写下的 ref 随 Project 的仓库一起由删除流程处理（契约 ADR 未解决项中的回收）。
- delete_workspace 不受影响：同步与发布是 Project 级的操作，不属于某个 Workspace。

## 不变量

1. clone 步骤交给 Controller 的输入只来自 Binding 当前快照，四个字段都指向 Git Server；输入中没有外部仓库地址或凭据引用。
2. Controller 领到的 clone 步骤总是带着冻结的 `request.clone`；一个 operation 的 clone 输入至多冻结一次，所有 clone 执行输入相同。
3. Binding 没有当前快照时，不登记 clone 执行、不回退外部 clone；最近的同步未结束时 operation 处于 `retry_wait` 与
   `snapshot_not_ready`，不消耗尝试次数；最近的同步已结束时 operation 以 `repository_sync_failed` 失败。
4. 领取不会因为推后等待者而落空：只要还有可领取的 operation，领取就交出一个。
5. Workspace 的 `base_commit_id` 等于冻结输入的 `baseCommit`。
6. `workspaceRef` 等于 `refs/heads/workspaces/<该 Workspace>/<requested_ref>`，`requested_ref` 等于 `Binding.branch`。
7. 创建 run Workspace 不触发同步。
8. Cloud 没有租户级 clone 的入口与存储；每个 clone 执行都属于一个 Workspace operation。
9. delete_project 只有在该 Project 没有未结束的同步与发布 Operation 时才交给 Controller 执行 quiesce。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 创建 Workspace 时就冻结快照输入 | 输入在创建时就可见；但 create_project 创建 main Workspace 时第一次同步还没完成，而且冻结得越早，代码越旧（D1） |
| 每次 clone 尝试重新读取当前快照 | 重试能用上更新的快照；但同一 operation 的执行输入各不相同，`RecordDispatch` 的核对与失败归因都变复杂，而快照不可变，旧快照重试本来就有效（D1） |
| 没有快照时直接失败 | 用户立刻得到结论；但几乎每个新 Project 的第一次创建都会失败，用户只能手动重试（D2） |
| 同步收尾时由一个共享函数结束等待者 | 失败能立即传到等待者；但外部线的收尾事务要调用内部线的逻辑，并要加锁防止等待者在收尾之后才进入等待（D2） |
| 分两条语句读取当前快照与最近一次同步 | 查询更简单；但在外部线登记成功的瞬间，会把有快照的步骤判为失败（D2） |
| 没有快照时把 clone 步骤交给 Controller，由它 defer | 领取不需要跳过等待者；但 Controller 只是把 Cloud 的结论报回去，契约多一个 defer 原因（D2） |
| 没有快照时由 Cloud 在这里触发同步 | 等待时间更短；但触发同步属于外部线的同步策略，写在这里会让两条线改同一段代码（契约 ADR 需求 6） |
| 建 run Workspace 时隐式同步 | run 总基于最新代码；但 run 的启动依赖外部仓库与凭据的可用性和延迟（D4） |
| 没有快照时回退为从外部仓库 clone | 不需要等待；但外部凭据会进入 Node，违反契约 ADR 需求 3 与 D5 第 2 项 |
| `requested_ref` 另起名字，例如 `ora/<workspaceId>` | 名字天然唯一；但 `workspaces/<workspaceId>/` 已经保证唯一，另起名字只让 Agent 在 checkout 里看到陌生的分支（D3） |
| isolated Workspace 接受任意 `baseRef`，内容都用同步分支的快照 | 不改公开 API；但分支名与内容不符，违背“不静默替代”（D3） |
| 保留租户级 clone，改为从 Git Server clone | 前端 clones 页可以保留；但需要为没有 Workspace 的执行新设授权规则，而它没有业务调用方（D5） |
| 删除 Project 时在 quiesce 推进时拒绝 | 不改领取；但按根 D5 会让删除失败并恢复准入（D6） |

## 与批准版本的偏离

对照 cloud `a9c5f20`，最终实现与本文批准时相比有以下差异，正文已按最终实现表述：

| 批准时的决策 | 最终实现 | 原因 |
| --- | --- | --- |
| 无法生成输入的情形只有 `git_server` 未配置（`blocked`、`git_server_unconfigured`） | 另有两种缺陷情形同样以 `blocked` 停下：没有 Binding（`binding_missing`）、`workspaceRef` 组不成合法 ref（`invalid_workspace_ref`）（D1） | 两者都不会随等待消失；停下而不是失败，避免一个缺陷让每次领取都失败，并保留人工重试的余地 |
| 失败条件写作“最近的同步以非成功终态（`failed`、`superseded`、`cancelled`）结束”，没有同步按缺陷失败 | 最近的同步只要不处于 `pending`、`leased`、`running` 且没有快照就失败，包括“成功却没有快照”（D2） | 外部线在同一事务里登记成功与快照，成功而无快照只可能是缺陷；按失败处理避免永远等待 |
| 未规定读取方式，只要求读取“在一个一致的快照里进行” | 当前快照与最近一次同步在同一条语句中读取（D2） | 领取事务运行在 READ COMMITTED 下，两条语句会看到不一致的组合，把有快照的步骤判为失败 |
| `preparation` 展示 `snapshot_not_ready`；失败时 `syncErrorCode` 取该同步的 `error_code` | 另展示 `git_server_unconfigured`；`syncErrorCode` 按记录的同步 ID 只在同一 Project 内读取，没有记录 ID 时取该 Binding 最近一次失败同步的错误码，同步本身没有错误码时为空（D2） | 部署错误也需要对用户可见；按 Project 限定查找，避免记录的 ID 指向别的租户的同步 |
| 未规定公开 operation 视图 | 公开 operation 视图去掉 `request.clone` 与 `result.syncOperationId`（D1、D2） | 冻结输入与同步 Operation 的身份属于 Cloud 与 Controller，不进入公开 API |
| `Binding.branch` 只要求具体、不是 `HEAD` | 建 Project 时还要求它是合法的 Git 分支名（`400 default_branch_required`）（D3） | 保证 `workspaceRef` 总能组成合法的 ref |
| 不迁移已有的租户级 clone 记录 | 升级迁移删除不属于 Workspace 的 clone 执行与回执，删除 `clone_executions.clone_request_id`，把 `workspace_id` 设为非空，再删除 `clone_requests`（D5） | 不变量 8 要求库中不再有租户级 clone 的存储；删除而不是迁移，符合破坏性切换 |

## 本决策未解决的问题

- **多 Binding、多分支**：第一阶段只同步一个分支，所以 `baseRef` 只能等于 `Binding.branch`。同步多个分支后，`baseRef` 怎样
  选择 Binding 与分支，由后续决策给出。
- **从别的 Workspace 分支开始的 Workspace**：本文的输入总是快照；以某个 Workspace 分支为起点需要另外的输入形状。

## 验收

内部线的端到端验收不依赖外部线，与 [快照 clone 输入决策](../controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)、
[Node 从快照 clone 决策](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md) 共用一个栈与一个脚本
（cluster `scripts/workspace-git-acceptance.py`）。每个场景都先经公开 API 创建 Project（这时 Binding 与一行 `pending` 的同步
Operation 已经存在），再用预置工具写快照 ref、当前快照记录，或推进同步 Operation 的状态。本文承担的场景与断言：

| 场景 | 断言 |
| --- | --- |
| 同步未结束时等待 | create_project 停在 `clone` 步骤，`retry_wait`、`snapshot_not_ready`；没有 `clone_executions` 行；main Workspace 准入关闭；沙盒网络为 `--internal`，外部地址不可达 |
| 快照登记后推进 | `request.clone` 等于当前快照生成的四个字段；恰有一个 `clone_executions` 行，其 `input` 与 `request.clone` 逐字相同；operation 走到 `done`；`base_commit_id` 等于 `baseCommit`，`requested_ref` 等于 `Binding.branch` |
| 同步失败时结束等待 | operation 以 `failed`、`repository_sync_failed` 结束，`result` 只记录该同步 Operation 的 ID；Project 为 `active`；没有 `clone_executions` 行 |
| 冻结后快照更新、run Workspace 不隐式同步 | 当前快照改为另一个提交后触发 IssueRun：run Workspace 的 `request.clone.baseCommit` 是新快照，main Workspace 的 `base_commit_id` 不变；触发前后当前快照记录（含登记时间）与同步 Operation 不变 |
| `baseRef` 不是同步的分支 | `400 base_ref_not_synced`，`workspaces` 与 `operations` 没有新行 |
| 删除 Project 等待同步结束 | 同步 `running` 期间 delete_project 停在 `quiesce`，`retry_wait`、`repository_operation_running`，沙盒未被终止；同步结束后走到 `done` |
| 租户级 clone 不存在 | `POST /api/v1/tenants/{tid}/clones` 为 `404`；数据库中没有 `clone_requests` 表；`clone_executions` 中没有不属于 Workspace operation 的执行 |

## 核心测试用例

本 ADR 的核心测试用例见[内部线用例](../../../test-cases/cloud/operation/clone-step-consumes-binding-snapshot.md)，按义务登记
cloud `a9c5f20` 与 cluster `b0186a6` 上的直接证据，缺口如实标为 Partial 或 Missing。
