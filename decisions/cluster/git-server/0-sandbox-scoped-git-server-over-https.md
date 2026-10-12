---
status: proposed
---

# Git Server 以 HTTPS 接收沙盒推送，沙盒证书只认证身份，仓库与 ref 授权在请求时判定

Revision 不再以 bundle 形式上传到对象存储，而是像日常开发一样 `git push` 到一个独立的 Git Server。
Git Server 与 Sandbox Server 并列，是 cluster 中的独立服务，只提供 smart HTTP（HTTPS），不提供 SSH。
沙盒以 Sandbox Server 按代签发的专用 git-access 客户端证书做双向 TLS，证书只回答“这是哪个租户、哪个运行时、
哪一代沙盒”，不携带任何仓库或 ref 权限。Git Server 在处理每个请求时，用证书中的身份查找 Controller 登记的有效访问
绑定，判断能否读写请求路径中的仓库；推送在 `pre-receive` 中再按 ref 逐项判定。没有绑定、绑定已撤销、
沙盒已换代或 ref 不在绑定内，一律拒绝。

本 ADR 处于 `proposed`，是新建领域 `cluster/git-server` 的根决策。涉及 cluster 仓库的 Sandbox Server
（签发 git-access 客户端身份）与新增的 Git Server、desktop 的 Controller（登记绑定）与 Node（交付时推送），以及
Cloud（登记 Revision 前的核验）。本文只确定 Git Server 的服务边界、协议与访问控制；Revision 的交付语义、
登记与核验、恢复如何改为基于 Git，由 `cloud/revision`、`node/revision` 与 `controller-integration` 的后续
ADR 修改（见“对 Revision 链路的影响”），在它们获批之前，现有对象存储路径保持不变。本文不迁移已经登记在
对象存储中的 Revision，不设计与用户上游仓库（GitHub 等）的同步。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| [Cloud Revision 根决策](../../cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)：Revision bundle 与会话记录上传到 Cloud 持有的对象存储 | 修改：代码改动的存放位置改为 Git Server（D1）；上传授权、HEAD 校验与 `revisions` 表的改动留给后续 ADR |
| [Node Revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) D2：内部快照提交、`refs/ora/revisions/<runId>` | 继承，不变；D3/D4 的 bundle 与上传由后续 ADR 改为推送（D5） |
| Sandbox Server 每代签发 WSS 服务端证书，仅 `serverAuth` 用途 | 继承，不变 |
| [模型客户端身份](../sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) D1：按租户、Workspace、代次另签专用 `clientAuth` 身份，材料只作为 Docker 私有启动材料、由 entrypoint 存入管理 tmpfs；WSS 证书不改为双用途 | 继承同一模式：另签 git-access 用途的客户端身份（D2），与 model-access 身份互不通用 |
| [Controller 只拨出](../../controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)，不暴露监听 | 继承，不变：Git Server 不调用 Controller，由 Controller 主动登记绑定（D3） |
| Node 不持长期凭据；管理身份不交给工作负载 | 继承，不变（D5） |
| [clone 执行契约](../../node/repository/20260918-minimal-clone-execution-contract.md)与 Node Cloud 策略：只允许 HTTPS，禁止 SSH 与静态凭据 helper | 继承；增加“Node 自有证书引用”这一种凭据来源（D5） |
| [仓库凭据连续性](../../cloud/project/20260927-repository-credential-continuity.md) D4：不回退全局静态凭据 | 继承：Git Server 不接受口令、静态 token 或匿名访问 |
| [模型客户端身份](../sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) D2：model-gateway 接入默认服务网络与沙盒网络，以不同监听区分模型请求与授权申请 | 继承同一模式：Git Server 以两个监听分别服务沙盒与 Controller（D6）；对象存储是否保留取决于会话记录的去向（未决） |

## 决策

### D1：独立服务，只提供 smart HTTP

- Git Server 是 cluster 仓库中的独立进程，与 Sandbox Server 并列部署。它在持久卷上托管 bare 仓库，
  仓库路径为 `/{tenantId}/{projectId}.git`，同一 Project 的所有 Workspace 共用一个仓库。
- 对沙盒只开放 smart HTTP 的三个端点：`GET /{repo}/info/refs?service=...`、`POST /{repo}/git-upload-pack`、
  `POST /{repo}/git-receive-pack`；协议处理交给 `git http-backend`。dumb HTTP、其他路径与其他方法返回 `404`。
- 仓库按 `projectId` 命名，而不是按上游 URL：上游 URL 可以改变，也可能带有用户信息；`projectId` 稳定且已在
  Cloud 中与租户绑定，路径中的 `tenantId` 只用于与证书身份做一致性检查，不作为授权依据。

为什么选 HTTPS 而不是 SSH：

| 维度 | HTTPS | SSH |
| --- | --- | --- |
| 沙盒身份 | 沿用 Sandbox Server 已有的 X.509 按代签发流程，只多一种用途 | 需要另建 SSH CA 或公钥注册表 |
| 服务端身份 | 复用同一 CA；Node 已通过 `sslCAInfo` 引用 CA | 需管理 host key 并向每个 Node 分发 `known_hosts` |
| Node 策略 | Cloud 模式本来只允许 HTTPS，只需增加证书引用 | Cloud 模式禁止 SSH，需放开策略并在镜像中带 ssh 客户端 |
| 授权挂点 | HTTP handler 按路径与 `service` 区分仓库与读写 | 需解析 `exec` 命令串 |
| 部署 | 普通 HTTP 监听，Compose 与网关都现成 | 需额外 TCP 端口，L7 网关不可用 |

SSH 对人类开发者更顺手，但这里的客户端是 Node，不是人。

### D2：认证只认沙盒的 git-access 证书

- Sandbox Server 创建已获许可的沙盒时，为该租户、Workspace 与代次另签一张 git-access 客户端证书：`clientAuth`
  用途，URI 标明 git-access 用途以及 `(tenantId, workspaceId, generation)`。签发与交付方式与 model-access
  身份相同：只作为 Docker 私有启动材料，由 Node entrypoint 存入管理 tmpfs，进程配置只引用绝对路径，不进入
  effect 请求、journal 或结果。
- 面向沙盒的监听要求客户端证书链到签发 CA、带 `clientAuth`、在有效期内，且恰好含一个 git-access 用途的 URI。
  解析出的三元组 `(tenantId, workspaceId, generation)` 是本次请求的主体。
- WSS 服务端证书与 model-access 证书都不被接受；git-access 证书也不能用于模型代理。
- 没有其他认证方式：不接受 Basic、Bearer、匿名访问，也不接受管理身份（Controller 证书）访问仓库内容。
- 认证失败在 TLS 握手阶段终止，或返回不含细节的 `401`。
- 证书不表示任何仓库权限。它的有效期只限定身份能被出示多久，不限定权限多久。

为什么另签而不是复用 WSS 证书：已批准的模型客户端身份决策明确不把 WSS 服务端证书改为双用途，以免扩大
Controller 侧签发权限的用途；按用途分开签发，一种身份泄露不能直接换用到另一服务。代价是每个沙盒多一份启动
材料，但签发流程与存放位置都已存在。

### D3：授权在请求时判定，来源是 Controller 登记的访问绑定

Git Server 不从证书推断权限，而是对照内存中的**访问绑定**判定每个请求：

```text
binding {
  bindingId
  tenantId, workspaceId, runtimeGeneration   // 必须与证书主体完全相等
  projectId                                  // 决定可访问的仓库
  access: read | write
  refs: [{ name, mode: create | fast_forward }]  // 仅 write 使用
  executionId, controllerEpoch
  expiresAt
}
```

- **来源**：Controller 经 Git Server 的管理监听（双向 TLS，固定 Controller 证书，与 Sandbox Server 相同）
  登记、续期和撤销绑定。Controller 在派发需要 Git 访问的执行时登记，内容取自 Cloud 为该执行签发的派发许可：
  交付执行得到写绑定，只含交付输入中的 Revision ref；恢复与续作得到读绑定。执行结束、运行时换代、Workspace
  释放或控制 epoch 变化时 Controller 撤销；更高 `controllerEpoch` 的登记使同一执行的旧绑定失效。
- **有效期**：绑定带短 `expiresAt`，由 Controller 续期。Controller 失联时绑定自然过期，访问随之关闭。
- **只在内存**：Git Server 不持久化绑定。重启后没有任何绑定，所有请求被拒绝，直到 Controller 重新登记；
  Controller 按现有对账流程对仍在进行的执行重新登记。
- **每个请求的判定**：
  1. 从证书取主体；
  2. 从路径取 `(tenantId, projectId)`，`tenantId` 必须等于主体中的租户；
  3. 查找主体与 `projectId` 都匹配且未过期的绑定；
  4. `info/refs?service=git-upload-pack` 与 `git-upload-pack` 需要 `read` 或 `write` 绑定，
     `info/refs?service=git-receive-pack` 与 `git-receive-pack` 需要 `write` 绑定；
  5. 不满足时返回 `403`。仓库不存在与无权访问返回相同响应，不泄露仓库是否存在。

为什么授权不绑在证书上：证书 24 小时有效且无法及时吊销；签发方（Sandbox Server）也不知道仓库关系。仓库改绑、
凭据冻结、执行结束或沙盒换代都需要立即生效，只有请求时判定才做得到。

为什么由 Controller 推送绑定，而不是 Git Server 去查询：Controller 只拨出，Git Server 无法调用它；这与
Sandbox Server“由 Controller 转交 Cloud 短期许可”的模式一致。直接查询 Cloud 的取舍见替代方案。

旧代沙盒：绑定的键含 `runtimeGeneration`，第 N 代的证书在有效期内依然能完成握手，但匹配不到第 N+1 代的绑定；
Controller 在换代时撤销第 N 代的绑定后，第 N 代证书没有任何可用权限。

### D4：推送的 ref 级判定在 `pre-receive` 中执行

- HTTP handler 把命中的 `bindingId` 通过环境变量交给 `git http-backend`，`pre-receive` hook 据此**重新读取**
  绑定：此时绑定必须仍然有效，否则拒绝整次推送。一次推送至少有 `info/refs` 与 `git-receive-pack` 两个请求，
  在写入前再判定一次，堵住两次请求之间被撤销的窗口。
- 对每个 ref 更新：
  - ref 名必须在绑定的 `refs` 中；
  - `mode = create` 时旧值必须为零（只能新建）；`mode = fast_forward` 时新值必须是旧值的后代；
  - 不允许删除；
  - 任一 ref 不合规则拒绝整次推送，不部分写入。
- 仓库固定配置 `receive.denyDeletes=true`、`receive.denyNonFastForwards=true`、`receive.fsckObjects=true`，
  不接受 push options；hook 与仓库配置由 Git Server 管理，客户端无法修改。

本文只规定“绑定之外的 ref 一律不可写，且不能删除或强推”，不规定 ref 的命名。第一版交付沿用
`refs/ora/revisions/<runId>` 与 `create` 模式；是否像日常开发一样推送 `refs/heads/...` 分支（使用
`fast_forward` 模式），由 Revision 后续 ADR 决定，不需要修改本文。

### D5：推送由 Node 执行，证书私钥不进入工作负载

- 交付时由 Node 进程执行 `git push`，git-access 证书与私钥来自管理 tmpfs 中的文件，只在当次命令中以
  `-c http.sslCert=... -c http.sslKey=... -c http.sslCAInfo=...` 传入；不写入 checkout 的 `.git/config`、
  全局 Git 配置、日志或 journal。推送按 URL 进行，checkout 中不新增 remote。
- Agent 在会话中自己执行 `git push` 没有凭据，会失败；这是预期行为，成果只经交付执行进入 Git Server。
- Node 的 Cloud 策略继续禁止静态凭据 helper 与 SSH；新增的只有 git-access 证书这一种来源。

已知缺口：只要工作负载能读到 git-access 私钥（模型客户端身份决策同样记录了既有 root 插件进程与 guardian
纳管缺口），它就能以本沙盒身份访问 Git
Server。此时的最坏结果被 D3、D4 限定在本沙盒当前有效的绑定内：只能读本 Project 的仓库，只能新建或快进绑定
列出的 ref。这一缺口应在交付证据中如实记录，由工作负载降权解决，不由 Git Server 解决。

### D6：两个监听与网络位置

- **沙盒监听**：只接入沙盒网络，只提供 D1 的 smart HTTP，只接受 D2 的沙盒证书。
- **管理监听**：只接入 Compose 默认网络，只接受固定的 Controller 证书（以及后续 ADR 确定的 Cloud 核验身份），
  提供绑定的登记、续期、撤销，以及按仓库查询 ref 当前值的只读接口，供 Cloud 登记 Revision 前核验。
- 两个监听分开，沙盒无法访问管理接口，管理身份也无法通过沙盒监听读写仓库。
- 仓库数据放在持久卷；Compose 不向宿主发布端口。

## 对 Revision 链路的影响

以下修改由后续 ADR 承担，本文只指出方向：

| 现有决策 | 需要的修改 |
| --- | --- |
| [Node Revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) D3/D4 | bundle 生成与上传改为推送最终提交到 Revision ref |
| [Cloud Revision 根决策](../../cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md) D1–D4 | 上传授权改为访问绑定；对象 HEAD 校验改为经管理监听核验 ref 指向声明的最终提交 |
| [Agent 运行执行与上传授权](../../cloud/controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md) | `GrantRevisionUpload` 改为 Controller 登记写绑定 |
| [恢复前序 Revision](../../node/revision/20261010-restore-prior-revision-before-session.md) | 下载授权改为读绑定与 `git fetch` |

推送不再需要“基础提交需从用户远端取得”这一恢复前提：Git Server 中已有完整对象。

## 不变量

1. Git Server 的沙盒监听只接受 git-access 用途的客户端证书，不接受 WSS 或 model-access 证书，也没有其他认证方式。
2. 证书不携带授权；不存在与证书主体及仓库都匹配的有效绑定时，读和写都被拒绝。
3. 绑定只来自 Controller 的管理接口且只保存在内存；Git Server 重启后，在 Controller 重新登记之前拒绝所有仓库请求。
4. 推送只能更新绑定列出的 ref，不能删除或强推；任一 ref 不合规则整次推送不写入。
5. 在 `info/refs` 之后、`git-receive-pack` 写入之前被撤销或过期的绑定，使该次推送被拒绝。
6. 旧代沙盒的证书无法使用新代的绑定。
7. 不存在的仓库与无权访问的仓库返回相同响应。
8. 沙盒证书私钥与绑定内容不写入 checkout、Git 配置、日志或 journal。
9. 沙盒无法访问管理监听，管理身份无法读写仓库内容。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| SSH 协议 | 对人更熟悉；但需要另一套 CA 或公钥表、host key 分发，并放开 Node 的 Cloud 策略（D1） |
| 复用 WSS 服务端证书，增加 `clientAuth` 用途 | 少签一张证书；但违反已批准的模型客户端身份决策，且一份私钥同时可用于被 Controller 连接与访问仓库 |
| 把仓库权限写进证书（如 SAN 中写仓库） | Git Server 无需状态；但 24 小时内无法撤销，签发方需知道仓库关系，改绑需重签，违反不变量 2、5、6 |
| Controller 签发短期 bearer token 交给 Node | Git Server 可以无状态验签；但撤销同样要等过期，token 也会被能读 Node 环境的工作负载转用，且是证书之外的第二种身份 |
| Git Server 每次请求查询 Cloud | Cloud 是权威，撤销最及时，这一点优于本方案；但需新增 Cloud 对 Git Server 的客户端信任，Cloud 不可用时所有 Git 操作失败，也偏离 Sandbox Server 由 Controller 转交许可的模式。本方案以短有效期与显式撤销弥补及时性 |
| Git Server 查询 Controller | 违反 Controller 只拨出 |
| 部署 Gitea / GitLab 等现成服务 | 自带界面与镜像功能；但其授权以用户与 token 为单位，无法按运行时代次和单个 ref 授权，要把沙盒映射成用户，运维成本也更高 |
| 继续使用对象存储 | 已实现；但成果不能像日常开发一样 fetch 与比较，恢复依赖从用户远端取基础提交 |

## 本决策未解决的问题

- 会话 JSONL 的存放位置：继续使用对象存储，还是另存一个 ref。决定前对象存储不能下线。
- 与用户上游仓库的关系：Git Server 中的仓库初始为空时，首次推送会携带完整历史；是否从上游镜像、Node 是否改从
  Git Server clone、是否推回用户远端，涉及仓库凭据连续性，另立 ADR。
- 仓库的配额、GC、删除 Project 时的清理。
- 用户或浏览器读取仓库内容（diff 预览、下载）的入口与授权。
- 生产部署的多副本与存储后端。

## 落地顺序

1. **Sandbox Server 签发 git-access 客户端身份**，沿用 model-access 的签发与存放方式。完成条件：Node 在管理
   tmpfs 中拿到证书，工作负载读不到；WSS 与 model-access 身份不受影响。行为不变。
2. **Git Server 服务本体**：双监听、证书解析、内存绑定、smart HTTP 与 `pre-receive`。完成条件：用真实 `git`
   客户端的集成测试证明不变量 1–7、9：无绑定拒绝、跨 Project 拒绝、绑定外 ref 拒绝、删除与强推拒绝、两次请求间
   撤销拒绝、旧代证书拒绝、重启后拒绝。此阶段没有调用方，Revision 仍走对象存储。
3. **Compose 接入**：Git Server 加入 cluster 栈并接入沙盒网络。行为不变。
4. **Controller 登记绑定、Node 交付时推送、Cloud 核验**：在 Revision 后续 ADR 获批后实施。从这一阶段开始，
   新交付的 Revision 存放在 Git Server。

先交付并独立验收访问控制，再改动 Revision 链路，使安全边界可以在没有业务调用方的情况下单独评审；任何一步失败
都不影响已有的对象存储路径。
