---
status: proposed
---

# Project 经 Repository Binding 同步外部仓库：Repo Executor 写快照，Cloud 核验后推进当前快照

每个 Project 有一个 Repository Binding，记录外部仓库地址、要同步的分支和凭据引用。外部仓库的内容只经同步进入 Ora：
Cloud 创建一个持有租约的同步 Operation，调用 [Repo Executor](../repo-executor/0-repo-executor-sole-external-git-boundary.md)；
执行器在临时裸仓库里 fetch 外部分支，再经 Git Server 的服务仓库监听，一次原子推送写入以提交 OID 命名的快照 ref 和跟踪 ref。
Cloud 经 ref 查询核验两个 ref 都指向报告的提交之后，才在事务内推进 Binding 的当前快照。Workspace 只从当前快照取得代码输入，
没有当前快照时，Workspace 的 clone 步骤只能等待，不回退为访问外部仓库，公开仓库也要先同步一次；等待只在 Binding 还有
同步在进行时成立，同步以“没有快照”收尾时，等待它的 operation 在同一事务中结束（D9）。

本 ADR 处于 `proposed`，是 `cloud/project` 领域在[仓库凭据连续性](20260927-repository-credential-continuity.md)之后的后续决策，
遵守 [Git Server 代码中心契约](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约”）的 D4、D5、D6。
涉及 Cloud（Binding、同步 Operation、ref 查询的调用方、公开 API 与前端）、Repo Executor 和 Credential Store。
发布（Publish）由[后续 ADR](20261017-explicit-fast-forward-publish.md) 承担；凭据的保管与能力验证由
[identity-access 后续决策](../identity-access/20261016-git-credentials-in-independent-credential-store.md) 承担。

兼容性边界：

- 现有 `projects.repository_url`、`default_branch`、`credential_ref_id` / `repository_credential_ref_id` 迁移为每个 Project 的第一个
  Binding（D1）。迁移后这几列不再是权威，只为兼容保留，本文不删除它们。
- Workspace 怎样消费当前快照、Node 怎样 clone，由内部线的 `cloud/operation`、`cloud/controller-integration`、`node/repository`
  后续 ADR 规定。本文只保证当前快照按契约 D5 第 2 项存在并被正确推进。
- 不迁移已有的 bundle Revision；不改变 Revision 交付。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [Project 根决策](0-project-space-scoping.md)：Project 按租户成员可见和授权，`owner_user_id` 只作创建者审计 | 继承，不变。同步的触发与查看按同一成员规则（D3、D8） |
| [仓库凭据连续性](20260927-repository-credential-continuity.md) D1、D2：引用分为团队、个人、未知；关联成员离开后个人和未知引用冻结，拒绝新的远程操作；已开始的任务可以收尾 | 继承，不变。同步是“新的远程操作”，创建 Operation 和签发授权前都检查引用可用（D7） |
| 凭据连续性 D3：Controller/Node 在受控目标下验证候选引用，“只验证读取不能被记录为已经拥有推送权限” | 修改：验证的执行者改为 Repo Executor；读取能力由第一次成功同步证明（D5），推送能力的验证见 identity-access 后续决策 |
| 凭据连续性 D4：执行端按引用选择，不回退全局静态凭据；凭据临时提供、结束后回收 | 继承，不变。执行端从 Controller/Node 改为 Repo Executor |
| [默认分支决策](../operation/20260926-require-concrete-default-branch.md)：Project 必须给出具体默认分支；Cloud 不访问远端仓库 | 继承。Binding 的分支就是这个具体分支（D1）；Cloud 仍不访问外部仓库，只经执行器访问 |
| 契约 D4：快照 ref create-only、名称等于目标；跟踪 ref tracking-cas，允许非快进 | 继承，不变。执行器的一次推送同时写两者（D2） |
| 契约 D5 第 2 项：每个 Binding 一个当前快照，只在核验后写，总是最新一次核验通过的同步结果 | 继承，不变。本文规定由哪些同步、在什么时机推进它（D3、D5） |
| 契约 D5 第 3 项：同步 Operation 的上下文（类型、Project、Binding、租约、可读可写） | 继承。本文规定 Operation 的状态、租约和 expected old 的取得方式（D2） |
| 契约落地顺序第 1 步：Binding 与当前快照两张表由底座建立 | 继承，并请底座再建立两样东西：`repository_operations` 的最小存储（Operation ID、类型、Binding、状态），以及结束等待者的函数（D9）。本文在其上增加租约、结果等字段和迁移（落地顺序第 1 步） |

## 决策

### D1：Repository Binding

Binding 是 Project 与一个外部仓库分支的绑定：

| 字段 | 含义 |
| --- | --- |
| `bindingId` | 稳定 ID，出现在 Git Server 的 ref 名 `refs/ora/upstreams/<bindingId>/...` 中，创建后不变 |
| `projectId`、`tenantId` | 所属 Project，复合外键保证同租户 |
| `repositoryUrl` | 外部仓库地址，第一阶段只接受 `https://`，不能带用户信息（`user:pass@`）、查询串和片段 |
| `branch` | 要同步的分支短名，必须是具体分支，不能是 `HEAD` |
| `credentialRefId` | 凭据引用；为空表示公开仓库（D4） |
| `version` | 乐观版本 |

- 第一阶段每个 Project 恰好一个 Binding，数据库以 `UNIQUE(project_id)` 保证；多 Binding 留给后续决策。
- `bindingId` 不复用 `projectId`：ref 名中的 ID 要能在多 Binding 时保持不变，第一个 Binding 也用独立 ID。
- 修改 `repositoryUrl` 或 `branch` 等于换了上游，第一阶段不允许修改，只能新建 Project。修改 `credentialRefId` 按凭据连续性 D3
  的候选验证切换：候选引用用一次同步证明读取能力，成功后才原子切换（D5）。

**迁移**：每个未删除的 Project 生成一个 Binding：

- `repositoryUrl` 取 `projects.repository_url`，`branch` 取 `projects.default_branch`；
- `credentialRefId` 取当前生效的引用 `COALESCE(repository_credential_ref_id, credential_ref_id)`，与现有
  `requireRepositoryAccess` 的取值一致；
- 不满足 D1 校验的旧数据（`default_branch` 为 `HEAD`、URL 不是 `https://` 或带用户信息）照样迁移，但 Binding 标为
  `invalid_configuration`，拒绝同步并在界面说明原因。不猜测分支、不改写 URL：与默认分支决策不变量 4 一致，不静默改写权威数据；
- 迁移不创建当前快照，也不触发同步。迁移后的 Project 在第一次同步成功前不能创建新 Workspace（契约 D5 第 2 项）。

### D2：同步 Operation

同步记录为 `repository_operations` 中 `kind = repository_sync` 的一行。发布 Operation 共用这张表（见发布 ADR），因为 Git Server
按 `Ora-Git-Operation-Id` 解析上下文时只需查一张表。

| 字段 | 含义 |
| --- | --- |
| `id` | Operation ID，也是执行器访问 Git Server 时的 `Ora-Git-Operation-Id` |
| `bindingId`、`projectId`、`tenantId` | 范围 |
| `trigger` | `project_created`、`manual`、`after_publish`、`credential_candidate` |
| `requestedBy` | 触发的用户；系统触发时为空并记录来源 Operation |
| `credentialRefId`、`credentialVersion` | 创建时固定的凭据引用与版本；公开仓库为空 |
| `expectedTrackingOid` | 创建时经 ref 查询读到的跟踪 ref 值，不存在时为全零 |
| `state` | 见下表 |
| `leaseOwner`、`leaseGeneration`、`leaseExpiresAt` | 租约 |
| `result` | 成功时的 `commitOid`、`forced`（是否非快进）、核验时间；失败时的错误码 |

状态：

```text
pending ──lease──▶ leased ──issue authorization──▶ running ──▶ succeeded
   │                 │                                │──────▶ superseded
   │                 └─lease expired─▶ pending        │──────▶ failed
   └──────────────▶ cancelled                         └─lease expired─▶ failed(lease_lost)
```

- **每个 Binding 同时至多一个未结束的同步**（`pending`、`leased`、`running`），以部分唯一索引保证。已有未结束的同步时，
  新的触发返回已有的 Operation，不另建。
- **租约**：Cloud 的同步 worker 以 `leaseGeneration + 1` 领取 `pending` 行，进入 `leased`；之后每次状态变化都以
  `(id, leaseGeneration)` 做条件更新，代次不符的 worker 写不进任何结果。worker 在调用执行器期间续约。
- **授权只在进入 `running` 时签发**，与状态变化在同一事务内。`leased` 状态下租约过期，回到 `pending` 是安全的：
  此时还没有一次性授权，旧 worker 不能调用执行器。
- **`running` 只发生一次**。`running` 中租约过期，Operation 以 `failed(lease_lost)` 结束，不回到 `pending`。
  契约 D5 第 3 项规定“Operation 结束或租约过期，同一个 ID 立即失效”；如果同一个 ID 可以被再次领取，失联的旧执行器就能借新租约
  写入，Git Server 无从区分。一次只运行一次，`Ora-Git-Operation-Id` 才能等同于一次执行。重试由新的 Operation 承担（D3）。
- **`expectedTrackingOid` 在创建时取得**：Cloud 在事务外经 ref 查询读取 `refs/ora/upstreams/<bindingId>/heads/<branch>`，
  再在创建事务内写入。不从当前快照记录取值：契约 D5 第 2 项允许记录落后于跟踪 ref（执行器推送后、Cloud 登记前的故障），
  用记录作 expected old 会让之后的每一次同步都 CAS 失败。
- **`superseded`**：执行器推送时跟踪 ref 不等于 `expectedTrackingOid`，并且也不等于本次 fetch 得到的提交，说明有别的写入者
  在本 Operation 创建之后推进了它（实际只可能是失联的旧 Operation 在租约过期前的写入）。Operation 以 `superseded` 结束，不登记，
  不自动重试：跟踪 ref 已经被推进，下一次同步会以新值为 expected old。跟踪 ref 已经等于本次的提交时按成功处理，见执行器根决策
  的重放规则。
- 取消只能发生在 `running` 之前。`running` 中的同步不能取消：推送一旦开始就由 Git Server 的原子性和 CAS 决定结果，取消只会
  让 Cloud 失去结果。
- **Project 删除**：只有 `active` 的 Project 能创建同步与发布 Operation。Project 进入删除流程（`deleting`）的事务里，该 Project
  尚未进入 `running` 的同步与发布 Operation 一律以 `cancelled(project_deleting)` 结束；已经 `running` 的照常收尾，它们写入的
  ref 随 Project 的仓库一起由删除流程处理。Project 删除的 quiesce 步骤等待这些 Operation 结束，和等待其他 Project 级操作一样。

### D3：触发时机

| 触发 | 时机 | 说明 |
| --- | --- | --- |
| `project_created` | 创建 Project 的事务内同时创建 Binding 和同步 Operation | Project 的创建本身不等待同步完成；main Workspace 的 clone 步骤等待这次同步（D9） |
| `manual` | 有 Project 修改权限的成员在界面或 API 触发 | 带 `Idempotency-Key`；已有未结束的同步时返回它 |
| `after_publish` | 发布 Operation 成功（包括 `uncertain` 核验为成功）且目标分支等于 Binding 的分支 | 由发布 ADR 规定 |
| `credential_candidate` | 管理员为 Binding 提交候选凭据引用 | 用候选引用同步一次，证明读取能力；成功后原子切换（D5） |

- **建 run Workspace 时不隐式同步。** Workspace 的代码输入总是当前快照（契约 D5 第 1 项）。隐式同步会让 Workspace 的创建
  依赖外部仓库的可用性和凭据状态，并让同一 Project 的两个 Workspace 因创建时间不同而得到不同的输入，这正是契约要消除的不确定性。
  需要外部的新提交时，用户先手动同步。
- 第一阶段没有定时同步和 webhook 同步，留给后续决策（见“本决策未解决的问题”）。
- 失败的同步不自动重试。`failed(lease_lost)` 和 `failed(executor_unreachable)`（Cloud 与执行器的连接在任务中断开）例外：
  Cloud 立即为同一 Binding 创建一个 `trigger` 相同的新 Operation，最多连续三次，避免进程重启或网络抖动让同步无声地停下；
  三次之后等用户手动触发。重放是安全的：执行器可能已经写入的快照 ref 和跟踪 ref，会被下一次同步当作空操作或新的 expected old。

### D4：公开仓库

`credentialRefId` 为空的 Binding 不兑换任何凭据，执行器匿名 fetch。其余步骤与私有仓库完全相同：同样经一次性授权调用执行器、
同样写快照和跟踪 ref、同样由 Cloud 核验。

- 匿名 fetch 时外部仓库要求认证（HTTP 401，或平台对私有仓库返回的 404），同步以 `failed(repository_requires_credential)` 结束，
  界面提示绑定凭据。不尝试任何部署级的默认凭据（凭据连续性 D4）。
- 公开仓库也必须先同步一次，Workspace 的 clone 步骤才能继续。Node 不再直接访问外部仓库，这一点对公开仓库同样成立（契约落地顺序第 3 步）。

### D5：推进当前快照

执行器报告 `{commitOid, snapshotRef, trackingRef, forced}` 后，Cloud 按契约 D6 的顺序处理：

1. 事务外经 ref 查询读取 `snapshotRef` 和 `trackingRef`。
2. 两者都指向 `commitOid`：在一个事务内以 `(id, leaseGeneration)` 条件把 Operation 置为 `succeeded`，把 Binding 的当前快照
   更新为 `(commitOid, snapshotRef, operationId, now)`；Operation 带凭据时，同一事务记录该引用版本的读取能力已验证（identity-access
   后续决策 D4）。
3. 不一致：Operation 以 `failed(verification_mismatch)` 结束，不更新当前快照。这表示执行器的报告与 Git Server 不符，记录为需要
   排查的错误。
4. ref 查询网络失败：在租约内重试；租约过期则 `failed(lease_lost)`，按 D3 由新 Operation 重放。快照 ref 已经存在，重放的 fetch
   和推送都是空操作，只需再核验一次。

当前快照总是更新为最新一次核验通过的结果，不比较新旧提交的先后（契约 D5 第 2 项）。同一 Binding 同时只有一个同步在运行（D2），
所以“最新一次”没有歧义。

**候选凭据**：`credential_candidate` 同步成功时，同一事务把 Binding 的 `credentialRefId` 切换为候选引用，前提是 Binding 的
`version` 自候选提交以来没有变化（凭据连续性 D3 的版本检查）。失败时不切换，原引用若已冻结则继续冻结。

### D6：同步的内容边界

| 内容 | 第一阶段的处理 | 理由 |
| --- | --- | --- |
| 分支 | 只同步 Binding 的一个分支 | 契约 D5 第 1 项只需要一个快照；多分支留在 namespace 中（`heads/<branch>`） |
| Tag | 不同步（`--no-tags`） | Workspace 的输入不依赖 tag；tag 可被外部改写，引入另一套不可变性问题 |
| 历史 | 完整历史，不做 shallow | shallow 快照会让 Workspace 的 `git log`、`blame`、merge-base 失效，发布时也无法在本地判定快进（发布 ADR D4） |
| LFS | 不下载 LFS 对象，仓库中的指针文件按普通文件同步 | 执行器的 fetch 禁用 LFS 过滤器；LFS 对象需要另一套存储与配额。同步成功，界面在检测到 `filter=lfs` 时提示“LFS 文件只有指针” |
| submodule | 不递归；gitlink 按提交指针保留 | 子模块是另一个外部仓库，需要自己的 Binding 和凭据；递归 fetch 会让一个 Binding 的凭据访问别的仓库 |
| 大小 | 单次 fetch 的数据量、对象数和耗时都有上限，取值与 Git Server 的 `receive.maxInputSize` 和仓库配额一致（契约 D7） | 超限以 `failed(repository_too_large)` 或 `failed(timeout)` 结束，不留半个快照：推送是原子的 |

**外部 force-push**：外部分支被改写时，下一次同步照常写入新的快照 ref；跟踪 ref 按 tracking-cas 允许非快进；当前快照更新为
新提交。执行器在推送前判定新提交是否是 `expectedTrackingOid` 的后代，结果写入 `forced`，界面据此提示“外部分支被改写”。

- 旧快照 ref 不删除（契约不变量 5），已经从旧快照开始的 Workspace 不受影响，它们的 Revision 和交付 ref 也不受影响。
- 基于旧快照发起的发布，`expectedOldOid` 与外部的当前值不一致，以 `conflict` 结束（发布 ADR）。
- **外部分支被删除**：同步以 `failed(upstream_branch_missing)` 结束，当前快照和跟踪 ref 都不变。Ora 不跟随删除：契约禁止删除
  任何 ref，而且已有快照是 Workspace 的输入。

### D7：凭据使用

- 创建同步 Operation 前、签发一次性授权前，各检查一次引用可用：未冻结、未撤销、范围与 `repositoryUrl` 匹配、关联成员仍有效
  （凭据连续性 D2）。不可用时创建返回 `409 repository_credential_unavailable` 或 `409 repository_credential_frozen`；已创建的
  Operation 在签发前发现不可用，以 `failed(credential_unavailable)` 结束。
- 引用被撤销后，新的同步一律拒绝。已经进入 `running` 的同步已兑换过凭据，按凭据连续性 D2 允许收尾。
- 同步只使用 Binding 当前的引用。个人引用被所有者绑定到 Binding，等于所有者同意该 Project 用它读取；读取不需要逐次批准，
  发布需要（发布 ADR D2）。
- 外部仓库以认证失败拒绝时（HTTP 401/403），同步以 `failed(credential_rejected)` 结束，并把该引用版本标为 `invalid`
  （凭据连续性 D3 表的最后一行），等用户更新凭据。

### D8：公开 API 与界面

API（均在 Project 作用域下，按 Project 根决策校验成员身份）：

| 路由 | 语义 |
| --- | --- |
| `GET /projects/{projectId}/repository` | Binding、当前快照、最近一次同步 Operation 的状态与错误码 |
| `POST /projects/{projectId}/repository/syncs` | 手动同步；带 `Idempotency-Key`；已有未结束的同步时返回 `200` 和它 |
| `GET /projects/{projectId}/repository/operations` | 同步和发布 Operation 的列表，按时间倒序分页 |
| `GET /projects/{projectId}/repository/operations/{operationId}` | 单个 Operation 的详情 |

Operation 状态变化后，沿用 [Space 事件流](../api-boundary/0-space-api-and-event-stream.md)只发失效通知，客户端重新读取 REST。

界面（Project 详情页的“代码来源”区块）：

- 显示外部仓库地址、分支、凭据名称与类型（团队、个人、未知）、凭据的读取和发布能力是否已验证。
- 显示当前快照：短 OID、来源同步的完成时间；`forced` 时提示外部分支被改写。
- 显示最近一次同步的状态。失败时把错误码映射为可操作的说明，例如：

  | 错误码 | 界面说明 |
  | --- | --- |
  | `repository_requires_credential` | 这个仓库需要凭据，请绑定一个凭据 |
  | `credential_rejected` | 外部仓库拒绝了凭据，请更新凭据 |
  | `repository_credential_frozen` | 凭据的所有者已离开，请管理员绑定团队凭据 |
  | `upstream_branch_missing` | 外部仓库里找不到分支 `<branch>` |
  | `repository_too_large` | 仓库超过同步上限 |
  | `invalid_configuration` | 仓库地址或分支不合法，需要重新创建 Project |

- “立即同步”按钮：有修改权限的成员可见；已有未结束的同步时显示进度并禁用；Binding 无效或凭据不可用时禁用并说明原因。
- 没有当前快照、但有同步在进行时，等待中的 Workspace 显示“等待首次同步”；没有当前快照、也没有同步在进行时，Workspace 的
  创建入口禁用，并说明“需要先完成一次同步”。
- 等待中的 operation 因同步失败而结束时（D9），Workspace 的失败说明沿用上表，并给出“立即同步”的入口。

### D9：等待快照的 operation 怎样结束

内部线的 `cloud/operation` 后续 ADR 规定：Binding 没有当前快照时，Workspace 的 clone 步骤以 `retry_wait`、错误码
`snapshot_not_ready` 等待，不计入 clone 的尝试次数，也不触发同步。等待本身由内部线负责，本文规定**等待成立的条件**和
**等待的结束**，保证 operation 不会无限期占着 Project 的操作槽。

**等待成立的条件**：该 Binding 有未结束的同步 Operation（`pending`、`leased`、`running`）。

- 条件成立：clone 步骤继续等待，不设时间上限。同步总会结束，结束时按下一条处理。
- 条件不成立：快照不会自己出现（例如迁移来的 Project 从没同步过，或首次同步已经失败）。clone 步骤不进入等待，直接以
  `failed(repository_not_synced)` 结束。

**同步以“没有快照”收尾时结束等待者**：同步 Operation 进入终态（`failed`、`superseded`、`cancelled`），Binding 仍然没有当前
快照，并且同一事务里没有按 D3 创建自动重放的新 Operation 时，在登记终态的**同一事务**中，把该 Binding 下所有处于
`snapshot_not_ready` 等待的 operation 以 `failed(repository_sync_failed)` 结束，并记录这次同步的 Operation ID 与错误码
（例如 `credential_rejected`、`upstream_branch_missing`），界面据此说明真正的原因。

- 自动重放（`lease_lost`、`executor_unreachable`，D3）在同一事务中创建新的同步 Operation，条件仍然成立，等待者继续等待；
  三次重放用完后的那次终态才结束等待者。
- 同步成功时不需要额外处理：当前快照写入后，等待者在下一次 `retry_at` 冻结输入继续执行。
- 并发：判断“是否进入等待”和“同步收尾”都先锁定 Binding 行（`SELECT … FOR UPDATE`），两者串行。这样不会出现等待者在同步
  收尾之后才进入等待、从而无人唤醒的情况。
- 唯一没有上限的情形是同步停在 `pending`、`leased` 不前进，即同步 worker 整体不可用。这属于基础设施故障，与其他
  `retry_wait` 的 operation 一样等待恢复，不另设超时：超时只会把一次可恢复的故障变成用户需要重新创建的失败。

**实现归属**：两条线判断的是同一件事，必须是同一段代码。

- 判断“有没有未结束的同步”，读的是底座建立的 `repository_operations` 最小存储（ID、类型、Binding、状态）。
- “按 Binding 把 `snapshot_not_ready` 的 operation 以 `repository_sync_failed` 结束，并记录同步 Operation 的 ID 与错误码”
  是底座提供的一个 Cloud 函数。本文在登记同步终态的事务里调用它；内部线的 clone 步骤读同一份存储，预置工具推进同步状态时
  也调用它。

这两样都放在底座，而不是任何一条线。这样外部线不依赖内部线的代码，内部线也不依赖外部线的租约与执行器，两边都能用预置工具
独立验收。

结束后怎样重新创建 Workspace，沿用内部线对失败的 create_project / create_workspace 的处理，本文不改变。

为什么用“有没有同步在进行”而不是固定的等待时长：首次同步的耗时取决于仓库大小，从几秒到十分钟（执行器根决策 D4）。固定时长
要么太短，让大仓库的首次创建失败；要么太长，让凭据错误这种立刻就能确定的失败也要等满。同步本身的终态才是“快照会不会出现”的
准确答案。

## 不变量

1. 每个 Project 第一阶段恰好一个 Binding；`bindingId` 创建后不变。
2. 每个 Binding 同时至多一个未结束的同步 Operation。
3. 一个同步 Operation 至多进入一次 `running`；租约过期后它的 ID 永久失效。
4. 一次性授权只在进入 `running` 的事务内签发。
5. 当前快照只在 Cloud 经 ref 查询确认快照 ref 和跟踪 ref 都指向报告的提交后更新。
6. 同步失败、取消、`superseded` 都不改变当前快照。
7. 外部 force-push 和分支删除不会删除或改写任何已有快照 ref。
8. 不可用的凭据引用不会被用于新的同步；公开 Binding 不使用任何凭据。
9. 创建 run Workspace 不触发同步。
10. 迁移不改写 `projects` 的原有列，不猜测分支和凭据归属。
11. 只有在 Binding 有未结束的同步时，operation 才会以 `snapshot_not_ready` 等待。
12. 同步以“没有快照”收尾且没有自动重放时，等待该 Binding 的 operation 在同一事务中结束。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 建 run Workspace 时隐式同步 | Workspace 的创建会依赖外部仓库的可用性，同一 Project 的两个 Workspace 输入不确定（D3）；同步失败要回滚 Workspace，牵连内部线 |
| 同步 Operation 复用现有 `operations` 表 | 现有表的 `kind`、`step` 是 Workspace 生命周期的步骤（`storage`、`worktree`、`sandbox`……），由 Controller 推进；同步与发布由 Cloud 自己的 worker 加租约推进，状态集合和终态都不同。混在一起会让 Git Server 的上下文解析与 Workspace 生命周期耦合 |
| 租约过期后重新领取同一个 Operation | 实现更简单，也少一行记录；但同一个 `Ora-Git-Operation-Id` 会先后属于两次执行，失联的旧执行器能借新租约写入，违反契约 D5 第 3 项“租约过期，同一个 ID 立即失效” |
| 以当前快照记录作为 expected old | 记录可能落后于跟踪 ref（契约 D5 第 2 项），会让后续同步永久 CAS 失败；ref 查询取得的值才是 Git Server 的事实 |
| shallow 同步 | 节省传输，但 Workspace 的历史不完整，发布时无法在本地判定快进 |
| 同步 tag 和所有分支 | 契约第一阶段只消费一个快照；全量同步放大数据量和凭据的使用面，namespace 已留出位置 |
| 同步时递归 submodule、下载 LFS | 一个 Binding 的凭据会访问别的仓库或别的存储；需要独立的 Binding 模型和配额 |
| 迁移时把 `HEAD` 改成 `main` 或解析远端 HEAD | 猜测会写入不存在的分支；Cloud 也不访问远端（默认分支决策） |
| `snapshot_not_ready` 设固定的等待上限 | 实现简单，不依赖同步的状态；但上限无法同时适应大仓库和立即可知的失败（D9） |
| 同步失败后让等待者继续等，直到用户手动同步成功 | 修好凭据后不用重新创建 Workspace；但等待期间 Project 的操作槽一直被占，其他操作都返回 busy，而用户可能根本不知道要去修 |

## 本决策未解决的问题

- **多 Binding、多分支、tag**：namespace 已经允许，模型与界面留给后续决策。
- **定时同步与 webhook**：需要外部平台的回调入口和去重，第一阶段只有手动和事件触发。
- **SSH 地址和 GitHub App 身份**：第一阶段只有 `https://` 加 token。凭据类型的扩展见 identity-access 后续决策。
- **LFS 与 submodule 的真正支持**：需要对象存储配额与子仓库的 Binding。
- **快照回收**：与契约一致，第一阶段全部保留。

## 落地顺序

1. **同步 Operation 与迁移**：在底座建立的 Binding、当前快照两张表，以及 `repository_operations` 最小存储与结束等待者的函数
   （D9）之上，增加租约与结果字段、租约 worker、`projects`
   到 Binding 的迁移、Git Server 上同步 Operation 的上下文来源。完成条件：迁移在新库和升级库上都通过；Git Server 对同步 Operation
   的授权符合契约 D5 第 3 项，有真实 `git` 客户端的集成测试。
2. **执行器接入**：Cloud 调用执行器完成同步，核验并推进当前快照；公开仓库和私有仓库都打通。完成条件：下面的独立验收全部通过。
3. **结束等待者与界面**：D9 的条件与收尾，代码来源区块、同步按钮、错误说明，Workspace 创建入口的禁用条件。
4. **汇合**：内部线切换 Workspace 输入（契约落地顺序第 3 步）时，本文的当前快照已经可用。

保持不变：Project 的成员授权、默认分支规则、Revision 交付。从第 2 步开始改变：Project 的代码来源由同步得到。

## 外部线的独立验收

不依赖内部线和 Node，不启动沙盒。

**起栈**：Cloud、Git Server（服务仓库监听和 ref 查询监听）、Credential Store、Repo Executor，加两个本地 Git 托管 fixture：
一个带 TLS、要 token 才能读的私有仓库 fixture，一个公开读取的 fixture。外部仓库地址经部署的允许列表放行（执行器默认拒绝私有
地址，见执行器根决策）。

**判定**：每个场景的同步结果都必须通过底座的共享契约断言（契约落地顺序第 1 步）：Git Server 上的快照 ref、跟踪 ref 与 Cloud 的
当前快照记录符合契约 D5。

| 场景 | 必须成立 |
| --- | --- |
| 公开仓库首次同步 | `succeeded`；快照 ref 与跟踪 ref 指向外部分支的提交；当前快照被写入；没有任何凭据兑换记录 |
| 私有仓库首次同步 | 同上；Credential Store 有且只有一次兑换；凭据引用的读取能力被标为已验证 |
| 私有仓库匿名同步 | `failed(repository_requires_credential)`；没有快照 ref |
| 重复同步同一提交 | 第二次 `succeeded`，Git Server 上 ref 数量不变，当前快照指向同一提交 |
| 并发触发 | 两次手动触发返回同一个 Operation |
| 外部 force-push 后同步 | 新快照写入，`forced = true`；旧快照 ref 仍然存在且指向原提交；当前快照指向新提交 |
| 外部分支删除后同步 | `failed(upstream_branch_missing)`；当前快照不变 |
| 撤销凭据后同步 | 创建返回 `409`，没有新的 Operation 和兑换 |
| 成员离开后个人凭据的同步 | 创建返回 `409 repository_credential_frozen` |
| worker 在 `running` 中被杀 | Operation 以 `failed(lease_lost)` 结束；之后用旧 Operation ID 访问服务仓库监听被拒绝；新 Operation 重放后 `succeeded` |
| 预置的跟踪 ref 被改动（模拟失联的旧执行器） | `superseded`；当前快照不变 |
| 超过大小上限 | `failed(repository_too_large)`；Git Server 上没有新的 ref |
| 首次同步失败结束等待者 | 预置工具为 Binding 写一个 Workspace 及其处于 `snapshot_not_ready` 等待的 operation（替身，不需要沙盒与 Node），再用错误的 token 同步：同步 `failed(credential_rejected)` 的同一事务里，该 operation 以 `failed(repository_sync_failed)` 结束并记录同步的错误码 |
| 自动重放期间不结束等待者 | 同上的替身；在 `running` 中杀掉 worker：等待者仍在等待；重放成功后当前快照写入，等待者没有被结束 |
| 没有同步时不进入等待 | 迁移来的、从未同步的 Binding：判定入口返回 `repository_not_synced`，不返回等待 |
| 迁移 | 升级库中每个 Project 都有一个 Binding；`HEAD` 分支的 Project 的 Binding 为 `invalid_configuration`，同步被拒绝 |
| 秘密扫描 | 用 canary token 跑完以上场景后，Cloud 与执行器日志、Git Server 仓库、Cloud 数据库、执行器临时目录中都找不到 canary |

**最终验收**：用 owner 提供的 GitHub 私有测试仓库，以真实 token 完成首次同步、外部推送后的再次同步、外部 force-push 后的同步。
