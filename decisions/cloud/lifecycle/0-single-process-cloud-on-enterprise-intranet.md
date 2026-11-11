---
status: proposed
---

# Cloud 收为单进程：由 v2-server 演进而来，部署在企业内网、Node 分布在多台机器上，崩溃和断开都按中断处理，取代 v1 的服务拆分与"恰好一次续跑"

正式 Cloud 不再是 v1 那套由 server、gateway、model-gateway、credential-store、git-server、repo-executor、
ora-controller、sandbox-server 组成的分布式系统，而是由 `ora-space/cloud` 的 `v2-server` 演进出的**一个 Cloud 进程**，
加上一个 PostgreSQL 和 Cloud 自己的数据卷。v1 各服务的职责要么收进这个进程，要么退役。部署目标是企业内网：
Cloud 只有一个进程，Node 跑在多台机器的容器里，每台机器一个[执行宿主](../../../domain/glossary.md#execution-host)（下称“宿主”；不是 desktop 的原生宿主，也不是 Node 的运行宿主 Process Host）主动连上 Cloud。跨机器的链路只用普通服务端 TLS
加令牌，不用 mTLS、SPIFFE 证书，也不要独立的 model-gateway。Cloud 重启、宿主断开、机器失联时，没跑完的执行一律标为
中断，不续跑，由业务层重做或对账；基础设施层不再承诺"每一步恰好执行一次、在任何一点崩溃都能续上"。
安全底线有两条：Agent 执行的代码碰不到 Cloud 的数据库、其他项目和凭据；模型 Key 不进沙盒。

本 ADR 处于 `proposed`，供 owner 评审；它是 `cloud/lifecycle` 的根决策，也是 phase3 基座（circus #1043）的根决策，
后续 A2–A6（多机执行层、容器隔离与出网、Git server、模型接入、企业登录）都以它为前提。依据是 circus #1043 的结论与
Q1–Q3 评论、#1023 的实测数据、#1017 的阶段划分，读取的代码版本为 cloud `ad324c77`（`v2-server/`）、cluster `0b8d913`、
desktop `f8e8deec`。

范围与兼容性边界：

- **涉及**：`ora-space/cloud` 的 `v2-server/`（成为正式 Cloud）和新写的执行宿主；desktop 的 Node 只换运行位置（容器）
  和代码来源（Git server），Node 协议不变。
- **不在本文**：宿主协议、容器与出网、Git server、模型代理、企业登录的具体规则分别由 A2–A6 规定；本文只定它们共同
  遵守的进程形态、崩溃语义、安全底线和规格范围。desktop 连接 cloud 的正式契约由 phase2 的 #1029 承担。
- **演示约束**：演示结束前，`task v2:demo` 必须照样一条命令就能起来。现在的"后端直接拉起本地 Node 子进程、从本地
  mirror clone"路径保留不动，新路径（内嵌宿主、容器、Git server）在旁边并行开发，演示结束后再切换默认并删除旧路径
  （D7）。
- **不承担数据迁移**：v1 Cloud 的数据库、对象存储和仓库不迁到新 Cloud。v1 在演示结束后整体退役（D1、D7）。
- 本文不立刻把任何 v1 ADR 标为 `superseded`。v1 组件退役时，由承接的 task 按"[被取代的 v1 决策](#被取代的-v1-决策)"
  一节逐份标记。

## 现状与依据

#1023 在同一场景（OpenCode、DeepSeek Flash、同一任务）下实测了 v1 和 v2-server。结论是 v1 的复杂度来自同一种思路：
**在基础设施层保证每一步恰好执行一次、崩溃后能续上**，而业务层本来就能靠重做和对账收尾。

| 观察 | v1（正式 Cloud） | v2-server | 说明 |
| --- | --- | --- | --- |
| 进程 | 9 种服务，每个 Workspace 一个 Node 容器 | 1 个后端，每个项目一个 Node | 规模：cloud Go 约 5.8 万行加 1.6 万行生成代码，对 v2-server 约 3.6 万行 |
| Node 侧专为 Cloud 写的 | ora-controller 1.2 万行、controller-proto 0.7 万行、host/guardian 1.1 万行 | Unix socket 直连 | host/guardian 已由 #1033 定为退役（O5） |
| 评论到首条输出 | 33.2 s（中位数） | 29.0 s | 两边共同的大头是 ora-node 的 clone（13–15 s），与架构无关，由 desktop 单独修 |
| 首条输出之后 | Revision 交付固定 8.0 s；删除被拒后等下次领取 1.5–8.6 s；clone 结果等下次领取 0.1–5.0 s | 没有这些步骤 | 慢在"意图登记、等证据、停放、等领取"的组合，不在单次往返 |
| 只读并发 | 吞吐锁死在约 170–200 req/s，并发 16 时 p50 82.8 ms，CPU 只用约 36% | 618 / 1003 / 1194 req/s | 所有事务（含只读）都取同一个 `pg_advisory_xact_lock(67420911)` |
| 规格 | 192 份 ADR | 不走 specs | |

在 v1 上逐项简化，全局锁、意图加证据接管、周期推进、Controller 领取和租约、一次性 Workspace、model-gateway 几乎都要改，
总量接近重写，还要背着 v1 的部署复杂度（#1043 否定的方案 A）。v2-server 已经具备单进程形态、重启标中断的语义和
进程内模型代理，缺的是企业部署必需的保证（见 D6 的保证清单）。

## 决策

### D1：Cloud 是一个进程，由 v2-server 演进而来

正式 Cloud 是 `v2-server` 演进出的单个进程，承载 HTTP API、SSE、执行编排、模型代理、Git server、外部仓库同步、
插件市场同步和定时调度。持久状态只有两处：PostgreSQL，以及 Cloud 数据卷上的 Git 仓库。

v1 各服务的去向：

| v1 服务 | 去向 | 规则由谁定 |
| --- | --- | --- |
| server（cloud core） | 退役；业务能力按需补进 v2-server，API 以 v2-server 为基准（#1017） | #1029（契约）、#1062（积压改写） |
| gateway | 收进 Cloud 进程：企业登录（IDaaS / OIDC）在 Cloud 内完成 | A6 #1050 |
| model-gateway | 收进 Cloud 进程：进程内模型代理 | A5 #1049 |
| credential-store、git-credential-ora | 收进 Cloud 进程：外部仓库凭据加密存 PostgreSQL，只在 Cloud 内使用 | A4 #1048 |
| git-server | 收进 Cloud 进程：内嵌 smart HTTP，推送完成在进程内发事件 | A4 #1048 |
| repo-executor | 收进 Cloud 进程：外部同步和推回外部由 Cloud 执行 | A4 #1048 |
| ora-controller、controller-proto | 退役：由每台机器的执行宿主转发 Node 协议帧 | A2 #1046 |
| sandbox-server、cluster 沙盒路由 | 退役：宿主直接调 Docker | A3 #1047 |
| ora-web（v1 前端） | 退役：frontend-v2 是正式前端 | #1061 |

为什么是一个进程：企业内网部署的规模是少量副本，v1 的服务拆分换来的是可以独立扩展和隔离故障，但代价是每条服务间链路
都要 mTLS、每个服务都要自己的持久化和恢复协议。#1023 的数据显示，耗时和吞吐的瓶颈都来自这些协调机制，而不是单个服务的
计算量。单进程里，组件之间是函数调用和进程内事件，不存在"消息丢了怎么补"的问题。

代价：Cloud 进程是单点，崩溃时所有在跑的执行都会中断（按 D3 处理）；一个组件的严重缺陷（例如 Git 处理把内存打满）会
拖垮整个 Cloud。需要水平扩展 Cloud 本身时要另做决策（见"本决策未解决的问题"）。

"一个进程"指的是 Cloud 的后端。执行宿主是另一个进程，跑在执行 Node 的机器上（D2）；单机部署时宿主以内嵌模式跑在
Cloud 进程里，这时整个部署只有 Cloud、PostgreSQL 和 Node 容器。

### D2：部署在企业内网，Node 分布在多台机器上，跨机器链路用服务端 TLS 加令牌

- **部署形态**：Cloud 和 PostgreSQL 在一处，执行 Node 的机器可以有多台。每台机器跑一个执行宿主，主动连上 Cloud，
  用一条长连接转发本机所有 Node 的协议帧；一个项目固定在一台机器上（A2 规定调度、容量和失联处理）。
- **网络和认证**：Cloud 只暴露 HTTPS 入口，用普通服务端 TLS（企业 CA 签发或自签）。调用方用令牌认证：
  - 宿主用**宿主令牌**，管理员在 Cloud 上为每台机器签发（A2）；
  - Node 用**项目令牌**，只能访问所属项目的 Git、模型代理和插件包（A2 签发，A4、A5 各自规定可用范围）；
  - 浏览器和 desktop 用用户会话（A6、#1029）。
- **不用 mTLS 和 SPIFFE 证书**：服务都在一个进程里，不再有服务间链路；Node 的身份由令牌表达，不再按代次签发
  Node 服务端、模型客户端、Git 客户端证书。
- **不要独立的 model-gateway**：模型请求经 Cloud 进程内代理；上游可以配成企业自己的模型网关（A5）。
- **运维**：备份 PostgreSQL 和 Cloud 数据卷（两者一起备份，Git 仓库和数据库记录才对得上）；提供离线部署包
  （Cloud、Node 镜像、宿主）、日志和健康检查（I9 #1059）。

为什么不用 mTLS：v1 用 mTLS 区分 9 种服务和每一代沙盒，是因为任何一个服务都可能被单独攻破。单进程之后，剩下的跨机器
链路只有"宿主 → Cloud"和"Node → Cloud"两类，调用方都由 Cloud 签发的令牌认证，服务端 TLS 足以防止窃听和冒充 Cloud。
代价是令牌一旦泄露，在吊销前可以被重放；所以令牌按项目或机器限定范围，可以吊销，并且 Node 的令牌对 Agent 进程不可见
的部分由 A3 的容器布局保证（见 D4）。

### D3：崩溃、断开和失联都按中断处理，不续跑

- 下文的“中断”指[中断](../../../domain/glossary.md#interrupted-execution)这一终态。
- **Cloud 重启**：启动时先把所有未结束的执行、会话轮次、任务运行和工作流节点标为中断（`interrupted`，原因写明是
  重启），再开始服务请求。v2-server 现在已经这样做（`execution.Restart`、`store` 里的 `server_restarted`）。
- **宿主断开或机器失联**：这台机器上正在跑的执行标为中断；第一版不做断线续传（A2）。
- **Node 进程退出**：正在跑的执行标为中断。
- **中断之后**：由业务层决定重做还是对账，例如对话由用户重新发送，任务运行回到可再次运行的状态，工作流节点按节点的
  重试策略处理。基础设施层不自动重放、不续跑、不保证同一步只执行一次。
- **权威数据只在 Cloud**：PostgreSQL 和 Cloud 的 Git 仓库是唯一的权威数据。Node 的数据目录、checkout 和本机缓存都
  不是权威数据，可以随时丢弃重建；机器挂掉时，checkout 里还没推送到 Git server 的改动会丢，这一点接受。

为什么接受"中断"：v1 为了续跑，在每一步登记意图、等待执行端证据、按稳定身份对账（意图加证据接管），配合租约、代次
围栏、停放和周期领取。#1023 测到这套机制在 happy path 上每次运行多出 5–17 s 的空等，而它保护的崩溃场景在企业内网的
少量副本部署里很少发生。业务层本来就要能处理"用户取消""模型失败"这类结局，"中断"只是多一种结局。

边界：中断只针对**执行**。已经提交到 PostgreSQL 的业务写入（建项目、改设置、保存工作流）按普通数据库事务保证，
不受影响；已经推送到 Git server 的 ref 同样保留。

反例：Agent 已经调用了外部系统（例如推回企业 GitLab）但 Cloud 在记录结果前崩溃。重启后这次推送标为中断，Cloud
不重放推送；用户再次发起时，A4 按外部仓库的当前状态重新判断（例如以 expected old 推送，已经推过就会发现快进条件变化）。
需要防重复的外部副作用，由发起它的领域在业务层用幂等键或前置条件处理，不由基础设施层统一兜底。

### D4：安全底线：Agent 碰不到 Cloud 的数据和凭据，Key 不进沙盒，项目是信任边界

1. **Agent 执行的代码碰不到 Cloud 的数据库、其他项目和凭据。**Node 跑在容器里；容器网络只能访问 Cloud 的入口
   （模型代理、Git、插件）和管理员配置的白名单；数据库、其他机器和宿主本机端口一律拒绝（A3 规定 `restricted` 和
   `open` 两种出网模式，两种模式都拒绝访问内网的数据库和其他机器）。
2. **模型 Key 不进沙盒。**Key 只在 Cloud 进程里解密和使用，Node 和 Agent 只拿到按会话签发、可吊销的临时令牌（A5）。
3. **外部仓库凭据不进沙盒。**外部同步和推回外部都由 Cloud 执行，Node 只访问 Cloud 的 Git server（A4）。
4. **项目是信任边界。**一个项目一个 Node 容器，同一项目里不同用户的会话共用这个容器；一个用户的 Agent 在技术上可能读到
   同项目其他会话的目录和临时令牌。团队成员本来共享项目的仓库，这一点接受（A3）。跨项目、跨租户的数据必须隔离。
5. **租户隔离**：每个 API 都按租户作用域和成员身份授权；I3 #1053 统一审计一遍现有接口。

与 v1 的差别：v1 的边界是"每次运行一个一次性沙盒"，并且 Key 连 Cloud 服务都不进（model-gateway 独立保管）。新的边界
放宽了两点：同项目的会话共用容器，Key 可以进 Cloud 进程。Cloud 被攻破时泄露面变大，这是去掉 model-gateway 的代价
（#1023 的 O2）。

### D5：持久化：不用全局锁，迁移版本化

- **不用全局串行锁。**普通请求用普通的行写入和数据库约束；需要互斥的地方按业务对象（例如同一会话、同一次工作流运行、
  同一个空间插件）取作用域锁，或在进程内互斥。只读请求不取锁。v2-server 现在就是这样做的（例如 `store/plugins.go`
  按空间加插件标识取 `pg_advisory_xact_lock`）。
- **跨表不变量**不能再靠"所有事务串行"兜底，每一条要写清由哪个行锁、约束或事务顺序保证；新增这类不变量的 ADR 要说明这一点。
- **迁移版本化**：v2-server 从"启动时建表"改为有编号的版本化迁移，升级要有测试覆盖（I1 #1051）。迁移编号规则沿用
  circus 记忆 `migration-numbering`：合并前按 main 最新编号顺延，已合并的迁移只追加、不改名。

为什么：全局锁是 v1 为了简化跨表不变量的推理而引入的，#1023 测到它让只读吞吐不随并发增加。单进程之后，大部分互斥
可以在进程内完成，剩下的跨请求互斥按业务对象就够了。

### D6：规格范围：只给保证清单写 ADR，v1 的 ADR 随组件退役标为 superseded

- v2-server 成为正式 Cloud 之后，phase3 基座的事项改 `v2-server/` 也要走 specs（根决策门禁、ADR、三个运行），
  不再享受 v2 的 specs 豁免（协议由 #1063 修改）。phase1 只改 v2 目录的演示事项仍按现行豁免。
- **只给下面清单里的保证写新 ADR**，不回头给 v2-server 已有的行为逐条补文档。v2-server 其余已有行为，以代码、
  候选契约（`*.openapi.json`）和 #1029 定下的正式契约为准；以后改动其中某一块时，再按正常流程为那一块补决策。
- v1 的 ADR 不在本文里改状态。对应组件退役时，由承接的 task 按"被取代的 v1 决策"一节标为 `superseded`，写明取代者
  或放弃的理由。

正式版必须有的保证：

| # | 保证 | v2-server 现状 | 决策 | 实现 |
| --- | --- | --- | --- | --- |
| 1 | 企业身份登录（IDaaS / OIDC） | 只有邮箱登录，仅用于演示 | A6 #1050 | I2 #1052 |
| 2 | 租户之间的数据隔离 | 已有租户作用域和成员校验 | 本文 D4 第 5 条 | I3 #1053 |
| 3 | Agent 执行隔离 | 没有：Node 是本地子进程 | A3 #1047 | I5 #1055 |
| 4 | Node↔Cloud 认证 | 自签证书 24 小时，不轮换 | 本文 D2，A2 #1046 | I4 #1054 |
| 5 | Key 不进沙盒、计量 | 已有（进程内代理、`requestmetrics`） | A5 #1049 | I8 #1058 |
| 6 | Git：内嵌事件驱动的 Git server、外部同步、Agent 交付 | 建项目时 mirror clone 一次，之后不 fetch | A4 #1048 | I6 #1056、I7 #1057 |
| 7 | 数据库迁移和升级 | 启动时建表 | 本文 D5 | I1 #1051 |
| 8 | 崩溃恢复 | 重启标中断 | 本文 D3 | 已有；新路径各自补齐 |
| 9 | 运维：备份、离线部署包、日志、健康检查 | 演示级 | 本文 D2 | I9 #1059 |
| 10 | 多机：Node 分布在多台机器上 | 单机本地子进程 | A2 #1046 | I4 #1054 |

### D7：演示期间新旧路径并行，演示结束后切换并退役 v1

1. **演示结束前**：`task v2:demo` 一条命令就能起来，走现在的"本地 Node 子进程 + 本地 mirror"路径。宿主、容器、
   Git server、企业登录作为新路径在旁边开发，默认关闭，不能改变演示路径的行为。
2. **演示结束后**：演示和默认部署切到内嵌宿主、容器和 Git server，删掉旧路径（I10 #1060）。
3. **然后 v1 退役**：删除 Controller、sandbox-server、v1 Git 线、model-gateway 和 v1 前端，同时按下一节把对应的
   v1 ADR 标为 `superseded`（I11 #1061）。

为什么并行而不是直接替换：演示日期未定，演示路径必须随时可用；新路径涉及容器和网络规则，开发期间不稳定。

## 被取代的 v1 决策

下表列出将被取代的 v1 ADR。"标记时机"之前，它们继续描述 v1 的行为，状态不变；到时由承接的 task 标为
`superseded`，并按 specs 的规则同步核心测试用例。

| 组 | v1 ADR | 取代者或放弃理由 | 承接 | 标记时机 |
| --- | --- | --- | --- | --- |
| 服务拆分与 Controller | [`controller/`](../../controller/README.md) 下全部（api-boundary、lifecycle、node-management、persistence、plugin、session） | A2：宿主转发 Node 协议帧，不做领取、租约和意图登记 | A2 #1046 | I11 #1061 删除 ora-controller 时 |
| Controller 集成与意图加证据 | [`cloud/controller-integration/`](../controller-integration/0-cloud-owned-internal-grpc-contract.md) 全部；[`cloud/operation/`](../operation/0-workspace-runtime-follows-desktop-node.md) 全部 | A2 加本文 D3：执行编排在进程内，崩溃即中断，不再登记意图、等证据接管 | A2 #1046 | I11 #1061 |
| 沙盒 | [`cluster/sandbox-server/`](../../cluster/sandbox-server/0-docker-label-resolver.md) 全部 | A3：宿主直接调 Docker，丢弃 effect journal、短期许可、代次围栏和 SPIFFE 证书 | A3 #1047 | I11 #1061 删除 sandbox-server 时 |
| Git 线 | [`cloud/git-server/`](../git-server/0-project-code-hub-git-server-over-https.md) 全部；[`cloud/repo-executor/`](../repo-executor/0-repo-executor-sole-external-git-boundary.md)；[`cloud/project/20261016-repository-binding-and-snapshot-sync.md`](../project/20261016-repository-binding-and-snapshot-sync.md)；[`cloud/project/20261017-explicit-fast-forward-publish.md`](../project/20261017-explicit-fast-forward-publish.md)；[`cloud/identity-access/20261016-git-credentials-in-independent-credential-store.md`](../identity-access/20261016-git-credentials-in-independent-credential-store.md)；[`cloud/revision/`](../revision/0-cloud-owned-object-store-and-verified-revisions.md) 全部 | A4：Git server 内嵌、事件驱动，外部同步和凭据都在 Cloud 进程内 | A4 #1048 | I11 #1061 删除 v1 Git 线时 |
| Node 拿代码与交付 | [`node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md`](../../node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)；[`node/revision/`](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) 中仍为 implemented / approved 的 | A4 规定 Node 从 Git server 首次 clone、之后增量 fetch，以及交付 ref；只改其中被修改的部分，按"历史决策"标注 | A4 #1048 | A4 approved 后，与 I6 #1056 一起 |
| 模型接入 | [`cloud/model-access/`](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) 全部；[`node/agent-runtime/20261010-run-scoped-model-proxy.md`](../../node/agent-runtime/20261010-run-scoped-model-proxy.md)；[`cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md`](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) | A5：Cloud 进程内代理，部署级模型连接，按会话签发临时令牌 | A5 #1049 | I11 #1061 删除 model-gateway 时 |
| 登录 | [`cloud/identity-access/0-gateway-mediated-external-identity.md`](../identity-access/0-gateway-mediated-external-identity.md)、[`20260923-huawei-global-id-alias.md`](../identity-access/20260923-huawei-global-id-alias.md)、[`20261019-deployment-bound-desktop-sessions.md`](../identity-access/20261019-deployment-bound-desktop-sessions.md) | A6：登录收进 Cloud 进程；IDaaS 的身份映射规则多半继承 | A6 #1050 | I11 #1061 删除 v1 gateway 时 |
| 全局串行锁 | 写在 [`cloud/workspace/0-collab-space-boundary.md`](../workspace/0-collab-space-boundary.md)、[`cloud/api-boundary/0-space-api-and-event-stream.md`](../api-boundary/0-space-api-and-event-stream.md)、[`cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md`](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md) 里，没有单独的 ADR | 本文 D5：不用全局锁 | 随所在 ADR 一起处理（见下一行） | 同下 |
| v1 业务领域 | `cloud/` 下其余目录：api-boundary、execution-admission、issue-collaboration、issue-run、persistence、plugin-marketplace、project（除上面列出的）、scheduled-task、squad、tenancy、thread、workflow、workspace | 待定，见"本决策未解决的问题"第 1 条 | #1062 | I11 #1061 之前由 #1062 逐个领域定 |
| Node 进程宿主 | [`node/process/`](../../node/process/README.md) 下 architecture、control、io、lifecycle、persistence、protocol、recovery、upgrade、integration | #1033 的 O5：退役 host/guardian，改为 ora-process 加 ora-reaper | #1033 的 O5 ADR task（待 @ObsisMc 开出） | O5 ADR approved 后 |

**不被取代、继续有效的**：

- [`node/protocol/`](../../node/protocol/0-controller-node-protocol.md)：Node 协议不改（A2 由宿主原样转发帧），文中的
  "Controller 侧"由 Cloud 承担。
- `node/` 下 agent-runtime（除模型代理那一份）、plugin-runtime、repository 的 clone 语义（除 Git server 来源那一份）：Node
  的本地行为不变。
- `desktop/` 下的全部决策：desktop 连接 cloud 的变化由 phase2（#249、#1029）处理，不在本文。

## 不变量

1. 正式 Cloud 的后端是一个进程；持久状态只在 PostgreSQL 和 Cloud 数据卷上的 Git 仓库。
2. Node 的数据目录、checkout 和本机缓存不是权威数据，丢弃后可以从 Cloud 重建。
3. Cloud 重启后，在开始服务请求之前，所有重启前未结束的执行都已标为中断。
4. 宿主断开或机器失联后，该机器上未结束的执行标为中断，不在别处续跑。
5. 基础设施层不自动重放被中断的执行。
6. Agent 执行的代码访问不到 Cloud 的数据库、其他项目的数据和任何凭据。
7. 模型 Key 和外部仓库凭据不进入 Node 容器。
8. 跨机器的链路都经 Cloud 的 HTTPS 入口，并用 Cloud 签发、可吊销、限定范围的令牌认证。
9. 只读请求不取全局锁；任何请求都不取全局串行锁。
10. 演示结束前，`task v2:demo` 一条命令就能起来，走旧路径，新路径不改变它的行为。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| A：在 v1 上做减法 | 保留 192 份 ADR 的积累，v1 的正确性证据可以继承 | 全局锁、意图加证据、周期推进、Controller 租约、一次性 Workspace、model-gateway 几乎都要改，总量接近重写，还要背着 9 种服务的部署（#1043）；与 D3 的崩溃语义冲突 |
| 在 v1 现有设计上逐项调优 | 改动最小，每步可以单独验证 | 解决不了根因：慢和复杂都来自"恰好一次续跑"的思路；#1017 已否定 |
| 保留恰好一次续跑，只合并进程 | 崩溃后不丢执行进度 | 续跑需要的意图登记、证据接管、代次围栏在单进程里照样要做，换来的场景（崩溃）很少发生；业务层已经要处理取消和失败（D3） |
| Cloud 多副本、水平扩展 | 消除 Cloud 单点 | 企业内网的规模不需要；多副本要重新引入进程间协调（事件广播、执行归属），与 D1 的简化方向相反。真有需要时另做决策 |
| 保留零信任 mTLS | 每条链路双向认证，令牌泄露不足以冒充 | 单进程后服务间链路不存在了；剩下两类链路用令牌加服务端 TLS 足够，且令牌可吊销（D2） |
| 保留独立 model-gateway | Key 不进 Cloud 进程，Cloud 被攻破时 Key 不泄露 | 多一个服务、三个 TLS 监听、主密钥卷和每代证书；企业内网通常已有自己的模型网关，上游可以直接配成它（A5） |

## 本决策未解决的问题

1. **v1 业务领域 ADR 的去向。**`cloud/` 下的业务领域（tenancy、squad、thread、workflow、scheduled-task、issue-* 等）
   描述的是 v1 Cloud 的行为，v2-server 各自另有实现，API 以 v2-server 为基准。它们是在 v1 退役时整体标为
   `superseded`（理由：由 v2-server 和 #1029 的正式契约取代），还是逐份判断其中仍成立的规则并继承到新 ADR，留给
   #1062 按领域决定，最迟在 I11 #1061 删除 v1 之前定完。
2. **Cloud 本身的高可用。**本文接受 Cloud 是单点（D1）。如果以后需要多副本，要另写决策，至少回答执行归属、
   事件广播和 Git 仓库存储三件事。
3. **断线续传。**宿主断开即中断（D3）；需要续传时由 A2 的后续决策处理。
4. **A2–A6 的目录。**由各 ADR task 在 `decisions/cloud/` 下选择叶子目录，作为各自领域的根决策，引用本文。

## 落地顺序

1. **本 ADR approved**：A2–A6 的 ADR task 开始起草；I1 #1051（版本化迁移）、I3 #1053（租户审计）可以开工。
   v2-server 现有行为不变。
2. **A2–A6 各自 approved**：对应实现 I2、I4–I8 开工，新路径默认关闭，演示路径不变（D7 第 1 步）。完成条件：每项
   实现都能在单机内嵌模式下验证，`task v2:demo` 仍然一条命令起来。
3. **离线部署包**（I9 #1059）：依赖 I4、I5，交付后企业内网可以多机部署。
4. **演示结束后切换**（I10 #1060）：默认走内嵌宿主、容器和 Git server，删掉旧路径。从这一步开始，演示环境的行为改变。
5. **v1 退役**（I11 #1061）：删除 v1 组件，按"被取代的 v1 决策"一节标记 ADR。从这一步开始，`cloud/` 下 v1 的
   ADR 不再描述任何运行中的系统。
