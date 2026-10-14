# Controller 以双向 TLS 直连沙盒 Node

依据[Controller 接入沙盒网络并以双向 TLS 直连 Node](../../../decisions/cluster/sandbox-server/20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md)。
本文件登记 Cloud 模式下 Controller 与沙盒 Node 之间连接的身份承诺；经 Sandbox Server 路由的明文连接见
[Docker 标签解析用例](docker-label-resolution.md)，Controller–Node 管理通道的业务授权见运行时控制决策的用例。
cluster 证据核对基线 `aef10f3`，desktop 证据核对基线 `a37ca7e0`，路径相对各自仓库根目录。

## Sandbox Node Accepts Only The Pinned Controller Over Mutual TLS

### 风险

沙盒网络上的其他主体（另一个沙盒、持有工作负载证书的 Agent、对象存储或模型代理所在网络上的进程）连上
Node 的管理入口并控制它；或者 Controller 连到冒充目标的 Node；或者配置错误让 Controller 或 Node 回退到明文。

### 前置状态

Controller 以 Cloud 持久化运行，接入沙盒网络；Sandbox Server 以管理目录运行，持有签发 CA；Cloud 已为
Workspace `W` 的运行代次 `G` 签发 ensure 许可，Sandbox Server 创建了容器 `ora-sandbox-S`。

### 触发

Controller 连接 `wss://ora-sandbox-S:<direct_node_port>/ora-node/v1`；另有一方以同一 CA 签发的其他证书、或不带
证书、或以明文连接该 Node。

### 必须成立

- Node 服务端证书只有 `serverAuth` 用途，DNS SAN 为 `ora-sandbox-S`，能由部署 CA 验证。
- Node 只接受与部署固定的 Controller 证书逐字节相同的客户端证书；同一 CA 签发的工作负载证书在 WebSocket
  升级前被拒绝。
- 带客户端 TLS 配置的端点不能是 `ws://`。
- Controller 以 Cloud 持久化启动时，没有 `direct_node_port` 或 effects 接口不是 `https://` 就拒绝启动；配置了
  `direct_node_port` 时，沙盒端点为 `wss://ora-sandbox-<sandboxId>:<port>/ora-node/v1`。
- Node 生产可执行文件拒绝明文 `websocket` 监听。
- Node 证书材料只经私有启动环境传入，不出现在 effect 响应、状态目录与日志中。

### 禁止结果

工作负载证书或其他 Controller 证书连上 Node；Controller 接受 DNS SAN 不是目标容器名的 Node 证书；任何一方回退
到明文；证书或私钥进入 journal、effect 结果或 Node 配置正文。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Node 服务端证书用途与主机名 | cluster `sandbox-server/internal/lifecycle/model_environment_test.go` 的 `TestSecuredEnsurePreservesNodeEnvAndIsolatesModelIdentity` 以部署 CA、容器名与 `serverAuth` 验证 `ORA_NODE_CERT`，并断言 Node 证书材料齐全 | Covered |
| Node 固定 Controller 证书，拒绝工作负载证书 | desktop `crates/node-transport/tests/mtls.rs` 的 `management_tls_authenticates_both_peers_and_refuses_workload_certificate`：同一 CA 签发的工作负载证书连接失败，固定的 Controller 证书完成双向认证并传输数据 | Covered |
| 带 TLS 的端点不回退明文 | desktop `crates/node-transport/tests/mtls.rs` 的 `management_tls_never_falls_back_to_plaintext_even_without_headers`：带客户端 TLS 配置的 `ws://` 端点校验失败 | Covered |
| Node 证书材料不泄露 | cluster 同上 `TestSecuredEnsurePreservesNodeEnvAndIsolatesModelIdentity` 断言正常路径下身份材料不出现在 effect 响应（含重放）、状态目录与日志中；ensure 失败路径尚无直接断言 | Partial |
| Controller 在 Cloud 模式只用直连 | desktop `apps/ora-controller/src/main.rs` 的启动校验与 `apps/ora-controller/src/cloud/fleet.rs` 的直连端点构造均无直接测试；`sandbox_targets_come_from_the_router_and_the_ensure_evidence` 只覆盖未配置 `direct_node_port` 的路由端点 | Missing |
| Node 生产可执行文件拒绝明文监听 | desktop `apps/ora-node/src/main.rs` 的拒绝分支无直接测试 | Missing |
| 真实链路 | cluster `scripts/runtime-control-acceptance.py` 在隔离 Compose 项目中完成 Controller 直连认证 Node、权限隔离与停止启动（记录见 cluster `docs/runtime-control.md`，2026-09-28）；未纳入 CI，需手工重跑 | Partial |

### 决策依据

Controller 直连 D1–D4，不变量 1–5；运行时控制同步交付对 Controller–Node 双向 TLS 身份的要求。
