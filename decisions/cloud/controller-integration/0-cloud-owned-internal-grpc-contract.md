---
status: implemented
---

# Cloud 拥有 Internal Control gRPC 契约并承担权威持久化：租约与 clone 执行登记经 proto 定义的服务提供，补提交身份幂等、派发登记、Node 事件接管与服务端流

Cloud（Go API Server）是 Controller 内部控制契约的唯一拥有者与服务端：`.proto` 放在 Cloud 仓库
`proto/ora/cloud/internal/v1/`，用 `buf` 以固定版本插件生成 grpc-go 代码并提交。第一版契约提供三个
服务：租约（`ControllerLeaseService`）、以执行为中心的 clone 协调（`ExecutionService`）与 Cloud→Controller
信号流（`ControlSignalService`）；它们与现有 `/internal/v1` JSON 接口共用 `internal/core` 的同一 `Control`
事务入口，并补上 JSON 接口缺失的四项承诺：每次逻辑提交的稳定提交身份与"同身份重传返回原结果"、
派发前登记执行身份与输入、Node 事件结果与收据的原子接管、Controller 拨出的服务端流信号。Controller
不访问 PostgreSQL，也不拥有云端权威状态。

本决策处于 `implemented` 状态，是 `cloud/controller-integration` 的根决策。契约与生成链已由
ora-space/cloud#19（`535ca76c`）合入 Cloud 主干；gRPC 服务端、执行登记、服务端流、公开 clone 入口与
调用方身份声明由 ora-space/cloud#20–#24 按栈实现（证据核对栈顶 `9b33596d`，PR 尚未合并），desktop
侧客户端与 Cloud 适配器见
[Controller 契约方向后续决策](../../controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)
与[持久协调双适配器](../../controller/persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md)。
验证义务与证据登记在
[核心用例](../../../test-cases/cloud/controller-integration/cloud-owned-internal-grpc-contract.md)。
本文承接 [Internal Control Protocol 提案](../../controller/protocol/0-internal-control-protocol.md)
"API Server 是持久化权威、Controller 是 reconcile worker"的原则，回答其未决的 Transport（gRPC over
HTTP/2，Controller 拨出）与提交身份字段。它覆盖 Cloud 的 proto 目录、生成包 `internal/controlpb`、gRPC
服务端 `internal/controlgrpc` 与 `internal/core` 的 clone 协调事务；不定义租户模型、Node 沙盒创建，
也不改变现有 `/internal/v1` 的业务事务规则。

## 依据与边界

| 来源 | 本文处理 |
|---|---|
| Internal Control Protocol D1：双向应用协议 | 采用：Controller→Cloud 为 unary 调用，Cloud→Controller 为服务端流 |
| 同 D2：API Server 是持久化权威；派发前先持久保存执行关联 | 采用为 D3、D4 的服务与事务规则 |
| 同 D3：Controller 只处理租户无关的已授权对象 | 继承；proto 不含租户字段 |
| 同 D4：领取、幂等与恢复；`submission_id` 语义 | 采用并固定为 proto 字段与 `control_submissions` 表 |
| 同 D6：Node 结果确认以 API Server 持久化为边界 | 采用为 D4 的接管方法 |
| Cloud 现有 `/internal/v1`：全局租约、epoch fencing、Effect 级 operation 动作、node_* | 租约改为 gRPC 表达，事务不变；其余动作第一版保持 JSON（D2） |
| Cloud 身份 ADR：Gateway 不代理内部 API | 继承；gRPC 监听器由 API Server 进程直接提供 |

## 决策

### D1：proto 是唯一契约源，Cloud 生成服务端，消费方按 commit 锁定

契约位于 `proto/ora/cloud/internal/v1/`（package `ora.cloud.internal.v1`），分为 `lease`、`executions`、
`signals`、`errors` 四个文件。`buf.yaml` 启用 `STANDARD` lint 与以目标分支为基线的 `FILE` 级 breaking
检查；`buf.gen.yaml` 以固定版本的 `protocolbuffers/go` 与 `grpc/go` 远程插件生成到 `internal/controlpb`，
生成物提交。Cloud CI 执行 lint、breaking 与"重新生成后无 diff"。契约不写 `go_package`，由各消费方在
managed 模式下指定自己的导入路径。

版本规则：`v1` 只接受非破坏性修改；破坏性变更以并列的 `v2` package 表达并保留 `v1`，直到没有锁定
`v1` 的消费方。消费方以 git commit 锁定契约，该 commit 必须位于 Cloud 主干或受保护分支。

Cloud 拥有 proto，因为服务端是执行事务、校验协调资格的一方，契约应随权威状态与数据库一起演进；
放在调用方或第三方仓库会让服务端跟随他人定义自己的事务边界。

### D2：第一版契约只覆盖租约与 clone 闭环，Effect 级动作保持 JSON

| 服务 | 方法 | 规则 |
|---|---|---|
| `ControllerLeaseService` | `AcquireLease`／`RenewLease`／`ReleaseLease` | 与 JSON `lease_*` 同一事务：全局单租约、30 秒过期、`epoch` 单调递增；其他持有者 `LEASE_HELD`，陈旧 epoch `STALE_CONTROLLER`；续期保持 epoch |
| `ExecutionService` | `ClaimWork`、`RecordDispatch`、`TakeOverNodeEvent`、`RecordQueriedResult`、`GetDispatch`、`ListPendingDispatches` | clone 闭环（D4） |
| `ControlSignalService` | `Watch` | 服务端流（D5） |

Cloud 现有的 Effect 级 operation 动作（`claim`／`snapshot`／`plan`／`effect_result`／`advance`／`defer`）、
执行准入（`access`／`admit`）与 Node 上报（`node_*`）**不在第一版契约中**，继续只经 JSON `/internal/v1`
提供。这偏离了提案中"逐项映射全部 `/internal/v1` 动作"的计划：第一版唯一的真实 Controller 对端只跑
clone 闭环，Controller 侧也没有消费 Effect 级动作的实现，提前把它们搬进 `v1` 只会冻结一组无人验证的
消息形状。它们何时迁移、是否并入 `ExecutionService`，留给后续决策。gRPC 与 JSON 共用 `Store.Control`，
不存在两套业务规则；JSON 接口的退役是独立变更。

### D3：每次写提交携带稳定提交身份，重传返回原结果

`RecordDispatch`、`TakeOverNodeEvent`、`RecordQueriedResult` 携带 `submission_id`（调用方生成、对同一
逻辑提交在重传时不变）。Cloud 在同一事务内保存 `(submission_id, 持有者, 请求摘要, 响应)`，摘要覆盖
动作、持有者与请求体：

| 情况 | 行为 |
|---|---|
| 未见过的 `submission_id` | 校验租约与业务规则，原子写入状态变化与提交记录，返回结果 |
| 已见过，摘要一致 | 返回记录的响应，不重新应用状态变化 |
| 已见过，摘要不同 | `ABORTED` + `CONFLICT`，原记录不变 |

必须成立的反例：写入成功但回复丢失后，同一 `submission_id` 重传返回原成功响应，而不是因当前状态已
前进而报冲突——这是仅靠 `epoch + version` 做不到的，它无法区分"从未提交"与"已提交但不知道"。

`ClaimWorkRequest` 同样带有 `submission_id` 字段，但领取是纯读（D4），Cloud 不为它保存提交记录；读取类
方法不携带提交身份。

**提交记录当前不清理**。提案要求提交记录按不短于 Controller 重传窗口（首版 24 小时）的保留期清理，
并以清理落地作为本文进入 `implemented` 的前提；最终实现保留全部记录。取消该前提的理由：清理只决定
"多久以前的重传仍能取回原结果"，不清理等于保留窗口无限长，不影响回放与冲突判断的正确性；第一版只有
clone 闭环写入，每次 clone 仅数条小记录，在当前的开发与 minicloud 规模下增长可忽略。表的有界性是生产
部署的前提，与 Controller 认证、多 worker 一起留给后续决策（见未决问题）。

### D4：派发前登记执行身份，Node 事件结果与收据原子接管

clone 在第一版是**独立的已接受工作项**：`clone_requests`（租户、提交用户、`request_id`、仓库、分支、
状态 `queued → dispatched → succeeded | failed`）由 Cloud 在自己的业务事务中接受（D8），不挂在 Effect 级
`operations` 状态机上。协调写入三张表：`clone_executions`（每请求至多一个执行）、`clone_event_receipts`
（`(execution_id, sequence)` 与原事件）与 `control_submissions`（D3）。

| 方法 | 事务内保存 | 成功返回的意义 |
|---|---|---|
| `ClaimWork` | 无（纯读） | 返回最早的 `queued` 请求；归属不在此决定，Controller 在领取与登记之间死亡不留下中间状态 |
| `RecordDispatch` | `execution_id`、目标 Node、完整输入、登记时的 `epoch`；请求 `queued → dispatched` | Controller 现在才可以向 Node 发送该执行；同一执行同输入幂等，输入与请求不符、同一请求的第二个执行、复用他人 `execution_id` 均冲突 |
| `TakeOverNodeEvent` | 执行结果与事件收据；请求进入终态 | Controller 现在才可以向 Node 发送该序号的 `EventAck`；同序号同内容幂等，不同内容冲突 |
| `RecordQueriedResult` | 查询得到的结果，无收据 | 结果可查询；不产生任何 Ack 依据 |
| `GetDispatch`、`ListPendingDispatches` | 无（恢复读取） | 恢复原输入与**尚无结果**的执行；缺失为 `NOT_FOUND` |

规则：

- 每个请求恰有一个执行；明确失败后的用户重试是新的请求。
- 结果中的 Node 必须是登记的目标 Node；已有结果时不同内容的结果被拒绝、不覆盖；历史结果中的
  `NodeIncarnationId` 原样保存。
- 写方法与 `ClaimWork` 受租约 `epoch` fencing；**恢复读取不要求持有租约**：替换 worker 必须先定位原执行
  才能决定是否接手，读不改变归属或状态。
- 输入与结果以固定 JSON 形状落库（输入 `{kind, repositoryUrl, branch}`；结果
  `{node{nodeId, nodeIncarnationId}, outcome, path/commit | reason, retainedPath?}`），幂等与冲突按该形状比较。
- 登记、接管、提交记录在同一事务提交，响应只在提交后返回。

### D5：Cloud→Controller 信号经 Controller 发起的服务端流

`Watch(WatchRequest{epoch})` 返回 `stream WatchResponse`（oneof `WorkAvailable`／`Drain`／`NodeAssignment`）。
建立流需要当前租约 epoch，陈旧 epoch 以 `STALE_CONTROLLER` 结束。

| 信号 | 产生时机 | Controller 义务 |
|---|---|---|
| `WorkAvailable` | 新 clone 请求的接受事务提交之后，携带 `operation_id` 作提示；幂等重复提交不再发 | 立即 `ClaimWork`；归属仍由 `RecordDispatch` 决定 |
| `Drain` | Cloud 关停前；之后流正常结束，新的 `Watch` 以 `UNAVAILABLE` 拒绝 | 不再领取；不取消用户任务、不销毁沙盒 |
| `NodeAssignment` | 第一版不产生 | 语义留给 scale 决策 |

流是至多一次、无历史回放的通知：信号不持久化，每个订阅者的缓冲有界，满时丢弃新信号而不阻塞接受
事务。Controller 在流不可用期间按周期领取，信号丢失只影响延迟。当前 desktop 的 Controller 只做周期
领取、尚未消费该流（见 Controller 契约方向后续决策），这不改变本条对 Cloud 的承诺。

### D6：错误以 gRPC 状态码为主分类，附 `ErrorDetail` 枚举

失败调用返回 gRPC 规范状态码，并把 `ErrorDetail{ErrorCode}` 作为 `google.rpc.Status` 的 detail 附上；
映射只在 `internal/controlgrpc` 一处由 `core.Fault` 产生，状态码与枚举不会不一致。

| Cloud 裁决 | gRPC 状态 | `ErrorCode` |
|---|---|---|
| `lease_held` | `FAILED_PRECONDITION` | `LEASE_HELD` |
| `stale_controller`、`stale_operation` | `FAILED_PRECONDITION` | `STALE_CONTROLLER` |
| 其余 `409`（提交身份、派发、结果、收据冲突及 Effect 级业务前置条件） | `ABORTED` | `CONFLICT` |
| `400 invalid_*` | `INVALID_ARGUMENT` | `INVALID_INPUT` |
| `403` | `PERMISSION_DENIED` | `SERVICE_FORBIDDEN` |
| `404 not_found` | `NOT_FOUND` | `NOT_FOUND` |
| 数据库不可用等非业务错误 | `UNAVAILABLE`（消息不泄露内部细节） | `UNAVAILABLE` |

最终枚举只有上表的七个值，`FAILED_PRECONDITION` 只表示协调资格失效。这偏离了提案中"为每个现有 HTTP
错误码建立同名枚举、业务前置条件也用 `FAILED_PRECONDITION`"的计划：第一版契约不含 Effect 级动作（D2），
那些细分码没有调用方；让 `FAILED_PRECONDITION` 专指资格失效，调用方只看状态码就能决定"重新获取租约"
还是"放弃该调用"。缺少调用方身份时返回无 detail 的 `INVALID_ARGUMENT`；无 detail 的状态（含传输层错误）
不是 Cloud 的业务裁决。

### D7：调用方身份与监听

gRPC 服务由 API Server 进程在独立地址（`control.grpc_addr`，必填、无默认值）监听，不经 Gateway。

当前项目阶段只保证业务语义，不建立机器身份：Controller 不出示凭据，Cloud 不验签。每次调用以
`x-ora-controller-id` metadata 声明自身持久 `ControllerId`（缺失、为空或超过 256 字节时
`INVALID_ARGUMENT`），unary／stream 拦截器把它构造成 `role=controller` 的服务主体交给 `Control`；租约
持有者、提交记录持有者与 epoch fencing 的语义不变。

声明不是证明：能连到监听地址的进程都可以冒名任一 `ControllerId`，因此部署必须把该地址限制在回环或
私网，与现有内部 HTTP 接口的部署约束一致；这是部署约束，Cloud 不在代码中校验地址范围。Controller
认证（凭据形态、TLS、轮换）需要时另立 ADR 替换本条，不改变契约的业务字段。

### D8：clone 请求的公开入口（第一版）

浏览器或其他调用方经 Gateway 会话调用 Cloud 公开 API：

- `POST /api/v1/tenants/{tid}/clones`（body `requestId`、`repository`、`branch`；必须带 `Idempotency-Key`）
  返回 `202` 与 pending 视图；
- `GET /api/v1/tenants/{tid}/clones`（分页）与 `GET /api/v1/tenants/{tid}/clones/{operationId}`。

视图字段与 desktop 过渡 JSON 接口逐字对应：`operationId`（即 `clone_requests.id`）、`requestId`、`repository`、
`branch`、`executionId`／`nodeId`（登记派发前为 `null`）、`state` 为
`pending | succeeded{path, commit} | failed{reason, retainedPath?}`（`reason` 为过渡 DTO 的 camelCase 值）。
`(租户, 提交用户, requestId)` 是持久请求身份：同输入跨幂等键返回原请求，不同输入 `409`；幂等键本身
仍按公开 API 规则拒绝改变的 body。请求只对提交它的用户可见，与 operation 的可见性一致。

接受时 Cloud 以 Controller 的 clone 输入规则校验仓库地址与分支，拒绝任何 Controller 都无法派发的请求：
否则请求会永远停在 `queued`，调用方看到的是"待协调"而不是失败。

这是 clone 闭环第一版的入口，因只服务本契约而放在本文；公开 API 的根决策建立后由 `cloud/api-boundary`
承接。

## 不变量

1. Cloud 是 Controller 契约的唯一拥有者与服务端；生成代码与 proto 无漂移，`v1` 不接受破坏性修改。
2. Cloud 是云端业务状态与执行登记的唯一权威写入者；Controller 不访问 PostgreSQL。
3. 每个状态变更提交都有稳定提交身份；同身份同内容返回原结果，同身份不同内容拒绝且不覆盖原记录。
4. `RecordDispatch` 成功先于任何 Node 派发；`TakeOverNodeEvent` 成功先于任何 `EventAck`；查询得到的结果
   不构成 Ack 依据；`ClaimWork` 不改变归属。
5. 每个 clone 请求至多一个执行；结果冲突不覆盖原事实。
6. 状态变更受租约 `epoch` 校验，陈旧 Controller 不能写入；恢复读取不受租约约束。
7. 服务端流不持久化、不保证送达、不改变归属；断流只影响延迟。
8. 契约不含租户、用户、成员或角色字段；授权在 Cloud 公开边界完成。
9. 状态码与 `ErrorDetail` 由 Cloud 一处产生、二者一致；`FAILED_PRECONDITION` 只表示协调资格失效。
10. 当前阶段 `ControllerId` 是 metadata 中的声明，不是经过认证的身份；监听地址只能位于回环或私网。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Controller 作为 gRPC 服务端、proto 放 desktop、Cloud 拨入（被撤回的 specs#37） | Cloud 要知道每个 Controller 的地址并维护其存活；服务端不是持有状态与授权的一方 |
| 保留 JSON `/internal/v1`，不引入 gRPC | 已能工作，但无类型契约，Rust 侧要手写 DTO 与 Go 平行维护；无服务端流，通知只能轮询 |
| 第一版就迁移全部 `/internal/v1` 动作 | 契约一次完整；但 Effect 级动作没有 Controller 对端，`v1` 会冻结未经验证的消息形状 |
| proto 放 specs 或独立契约仓库 | specs 不是可锁定的构建输入；独立仓库让每次变更需要三处 PR，当前只有一个消费方 |
| 只用 `epoch + version` 去重，不加 `submission_id` | 少一张表，但无法区分"已提交回复丢失"与"陈旧写入" |
| Cloud 经流推送工作并以推送为归属 | 省一次领取往返；但至多一次投递使断线时归属不可判定，Cloud 还要为流维护 outbox |

## 本决策未解决的问题

- `control_submissions` 的保留窗口与清理任务；进入生产部署前必须补齐，窗口不短于 Controller 的最大重传窗口。
- Effect 级 operation、执行准入与 Node 上报迁入契约的时点与形状，以及 `ExecutionService` 与 Effect 级
  operation 模型的对齐。
- 多 worker 的租约分区、到期恢复与分工；fencing 如何同时约束 Node 变更指令；`NodeAssignment` 的语义。
- Controller 认证、TLS 与非私网监听。
- JSON `/internal/v1` 的退役时点。

## 落地结果

- 契约与生成链（cloud#19，已合入）：四个 proto 文件、`internal/controlpb` 生成物、CI 中的 lint／breaking／
  生成漂移检查。
- gRPC 服务端与租约（cloud#20）、执行登记与提交记录（cloud#21）、`Watch` 与关停前 `Drain`（cloud#22）、
  公开 clones API 与 OpenAPI／前端生成客户端（cloud#23）、以 metadata 声明 `ControllerId` 取代服务凭据
  拦截器（cloud#24），均以 PostgreSQL 集成测试验证。
- 与提案不同的最终决定：契约范围收窄为租约与 clone 闭环（D2）；错误枚举收敛为七个值，业务冲突统一为
  `ABORTED`（D6）；`ClaimWork` 字段含提交身份但不记录（D3）；提交记录暂不清理（D3）。
- 以真实 `ora-controller`（Cloud 适配器）为对端的端到端链路已在本机手工验证（见 desktop#602、#610），
  尚无自动化的跨仓库契约测试。
