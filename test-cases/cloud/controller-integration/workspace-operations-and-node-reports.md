# Workspace 操作与 Controller 代报 Node 的 gRPC 契约

依据[已批准的 Workspace 操作与 Node 上报契约决策](../../../decisions/cloud/controller-integration/20260926-workspace-operations-and-controller-reported-nodes.md)。
本文件登记 gRPC 新增方法的授权、fencing 与 Node 身份承诺；步骤语义见
[运行时 Workspace 生命周期用例](../operation/workspace-runtime-lifecycle.md)。实现尚未开始，证据均为 `Missing`。

## Only The Lease Holder Can Report Nodes For The Current Generation

### 风险

失去租约的 Controller、陈旧 generation 或已终止的 sandbox 仍能把 Node 报成 connected，使 Workspace 在
没有可用 Node 时开放准入；或同一 sandbox 同时存在两个活 incarnation。

### 前置状态

Workspace 当前 generation 的 sandbox 已由 `sandbox_ensure` 成功创建，结果 `nodeId = N`。

### 触发

Controller 调用 `RegisterNode`、`ReportNodeStatus`、`EndNode`、`ReportNodeIdle`，分别使用：有效 epoch、
陈旧 epoch、旧 generation 的 sandbox、已终止 sandbox、与 `N` 不同的 `node_id`、第二个 incarnation。

### 必须成立

- 只有有效 epoch 且 sandbox 属于当前 generation、未终止时写入成功。
- `node_id` 必须等于 `N`；按 `(sandbox_instance_id, node_incarnation_id)` 幂等。
- 旧 incarnation 未 `EndNode` 时登记新 incarnation 返回 `ABORTED`/`CONFLICT`。
- 同 `submission_id` 同请求重传返回原结果，不同请求 `CONFLICT`。

### 禁止结果

陈旧 epoch 或陈旧 sandbox 的上报被接受；同一 sandbox 两个未结束 incarnation；`node_id` 与 ensure 结果不符
仍登记成功。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| epoch 与 generation fencing | gRPC 集成测试覆盖上述每种调用方状态，断言状态码与数据库不变 | Missing |
| 单一活 incarnation | 未结束旧 incarnation 时登记新 incarnation，断言冲突；`EndNode` 后成功 | Missing |
| 提交身份 | 写入后丢弃响应重传，断言返回原结果 | Missing |

### 决策依据

Workspace 操作与 Node 上报契约 D2，不变量 3、4；根决策 D3。

## Workspace Clone Dispatch Is Bound To The Workspace State

### 风险

Controller 为 Workspace 的 clone 登记错误的仓库、分支或目标 Node，结果被写到错误的 Workspace。

### 前置状态

Workspace 操作处于 `clone` 步骤，当前 generation 的 Node 已登记。

### 触发

`RecordDispatch` 的 `operation_id` 指向该 Workspace 操作，输入与 `node_id` 分别正确、仓库不符、分支不符、
Node 不是当前 generation。

### 必须成立

- 只有输入等于 Project `repositoryUrl` 与 Workspace `requestedRef`、`node_id` 为当前 Node 时登记成功。
- 同一 operation 的 clone 阶段至多一个进行中执行；`ListPendingDispatches` 返回尚无结果的 Workspace clone。
- `ClaimWork` 不返回 Workspace clone。

### 禁止结果

不符的输入或 Node 被登记；Workspace clone 出现在 `ClaimWork` 中。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 输入与 Node 校验 | gRPC 集成测试覆盖每种不符，断言 `CONFLICT` | Missing |
| 恢复读取覆盖两类执行 | 同时存在租户级与 Workspace clone 时断言 `ListPendingDispatches` 与 `ClaimWork` 的结果 | Missing |

### 决策依据

Workspace 操作与 Node 上报契约 D3，不变量 5；运行时 Workspace 生命周期 D2。
