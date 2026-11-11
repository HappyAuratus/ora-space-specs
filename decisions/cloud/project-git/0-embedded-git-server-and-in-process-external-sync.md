---
status: approved
---

# 项目 Git：Git server 内嵌在 Cloud 进程里并在推送完成时发事件，Node 用项目令牌从 origin 直接拉取和推送，每轮结束交付一次，外部仓库的同步、推回和凭据都在 Cloud 进程内，取代 v1 的 git-server、repo-executor 和 credential-store

每个项目在 Cloud 的数据卷上有一个裸仓库，由 Cloud 进程自己提供 smart HTTP，路径是 `/{tenantId}/{projectId}.git`。
推送由 Cloud 处理：先按 ref 规则判定，Git 写入完成后在进程内发出 ref 变化事件，不用 hook、不用 ref 查询、不轮询。
Node 用[项目令牌](../../../domain/glossary.md#project-token)访问：整个项目仓库可读，只能写工作分支命名空间和交付 ref。
Node 为每个项目保留一份本地副本，只在第一次 clone，之后增量 fetch；每个会话的目录从这份副本复制出来，并配置指向
Git server 的 `origin`，Agent 自己 `git fetch`、`git push`，`ora-git` 退役。每一轮对话结束时，Node 把工作目录做成一个
内部快照提交，推成一个只能新建的交付 ref；交付 ref 到达后 Cloud 才登记成果、让任务进入待验收，变更视图也从 Git server
计算。外部仓库的拉取（webhook 加手动同步）和推回（用户显式发起、只快进）都在 Cloud 进程里执行；外部仓库令牌每个项目
一个，用部署密钥加密后存在 PostgreSQL，不进 Node。

本 ADR 处于 `approved`：owner 已评审通过，尚未实现；它是新建领域 `cloud/project-git` 的根决策。前提是
[Cloud 单进程根决策](../lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（下称“A1”）、
[多机执行层根决策](../execution-host/0-host-relayed-control-and-pinned-projects.md)（下称“A2”）和
[Node 容器根决策](../node-container/0-host-managed-docker-and-two-egress-modes.md)（下称“A3”）。依据是 circus #1043 的
Q3 结论（G1–G6），以及 owner 在 #1072 定下的六点：origin 直推、每轮结束交付、交付 ref 到达才进待验收、对话也交付、
推回结果不明时标中断、每个项目一个团队令牌。读取的代码版本为 cloud `80e9c9f1`（`v2-server/internal/projectrepos`、
`nodeservices`、`workspacefs`、`taskrun`，以及 v1 的 `internal/gitserver`、`internal/repoexec`）、desktop `dcff8afe`
（`apps/ora-node`、`apps/ora-git`、`crates/node-protocol`）、cluster `0b8d913`。核心测试用例见
[项目 Git 用例](../../../test-cases/cloud/project-git/embedded-git-server-and-external-sync.md)。

范围与兼容性边界：

- **涉及**：cloud 的 `v2-server/`（Git server、事件、外部同步、推回、凭据、交付登记、变更视图）；desktop 的 Node
  （项目副本、会话目录、凭据交给 Git 的方式、回合结束交付）。
- **Node 的消息要改**：A1 和 A2 说的“Node 协议不变”指帧、握手、心跳和 ACK。本文修改其中 clone 和交付两组消息的内容
  （D6、D7），并删除 `ora-git`；Node 一侧的细则由 I6（circus #1056）在 `decisions/node/` 下写后续 ADR，本文只定 Cloud 和
  Node 之间的契约。
- **不在本文**：项目令牌的签发和吊销见 A2 D6；容器里文件的位置和权限见 A3 D2；模型代理见 A5（#1049）。
- **演示约束**：继承 A1 D7。演示结束前 `task v2:demo` 仍走“建项目时 mirror clone 一次、Node 从本机配套服务 clone、
  后端直接读 checkout”的路径；本文的路径默认关闭，演示结束后由 I10（#1060）切换。
- **不做数据迁移**：v1 Git server 的仓库、Binding、Operation 和 Credential Store 里的凭据都不迁移。切换时现有 v2 项目的
  mirror 按 D1 的布局重新整理，由 I6 给出步骤。

## 现状与继承

| 前序约定或现状 | 本文处理 |
| --- | --- |
| A1 D1：git-server、repo-executor、credential-store 收进 Cloud 进程 | 继承，并规定收进来之后的规则（D1、D8–D10） |
| A1 D3：崩溃和断开按中断处理，不续跑；已推送到 Git server 的 ref 保留 | 继承。交付、同步和推回被打断时都按中断处理（D7、D8、D9） |
| A1 D4：外部仓库凭据不进沙盒；项目是信任边界 | 继承。写权限按项目判定，不按会话收窄（D3） |
| A2 D6、D7：Node 用项目令牌访问 Git，令牌经请求头送出，不写进 checkout 和 git 配置 | 继承，并规定令牌能读写什么（D3），以及 Agent 自己的 git 怎么取得令牌（D6） |
| A3 D2：项目令牌在 `/run/ora`，工作负载读不到 | 继承文件的位置和权限；Agent 经 Node 提供的凭据助手取得令牌（D6），A3 已接受同项目的 Agent 能拿到它 |
| v1 [Git Server 代码中心契约](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md) D4：四类 ref 和三种写规则 | 继承 ref 的名字和规则，写入者改变：外部同步的两类由 Cloud 自己写（D2） |
| v1 Git Server 根决策：只认客户端证书、Cloud 不写 Git Server、靠 ref 查询确认 | 取代：令牌认证、Cloud 直接写、进程内事件（D1、D3、D4） |
| v1 [Node 快照 clone 与 ora-git](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)：每个 Workspace 完整 clone，checkout 没有 remote，Agent 经 `ora-git` 请 Node 代推 | 修改：项目副本加增量 fetch，会话目录有 `origin`，`ora-git` 退役（D6） |
| v1 [Node 交付推送](../../node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)：会话结束后交付，推交付 ref 并顺带快进 Workspace 分支，上传会话 JSONL | 修改：每轮结束交付，只推交付 ref，不上传 JSONL（D7）。内部快照的做法继承 |
| v1 外部同步、显式发布、凭据连续性、Credential Store | 取代：同步和推回在进程内执行，凭据加密入库，规则大幅收窄（D8–D10） |
| v2-server 现状：不 push；任务在回合结束时进待验收；变更视图读本机 checkout | 修改（D5、D7） |

## 决策

### D1：每个项目一个裸仓库，由 Cloud 进程提供 smart HTTP

- **仓库**：每个项目有且只有一个裸仓库，放在 Cloud 数据卷的 `repositories/<tenantId>/<projectId>.git`，建项目时创建。
  项目之间不共享对象目录，也不用 `objects/info/alternates`。数据库只存租户和项目标识，不存仓库的绝对路径。
- **入口**：Cloud 的 HTTPS 入口提供 `/{tenantId}/{projectId}.git` 下的三个 smart HTTP 请求：`info/refs`（两种 service）、
  `git-upload-pack`、`git-receive-pack`。dumb HTTP 和其他路径返回 `404`。两个标识都必须是规范的小写 UUID。
- **执行**：每个请求由 Cloud 启动一个 Git 子进程处理，子进程的环境清空，hooks 关闭，不读系统和全局配置。A1 的“一个
  进程”指 Cloud 的服务只有一个；这些短命的 Git 子进程不保存状态，也不监听端口。
- **固定配置**：`receive.denyDeletes=true`、`receive.denyNonFastForwards=true`、`receive.fsckObjects=true`，不接受
  push options，`uploadpack.allowAnySHA1InWant=false`。单次推送的大小上限由部署配置给出（`receive.maxInputSize`）。
- **备份**：仓库目录和 PostgreSQL 一起备份（A1 D2）。
- **维护**：Cloud 按仓库串行地运行 `git gc --auto`，与同一仓库的写入互斥。第一版所有 ref 都保留，不清理。

为什么内嵌：独立的 Git server 要有自己的监听、身份和授权查询，推送完成后还得想办法通知 Cloud。内嵌之后授权是一次
函数调用，事件是进程内调用（D4）。代价见 A1 D1：一次很大的推送占用的是 Cloud 进程所在机器的资源，所以设大小上限。

### D2：ref 命名空间和写规则

沿用 v1 契约的名字，因为 Node 协议已经按这些名字校验：

| ref | 含义 | 谁写 | 规则 |
| --- | --- | --- | --- |
| `refs/ora/upstreams/<projectId>/heads/<branch>` | 外部分支的跟踪 ref | 只有 Cloud（D8） | 跟随外部，允许非快进；不删除 |
| `refs/ora/upstreams/<projectId>/snapshots/<commitOid>` | 一次同步得到的快照 | 只有 Cloud（D8） | 只能新建；名字里的提交必须等于它指向的提交 |
| `refs/heads/workspaces/<workspaceId>/<branch>` | Agent 的工作分支 | 项目令牌 | 只能新建或快进 |
| `refs/ora/deliveries/<deliveryId>` | 一次交付的成果 | 项目令牌 | 只能新建，之后不可变 |

- `<workspaceId>` 是 Cloud 给一个会话目录分配的标识（规范的小写 UUID）；同一次工作流运行的 Agent 节点共用一个。
- `<deliveryId>` 由 Cloud 在要求交付时生成（规范的小写 UUID），Node 不能自己选。
- 第一版一个项目只绑定一个外部仓库，所以 v1 名字里 Binding 标识的位置直接用项目标识。
- 表外的任何 ref（包括 `refs/heads/` 下其他名字和 `refs/tags/`）谁都不能经推送写入。任何 ref 都不能经推送删除。

### D3：项目令牌能读整个项目仓库，只能写工作分支和交付 ref

- **认证**：请求头 `Authorization: Bearer <项目令牌>`。Cloud 按 A2 D6 查到令牌所属的项目，路径里的租户和项目必须与它
  一致。令牌无效、已吊销、项目不符或仓库不存在，都返回同一个 `404`，不透露仓库是否存在。
- **读**：获准后可以读整个项目仓库，ref 广告不隐藏任何 ref。隔离边界是项目仓库本身。
- **写**：只接受 D2 表里“项目令牌”两行。Cloud 在把请求交给 Git 之前解析推送的命令列表，逐条判定：
  - 目标必须落在两个可写命名空间之一；
  - 交付 ref 必须是新建，并且等于一个 Cloud 已要求、尚未登记的交付（D7）；
  - 不允许删除。

  任何一条不符合，整次推送被拒绝，什么都不写。工作分支的“只能快进”由 Git 的 `receive.denyNonFastForwards` 保证。
- **不按会话收窄**：同一项目的任何会话都能写这个项目的任何工作分支。
- **吊销时效**：每个请求开始时校验一次令牌。传输中途不复查，所以令牌吊销时已经在进行的那次推送仍可能写入；它产生的
  事件按 D4、D7 处理，不会改变已结束执行的业务记录。
- **人直接 clone**：第一版不提供。用户会话不能访问 Git 入口。

为什么不按会话收窄：v1 靠证书里的 Workspace 标识把写权限限制在自己的分支前缀，并随运行阶段失效。项目令牌是按项目
签发的（A2 D6），同项目的会话本来就共用一个容器、能读写彼此的目录（A1 D4 第 4 条），在 Git 入口再分一层挡不住什么。
快进规则保证互相写也丢不了提交，交付 ref 不可变保证成果不被改写。

为什么不用 hook 判定：要判的三条都能在命令列表里看出来（命令列表在 pack 数据之前），快进由 Git 自己保证。不用 hook，
就不需要子进程回调 Cloud 的通道，也不存在“hook 没装上”的失败模式。

### D4：推送完成后在进程内发 ref 变化事件

- Git 子进程成功退出后，Cloud 读出这次推送改动的每个 ref 的新值，在返回 HTTP 应答之前逐个发出事件
  `{tenantId, projectId, ref, 旧值, 新值}`。Cloud 自己写 ref（D8）时发同样的事件。
- 事件只在进程内传递，不落盘、不重放。订阅者的处理必须幂等，并且以仓库里 ref 的当前值为准，不以事件里的值为准。
- **事件丢了怎么办**：ref 写入和事件处理不是一个原子操作，所以每个靠事件推进的业务状态都另有一次直接读仓库的核对：
  - 交付：Node 在控制协议里报告交付结果时，Cloud 直接读交付 ref 再登记一次（D7）；
  - 外部同步：ref 由同步流程自己写，写完直接推进当前快照，不依赖事件（D8）；
  - 工作分支：事件只用来刷新页面，丢了的后果是页面晚一点更新。
- Cloud 重启时不补发事件：重启前未结束的执行都已标为中断（A1 D3），它们的交付不再登记。

为什么不是 v1 的“事务外 ref 查询再登记”：那是因为 Git server 和 Cloud 是两个进程，Cloud 无从知道推送何时完成。现在
读 ref 是读本机文件，事件只是省掉等待，核对仍然存在，只是不需要网络。

### D5：Cloud 订阅的事件和各自推进的状态

| 事件 | Cloud 的动作 |
| --- | --- |
| 工作分支新建或前进 | 找到这个工作区对应的会话或任务运行，经 SSE 通知页面刷新变更视图 |
| 交付 ref 新建 | 按 D7 登记成果；任务运行的任务进入待验收；经 SSE 通知页面 |
| 跟踪 ref 前进并写入新快照（D8） | 项目的当前快照指向新提交；之后新建的会话目录以它为基准，已有的会话目录不变 |

- **变更视图从 Git server 计算**：取这个工作区最近到达的提交（最新的交付 ref，或更晚到达的工作分支提交），与会话目录
  的基准提交比较。Cloud 不再读 Node 的 checkout。
- **回合进行中**：只有 Agent 自己提交并推送时变更视图才更新；未提交的改动要等这一轮结束交付后才看得到。这是相对
  v2-server 现状的退步，owner 已接受。
- “推送时触发工作流”不在第一版。

### D6：Node 每个项目保留一份副本，会话目录配 origin，Agent 直接拉取和推送

- **项目副本**：Node 在数据卷上为项目保留一份裸仓库，只有 Node 管理进程可写。第一次需要时从 Git server clone，之后每次
  建会话目录前增量 fetch，取到 Cloud 指定的基准提交为止。副本不是权威数据（A1 D3），丢了重新 clone。
- **会话目录**：Cloud 下发 `{仓库地址, 基准提交, 工作分支 ref}`。Node 从项目副本把对象复制进新目录（独立的对象库，
  不用 alternates 和 worktree，不是浅克隆），检出工作分支的短名并指向基准提交，核对 HEAD 等于基准提交后才报告成功。
  基准提交在 Git server 上取不到时失败，不用别的提交代替。
- **origin**：会话目录配置 `origin` 指向 Git server：
  - fetch 把 `refs/ora/upstreams/<projectId>/heads/*` 取到 `refs/remotes/origin/*`，把
    `refs/heads/workspaces/*` 取到 `refs/remotes/workspaces/*`；
  - push 把本地分支 `<branch>` 推到 `refs/heads/workspaces/<自己的 workspaceId>/<branch>`。

  这些配置只是默认值，Agent 可以改；能写什么由 D3 在服务端判定。
- **凭据**：Agent 的 git 经 Node 提供的凭据助手取得项目令牌。助手经本地 Unix socket 向 Node 管理进程要令牌，只对
  Git server 的地址作答；令牌文件仍然只有 Node 管理进程可读（A3 D2），轮换后下一次请求就用新令牌（A2 D6）。助手经
  会话的环境变量配置，不写进 checkout 的 `.git/config`。
- **`ora-git` 退役**，`ora-git` 的 socket、账本和固定动词都删除。
- **Node 自己发起的 Git**（项目副本的 fetch、D7 的交付推送）不读会话目录里的配置和钩子：Agent 改过的 `.git/config`
  不能改变 Node 把什么推到哪里。

为什么让 Agent 直接用 origin（owner 2026-11-11 定）：Agent 和它调用的工具本来就会用 `git fetch`、`git push`，固定动词
的 `ora-git` 要靠提示词教会每一种 Agent。v1 不给 remote，是因为证书私钥不能交给工作负载；项目令牌按 A2 D7 已经对
同项目的 Agent 可见，这条理由不再成立。代价：Agent 可以向本项目任何工作分支推送（D3 已接受）；Node 的 clone、交付和
checkout 校验都要改。

### D7：每轮结束交付一次，交付 ref 到达后才登记成果

1. **要求交付**：一轮对话结束（成功或失败）后，Cloud 生成 `deliveryId`，在 PostgreSQL 记下一条等待中的交付
   `{deliveryId, 工作区, 会话, 轮次, 基准提交}`，再让 Node 交付，输入是 `{仓库地址, 交付 ref, 基准提交}`。普通对话和
   任务运行走同一条路径。
2. **Node 交付**：这一轮的 Agent 进程都停下之后，Node 按 v1 内部快照的做法得到最终提交：工作目录没有未提交的改动时
   就是 HEAD，否则是以 HEAD 为父的一个快照提交。快照不改工作区、index 和本地分支，不跑提交钩子，被 `.gitignore` 忽略
   的文件不收录。Node 把最终提交推到交付 ref，然后在控制协议里报告 `{最终提交, 有无改动}`。没有改动也推。
3. **登记**：交付 ref 的事件到达（或 Node 的报告到达，见 D4）时，Cloud 读出交付 ref 指向的提交，核对这条交付仍在
   等待、所属的执行没有结束，然后在一个事务里登记成果 `{deliveryId, 最终提交, 基准提交, 有无改动}`，任务运行的任务
   同时进入待验收。登记是幂等的。
4. **失败**：Node 报告交付失败，或等待超过时限（默认 10 分钟，部署可配）时，这条交付记为失败，写明原因；任务不进
   待验收，回到可以再次运行的状态。会话目录还在，下一轮结束时会再交付一次，届时的快照包含之前的改动。
5. **中断**：执行被中断（A1 D3、A2 D3）后，它等待中的交付全部作废。之后才到达的交付 ref 留在仓库里，但不登记，也不
   改变任何业务记录。

其他规则：

- Node 不再替 Agent 快进工作分支。Agent 没有提交时，相邻两轮的快照是同一个父提交下的兄弟，快进不了；工作分支只在
  Agent 自己推送时前进。
- 交付不上传会话记录：会话历史已经在 PostgreSQL 里，v1 的对象存储、`history_key` 和上传授权都不要。
- 同一个工作区的交付一次只进行一个；上一条还在等待时，新一轮不开始。

为什么等交付 ref 到达才进待验收：多机之后 Cloud 看不到 Node 的目录，待验收的任务必须在 Cloud 上有成果可看。回合结束
就进待验收，会出现“待验收却看不到改动”的窗口，交付失败还要另外撤回状态。

为什么每轮一个新的交付 ref：只能新建的 ref 不存在覆盖和非快进的问题，重试得到不同提交时也不会改写上一次的成果。代价
是 ref 的数量随轮次增长，清理留给后续决策。

### D8：外部仓库的拉取在 Cloud 进程内执行，由 webhook 和手动同步触发

- **绑定**：一个项目绑定一个外部仓库的一个分支。地址只接受 `https://`，不带用户信息、查询串和片段；分支是具体的
  分支名。两者建项目后不能改。这个分支就是项目的默认分支。
- **触发**：建项目时、外部平台的 push webhook、用户点“立即同步”、推回外部成功之后。不做周期轮询，建会话目录时也不
  隐式同步。
- **一次只跑一个**：每个项目同时最多一次同步。同步进行中又来了触发，只记下“还要再同步一次”，这次结束后再跑一次；
  多次触发合并成一次。
- **步骤**：Cloud 用加固的配置（只允许 `https`、不跟随重定向、hooks 关闭、不取 tag、不递归 submodule、LFS 只同步指针、
  完整历史）把外部分支直接取进项目仓库的临时 ref，然后在一次 ref 事务里写入快照 ref 并移动跟踪 ref，再在 PostgreSQL
  里把项目的当前快照指向新提交。外部分支被强推时照常跟随，旧快照保留，结果里注明“外部分支被改写”。外部分支被删除时
  同步失败，当前快照不变。
- **失败与中断**：失败不自动重试，结果和原因（凭据被拒、地址不可达、分支不存在、超过大小或时间上限）显示在项目页。
  Cloud 重启时进行中的同步标为中断，不自动重跑；仓库里可能留下已取入但未被引用的对象，由 `git gc` 回收。
- **webhook**：每个项目一个入口地址和一个密钥，密钥由 Cloud 生成，在项目设置里显示给管理员，按 D10 的方式加密保存。
  Cloud 校验密钥（GitLab 的 `X-Gitlab-Token`，GitHub 的 `X-Hub-Signature-256`），校验不过返回 `401`。请求体只当作
  “该同步了”的信号，里面的仓库、分支和提交都不采信；同步取到什么，以 Cloud 自己 fetch 的结果为准。入口按来源限速。
- **出网**：外部地址解析出的 IP 必须落在部署配置的外部 Git 白名单内，或者是公网地址；回环地址、Cloud 自己和数据库
  所在的地址一律拒绝。每条 Git 命令固定使用已检查过的 IP，防止解析结果中途变化。

为什么可以在 Cloud 进程里执行：v1 把这件事放在无状态的 repo-executor 里，是为了不让“持有外部凭据和外网出口”的组件
同时连着数据库。A1 D4 已经接受 Cloud 进程持有凭据；剩下的风险是用户填的地址把 Cloud 引向内网服务，由上面的出网规则
和加固配置限制。

### D9：推回外部由用户显式发起，只快进，结果不明时标中断

- **发起**：只有团队管理员或项目创建者能发起。输入是 `{来源, 目标分支, 期望的旧值}`：来源是本项目一个已登记的交付，
  或一个工作分支当前指向的提交；目标是所绑定外部仓库的一个分支；期望的旧值是发起人看到的外部分支当前值，新建分支时
  为空。Agent 和 Node 的推送永远到不了外部仓库。
- **只快进**：期望的旧值必须是来源提交的祖先，否则直接拒绝，不访问外部仓库。不支持强推、删除外部分支和推 tag。
- **步骤**：读外部分支的当前值，不等于期望的旧值就以“外部已变化”结束，不推送；相等则带着期望的旧值推送
  （`--force-with-lease`），外部平台的拒绝原因脱敏后展示。成功且目标是绑定的分支时，触发一次同步（D8）；推回不直接
  改当前快照。
- **一次只跑一个**：同一个目标分支同时最多一次推回。
- **结果不明**：Cloud 在推送途中重启，或连接在得到结果前断开时，这次推回标为中断，Cloud 不在后台核验，也不重推。
  用户再次发起同一来源和目标时，Cloud 先读外部分支的当前值：已经等于来源提交就直接记为成功；否则按上面的规则重新
  判断。

为什么不保留 v1 的 `uncertain` 自动核验（owner 2026-11-11 定）：它需要一套后台核验、退避和状态机。A1 D3 的反例已经说明
做法：外部副作用由发起它的领域用前置条件防重复，这里的前置条件就是期望的旧值。代价是用户要多点一次。

### D10：外部仓库令牌每个项目一个，加密后存在 PostgreSQL

- **归属**：第一版每个项目至多一个外部仓库令牌，属于团队，由团队管理员填写、更换和删除。不分个人凭据，没有冻结和
  逐次审批。同步和推回都用它。公开仓库可以不填，匿名拉取。
- **保存**：令牌用部署配置的密钥以 AES-256-GCM 加密后存在 PostgreSQL，附加数据绑定租户、项目和外部仓库地址，密文
  不能挪到别的项目上用。部署配置是一组带标识的密钥，密文记下所用密钥的标识，新写入用当前密钥，旧密钥只用来解密。
  密钥缺失时，需要令牌的同步和推回失败并写明原因，Cloud 照常启动。
- **使用**：明文只在 Cloud 进程里解密，在执行一条外部 Git 命令时交给这个子进程，不出现在命令行参数、地址、Git 配置、
  日志和 API 应答里。API 只返回“已配置”和最后更换的时间。
- **Node 拿不到**：外部仓库令牌不进入 Node 容器，也不出现在下发给 Node 的任何消息里（A1 D4 第 3 条）。
- **被外部平台拒绝**：同步或推回因认证失败结束时，项目页提示更换令牌；Cloud 不自动停用它。

为什么不继承 v1 的团队/个人归属和冻结（owner 2026-11-11 定）：那套规则要带上归属、冻结、审批三组状态。企业部署里
外部仓库通常用团队的服务账号。代价：推回是以团队令牌的身份写外部仓库，分不出是谁，所以发起人限制为管理员或项目
创建者，并记录发起人。需要个人凭据时另写决策。

## 被取代的 v1 决策

继承 A1“被取代的 v1 决策”中 Git 线和“Node 拿代码与交付”两组的安排，取代者为本文；标记之前它们的状态不变。

| v1 ADR | 被本文哪一条取代 | 标记时机 |
| --- | --- | --- |
| [`cloud/git-server/`](../git-server/0-project-code-hub-git-server-over-https.md) 全部 | D1–D4；ref 的名字和规则由 D2 继承 | I11（#1061）删除 v1 Git 线时 |
| [`cloud/repo-executor/`](../repo-executor/0-repo-executor-sole-external-git-boundary.md) | D8、D9 | 同上 |
| [`cloud/project/20261016-repository-binding-and-snapshot-sync.md`](../project/20261016-repository-binding-and-snapshot-sync.md) | D8 | 同上 |
| [`cloud/project/20261017-explicit-fast-forward-publish.md`](../project/20261017-explicit-fast-forward-publish.md) | D9 | 同上 |
| [`cloud/project/20260927-repository-credential-continuity.md`](../project/20260927-repository-credential-continuity.md) | D10。A1 把它归在“v1 业务领域”一组，这里写明取代者 | 同上 |
| [`cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md`](../identity-access/20261016-git-credentials-in-independent-credential-store.md) | D10 | 同上 |
| [`cloud/revision/`](../revision/0-cloud-owned-object-store-and-verified-revisions.md) 全部 | D7 | 同上 |
| [`node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md`](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md) | D6：整份取代 | I6（#1056）的 Node 后续 ADR 合入时 |
| [`node/revision/`](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) 中交付推送和续作恢复的各份 | D7：交付时机、Workspace 分支快进、会话记录上传、从前序交付 ref 恢复都不再适用；内部快照的做法继续有效，按“历史决策”标注 | 同上 |

`node/revision` 里恢复截止和停止确认两份讲的是 Node 本地的进程收尾，本文不改。

## 不变量

1. 每个项目有且只有一个裸仓库；项目之间不共享对象。
2. 经 Git 入口访问仓库必须带有效的项目令牌，并且只能访问令牌所属项目的仓库。
3. 项目令牌只能新建或快进 `refs/heads/workspaces/**`，只能新建 `refs/ora/deliveries/*`，其余 ref 都写不了。
4. 任何 ref 都不能经推送删除；交付 ref 和快照 ref 一经写入不再改变。
5. `refs/ora/upstreams/**` 只由 Cloud 的同步写入。
6. 一次推送里任何一条更新不合规则，整次推送什么都不写。
7. 靠事件推进的业务状态，都另有一次直接读仓库的核对；事件丢失不会让已到达的交付永远不被登记。
8. 任务进入待验收时，它的成果已经登记，并且指向一个存在的交付 ref。
9. 已结束或已中断的执行，之后到达的推送不改变业务记录。
10. 会话目录建好时 HEAD 等于 Cloud 指定的基准提交。
11. 外部仓库令牌的明文只出现在 Cloud 进程和它启动的外部 Git 子进程里，不进 Node，不进日志和 API 应答。
12. 推回外部只做快进，Cloud 从不自动重推。
13. webhook 的内容只触发同步，不决定同步到哪个仓库、分支或提交。
14. 每个项目同时最多一次外部同步；每个外部目标分支同时最多一次推回。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| Git server 做成独立进程，靠 hook 回调 Cloud | 大推送不占 Cloud 进程的资源 | 多一个进程、一条需要认证的回调链路和一套授权查询；#1043 Q3 已否定 |
| 用 pre-receive hook 判定 ref 规则 | 能在写入前看到对象，可以判更复杂的规则 | 本文的规则在命令列表里就能判定；hook 需要子进程回调通道（D3） |
| 保留 `ora-git`，checkout 不配 remote | Node 改动最小，令牌不必交给 Agent | Agent 要学专用命令；令牌按 A2 D7 本来就对 Agent 可见（D6） |
| 回合中途经 Node 读 checkout 的变更 | 变更视图实时，包含未提交的改动 | 变更视图有两个数据来源，Node 协议多一组读命令（D5） |
| 工具调用完成后把快照推到每会话可覆盖的 ref | 数据来源只有 Git server，接近实时 | 多一个允许非快进的命名空间，推送频繁；需要时在本文之上追加 |
| 回合结束即进待验收，交付异步 | 状态变化更快 | 会出现待验收却看不到成果的窗口（D7） |
| 只有任务交付，对话不交付 | 交付 ref 少 | 多机下对话里未提交的改动完全看不到 |
| 推回结果不明时后台自动核验 | 用户不用再点一次 | 需要后台核验和状态机；期望的旧值已经能防重复（D9） |
| 继承团队/个人凭据、冻结和审批 | 推回能对应到个人身份 | 三组状态，第一版用不上（D10） |
| 外部仓库令牌放在独立的凭据服务里 | Cloud 被攻破时令牌不泄露 | 多一个服务和一条兑换链路；A1 D1、D4 已定收进 Cloud 进程 |
| 周期轮询外部仓库 | 不依赖外部平台配置 webhook | 延迟和无效请求都更多；#1043 Q3 已否定，手动同步兜底 |

## 本决策未解决的问题

1. **回合进行中的实时变更视图**：见替代方案表第 5 行，需要时另写决策。
2. **交付 ref 和旧快照的清理、仓库配额**：第一版全部保留，只限制单次推送的大小。
3. **个人凭据、SSH 和 GitHub App**：第一版只有 HTTPS 加团队令牌。
4. **一个项目绑定多个外部仓库或多个分支，以及修改绑定**：第一版不支持。
5. **推回时自动建合并请求**：#1043 Q3 定为以后再做。
6. **人直接 clone 项目仓库**：需要用户身份的 Git 凭据，与 A6 的登录一起考虑。
7. **续作**：项目换机器后会话目录丢失（A2 D4），新会话从当前快照开始，不自动恢复到之前的交付；需要时另写决策。

## 落地顺序

1. **I6（#1056）Git server 与事件**：D1–D4，默认关闭。完成条件：项目令牌能 clone、fetch，能按 D2 的规则推送，
   违规的推送整次被拒；推送后事件到达订阅者。演示路径不变。
2. **I6 Node 一侧**：D6、D7 的 Node 修改，以及 `decisions/node/` 下的后续 ADR。完成条件：第二次建会话目录不再完整
   clone；Agent 在会话目录里 `git push` 成功；一轮结束后交付 ref 出现。
3. **I6 交付登记与变更视图**：D5、D7 的 Cloud 一侧。完成条件：交付 ref 到达后任务进入待验收，变更视图不读 Node 的
   目录；交付失败和执行中断各有测试。
4. **I7（#1057）外部同步、推回和凭据**：D8–D10。完成条件：webhook 和手动同步都能推进当前快照；推回的快进、外部已
   变化、中断后重新发起各有测试；令牌不出现在日志和 API 应答里。
5. **I10（#1060）**：演示切到本文的路径，删除 `projectrepos` 的 mirror 路径、`nodeservices` 的 Git 和 `workspacefs` 读
   本机目录的做法。从这一步开始，演示环境的行为改变。
6. **I11（#1061）**：删除 v1 Git 线，按“被取代的 v1 决策”一节标记 ADR。
