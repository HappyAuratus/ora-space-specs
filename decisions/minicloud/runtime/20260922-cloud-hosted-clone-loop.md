---
status: implemented
---

# minicloud：clone 闭环经 Cloud 承载，Controller 以云端持久模式运行

minicloud 增加一种运行形态：clone 请求不再进入 `ora-controller` 的过渡 JSON 接口，而是经认证 Gateway
→ Cloud 公开 API（`/api/v1/tenants/{tid}/clones`）提交与查询；Cloud 在 PostgreSQL 中接受请求，
`ora-controller` 以 `persistence: cloud` 运行，通过内部控制契约领取工作、登记派发、接管 Node 结果；Node
与 Git 链路不变。调用方可观察的 clone 字段与状态语义与过渡接口逐字相同，改变的只有入口与权威存储。
本机形态（`persistence: sqlite`，浏览器经 Vite proxy 直连 Controller）保留为无 Cloud 的开发路径。

本决策处于 `implemented` 状态，基于[根决策](0-local-web-client-embedded-controller.md)与
[Controller 承载 server 的后续决策](20260921-controller-hosted-server.md)，承接
[Cloud 内部控制契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md) D4／D8 与
[持久协调双适配器](../../controller/persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md)
在 minicloud 侧的落地。启动器云端模式由 desktop#607 落地（已合入），经 Gateway 开发登录提交的流程由
desktop#610 落地（尚未合并，证据核对 `cf501280`）；验证证据登记在
[核心用例](../../../test-cases/minicloud/runtime/cloud-hosted-clone-loop.md)。它不改变 minicloud 的非生产
定位，不引入 Desktop 集成，也不实现沙盒或远程 Node。

## 继承与修改

| 先前约定 | 本文处理 |
|---|---|
| 根决策 D2：复用唯一协调实现，不另建任务库 | 继承：云端形态下协调实现仍是 `ora-controller`，权威存储改由 Cloud 持有 |
| 根决策 D3／后续决策 D2：仅本机访问、回环监听 | 继承：Cloud、Gateway、Controller、Node 都在本机运行；Controller 的 gRPC 拨出目标是回环地址，自身不开监听 |
| 根决策 D4：提交 clone 与轮询查询、幂等重传、未知不伪装失败 | 继承：语义逐字保留，入口改为 Cloud 公开 API，`requestId` 仍是持久请求身份 |
| 后续决策 D1：浏览器 → Vite proxy → `ora-controller` | 修改：云端形态不启动 minicloud 前端与 Vite，调用方直接经 Gateway 访问 Cloud；`ora-controller` 不提供 JSON 接口 |
| 后续决策 D3：JSON 接口是过渡契约 | 继承：本机形态继续使用；云端形态不使用 |
| 浏览器 DTO 由 `ora-contracts::controller_api` 生成 | 云端形态的客户端契约来自 Cloud 的 OpenAPI，字段与语义与过渡 DTO 相同 |

## 决策

### D1：两种运行形态由启动器的一个显式开关选择

```text
本机：浏览器 → Vite proxy → ora-controller(sqlite) → Unix socket → Node → Git
云端：调用方 → Gateway(:8081) → Cloud(:8080, PostgreSQL)
                                   ↑ gRPC(:8082)，Controller 拨出
                 ora-controller(cloud) → Unix socket → Node → Git
```

`task run:minicloud -- --cloud` 启动 host、`ora-controller --single-node`（云端持久模式）与 Node，不启动
minicloud 前端与 Vite；状态位于 `~/.ora/cloud/<digest>/`，与本机形态的 `~/.ora/minicloud/<digest>/` 分开，
因为 Node 的持久记录只属于一个持久权威，SQLite 接受的工作不能报给 Cloud，clone 目标目录也不能共用。
启动器为云端形态生成 `persistence: { kind: cloud, endpoint, claim_interval_ms }` 且无 `api` 段的 Controller
配置，保留用户修改过的 Cloud 地址，拒绝持久模式与启动模式不一致的已有配置；它不生成也不分发任何
密钥，Controller 只以 `ControllerId` 声明自身。

Cloud、Gateway 与 PostgreSQL 由开发者按 Cloud 仓库的本地方式另行启动。启动器探测 Cloud 的 gRPC 地址，
不可达时只提示一次并继续，因为 Controller 自身会持续重试 Cloud；这比提案中"只校验可达"更宽松，是为了
允许先启动 minicloud、后启动 Cloud。两种形态之间没有隐式降级：Cloud 不可达时不退回本机形态。

### D2：登录与租户

调用方使用 Gateway 的开发登录取得会话，按用户幂等地创建或取得租户，再经
`/api/v1/tenants/{tid}/clones` 提交与查询；minicloud 一侧不持有任何密钥或 token。Controller 与 Node 不感知
租户：Cloud 交给 Controller 的只有 `operation_id` 与输入。

与提案不同，minicloud 浏览器客户端尚未接入云端形态，提交与查询目前由文档给出的命令经 Gateway 完成。
原因是本阶段的目标是 Cloud 承载的 clone 闭环本身；把浏览器客户端改为经 Gateway 登录并使用 Cloud 生成
客户端，是独立的前端变更，不改变本文的入口、权威与语义承诺。

### D3：验收

以真实 Cloud（PostgreSQL）、Gateway、`ora-controller --single-node`（云端模式）与 Node 在一台机器上完成
HTTPS clone，查询看到 `succeeded{path, commit}` 且磁盘 HEAD 与 `commit` 一致；以同一 `requestId` 重复提交
返回原操作；登记派发后强杀 Controller，Node 独立完成 clone，重启后 Controller 接管 Node 重放的原事件，
Cloud 只有一次执行。该验收目前为本机手工验证，尚无自动化测试。

## 不变量

1. 调用方可观察的 clone 语义在两种形态下相同：接受即持久、幂等重传、未知不伪装失败。
2. 云端形态下 Controller 不打开 SQLite，本机形态下不连接 Cloud；两者不互为后备，状态目录不共用。
3. Controller 不为 Cloud 暴露服务，云端形态下不开任何监听；启动器不生成或分发密钥。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 调用方继续直连 Controller，Controller 再转发到 Cloud | 少改客户端；但让 Controller 承担用户入口与租户上下文，违反"Controller 不感知租户" |
| 只保留云端形态 | 少一种组合；但无 Cloud 的开发路径是 Desktop 场景的硬需求 |
| 两种形态共用一个状态目录 | 少一套目录；但 Node 的持久记录会同时被两个权威认领，SQLite 接受的工作可能被报给 Cloud |

## 本决策未解决的问题

- minicloud 浏览器客户端经 Gateway 登录并使用 Cloud 生成客户端。
- 云端形态闭环的自动化验收。
- Controller 认证：产品需要机器身份时另立决策，届时补充凭据的发放方式。
- 云端形态下的 Node 远程连接与沙盒。

## 落地结果

- 启动器云端模式（desktop#607）：`--cloud` 开关、独立状态目录、云端持久配置的生成与一致性校验、Cloud
  地址探测；启动器测试覆盖。
- 经 Gateway 开发登录提交（desktop#610）：Cloud 删除 devgateway 后，文档改为开发登录 → 租户 → clones。
- 与提案不同的最终决定：云端形态不启动 minicloud 前端，调用方直接经 Gateway 访问（D1、D2）；Cloud 不可达
  时启动器只提示不失败（D1）。
