---
status: implemented
---

# Cloud↔Controller 契约改由 Cloud 拥有：Controller 拨出、以 submodule 锁定 commit 消费 proto 并生成客户端

Cloud↔Controller 的 gRPC 契约不再由本仓库（desktop）拥有，而是由 `ora-space/cloud` 仓库的
`proto/` 唯一定义；Controller 是该契约的**客户端**，Cloud（Go API Server）是服务端。调用方向固定为
Controller 拨出：Cloud 不向 Controller 发起连接，Cloud→Controller 的信号经同一契约上的服务端流提供。
desktop 以 git submodule `third_party/cloud` 锁定 Cloud 的契约 commit，sparse-checkout 只展开 `proto/`，
生成 tonic 客户端代码提交到 `ora-controller-proto`。`ora-controller` 不为 Cloud 暴露任何监听器；云端
部署下它不打开任何监听。

本决策处于 `implemented` 状态。它修改[根决策](0-controller-hosted-client-api.md)的 D4（契约归属与生成
方向）和 D5（调用方向），继承 D1–D3。契约消费链由 desktop#598 落地，Cloud 适配器与拨出任务由
desktop#601、desktop#602 落地（均已合入）；Cloud 侧契约见
[Cloud 拥有 Internal Control gRPC 契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)，
Controller 侧持久化抽象见
[持久协调接口与双适配器](../persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md)。
验证证据登记在
[API 边界核心用例](../../../test-cases/controller/api-boundary/controller-hosted-client-api.md#generated-cloud-contracts-preserve-the-boundary)。
本文只回答"契约归谁、谁连谁、desktop 怎么拿到 proto、错误如何进入 Controller"；它不定义认证、TLS，
也不改变本机模式下浏览器经 Vite proxy 使用的过渡 JSON 接口。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：`ora-controller` 是 API 唯一宿主，库与传输无关 | 继承，不变；面向 Cloud 的 RPC 客户端同样是适配模块 |
| D2：Transport 由启动参数显式选择，默认回环 | 继承，不变；该监听器不承载 gRPC，只承载本机模式的过渡 JSON 接口 |
| D3：JSON HTTP 接口是过渡契约，不是 Cloud 契约 | 继承，不变 |
| D4：`.proto` 放本仓库 `proto/`，由 Controller（服务端）拥有；Cloud 以 git tag 消费 | 修改：`.proto` 放 `ora-space/cloud`，由 Cloud（服务端）拥有；desktop 以 submodule 锁定 commit 消费，只生成客户端 |
| D4：`buf lint`／`buf breaking` 在本仓库执行；破坏性变更以 `v2` package 表达 | 修改归属，规则不变：lint 与 breaking 在 Cloud 仓库执行；desktop 只做 submodule 一致性与生成漂移检查 |
| D4：错误分类以 proto 枚举携带在 status details，Rust 侧映射只写一次 | 修改方向，规则不变：状态码与 `ErrorDetail` 由 Cloud 产生；Rust 侧在 Cloud 适配器内映射一次为持久协调接口的错误分类 |
| D4：契约不含 tenant、user、membership 字段 | 继承，不变 |
| D5：API Server 主动拨号 Controller；NAT 场景下的反向连接留作未决项 | 修改：所有部署中 Controller 拨出；不再有 Cloud 拨入 Controller 的形态 |
| D5：浏览器面向 Cloud 保持 JSON 与 SSE；不为 PTY 预留 gRPC 服务 | 继承，不变 |

## 决策

### D1：契约唯一源在 Cloud 仓库，desktop 以 submodule 锁定 commit 消费

desktop 通过 git submodule `third_party/cloud` 引用 Cloud 仓库，gitlink 锁定一个位于 Cloud 主干的 commit；
`task proto:init` 以 partial clone 初始化并 sparse-checkout 只展开 `proto/`。生成的 prost／tonic 客户端代码
（固定版本的 `buf` 远程插件，`no_server`）提交到 `crates/controller-proto/src/gen`；desktop 不保留任何
`.proto` 副本，也不再有 `proto/` 目录。已提交的生成物让不需要改契约的构建（包括 Windows）不依赖
submodule 或 `buf`。

`task proto:check` 在 `task lint:crates` 与 PR CI 中执行：

| 检查 | 规则 |
|---|---|
| gitlink 一致 | submodule 实际 checkout 的 commit 与 desktop 提交的 gitlink 相同 |
| 无本地修改 | submodule 的 `proto/` 没有未提交修改 |
| 生成可重现 | 用锁定 commit 重新生成后 `src/gen` 无 diff |
| 不做 lint／breaking | 属于 Cloud 仓库的义务，desktop 不重复执行 |

更新契约的流程：Cloud 合入契约变更 → desktop 移动 submodule 指针 → 重新生成 → 修改适配器 → 与生成物、
指针一起提交。CI 使用锁定 commit，不跟踪 Cloud 分支最新值。

选择 submodule 而不是自定义锁文件加 proto 快照，是因为 gitlink 是 Git 原生的版本锁定，不需要另写"提取、
校验、复制"逻辑，也不会出现锁文件与快照不一致的第三种状态；代价是修改契约时需要初始化 submodule 并能
读取 Cloud 仓库。`/specs` 是未被 desktop 锁定的独立 checkout，不能作为契约依赖。

### D2：调用方向固定为 Controller 拨出，Cloud 部署不打开监听

Controller 是 gRPC 客户端，向 Cloud 发起所有调用：租约获取／续期／释放、领取、派发登记、事件接管与
恢复读取都是 Controller 发出的 unary 请求。Cloud 在任何部署中都不拨号 Controller，`ora-controller` 也不为
Cloud 暴露服务；云端持久模式下 `ora-controller` 拒绝 `--port`／`--socket` 等监听参数，不打开任何监听器。

面向 Cloud 的协调是运行时内一个长驻任务（Cloud 适配器的 `serve`），与 Node 会话重连循环并列：每 10 秒
续期 30 秒的租约，按部署配置的 `claim_interval_ms` 周期领取，登记派发后交给 Node 会话投递。关闭时先停止
API 受理，再关闭 Node 会话，最后停止该任务并释放租约；释放失败只意味着后继者等待租约过期。Cloud 不可达
时进程保持运行且不具备写资格，持续重试，不退出也不降级。

Cloud 契约提供 `Watch` 服务端流（`WorkAvailable`／`Drain`／`NodeAssignment`）。流上的信号只是加速器：工作
归属由 Cloud 事务决定，信号丢失只影响延迟。**当前 Controller 不消费该流，只做周期领取**；这偏离了提案
中"拨出任务同时维持 `Watch`"的计划。原因是周期领取已满足第一版的正确性与 minicloud 的延迟要求，而
`Drain` 的排空语义依赖尚未建立的 scale 决策；接入 `Watch` 只改变延迟与排空时机，不改变本文的方向与
持久化承诺，留作后续工作。

固定拨出方向的原因：Controller 是可替换、可缩放的 worker，它的地址与存活不应成为 Cloud 的配置项；拨出
天然穿越 NAT 与动态寻址，将来的认证也只需在一个方向建立。

### D3：错误分类由 Cloud 产生，Rust 侧在适配器内映射一次

Cloud 以 gRPC 状态码作为主分类并附 `ErrorDetail`（映射表由 Cloud 决策拥有）。desktop 的 Cloud 适配器在
`cloud/fault` 一处把状态码映射为持久协调接口的错误分类，协调逻辑与会话不感知 gRPC；`ErrorDetail` 只用于
日志与消息。

| Cloud 返回 | 接口分类 | Controller 行为 |
|---|---|---|
| `ABORTED`、`INVALID_ARGUMENT`、`ALREADY_EXISTS`、`OUT_OF_RANGE` | 冲突 | 不重试同一调用；不派发、不确认 |
| `NOT_FOUND` | 缺失：读取返回"无记录"，要求事实存在的写入视为冲突 | 与"已接受但无终态"区分 |
| `FAILED_PRECONDITION` | 资格失效 | 丢弃持有的租约 epoch，下一次续期周期重新获取；期间不写入 |
| `UNAVAILABLE`、`RESOURCE_EXHAUSTED`、`UNIMPLEMENTED`、`UNAUTHENTICATED`、`PERMISSION_DENIED`；读取超时 | 不可用：未提交，可稍后重试 | 保持未派发、未确认 |
| 写入超时、`DEADLINE_EXCEEDED`、`CANCELLED`、`UNKNOWN`、`INTERNAL`、`DATA_LOSS` | 未知：可能已提交 | 以同一提交身份重传，至多三次；仍未知则上报未知 |

与提案相比，`PERMISSION_DENIED` 归为"不可用"而不是"资格失效"：当前阶段 Cloud 不认证 Controller，
拒绝只能说明两端部署不一致，不是租约陈旧，丢弃租约无济于事。读取超时归为"不可用"，因为读取不提交
任何东西。"未知"是远程适配器特有的分类：本机 SQLite 事务不会产生"已提交但回复丢失"。

### D4：租户边界与过渡接口不变

契约只携带 Cloud 已授权的 opaque 身份，不出现 tenant、user、membership 或角色字段；Controller 不做租户
筛选，也不解析或校验任何 token——当前阶段它只在每次调用的 `x-ora-controller-id` metadata 中声明自己的
`ControllerId`，不出示凭据（Cloud 根决策 D7）。适配器在启动时拒绝无法作为 gRPC ASCII metadata 传输的
`ControllerId`。本机 SQLite 模式下过渡 JSON 接口、`ora_contracts::controller_api` DTO 与浏览器链路不变；
云端模式下 Controller 不提供该接口，接受入口是 Cloud 公开 API（见
[minicloud 云端形态](../../minicloud/runtime/20260922-cloud-hosted-clone-loop.md)）。

## 不变量

1. Cloud↔Controller 契约唯一来源是 Cloud 仓库的 proto；desktop 只持有由锁定 commit 生成的客户端代码，
   且生成物与锁定 commit 无漂移。
2. Controller 只拨出，不为 Cloud 暴露服务；云端部署不打开任何监听器；Cloud 不持有 Controller 的地址。
3. 服务端流上的信号不构成工作归属或状态变更；Controller 不依赖信号保证正确性。
4. gRPC 状态码由 Cloud 产生；desktop 只在 Cloud 适配器内映射一次。
5. 契约与 Controller API 不含租户、用户或成员字段；Controller 不解析任何 token。
6. 本机模式的过渡 JSON 接口行为、DTO 与退役承诺不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 维持根决策 D4：proto 放 desktop，Controller 作为服务端，Cloud 拨入 | 与 Cloud 是权威持久化、Controller 是无状态 worker 的架构相反；Cloud 还要维护每个 Controller 的地址与存活 |
| proto 放 specs 仓库 | specs 是未被 desktop 锁定的 checkout，不能作为可重现构建输入 |
| 自定义锁文件 + 复制 proto 快照到 desktop | 评审可直接看 proto diff、无需初始化 submodule；但要自写提取／校验／复制逻辑，并可能出现锁文件、快照、Cloud 三者不一致 |
| Controller 同时暴露一个小 gRPC 服务供 Cloud 拨入 | 需要 Controller 可寻址、双向认证与 TLS；服务端流在单向连接下提供同样的通知能力 |
| 第一版就消费 `Watch` | 派工延迟更低、可响应 `Drain`；但排空语义依赖尚未建立的 scale 决策，而周期领取已满足第一版正确性 |

## 本决策未解决的问题

- Cloud 连接的认证凭据、TLS 与凭据轮换，留给认证 ADR。
- Controller 消费 `Watch`，以及 `Drain`、`NodeAssignment` 的完整语义与多 worker 租约，留给 scale 决策。
- 过渡 JSON 接口何时退役，仍取决于浏览器经 Cloud 接入的进度。

## 落地结果

- 契约消费链（desktop#598）：`third_party/cloud` submodule 锁定 Cloud 主干上的契约 commit，
  `proto:init`／`proto:generate`／`proto:check` 任务与 PR CI 检查，`ora-controller-proto` 客户端生成物；
  desktop 原有 `proto/` 目录与 lint／breaking 任务已移除。
- Cloud 适配器与拨出任务（desktop#601、#602）：租约维持、周期领取、派发登记、事件接管、恢复读取与
  D3 的错误映射；云端部署不开监听、不创建本地状态。
- 与提案不同的最终决定：Controller 暂不消费 `Watch`（D2）；`PERMISSION_DENIED` 与读取超时归为不可用（D3）。
- 以真实 Cloud gRPC 服务端为对端的 clone 闭环（含登记后强杀 Controller、重启后接管原事件）已在本机
  手工验证（desktop#602），尚无自动化契约测试。
