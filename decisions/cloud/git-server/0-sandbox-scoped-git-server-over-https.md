---
status: proposed
---

# Git Server 以 HTTPS 接收沙盒推送，沙盒证书只认证身份，授权按 Cloud 当前状态逐请求判定

Revision 不再以 bundle 形式上传到对象存储，而是像日常开发一样 `git push` 到一个独立的 Git Server。
Git Server 是 Cloud 仓库中与 model-gateway 并列的独立程序，只提供 smart HTTP（HTTPS），不提供 SSH。
沙盒以 Sandbox Server 按代签发的专用 git-access 客户端证书做双向 TLS，证书只回答“这是哪个租户、哪个运行时、
哪一代沙盒”，不携带任何仓库或 ref 权限。每个请求声明自己服务的执行，Git Server 直接读取 Cloud 数据库，
按证书目标与当前数据库状态判定能否读写该仓库；推送在 `pre-receive` 中再按 ref 逐项判定。Git Server 不保存
任何授权状态，Controller 不参与授权。

本 ADR 处于 `proposed`，是新建领域 `cloud/git-server` 的根决策。涉及 Cloud 仓库新增的 Git Server 程序、
cluster 仓库的 Sandbox Server（签发 git-access 客户端身份）与 Compose 部署、desktop 的 Node（交付时推送、
恢复时拉取）。本文只确定 Git Server 的服务边界、协议与访问控制；Revision 的交付语义、登记与核验、恢复如何
改为基于 Git，由 `cloud/revision`、`node/revision` 与 `controller-integration` 的后续 ADR 修改（见“对
Revision 链路的影响”），在它们获批之前，现有对象存储路径保持不变。本文不迁移已经登记在对象存储中的
Revision，不设计与用户上游仓库（GitHub 等）的同步。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [Cloud Revision 根决策](../revision/0-cloud-owned-object-store-and-verified-revisions.md)：Revision bundle 与会话记录上传到 Cloud 持有的对象存储 | 修改：代码改动的存放位置改为 Git Server（D1）；上传授权、HEAD 校验与 `revisions` 表的改动留给后续 ADR |
| [Node Revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) D2：内部快照提交、`refs/ora/revisions/<runId>` | 继承本地快照语义；D3/D4 的 bundle 与上传由后续 ADR 改为推送（D5），远端 ref 命名受 D4 约束 |
| [Cloud 决策目录](../README.md)：Cloud 是多租户业务状态的权威，Controller 与 Node 不建立另一份业务权威 | 继承：授权只依据 Cloud 数据库的当前状态（D3），Git Server 不保存授权副本 |
| [个人模型访问](../model-access/0-personal-model-connections-and-run-scoped-proxy.md) D2、D4：model-gateway 是 Cloud 仓库中的独立程序，直接读 Cloud 数据库；授权由证书目标与当前数据库状态共同决定，请求声明执行 ID | 继承同一模式（D1、D3）；不继承临时令牌，理由见 D3 |
| Sandbox Server 每代签发 WSS 服务端证书，仅 `serverAuth` 用途 | 继承，不变 |
| [模型客户端身份](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) D1：按租户、Workspace、代次另签专用 `clientAuth` 身份，材料只作为 Docker 私有启动材料、由 entrypoint 存入管理 tmpfs；WSS 证书不改为双用途 | 继承同一模式：另签 git-access 用途的客户端身份（D2），与 model-access 身份互不通用 |
| [模型客户端身份](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) D2：model-gateway 接入默认服务网络与沙盒网络 | 继承同一模式：Git Server 的仓库监听只面向沙盒网络（D6）；对象存储是否保留取决于会话记录的去向（未决） |
| [Controller 只拨出](../../controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)，不暴露监听 | 继承，不变：Controller 只负责派发执行和转达结果，不向 Git Server 登记或撤销任何授权 |
| Node 不持长期凭据；管理身份不交给工作负载 | 继承，不变（D5） |
| [clone 执行契约](../../node/repository/20260918-minimal-clone-execution-contract.md)与 Node Cloud 策略：只允许 HTTPS，禁止 SSH 与静态凭据 helper | 继承；增加“git-access 证书引用”这一种凭据来源（D5） |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md) D4：不回退全局静态凭据 | 继承：Git Server 不接受口令、静态 token 或匿名访问 |

## 决策

### D1：Cloud 仓库中的独立程序，只提供 smart HTTP

- Git Server 是 Cloud 仓库中的独立程序（与 `cmd/model-gateway` 并列），单独部署。它在持久卷上托管 bare 仓库，
  仓库路径为 `/{tenantId}/{projectId}.git`，同一 Project 的所有 Workspace 共用一个仓库。
- 对沙盒只开放 smart HTTP 的三个端点：`GET /{repo}/info/refs?service=...`、`POST /{repo}/git-upload-pack`、
  `POST /{repo}/git-receive-pack`；协议处理交给 `git http-backend`。dumb HTTP、其他路径与其他方法返回 `404`。
- 仓库按 `projectId` 命名，而不是按上游 URL：上游 URL 可以改变，也可能带有用户信息；`projectId` 稳定且已在
  Cloud 中与租户绑定。路径中的 `tenantId` 只用于与证书身份做一致性检查，不作为授权依据。
- 仓库在第一次获得授权的写请求时由 Git Server 创建；读请求不创建仓库。

为什么放在 Cloud 仓库：授权依据全部是 Cloud 数据库中的状态，数据库 schema 由 Cloud 拥有。把读取这些状态的
查询放在 Cloud 仓库，schema 变化与查询在同一仓库内一起评审和测试，不形成跨仓库共享 schema；这也是
model-gateway 已采用的结构。Compose 部署仍在 cluster 仓库。

为什么选 HTTPS 而不是 SSH：

| 维度 | HTTPS | SSH |
| --- | --- | --- |
| 沙盒身份 | 沿用 Sandbox Server 已有的 X.509 按代签发流程，只多一种用途 | 需要另建 SSH CA 或公钥注册表 |
| 服务端身份 | 复用同一 CA；Node 已通过 `sslCAInfo` 引用 CA | 需管理 host key 并向每个 Node 分发 `known_hosts` |
| Node 策略 | Cloud 模式本来只允许 HTTPS，只需增加证书引用 | Cloud 模式禁止 SSH，需放开策略并在镜像中带 ssh 客户端 |
| 授权挂点 | HTTP handler 按路径、`service` 与请求头区分仓库、读写与执行 | 需解析 `exec` 命令串，执行 ID 需另找通道传递 |
| 部署 | 普通 HTTP 监听，Compose 与网关都现成 | 需额外 TCP 端口，L7 网关不可用 |

SSH 对人类开发者更顺手，但这里的客户端是 Node，不是人。

### D2：认证只认沙盒的 git-access 证书

- Node 配置启用 Git Server 时，Sandbox Server 创建已获许可的沙盒，为该租户、Workspace 与代次另签一张
  git-access 客户端证书：`clientAuth` 用途，URI 为
  `spiffe://ora.local/git-access/tenant/{tenantId}/runtime/{workspaceId}/generation/{generation}`。签发与交付方式
  与 model-access 身份相同：只作为 Docker 私有启动材料，由 Node entrypoint 存入管理 tmpfs，进程配置只引用
  绝对路径，不进入 effect 请求、journal 或结果。未启用时不签发。
- 仓库监听要求客户端证书链到签发 CA、带 `clientAuth`、在有效期内，且恰好含一个 git-access 用途的 URI。
  解析出的三元组 `(tenantId, workspaceId, generation)` 是本次请求的认证目标。
- WSS 服务端证书与 model-access 证书都不被接受；git-access 证书也不能用于模型代理。
- 没有其他认证方式：不接受 Basic、Bearer、匿名访问，也不接受管理身份访问仓库内容。
- 认证失败在 TLS 握手阶段终止，或返回不含细节的 `401`。
- 证书不表示任何仓库权限。它的有效期只限定身份能被出示多久，不限定权限多久。

为什么另签而不是复用 WSS 证书：已批准的模型客户端身份决策明确不把 WSS 服务端证书改为双用途；按用途分开签发，
一种身份泄露不能直接换用到另一服务。代价是每个沙盒多一份启动材料，但签发流程与存放位置都已存在。

### D3：每个请求按证书目标、声明的执行与 Cloud 当前状态判定

每个仓库请求都带 `Ora-Execution-Id` 请求头，声明它服务的执行；证书决定“是谁”，请求头只说明“为哪个执行”，
不能自行声明认证目标。Git Server 在不持有事务的只读查询中，依据 Cloud 数据库的当前状态判定：

1. 证书目标的 `(tenantId, workspaceId, generation)` 是该 Workspace 当前存活的沙盒代次；旧代或已终止的沙盒拒绝。
2. 声明的执行属于该 Workspace 的当前 Node/沙盒，尚无结果，且未被强停终止。
3. 执行类型决定权限：
   - 交付执行：可读写；可写的 ref 只来自该执行的固定输入（D4）；
   - 恢复或续作所在的会话执行：只读；
   - 其他执行：拒绝。
4. 请求路径中的 `tenantId` 等于证书目标的租户，`projectId` 等于该 Workspace 所属的 Project。
5. 执行所在运行仍处于允许该操作的阶段，触发成员与账号仍然有效；具体阶段条件由 Revision 后续 ADR 列出。
6. `info/refs?service=git-upload-pack` 与 `git-upload-pack` 需要读权限，
   `info/refs?service=git-receive-pack` 与 `git-receive-pack` 需要写权限。

结果：

| 情况 | 响应 |
| --- | --- |
| 判定通过 | 交给 `git http-backend` |
| 任一条件不满足 | `403`，不含细节；仓库不存在与无权访问返回相同响应 |
| 缺少或格式错误的 `Ora-Execution-Id` | `400` |
| 数据库不可用或查询超时 | `503`，不放行；Node 可以按执行的重试规则重试 |

为什么授权不绑在证书上：证书 24 小时有效且无法及时吊销；签发方（Sandbox Server）也不知道仓库关系。执行结束、
沙盒换代、成员离开或运行取消都需要立即生效，只有逐请求读取当前状态才做得到。

为什么直接读 Cloud 数据库：判定所需的事实（当前代次、执行状态、Workspace 与 Project 的关系、成员状态）都只在
Cloud。Git Server 不保存任何授权副本，重启不丢失任何授权，状态变化在下一个请求生效。

为什么不像 model-gateway 那样再发临时令牌：model-gateway 需要令牌，是因为实际调用模型的是 OpenCode 这一工作负载，
而工作负载不能拿到证书私钥，只能拿到短期令牌。Git 访问由 Node 自己执行（D5），证书已能证明身份，再加一层令牌
只多一份需要续期和撤销的凭据。

### D4：推送的 ref 级判定在 `pre-receive` 中执行

- HTTP handler 把认证目标与执行 ID 通过环境变量交给 `git http-backend`，`pre-receive` hook 在写入前**再次**按 D3
  判定：一次推送至少有 `info/refs` 与 `git-receive-pack` 两个请求，且对象传输可能很长，写入前再判定一次，堵住
  其间执行结束或沙盒换代的窗口。
- 对每个 ref 更新：
  - ref 名必须是交付执行固定输入给出的 ref；
  - 不允许删除，不允许非快进覆盖，除非后续 ADR 为“同一运行较新的交付尝试替换较早尝试”给出显式规则；
  - 任一 ref 不合规则拒绝整次推送，不部分写入。
- 仓库固定配置 `receive.denyDeletes=true`、`receive.fsckObjects=true`，不接受 push options；hook 与仓库配置由
  Git Server 管理，客户端无法修改。
- Git 对象传输与 hook 执行都不在数据库事务或锁中进行；每次判定是独立的短只读查询。

ref 命名与重试的约束：当前 Cloud 为每个运行固定 `refs/ora/revisions/<runId>`，交付重试沿用同一名字；而内部快照
提交带有时间戳，重试会得到不同的提交。若远端仍用每运行一个 ref 且只允许新建，“首次推送已成功但结果未送达
Cloud”之后的重试将永远失败。Revision 后续 ADR 必须二选一：远端 ref 按交付尝试区分（例如包含交付工作项 ID），
或明确允许同一运行较新的尝试替换较早尝试。本文只规定“可写 ref 来自执行输入，且不能删除”，不规定命名。

### D5：Node 执行 Git 访问，证书私钥不进入工作负载

- 交付时由 Node 进程执行 `git push`，恢复时由 Node 执行 `git fetch`。git-access 证书与私钥来自管理 tmpfs 中的
  文件，只在当次命令中以 `-c http.sslCert=... -c http.sslKey=... -c http.sslCAInfo=...` 传入，执行 ID 以
  `-c http.extraHeader=Ora-Execution-Id: ...` 传入；不写入 checkout 的 `.git/config`、全局 Git 配置、日志或
  journal。按 URL 访问，checkout 中不新增 remote。
- Git Server 的地址由 Controller 派发执行时随执行输入给出，与现有派发方式一致；Controller 不为此获得额外权限。
- Agent 在会话中自己执行 `git push` 没有凭据，会失败；这是预期行为，成果只经交付执行进入 Git Server。
- Node 的 Cloud 策略继续禁止静态凭据 helper 与 SSH；新增的只有 git-access 证书这一种来源。

已知缺口：只要工作负载能读到 git-access 私钥（模型客户端身份决策同样记录了既有 root 插件进程与 guardian 纳管
缺口），它就能以本沙盒身份访问 Git Server。此时的最坏结果被 D3、D4 限定在本沙盒当前进行中的执行内：只能读本
Project 的仓库，只能写该交付执行输入给出的 ref。这一缺口应在交付证据中如实记录，由工作负载降权解决，不由
Git Server 解决。

### D6：监听与网络位置

- **仓库监听**：只接入沙盒网络，只提供 D1 的 smart HTTP，只接受 D2 的 git-access 证书。
- Git Server 另需访问 Cloud 数据库，因此接入 Compose 默认网络；默认网络上不提供仓库访问。
- Cloud 登记 Revision 前如何核验推送结果（经 Git Server 的只读查询，或由 Git Server 在推送成功后把收据写入数据库），
  由 Cloud Revision 后续 ADR 决定；无论哪种，都不向沙盒网络开放，也不经 Controller。
- 仓库数据放在持久卷；Compose 不向宿主发布端口。

## 对 Revision 链路的影响

以下修改由后续 ADR 承担，本文只指出方向：

| 现有决策 | 需要的修改 |
| --- | --- |
| [Node Revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) D3/D4 | bundle 生成与上传改为推送最终提交；远端 ref 命名满足 D4 的重试约束 |
| [Cloud Revision 根决策](../revision/0-cloud-owned-object-store-and-verified-revisions.md) D1–D4 | 去掉上传授权；对象 HEAD 校验改为核验 Git Server 上的 ref 指向声明的最终提交（D6） |
| [Agent 运行执行与上传授权](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md) | 不再需要 `GrantRevisionUpload`；交付输入增加 Git Server 地址，授权由执行状态本身决定 |
| [恢复前序 Revision](../../node/revision/20261010-restore-prior-revision-before-session.md) | 下载授权改为恢复执行凭证书与执行 ID `git fetch` |

推送不再需要“基础提交需从用户远端取得”这一恢复前提：Git Server 中已有完整对象。

## 不变量

1. Git Server 的仓库监听只接受 git-access 用途的客户端证书，不接受 WSS 或 model-access 证书，也没有其他认证方式。
2. 证书不携带授权；每个仓库请求都按证书目标、声明的执行与 Cloud 数据库当前状态判定，任一条件不满足即拒绝。
3. Git Server 不保存授权状态；它的重启不改变任何判定结果。
4. 无法读取 Cloud 当前状态时，Git Server 拒绝请求，不以任何缓存或证书有效性放行。
5. 推送只能更新交付执行输入给出的 ref，不能删除；任一 ref 不合规则整次推送不写入。
6. 在 `info/refs` 之后、`git-receive-pack` 写入之前执行结束或沙盒换代，该次推送被拒绝。
7. 旧代沙盒的证书不能访问任何仓库。
8. 不存在的仓库与无权访问的仓库返回相同响应。
9. git-access 私钥不写入 checkout、Git 配置、日志或 journal。
10. Git 对象传输与 hook 执行不在数据库事务或锁中进行。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| SSH 协议 | 对人更熟悉；但需要另一套 CA 或公钥表、host key 分发，并放开 Node 的 Cloud 策略（D1） |
| 复用 WSS 服务端证书，增加 `clientAuth` 用途 | 少签一张证书；但违反已批准的模型客户端身份决策，且一份私钥同时可用于被 Controller 连接与访问仓库 |
| 把仓库权限写进证书（如 SAN 中写仓库） | Git Server 无需查询；但 24 小时内无法撤销，签发方需知道仓库关系，改绑需重签，违反不变量 2、6、7 |
| Controller 向 Git Server 推送访问绑定 | Git Server 不依赖 Cloud 数据库；但绑定是 Cloud 状态的副本，撤销要等 Controller 察觉并发出或等绑定过期，需新增登记、续期、撤销协议，Git Server 重启后要等重新登记，违反不变量 3 |
| Git Server 查询 Controller | 违反 Controller 只拨出 |
| Git Server 留在 cluster 仓库，调用 Cloud 新增的内部鉴权接口 | 服务按部署层次划分更整齐；但要多设计一个接口和一套调用身份，且偏离 model-gateway 已采用的结构（D1） |
| 像 model-gateway 一样再签发临时令牌 | 与先例完全一致；但 Git 访问由 Node 自己执行，证书已足够，令牌只多一份需续期与撤销的凭据（D3） |
| 部署 Gitea / GitLab 等现成服务 | 自带界面与镜像功能；但其授权以用户与 token 为单位，无法按运行时代次和执行授权，要把沙盒映射成用户，运维成本也更高 |
| 继续使用对象存储 | 已实现；但成果不能像日常开发一样 fetch 与比较，恢复依赖从用户远端取基础提交 |

## 本决策未解决的问题

- 会话 JSONL 的存放位置：继续使用对象存储，还是另存一个 ref。决定前对象存储不能下线。
- 远端 Revision ref 的命名与重试替换规则（D4 的约束由 Revision 后续 ADR 落实）。
- Cloud 核验推送结果的方式（D6）。
- 与用户上游仓库的关系：Git Server 中的仓库初始为空时，首次推送会携带完整历史；是否从上游镜像、Node 是否改从
  Git Server clone、是否推回用户远端，涉及仓库凭据连续性，另立 ADR。
- 仓库的配额、GC、删除 Project 时的清理。
- 用户或浏览器读取仓库内容（diff 预览、下载）的入口与授权。
- 生产部署的多副本与存储后端。

## 落地顺序

1. **Sandbox Server 签发 git-access 客户端身份**，沿用 model-access 的签发与存放方式。完成条件：启用时 Node 在管理
   tmpfs 中拿到证书，工作负载读不到，未启用时不签发；WSS 与 model-access 身份不受影响。行为不变。
2. **Git Server 程序本体**：证书解析、逐请求的数据库判定、smart HTTP、`pre-receive` 与按需建仓。完成条件：用真实
   `git` 客户端与真实 PostgreSQL 的集成测试证明不变量 1–8、10：缺执行 ID、执行属于别的沙盒、执行已结束、旧代证书、
   跨 Project、写执行输入之外的 ref、删除、两次请求之间执行结束、数据库不可用，均被拒绝；Git Server 重启前后判定
   一致。此阶段没有调用方，Revision 仍走对象存储。
3. **Compose 接入**：Git Server 加入 cluster 栈，接入沙盒网络与默认网络。行为不变。
4. **Node 交付时推送、恢复时拉取，Cloud 核验并登记**：在 Revision 后续 ADR 获批后实施。从这一阶段开始，新交付的
   Revision 存放在 Git Server。

先交付并独立验收访问控制，再改动 Revision 链路，使安全边界可以在没有业务调用方的情况下单独评审；任何一步失败都不
影响已有的对象存储路径。
