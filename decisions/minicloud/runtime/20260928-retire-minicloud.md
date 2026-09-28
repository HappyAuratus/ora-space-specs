---
status: implemented
---

# minicloud 退役：本机联调改由 cluster Compose 栈承担，删除 minicloud 全部形态

删除 minicloud：包括浏览器经 Vite proxy 直连 `ora-controller` 的本机形态，以及启动器驱动 Controller
以云端持久模式运行的云端形态。删除范围是 client、启动器、`run:minicloud`／`test:minicloud` 任务和
文档。本机端到端联调改在 cluster 仓库的 Compose 栈中进行，链路为 Gateway → Cloud → Controller 拨出 →
Sandbox Server → Node。minicloud 只服务开发，没有生产用户，不承担迁移义务；启动器留在
`~/.ora/minicloud/` 与 `~/.ora/cloud/` 下的状态不删除、不迁移。

本决策处于 `implemented` 状态，基于[根决策](0-local-web-client-embedded-controller.md)、
[Controller 承载 server 的后续决策](20260921-controller-hosted-server.md)与
[云端形态的后续决策](20260922-cloud-hosted-clone-loop.md)。它与
[Controller 过渡 JSON 接口退役决策](../../controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md)
一起落地，该决策负责 Controller 一侧的接口、监听器与 DTO。由 [desktop#627](https://github.com/ora-space/desktop/pull/627) 实现（尚未合并）。

## 问题

minicloud 的前提是：浏览器可以直接调用 Controller 承载的接口，由此方便地验证 clone 闭环。实际部署中
不存在这条路径。Controller 只拨出 Cloud
（[拨出决策](../../controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md) D2），
不开端口等调用方来连。本机形态因此验证的是一个不会部署的入口，它还迫使 Controller 长期背着一个只服务
开发的监听器、CLI 参数和 DTO。云端形态虽然走了真实的拨出方向，但它由 minicloud 启动器拼装，Node 以
`--single-node` 托管在本机，没有经过沙盒生命周期和路由。现在 cluster 的 Compose 栈已经承担了同一件事，
而且更接近部署。

## 继承与修改

| 先前约定 | 本文处理 |
|---|---|
| 根决策 D1：新建非生产 `apps/minicloud/client` 与 server 承载首个 Client | 修改：client 删除（server 已由 20260921 决策删除）；首个 Client 角色由 Cloud 公开 API 与前端承担 |
| 根决策 D2：复用唯一协调实现，不另建任务库 | 继承，不变：Controller 与 Node 的协调实现不因入口删除而改变 |
| 根决策 D3：仅本机访问、回环监听 | 失效：不再有 minicloud 监听；Controller 不再有任何监听 |
| 根决策 D4：提交／轮询、幂等重传、未知不伪装失败 | 继承：由 Cloud 公开 API 按相同语义承担 |
| 20260921 决策 D1–D3：浏览器 → Vite proxy → `ora-controller` 的过渡 JSON 接口 | 修改：该链路删除，接口本身的退役见 Controller 过渡 JSON 接口退役决策 |
| 20260922 决策 D1：启动器以 `--cloud` 选择两种形态 | 修改：启动器删除，两种形态均不再存在 |
| 20260922 决策 D2／D3：经 Gateway 登录与租户提交 clone、Controller 以云端持久模式运行 | 继承其语义：cluster Compose 栈中的 Controller 同样以云端持久模式运行，调用方同样经 Gateway 访问 Cloud |

## 决策

### D1：删除 minicloud 的全部代码与任务

删除 `apps/minicloud/client`、`scripts/run-minicloud.ts` 及其测试、Taskfile 的 `run:minicloud` 与
`test:minicloud`、前端 lint／test 中的 minicloud 过滤项、`deno.json` workspace 条目、CI 路径过滤、
`docs/minicloud/`，以及只服务 minicloud 的 Node standalone 测试 `repository_minicloud.rs`。
`repository_single_node.rs` 原先借用 minicloud 的可执行入口启动函数，现在改为自带启动函数。

两种形态一起删除，因为它们共享同一个启动器和同一份文档，各自的验证目标也都已被 cluster 覆盖：本机
形态验证的入口不会部署，云端形态验证的拨出链路在 cluster 中以更接近部署的形状存在。

### D2：本机端到端联调由 cluster 承担

本机联调入口是 cluster 仓库的 `task up`。前端由 Gateway 提供，Cloud 接受请求，Controller 以云端持久模式
拨出 Cloud，并经 Sandbox Server 的生命周期接口创建沙盒，经其路由连接沙盒中的 Node。本决策不改变 cluster
的设计，只确认它取代 minicloud 成为本机闭环的承载者。

代价是 `test:minicloud` 曾是唯一一条自动化的真实 HTTP clone 闭环测试，删除后 cluster 链路还没有自动化
端到端测试。这一缺口在核心用例中按 `Missing` 如实记录，不以下层测试填补。

### D3：启动器状态原样保留

启动器写在 `~/.ora/minicloud/<digest>/` 与 `~/.ora/cloud/<digest>/` 的配置、数据库、clone 目录和 guardian
副本不删除、不迁移。这些目录位于家目录，不是仓库内产物，也没有新的布局会占用这些路径，所以保留不会
造成路径冲突；是否清理由开发者自行决定。

## 不变量

1. 仓库中不再有浏览器直连 Controller 的入口或其启动器。
2. 删除 minicloud 不改变 Controller 与 Node 的协调、持久接管和恢复规则。
3. 删除不修改或删除 minicloud 启动器在家目录下留下的状态。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 只删本机形态，保留启动器的云端形态 | 云端形态确实走真实拨出方向，但它以 `--single-node` 本机托管 Node，绕开沙盒生命周期和路由；cluster 已覆盖同一链路且更接近部署，保留两套本机联调入口只会分散维护 |
| 保留 minicloud 作为无 Cloud 的轻量调试入口 | 启动更快，但它验证的是一个部署中不存在的入口，并要求 Controller 为它保留监听器与 DTO，与拨出决策相悖 |
| 先补 cluster 自动化端到端测试再删除 | 不留测试空窗，但会让一个错误方向的入口继续存在并继续约束 Controller；缺口已作为 `Missing` 登记，可独立补齐 |

## 本决策未解决的问题

- cluster 链路的自动化端到端测试：需要在不本地构建镜像的约束下决定运行环境（例如 CI），另行设计。

## 落地结果

- desktop#627 的 `chore: remove minicloud` 删除了 D1 列出的全部代码、任务、文档与测试；
  `repository_single_node.rs` 自带可执行入口启动函数。前端 lint、工具测试与 Node standalone 测试通过。
- 从该提交起，`task run:minicloud` 与 `task test:minicloud` 不再存在；Controller 与 Node 的行为不变。
- cluster 链路的自动化端到端测试仍缺失，登记在[核心用例](../../../test-cases/minicloud/runtime/cloud-hosted-clone-loop.md)中。
