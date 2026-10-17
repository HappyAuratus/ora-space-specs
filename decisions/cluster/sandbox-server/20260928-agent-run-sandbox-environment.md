---
status: implemented
---

# 沙盒为 Agent 运行提供部署环境变量、出网与本地对象存储的可达地址

> 历史决策：模型凭据经环境变量文件注入的路线已被
> [模型客户端身份决策](20261010-model-client-identity-replaces-shared-model-key-injection.md)取代，个人模型 Key
> 保留在平台代理，env 文件不再是模型认证入口；该决策也让 model-gateway 接入沙盒网络。
> [Controller 接入沙盒网络并以双向 TLS 直连 Node](20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md)
> 让 Controller 接入沙盒网络。因此 D3 的“RustFS 是除 Sandbox Server 外唯一接入沙盒网络的服务”与不变量 3
> 中关于 Controller 的部分已被修改；其余规则不变。

为了在本地跑通 Agent 运行，Sandbox Server 创建沙盒容器时可以读入一份由部署提供的环境变量文件，把其中的
变量原样注入 Node 容器（出网代理、Agent CLI 的配置等）；沙盒网络保持可出网，供插件下载与模型调用；cluster
Compose 提供 S3 兼容对象存储（RustFS）作为 Cloud 的对象存储，并同时接入沙盒网络，使 Node 能用 Cloud 签发的
预签名 URL 直接上传。

本 ADR 处于 `implemented`，继承 [Docker 沙盒生命周期决策](20260926-docker-sandbox-lifecycle-via-substrate-effects.md)
与其根决策 [Docker 标签解析](0-docker-label-resolver.md)。它落实
[Node agent-runtime 根决策](../../node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D5 的“沙盒环境
由部署提供”，以及 [Cloud Revision 根决策](../../cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)
D1 的 `public_endpoint`。实现位于 cluster 的 `sandbox-server/internal/lifecycle/nodeenv.go`、`compose.yaml`、
`compose/node.env.example`、`compose/objectstore-credentials.sh` 与 `compose/create-bucket.sh`，验证见
[沙盒部署环境核心用例](../../../test-cases/cluster/sandbox-server/agent-run-sandbox-environment.md)。只适用于本地
联调，不代表生产 Substrate 的凭据或网络方案。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 生命周期 D1：只实现 `sandbox_ensure`、`sandbox_terminate`、`workspace_data_delete`，其他 kind `400` | 继承，不变；插件由 Node 执行，Sandbox Server 不实现插件 effect |
| 生命周期 D4：配置经 `ORA_NODE_CONFIG`，配置不含凭据 | 修改：增加 `-node-env-file`，其中变量原样注入容器环境（D1）；`ORA_NODE_CONFIG` 仍不含凭据 |
| 根 D2：容器只接入沙盒网络、不发布端口 | 继承，不变 |

## 决策

### D1：`-node-env-file` 注入部署提供的环境变量

- Sandbox Server 启动参数 `-node-env-file <path>`（可选，只能与 `-lifecycle-addr` 一起使用）指向 `KEY=VALUE`
  文件；创建容器时把其中每个变量追加到容器环境，排在 `ORA_NODE_CONFIG` 与管理 TLS 变量之后。空行与 `#` 开头
  的行忽略；值是第一个 `=` 之后的全部内容，按字面使用，不处理引号、不展开变量。
- 变量名必须匹配 `^[A-Z][A-Z0-9_]*$`，且不得以 `ORA_` 开头（`ORA_` 前缀留给 Sandbox Server 自己设置的配置与
  身份材料，部署文件因此不能替换 Node 的配置或身份）；同名变量出现两次、或只写变量名（Docker env-file 中
  “从宿主环境复制”的写法）同样拒绝。任一不合规、文件缺失都使启动失败；错误信息只含行号与变量名，不含值。
- 变量值不写入 journal、日志或 effect 结果。
- 文件内容在 Sandbox Server 启动时读取一次；修改后需要重启 Sandbox Server，已运行的沙盒不受影响。
- cluster 提供 `compose/node.env.example`，真实的 `compose/node.env` 被 Git 忽略。Compose 始终以只读方式挂载
  该文件（`ORA_NODE_ENV_FILE` 可改路径）；`task up` 在文件不存在时从示例复制一份（权限 `600`），示例中的变量均已注释，因此
  默认不注入任何变量。
- Node 不过滤环境：注入的变量经 ora-node 传给插件进程与它启动的 Agent CLI，沙盒内能执行代码的主体都能读到
  这些值。Agent 以工作负载 UID 1000 运行（Node 配置模板的 `agent.workload_directory` 与
  `process.workload_uid`），读不到 Node home、管理密钥与账本。
- 典型用途：宿主只能经代理出网时的 `HTTPS_PROXY`/`NO_PROXY`（`NO_PROXY` 需包含对象存储在沙盒网络上的别名，
  交付上传才不经代理），以及 Agent CLI 自身的配置（如 `OPENCODE_CONFIG_CONTENT`）。

为什么放在 Sandbox Server 而不是 Node 镜像：环境不应进入镜像层；Sandbox Server 已经是本地“平台”，由它注入与
平台向沙盒注入配置的位置一致，Cloud 与 Controller 都不经手。

实际决策与偏离：原提案把 env 文件定位为向所有沙盒注入部署共享模型 Key 的入口。M4 改为个人模型配置加平台
代理（模型客户端身份决策），共享 Key 不再是推荐路线；env 文件机制按原规则保留，承载代理与 Agent CLI 配置。

### D2：沙盒网络可出网

沙盒网络（默认 `ora-sandboxes`，由 `task up` 以普通 `docker network create` 创建并在 `down` 后保留）是非
internal 的 bridge 网络，沙盒可以访问插件 release 地址与模型 API。不做出网白名单。

### D3：RustFS 同时接入 Compose 默认网络与沙盒网络

- Compose 提供 `objectstore` 服务（`rustfs/rustfs`）、一次性凭据生成任务与一次性建桶任务；凭据与对象数据都放在
  持久卷，重复启动不重置凭据，重复建桶不清空已有对象。Cloud 的 `object_store.endpoint` 为 Compose 网络内
  地址，`public_endpoint` 为 RustFS 在沙盒网络上的别名地址（默认 `ora-revisions`，`ORA_OBJECT_STORE_HOST` 可改），
  预签名 URL 使用后者。
- 存储管理员凭据只挂载给建桶任务与 Cloud；Node 不持有凭据，只使用 Cloud 签发的预签名 URL。
- Cloud 不接入沙盒网络；Node 在对象存储上能访问的只有它的 S3 端口。

选择 RustFS 而不是 MinIO 是部署选择；决策只依赖 S3 兼容接口（预签名 PUT、HEAD、path-style 寻址），换用其他
S3 兼容服务不改变本决策的网络与凭据边界。`latest` 标签会变化，验收时记录实际镜像摘要以便追溯。

## 不变量

1. 注入的环境变量不出现在 journal、日志与 effect 结果中。
2. 部署文件不能设置 `ORA_` 前缀的变量，不能替换 Node 配置或身份材料。
3. 沙盒容器仍只接入沙盒网络、不发布端口。
4. Cloud 不接入沙盒网络；对象存储凭据不进入沙盒。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 环境变量放进 `ORA_NODE_CONFIG` | 少一个参数；Node 配置被 journal 与日志视为非敏感内容，混入敏感值会泄露 |
| 对象存储只在 Compose 默认网络，Cloud 代理上传 | 沙盒网络更封闭；但 Cloud 要中转大文件，违背 Revision 直传的决策 |
| 沙盒网络设为 internal、经代理出网 | 更接近生产隔离；本地联调没有需要，另作安全决策 |

## 落地

Sandbox Server 的 `-node-env-file` 与注入由 `nodeenv.go` 实现；Compose 提供 RustFS、凭据与建桶任务、env 示例与
Cloud 对象存储配置。保持不变：生命周期 effect 种类、容器网络与端口约束。开始改变：部署可以向新建沙盒追加
环境变量，Node 可以直接上传 Revision 对象。
