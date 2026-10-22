---
status: approved
---

# Git Server 是 Project 的代码中心：Cloud 仓库中的独立 HTTPS 服务，证书只认证身份，授权按 Cloud 当前状态逐请求判定

Git Server 是 Cloud 仓库中与 model-gateway 并列的独立程序，为每个 Project 托管一个裸仓库，是平台管理的代码输入、
Workspace 协作与 Revision 交付的唯一存放处。它只提供 smart HTTP（HTTPS），不提供 SSH。每类访问者持部署签发的专用
客户端证书做双向 TLS，证书只回答“这是谁”，不携带任何仓库或 ref 权限；每个请求再声明自己服务的上下文，Git Server
直接读取 Cloud 数据库，按证书身份、上下文与当前状态判定能否访问该仓库、能写哪些 ref，推送在 `pre-receive` 中再判定一次。
Git Server 不保存授权状态，不持有外部仓库凭据，也不访问外部仓库；Cloud 不写 Git Server，只在事务外查询、在事务内登记。

本 ADR 已经 owner 批准，处于 `approved`，是 `cloud/git-server` 领域的根决策；它的不变量与契约 ADR 共用一组核心测试用例，见[代码中心契约用例](../../../test-cases/cloud/git-server/project-repository-code-hub-contract.md)。现有 Git Server 实现与本文的程序形态、协议和沙盒认证一致，契约 ADR 的两条开发线都落地并验收后，本文才改为 `implemented`。它取代
[Git Server 接收沙盒推送的原根决策](0-sandbox-scoped-git-server-over-https.md)：原根决策写在 Git Server 只存交付 ref、
Node 仍从外部仓库 clone 的设计上，Git Server 成为 Project 代码中心之后整体作废；本文按新设计重新写出当前的根决策，
不需要再读原文。ref namespace、规则集授权、各监听接受的身份与两条开发线交接的数据形状，由
[代码中心契约决策](20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约 ADR”）规定；Revision 怎样交付与
续作由 [Revision Git 原生交付决策](../revision/20261017-git-native-delivery-refs-and-resume.md) 规定。涉及 Cloud 仓库的
Git Server 程序、cluster 仓库的 Sandbox Server（签发 git-access 身份）与 Compose 部署、desktop 的 Node。

兼容性边界：本文不改变现有 Git Server 实现的程序形态、协议、仓库路径与认证方式；它改变的是 Git Server 在系统中的角色
（D7），具体落地按契约 ADR 的落地顺序，并随之破坏性切换，不保留 bundle 与外部 clone 的旧路径。会话 JSONL 仍按
[Cloud Revision 根决策](../revision/0-cloud-owned-object-store-and-verified-revisions.md)存放在对象存储，不经过 Git Server。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [原 Git Server 根决策](0-sandbox-scoped-git-server-over-https.md) D1–D6：独立程序、smart HTTP、按 Project 建仓；沙盒 git-access 证书；逐请求判定与 `pre-receive` 复核；Node 执行 Git；监听分开、Cloud 经 ref 查询核验 | 按新设计重写为本文 D1–D6。编号与原文一一对应，契约 ADR 引用的“根决策 Dn”指本文 |
| [原 Git Server 根决策](0-sandbox-scoped-git-server-over-https.md) D7：Node 照旧从外部仓库 clone，Git Server 不镜像外部仓库 | 修改：外部仓库的内容只经持有外部凭据的组件进出 Git Server，Workspace 的代码输入只来自 Git Server（D7） |
| [Cloud Revision 根决策](../revision/0-cloud-owned-object-store-and-verified-revisions.md) D2、D4：每次交付尝试一个只能首次创建的对象；Cloud 在事务外核验、事务内登记 | 继承同一结构：Cloud 在事务外经 ref 查询核验 Git Server 上的 ref，再在事务内登记（D6）；代码不再存对象存储 |
| [Cloud 决策目录](../README.md)：Cloud 是多租户业务状态的权威，Controller 与 Node 不建立另一份业务权威 | 继承：授权只依据 Cloud 数据库的当前状态（D3），Git Server 不保存授权副本 |
| [个人模型访问](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) D2、D4：model-gateway 是 Cloud 仓库中的独立程序，直接读 Cloud 数据库，授权由证书目标与当前状态共同决定 | 继承同一模式（D1、D3）；不继承临时令牌，理由见 D3 |
| [模型客户端身份](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) D1、D2：按用途另签 `clientAuth` 身份，只作为私有启动材料存入管理 tmpfs；以不同监听区分用途 | 继承同一模式：git-access 身份与 model-access 身份互不通用（D2）；每类身份一个监听（D6） |
| [Controller 只拨出](../../controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)，不暴露监听 | 继承，不变：Controller 不向 Git Server 登记或撤销任何授权 |
| [clone 选定分支](../../node/repository/0-clone-selected-repository-branch.md)：Node 从外部仓库 clone | 修改：Workspace 的代码输入只来自 Git Server（D7），Node 怎样 clone 由 `node/repository` 后续 ADR 规定 |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md)：Cloud 只管理凭据引用，执行端不回退全局静态凭据 | 继承：Git Server 不接受口令、静态 token 或匿名访问，也不持有外部仓库凭据（D2、D7） |

## 决策

### D1：Cloud 仓库中的独立程序，只提供 smart HTTP，每个 Project 一个仓库

- Git Server 是 Cloud 仓库中的独立程序（与 `cmd/model-gateway` 并列），单独部署，在持久卷上托管裸仓库。
- 每个 Project 有且只有一个仓库 `/{tenantId}/{projectId}.git`，同一 Project 的所有 Workspace 与该 Project 的外部仓库同步都
  使用它。租户之间、Project 之间不共享仓库、对象目录或 `objects/info/alternates`。
- 只开放 smart HTTP 的三个端点：`GET /{repo}/info/refs?service=...`、`POST /{repo}/git-upload-pack`、
  `POST /{repo}/git-receive-pack`，协议处理交给 `git http-backend`；dumb HTTP、其他路径与方法返回 `404`。
- 仓库按 `projectId` 命名，不按外部仓库地址：外部地址可以改变，也可能带有用户信息；`projectId` 稳定，且已在 Cloud 中与租户
  绑定。路径中的 `tenantId` 只用于与身份做一致性检查，不作为授权依据。
- 仓库在第一次获授权的写入时以原子方式创建（临时目录初始化后 rename 发布）；读请求不建仓。

为什么放在 Cloud 仓库：授权依据全部是 Cloud 数据库中的状态，schema 由 Cloud 拥有。读取这些状态的查询与 schema 在同一仓库内
一起评审和测试，不形成跨仓库共享 schema，这也是 model-gateway 已采用的结构。Compose 部署仍在 cluster 仓库。

为什么选 HTTPS 而不是 SSH：

| 维度 | HTTPS | SSH |
| --- | --- | --- |
| 身份 | 沿用 Sandbox Server 已有的 X.509 按代签发流程，服务身份同样用 X.509 | 需要另建 SSH CA 或公钥注册表，并分发 host key |
| Node 策略 | Cloud 模式本来只允许 HTTPS，只需增加证书引用 | Cloud 模式禁止 SSH，需放开策略并在镜像中带 ssh 客户端 |
| 授权挂点 | HTTP handler 按路径、`service` 与请求头区分仓库、读写与上下文 | 需解析 `exec` 命令串，上下文需另找通道传递 |
| 部署 | 普通 HTTP 监听，Compose 与网关都现成 | 需额外 TCP 端口，L7 网关不可用 |

### D2：认证只认部署签发的专用客户端证书，证书不携带权限

- 每类访问者持一种专用的 `clientAuth` 证书，用途写在 URI SAN 的第一段，trust domain 为 `ora.local`：
  - 沙盒：Sandbox Server 为租户、Workspace 与代次另签的 git-access 证书，URI 为
    `spiffe://ora.local/git-access/tenant/{tenantId}/runtime/{workspaceId}/generation/{generation}`；只作为 Docker 私有启动材料，
    由 Node entrypoint 存入管理 tmpfs，进程配置只引用绝对路径，不进入 effect 请求、journal 或结果；
  - 平台服务（例如持有外部凭据、访问外部仓库的组件）：由独立 CA 签发的服务身份，具体身份由契约 ADR 规定；
  - Cloud：ref 查询使用的固定客户端身份，与 Controller 访问 Cloud 的身份分开。
- 每个监听只信任自己那类身份的 CA，证书必须在有效期内、带 `clientAuth`、恰好含一个该用途的 URI。WSS 服务端证书与
  model-access 证书都不被接受，git-access 证书也不能用于模型代理。
- 没有其他认证方式：不接受 Basic、Bearer、匿名访问，也不接受管理身份访问仓库内容。认证失败在 TLS 握手阶段终止，或返回不含
  细节的 `401`。
- 证书不表示任何仓库或 ref 权限。它的有效期只限定身份能被出示多久，不限定权限多久。

为什么按用途另签，而不复用 WSS 证书：已批准的模型客户端身份决策明确不把 WSS 服务端证书改为双用途；按用途分开签发，一种身份
泄露不能直接换用到另一服务。

### D3：每个请求按身份、声明的上下文与 Cloud 当前状态判定

每个仓库请求都带一个上下文头，声明它服务的执行或 Operation：证书决定“是谁”，上下文头只说明“为了什么”，不能自行声明身份。
具体的头、它与监听的对应关系由契约 ADR 规定。Git Server 在不持有事务的只读查询中，依据 Cloud 数据库的当前状态判定：

- 身份是当前有效的访问者。对沙盒：证书中的 `(tenantId, workspaceId, generation)` 是该 Workspace 当前存活的沙盒代次，声明的执行
  属于它、尚无结果、未被强停终止，执行所在运行仍处于允许该操作的阶段，触发成员与账号仍然有效。
- 上下文与目标仓库一致：请求路径中的 `tenantId` 等于身份的租户，`projectId` 等于上下文所属的 Project。
- 判定的结果是一份授权：能否读该 Project 仓库，以及能写哪些 ref、各按什么规则写。授权内容由上下文的类型决定，具体规则集由
  契约 ADR 规定；读端点要求读权限，`receive-pack` 要求非空的写权限。

| 情况 | 响应 |
| --- | --- |
| 判定通过 | 交给 `git http-backend` |
| 任一条件不满足 | `403`，不含细节；仓库不存在与无权访问返回相同响应 |
| 缺少或格式错误的上下文头 | `400` |
| 数据库不可用或查询超时 | `503`，不放行，不使用任何缓存 |

为什么授权不绑在证书上：证书在有效期内无法及时吊销，签发方也不知道仓库关系。执行结束、沙盒换代、Operation 结束、成员离开
都需要立即生效，只有逐请求读取当前状态才做得到。Git Server 不保存授权副本，重启不丢失任何授权，状态变化在下一个请求生效。

为什么不像 model-gateway 那样再发临时令牌：model-gateway 需要令牌，是因为调用模型的是工作负载，而工作负载不能拿到证书私钥。
Git 访问由持证组件自己执行（D5），证书已能证明身份，再加一层令牌只多一份需要续期和撤销的凭据。

### D4：写入只经 `pre-receive` 按授权逐个 ref 判定

- HTTP handler 把身份与上下文通过环境变量交给 `git http-backend`，`pre-receive` 在写入前**重新**按 D3 判定：一次推送至少有
  `info/refs` 与 `git-receive-pack` 两个请求，对象传输可能很长，写入前再判定一次，堵住其间执行结束、代次变化或 Operation
  结束的窗口。
- 每个 ref 更新都必须落在授权列出的 ref 与规则内；任一 ref 不合规则，整次推送不写入，不部分写入。
- 任何 ref 都不能删除：仓库固定 `receive.denyDeletes=true`、`receive.fsckObjects=true`，不接受 push options。hook 与仓库配置由
  Git Server 管理，客户端无法修改；需要按监听区分的配置，由 Git Server 按请求注入。
- Git 对象传输与 hook 执行都不在数据库事务或锁中进行；每次判定是独立的短只读查询。

ref namespace 和 create-only、fast-forward、tracking-cas 等规则由契约 ADR 规定。根决策只固定“写入前复核、逐 ref 判定、不能删除、
整次拒绝”这几条，使新增写入者只需增加规则来源，而不放宽这些底线。

### D5：Git 访问由持证组件自己执行，私钥不进入工作负载

- 沙盒里由 Node 进程执行 `git` 命令。git-access 证书与私钥来自管理 tmpfs 中的文件，只在当次命令中以
  `-c http.sslCert=... -c http.sslKey=... -c http.sslCAInfo=...` 传入，上下文头以 `-c http.extraHeader=...` 传入；不写入 checkout 的
  `.git/config`、全局 Git 配置、日志或 journal。
- 平台服务以自己的服务身份访问，私钥同样不交给任何工作负载。
- Git Server 的地址由 Controller 随执行输入原样转交；Controller 不为此获得额外权限。
- Agent 在会话中自己执行 `git push` 没有凭据，会失败；平台管理的写入只经持证组件进入 Git Server。

已知缺口：只要工作负载能读到 git-access 私钥（模型客户端身份决策同样记录了既有 root 插件进程与 guardian 纳管缺口），它就能以
本沙盒身份访问 Git Server。最坏结果被 D3、D4 限定在本沙盒当前进行中的执行内：只能读本 Project 的仓库，只能按该执行的授权写入。
这一缺口由工作负载降权解决，不由 Git Server 解决。

### D6：每类身份一个监听；Cloud 只经只读 ref 查询核验

- 每类身份一个监听，各自只接入一个网络：沙盒的仓库监听只接入沙盒网络；平台服务的监听只接入内部服务网络；ref 查询监听只接入
  Compose 默认网络，只接受 Cloud 身份。沙盒访问不到沙盒网络之外的监听。
- ref 查询只读：给定 Project 与 ref（或 ref 前缀），返回 ref 当前指向的提交或“不存在”；不返回仓库内容，不提供任何写操作。
- Git Server 访问 Cloud 数据库经默认网络；默认网络上不提供仓库读写。仓库数据放在持久卷，Compose 不向宿主发布端口。
- Cloud 不写 Git Server。Cloud 需要登记的事实都由写入者先写 ref，Cloud 再按“事务外经 ref 查询确认 ref 指向声明的提交，事务内
  登记”的顺序处理；查询失败不登记、可重放，ref 不符按核验失败处理。

为什么由 Cloud 查询，而不是由 Git Server 写收据：收据方案下，ref 已写入而收据写入失败时会出现“推上去了却没有记录”，需要另做
对账；登记的一方也应亲自核验它所承诺的事实。查询方案省掉数据库与 Git 之间的对账，也不需要 Cloud 写接口、outbox 或 reconciler。

### D7：外部仓库只经持有外部凭据的组件进出 Git Server

- Git Server 不持有外部仓库凭据，也不访问外部仓库。外部仓库的内容只经持有外部凭据的平台服务写入 Git Server，向外部仓库的写入
  同样只经它从 Git Server 读取源提交后完成；这个组件、它的身份与写入规则由契约 ADR 与后续决策规定。
- Workspace 的代码输入只来自 Git Server。Node 不为平台管理的代码输入访问外部仓库，沙盒里没有外部仓库凭据。
- 沙盒的推送只到 Git Server，不会触达外部仓库。

为什么这样划分：外部凭据集中在一处，Node、Workspace、Agent 和 Git Server 都不接触它；多个 Workspace 经同一个 Project 仓库协作；
Revision 的交付与续作都在 Git Server 上完成，不依赖从外部仓库取得基础提交。

## 不变量

1. 每个监听只接受自己那一类身份的客户端证书，没有口令、令牌或匿名访问。
2. 证书不携带授权；每个请求和每次 `pre-receive` 都按身份、声明的上下文与 Cloud 数据库当前状态判定。
3. Git Server 不保存授权状态；它的重启不改变任何判定结果。
4. 无法读取 Cloud 当前状态时，Git Server 拒绝请求，不以任何缓存或证书有效性放行。
5. 推送只能写授权列出的 ref；任何 ref 不能删除；任一 ref 不合规则，整次推送不写入。
6. 在 `info/refs` 之后、`git-receive-pack` 写入之前上下文失效，该次推送被拒绝。
7. 不存在的仓库与无权访问的仓库返回相同响应；跨租户、跨 Project 的仓库与对象不可达。
8. git-access 私钥不写入 checkout、Git 配置、日志或 journal。
9. Git 对象传输与 hook 执行不在数据库事务或锁中进行。
10. ref 查询只对 Cloud 身份开放且只读；沙盒访问不到沙盒网络之外的监听。
11. Cloud 不写 Git Server，只在 ref 查询确认之后登记。
12. Git Server 不持有外部仓库凭据，不访问外部仓库；Workspace 的代码输入只来自 Git Server。
13. 会话 JSONL 不经过 Git Server。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| SSH 协议 | 对人更熟悉；但需要另一套 CA 或公钥表、host key 分发，并放开 Node 的 Cloud 策略（D1） |
| 复用 WSS 服务端证书，增加 `clientAuth` 用途 | 少签一张证书；但违反已批准的模型客户端身份决策，一份私钥同时可用于被 Controller 连接与访问仓库（D2） |
| 把仓库权限写进证书 | Git Server 无需查询；但有效期内无法撤销，签发方需要知道仓库关系，违反不变量 2、6 |
| Controller 向 Git Server 推送访问绑定 | Git Server 不依赖 Cloud 数据库；但绑定是 Cloud 状态的副本，撤销有延迟，Git Server 重启后要等重新登记，违反不变量 3 |
| Git Server 留在 cluster 仓库，调用 Cloud 新增的内部鉴权接口 | 按部署层次划分更整齐；但要多设计一个接口和一套调用身份，且偏离 model-gateway 已采用的结构（D1） |
| 推送成功后由 Git Server 写收据，Cloud 不再查询 | 少一个监听；但 ref 写入与收据写入不是一个原子操作，失败时需要对账，登记方也不再亲自核验（D6） |
| Node 继续从外部仓库 clone | 不需要同步；但外部凭据要进入 Node，多个 Workspace 没有共同的仓库，续作依赖外部仓库保留基础提交（D7） |
| 部署 Gitea / GitLab 等现成服务 | 自带界面与镜像功能；但授权以用户与 token 为单位，无法按运行时代次、执行和 Operation 授权，运维成本也更高 |
| 像 model-gateway 一样再签发临时令牌 | 与先例一致；但 Git 访问由持证组件自己执行，证书已足够（D3） |

## 本决策未解决的问题

- 仓库的配额、GC，以及失败尝试留下的 ref 与对象、删除 Project 后的仓库清理。
- 用户或浏览器读取仓库内容（diff 预览、下载）的入口与授权。
- 生产部署的多副本、备份与存储后端。

## 落地顺序

本文与现有 Git Server 实现的程序形态、协议、路径和沙盒认证一致，不单独安排落地步骤。Git Server 成为代码中心所需的改动
（规则集授权、服务仓库监听、两条开发线）按契约 ADR 的落地顺序进行；契约 ADR 的各步都必须满足本文的不变量。
