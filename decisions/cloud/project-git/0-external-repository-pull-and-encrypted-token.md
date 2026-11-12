---
status: approved
---

# 项目仓库：一个项目绑定一个外部仓库的一个分支，Cloud 在进程内拉取到本机镜像并支持手动同步，私有仓库令牌加密存 PostgreSQL；第一版没有 Git server，不推回外部

> 历史决策：D5 里“交付：Agent 的改动留在 Node 的工作目录里”一项由[回合交付](20261112-single-machine-turn-delivery.md)修改：每轮结束由 Cloud 交付到镜像里的交付 ref，变更视图和验收读交付。其余决策不变。

每个项目绑定一个外部仓库的一个分支。Cloud 进程自己把它拉取到数据卷上的镜像仓库：建项目时拉一次，之后由用户点
“同步”再拉，不轮询，没有 webhook。每次拉取成功后，项目的当前快照指向新提交，之后新建的会话以它为基准。私有仓库用
一个按项目保存的令牌访问，令牌用部署配置的密钥加密后存在 PostgreSQL，只在 Cloud 进程里使用，不下发给 Node。
Node 仍像现在一样从 Cloud 本机的镜像只读 clone。第一版没有 Git server，Agent 的改动不推送到 Cloud，也不推回外部仓库。

本 ADR 处于 `approved`：owner 已评审通过，手动同步和令牌尚未实现；它是领域 `cloud/project-git` 的根决策。前提是
[Cloud 单进程根决策](../lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（下称“A1”）。2026-11-11 owner 在
circus #1073 把第一版收窄为“只拉进来，支持私有仓库，推回以后再设计”，本文据此重写：原先的内嵌 Git server、ref 写入
规则、推送事件、Node 改造、每轮交付、webhook 和推回都已删除。读取的代码版本为 cloud `80e9c9f1`
（`v2-server/internal/projectrepos`、`nodeservices`、`pluginmarket`）。核心测试用例见
[项目仓库用例](../../../test-cases/cloud/project-git/external-repository-pull.md)。

范围与兼容性边界：

- **涉及**：cloud 的 `v2-server/`（项目仓库、同步、令牌）。desktop 的 Node 不改。
- **不做数据迁移**：v1 的仓库、Binding、Operation 和 Credential Store 里的凭据都不迁移。现有 v2 项目的镜像原地继续用。

## 现状与继承

| 前序约定或现状 | 本文处理 |
| --- | --- |
| A1 D1：repo-executor、credential-store 收进 Cloud 进程；git-server 退役 | 继承 |
| A1 D4：外部仓库令牌不下发给 Node | 继承（D4） |
| A1 D3：Cloud 重启时未结束的工作标为中断，不自动重做 | 继承，适用于同步（D3） |
| v2-server `projectrepos`：建项目时 `clone --mirror` 一次，之后不 fetch，并删掉镜像的 `remote.origin`；只接受不带凭据的 `https` 地址；默认分支由用户给出 | 修改：增加手动同步和私有仓库令牌（D3、D4）。地址和分支的规则继承（D1） |
| v2-server：项目记录一个固定的提交和快照 ref `refs/ora/upstreams/<projectId>/snapshots/<commitOid>`，所有会话从它 clone | 修改：当前快照随同步推进（D2） |
| v2-server `nodeservices`：Node 经本机服务只读访问镜像（只放行 upload-pack） | 继承，不变 |
| v2-server `pluginmarket`：市场来源的凭据用环境变量里的一把密钥以 AES-256-GCM 加密后存 `bytea` | 沿用同样的做法（D4） |

## 决策

### D1：一个项目绑定一个外部仓库的一个分支

- 地址只接受 `https://`，不带用户信息、查询串和片段；分支是具体的分支名，不能是 `HEAD`。
- 两者在建项目时给出，之后不能修改；填错了删除项目重建。这个分支就是项目的默认分支。
- 建项目时同步拉取一次；拉取失败则项目不创建。

### D2：镜像和当前快照

- 每个项目在 Cloud 数据卷上有一个裸仓库（镜像）。项目之间不共享对象。
- 每次拉取成功，Cloud 在镜像里写一个快照 ref `refs/ora/upstreams/<projectId>/snapshots/<commitOid>`，并在 PostgreSQL 里
  把项目的**当前快照**指向这个提交。快照 ref 只新建、不改写、不删除。
- 新建的会话目录以当时的当前快照为基准；已经建好的会话目录不随同步变化。
- Node 仍经 Cloud 的本机服务只读 clone 镜像，拿到的输入是 `{仓库地址, 快照 ref, 基准提交}`，clone 后核对提交。

### D3：手动同步

- **触发**：项目所在团队的成员在项目页点“同步”。没有 webhook，不轮询，建会话时也不隐式同步。
- **一次只跑一个**：每个项目同时最多一次同步；进行中再点，返回正在进行的那一次。
- **步骤**：Cloud 把绑定的分支从外部仓库取进镜像（每次显式给出地址，镜像不保留 remote 配置），写入新的快照 ref，
  再推进当前快照。只取这一个分支，不取 tag，完整历史，不递归 submodule，LFS 只同步指针。
- **外部分支被强推**：照常跟随，旧快照保留，结果里注明“外部分支被改写”。
- **外部分支被删除、地址不可达、认证失败、超时**：同步失败，当前快照不变，原因显示在项目页。失败不自动重试。
- **Cloud 重启**：进行中的同步按中断结束，不自动重跑；镜像里可能留下已取入但未被引用的对象，不影响使用。
- **Git 的加固配置**沿用现状：只允许 `https`，不跟随重定向，hooks 关闭，环境清空。

### D4：私有仓库令牌按项目保存，加密后存在 PostgreSQL

- **归属**：每个项目至多一个令牌。建项目时由创建者填写；之后由团队管理员或项目创建者更换和删除。公开仓库不填，
  匿名拉取。
- **保存**：用部署配置的一把密钥以 AES-256-GCM 加密后存在 PostgreSQL，附加数据绑定租户、项目和仓库地址，密文不能
  挪到别的项目上用。密钥缺失时 Cloud 照常启动，需要令牌的拉取失败并写明原因。
- **使用**：明文只在 Cloud 进程里解密，在执行一条外部 Git 命令时交给这个子进程，不出现在命令行参数、地址、Git 配置、
  日志和 API 应答里。API 只返回“已配置”和最后更换的时间。
- **不下发给 Node**（A1 D4）。
- **被外部平台拒绝**：拉取因认证失败结束时，项目页提示更换令牌；Cloud 不自动停用它。

为什么按项目存一个令牌：v1 分团队凭据和个人凭据，带归属、冻结和审批三组状态，并放在独立的凭据服务里。第一版只需要
“能拉私有仓库”，一个项目一个令牌就够。

### D5：第一版不做的

- **Git server**：Cloud 不提供可写的 Git 入口，Node 和 Agent 不能向 Cloud 推送。
- **交付**：Agent 的改动留在 Node 的工作目录里（A1 D3）；变更视图由 Cloud 读本机的工作目录得到，任务在回合结束时
  进入待验收，都是 v2-server 的现状。
- **推回外部仓库、webhook、修改绑定、多个仓库或分支**。

## 被取代的 v1 决策

继承 A1“被取代的 v1 决策”中 Git 线一组的安排；标记之前它们的状态不变，标记时机是 I11（circus #1061）删除 v1 Git 线时。

| v1 ADR | 处理 |
| --- | --- |
| [`cloud/project/20261016-repository-binding-and-snapshot-sync.md`](../project/20261016-repository-binding-and-snapshot-sync.md) | 由 D1–D3 取代 |
| [`cloud/project/20260927-repository-credential-continuity.md`](../project/20260927-repository-credential-continuity.md)、[`cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md`](../identity-access/20261016-git-credentials-in-independent-credential-store.md) | 由 D4 取代 |
| [`cloud/repo-executor/`](../repo-executor/0-repo-executor-sole-external-git-boundary.md) | 由 D3 取代：拉取在 Cloud 进程内执行 |
| [`cloud/git-server/`](../git-server/0-project-code-hub-git-server-over-https.md) 全部、[`cloud/revision/`](../revision/0-cloud-owned-object-store-and-verified-revisions.md) 全部、[`cloud/project/20261017-explicit-fast-forward-publish.md`](../project/20261017-explicit-fast-forward-publish.md) | 放弃：第一版没有 Git server、交付和推回（D5），以后需要时重新写决策 |

## 不变量

1. 每个项目有且只有一个镜像仓库；项目之间不共享对象。
2. 项目的当前快照总是指向镜像里一个存在的快照 ref 所指的提交，并且只在一次拉取成功之后改变。
3. 快照 ref 一经写入不再改变。
4. 每个项目同时最多一次同步。
5. 会话目录的基准提交等于建它时的当前快照，之后不随同步变化。
6. 外部仓库令牌的明文只出现在 Cloud 进程和它启动的外部 Git 子进程里，不下发给 Node，不进日志和 API 应答。
7. Cloud 只在建项目和用户发起同步时访问外部仓库，并且只读。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 内嵌 Git server，Agent 推送，每轮交付（本文 2026-11-11 之前的版本） | 成果在 Cloud 上有权威副本，多机可用 | 要改 Node、交付登记和变更视图；单机下 Cloud 读得到工作目录，第一版用不上（A1 D2、D7） |
| webhook 触发同步 | 外部一推送就同步 | 要对外暴露入口、管理密钥和去重；手动同步够用 |
| 周期轮询 | 不用人点 | 多数请求无效；#1043 Q3 已否定 |
| 第一版就支持推回 | Agent 的成果能回到外部仓库 | owner 定为推回以后再设计；它依赖交付怎么做 |
| 令牌放在独立的凭据服务里 | Cloud 被攻破时令牌不泄露 | 多一个服务和一条兑换链路；A1 D1 已定收进 Cloud 进程 |
| 团队和个人两类凭据 | 能区分是谁的身份 | 三组状态，第一版只读拉取用不上（D4） |

## 本决策未解决的问题

1. **推回外部仓库和 Agent 改动的交付**：以后一起设计。
2. **外部地址的限制**：第一版不限制地址指向哪里（企业的 Git 通常就在内网），能建项目的成员可以让 Cloud 向任意
   `https` 地址发 Git 请求。部署给互不信任的成员之前要先定规则。
3. **密钥轮换**：第一版只有一把密钥，更换它需要重新填写所有令牌。
4. **自动同步、修改绑定、多仓库**。

## 落地顺序

1. **I7（circus #1057）**：D2 的当前快照、D3 的手动同步、D4 的令牌。完成条件：私有仓库能建项目和同步；同步后新会话
   从新提交开始、旧会话不变；强推、分支删除、认证失败各有测试；令牌不出现在日志、API 应答和 Node 的配置里；
   `task v2:demo` 照常可用。
2. **I11（circus #1061）**：删除 v1 Git 线，按上一节标记 ADR。
