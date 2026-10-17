---
status: implemented
---

# 沙盒注入专用模型客户端证书，个人模型 Key 保留在平台代理

本决策基于 [Docker 生命周期](20260926-docker-sandbox-lifecycle-via-substrate-effects.md)，修改
[沙盒部署环境决策](20260928-agent-run-sandbox-environment.md)中向所有沙盒注入部署共享 Key 的路线：模型 Key
由用户在个人页面配置、保存在平台代理（model-gateway），沙盒只取得一张按运行实例签发的 model-access 客户端
证书，凭它向代理申请模型访问。`-node-env-file` 不再是模型认证入口。

本 ADR 处于 `implemented`：签发在 cluster 的 `sandbox-server/internal/management/model_identity.go`，注入在
`sandbox-server/internal/lifecycle/authorization.go`，Compose 部署在 `compose.yaml` 与 `compose/model-*.sh`；Node 侧
落盘在 desktop 的 `docker/node-entrypoint.sh`。验证见
[模型客户端身份核心用例](../../../test-cases/cluster/sandbox-server/model-client-identity.md)。不宣称生产完整执行
隔离：既有 root 插件进程与 guardian 纳管缺口不由本决策修复。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| Docker 生命周期种类、确认授权、代次 fencing、Workspace 独立数据卷 | 继承，不变 |
| WSS 为 Node 提供仅 `serverAuth` 的身份 | 继承，不变，另签专用 `clientAuth` 身份 |
| 部署共享 Key 注入容器 | 修改：由个人配置和平台代理替代，沙盒不取得真实 Key |
| RustFS 直传及公开端点 | 继承，不变 |

## 决策

### D1：只有启用模型代理的 Node 才领取专用身份

Node 配置模板含 `agent.model_proxy` 时，Sandbox Server 在创建已获许可的沙盒时，为该租户、Workspace 和运行代次
另签一张 model-access 客户端证书：独立密钥，用途只有 `clientAuth`，URI SAN 为
`spiffe://ora.local/model-access/tenant/<tenant>/runtime/<workspace>/generation/<generation>`，有效期 24 小时。
证书、私钥和公开 CA 只作为容器私有启动环境（`ORA_MODEL_CLIENT_CERT`、`ORA_MODEL_CLIENT_KEY`、`ORA_MODEL_CA`）
传入，不进入 effect 请求、journal 或结果。Node 入口脚本把它们写入管理 tmpfs（`/run/ora`，权限 `0600`）后从
环境中清除，Node 配置只引用绝对路径；以工作负载 UID 运行的 Agent 读不到这些文件。

原 WSS 服务端证书不改为双用途，Controller 也不获得签发权：签发 CA 的私钥只在 Sandbox Server 的管理卷中。

为什么另签一张而不是复用 WSS 身份：WSS 证书证明“这是哪个 Node”，供 Controller 校验；模型访问需要的是
“这个运行实例可以申请模型”。两种用途共用一把私钥，任何一方泄露都会扩大另一方的权限。

### D2：Compose 为代理配置独立信任与加密卷

- model-gateway 接入默认服务网络和沙盒网络（沙盒网络别名 `ora-model-gateway`）；模型请求与授权申请是不同的
  HTTPS 监听入口。
- 浏览器 Gateway 只挂载公开 CA，并把凭据请求转给 model-gateway；加密主密钥只挂载给 model-gateway。
- 重复启动复用证书、主密钥和数据库；材料缺失或不匹配时失败关闭，不覆盖已有加密身份。
- 独立验收栈使用单独的 Compose 项目、网络、端口和镜像标签，不覆盖已有部署。

## 不变量

1. model-access 证书只有 `clientAuth` 用途，与 Node 的 WSS 服务端证书使用不同密钥。
2. 证书范围精确到租户、Workspace 与运行代次。
3. 证书与私钥不出现在 effect 请求、journal、结果或 Node 配置正文中。
4. 真实模型 Key 不进入沙盒。

## 取舍与验收

相较共享环境 Key，增加一个受控代理和客户端证书，但真实 Key 不暴露给执行代码的 Agent。用途隔离由 Sandbox
Server 与 desktop 的 TLS 测试验证；实际模型与浏览器链路证据见
[个人模型访问](../../../test-cases/cloud/model-access/personal-model-access.md)。

## 本决策未解决的问题

- 证书刷新：沙盒运行超过 24 小时后证书过期，尚无刷新机制。
- root 插件进程与 guardian 纳管缺口：由 Node 进程管理系列处理。
