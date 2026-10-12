---
status: approved
---

# 沙盒注入专用模型客户端证书，个人模型 Key 保留在平台代理

本决策基于 [Docker 生命周期](20260926-docker-sandbox-lifecycle-via-substrate-effects.md)，修改
[模型环境提案](20260928-agent-run-sandbox-environment.md) 中向所有沙盒注入部署共享 Key 的路线。
2026-10-10 已批准的 M4 计划改为个人页面配置与平台代理；`-node-env-file` 不作为本轮模型认证入口。
状态 `approved`，独立真实栈验收未完成，不宣称生产完整执行隔离。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| Docker 生命周期种类、确认授权、代次 fencing、Workspace 独立数据卷 | 继承，不变 |
| WSS 为 Node 提供仅 serverAuth 的身份 | 继承，不变，另签专用 clientAuth 身份 |
| 部署共享 Key 注入容器 | 由个人配置和平台代理替代，沙盒不取得真实 Key |
| RustFS 直传及公开端点 | 继承，不变 |

## 决策

### D1：只有启用模型代理的 Node 才领取专用身份

`agent.model_proxy` 存在时，Sandbox Server 为已获许可的租户、Workspace 和代次签发 model-access 客户端身份。
证书、私钥和公开 CA 仅作为 Docker 私有启动材料提供；不进入 effect 请求、journal 或结果。
Node entrypoint 在管理 tmpfs 保存这些材料，进程配置只引用绝对路径。
原 WSS 服务端证书不被改为双用途，不扩大 Controller 的签发权限。

### D2：Compose 为代理配置独立信任与加密卷

model-gateway 接入默认服务网络和 sandbox 网络，模型请求和授权申请为不同的 HTTPS 监听入口。
浏览器 Gateway 仅挂载公开 CA 并把凭据路径转给该服务；加密主密钥只挂载给 model-gateway。
重复启动复用证书、主密钥和数据库；缺失/不匹配材料失败关闭，不覆盖已有加密身份。
独立验收栈使用单独的 Compose 项目、网络、端口和镜像标签，保留已有 Echo 栈。

## 取舍与验收

相较共享环境 Key，增加一个受控代理和客户端证书，但使真实 Key 不暴露给执行代码的 agent。
既有 root 插件进程与 guardian 纳管缺口不由本决策修复。
用途隔离由 Sandbox Server 和 Desktop 的 TLS 测试验证；实际模型与浏览器链路证据见
[个人模型访问](../../../test-cases/cloud/model-access/personal-model-access.md)。
