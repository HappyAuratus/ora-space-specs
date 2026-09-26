---
status: proposed
---

# Internal Control gRPC 增加 Workspace 操作与 Controller 代报 Node，clone 登记扩展到 Workspace 的 clone 步骤

Cloud 的 Internal Control gRPC `v1` 以只增不改的方式增加两个服务：`WorkspaceOperationService` 让 Rust
Controller 领取并推进运行时 Workspace 操作（替代只有 JSON `/internal/v1` 与 Go 模拟器使用的 Effect 级
动作），`NodeReportService` 让 Controller 代 Node 登记、上报心跳、断开与 idle。`ExecutionService` 的
`RecordDispatch` 等方法不改形状，`operation_id` 可以指向 Workspace 操作，使同一套派发前登记与原子接管
覆盖 Workspace 的 clone 步骤。

本 ADR 处于 `proposed`，等待评审纠偏，不维护核心测试用例。它是
[Cloud 拥有 Internal Control gRPC 契约](0-cloud-owned-internal-grpc-contract.md)与
[控制监听器保活](20260924-control-listener-keepalive.md)之后的第三个决策，只描述契约形状与事务边界；
步骤与语义由[运行时 Workspace 生命周期](../operation/0-workspace-runtime-follows-desktop-node.md)决定。
兼容性边界：`v1` 已发布的消息与方法不变（buf breaking 检查必须通过）；JSON `/internal/v1` 保持可用，
直到 Go 模拟器迁移或退役。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D1：proto 是唯一契约源，`v1` 只接受非破坏性修改 | 继承，不变；本文只新增服务、方法与字段 |
| 根 D2：Effect 级动作、执行准入与 Node 上报第一版不进 gRPC，留给后续决策 | 修改：Effect 级操作动作与 Node 上报进入 gRPC（D1、D2）；执行准入（`access`/`admit`）仍只在 JSON |
| 根 D3：写提交携带 `submission_id`，重传返回原结果 | 继承，扩展到本文全部写方法 |
| 根 D4：clone 执行登记、原子接管、恢复读取不要求租约 | 继承；`operation_id` 可指向 Workspace 操作（D3） |
| 根 D5：`Watch` 信号；`NodeAssignment` 第一版不产生 | 修改：新增 `OperationAvailable` 信号（D4）；`NodeAssignment` 仍不产生 |
| 根 D6：错误码映射只在 `internal/controlgrpc` 一处 | 继承，不变 |
| 保活决策：HTTP/2 keepalive 与流的保活 | 继承，不变 |

## 决策

### D1：`WorkspaceOperationService` 一一映射现有 Effect 级事务

| 方法 | 对应 JSON 动作 | 规则 |
|---|---|---|
| `ClaimOperation` | `claim` + `snapshot` | 纯读；返回一个可推进的 operation 及其 Workspace、sandbox、Node、effect 快照；不含 storage/worktree 数据 |
| `PlanEffect` | `plan` | 为当前步骤登记 effect plan，返回 effect ID（即 Substrate 幂等键）与请求体 |
| `RecordEffectResult` | `effect_result` | 登记 Substrate 的观察（含旧 epoch effect 的对账） |
| `AdvanceOperation` | `advance` | 按生命周期决策 D1 的步骤表推进 |
| `DeferOperation` | `defer` | 设置 `retry_wait`/`blocked` 与有限错误码 |

- 事务不重写：每个方法调用 `internal/core` 中与 JSON 动作相同的 `Control` 入口，gRPC 只是另一种表达，
  与根 D2“gRPC 与 JSON 共用同一事务、不存在两套业务规则”一致。
- 写方法（除 `ClaimOperation`）携带 `submission_id` 与 `epoch`，按根 D3 保存提交记录；`operation` 的
  `version` 继续用于 fencing 同一持有者重启前的旧快照。
- effect 请求与结果在 proto 中是按 `kind` 区分的 oneof，只包含生命周期决策保留的三种 effect 与插件
  effect；不在 proto 中出现 storage/worktree 的消息。

### D2：`NodeReportService` 让 Controller 代报 Node

| 方法 | 请求 | 规则 |
|---|---|---|
| `RegisterNode` | `sandbox_instance_id`、`generation`、`node_id`、`node_incarnation_id`、`protocol_version` | 按 `(sandbox_instance_id, node_incarnation_id)` 幂等，返回 Cloud 的 Node 行与 `version`；`node_id` 必须等于 `sandbox_ensure` 结果；同一 sandbox 已有其他未结束 incarnation 时 `ABORTED`/`CONFLICT` |
| `ReportNodeStatus` | Node 行 ID、`version`、`connected`/`disconnected`、`initialized` | 与 JSON `node_status` 同一事务；刷新心跳时间 |
| `EndNode` | Node 行 ID、`version` | 标记该 incarnation 结束（Controller 观察到 Node 以新 incarnation 重连，或 sandbox 已终止），之后才允许登记新 incarnation |
| `ReportNodeIdle` | Node 行 ID、`version`、`operation_id`、`admission_epoch`、`idle` | 与 JSON `node_idle` 同一事务与证据绑定 |

- 身份与授权：调用方是持有租约的 Controller 服务身份；每个写方法带 `epoch` 与 `submission_id`。Cloud
  另外校验 sandbox 属于当前 generation 且未终止——即原 Node 凭据中的 `workspaceId`/`sandboxId`/
  `generation` 绑定，改为由请求字段与数据库状态共同校验。
- 为什么是独立服务：Node 上报与操作推进有不同的调用节奏（心跳每 10 秒一次，操作推进按事件），分开
  让实现可以对心跳做独立的限速与监控；它们仍共用租约与提交身份规则。

### D3：Workspace 的 clone 步骤复用 `ExecutionService`

`RecordDispatch.operation_id` 可以是租户级 clone 请求，也可以是处于 `clone` 步骤的 Workspace 操作；Cloud
依据 ID 所属的表区分。Workspace 操作的 clone：

- 输入必须等于该 Workspace 的 Project `repositoryUrl` 与 `requestedRef`，`node_id` 必须是该 Workspace
  当前 generation 已登记 Node 的 `node_id`；否则 `ABORTED`/`CONFLICT`。
- `TakeOverNodeEvent`/`RecordQueriedResult` 写入结果后，Controller 再调用 `AdvanceOperation`（成功）或
  `DeferOperation`（失败）；结果写入与阶段推进是两次提交，因为推进还要检查租约与 operation version，
  而结果接管必须在向 Node 发送 `EventAck` 之前完成，不能等推进。
- `ClaimWork` 仍只返回租户级 clone 请求；Workspace 的 clone 由 `ClaimOperation` 领取。
- `ListPendingDispatches` 同时返回两类尚无结果的执行，使恢复流程不区分来源。

### D4：`Watch` 增加 `OperationAvailable`

Workspace 操作进入 `queued` 或到期 `retry_wait` 的事务提交之后，Cloud 在 `Watch` 流上发送
`OperationAvailable{operation_id}`，语义与 `WorkAvailable` 相同：至多一次、无历史回放，只影响延迟；
Controller 在流不可用期间按周期调用 `ClaimOperation`。

## 不变量

1. `v1` 的已发布消息与方法不变，新增内容通过 buf breaking 检查。
2. 每个 gRPC 写方法与其 JSON 对应动作走同一个 `internal/core` 事务。
3. Node 的登记、心跳、断开与 idle 只能由持有有效租约 epoch 的 Controller 写入，且只能针对当前
   generation 未终止的 sandbox。
4. 同一 sandbox 同时至多一个未结束的 Node incarnation；新 incarnation 必须在旧 incarnation 结束后登记。
5. Workspace 的 clone 执行输入与目标 Node 由 Cloud 按 Workspace 状态校验，不由 Controller 自选。
6. proto 中不出现 storage/worktree effect。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| Rust Controller 直接调用 JSON `/internal/v1` | 根 D1 选择 proto 作为 Controller 契约，JSON 没有提交身份；两个协议并行会让 Controller 同时实现两套重试语义 |
| 以 `v2` package 表达 | 所有改动都是新增，`v1` 可以承载；`v2` 只在破坏性变更时需要（根 D1） |
| Node 上报并入 `WorkspaceOperationService` | 调用节奏与监控需求不同（D2）；合并不减少事务 |
| Workspace clone 另建 `WorkspaceCloneService` | 重复派发前登记与接管规则（D3） |
| 结果接管时顺带推进 operation | 接管必须先于 `EventAck`，推进需要额外的租约与版本检查，合并会让 Ack 等待推进的失败重试（D3） |

## 本决策未解决的问题

- 执行准入 `access`/`admit` 何时进入 gRPC：取决于 Controller 何时承担任务与交互执行。
- JSON `/internal/v1` 与 Go 模拟 Controller 的退役。
- 多 Controller worker 的分工与 `NodeAssignment`。

## 落地顺序

1. proto 新增服务、方法与字段，重新生成并提交；buf lint/breaking 通过。
2. `internal/controlgrpc` 实现，映射到 `internal/core` 现有与生命周期决策新增的事务；gRPC 集成测试覆盖
   每个方法的幂等、冲突与 fencing。
3. desktop 按 commit 锁定新契约（Controller 决策）。

保持不变的行为：现有 gRPC 方法、JSON 接口与租户级 clone 闭环。从第 2 步开始，Controller 可以经 gRPC
推进 Workspace 操作与代报 Node。
