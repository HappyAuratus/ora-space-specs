---
status: proposed
---

# minicloud：clone 闭环经 Cloud 承载，Controller 以云端持久模式运行

minicloud 增加一种运行形态：浏览器不再直接调用 `ora-controller` 的过渡 JSON 接口，而是经 Vite
proxy → 认证 Gateway → Cloud 公开 API（`/api/v1/tenants/{tid}/clones`）提交与查询 clone；Cloud
在 PostgreSQL 中接受请求，`ora-controller` 以 `persistence: cloud` 运行、通过内部控制契约领取工作、
登记派发、接管 Node 结果；Node 与 Git 链路不变。浏览器可观察的 DTO 字段与状态语义逐字保留，
改变的只有入口与权威存储。现有本机形态（`persistence: sqlite`，浏览器直连 Controller）保留为
无 Cloud 的开发路径。

本决策处于 `proposed` 状态，基于[根决策](0-local-web-client-embedded-controller.md)与
[Controller 承载 server 的后续决策](20260921-controller-hosted-server.md)，承接
[Cloud 内部控制契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)
D4／D8 与[持久协调双适配器](../../controller/persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md)
在 minicloud 侧的落地。它不改变 minicloud 的非生产定位，不引入 Desktop 集成，也不实现沙盒或
远程 Node。

## 继承与修改

| 先前约定 | 本文处理 |
|---|---|
| 根决策 D2：复用唯一协调实现，不另建任务库 | 继承：云端形态下协调实现仍是 `ora-controller`，权威存储改由 Cloud 持有 |
| 根决策 D3／后续决策 D2：仅本机访问、回环监听 | 继承：Cloud、Gateway、Controller、Node 全部在本机回环上运行；Controller 的 gRPC 拨出目标是回环地址 |
| 根决策 D4：提交 clone 与轮询查询、幂等重传、未知不伪装失败 | 继承：语义逐字保留，入口改为 Cloud 公开 API，`Idempotency-Key` 承担 `request_id` 的幂等 |
| 后续决策 D1：浏览器 → Vite proxy → `ora-controller` | 修改：云端形态下 Vite proxy 指向 Gateway；`ora-controller` 不再提供 JSON 接口 |
| 后续决策 D3：JSON 接口是过渡契约 | 继承：本机形态继续使用；云端形态退役它 |
| 浏览器 DTO 由 `ora-contracts::controller_api` 生成 | 修改：云端形态的 DTO 来自 Cloud 的 OpenAPI 生成客户端，字段与语义与过渡 DTO 相同 |

## 决策

### D1：两种运行形态由启动器的一个显式开关选择

```text
本机：浏览器 → Vite proxy → ora-controller(sqlite) → Unix socket → Node → Git
云端：浏览器 → Vite proxy → Gateway(:8081) → Cloud(:8080, PostgreSQL)
                                                  ↑ gRPC(:8082)，Controller 拨出
                                ora-controller(cloud) → Unix socket → Node → Git
```

启动器为云端形态生成 `persistence: { kind: cloud, endpoint, claim_interval_ms }` 的 Controller
配置，不生成也不分发任何密钥：当前阶段 Controller 只以 `ControllerId` 声明自身（Cloud 根决策 D7）；Cloud、Gateway 与 PostgreSQL 由
开发者按 Cloud 仓库的本地方式启动，启动器只校验它们可达。不提供"Cloud 不可达时退回本机"的
隐式降级：两种形态是不同的权威存储，切换必须显式。

### D2：登录与租户

云端形态复用 Gateway 的登录流程，浏览器持有 Gateway 会话；minicloud 客户端调用 `/api/v1/me`
取得租户后再提交 clone。Controller 与 Node 仍不感知租户：Cloud 交给 Controller 的只有
`operation_id` 与输入。

### D3：验收

以真实 Cloud（PostgreSQL）、Gateway、`ora-controller --single-node`（云端模式）与 Node 在一台
机器上完成一次 HTTPS clone；浏览器查询看到 `succeeded{path, commit}`；中途强杀 Controller 后
重启，不产生第二次 clone，结果与 Ack 唯一。

## 不变量

1. 浏览器可观察的 clone 语义在两种形态下相同：接受即持久、幂等重传、未知不伪装失败。
2. 云端形态下 Controller 不打开 SQLite，本机形态下不连接 Cloud；两者不互为后备。
3. 全部组件只在回环上监听；Controller 不为 Cloud 暴露服务。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 浏览器继续直连 Controller，Controller 再转发到 Cloud | 少改客户端；但让 Controller 承担用户入口与租户上下文，违反"Controller 不感知租户" |
| 只保留云端形态 | 少一种组合；但 minicloud 的无 Cloud 开发路径是 Desktop 场景的硬需求 |

## 本决策未解决的问题

- Controller 认证：等产品需要机器身份时另立决策，本形态届时再补充密钥或凭据的发放方式。
- 云端形态下的 Node 远程连接与沙盒，仍由后续决策承担。
