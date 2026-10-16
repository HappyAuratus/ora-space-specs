---
status: proposed
---

# Git Server 成为每个 Project 的代码中心：ref namespace、规则集授权与两类写入者的契约

Git Server 不再只存放交付 ref。从本决策起，它是每个 Project 唯一的代码中心，仓库仍是 `/{tenantId}/{projectId}.git`。
这个仓库里只有两类写入者：

- **沙盒执行**：经现有仓库监听、用 git-access 证书访问，写 Workspace 分支和交付 ref；
- **Repo Executor**：经新的服务仓库监听、用服务证书访问，写外部仓库同步得到的快照和跟踪 ref。

每个请求带一个上下文头。Git Server 按 Cloud 当前状态把它解析成一份 ref 规则集，`pre-receive` 再按规则逐个 ref 判定。
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

- Revision 的交付与续作继承 [Revision Git 交付决策](../revision/20261012-git-server-delivery-and-verified-delivery-refs.md)，本文不改；
- 会话 JSONL 仍在对象存储；
- 本文不迁移已有的 bundle Revision；
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
  project:  (tenantId, projectId)
  readable: [ref 或 ref 前缀]                  // 本次请求能看到的 ref
  writable: [{ ref 或 ref 前缀, rule, expectedOld? }]
            // rule ∈ create-only | fast-forward | tracking-cas
}
```

- **沙盒执行**：沿用根决策 D3 的有效性条件。执行属于证书中的运行时和当前代次，未结束，未被强停，Workspace 处于允许的阶段。
  授权内容由执行类型决定，具体映射由 `cloud/controller-integration` 后续 ADR 规定，但不能超出 D4 中“沙盒”一列允许的范围。
- **Operation**：Operation 必须存在，处于持有租约的运行状态，租约未过期，Project 与 Binding 匹配。授权内容由 Operation 类型
  决定（见 D5），不能超出 D4 中“执行器”一列允许的范围。
- `pre-receive` 在写入前重新解析同一份授权，逐个 ref 核对规则。两次请求之间执行结束、代次变化或 Operation 被取消，写入都会被拒绝。
- 数据库不可用或判定出错，一律返回 `503`，不缓存授权。

**ref 广告**：读取（upload-pack）和推送（receive-pack）开始时，Git 都会先列出仓库里的 ref 和它们指向的 OID。Git Server
按 `readable` 为每次请求注入 `transfer.hideRefs`，让两种广告都只列出授权内的 ref；推送时再加上本次可写的 ref，以便客户端
知道旧值。不能只隐藏读取的广告：否则执行器同步推送时、沙盒推送 Workspace 分支时，都能看到整个 Project 的全部 ref。

**按 OID 读取**：显式把 `uploadpack.allowAnySHA1InWant`、`allowTipSHA1InWant`、`allowReachableSHA1InWant` 设为 `false`，
未广告的对象不能按 OID 直接请求。protocol v0 与 v2 下的这一行为，都必须有集成测试证明。

**边界**：同一 Project 仓库内，ref 可见性是最小权限的整理，**不是**对象级的保密边界。保密边界是 Project 仓库本身：跨 Project、
跨租户的对象永远不可达。同一 Project 的 Workspace 本来就是协作关系。

为什么改成规则集：现有实现的判定结果只有 `Mode` 和一个 `DeliveryRef`，`pre-receive` 也只认这一个 ref。新增调用方后，如果
继续在判定函数里写 if-else，每加一种调用方都要同时改 SQL、返回结构和 hook。改成规则集后，“谁能写什么”由 Cloud 的数据一处决定，
Git Server 只负责执行规则。这也让两条线可以各自添加自己的规则来源，而不改同一段判定代码。

### D4：ref namespace 与写入规则

| ref | 写入者 | 规则 | 读取者 |
| --- | --- | --- | --- |
| `refs/ora/upstreams/<bindingId>/snapshots/<commitOid>` | 执行器（同步 Operation） | create-only；名称中的 OID 必须等于 ref 的目标 | 该 Project 中获授权的沙盒执行；执行器 |
| `refs/ora/upstreams/<bindingId>/heads/<branch>` | 执行器（同步 Operation） | tracking-cas：旧值必须等于 Operation 记录的 expected old（零表示首次创建）；新值必须有对应的快照 ref，可以是仓库里已有的，也可以是同一次推送里新建的；允许非快进 | 同上 |
| `refs/heads/workspaces/<workspaceId>/<branch>` | 沙盒（该 Workspace 当前代次的执行） | fast-forward：只能新建或快进 | 该 Project 中获授权的沙盒执行；执行器（发布 Operation） |
| `refs/ora/deliveries/<deliveryWorkId>` | 沙盒（对应的交付执行） | create-only（根决策 D4，不变） | Cloud ref 查询；续作执行；执行器（发布 Operation） |

其他规则：

- 任何调用方都不能删除 ref，也不能写表外的 ref。
- 沙盒永远不能写 `refs/ora/upstreams/**`；执行器永远不能写 `refs/heads/workspaces/**` 和 `refs/ora/deliveries/**`。
- 仓库级的 `receive.denyNonFastForwards` 改为由 hook 按 namespace 执行：跟踪 ref 要能跟随外部 force-push，其余 namespace
  仍然拒绝非快进。`receive.denyDeletes` 保留为仓库级配置。
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
| 可写 | 本 Binding 下 `snapshots/` 中名称与目标一致的新 ref（create-only）；`heads/<branch>`（tracking-cas，expected old 在 Operation 创建时记录） | 无 |
| 可读 | 该 Binding 的 upstream namespace | 只有 Operation 指定的源 ref |

- 同步的快照授权是按前缀给出的，不是某个具体 OID：创建 Operation 时，Cloud 还不知道外部仓库会同步到哪个提交，执行器 fetch
  之后才知道。约束靠 D4 的 create-only 和“名称与目标一致”两条规则。
- 发布 Operation 记录“源 ref + 源提交”，执行器只读源 ref。创建之后，源 ref（比如 Workspace 分支）可能继续前进；但 Workspace
  分支只能快进（D4），源提交始终是它顶端的祖先，所以从源 ref 一定取得到源提交。以后如果允许 Workspace 分支非快进改写，必须
  同时保证进行中的发布仍然取得到源提交。
- 权限跟着 Operation 走：同一个执行器进程处理不同 Operation 时，各自只有对应的权限；Operation 结束、取消或租约过期，同一个
  ID 立即失效。

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
8. 读取和推送的 ref 广告都只列出授权内的 ref；protocol v0 和 v2 下按 OID 请求未广告对象的行为都有测试证据。
9. 跨租户、跨 Project 的对象不可达。
10. 沙盒访问不到服务仓库监听和 ref 查询监听。
11. Workspace 的代码输入总是固定的快照和提交。
12. Binding 的当前快照只在 Cloud 核验快照 ref 之后更新，总是等于最新一次核验通过的同步结果。
13. Cloud 不写 Git Server。

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
| Gitea、Forgejo | 自带用户、权限和 Token 体系，与 Cloud 的租户和授权重复；本期不需要 PR、Review、Issue |

## 本决策未解决的问题

- **多副本、备份与恢复**：第一阶段单实例，留给运维 ADR。
- **快照、Workspace 分支和交付 ref 的回收与保留期**：第一阶段全部保留，回收策略要和 Revision 的保留策略一起定。
- **Workspace 分支的非快进改写**：需要另行设计受控的改写操作，并保证进行中的发布仍然取得到源提交（D5）。
- **多分支、Tag 的全量同步，以及 LFS 和 submodule**：由 `cloud/project` 的同步 ADR 决定第一阶段怎样处理，本文的 namespace 已经为多分支留出位置。

## 对现有 Git Server 实现的复用

- **直接复用**：
  - Git Server 程序框架、`git http-backend` 调用、监听结构；
  - 503 fail closed；
  - `pre-receive` 的 token 加 socket 通道；
  - 原子建仓；
  - 交付 ref 规则；
  - Sandbox Server 按根决策 D2 签发 git-access 身份。
- **重写**：
  - 判定函数和返回结构，改为 D3 的规则集；
  - `pre-receive` 的逐 ref 判定，改为按 namespace 执行规则；
  - 仓库级的 `denyNonFastForwards`，改为由 hook 执行。
- **新增**：
  - 读取和推送两种 ref 广告的 `transfer.hideRefs` 注入；
  - 服务仓库监听和服务身份；
  - Operation 上下文的解析；
  - 配额。

## 落地顺序

1. **底座**：实现 D3 的规则集和 `pre-receive` 改造，只产出现有的交付规则；同时提供只用于验收的预置工具，可以直接写入任意
   namespace 的 ref 和 D5 的当前快照记录，不进生产镜像。
   - 预置工具还能写入验收所需的最小 Cloud 记录，不启动沙盒与 Node：Workspace 记录（带创建者，使运行时使用权限的检查可以进行）、
     处于给定步骤和状态的 Workspace operation（例如等待快照的 clone 步骤）。外部线用它验收发布的源 ref 权限和结束等待者，
     不需要内部线的 Controller 与 Node。
   - D5 第 2 项需要的 Binding 与当前快照两张表也由底座建立。
   - 底座同时建立同步与发布 Operation 的最小存储：Operation ID、类型（`repository_sync`、`repository_publish`）、Binding、状态。
     外部线在其上增加租约、结果等字段；内部线读它判断一个 Binding 有没有未结束的同步，以决定 Workspace 的 clone 步骤是否等待快照。
   - 底座提供一个 Cloud 函数：按 Binding 把等待快照的 Workspace operation 以失败结束，并记录导致失败的同步 Operation 与错误码。
     外部线在登记同步终态的事务里调用它，预置工具推进同步状态时也调用它。
   - 这些存储与函数放在底座，两条线开工前交接的存储与规则就已经存在：两条线都只依赖底座，判断的是同一段代码。
   - 底座同时提供共享的契约断言：检查 Git Server 上的 ref 和 Cloud 的记录是否符合 D5。内部线用它确认预置数据，外部线用它确认
     真实同步的结果，保证两边验收的是同一份契约。
   - 完成条件：交付行为不变，现有测试全部通过。
   - 意义：两条线从这一步之后都能用预置数据独立验收。
2. **内部线和外部线并行**：
   - 内部线：沙盒的 workspace 规则、`transfer.hideRefs` 注入；
   - 外部线：服务仓库监听、服务身份、Operation 规则。
   - 两者改动的是不同的规则来源和监听，互不阻塞。
   - 完成条件：不变量 1–10 有使用真实 `git` 客户端的集成测试证据，测试同时带两种身份。
3. **汇合**：Workspace 的输入切换为 D5 的快照输入。从这一步起，Node 不再从外部仓库 clone；这一步依赖外部线的同步（包括公开仓库）
   已经可用。
4. **清理**：删除 bundle 的协议字段和对象存储中的代码路径。

保持不变：交付和续作的行为、会话 JSONL 的存放位置、根决策的网络划分。
从第 3 步开始改变：Workspace 的代码来源。
