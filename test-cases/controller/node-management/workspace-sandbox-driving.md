# Controller 驱动 Workspace 沙盒

依据[已批准的 Controller 驱动 Workspace 沙盒决策](../../../decisions/controller/node-management/0-controller-drives-workspace-sandboxes.md)。
本文件登记云端 Controller 维护动态 Node 集合、调用 Substrate、代报 Node 与 quiesce 的承诺；Cloud 侧校验见
[Workspace 操作与 Node 上报用例](../../cloud/controller-integration/workspace-operations-and-node-reports.md)。
证据来自 `ora-space/desktop` 的 `apps/ora-controller`（desktop#619）。实现与 ADR 的已知偏离：Cloud
`ClaimOperation` 一次只交出最早的可运行操作，因此操作依次执行；会话只按领取到的操作快照重建，
Controller 重启后存活沙盒要等其 Workspace 再被领取操作才恢复会话，待 Cloud 提供只读的存活沙盒列表后补齐。

## Dynamic Node Targets Follow Cloud Sandboxes Only

### 风险

Controller 连接不属于当前 generation 的沙盒、自行重建找不到的沙盒，或在终止沙盒时仍保持会话，导致两个
Node 写同一 Workspace 卷或错误上报状态。

### 前置状态

云端 Controller 配置了 `substrate` 段与至多一个静态 Node；Cloud 中有若干未终止 sandbox。

### 触发

sandbox 阶段成功；Controller 重启或租约交接；terminate 阶段开始；某个沙盒的路由返回 `404`；沙盒结果
NodeId 与静态 Node 冲突。

### 必须成立

- `NodeTarget` 只来自 Cloud 快照与 `sandbox_ensure` 结果，端点为 `router_url` 加
  `ate-target-actor: <atespace>/<externalId>`；重启后从快照重建。
- 计划 `sandbox_terminate` 之前已停止该 Node 的会话并移除 `NodeTarget`。
- 路由 `404` 只记录告警并重连，不创建沙盒。
- 与静态 Node 冲突的沙盒 NodeId 被拒绝加入并告警；静态 Node 只接收租户级 clone。
- effect 调用超时后以同一 ID `GET`，不判定不存在。

### 禁止结果

会话仍在时终止沙盒；Controller 自行创建或重建沙盒；同一 NodeId 两个来源同时生效；超时后以新 ID 重试。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 目标来源与重建 | 伪造 Cloud 与 Substrate，重启 Controller 后断言 `NodeTarget` 集合等于快照中的未终止 sandbox | Partial：`src/cloud/fleet.rs` `sandbox_targets_come_from_the_router_and_the_ensure_evidence` 只证明端点与目标来自路由与 ensure 结果；重启后按快照重建尚无证据 |
| 先停会话再终止 | 记录会话结束与 Substrate `PUT` 的顺序 | Covered：`apps/ora-controller/tests/workspaces.rs` `create_and_stop_drive_the_workspace_sandbox_and_its_node` |
| 不自行重建 | 路由返回 `404` 时断言没有 Substrate 写调用 | Missing |
| 超时后查询 | 伪造 Substrate 超时，断言下一次调用是同 ID 的 `GET` | Covered：`src/cloud/substrate.rs` `a_timed_out_effect_is_queried_by_the_same_id` |

### 决策依据

Controller 驱动 Workspace 沙盒 D1、D2，不变量 1–4、7。

## Controller Reports Idle Only After Stopping Dispatch

### 风险

Controller 上报 idle 后又向该 Node 派发工作，stop 在 Node 仍有活动时推进；或会话已断仍上报 connected。

### 前置状态

Workspace 处于 quiesce 阶段，Node 上可能有尚无终态的派发。

### 触发

Controller 执行 quiesce；同时有新的 clone 待派发；会话在上报周期中断开。

### 必须成立

- 先把该 Node 标记为不接受新派发，再检查派发账本；有尚无终态的派发（含未知）时上报 `idle=false`。
- 标记期间不向该 Node 派发；operation 失败或重新开放准入时撤销标记。
- 只在会话存活时上报 `connected`，断开后上报 `disconnected`；incarnation 变化时先 `EndNode` 再 `RegisterNode`。

### 禁止结果

上报 idle 之后出现新派发；存在未知执行时上报 `idle=true`；会话断开后仍上报 `connected`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 停派发与 idle 的顺序 | 同步 quiesce 与新 clone 的开始，断言所有允许的交错下 idle 后无派发 | Missing |
| 未知执行阻止 idle | 构造状态未知的派发，断言上报 `idle=false` | Partial：`apps/ora-controller/tests/workspaces.rs` `quiesce_reports_busy_while_a_dispatch_has_no_result` 只覆盖尚无结果的派发，未构造状态未知的执行 |
| 连接状态与 incarnation | 断开会话与以新 incarnation 重连，断言上报序列 | Partial：`apps/ora-controller/tests/workspaces.rs` `create_and_stop_drive_the_workspace_sandbox_and_its_node` 断言登记先于 clone、主动停止不上报 `disconnected`；断线与新 incarnation 重连尚无证据 |

### 决策依据

Controller 驱动 Workspace 沙盒 D4、D5，不变量 5、6；运行时 Workspace 生命周期 D4、D5。
