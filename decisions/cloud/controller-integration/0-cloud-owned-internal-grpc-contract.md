---
status: proposed
---

# Cloud 拥有 Internal Control gRPC 契约并承担权威持久化：`/internal/v1` 升级为 proto 定义的服务，补提交身份幂等、派发登记、Node 事件接管与服务端流

Cloud（Go API Server）是 Controller 内部控制契约的唯一拥有者与服务端：`.proto` 放在 Cloud 仓库的
proto 目录，用 `buf` 以固定版本插件生成 grpc-go 服务端代码并提交；现有 `/internal/v1` JSON 内部
接口的动作集合（租约、领取、快照、effect 规划与结果、推进与延后、Node 上报）按同一语义升级为
gRPC 服务，并补上现状缺失的四项承诺：每次逻辑提交的稳定提交身份与"同身份重传返回原结果"、
派发前登记执行身份与输入、Node 事件结果与收据的原子接管、Controller 拨出的服务端流信号。
Controller 不直接访问 PostgreSQL，也不再拥有云端权威状态；本机 single-node 场景的 SQLite 是
Controller 侧同一持久协调接口的另一个适配器，与本文无关。

本决策处于 `proposed` 状态，是 `cloud/controller-integration` 的根决策。它承接
[Internal Control Protocol 提案](../../controller/protocol/0-internal-control-protocol.md)中
"API Server 是持久化权威、Controller 是 reconcile worker"的原则，回答其未决的 Transport
（gRPC over HTTP/2，Controller 拨出）与提交身份字段；与
[Controller 契约方向后续决策](../../controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)
一起替代被撤回的 specs#37（其中"Cloud 消费 Controller gRPC"的根决策方向相反，"状态码 + `ErrorDetail`"
的错误携带方式在本文 D6 保留）。本文覆盖 Cloud 仓库的 proto 目录、生成包、gRPC 服务端与
`internal/core` 的控制事务；不定义公开 Web API、租户模型、Node 沙盒创建，也不改变现有
`/internal/v1` 的业务事务规则——它们是被升级的对象而不是被重写的对象。

## 依据与边界

| 来源 | 本文处理 |
|---|---|
| Internal Control Protocol D1：双向应用协议 | 采用：Controller→Cloud 为 unary 调用，Cloud→Controller 为服务端流 |
| 同 D2：API Server 是持久化权威；派发前先持久保存执行关联 | 采用为 D3、D4 的服务与事务规则 |
| 同 D3：Controller 只处理租户无关的已授权对象 | 继承；proto 不含租户字段 |
| 同 D4：领取、幂等与恢复；`submission_id` 语义 | 采用并固定为 proto 字段与 Cloud 表 |
| 同 D6：Node 结果确认以 API Server 持久化为边界 | 采用为 D4 的接管服务 |
| Cloud 现有 `/internal/v1`：全局租约、epoch fencing、claim、snapshot、plan／effect_result／advance／defer、node_* | 继承其事务规则与状态机；改为 gRPC 表达，字段与错误枚举化 |
| Cloud 现有服务凭据：`kind="service"`、`role=controller|node` 的 token | 继承；作为 gRPC metadata 携带 |
| Cloud 身份 ADR：Gateway 不代理内部 API | 继承；gRPC 监听器由 API Server 进程直接提供 |

## 决策

### D1：proto 是唯一契约源，Cloud 生成服务端，消费方按 commit 锁定

`.proto` 放在 Cloud 仓库根目录的 `proto/` 下，package 为 `ora.cloud.internal.v1`（目录
`proto/ora/cloud/internal/v1/`）。`buf.yaml` 启用 `STANDARD` lint 与以主干为基线的 `FILE` 级
breaking 检查；`buf.gen.yaml` 用固定版本的 `protocolbuffers/go` 与 `grpc/go` 远程插件生成到
`internal/controlpb`（名称在落地时固定），生成物提交，CI 重新生成后 diff 必须为空。契约不写
`go_package`，由消费方在 managed 模式指定自己的导入路径。

版本规则：`v1` 只接受非破坏性修改；破坏性变更以并列的 `v2` package 表达并保留 `v1`，直到没有
锁定 `v1` 的消费方。消费方（desktop）以 git commit 锁定契约版本；tag 可以为版本命名，但不是锁定
依据。契约 commit 必须位于主干或受保护分支，保证长期可获取。

Cloud 拥有 proto 的原因：服务端是执行事务、校验授权与协调资格的一方，契约随权威状态与数据库
一起演进；把契约放在调用方或第三方仓库都会让服务端跟随他人定义自己的事务边界。

### D2：服务集合以现有 `/internal/v1` 为起点，语义逐项保留

现有 JSON 内部动作按下表映射为 gRPC 方法，事务规则、状态机与错误条件不变；请求体中的
`epoch`、`version` 成为显式字段，字符串错误码成为 `ErrorCode` 枚举（D6）。方法名用于说明语义，
最终以 proto 为准。

| 现有动作 | gRPC 方法 | 保留的规则 |
|---|---|---|
| `lease_acquire`／`lease_renew`／`lease_release` | `ControllerLease.Acquire`／`Renew`／`Release` | 全局单租约、30 秒过期、`epoch` 单调递增；陈旧持有者被 `stale_controller` 拒绝 |
| `claim` | `Operations.Claim` | 领取 `queued`／到期 `retry_wait`／`running` 的最早 operation，写入 `controller_epoch`，返回快照 |
| `snapshot` | `Operations.Snapshot` | operation、project、storage、workspaces、sandboxes、nodes、effects 的一致读取 |
| `plan` | `Operations.PlanEffect` | 同 kind 与 workspace 的 effect 幂等返回；`reconcile_required`、`invalid_step` 等校验不变 |
| `effect_result` | `Operations.ReportEffectResult` | 状态集合与外部身份规则不变 |
| `advance`／`defer` | `Operations.Advance`／`Defer` | 版本校验与延后窗口不变 |
| `access`／`admit` | `Admission.Access`／`Admit` | 执行准入规则不变 |
| `node_register`／`node_status`／`node_idle`／`node_finish` | `Nodes.Register`／`ReportStatus`／`ReportIdle`／`FinishTicket` | `role=node` 凭据；规则不变 |

JSON `/internal/v1` 在 gRPC 服务可用并有真实 Controller 对端验证后退役，退役是独立变更；期间两者
共用 `internal/core` 的同一 `Control` 事务入口，不出现两套业务规则。

### D3：每次逻辑提交携带稳定提交身份，重传返回原结果

所有改变状态的方法请求携带 `submission_id`（调用方生成、全局唯一、对同一逻辑提交在重传时保持
不变）。Cloud 在提交事务内保存 `(submission_id, 请求摘要, 响应)`，并按下表处理：

| 情况 | 行为 |
|---|---|
| 未见过的 `submission_id` | 校验 `epoch`／`version`／业务规则，原子写入状态变化与提交记录，返回结果 |
| 已见过，请求摘要一致 | 返回原响应，不重新应用状态变化，不续期租约 |
| 已见过，请求摘要不同 | `ABORTED` + `CONFLICT`，不覆盖原记录 |
| 新提交但版本／epoch 校验失败 | 拒绝写入，要求重新读取快照或重新获取租约 |

一个必须成立的反例：基于版本 7 的提交成功写入版本 8 但响应丢失，同一 `submission_id` 重传应返回
原成功响应，不能因当前版本已是 8 而报版本冲突。这是现有 `epoch + version` 校验做不到的：它会把
"已提交的重传"当作陈旧写入拒绝，使 Controller 无法区分"从未提交"和"已提交但不知道"。

提交记录按 operation 终态后的保留窗口清理（具体时长在落地时固定，不短于 Controller 的最大重传窗口）；
读取类方法不携带 `submission_id`。

### D4：派发前登记执行身份，Node 事件结果与收据原子接管

为 Controller 向 Node 派发执行与接管 Node 事件，新增以执行为中心的服务（示意名 `Executions`），
它是 Controller 侧持久协调接口在云端的直接对应，第一版只覆盖 clone 闭环：

| 方法 | 事务内保存 | 成功返回的意义 |
|---|---|---|
| `RecordDispatch` | `operation_id`、`execution_id`、目标 Node、完整执行输入、待协调责任 | Controller 现在才可以向 Node 发送该执行；同一 `execution_id` 相同输入幂等，不同输入冲突 |
| `TakeOverNodeEvent` | 执行事实（结果）、事件收据 `(execution_id, sequence, 事件内容)`、必要后续责任 | Controller 现在才可以向 Node 发送该序号的 `EventAck`；同序号相同内容幂等，不同内容冲突 |
| `RecordQueriedResult` | 由查询得到的 Completed 结果，无收据 | 保存结果；不产生任何 Ack 依据 |
| `GetDispatch`、`ListPendingDispatches` | 无 | 恢复原输入与待协调责任；新 worker 用它定位原执行而不是创建第二次执行 |

规则：Cloud 不可用时 Controller 不能派发尚未登记的新执行，也不能确认尚未接管的事件；接管请求
中的 Node 身份、原派发身份与输入必须与登记记录一致，历史事件中的 `NodeIncarnationId` 不被改写；
结果内容冲突拒绝、不覆盖原事实。这些方法同样携带 `submission_id`，并受租约 `epoch` fencing。

Cloud 侧为此新增执行登记与事件收据的表；它们与现有 `operations`／`external_effects` 的关联方式
属于 Cloud 的 `persistence` 决策，本文只要求：登记、接管、提交记录在同一事务提交，响应只在
提交后返回。

### D5：Cloud→Controller 信号经 Controller 发起的服务端流

契约提供 `Watch(WatchRequest) returns (stream ControlSignal)`，Controller 持有有效租约后建立并
在断开后重建。`ControlSignal` 是 oneof：

| 信号 | 含义 | Controller 义务 |
|---|---|---|
| `WorkAvailable` | 有 operation 可领取（可带 operation 身份作提示） | 立即 `Claim`；归属仍由 `Claim` 事务决定 |
| `Drain` | 停止领取，排空或交接在途协调 | 不再 `Claim`；不取消用户任务、不销毁沙盒；排空完成后释放租约 |
| `NodeAssignment` | 某 Node／沙盒的控制归属变化 | 第一版只记录；语义留给 scale 决策 |

流是至多一次、无历史回放的通知：Cloud 不持久化信号，不保证送达；Controller 在流不可用期间按
周期 `Claim`，因此信号丢失只影响延迟。这样 Cloud 不需要为通知维护 outbox，也不必知道 Controller
的地址；需要可靠性的动作都已经是 Controller 发起的 unary 提交。

### D6：错误以 gRPC 状态码为主分类，附 `ErrorDetail` 枚举

失败调用返回 gRPC 规范状态码，并把 `ErrorDetail{ErrorCode}` 作为 `google.rpc.Status` 的 detail
附上；映射在 Cloud 一处产生，状态码与枚举不会不一致。调用方按任一方式分支，不解析消息文本。

| 现有 HTTP 错误 | gRPC 状态 | `ErrorCode` |
|---|---|---|
| `409 stale_controller`、`stale_operation`、`lease_held` | `FAILED_PRECONDITION` | `STALE_CONTROLLER`／`STALE_OPERATION`／`LEASE_HELD` |
| `409 reconcile_required`、`invalid_step`、`termination_unconfirmed`、`maintenance_unconfirmed`、`no_current_sandbox`、`resource_unavailable` | `FAILED_PRECONDITION` | 对应枚举值 |
| 提交身份冲突、内容冲突 | `ABORTED` | `CONFLICT` |
| `400 invalid_*` | `INVALID_ARGUMENT` | `INVALID_INPUT` 或细分值 |
| `403 service_forbidden`、`invalid_effect_scope` | `PERMISSION_DENIED` | 对应枚举值 |
| `404 not_found` | `NOT_FOUND` | `NOT_FOUND` |
| 数据库不可用、事务未开始 | `UNAVAILABLE` | `UNAVAILABLE`（未接受，可重试） |

状态码必须承载分类，因为 grpc-go 重试策略、拦截器、观测与通用工具都以状态码为键；枚举保留以便
细分而不改变状态码。截止时间与取消不附 detail，它们是调用方自己的结果而不是 Cloud 的裁决。
无 detail 的状态（如传输层错误）不是 Cloud 的裁决，Controller 按"未知"处理。

### D7：认证与监听

gRPC 服务由 API Server 进程直接监听（与 HTTP 分端口或复用同一端口在落地时固定），不经 Gateway。
调用方凭据沿用现有服务 token（`role=controller`／`node`），作为 gRPC metadata 携带并在拦截器中
校验，进入 `Control` 事务的 `Service` 主体与今天相同。TLS 与凭据轮换留给认证 ADR；在其落地前，
gRPC 监听器只允许回环或私网地址，与现有内部 HTTP 接口的部署约束一致。

## 不变量

1. Cloud 是 Controller 契约的唯一拥有者与服务端；生成代码与 proto 无漂移，`v1` 不接受破坏性修改。
2. Cloud 是云端业务状态与执行登记的唯一权威写入者；Controller 不访问 PostgreSQL，也不在别处持有权威副本。
3. 每个状态变更提交都有稳定提交身份；同身份同内容返回原结果，同身份不同内容拒绝，重传不续期租约。
4. `RecordDispatch` 成功先于任何 Node 派发；`TakeOverNodeEvent` 成功先于任何 `EventAck`；
   查询得到的结果不构成 Ack 依据。
5. 状态变更受租约 `epoch` 与资源版本校验；陈旧 Controller 不能覆盖新状态。
6. 服务端流不持久化、不保证送达、不改变归属；断流只影响延迟。
7. 契约不含租户、用户、成员或角色字段；授权在 Cloud 公开边界完成。
8. 状态码与 `ErrorDetail` 由 Cloud 一处产生，二者一致；无 detail 的失败不是 Cloud 裁决。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Controller 作为 gRPC 服务端、proto 放 desktop、Cloud 拨入（被撤回的 specs#37 方向） | Cloud 需要知道每个 Controller 的地址并维护其存活；服务端不是持有状态与授权的一方；Controller 无法作为可替换 worker 缩放 |
| 保留 JSON `/internal/v1`，不引入 gRPC | 已能工作，但无类型契约、无生成代码，Rust 侧需手写 DTO 与 Go 平行维护；无双向流，通知只能轮询 |
| proto 放 specs 或独立契约仓库 | specs 不是可锁定的构建输入；独立仓库让每次变更需要三处 PR，当前只有一个消费方 |
| 只用 `epoch + version` 去重，不加 `submission_id` | 少一张表，但无法区分"已提交回复丢失"与"陈旧写入"，Controller 只能重新读快照再猜 |
| Cloud 经流推送工作并以推送为归属 | 省一次领取往返；但至多一次投递使断线时归属不可判定，Cloud 还要为流维护 outbox |
| Controller 拥有独立 PostgreSQL，与 Cloud 双向交接 | Cloud 不可用时仍可协调；但需要两套权威、两向可靠交接与投影规则，Controller 不再是无状态 worker |

## 本决策未解决的问题

- 多 worker 的租约分区、到期恢复与分工；fencing 如何同时约束 Node 变更指令，留给 scale 决策。
- 新 worker 接管原沙盒与原 Node 连接的规则；沙盒服务接口。
- `Executions` 服务与 Cloud 现有 Effect 级 operation 模型的对齐：第一版只服务 clone 闭环，
  worktree、sandbox 等 effect 由 Controller 用现有 `Operations` 方法处理还是并入 `Executions`，留给后续决策。
- TLS、凭据轮换与非私网监听。
- JSON `/internal/v1` 的退役时点。

## 落地顺序

1. **契约与生成链**：`proto/` 与 `buf` 配置、生成包、`proto:lint`／`proto:breaking`／`proto:check`
   与 CI。完成条件：主干上生成无 diff；lint 通过。行为不变：无运行时接入。
2. **gRPC 服务端与提交记录**：拦截器认证、D2 方法直接调用 `Control` 事务、`submission_id` 表与
   D3 规则、D6 错误映射。完成条件：同一事务规则的 gRPC 与 JSON 测试等价；提交回复丢失重传返回原结果。
   行为不变：JSON 接口继续可用。
3. **执行登记与事件接管**：`Executions` 服务及其表（D4）。完成条件：登记先于派发、接管先于确认、
   冲突拒绝、恢复读取以 PostgreSQL 事务测试证明。
4. **服务端流**：`Watch` 与 `WorkAvailable`／`Drain`（D5）。完成条件：断流后 Controller 按周期
   领取仍正确；`Drain` 后不再领取且不取消任务。
5. **真实对端契约测试**：以真实 `ora-controller`（Cloud 适配器模式）为对端，`ORA_CONTROLLER_BIN`
   门控。进入 `approved` 后在 `test-cases/cloud/controller-integration/` 登记上述义务与证据。
