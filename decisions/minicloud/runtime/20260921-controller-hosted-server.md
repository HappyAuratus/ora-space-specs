---
status: approved
---

# minicloud：移除独立 server，client 直接访问 `ora-controller` 承载的接口

`apps/minicloud/server` 被移除，其 HTTP 接口、组合与关闭逻辑迁入 `ora-controller` 可执行入口；
`apps/minicloud/client` 与 Vite proxy 不变，proxy 目标从 minicloud server 改为
`ora-controller --transport tcp` 的回环监听。开发启动器改为启动 host → `ora-controller --single-node`
→ Vite。浏览器可观察行为、持久接受与恢复规则、回环访问约束均不变；改变的只有可执行程序、
启动参数与 DTO 模块名。

本决策处于 `approved` 状态，方案已批准，入口实现见 desktop#595，剩余验收义务见核心用例。本文基于[根决策](0-local-web-client-embedded-controller.md)，承接
[Controller Client API 决策](../../controller/api-boundary/0-controller-hosted-client-api.md)与
[可执行入口组合决策](../../controller/lifecycle/0-executable-composition-and-single-node.md)在
minicloud 侧的落地。它不改变 minicloud 的非生产定位，不引入认证、Cloud 接入或 Desktop 集成，
也不迁移已有数据目录。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：client／server 分目录的独立应用；不依赖 Desktop 或旧 Backend | 修改：只保留 `apps/minicloud/client`；HTTP 适配归 `ora-controller`。不依赖 Desktop／旧 Backend 的要求不变 |
| D2：server 内嵌 Controller，复用唯一协调实现；不另建任务库、不直接操作内部表 | 修改宿主，规则不变：承载接口的进程就是 Controller 可执行入口，接口层只调用 `ControllerHandle` |
| D2：同一数据目录不能同时运行独立 `ora-controller` 与内嵌实例 | 继承并简化：现在只有一种 Controller 进程，独占规则由数据库租约维持 |
| D2：server 关闭停止 HTTP 受理与自身会话，不取消 Node 已接受执行 | 继承，不变；关闭顺序由组合决策 D3 固定 |
| D3：仅本机访问，server 拒绝非回环监听 | 修改：拒绝非回环从程序内检查改为启动器与启动参数约束——启动器只生成 `--host 127.0.0.1`，Controller 默认回环；minicloud 部署中不出现非回环监听 |
| D3：开发启动器以私有目录为状态根，停止时先 Node 收尾再停 host／guardian | 修改：Node 的启动与收尾交给 `ora-controller --single-node`；启动器仍负责 guardian 部署、host 与 Vite，停止顺序不变 |
| D4：提交 clone 与轮询查询、幂等重传、未知不伪装失败 | 继承，不变；HTTP 路径与状态语义逐字保留 |
| 浏览器 DTO 由 `ora-contracts::minicloud` 经 ts-rs 生成 | 修改：模块更名为 `ora_contracts::controller_api`，生成文件同步更名；字段与语义不变 |

## 决策

### D1：目录与进程组合

```text
浏览器 → Vite dev server → proxy → ora-controller（API + Controller 运行时）
                                        ↓ 本机 IPC
                                       ora-node（--single-node 托管）→ host／guardian → Git
```

`apps/minicloud/` 下只剩 `client`。启动器生成的配置从 `server.json{listen, node_id, controller}`
改为 Controller 配置（含 `api.node_id` 与 `single_node` 段），监听地址与端口以启动参数传给
`ora-controller`。`task run:minicloud`、`task test:minicloud` 与手动部署文档相应改为构建和
启动 `ora-controller`。

### D2：回环约束的落点

根决策把“拒绝非回环”放在 server 程序内，因为 server 只服务 minicloud。`ora-controller` 还要
服务未来的 Go Cloud，不能在程序内一律拒绝非回环；因此 minicloud 的回环约束改由两层保证：
Controller 默认 `127.0.0.1`，启动器显式传 `--host 127.0.0.1` 并继续校验端口为回环。这仍是部署
约束而非安全保证，与根决策 D3 的定位一致。

### D3：过渡性质

迁入 `ora-controller` 的 JSON 接口在 Client API 决策中被定义为过渡契约，浏览器经 Go Cloud
接入后删除。minicloud client 在那之前继续作为该接口的唯一浏览器消费者；本文不为 client
增加新的接口或功能。

## 不变量

1. 浏览器经 Vite proxy 访问的路径、状态语义、错误映射与根决策相同。
2. minicloud 部署中的 Controller 只监听回环，由启动器参数与默认值共同保证。
3. 同一数据目录仍只有一个 Controller 运行所有者；迁移不清库、不重建、不迁移 checkout。
4. 启动器停止顺序保持 Vite → Controller（含 Node 收尾）→ host → guardian。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保留 minicloud server 作为 `ora-controller` 之上的薄代理 | 与根决策的目录结构一致，但多一跳、多一份配置与关闭逻辑，而它已没有独立职责 |
| 让 client 直连 `ora-controller --transport unix` | 少一个 TCP 端口，但浏览器无法直接访问 Unix socket，Vite proxy 仍需 TCP |
| 迁移时顺带把 client 改为经 Go Cloud 访问 | 一步到位，但 Go Cloud 尚未调用 Controller；把两项变更绑在一起会让本次迁移无法用现有测试逐字验证 |

## 本决策未解决的问题

- client 何时改为经 Go Cloud 访问并退役过渡接口，取决于 Cloud 侧接入进度。
- 非 Linux 开发环境仍不在 minicloud 范围内。

## 落地顺序

1. 与 Client API 决策落地顺序第 1 步同步完成：迁移接口与测试、删除 server、更名 DTO、改写启动器
   与文档。完成条件：[本机 Web 闭环验收](../../../test-cases/minicloud/runtime/local-web-clone-loop.md)
   全部义务在新入口上保持 `Covered`，证据路径改指 `ora-controller` 的测试；
   DOM 测试不变。
2. 已根据 desktop#595 更新上述用例页的迁移义务、ora-controller 测试路径与证据状态；
   完整启动器关闭顺序等剩余缺口保留 `Partial`，补齐直接证据后再更新为 `Covered`。
