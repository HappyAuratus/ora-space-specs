---
status: proposed
---

# 模型接入：模型代理放在 Cloud 进程里，模型连接由部署配置文件给出，Node 用项目令牌为每个 Agent 会话申请临时令牌，Key 只在 Cloud 进程内，取代 v1 的 model-gateway 和个人模型连接

模型请求由 Cloud 进程自己代理，不再有独立的 model-gateway。能用哪些上游和模型由部署配置文件决定，所有租户共用；
上游可以是模型供应商，也可以是企业自己的模型网关。Key 只在 Cloud 进程的内存里，不入库，不进 Node 容器。Node 用
[项目令牌](../../../domain/glossary.md#project-token)向 Cloud 为每个 Agent 会话申请一个临时令牌，Agent 只拿到临时令牌和
Cloud 入口上的代理地址；会话结束、执行中断或项目令牌吊销时，临时令牌立即失效，进行中的请求被取消。Cloud 转发时强制
使用会话冻结的模型，客户端不能选择上游地址、认证头和模型。对话和任务的 Agent 会话，每次模型请求在发往上游之前先登记，
登记失败就不发。第一版没有个人或团队的模型连接，也没有限流和配额。

本 ADR 处于 `proposed`：等待 owner 评审，尚未实现；它是新建领域 `cloud/model-proxy` 的根决策。前提是
[Cloud 单进程根决策](../lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（下称“A1”）、
[多机执行层根决策](../execution-host/0-host-relayed-control-and-pinned-projects.md)（下称“A2”）和
[Node 容器根决策](../node-container/0-host-managed-docker-and-two-egress-modes.md)（下称“A3”）。依据是 circus #1043 的
结论 4、#1023 第四节的 O2 加 O3，以及 owner 在 #1073 定下的四点：连接放在部署配置文件、#809 保留并改写、计量范围维持
现状、第一版不做限流配额。读取的代码版本为 cloud `80e9c9f1`（`v2-server/internal/nodeservices`、`requestmetrics`、
`executionruntime`、`agentruntime`，以及 v1 的 `internal/modelgateway`）、desktop `dcff8afe`（`apps/ora-node` 的
`model_proxy`、`crates/node-protocol`）。

范围与兼容性边界：

- **涉及**：cloud 的 `v2-server/`（模型连接配置、授权接口、代理、计量）；desktop 的 Node 只按 A2 D7 把申请授权的认证
  从客户端证书改为项目令牌，其余不变。
- **Node 不用为本文另外修改**：授权接口的路径、请求和应答的形状，以及代理地址的两条路径都保持 Node 现在校验的样子
  （D2、D3）。
- **不在本文**：项目令牌的签发和吊销见 A2 D6；容器的出网规则见 A3 D3；谁能在别人发起的会话里追加消息，属于会话领域，
  以 v2-server 和 #1029 的契约为准。
- **演示约束**：继承 A1 D7。演示结束前，模型代理仍由每个项目的本机配套服务提供，连接仍读 `V2_MODEL_*`；本文的路径
  默认关闭，演示结束后由 I10（#1060）切换。
- **不做数据迁移**：v1 的个人模型连接、密文、运行绑定和用量记录都不迁移。

## 现状与继承

| 前序约定或现状 | 本文处理 |
| --- | --- |
| A1 D1、D2：model-gateway 收进 Cloud 进程；上游可以配成企业自己的模型网关 | 继承，并规定收进来之后的规则 |
| A1 D4 第 2 条：Key 只在 Cloud 进程里使用，Node 和 Agent 只拿按会话签发、可吊销的临时令牌 | 继承（D1、D3） |
| A1 D4 第 4 条：同项目的 Agent 可能读到其他会话的临时令牌 | 继承，并写明后果（D3） |
| A2 D1、D7：Node 直连 Cloud 入口；申请临时令牌时请求头带项目令牌 | 继承，并规定授权接口校验什么（D3） |
| v2-server `nodeservices`：每个项目一份 loopback 服务，授权接口要求客户端证书；临时令牌 10 分钟，只在内存 | 修改：接口挂到 Cloud 入口，校验项目令牌；令牌的期限和存放方式继承（D2、D3） |
| v2-server `executionruntime`：连接来自 `V2_MODEL_*`，每种协议一个上游、一个模型 | 修改：部署配置文件，可以有多个上游和模型（D1） |
| v2-server `requestmetrics`：对话和任务的 Agent 会话，发送前登记，登记失败不发上游 | 继承，范围不变（D5） |
| v1 [个人模型连接根决策](../model-access/0-personal-model-connections-and-run-scoped-proxy.md)：连接和 Key 归个人；D2 与不变量 2“原始 Key 不进入 Cloud 服务”；替代方案里否决了“共享部署 Key”和“Cloud 转发并解密 Key” | 取代：这两个被否决的方案就是本文的做法，理由见 D1 和“为什么不是这些替代方案” |
| v1 同一决策 D4、D5：授权只带绑定与执行标识、续期延长同一令牌；只有两种协议的 POST，客户端不能选地址、认证头和模型，不自动重试，上游错误不外泄 | 继承这些规则（D3、D4） |
| v1 [模型用量按 HTTP 尝试登记](../model-access/20261016-recorded-http-attempt-statistics.md) | 继承登记和解析的规则；归属和读取范围按 v2-server（D5） |
| v1 [部署方出网白名单](../model-access/20261017-deployment-owned-egress-and-explicit-resolution.md) | 取代：上游地址不再由用户填写，改由部署配置给出（D6） |

## 决策

### D1：模型连接由部署配置文件给出，Key 不入库

- **配置**：部署配置文件列出若干上游，每个上游有地址、协议、Key（或存放 Key 的文件路径）、可选的 CA 证书，以及它提供
  的模型（上游的模型名、显示名、上下文窗口、最大输出）。Cloud 启动时读一次；配置不合法时拒绝启动。
- **协议**：只有 `openai-completions` 和 `anthropic-messages` 两种。Anthropic 协议的上游可以配成用 `x-api-key` 或
  `Authorization: Bearer` 送 Key。
- **地址**：必须是 `https`，不带用户信息、查询串和片段。
- **共用**：整个部署一份模型目录，所有租户、项目和用户都能用，不按租户区分。执行器和模型的配对规则沿用 v2-server。
- **Key 的位置**：只在配置文件（或它指向的文件）和 Cloud 进程的内存里。不写进 PostgreSQL，不出现在 API 应答、日志和
  下发给 Node 的任何消息里。
- **修改**：改配置文件后重启 Cloud。重启会中断正在跑的执行（A1 D3）。
- **没有管理界面**：Cloud 只提供只读的模型目录接口，返回标识和显示名，不含地址和 Key。
- **没有个人和团队连接**：第一版用户不能自己添加模型连接或 Key。是否在部署级连接之外再加，由 circus #809 决定。

为什么是部署级连接而不是个人连接：v1 让每个人配自己的 Key，换来的是额度按人隔离，代价是没有 Key 的成员跑不了任何
Agent，协作时别人也接不上手（#809）。企业内网部署通常由企业统一采购模型或自建网关。代价：用量不能按个人的 Key 自然
隔离，只能靠计量记录区分（D5）；一个项目可以用掉整个部署的上游额度（D7）。

为什么 Key 可以进 Cloud 进程：v1 把解密单独放在 model-gateway，是为了 Cloud 被攻破时 Key 不泄露，代价是多一个服务、
三个 TLS 监听和一个要与数据库一起备份的主密钥卷。A1 D4 已经接受这个泄露面。Key 不入库，所以数据库和它的备份里
没有 Key。

### D2：授权接口和代理都在 Cloud 的 HTTPS 入口上

| 路径 | 调用方 | 认证 |
| --- | --- | --- |
| `POST /internal/v1/model-grants` | Node | 项目令牌 |
| `POST /internal/v1/model-grants/{grantId}/renew`、`DELETE /internal/v1/model-grants/{grantId}` | Node | 项目令牌 |
| `POST /runtime/openai/v1/chat/completions` | Agent | 临时令牌（`Authorization: Bearer`） |
| `POST /runtime/anthropic/v1/messages` | Agent | 临时令牌（`x-api-key`） |

- 这些路径固定在 Cloud 入口的根下，与 Git、API 同一个主机。Node 现在就要求代理地址的主机等于授权入口的主机、路径
  等于 `/runtime/openai/v1` 或 `/runtime/anthropic/v1`；部署在反向代理后面时不能给它们加路径前缀。
- 其他路径和方法一律拒绝。不提供模型列表、token 计数等接口；OpenAI Responses 等其他协议不在第一版。
- 授权和代理的状态从“每个项目一份”变成整个进程一份，项目之间的隔离由 D3 的校验保证。

### D3：Node 用项目令牌为每个 Agent 会话申请临时令牌

- **先登记，再申请**：Cloud 在让 Node 启动一个 Agent 会话之前，先在进程内记下“这个会话的执行标识可以用这个模型绑定”。
  这里的执行标识是 Cloud 下发给 Node 的那次 Agent 会话的标识，一个会话的多轮对话共用一个。
- **申请**：Node 带项目令牌请求授权，请求体只有 `{bindingId, executionId}`。Cloud 核对三件事：项目令牌有效；这个执行
  属于令牌所属的项目，并且是在令牌对应的这一次容器启动里下发的；执行标识和模型绑定与登记的一致。任何一项不符都拒绝，
  应答不区分原因。
- **应答**：`{grantId, token, expiresAt, protocol, proxyBaseUrl, model}`，形状与现在相同，不增加字段（Node 拒绝未知
  字段）。`proxyBaseUrl` 是 Cloud 入口加上 D2 的协议路径。
- **期限**：临时令牌 10 分钟有效；Node 在到期前续期，续期延长同一个令牌。同一个执行再次申请时，旧令牌作废。
- **吊销**：会话结束、执行中断（A1 D3、A2 D3）、项目令牌吊销、Node 主动删除授权时，临时令牌立即失效，用它发起的
  进行中的请求同时被取消。会话正在按用户的要求结束时，续期得到 `403 model_session_ending`，其余情况得到
  `403 model_access_revoked`。
- **存放**：临时令牌只在 Cloud 进程的内存里，不入库。Cloud 重启后全部失效；重启前的执行都已标为中断，不需要它们。
- **令牌只在自己的协议路径上有效**。

**同项目的 Agent 能借用临时令牌**：令牌经环境变量或内存交给 Agent，同一容器里别的会话在技术上读得到（A1 D4 第 4 条
已接受）。借用产生的请求用的是原会话冻结的模型，用量记在原会话名下；它越不过项目边界，也拿不到 Key。

为什么临时令牌不存哈希入库：项目令牌要跨 Cloud 重启之外的整个容器生命周期核对，所以存哈希（A2 D6）；临时令牌只有
10 分钟，又随执行一起作废，入库只会多出一批要清理的记录。

### D4：转发时强制使用冻结的模型，进行中的请求随授权吊销而取消

- **客户端不能选择**上游地址、认证头和模型。Cloud 把请求体里的模型名改成这个会话冻结的那一个，换上上游的 Key，其余
  字段原样转发。
- **请求头**：Agent 的请求头和临时令牌都不转发；只有 `Anthropic-Version` 和 `Anthropic-Beta` 透传。
- **应答**：成功时只接受 JSON 或 SSE，边收边转，不缓冲。上游的非成功应答不转给 Agent，只返回固定的错误码
  （上游拒绝、上游不可达、应答不合法）。
- **不重试，不跟随重定向，不使用环境里的代理设置**。
- **大小与时限**：请求体上限 4 MiB；单次请求有时限，默认 30 分钟，部署可配。
- **进行中的请求**：只要授权没有被吊销、时限没到，就一直转发到上游结束；授权吊销时立即取消。请求开始之后令牌是否
  续期，不影响这次请求。
- **模型请求不在数据库事务里进行。**

为什么进行中的请求不按令牌到期时间截断：v2-server 现在把请求的截止时间定在开始那一刻的令牌到期时间，而 Node 在到期前
一分钟才续期，一次较长的推理可能在一两分钟后被切断。吊销才是“不该再用”的信号，到期只约束新请求。

为什么 `Anthropic-Beta` 透传而不设白名单：v1 的白名单是按某个固定的 CLI 版本写的，CLI 升级就要改 Cloud。上游由部署方
配置并承担后果，Agent 能影响的只是自己这次请求的功能开关。

### D5：对话和任务的 Agent 会话，模型请求先登记再发送

- **范围**：对话和任务运行的 Agent 会话经代理发出的每一次模型请求。工作流节点和定时任务触发的 Agent 会话、以及不经
  Node 的模型调用（D8）不登记，这是 v2-server 的现状，owner 定为第一版维持不变。
- **发送前登记**：在本地校验通过之后、发往上游之前写入一条“已开始”的记录；写入失败时返回
  `503 model_metrics_unavailable`，不发往上游。会话或这一轮已经结束时不能登记。
- **结束后补全**：记录结果（成功、失败、取消、中断、不完整）、上游状态码、耗时、首次输出时间和 token 用量。补全用不随
  请求取消的短时限重试几次；仍然失败时记录停在“已开始”。
- **用量的解析**沿用 v1 的规则：只取白名单里的字段，缺失或不合法的保持未知而不是零，流式应答里的用量是快照、不累加，
  不替 Agent 注入要求返回用量的参数，不推算费用。
- **不记录**请求和应答的正文、请求头、临时令牌和 Key。
- **归属**由记录所属的会话决定，客户端不能指定；租户、用户和项目经会话得到。
- **读取**：只有按单个会话的统计和明细，可见范围跟随会话。

为什么登记失败就不发：用量记录是部署方了解共用 Key 被谁用掉的唯一依据（D1）；先发后记，一旦记录失败就出现没有记录的
消耗。代价是数据库不可用时这些会话的模型请求也不可用。

### D6：上游地址只来自部署配置，不另设出网白名单

- Cloud 只向部署配置文件里列出的上游地址发模型请求，用户和 Agent 都不能让它访问别的地址。
- 上游可以在企业内网（私有地址），可以用企业 CA 签发的证书（D1 的 CA 配置）；TLS 按配置的主机名校验。
- 不继承 v1 的 `deployment-egress.json`、DoH 和地址分类：那些规则防的是用户自己填写上游地址，这个入口已经不存在。

### D7：第一版没有限流和配额

Cloud 不限制每个用户、项目或租户的模型请求速率、并发和 token 用量，也没有预算（owner 2026-11-11 定）。上游的限额
由上游自己执行，上游拒绝时 Agent 得到 D4 的固定错误码。代价：一个项目可以占满整个部署的上游额度，其他项目的请求
随之失败；这种情况只能事后从 D5 的记录里看出来，而且只覆盖 D5 范围内的会话。

### D8：不经 Node 的模型调用在进程内直接使用同一份连接

Cloud 自己发起的模型调用（现在只有没有项目的定时对话）在进程内直接使用 D1 的连接，不经过 D2 的 HTTP 路径，不签发
临时令牌，也不登记（D5）。它同样遵守 D4 的“不重试、不跟随重定向、上游错误不外泄”和 D6。

## 被取代的 v1 决策

继承 A1“被取代的 v1 决策”中模型接入一组的安排，取代者为本文；标记之前它们的状态不变。

| v1 ADR | 被本文哪一条取代 | 标记时机 |
| --- | --- | --- |
| [`cloud/model-access/0-personal-model-connections-and-run-scoped-proxy.md`](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) | D1–D4；授权和转发的规则由 D3、D4 继承 | I11（#1061）删除 model-gateway 时 |
| [`cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md`](../model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md) | D1：没有个人的槽位映射；执行器和模型的配对按 v2-server | 同上 |
| [`cloud/model-access/20261016-recorded-http-attempt-statistics.md`](../model-access/20261016-recorded-http-attempt-statistics.md) | D5 | 同上 |
| [`cloud/model-access/20261017-deployment-owned-egress-and-explicit-resolution.md`](../model-access/20261017-deployment-owned-egress-and-explicit-resolution.md) | D6。A4 的外部 Git 出网规则另见[项目 Git 根决策](../project-git/0-embedded-git-server-and-in-process-external-sync.md) D8 | 同上 |
| [`cloud/model-access/20261017-personal-agent-configurations.md`](../model-access/20261017-personal-agent-configurations.md) | D1：没有个人连接和个人模型选择 | 同上 |
| [`cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md`](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) | D3，以及 A2 D7 | 同上 |
| [`node/agent-runtime/20261010-run-scoped-model-proxy.md`](../../node/agent-runtime/20261010-run-scoped-model-proxy.md) 中“用模型客户端证书申请授权” | 部分取代：改为项目令牌（A2 D7）；其余继续有效，按“历史决策”标注 | circus #1066 的 desktop 修改合入时 |
| [`cloud/thread/20261010-personal-model-initiator-controls-input.md`](../thread/20261010-personal-model-initiator-controls-input.md) | 它的理由（保护个人额度）随 D1 消失。A1 把它归在“v1 业务领域”一组，随该组整体标记 | I11（#1061） |

[`node/agent-runtime/20261017-plugin-owned-model-adapters.md`](../../node/agent-runtime/20261017-plugin-owned-model-adapters.md)
讲的是 Node 和插件之间怎么分工，本文不改。

## 不变量

1. 上游的 Key 只出现在部署配置和 Cloud 进程的内存里，不入库，不进 Node 容器，不出现在 API 应答和日志里。
2. Cloud 只向部署配置里列出的上游地址发模型请求。
3. 申请临时令牌必须带有效的项目令牌，并且只能为这个项目、这一次容器启动里下发的执行申请。
4. 临时令牌只能用于它绑定的那个会话的模型绑定，转发时的模型一定是这个绑定的模型。
5. 会话结束、执行中断或项目令牌吊销之后，临时令牌不能发起新请求，用它发起的进行中的请求已被取消。
6. Cloud 重启后，重启前签发的临时令牌全部无效。
7. 对话和任务的 Agent 会话，每一次发往上游的模型请求在发送前都有一条登记记录；登记失败的请求没有发往上游。
8. 用量记录不含正文、请求头、临时令牌和 Key；缺失的用量是未知，不是零。
9. 上游的错误应答正文不会转给 Agent。
10. Cloud 不自动重试模型请求。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 保留独立的 model-gateway（#1023 的 O1） | Key 不进 Cloud 进程 | 多一个服务、三个监听和主密钥卷；#1043 结论 4 已否定 |
| Key 注入 Node 容器，Agent 直连上游（O4） | 最简单，没有代理 | 违反 A1 D4“Key 不进沙盒”，也没有用量记录 |
| 管理员经 API 配置连接，Key 加密存 PostgreSQL | 改连接不用重启，可以有管理界面 | 多一组管理接口和权限，Key 进入数据库备份；部署级连接很少变化（D1） |
| 按租户设置能用哪些模型 | 多租户部署可以区分 | 企业内网部署通常只有少数租户；需要时在本文之上追加 |
| 第一版就支持个人或团队连接 | 用量按付费方隔离 | 要带上连接归属、付费方选择和追加权限三组规则；由 #809 另行决定 |
| 所有模型请求都登记（含工作流、定时任务和进程内调用） | 部署方能看到全部消耗 | owner 定为第一版维持 v2-server 的范围；缺口写在未解决问题里 |
| 每个项目设并发上限或 token 预算 | 一个项目占不满上游 | owner 定为第一版不做（D7） |
| 临时令牌存哈希入库 | 与项目令牌一致，Cloud 重启后可以继续用 | 重启前的执行都已中断，不需要继续用（D3） |
| 进行中的请求在令牌到期时截断 | 实现简单，令牌的寿命有硬上限 | 长推理会被无故切断；吊销已经能及时取消（D4） |

## 本决策未解决的问题

1. **计量的覆盖范围**：工作流节点和定时任务触发的 Agent 会话、不经 Node 的模型调用都不登记，部署方看不到这部分消耗。
2. **跨会话的用量读取**：没有按用户、项目、租户或时间段的汇总。
3. **限流、配额和预算**：见 D7。
4. **个人或团队的模型连接**：由 circus #809 决定。
5. **不重启就更换连接或 Key**：第一版要重启 Cloud。
6. **更多协议**：OpenAI Responses 等，接入对应的 Agent 时另写决策。

## 落地顺序

1. **Node 改用项目令牌申请授权**（circus #1066，A2 落地顺序第 1 步）：与本文无关的部分已由 A2 规定。
2. **I8（#1058）连接配置**：D1、D6，`V2_MODEL_*` 作为演示路径的写法保留到 I10。完成条件：配置文件里的多个上游和模型
   出现在模型目录里；Key 不出现在 API 应答和日志里；企业 CA 的上游能连通。
3. **I8 入口上的授权和代理**：D2–D4，默认关闭，演示路径不变。完成条件：跨项目、跨容器启动的申请被拒绝；会话结束、
   执行中断和项目令牌吊销各自让进行中的请求取消；一次超过令牌期限的流式请求不被截断。
4. **I8 计量**：D5 的规则在新入口上成立。完成条件：登记失败时上游收不到请求。
5. **I10（#1060）**：演示切到 Cloud 入口上的代理，删除本机配套服务里的模型路由和 `V2_MODEL_*`。从这一步开始，演示
   环境的行为改变。
6. **I11（#1061）**：删除 model-gateway，按“被取代的 v1 决策”一节标记 ADR。
