---
status: proposed
---

# Cloud↔Controller 契约改由 Cloud 拥有：Controller 拨出、服务端流下发信号、以 submodule 消费 proto

Cloud↔Controller 的 gRPC 契约不再由本仓库（desktop）的 `proto/` 拥有，而是由 `ora-space/cloud`
仓库的 proto 目录唯一定义；Controller 是该契约的**客户端**，Cloud（Go API Server）是服务端。
调用方向固定为 Controller 拨出：Cloud 不向 Controller 发起连接，需要主动通知 Controller 的信号
（新工作、排空、Node 指派）经同一契约上的服务端流下发。desktop 以 git submodule 锁定 Cloud 的
契约 commit，用 sparse-checkout 只展开 proto 目录，生成 tonic 客户端代码并提交。`ora-controller`
不再需要面向 Cloud 的 gRPC 监听器；根决策 D2 的显式 Transport 监听器只服务过渡 JSON 接口与本机
调用方。

本决策处于 `proposed` 状态。它修改[根决策](0-controller-hosted-client-api.md)的 D4（契约归属与
生成方向）和 D5（调用方向），继承 D1–D3；被撤回的 desktop#597 / cloud#18 / specs#37 曾按相反方向
（Controller 作为服务端、proto 放 desktop）落地契约基础设施，本文以其中可复用的 buf 工具链、生成物
提交与漂移检查、错误分类映射为基础重新定向。契约内容本身由
[Cloud 拥有 Internal Control gRPC 契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)
定义，Controller 侧持久化抽象由
[持久协调接口与双适配器](../persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md)
定义；本文只回答"契约归谁、谁连谁、desktop 怎么拿到 proto"。它不定义认证、TLS，也不改变浏览器
经 Vite proxy 使用的过渡 JSON 接口。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：`ora-controller` 是 API 唯一宿主，库与传输无关 | 继承，不变；面向 Cloud 的 RPC 客户端同样是适配模块，库不依赖 tonic |
| D2：Transport 由启动参数显式选择，默认回环 | 继承，不变；该监听器不再承诺承载 gRPC，只承载过渡 JSON 接口与本机调用方 |
| D3：JSON HTTP 接口是过渡契约，不是 Cloud 契约 | 继承，不变 |
| D4：`.proto` 放本仓库 `proto/`，由 Controller（服务端）拥有；Cloud 以 git tag 消费 | 修改：`.proto` 放 `ora-space/cloud`，由 Cloud（服务端）拥有；desktop 以 submodule 锁定 commit 消费，生成客户端代码 |
| D4：`buf lint`／`buf breaking` 在本仓库执行；破坏性变更以 `v2` package 表达 | 修改归属，规则不变：lint 与 breaking 在 Cloud 仓库执行；desktop 只做生成漂移与 submodule 一致性检查 |
| D4：错误分类以 proto 枚举携带在 status details，Rust `Error` 到枚举的映射只写一次 | 修改方向，规则不变：状态码与 `ErrorDetail` 由 Cloud 产生；Rust 侧把它们映射为持久协调接口的错误分类，映射只写一次 |
| D4：契约不含 tenant、user、membership 字段 | 继承，不变 |
| D5：单集群部署中 API Server 主动拨号 Controller；NAT 场景下的反向连接留作未决项 | 修改：所有部署中 Controller 拨出；Cloud→Controller 信号经服务端流；不再有 Cloud 拨入 Controller 的形态 |
| D5：浏览器面向 Cloud 保持 JSON 与 SSE；不为 PTY 预留 gRPC 服务 | 继承，不变 |

## 决策

### D1：契约唯一源在 Cloud 仓库，desktop 以 submodule 锁定 commit 消费

`.proto` 的唯一来源是 `ora-space/cloud` 仓库中的 proto 目录（具体路径由 Cloud 决策固定）。
desktop 通过 git submodule 引用 Cloud 仓库并把 gitlink 锁定到一个 commit；submodule 挂载在
desktop 的独立目录下（示意为 `third_party/cloud/`，最终路径在落地时固定），用 sparse-checkout
只展开 proto 目录及其 imports，配合 partial clone 减少传输。生成的 tonic **客户端**代码提交到
`ora-controller-proto` crate 的 `src/gen/`，生成器为固定版本的 `buf` 远程插件。

desktop 侧的 CI 与本地检查义务：

| 检查 | 规则 |
|---|---|
| gitlink 一致 | submodule 实际 checkout 的 commit 与 desktop 提交的 gitlink 相同 |
| 无本地修改 | submodule 工作区内契约文件没有未提交修改 |
| 生成可重现 | 用锁定 commit 重新生成后 `src/gen` 无 diff |
| 不做 lint／breaking | `buf lint` 与 `buf breaking` 属于 Cloud 仓库的义务，desktop 不重复执行 |

更新契约的流程：Cloud 合入契约变更并推送 → desktop 切换 submodule 指针到该 commit → 重新生成
→ 修改适配器 → 与生成物、指针一起提交。CI 使用锁定 commit，不自动跟踪 Cloud 分支最新值；正式合入
前引用的 commit 必须长期可获取（主干或受保护分支上的 commit，而非会被 rebase 掉的开发分支）。
Cloud 可以为契约版本打 tag 方便阅读，但锁定依据始终是 commit。

选择 submodule 而不是自定义锁文件加 proto 快照，是因为 gitlink 是 Git 原生的版本锁定，不需要
另写一套"提取、校验、复制"逻辑，也不会出现锁文件与快照不一致的第三种状态。代价是首次开发与
CI 需要初始化 submodule 并具备 Cloud 仓库的读取权限，评审契约变更时需要展开子模块 diff；这两点
由初始化 task 与 PR 描述中链接 Cloud 变更来缓解。

`/specs` 仍是被 `.gitignore` 忽略的独立 checkout，不受 desktop commit 锁定，因此不能作为契约依赖；
proto 也不放在 specs：specs 承载语义与验收义务，不承载构建输入。

### D2：调用方向固定为 Controller 拨出，Cloud→Controller 信号经服务端流

Controller 是 gRPC 客户端，向 Cloud 发起所有调用：领取、读取、提交与心跳类操作都是 Controller
发出的 unary 请求；Cloud 需要主动通知 Controller 的事件通过 Controller 发起的服务端流下发
（示意为 `Watch`，消息集合由 Cloud 决策定义：新工作可领取、排空、Node 指派）。Cloud 在任何部署
中都不拨号 Controller，`ora-controller` 也不为 Cloud 暴露 gRPC 服务。

服务端流的语义边界：

- 流上的信号是**加速器**，不是权威：`WorkAvailable` 只提示 Controller 去领取，工作归属仍由
  领取调用在 Cloud 事务中决定；流断开或信号丢失只会让 Controller 退回按周期领取，不会丢失工作。
- 流断开不改变任何持久状态；Controller 重连后重新建立流，不需要补发历史信号。
- `Drain` 让 Controller 停止领取并排空在途协调；它不取消用户任务、不销毁沙盒，具体缩容规则由
  后续 scale 决策承担，本文只固定"排空信号经此流下发"。

固定拨出方向的原因：Controller 是可替换、可缩放的 worker，它的地址与存活不应成为 Cloud 的
配置项；拨出天然穿越 NAT 与动态寻址，认证只需在一个方向建立；根决策 D5 把反向连接列为未决项，
本文用服务端流回答它，而不是同时维护拨入与拨出两种形态。

由此 `ora-controller` 的进程组合简化：D2 的 Transport 监听器只服务过渡 JSON 接口与本机调用方，
Cloud 部署可以不开监听器；面向 Cloud 的连接是运行时内一个长驻的拨出任务，与 Node 会话重连循环
并列，受同一关闭顺序约束（先停止领取与 API 受理，再关闭 Node 会话，最后断开 Cloud 连接）。

### D3：错误分类由 Cloud 产生，Rust 侧映射一次到持久协调接口

Cloud 以 gRPC 规范状态码作为主分类，并把契约中的 `ErrorDetail` 枚举作为 `google.rpc.Status`
的 detail 附上；映射表由 Cloud 决策拥有。desktop 侧的 Cloud RPC 适配器把状态码与枚举映射为
持久协调接口的错误分类，映射只在适配器内写一次，协调逻辑不感知 gRPC：

| Cloud 返回 | 持久协调接口分类 | Controller 行为 |
|---|---|---|
| `ABORTED` + `CONFLICT` | 冲突：身份或内容与权威记录不符 | 不重试同一调用；不派发、不确认 |
| `INVALID_ARGUMENT` | 冲突（输入非法） | 同上 |
| `NOT_FOUND` | 缺失 | 与"已接受但无终态"区分 |
| `UNAVAILABLE`、连接失败 | 不可用：未接受，可重试 | 保留提交身份重试 |
| 截止时间到期、连接中断且无回复 | 未知：可能已提交 | 保留同一提交身份重传，返回原结果或冲突 |
| `PERMISSION_DENIED`、租约／epoch 陈旧 | 协调资格失效 | 停止使用该资格，重新获取租约 |

"未知"是远程适配器特有的分类：本机 SQLite 事务不会产生"已提交但回复丢失"，因此该分类的处理
（同身份重传）是 RPC 适配器必须单独验证的行为。

### D4：租户边界与过渡接口不变

契约仍只携带 Cloud 已授权的 opaque 身份，不出现 tenant、user、membership 或角色字段；Controller
不做租户筛选，也不解析或校验任何 token——它只以自签的短期服务 token 出示自己的机器身份
（Cloud 根决策 D7）。这一规则从根决策继承，评审 Cloud 侧 proto 变更时同样以此拒绝。本机 SQLite
模式下过渡 JSON 接口、`ora_contracts::controller_api` DTO 与浏览器链路不因本文改变；云端模式下
Controller 不提供该接口，浏览器经 Cloud 公开 API 接入（见 minicloud 后续决策），过渡接口随本机
模式退役的承诺不变。

## 不变量

1. Cloud↔Controller 契约唯一来源是 Cloud 仓库的 proto；desktop 只持有由锁定 commit 生成的客户端代码，
   且生成物与锁定 commit 无漂移。
2. Controller 只拨出，不为 Cloud 暴露服务；Cloud 不持有 Controller 的地址。
3. 服务端流上的信号不构成工作归属或状态变更；丢失信号只影响延迟，不影响正确性。
4. gRPC 状态码与 `ErrorDetail` 由 Cloud 产生；desktop 只在 RPC 适配器内映射一次。
5. 契约与 Controller API 不含租户、用户或成员字段。
6. 过渡 JSON 接口的行为、DTO 与退役承诺不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 维持根决策 D4：proto 放 desktop，Controller 作为服务端，Cloud 拨入 | 与 Cloud 是权威持久化、Controller 是无状态 worker 的架构相反：服务端应是拥有状态与授权的一方；Cloud 还需要维护每个 Controller 的地址与存活 |
| proto 放 specs 仓库 | specs 是独立、未被 desktop 锁定的 checkout，不能作为可重现构建输入；它承载语义而非构建物 |
| 自定义锁文件 + 复制 proto 快照到 desktop | 评审可直接看 proto diff、首次 checkout 无需初始化；但需要自写提取／校验／复制逻辑，并可能出现锁文件、快照、Cloud 三者不一致 |
| Controller 同时暴露一个小 gRPC 服务供 Cloud 拨入（派工、排空） | 需要 Controller 可寻址、双向认证与 TLS；服务端流在单向认证下提供同样的通知能力 |
| 纯轮询、无服务端流 | 实现最简，但派工延迟等于轮询周期，排空只能靠租约过期，无法优雅缩容 |
| Cloud 经流直接推送工作作为归属依据 | 省一次领取往返，但流的至多一次投递使归属无法在断线时判定，与"权威在事务中决定"冲突 |

## 本决策未解决的问题

- Cloud 连接的认证凭据、TLS 与凭据轮换，留给认证 ADR。
- submodule 的最终挂载路径与初始化 task 的具体形式，在落地 PR 中固定。
- `Drain`、`NodeAssignment` 的完整语义及多 worker 租约，留给 scale 决策。
- 过渡 JSON 接口何时退役，仍取决于浏览器经 Cloud 接入的进度。

## 落地顺序

1. **契约与工具链**：Cloud 合入 proto 与 grpc-go 服务端（Cloud 决策）；desktop 加入 submodule、
   初始化 task、`buf` 生成配置与 `ora-controller-proto` 客户端生成物、漂移与一致性检查。
   完成条件：desktop CI 在锁定 commit 上生成无 diff；desktop 不再含 `proto/` 目录与 lint／breaking 任务。
   行为不变：`ora-controller` 的 JSON 接口与 Node 会话。
2. **Cloud RPC 适配器与拨出任务**：按持久协调接口实现 Cloud 适配器与 `Watch` 长驻任务，纳入关闭顺序。
   完成条件：以真实 Cloud gRPC 服务端为对端的契约测试通过，含未知结果重传。
   从此阶段起改变：Cloud 部署模式下 `ora-controller` 不打开 SQLite。
3. **进入 `approved` 后**：改写
   [API 边界核心用例](../../../test-cases/controller/api-boundary/controller-hosted-client-api.md)中
   "Generated Cloud Contracts Preserve The Boundary" 的前置状态与义务（契约归 Cloud、方向为拨出），
   并登记 desktop 侧的一致性与生成证据。
