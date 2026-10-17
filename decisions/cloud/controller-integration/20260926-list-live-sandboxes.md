---
status: implemented
---

# Internal Control gRPC 增加只读 ListLiveSandboxes，供 Controller 重启后重建 Node 会话

> 后续已实现决策：[运行时控制同步交付](20260927-fenced-runtime-control-delivery.md)增加控制身份验证、运行时操作绑定与旧执行隔离，状态为 `implemented`。完整联合故障矩阵另有验收归属；本文保留原状态，恢复会话能力不等同于多人独占授权。

`WorkspaceOperationService` 以只增不改的方式增加 `ListLiveSandboxes(epoch)`：返回租约持有者应当持有 Node
会话的全部 sandbox，每项带 sandbox 记录、`sandbox_ensure` 报告的 `node_id` 与该 sandbox 未结束的 Node 记录。
它是只读的：不领取 operation、不改变任何 version，只受租约 fencing。

本 ADR 处于 `implemented`，由 ora-space/cloud#31 落地（`proto/ora/cloud/internal/v1/operations.proto`、
`internal/core/live_sandboxes.go`），验证证据见[Workspace 操作与 Node 上报核心用例](../../../test-cases/cloud/controller-integration/workspace-operations-and-node-reports.md#a-restarted-controller-lists-the-live-sandboxes-it-must-reconnect)。
它补齐[Controller 驱动 Workspace 沙盒](../../controller/node-management/0-controller-drives-workspace-sandboxes.md) D2 留下的
“重启后重建全部存活沙盒的会话”。兼容性边界：`v1` 已发布的消息与方法不变（buf breaking 通过）；没有 JSON 路由。

## 继承与修改

| 前序约定（[Workspace 操作与 Node 上报](20260926-workspace-operations-and-controller-reported-nodes.md)） | 本文处理 |
|---|---|
| 写方法以 `epoch` 与 operation `version` fencing，并携带 `submission_id` | 继承；本方法只读，只要求有效租约，不带 `submission_id` |
| 快照只包含被领取 operation 所在 Project 的 sandbox 与 Node | 继承，不变；新增跨 Project 的只读列表（D1） |
| JSON `/internal/v1` 与 gRPC 共享 control 动作 | 修改：`live_sandboxes` 只在 gRPC 上提供，JSON 路由供 Go 模拟器使用，它不需要重建 |

## 决策

### D1：列出“应当持有会话”的 sandbox，而不是全部未终止记录

一个 sandbox 被列出，当且仅当：`terminated_at` 为空、`observed_state` 不是 `terminating`、`generation` 等于其
Workspace 的 `runtime_generation`、ensure effect 已 `succeeded`。每项的 `substrate_sandbox_id` 总有值：sandbox
步骤尚未推进时取 ensure effect 的外部标识。

- 为什么排除 `terminating`：terminate 步骤在计划终止前已主动停止该会话，重连只会被下一轮撤销。
- 为什么要求 ensure 成功：此前 Cloud 不知道握手必须出示的 `node_id`，也没有路由所需的外部标识，无从连接。
- 为什么附带未结束的 Node 记录：Controller 需要据此判断重连后的 incarnation 是否与 Cloud 记录一致，决定
  `RegisterNode` 还是先 `EndNode` 旧行。

### D2：只读，但只对当前租约持有者开放

`epoch` 不是当前有效租约时返回 `FAILED_PRECONDITION`（`STALE_CONTROLLER`）。

- 为什么 fencing 一个读：Node 只绑定一个 Controller，只有持有者应当连接 Node；非持有者拿到列表只会去抢会话。
- 为什么不带 `submission_id`、不领取：重复调用没有副作用，Controller 可以在每个新 epoch 自由重试。

## 不变量

1. 列表调用不改变任何 operation、sandbox 或 Node 记录。
2. 列出的每一项都有非空 `node_id` 与 `substrate_sandbox_id`，且属于当前 generation、未在终止。
3. 只有有效租约可以列出。
4. 契约不含租户字段。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| Controller 本地持久保存沙盒与端点 | 与 Cloud 形成两份事实（Controller 决策已否决） |
| 重启后领取一个伪 operation 触发快照 | 领取会提升 version、占用串行队列，并混淆“需要推进”与“需要重连” |
| 列出所有未终止 sandbox，由 Controller 过滤 | 过滤规则就是 Cloud 的 generation 与终止事实，放在 Cloud 一处维护更不易漂移（D1） |

## 本决策未解决的问题

- 分页：响应一次返回全部；阶段一的沙盒数量远低于 gRPC 默认消息上限，需要时再加分页字段（只增不改）。

## 落地

cloud#31：proto 与生成物、`live_sandboxes` control 动作、gRPC handler、集成测试
`TestGRPCListLiveSandboxesRebuildsSessionTargets`。desktop#621 在 Controller 侧使用它。
