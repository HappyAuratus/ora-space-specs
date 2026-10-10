---
status: proposed
---

# 沙盒为 Agent 运行提供模型凭据环境变量、出网与本地对象存储的可达地址

为了在本地跑通 Agent 运行，Sandbox Server 创建沙盒容器时增加一份由部署提供的环境变量文件，
把其中的变量原样注入 Node 容器（用于 Agent 调用模型的凭据，例如 `ANTHROPIC_API_KEY`）；沙盒网络保持
可出网，供插件下载与模型调用；cluster Compose 增加 S3 兼容对象存储（当前按用户指定使用 RustFS）作为
Cloud 的对象存储，并同时接入沙盒网络，使 Node 能用 Cloud 签发的预签名 URL 直接上传。

本 ADR 处于 `proposed`，继承 [Docker 沙盒生命周期决策](20260926-docker-sandbox-lifecycle-via-substrate-effects.md)
与其根决策 [Docker 标签解析](0-docker-label-resolver.md)。它落实
[Node agent-runtime 根决策](../../node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D5 的
“模型凭据由沙盒部署提供”，以及 [Cloud Revision 根决策](../../cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)
D1 的 `public_endpoint`。只适用于本地联调，不代表生产 Substrate 的凭据或网络方案。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 生命周期 D1：只实现 `sandbox_ensure`、`sandbox_terminate`、`workspace_data_delete`，其他 kind `400` | 继承，不变；插件改由 Node 执行，Sandbox Server 不实现插件 effect |
| 生命周期 D6 与 Node 镜像约定：配置只经 `ORA_NODE_CONFIG`，配置不含凭据 | 修改：增加 `-node-env-file`，其中变量原样注入容器环境（D1）；`ORA_NODE_CONFIG` 仍不含凭据 |
| 根 D2：容器只接入沙盒网络、不发布端口 | 继承，不变 |

## 决策

### D1：`-node-env-file` 注入部署提供的环境变量

- Sandbox Server 启动参数 `-node-env-file <path>`（可选，只能与 `-lifecycle-addr` 一起使用）指向
  `KEY=VALUE` 文件；创建容器时把其中每个变量追加到容器环境（排在 `ORA_NODE_CONFIG` 与管理 TLS 变量之后）。
  空行与 `#` 开头的行忽略；值是第一个 `=` 之后的全部内容，按字面使用，不处理引号、不展开变量。
- 变量名必须匹配 `^[A-Z][A-Z0-9_]*$`，且不得以 `ORA_` 开头（`ORA_NODE_CONFIG` 与管理 TLS 变量由 Sandbox
  Server 自己设置）；同名变量出现两次、或只写变量名（Docker env-file 中“从宿主环境复制”的写法）同样拒绝。
  任一不合规、文件缺失都使启动失败；错误信息只含行号与变量名，不含值。
- 变量值不写入 journal、日志或 effect 结果；journal 中的请求与结果保持原样。
- 文件内容在 Sandbox Server 启动时读取一次；修改后需要重启 Sandbox Server，已运行的沙盒不受影响。
- cluster 提供 `compose/node.env.example`，真实的 `compose/node.env` 被 Git 忽略。Compose 始终以只读方式
  挂载该文件（`ORA_NODE_ENV_FILE` 可改路径，且不允许 Docker 在缺失时代建目录）；`task up` 与
  `task agent:fixture:up` 在文件不存在时从示例复制一份（权限 `600`），示例中的变量均已注释，因此默认不注入任何变量。
- Node 不过滤环境：注入的变量经 ora-node 传给插件进程与它启动的 Agent CLI。这意味着沙盒内能执行代码的
  主体（Agent、用户工作负载）都能读到这些值，本地联调接受这一点。
- Agent 以工作负载 UID 1000 运行，而不是 Node 的 root：cluster 的 Node 配置模板设置 `agent.workload_directory`
  （`/var/lib/ora/agent`，由镜像入口脚本创建），Node 在其中为每个会话建立只读插件包视图与私有 HOME（见
  [agent-runtime 根决策](../../node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D2 的实现补充）。
  注入的变量因此能被 Agent 读取，但 Agent 读不到 Node home、管理密钥与账本。
- 模型凭据只是最常见的用途；宿主只能经代理出网时，同一文件也承载 `HTTPS_PROXY`/`NO_PROXY`（`NO_PROXY` 需包含
  对象存储在沙盒网络上的别名，交付上传才不经代理）。OpenCode 的免费模型不需要凭据。

为什么放在 Sandbox Server 而不是 Node 镜像：凭据不应进入镜像层；Sandbox Server 已经是本地“平台”，由它注入
与平台向沙盒注入 Secret 的位置一致，Cloud 与 Controller 都不经手。

### D2：沙盒网络可出网

`ora-sandboxes` 保持为非 internal 的 bridge 网络，沙盒可以访问插件 release 地址与模型 API。第一版不做出网
白名单。

### D3：RustFS 同时接入 Compose 默认网络与沙盒网络

- Compose 增加 `objectstore` 服务（镜像 `rustfs/rustfs:latest`）、一次性凭据生成任务与一次性建桶任务；
  凭据与对象数据都放在持久卷，重复启动不重置凭据，重复建桶不清空已有对象。Cloud 的
  `object_store.endpoint` 为 Compose 网络内地址，`public_endpoint` 为 RustFS 在沙盒网络上的别名地址
  （默认 `ora-revisions`），预签名 URL 使用后者。
- 存储管理员凭据只挂载给建桶任务与 Cloud 等可信服务；Node 不持有凭据，只使用 Cloud 签发的预签名 URL。
- RustFS 是 cluster 中除 Sandbox Server 外唯一接入沙盒网络的服务；Node 能访问的只有它的 S3 端口，
  Controller 与 Cloud 仍不在沙盒网络上。

选择 RustFS 而不是 MinIO 是用户指定的部署选择；决策只依赖 S3 兼容接口（预签名 PUT、HEAD、
path-style 寻址），换用其他 S3 兼容服务不改变本决策的网络与凭据边界。`latest` 标签会变化，验收时记录
实际镜像摘要以便追溯。

## 不变量

1. 注入的环境变量不出现在 journal、日志与 effect 结果中。
2. 沙盒容器仍只接入沙盒网络、不发布端口。
3. Cloud 与 Controller 不接入沙盒网络。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 凭据放进 `ORA_NODE_CONFIG` | 少一个参数；Node 配置被 journal 与日志视为非敏感内容，混入凭据会泄露 |
| 对象存储只在 Compose 默认网络，Cloud 代理上传 | 沙盒网络更封闭；但 Cloud 要中转大文件，违背 Revision 直传的决策 |
| 沙盒网络设为 internal、经代理出网 | 更接近生产隔离；本地联调没有需要，另作安全决策 |

## 落地

Sandbox Server 增加参数与注入（单元测试覆盖变量名校验与不写 journal）；Compose 增加 RustFS、凭据与建桶
任务、env 示例与 Cloud 配置；README 更新。其中 RustFS、凭据、建桶与 Cloud `object_store` 配置已由
cluster#8 落地；`-node-env-file`、`compose/node.env.example` 与 Compose 挂载在 M4 落地
（`sandbox-server/internal/lifecycle/nodeenv.go`，单元测试 `nodeenv_test.go` 覆盖变量名校验、错误不回显值，
以及值只进入容器环境、不出现在 journal 文件、effect 结果与日志中）。
