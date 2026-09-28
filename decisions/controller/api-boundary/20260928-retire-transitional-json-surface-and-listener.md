---
status: proposed
---

# 退役 Controller 过渡 JSON 接口与监听器：可执行入口在任何形态下都不开监听，本机受理只经内嵌句柄

删除 `ora-controller` 可执行入口承载的过渡 JSON clone 接口（`/api/clones`），以及服务于它的监听
Transport、`--transport`／`--host`／`--port`／`--socket` 参数、部署配置的 `api` 段和
`ora_contracts::controller_api` DTO 及其生成的 TypeScript。删除后 Controller 在 SQLite 与云端两种
持久模式下都不打开任何监听。SQLite 持久适配器与 `CloneIntake` 保留，因为 Desktop 本机模式需要本地
Controller；本机工作由内嵌 Controller 的调用方经 `ControllerHandle` 受理。

本决策处于 `proposed` 状态，基于[根决策](0-controller-hosted-client-api.md)、
[Cloud 拥有契约与 Controller 拨出的后续决策](20260922-cloud-owned-contract-and-controller-dial-out.md)与
[Watch 信号决策](20260924-controller-consumes-watch-signals.md)。它同时修改
[可执行入口组合决策](../lifecycle/0-executable-composition-and-single-node.md)中涉及 API 的约定，并与
[minicloud 退役决策](../../minicloud/runtime/20260928-retire-minicloud.md)一起落地。影响范围是
`apps/ora-controller`、`crates/contracts` 与 `packages/contracts`。Controller–Node 协议、Cloud 契约与
SQLite 布局都不变。实现见 desktop 分支 `remove-minicloud`。

## 问题

根决策 D3 把 JSON 接口定为过渡契约：唯一消费者是 minicloud client，浏览器经 Cloud 接入后连同 DTO 一起
删除，不做长期兼容。现在浏览器经 Gateway 访问 Cloud，minicloud 也已决定删除，退役条件已经满足。
同时，拨出决策 D2 已确立 Controller 只拨出、Cloud 不持有 Controller 地址，所以监听器在部署中没有调用方。
保留它会让 Controller 继续维护一套只为开发存在的 Transport 规则、非回环告警、socket 私有创建规则和
错误映射，而且这些规则的测试证据全部依赖已删除的 minicloud 链路。

## 继承与修改

| 先前约定 | 本文处理 |
|---|---|
| 根决策 D1：`ora-controller` 是 API 的唯一宿主，库与传输无关 | 修改：不再有面向调用方的网络 API；库仍与传输无关，`ControllerHandle` 是本机受理的唯一入口 |
| 根决策 D1：派发目标来自 `api.node_id` | 修改：`api` 段删除；目标由内嵌调用方在 `CloneExecutionSpec` 中给出，`ControllerHandle::accept_clone` 只接受已配置的 Node |
| 根决策 D2：Transport 由启动参数显式选择，默认回环 | 删除：可执行入口不再接受任何监听参数，传入即作为未知参数拒绝 |
| 根决策 D3：JSON HTTP 接口是过渡契约 | 按其退役承诺执行：接口、`ora_contracts::controller_api` 与生成的 TypeScript 一起删除 |
| 根决策 D4、D5 及拨出决策 D1–D4：Cloud 拥有 proto，Controller 拨出，云端部署不开监听 | 继承，不变；“不开监听”从云端形态扩展为所有形态 |
| Watch 信号决策 | 继承，不变 |
| 持久协调双适配器：SQLite 与 Cloud 适配器、`CloneIntake` 只由 SQLite 实现 | 继承，不变；`CloneIntake` 的消费者从 JSON 接口改为内嵌调用方 |
| 组合决策 D1：配置文件含 `api.node_id`，`--single-node` 要求唯一 Node 等于 `api.node_id` | 修改：配置文件不含 `api`；`--single-node` 要求 `nodes` 恰有一个 `ipc` Node |
| 组合决策 D2：托管 Node 就绪后才启动 API 与重连 | 修改：就绪后才启动重连并报告组合完成（日志 `ora-controller coordinating locally`） |
| 组合决策 D3 与不变量 3：关停顺序为 API 受理 → Node 会话 → 托管 Node → 数据库租约 | 修改：关停顺序为 Node 会话 → 适配器自有协调 → 托管 Node → 数据库租约 |
| 组合决策 D4：minicloud 启动器的职责 | 失效：启动器随 minicloud 删除 |
| 组合决策不变量 5：托管 Node 意外退出时不继续受理 | 继承，改写为：托管 Node 意外退出时 Controller 失败退出，不改写已接受记录 |

## 决策

### D1：可执行入口在所有持久模式下都不开监听

`ora-controller` 只接受 `--config` 与 `--single-node`。SQLite 形态打开本机所有者，派发并接管该所有者
已接受的工作，并在组合完成后记录 `ora-controller coordinating locally`。云端形态行为不变。两种形态都不
绑定 TCP 端口或 Unix socket。

这样做是因为部署中没有任何调用方会连 Controller：云端的调用方向已固定为拨出，本机的调用方（将来的
Desktop 本机模式）与 Controller 在同一进程内，或者通过尚未设计的本机入口交互。为一个不存在的调用方
保留监听，只会继续带来非回环告警、socket 规则和无认证暴露的风险面。

### D2：本机受理只经内嵌 `ControllerHandle`

`CloneIntake`（`accept_request`、`operations`、`operation`）与 `ControllerHandle` 的受理、列表和查询方法
原样保留。它们的语义不变：持久提交后才返回、相同请求返回原执行、冲突拒绝、缺失与无终态区分。可执行
程序自身没有受理入口，只处理其所有者已经接受的工作；新工作由内嵌 `ControllerRuntime` 的调用方经
`ControllerHandle` 提交。测试以同一所有者在 Controller 进程启动前接受工作、在其停止后读取结果，
这利用的正是数据库租约的独占性。

Desktop 本机模式将以何种形态使用本地 Controller（进程内嵌入，或另立本机入口）不在本文范围内。本文只
保证这一层不预设答案，也不删除它需要的持久适配器。

### D3：删除的契约不留兼容层

`ora_contracts::controller_api`、`packages/contracts/src/dto/controller-api.ts` 以及生成的 DTO barrel 条目
一起删除，重新生成后不应再出现。根决策 D3 已声明不做长期兼容承诺；浏览器的 clone 契约来自 Cloud 的
OpenAPI，不从本仓库生成。

## 不变量

1. `ora-controller` 在任何持久模式下都不打开监听，也不接受监听参数。
2. 非法参数与配置在获取数据库租约前拒绝，不创建 Controller 状态。
3. 本机受理只经 `ControllerHandle`／`CloneIntake`，语义与持久记录一致；可执行程序不另建受理通道。
4. 正常关停顺序为 Node 会话 → 适配器自有协调 → 托管 Node → 数据库租约；不向 Node 升级 `SIGKILL`。
5. 托管 Node 意外退出时 Controller 失败退出，已接受记录保留且不被改写。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保留 JSON 接口，只默认关闭 | 删除成本更低，但接口与 DTO 仍需随 Controller 演进维护，且其测试证据依赖已删除的 minicloud 链路，会长期处于无证据状态 |
| 连同 SQLite 适配器一起删除，Controller 只剩 Cloud 适配器 | Controller 更简单，测试可全部迁到假 Cloud；但 Desktop 本机模式需要本地 Controller，届时要重建一个已实现并有证据的适配器 |
| 可执行入口拒绝 SQLite 持久模式，只允许嵌入 | 语义更干净，但 `--single-node` 托管、就绪、关停顺序等组合证据都建立在 SQLite 形态的可执行入口上，改动面大且没有新收益；保留它作为“派发已接受工作”的本机组合 |
| 为 Desktop 本机模式预先设计一个 Unix socket 受理入口 | 可避免可执行程序暂时没有受理入口，但 Desktop 的集成方式未定，预先设计会把 Transport 规则重新带回来 |

## 本决策未解决的问题

- Desktop 本机模式如何承载或连接本地 Controller，以及那时的受理入口，留给 Desktop 集成决策。
- `--single-node` 在 minicloud 删除后主要服务测试与未来的本机部署，是否保留另行讨论，本文不改变它。

## 落地顺序

1. desktop：删除 `api.rs`、`transport.rs`、监听参数、`api` 段、`controller_api` DTO 与生成的 TypeScript；
   `tests/api.rs` 中与 HTTP 无关的组合拒绝与独占所有者检查改为 `tests/service.rs`；可执行入口与 Node standalone
   测试改为经 `SqliteStore` 受理与读取。完成条件：`cargo test -p ora-controller`、Node standalone 测试、
   `task check:contracts` 与前端 lint 通过。
2. desktop：同步 `docs/controller/local-runtime`、`docs/node/minimal-loop` 与 `docs/protocols/controller-cloud-contract`。
3. specs：同步 API 边界与组合核心用例。

从第 1 步起，可执行入口拒绝 `--transport`／`--host`／`--port`／`--socket` 与 `api` 段，不再监听；已接受工作的
派发、接管和恢复行为不变。
