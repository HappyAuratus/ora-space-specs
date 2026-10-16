---
status: proposed
---

# Git Server 成为每个 Project 的代码中心：ref namespace、规则集授权与两类写入者的契约

Git Server 不再只存放交付 ref。从本决策起，它是每个 Project 唯一的代码中心，仓库仍是 `/{tenantId}/{projectId}.git`。
这个仓库里只有两类写入者：

- **沙盒执行**：经现有仓库监听、用 git-access 证书访问，写 Workspace 分支和交付 ref；
- **Repo Executor**：经新的服务仓库监听、用服务证书访问，写外部仓库同步得到的快照和跟踪 ref。

每个请求带一个上下文头。Git Server 按 Cloud 当前状态判定这个请求能不能访问该 Project 仓库，再把它解析成一份写入规则集，
由 `pre-receive` 按规则逐个 ref 判定。读侧不做 ref 隐藏：同一 Project 内的代码对获授权的身份共享，隔离边界是 Project 仓库本身。
Cloud 仍然不写 Git Server，只在事务外查询、在事务内登记。外部仓库只经 Repo Executor 访问，Node、Workspace 与 Git
Server 不接触外部凭据。

本 ADR 处于 `proposed`，是 [Git Server 根决策](0-sandbox-scoped-git-server-over-https.md) 的第一个后续决策。它的定位是
**两条开发线共同遵守的契约**：

- 内部线：Git Server 与 Node 之间；
- 外部线：外部仓库、Repo Executor 与 Git Server 之间。

本文只规定 Git Server 一侧的承诺，以及两条线交接的数据形状。以下内容由各自领域的后续 ADR 承担，必须遵守本文：

| 内容 | 承担的后续 ADR |
| --- | --- |
| Workspace 怎样拿到快照输入 | `cloud/operation`、`cloud/controller-integration` |
| Node 怎样 clone、fetch、推送 Workspace 分支 | `node/repository` |
| 外部仓库的 Binding、同步与发布的业务流程 | `cloud/project` |
| 凭据管理 | `cloud/identity-access` |
| 执行器程序本身 | `cloud/repo-executor` 根决策 |

范围边界：

- Revision 的交付与续作继承 [Revision Git 交付决策](../revision/20261012-git-server-delivery-and-verified-delivery-refs.md)，本文不改它的
  规则，由内部线一并实现；
- 会话 JSONL 仍在对象存储；
- **破坏性切换**：Cloud 目前没有用户，本组决策不设过渡开关、不做数据迁移，也不保留 bundle 与外部 clone 的旧路径。已有的开发
  环境数据（Project、Workspace、bundle Revision）在切换后重建，不承诺可用；
- 不承诺多副本、备份和 ref 回收（见“本决策未解决的问题”）；
- 不改变沙盒的出网策略。本文保证的是：平台管理的代码输入、协作和交付都只经 Git Server，沙盒里没有外部仓库凭据。Agent 在沙盒里
  自行访问公开网络上的仓库，不在本文约束之内；由于没有凭据，它无法以平台的名义读取私有仓库或写入外部仓库。

## 需求

1. 代码输入、Workspace 协作、Revision 交付与恢复全部使用 Git 原生对象和 ref，不使用 bundle。
2. 多个 Workspace 经 Git Server 协作。
3. 外部仓库凭据不进入 Node、Workspace、Agent 和 Git Server 的仓库。
4. Workspace 的推送不会触达外部仓库。
5. 权限撤销、Workspace 代次变化、执行结束、Operation 结束后，新的 Git 操作立即被拒绝。
6. 内部线与外部线能并行开发、分别验收：两边只经本文规定的 ref 与数据形状交接。
7. 目标是先跑起来、行为正确；纵深的安全加固留到后续决策（见“本决策未解决的问题”）。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根决策 D1：独立程序；只提供 smart HTTP；仓库 `/{tenantId}/{projectId}.git`；首次获授权的写入时建仓 | 继承程序形态、协议、路径和原子建仓。修改：Git Server 是 Project 的代码中心；获授权的写入可以来自交付执行，也可以来自同步 Operation（D1） |
| 根决策 D2：认证只认沙盒 git-access 证书，SAN 为 `spiffe://ora.local/git-access/tenant/{t}/runtime/{ws}/generation/{g}` | 沙盒身份继承，不变。修改：新增只接受 Repo Executor 服务证书的服务仓库监听（D2） |
| 根决策 D3：每个请求带 `Ora-Execution-Id`，按 Cloud 当前状态判定，数据库不可用返回 `503` | 继承逐请求判定和 fail closed。修改：判定结果从“模式 + 单个交付 ref”改为 ref 规则集；执行器请求带 `Ora-Git-Operation-Id`（D3） |
| 根决策 D4：每次交付尝试一个只能新建的 `refs/ora/deliveries/<deliveryWorkId>`；`refs/heads` 留给后续 ADR | 交付 ref 继承，不变。修改：新增 upstream 和 workspace 两个 namespace 及其规则（D4） |
| 根决策 D5：Node 执行 Git 访问，私钥不进入工作负载 | 继承，不变。Node 新增的操作由 `node/repository` 后续 ADR 规定 |
| 根决策 D6：两个监听分开；ref 查询只读、只对 Cloud；Cloud 事务外核验、事务内登记；不由 Git Server 写收据 | 继承。修改：ref 查询可以查询本文新增 namespace 中的 ref，仍然只读（D6） |
| 根决策 D7：Node 照旧从上游 clone，Git Server 不镜像上游、也不推回上游 | 修改：上游内容只经 Repo Executor 进入 Git Server；Workspace 的代码输入只来自 Git Server 的快照（D5）。这正是根决策 D7 为后续 ADR 预留的方向 |
| [Revision Git 交付决策](../revision/20261012-git-server-delivery-and-verified-delivery-refs.md)：Revision 记录交付 ref 与最终提交 | 继承，不变。不另建 Revision ref（D4） |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md)：Cloud 只管理凭据引用；执行端按引用选择，不回退全局凭据 | 继承。凭据只在 Repo Executor 中使用，本文不涉及凭据本身 |
| [运行时 Workspace 使用权限](../workspace/20260927-runtime-workspace-use-permissions.md) D2：代码、分支详情只供可使用目标者（创建者或管理员） | 修改，只改 Git 层面：同一 Project 内的 Workspace 分支与交付 ref，对该 Project 中获授权的沙盒执行与执行器共享，可以 fetch、merge，也可以作为发布源（D3）。界面上谁能查看代码与分支详情，仍按原规则 |

## 决策

### D1：Git Server 是 Project 的代码中心

每个 Project 在 Git Server 上有且只有一个裸仓库 `/{tenantId}/{projectId}.git`。仓库里有：

- 外部仓库同步得到的快照和跟踪 ref；
- 各 Workspace 的协作分支；
- 每次执行的交付 ref。

Revision 与外部发布的源提交，都指向这个仓库里的对象。

- 仓库在第一次获授权的写入时创建，沿用现有的原子方式：在临时目录初始化后 rename 发布。读请求不建仓。
- Cloud 不写 Git Server。写入者只有两类：沙盒执行与 Repo Executor。
- 租户之间不共享仓库、对象目录或 `objects/info/alternates`。

为什么 Cloud 不写：根决策 D6 已经论证过，由登记一方亲自核验、Git Server 不写收据，可以省掉数据库与 Git 之间的对账。
本文需要 Cloud 登记的事实（交付、同步结果）都能表达为“写入者写 ref，Cloud 查询后登记”，因此不需要 Cloud 写接口、
outbox 或 reconciler。

### D2：三个监听，两种写入身份

| 监听 | 网络 | 接受的身份 | 能做什么 |
| --- | --- | --- | --- |
| 仓库监听（现有） | 沙盒网络 | 沙盒 git-access 证书 | smart HTTP 读写，受 D3 约束 |
| 服务仓库监听（新增） | 内部服务网络 | Repo Executor 服务证书 `spiffe://ora.local/repo-executor` | smart HTTP 读写，受 D3 约束 |
| ref 查询监听（现有） | Compose 默认网络 | Cloud 的固定客户端身份 | 只读查询 ref 指向的提交 |

- 沙盒访问不到服务仓库监听和 ref 查询监听。执行器需要外网出口，不接入沙盒网络。
- 服务证书与沙盒证书由不同的 CA 签发，每个监听只信自己的 CA。
- 服务证书只证明“这是 Repo Executor”，不携带仓库或 ref 权限。
- 服务身份沿用现有 trust domain `ora.local`，用途段在前，格式与 `git-access` 一致。
- 仓库配置按监听注入（现有实现已经按请求以环境变量注入配置）：仓库监听保持 `receive.denyNonFastForwards=true`；服务仓库监听
  关闭它，以便跟踪 ref 跟随外部 force-push（D4）。两个监听都保持 `receive.denyDeletes=true`。

为什么不让执行器走仓库监听：仓库监听只在沙盒网络。执行器持有外部凭据，让它加入沙盒网络，就等于把它放到工作负载能直接
访问的地方。分开监听，根决策 D6 的网络隔离得以保持。

### D3：每个请求解析为一份 ref 规则集，`pre-receive` 再判定一次

每个请求必须带且只带一个上下文头：

- 仓库监听上的请求带 `Ora-Execution-Id`；
- 服务仓库监听上的请求带 `Ora-Git-Operation-Id`。

监听与上下文头不匹配、缺少上下文头、或者同时带两个，一律拒绝。

Git Server 按“证书身份 + 上下文 + 目标 Project”读取 Cloud 数据库的当前状态，解析出一份授权：

```text
Grant {
  project:  (tenantId, projectId)              // 获授权后可读整个 Project 仓库
  writable: [{ ref 或 ref 前缀, rule, expectedOld? }]
            // rule ∈ create-only | fast-forward | tracking-cas；为空表示只读
}
```

- **沙盒执行**：沿用根决策 D3 的有效性条件。执行属于证书中的运行时和当前代次，未结束，未被强停，Workspace 处于允许的阶段。
  授权内容由执行类型决定，具体映射由 `cloud/controller-integration` 后续 ADR 规定，但不能超出 D4 中“沙盒”一列允许的范围。
- **Operation**：Operation 必须存在，处于 `running` 且租约未过期，Project 与 Binding 匹配，Project 未删除。授权不要求 Project
  处于 `active`：首次同步发生在 `provisioning` 期间，进入 `deleting` 时已经 `running` 的 Operation 也要能照常收尾（见 D5 第 3 项的
  生命周期表）；现有的沙盒判定要求 `active`，Operation 的判定不能照抄。授权内容由 Operation 类型决定（见 D5），不能超出 D4 中
  “执行器”一列允许的范围。
- `pre-receive` 在写入前重新解析同一份授权，逐个 ref 核对规则。两次请求之间执行结束、代次变化或 Operation 被取消，写入都会被拒绝。
- 数据库不可用或判定出错，一律返回 `503`，不缓存授权。

**读侧不做 ref 隐藏**：获授权的请求可以读整个 Project 仓库，ref 广告列出仓库里的全部 ref。授权只回答两件事：能不能访问这个
Project 仓库，以及能写哪些 ref。隔离边界是 Project 仓库本身：每个 Project 是独立的裸仓库，跨 Project、跨租户的对象天然不可达。

为什么不按 ref 隐藏：

- 隐藏做不成边界。实测 protocol v2 下，即使用 `transfer.hideRefs` 隐藏 ref，并把 `uploadpack.allowAnySHA1InWant`、
  `allowTipSHA1InWant`、`allowReachableSHA1InWant` 都设为 `false`，客户端仍能按 OID 取到被隐藏 ref 引用的对象；现有实现会
  把客户端的 `Git-Protocol` 头透传给 `git http-backend`，线上实际走 v2。
- 隐藏推送广告会让客户端看不到服务端已有的对象。实测一个 50 个提交的仓库，只广告交付 ref 时推送 150 个对象，完整广告时只推 3 个；
  每次交付都会接近全量上传。
- 同一 Project 的 Workspace 本来就是协作关系（运行时使用权限的修改见“继承与修改”）。

为什么改成规则集：现有实现的判定结果只有 `Mode` 和一个 `DeliveryRef`，`pre-receive` 也只认这一个 ref。新增调用方后，如果
继续在判定函数里写 if-else，每加一种调用方都要同时改 SQL、返回结构和 hook。改成规则集后，“谁能写什么”由 Cloud 的数据一处决定，
Git Server 只负责执行规则。这也让两条线可以各自添加自己的规则来源，而不改同一段判定代码。

### D4：ref namespace 与写入规则

| ref | 写入者 | 规则 | 读取者 |
| --- | --- | --- | --- |
| `refs/ora/upstreams/<bindingId>/snapshots/<commitOid>` | 执行器（同步 Operation） | create-only；名称中的 OID 必须等于 ref 的目标 | 本 Project 的任何获授权身份 |
| `refs/ora/upstreams/<bindingId>/heads/<branch>` | 执行器（同步 Operation） | tracking-cas：旧值必须等于 Operation 记录的 expected old（零表示首次创建）；新值必须有对应的快照 ref，可以是仓库里已有的，也可以是同一次推送里新建的；允许非快进 | 同上 |
| `refs/heads/workspaces/<workspaceId>/<branch>` | 沙盒（该 Workspace 当前代次的执行） | fast-forward：只能新建或快进 | 同上 |
| `refs/ora/deliveries/<deliveryWorkId>` | 沙盒（对应的交付执行） | create-only（根决策 D4，不变） | 同上；另有 Cloud ref 查询 |

其他规则：

- 任何调用方都不能删除 ref，也不能写表外的 ref。
- 沙盒永远不能写 `refs/ora/upstreams/**`；执行器永远不能写 `refs/heads/workspaces/**` 和 `refs/ora/deliveries/**`。
- 快进约束有两道：`pre-receive` 按 namespace 执行规则；仓库监听上 `receive.denyNonFastForwards` 保持开启（D2）。服务仓库监听关闭
  它，跟踪 ref 才能跟随外部 force-push，其余 namespace 由 hook 拒绝非快进。
- Workspace 分支的 CAS 由 Git 本身承担：receive-pack 在 ref 锁内比对客户端声明的旧值，`pre-receive` 再核对快进关系。并发推送时，
  后到的一方收到冲突。
- 不新增 `refs/ora/revisions/*`。交付 ref 已经不可变，Revision 记录交付 ref 与提交即可。

快照为什么用提交 OID 命名而不用 sync-id：

- 同一提交只有一个快照 ref，重复同步不会让 ref 无限增长；
- 名称本身证明内容，create-only 的重放天然幂等；
- “哪次同步得到了哪个提交”记录在 Cloud 的同步 Operation 里，审计不依赖 ref 名。

### D5：两条线交接的数据形状

两条线只经下面三项交接。各自的后续 ADR 可以增加字段，但不能改变这些字段的语义。

**1. Workspace 的代码输入（外部线保证存在，内部线消费）**

```json
{
  "repositoryUrl": "https://<git-server>/<tenantId>/<projectId>.git",
  "snapshotRef": "refs/ora/upstreams/<bindingId>/snapshots/<commitOid>",
  "baseCommit": "<commitOid>",
  "workspaceRef": "refs/heads/workspaces/<workspaceId>/<branch>"
}
```

- 四个字段都指向 `repositoryUrl` 所指的 Git Server 仓库：`snapshotRef` 是这个仓库里的 ref，`baseCommit` 是它指向的提交。
  输入里不出现任何外部仓库地址，Node 只从 `repositoryUrl` 读取。
- 输入总是固定的快照和提交，从不引用跟踪 ref，所以同步过程中的变化不会影响已经开始的 Workspace。
- `baseCommit` 由 Cloud 从第 2 项的记录中取得，是 Cloud 核验过的值。消费方 clone 后，在本地比对 Git Server 返回的提交是否等于
  `baseCommit`，不一致就 fail closed。
- Workspace 自己的提交只写入 `workspaceRef`，交付时写入交付 ref，不会写入快照。
- `workspaceRef` 中的分支短名怎样取、续作时这份输入怎样与前序 Revision 的交付 ref 组合，由 `node/repository` 与
  `cloud/controller-integration` 的后续 ADR 规定。

**2. Binding 的当前快照（外部线写，内部线读）**

Binding 是 Project 与一个外部仓库的绑定，包括外部仓库、要同步的分支，以及凭据引用（公开仓库不需要凭据）。第一阶段每个 Project
只有一个 Binding，多 Binding 留给后续决策。

Cloud 为每个 Binding 记录当前快照：

| 字段 | 含义 |
| --- | --- |
| `bindingId` | 哪个 Binding |
| `commitOid` | 外部仓库同步得到的提交，不是 Workspace 产生的提交 |
| `snapshotRef` | Git Server 上指向 `commitOid` 的快照 ref |
| 同步 Operation | 产生这个快照的同步，用于审计 |
| 登记时间 | 写入这条记录的时间 |

- 只有在 Cloud 经 ref 查询确认快照 ref 存在、并且指向记录的提交之后，这条记录才会写入或更新。
- 记录总是更新为**最新一次核验通过的同步结果**，不要求比上一个提交更新。外部仓库 force-push 回旧提交时，记录同样跟着更新。
- 记录可能短暂落后于 Git Server 上的跟踪 ref，比如执行器推送之后、Cloud 登记之前发生了故障。这是安全的：快照不可变，落后的
  记录仍然指向一份完整、确定的代码。记录永远不会领先于 Git Server。
- Workspace 的 clone 步骤从这条记录生成第 1 项输入。没有当前快照时 clone 步骤不能继续，也不会回退为让 Node 直接访问外部仓库，
  所以新建 Project 后必须先完成一次同步，公开仓库也一样。等待快照的条件与结束方式由 `cloud/operation` 与 `cloud/project`
  的后续 ADR 规定。
- 外部仓库的内容变化（包括经 Publish 写入的内容）只经同步进入这条记录，不由其他路径直接改写。

**3. Operation 上下文（外部线写，Git Server 读）**

Git Server 解析 `Ora-Git-Operation-Id` 时，从 Cloud 读取以下信息：

| 字段 | 同步 Operation | 发布 Operation |
| --- | --- | --- |
| 类型 | `repository_sync` | `repository_publish` |
| Project、Binding | 必填 | 必填 |
| 状态与租约 | 必须持有未过期的租约 | 同左 |
| 可写 | 本 Binding 下 `snapshots/` 中名称与目标一致的新 ref（create-only）；`heads/<branch>`（tracking-cas，expected old 在进入 `running`、签发执行授权之前读取并记录） | 无 |
| 可读 | 整个 Project 仓库（D3） | 整个 Project 仓库；执行器只取 Operation 记录的源 ref 与源提交 |
| 源 | — | 本 Project 的 Workspace 分支或交付 ref（可以直接发布一次 run 的 Revision） |

- 同步的快照授权是按前缀给出的，不是某个具体 OID：创建 Operation 时，Cloud 还不知道外部仓库会同步到哪个提交，执行器 fetch
  之后才知道。约束靠 D4 的 create-only 和“名称与目标一致”两条规则。
- 发布 Operation 记录“源 ref + 源提交”，执行器从源 ref 取得源提交。交付 ref 不可变，以它为源时源提交就是它的目标。创建之后，源 ref（比如 Workspace 分支）可能继续前进；但 Workspace
  分支只能快进（D4），源提交始终是它顶端的祖先，所以从源 ref 一定取得到源提交。以后如果允许 Workspace 分支非快进改写，必须
  同时保证进行中的发布仍然取得到源提交。
- 权限跟着 Operation 走：同一个执行器进程处理不同 Operation 时，各自只有对应的权限；Operation 结束、取消或租约过期，同一个
  ID 立即失效。
- 一个 Operation 至多进入一次 `running`，不会被重新领取。`running` 中租约过期时，同步以失败结束、由新的 Operation 重放；发布进入
  `uncertain`，按发布 ADR 核验。这样旧执行器无法借新租约以同一个 ID 写入。

**Operation 的状态与生命周期**（两条线都按这里实现，不各自定义）：

| 类型 | 未结束 | 终态 |
| --- | --- | --- |
| 同步 `repository_sync` | `pending`、`leased`、`running` | `succeeded`、`failed`、`superseded`、`cancelled` |
| 发布 `repository_publish` | `awaiting_approval`、`pending`、`leased`、`running`、`uncertain` | `succeeded`、`conflict`、`rejected`、`failed`、`cancelled` |

- `uncertain` 算未结束，但核验次数有上限，用完后进入终态，所以任何 Operation 都会在有限时间内结束。
- 每个 Binding 至多一个未结束的同步，由部分唯一索引保证。登记同步终态的事务至多新建一个后继同步：自动重放与发布后补同步同时
  需要时只建一个，并清除 Binding 上的补同步标记。系统触发的同步因条件不满足建不了时，只记录原因，不让所在事务回滚。
- 按 Project 的生命周期，能新建哪些 Operation：

| Project | 同步 | 发布 |
| --- | --- | --- |
| `provisioning` | 只有建 Project 时底座建立的那一行，以及它的自动重放 | 不能新建 |
| `active` | 全部触发方式 | 可以新建 |
| `deleting` | 不能新建；进入 `deleting` 的事务清除补同步标记 | 不能新建 |

- 进入 `deleting` 的事务把该 Project 尚未进入 `running` 的同步与发布（`pending`、`leased`、`awaiting_approval`）以
  `cancelled`、`project_deleting` 结束；已在 `running` 或 `uncertain` 的照常收尾。delete_project 等到该 Project 没有未结束的
  Operation 才开始。这段取消逻辑和“未结束”的判定都在底座，两条线共用。
- create_project 无论因什么原因失败，同一事务都把 Project 从 `provisioning` 置为 `active`，Project 不会停在 `provisioning`。

为什么把这三项放进 Git Server 的 ADR：它们是两条线之间唯一的耦合点。在这里固定下来，内部线可以用预置的快照独立开发和验收，
外部线可以用预置的 Workspace 分支独立开发和验收，互不等待。

### D6：Cloud 只查询、只登记

继承根决策 D6 的顺序：写入者写 ref；Cloud 在事务外经 ref 查询确认；再在事务内登记。

- 交付：不变。
- 同步：执行器报告结果后，Cloud 查询快照 ref 和跟踪 ref，确认都指向报告的提交，再登记同步结果，更新 D5 中的当前快照。
  查询结果不一致，按核验失败处理；网络失败不接管，可以重放。
- Workspace 分支：Cloud 不逐次登记，需要时按需查询。

ref 查询监听可以查询本文所有 namespace 中的 ref，仍然只读，仍然只接受 Cloud 的固定身份。查询有两种形式：

- 查询单个 ref 指向的提交（现有）；
- 按前缀列出一个 Project 仓库中的 ref 及其提交，前缀必须落在本文的某个 namespace 内（例如
  `refs/heads/workspaces/<workspaceId>/`）。Cloud 用它展示一个 Workspace 的分支，例如作为发布的源 ref 供用户选择。
  列出的结果只是查询时刻的事实，Cloud 不据此登记任何东西。

### D7：第一阶段的存储与运维

- 单实例 Git Server，仓库放在持久卷，不做多副本。
- 每次推送设置 `receive.maxInputSize`；仓库超过配额时，`pre-receive` 拒绝新的写入并返回明确的错误。
- 定期执行 `git gc --auto` 和 `git fsck`，不清理任何可达对象。第一阶段，快照、Workspace 分支和交付 ref 一律保留。
- 对象存储不再承担代码的输入输出，只存放会话 JSONL。

## 不变量

1. 证书不携带授权；每个请求和每次 `pre-receive` 都按 Cloud 当前状态判定；判定不可用时返回 `503`。
2. 每个请求恰好带一个上下文头，而且与所在监听匹配。
3. 沙盒只能写本 Workspace 的分支和当前执行的交付 ref。
4. 执行器只能写同步 Operation 指定 Binding 下的快照和跟踪 ref；发布 Operation 在 Git Server 上没有写权限。
5. 快照 ref 和交付 ref 只能新建；任何 ref 都不能删除。
6. 跟踪 ref 只能以 CAS 方式更新，新值必须有对应的快照 ref。
7. Workspace 分支只能新建或快进。
8. 获授权的请求可读整个 Project 仓库；写入只限规则集列出的 ref。
9. 跨租户、跨 Project 的对象不可达。
10. 沙盒访问不到服务仓库监听和 ref 查询监听。
11. Workspace 的代码输入总是固定的快照和提交。
12. Binding 的当前快照只在 Cloud 核验快照 ref 之后更新，总是等于最新一次核验通过的同步结果。
13. Cloud 不写 Git Server。
14. 一个 Operation 至多进入一次 `running`。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Node 直接访问外部仓库（维持根决策 D7） | 外部凭据会进入 Node，违反需求 3；多 Workspace 协作也没有共同的仓库 |
| 执行器写共享卷，匿名监听按来源 IP 和容器 label 识别 Workspace | 身份依赖部署拓扑，与 git-access 证书形成两套认证；执行器绕过授权直接写仓库，违反不变量 1 |
| 执行器 checkout 后交给 Workspace | 执行器要挂载 Workspace 卷，耦合部署拓扑；内容来源不经 Git Server，违反需求 1 |
| 新增 `refs/ora/revisions/*`，由 Cloud 经内部写接口创建，再配 outbox 和 reconciler | 交付 ref 已经不可变，再加一层只是重复；Cloud 写接口违反根决策 D6，并引入跨库的最终一致和对账 |
| 快照按 sync-id 命名 | 重复同步同一提交会让 ref 无限增长；OID 命名天然幂等 |
| 执行器走仓库监听 | 持有外部凭据的组件会进入沙盒网络，破坏根决策 D6 的网络隔离 |
| 每个 Workspace 一个仓库 | 多份存储；协作要跨仓库 fetch；与 Project 级的 Revision 不一致。它的优点是对象级隔离更强，但同一 Project 的 Workspace 本来就是协作关系，不需要这层隔离 |
| 同一 Project 内按 ref 隐藏、按运行时使用权限收窄读权限 | protocol v2 下隐藏不住对象（D3 的实测）；要做成边界就得强制 v0，还要为每种写入补上基础 ref 的广告，否则交付推送近乎全量。同一 Project 的协作本来就需要共享 |
| Gitea、Forgejo | 自带用户、权限和 Token 体系，与 Cloud 的租户和授权重复；本期不需要 PR、Review、Issue |

## 本决策未解决的问题

- **多副本、备份与恢复**：第一阶段单实例，留给运维 ADR。
- **快照、Workspace 分支和交付 ref 的回收与保留期**：第一阶段全部保留，回收策略要和 Revision 的保留策略一起定。
- **Workspace 分支的非快进改写**：需要另行设计受控的改写操作，并保证进行中的发布仍然取得到源提交（D5）。
- **多分支、Tag 的全量同步，以及 LFS 和 submodule**：由 `cloud/project` 的同步 ADR 决定第一阶段怎样处理，本文的 namespace 已经为多分支留出位置。
- **纵深安全加固**：第一阶段以跑通与正确为目标，以下留给后续决策，各后续 ADR 在自己的“未解决的问题”里登记：
  - Node 以 Node 身份读取 checkout 对象库（借用 alternates）时，工作负载可控文件带来的攻击面；工作负载与插件进程读不到 git-access 私钥的前提；
  - 执行器出口检查的 DNS rebinding；
  - 发布审批展示 diff、限制目标分支、防止经 workflow 读取外部仓库的 secrets；
  - Credential Store 兑换时核对仓库作用域。

## 对现有 Git Server 实现的复用

- **直接复用**：
  - Git Server 程序框架、`git http-backend` 调用、监听结构；
  - 503 fail closed；
  - `pre-receive` 的 token 加 socket 通道；
  - 原子建仓；
  - 交付 ref 规则；
  - 未合并的 cluster#17 只作参考：Sandbox Server 签发 git-access 身份由内部线重新实现。
- **重写**：
  - 判定函数和返回结构，改为 D3 的规则集；
  - `pre-receive` 的逐 ref 判定，改为按 namespace 执行规则；
  - 仓库配置按监听注入（D2）。
- **新增**：
  - 去掉对客户端 `Git-Protocol` 头的任何假设：读侧不隐藏，v0、v2 行为一致；
  - 服务仓库监听和服务身份；
  - Operation 上下文的解析；
  - 配额。

## 落地顺序

1. **底座**：实现 D3 的规则集和 `pre-receive` 改造，只产出现有的交付规则；同时提供只用于验收的预置工具，不进生产镜像。
   - **Cloud 存储**：Binding、当前快照，以及同步与发布 Operation 的最小存储，字段一次定齐：`id`、`type`、`tenant_id`、`project_id`、
     `binding_id`、`state`、`trigger`、`error_code`、`created_at`、`finished_at`；Binding 上的补同步标记 `resync_requested`。
     状态枚举与终态按 D5 第 3 项，以 CHECK 约束固定；每个 Binding 至多一个未结束同步的部分唯一索引也在这里建。外部线只增加
     租约、凭据版本、结果明细等字段，不增删状态；内部线只读这些字段。
   - **建 Project**：同一事务建立它的 Binding 和一行 `pending`、`trigger = project_created` 的同步 Operation。两条线看到的 Project
     从一开始就是这个形状，替身和真实流程同形。
   - **生命周期**：实现 D5 第 3 项中进入 `deleting` 时的取消、“未结束”的判定，以及 create_project 失败时把 Project 置为 `active`。
   - Workspace 的 clone 步骤怎样等待和结束，由内部线在领取时复核：有当前快照就继续；没有快照、而该 Binding 最近的同步已经以
     终态失败，就以失败结束；否则继续等待。底座不提供额外的“结束等待者”函数。
   - **ref 查询的扩展**（D6）：覆盖本文全部 namespace，并支持按前缀列出。现有实现只接受交付 ref；外部线的核验、
     `expectedTrackingOid` 的读取和发布源校验都依赖它。
   - **部署**：Git Server 以三个监听接进 cluster 的 Compose。服务仓库监听在底座就开启，并由管理面签发 Repo Executor 的服务证书与 CA，
     外部线开工时不必再改 Git Server 的部署和启动参数。
   - **预置工具** `git-server-preset`：Go 程序，放在 cloud 仓库的 `cmd/`，随开发镜像（devsetup）进入 cluster 开发栈，不进生产镜像。
     它在 Git Server 容器里直接操作仓库卷（不存在时以根决策 D1 的原子方式建仓），并直接写 Cloud 数据库。子命令：

     | 子命令 | 作用 |
     | --- | --- |
     | `snapshot --project <id> [--commit <oid> \| --synthetic <n>]` | 写入 `snapshots/<oid>` 与跟踪 ref，登记一行 `succeeded` 的同步，推进当前快照；`--synthetic` 生成一条有 n 个提交的历史 |
     | `ref --project <id> --ref <name> --commit <oid>` | 写入任意 namespace 的一个 ref，例如作为发布源的 Workspace 分支 |
     | `operation --project <id> --type <sync\|publish> --state <state> [--error-code <code>]` | 写入一行给定状态的 Operation，用于等待、失败、删除等场景 |
     | `check --project <id>` | 契约断言：检查 Git Server 上的 ref 与 Cloud 记录是否符合 D5 |

     输出一行 JSON（`{"ok": true, ...}` 或 `{"ok": false, "violations": [...]}`）；成功退出码为 0，断言失败为 1，用法错误为 2。
     内部线的验收脚本和外部线都调用它。`check` 也用于外部线确认真实同步的结果，保证两边验收的是同一份契约。
   - **公开 API 的约定**：两条线新增的公开路由都放在现有前缀 `/api/v1/tenants/{tenantId}/` 下，Project 相关的放在
     `/api/v1/tenants/{tenantId}/projects/{projectId}/` 下。运行公开的 `preparation` 新增 `syncErrorCode`：clone 步骤以
     `repository_sync_failed` 结束时，填入该同步 Operation 的 `error_code`（只取外部线定义的安全错误码），否则为 `null`。
   - **迁移编号**：迁移 append-only，文件名就是迁移身份。两条线不预占号段，各自的 PR 在合并前按 main 上的最新编号顺延重编号；
     底座的表一次建齐，减少两条线的迁移互相依赖。
   - 完成条件：交付行为不变，现有测试全部通过。
2. **内部线和外部线并行**：
   - 内部线：git-access 签发、沙盒的 workspace 规则、Node 从快照 clone 与 `ora-git`，以及按
     [Revision Git 交付决策](../revision/20261012-git-server-delivery-and-verified-delivery-refs.md)实现交付推送与续作拉取，同时删除
     bundle 的代码与协议字段。
   - 外部线：服务仓库监听、服务身份、Operation 规则、执行器、Credential Store、同步与发布。
   - 两者改动的是不同的规则来源和监听，互不阻塞；各自用预置工具替代对方，独立做端到端验收。
   - 完成条件：不变量 1–10 有使用真实 `git` 客户端的集成测试证据，测试同时带两种身份。
3. **全链路验收**：两条线都合并后，用真实外部仓库跑一遍从同步到 Workspace、交付、发布再同步的完整链路。

从第 2 步内部线合并起，Workspace 只从快照 clone，不再有外部 clone 与 bundle；外部线合并之前，只有用预置工具准备了快照的
Project 能创建 Workspace。保持不变：会话 JSONL 的存放位置、根决策的网络划分。
