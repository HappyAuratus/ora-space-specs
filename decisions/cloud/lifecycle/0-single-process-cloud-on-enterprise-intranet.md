---
status: approved
---

# Cloud 是一个进程：由 v2-server 演进而来，第一版单机部署、Node 是本地子进程，崩溃按中断处理，取代 v1 的服务拆分与"恰好一次续跑"

> 历史决策：D3 里“Agent 的改动只存在于 Node 的工作目录里，第一版不交付”由[回合交付](../project-git/20261112-single-machine-turn-delivery.md)修改：每轮结束由 Cloud 交付到项目镜像，仍不推送到外部仓库。其余决策不变。
>
> 历史决策：D3 隐含的“Cloud 从不删除工作目录”与 D7 第 3 条“只读”由[删除对话时回收工作目录](../project-git/20261119-reclaim-deleted-conversation-workspace.md)（proposed）修改：用户删除普通对话时，Cloud 经 `workspacefs` 按该决策回收它的工作目录。

正式 Cloud 不再是 v1 那套由 server、gateway、model-gateway、credential-store、git-server、repo-executor、
ora-controller、sandbox-server 组成的分布式系统，而是由 `ora-space/cloud` 的 `v2-server` 演进出的**一个 Cloud 进程**，
加上一个 PostgreSQL 和 Cloud 自己的数据卷。第一版部署在一台机器上：每个项目一个 Node，作为 Cloud 的本地子进程运行，
经本机 Unix socket 通信；这就是现在演示用的路径，没有第二条路径。第一版不做多机，不做容器隔离，不提供 Git server，
Agent 的改动不推送到任何地方。Cloud 重启或 Node 退出时，没跑完的执行一律标为中断，不续跑，由业务层重做或对账；
基础设施层不承诺"每一步恰好执行一次、在任何一点崩溃都能续上"。多机和隔离以后再做，本文只规定第一版不能破坏的
几条接缝（D7）。

本 ADR 处于 `approved`：owner 已评审通过；它是 `cloud/lifecycle` 的根决策，也是 phase3 基座（circus #1043）的根决策。
2026-11-11 owner 在 circus #1073 把第一版的范围收窄为单机、不隔离，本文据此重写：原先关于多机、执行宿主、容器、
令牌体系和"新旧路径并行"的内容已删除，同时删除了两份尚未被使用的后续根决策（多机执行层、Node 容器），
多机和隔离到时重新写决策。依据是 circus #1043 的结论、#1023 的实测数据、#1017 的阶段划分，读取的代码版本为
cloud `80e9c9f1`（`v2-server/`）。2026-11-12 随 A6 修订了登录（D1、D6）、租户授权（D4）和离线部署（D2、D6）的说法。核心测试用例见
[Cloud 单进程用例](../../../test-cases/cloud/lifecycle/single-process-cloud.md)。

范围与兼容性边界：

- **涉及**：`ora-space/cloud` 的 `v2-server/`（成为正式 Cloud）。desktop 的 Node 不改。
- **不在本文**：外部仓库的拉取和令牌见[项目仓库根决策](../project-git/0-external-repository-pull-and-encrypted-token.md)
  （下称“A4”）；模型接入见[模型接入根决策](../model-proxy/0-in-process-model-proxy-with-deployment-connections.md)
  （下称“A5”）；登录见[登录根决策](../login/0-idaas-or-email-login-with-shared-web-and-desktop-sessions.md)（下称“A6”）：部署版只走华为 IDaaS，开发版用无密码邮箱。desktop 连接 cloud 的正式契约由 phase2 的 #1029 承担。
- **不承担数据迁移**：v1 Cloud 的数据库、对象存储和仓库不迁到新 Cloud。
- 本文不立刻把任何 v1 ADR 标为 `superseded`。v1 组件退役时，由承接的 task 按“[被取代的 v1 决策](#被取代的-v1-决策)”
  一节逐份标记。

## 现状与依据

#1023 在同一场景（OpenCode、DeepSeek Flash、同一任务）下实测了 v1 和 v2-server。结论是 v1 的复杂度来自同一种思路：
**在基础设施层保证每一步恰好执行一次、崩溃后能续上**，而业务层本来就能靠重做和对账收尾。

| 观察 | v1（正式 Cloud） | v2-server | 说明 |
| --- | --- | --- | --- |
| 进程 | 9 种服务，每个 Workspace 一个 Node 容器 | 1 个后端，每个项目一个 Node | 规模：cloud Go 约 5.8 万行加 1.6 万行生成代码，对 v2-server 约 3.6 万行 |
| Node 侧专为 Cloud 写的 | ora-controller 1.2 万行、controller-proto 0.7 万行、host/guardian 1.1 万行 | Unix socket 直连 | host/guardian 已由 #1033 定为退役（O5） |
| 首条输出之后 | Revision 交付固定 8.0 s；删除被拒后等下次领取 1.5–8.6 s；clone 结果等下次领取 0.1–5.0 s | 没有这些步骤 | 慢在"意图登记、等证据、停放、等领取"的组合，不在单次往返 |
| 只读并发 | 吞吐锁死在约 170–200 req/s，CPU 只用约 36% | 618 / 1003 / 1194 req/s | 所有事务（含只读）都取同一个 `pg_advisory_xact_lock(67420911)` |
| 规格 | 192 份 ADR | 不走 specs | |

在 v1 上逐项简化，全局锁、意图加证据接管、周期推进、Controller 领取和租约、一次性 Workspace、model-gateway 几乎都要改，
总量接近重写（#1043 否定的方案 A）。v2-server 已经具备单进程形态、重启标中断的语义和进程内模型代理。

项目还没有过第一次部署。第一版只补部署必需的几项（D6），不为还没有出现的需求预先设计。

## 决策

### D1：Cloud 是一个进程，由 v2-server 演进而来

正式 Cloud 是 `v2-server` 演进出的单个进程，承载 HTTP API、SSE、执行编排、模型代理、外部仓库拉取、插件市场同步和
定时调度。持久状态只有两处：PostgreSQL，以及 Cloud 数据卷（项目仓库的镜像、Node 的数据目录）。

v1 各服务的去向：

| v1 服务 | 去向 | 规则由谁定 |
| --- | --- | --- |
| server（cloud core） | 退役；业务能力按需补进 v2-server，API 以 v2-server 为基准（#1017） | #1029（契约）、#1062（积压改写） |
| gateway | 收进 Cloud 进程：部署版华为 IDaaS，开发版无密码邮箱 | A6 |
| model-gateway | 收进 Cloud 进程：进程内模型代理 | A5 |
| credential-store、git-credential-ora | 收进 Cloud 进程：外部仓库令牌加密存 PostgreSQL | A4 |
| repo-executor | 收进 Cloud 进程：外部仓库由 Cloud 拉取；第一版不推回 | A4 |
| git-server | 退役：第一版没有 Git server，Node 从 Cloud 本机的镜像只读 clone | A4 |
| ora-controller、controller-proto | 退役：Cloud 自己担任 Node 协议的 Controller 一侧 | 本文 D2 |
| sandbox-server、cluster 沙盒路由 | 退役：第一版 Node 是本地子进程 | 本文 D2 |
| ora-web（v1 前端） | 退役：frontend-v2 是正式前端 | #1061 |

为什么是一个进程：v1 的服务拆分换来的是可以独立扩展和隔离故障，代价是每条服务间链路都要 mTLS、每个服务都要自己的
持久化和恢复协议。#1023 的数据显示，耗时和吞吐的瓶颈都来自这些协调机制。单进程里，组件之间是函数调用。

代价：Cloud 进程是单点，崩溃时所有在跑的执行都会中断（D3）；一个组件的严重缺陷会拖垮整个 Cloud。

### D2：第一版部署在一台机器上，Node 是 Cloud 的本地子进程

- **部署形态**：一台机器上跑 Cloud 进程和 PostgreSQL。每个项目一个 Node，由 Cloud 作为子进程拉起，和 Cloud 同一个
  系统用户；Cloud 经本机 Unix socket 和它跑 Node 协议。Node 需要的 Git、模型和插件服务由 Cloud 在本机回环地址上提供。
- **第一版就是演示的路径**：`task v2:demo` 起来的就是这个形态，没有另一条"正式路径"在旁边并行开发。
- **不做的**：多机、执行宿主、容器、出网规则、宿主令牌和项目令牌。
- **长期运行**：部署要能连续运行，不因 Cloud 内部签发的凭据有固定有效期而失效。现在本机服务的自签证书 24 小时过期
  且不轮换，由 circus #1074 修。
- **运维**：备份 PostgreSQL 和 Cloud 数据卷。第一次内网部署之前由 circus #1059 准备：部署版启动不清空数据，
  v2-server 托管前端构建产物，在外网准备离线资源，插件市场走内网 Git；HTTPS 由谁终止也在 #1059 定。

为什么先单机：多机需要宿主程序、放置、失联处理和一套令牌；隔离需要容器镜像和网络规则。它们解决的问题在第一次部署时
还不存在。先部署，再按实际遇到的规模和信任要求设计。

**不隔离的后果要说清楚**：Agent 执行的代码和 Cloud 是同一个系统用户，技术上能读到 Cloud 的数据目录、配置和其他项目的
工作目录，也能连上数据库。所以第一版只适合成员互相信任的部署，不能给互不信任的团队共用一套。要给互不信任的团队用，
先做隔离。

### D3：崩溃和 Node 退出都按中断处理，不续跑

- 下文的“中断”指[中断](../../../domain/glossary.md#interrupted-execution)这一终态。
- **Cloud 重启**：启动时先把所有未结束的执行、会话轮次、任务运行和工作流节点标为中断，原因写明是重启，再开始服务请求。
  v2-server 现在已经这样做（`execution.Restart`、`store` 里的 `server_restarted`）。
- **Node 进程退出或连接断开**：这个 Node 上正在跑的执行结束，不在别处续跑；确认旧 Node 已停止之前不重建。
- **中断之后**：由业务层决定重做还是对账，例如对话由用户重新发送，任务运行回到可再次运行的状态。基础设施层不自动
  重放、不续跑、不保证同一步只执行一次。
- **数据在哪**：业务数据在 PostgreSQL；项目仓库的镜像和 Node 的数据目录在 Cloud 数据卷。**Agent 的改动只存在于
  Node 的工作目录里**，第一版不交付、不推送到任何地方；工作目录被删除，改动就没有了。

为什么接受"中断"：v1 为了续跑，在每一步登记意图、等待执行端证据、按稳定身份对账，配合租约、代次围栏、停放和周期领取。
#1023 测到这套机制在正常路径上每次运行多出 5–17 s 的空等，而它保护的崩溃场景很少发生。业务层本来就要能处理"用户取消"
"模型失败"这类结局，"中断"只是多一种结局。

边界：中断只针对**执行**。已经提交到 PostgreSQL 的业务写入按普通数据库事务保证，不受影响。

### D4：Cloud 不把模型 Key 和外部仓库令牌交给 Node，API 按租户授权

1. **模型 Key 不下发给 Node。**Key 只由 Cloud 进程内的代理加到上游请求上，Node 和 Agent 只拿到按会话签发、可吊销的
   临时令牌（A5）。
2. **外部仓库令牌不下发给 Node。**外部仓库只由 Cloud 访问（A4）。
3. **租户隔离**：每个租户 API 经同一个入口判定成员身份：一个判定函数、一个路由中间件，非成员一律 `404`，
   只有 owner 能做的操作在路由表上声明，后台任务开始执行时用同一个判定；业务 SQL 只按 `tenant_id` 过滤，
   不再各自核对成员。由 I3 #1053 实现，路由表测试兜底。

这两条"不下发"在第一版只说明 Cloud 不主动交出去：因为不隔离（D2），同机的 Agent 在技术上仍可能读到 Cloud 的环境和
文件。它们的意义是保住接缝，隔离做好之后不用再改凭据的走法（D7）。

### D5：持久化：不用全局锁，迁移版本化

- **不用全局串行锁。**普通请求用普通的行写入和数据库约束；需要互斥的地方按业务对象（例如同一会话、同一次工作流运行、
  同一个空间插件）取作用域锁，或在进程内互斥。只读请求不取锁。v2-server 现在就是这样做的。
- **跨表不变量**不能靠"所有事务串行"兜底，每一条要写清由哪个行锁、约束或事务顺序保证。
- **迁移版本化**：v2-server 从"启动时建表"改为有编号的版本化迁移，升级要有测试覆盖（I1 #1051）。第一次部署之后库里
  就有要保留的数据，所以要在第一次部署之前完成。迁移编号规则沿用 circus 记忆 `migration-numbering`。

### D6：规格范围：只给第一版必需的保证写 ADR

- phase3 基座的事项改 `v2-server/` 也要走 specs（协议由 #1063 修改）。phase1 只改 v2 目录的演示事项仍按现行豁免。
- **只给下表里的保证写 ADR**，不回头给 v2-server 已有的行为逐条补文档，也不为第一版不做的能力预先写决策。
  v2-server 其余已有行为，以代码、候选契约（`*.openapi.json`）和 #1029 定下的正式契约为准。
- v1 的 ADR 不在本文里改状态，对应组件退役时再标为 `superseded`。

第一次部署必需的保证：

| # | 保证 | v2-server 现状 | 决策 | 实现 |
| --- | --- | --- | --- | --- |
| 1 | 登录：部署版华为 IDaaS，开发版无密码邮箱 | 只有邮箱登录，仅用于演示 | A6 | I2 #1052 |
| 2 | 租户之间的数据隔离：统一入口判定成员身份 | 各 handler 在 SQL 里各自核对，条件不一致 | 本文 D4 | I3 #1053 |
| 8 | 第一次内网部署：部署版启动、托管前端、离线资源 | 演示形态，每次启动清空 | 本文 D2 | #1059 |
| 3 | 外部仓库：拉取、手动同步、私有仓库 | 建项目时 clone 一次，只支持公开仓库 | A4 | I7 #1057 |
| 4 | 模型接入：Key 不下发给 Node | 已有 | A5 | 已有 |
| 5 | 数据库迁移和升级 | 启动时建表 | 本文 D5 | I1 #1051 |
| 6 | 崩溃恢复 | 重启标中断 | 本文 D3 | 已有 |
| 7 | 连续运行超过 24 小时 | 本机服务证书 24 小时过期 | 本文 D2 | #1074 |

第一版不做、以后需要时再写决策的：多机、Agent 执行隔离、Git server 与交付、推回外部仓库、个人或团队的模型连接、
用量计量与配额。

### D7：为多机和隔离保留的接缝

第一版不做多机和隔离，但下面几条现在已经成立，之后的修改不能破坏它们；破坏任何一条都要先改本文。

1. **Cloud 和 Node 只经 Node 协议通信。**Cloud 不直接操作 Node 的进程内部状态。以后在两者之间插入转发，协议不用改。
2. **Node 拿代码、模型和插件都是访问 Cloud 给它的 URL。**Node 不读 Cloud 的文件路径。以后把地址从本机换成 Cloud 的
   入口即可。
3. **Cloud 读 Node 的工作目录只经一个模块**（现在是 `workspacefs`）。其他代码不直接拼工作目录的路径。以后这个模块的
   数据来源可以换掉。
4. **模型 Key 和外部仓库令牌不下发给 Node**（D4）。
5. **每个 API 按租户作用域授权**（D4）。

为什么只写接缝不写方案：多机和隔离的具体做法取决于第一次部署之后才知道的规模和信任要求。接缝保证的是到时候改的是
局部，而不是推倒重来。

多机之前必须先解决的一件事：Agent 的改动只在 Node 的工作目录里（D3）。Node 一旦不和 Cloud 在同一台机器上，Cloud 就
读不到它，届时要先设计交付。

## 被取代的 v1 决策

下表列出将被取代的 v1 ADR。"标记时机"之前，它们继续描述 v1 的行为，状态不变；到时由承接的 task 标为
`superseded`，并按 specs 的规则同步核心测试用例。

v1 业务领域的 ADR 不逐份挑出仍成立的规则继承：#1043 已定不回头给 v2-server 的行为补文档。它们在 v1 退役时整体标为
`superseded`，理由写"由 v2-server 和正式契约取代"。

| 组 | v1 ADR | 取代者或放弃理由 | 标记时机 |
| --- | --- | --- | --- |
| 服务拆分与 Controller | [`controller/`](../../controller/README.md) 下全部；[`cloud/controller-integration/`](../controller-integration/0-cloud-owned-internal-grpc-contract.md) 全部；[`cloud/operation/`](../operation/0-workspace-runtime-follows-desktop-node.md) 全部 | 本文 D1、D3：Cloud 自己跑 Node 协议，崩溃即中断，不做领取、租约和意图登记 | I11 #1061 删除 ora-controller 时 |
| 沙盒 | [`cluster/sandbox-server/`](../../cluster/sandbox-server/0-docker-label-resolver.md) 全部 | 本文 D2：第一版 Node 是本地子进程；隔离以后重新写决策 | I11 #1061 删除 sandbox-server 时 |
| Git 线 | [`cloud/git-server/`](../git-server/0-project-code-hub-git-server-over-https.md) 全部；[`cloud/repo-executor/`](../repo-executor/0-repo-executor-sole-external-git-boundary.md)；[`cloud/project/20261016-repository-binding-and-snapshot-sync.md`](../project/20261016-repository-binding-and-snapshot-sync.md)；[`cloud/project/20261017-explicit-fast-forward-publish.md`](../project/20261017-explicit-fast-forward-publish.md)；[`cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md`](../identity-access/20261016-git-credentials-in-independent-credential-store.md)；[`cloud/revision/`](../revision/0-cloud-owned-object-store-and-verified-revisions.md) 全部 | A4：外部仓库由 Cloud 进程拉取，令牌加密入库；Git server、交付和推回第一版不做 | I11 #1061 删除 v1 Git 线时 |
| 模型接入 | [`cloud/model-access/`](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) 全部 | A5：Cloud 进程内代理，部署级模型连接 | I11 #1061 删除 model-gateway 时 |
| 登录 | [`cloud/identity-access/0-gateway-mediated-external-identity.md`](../identity-access/0-gateway-mediated-external-identity.md)、[`20260923-huawei-global-id-alias.md`](../identity-access/20260923-huawei-global-id-alias.md)、[`20261019-deployment-bound-desktop-sessions.md`](../identity-access/20261019-deployment-bound-desktop-sessions.md) | A6：登录收进 Cloud 进程，部署版华为 IDaaS、开发版无密码邮箱 | I11 #1061 删除 v1 gateway 时 |
| 全局串行锁 | 写在 [`cloud/workspace/0-collab-space-boundary.md`](../workspace/0-collab-space-boundary.md)、[`cloud/api-boundary/0-space-api-and-event-stream.md`](../api-boundary/0-space-api-and-event-stream.md)、[`cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md`](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md) 里，没有单独的 ADR | 本文 D5：不用全局锁 | 随所在 ADR 一起处理（见下一行） |
| v1 业务领域 | `cloud/` 下其余目录：api-boundary、execution-admission、issue-collaboration、issue-run、persistence、plugin-marketplace、project（除上面列出的）、scheduled-task、squad、tenancy、thread、workflow、workspace | 放弃，不逐份继承：v2-server 各自另有实现，API 以 v2-server 和 #1029 的正式契约为准（D6） | I11 #1061 删除 v1 时整体标记，由 #1062 按领域执行 |
| Node 进程宿主 | [`node/process/`](../../node/process/README.md) 下 architecture、control、io、lifecycle、persistence、protocol、recovery、upgrade、integration | #1033 的 O5：退役 host/guardian | O5 的 ADR（circus #1069）approved 后 |

**不被取代、继续有效的**：

- `node/` 下的决策（除上表的 `node/process`）：Node 不改。其中写着"Controller 侧"的地方由 Cloud 承担；v2-server 现在
  用到的只是 clone、插件安装和 Agent 会话，交付和续作这些能力 Node 仍然具备，只是 Cloud 第一版不调用。
- `desktop/` 下的全部决策：desktop 连接 cloud 的变化由 phase2（#249、#1029）处理，不在本文。

## 不变量

1. 正式 Cloud 的后端是一个进程；持久状态只在 PostgreSQL 和 Cloud 数据卷。
2. Cloud 重启后，在开始服务请求之前，所有重启前未结束的执行都已标为中断。
3. Node 退出或断开后，它上面未结束的执行已经结束，不在别处续跑。
4. 基础设施层不自动重放被中断的执行。
5. Cloud 不把模型 Key 和外部仓库令牌下发给 Node。
6. 只读请求不取锁；任何请求都不取全局串行锁。
7. 部署连续运行不因 Cloud 内部签发的凭据到期而失效。
8. Cloud 和 Node 只经 Node 协议通信；Cloud 读 Node 的工作目录只经一个模块。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| A：在 v1 上做减法 | 保留 192 份 ADR 的积累，v1 的正确性证据可以继承 | 全局锁、意图加证据、周期推进、Controller 租约、一次性 Workspace、model-gateway 几乎都要改，总量接近重写（#1043）；与 D3 的崩溃语义冲突 |
| 保留恰好一次续跑，只合并进程 | 崩溃后不丢执行进度 | 续跑需要的意图登记、证据接管、代次围栏在单进程里照样要做，换来的场景很少发生（D3） |
| 第一版就做多机和容器隔离 | 一步到位，不用二次改造 | 要先写宿主、令牌、容器镜像、网络规则、Git server 和交付，才能第一次部署；这些设计没有实际部署的反馈可依据（D2）。本文 2026-11-11 之前的版本就是这样，已被 owner 收窄 |
| 新路径在演示路径旁并行开发，演示后切换 | 演示路径不受新开发影响 | 每个功能要维护两套；第一版范围收窄后已经没有第二条路径（D2） |
| Cloud 多副本、水平扩展 | 消除 Cloud 单点 | 第一版的规模不需要；多副本要重新引入进程间协调，与 D1 的简化方向相反 |

## 本决策未解决的问题

1. **多机部署。**需要时另写决策，至少回答 Cloud 怎么在别的机器上起停 Node、怎么通信和认证、Node 怎么拿到代码。
2. **Agent 执行隔离。**需要时另写决策；在那之前部署只适合互相信任的成员（D2）。
3. **Agent 改动的交付与推回外部仓库。**第一版改动只在工作目录里；推回由 A4 的后续决策设计。
4. **Cloud 本身的高可用。**本文接受 Cloud 是单点（D1）。

## 落地顺序

1. **第一次部署之前**：I1 #1051（版本化迁移）、I3 #1053（统一租户授权）、#1074（连续运行超过 24 小时，已修复）、
   I7 #1057（外部仓库同步与私有仓库，按 A4）、I2 #1052（登录，按 A6）、#1059（第一次内网部署）。每一项都直接改现有路径，
   完成条件是 `task v2:demo` 照常一条命令起来、演示场景可用。
2. **v1 退役**（I11 #1061）：删除 v1 组件，按“被取代的 v1 决策”一节标记 ADR。
3. **已作废**：执行宿主（I4 #1054）、Node 镜像（I5 #1055）、Git server（I6 #1056）、模型代理收口（I8 #1058）、
   演示切换（I10 #1060）和 Node 改项目令牌（#1066）已在 2026-11-12 关闭。多机或隔离以后按 D7 的接缝重新立项。
