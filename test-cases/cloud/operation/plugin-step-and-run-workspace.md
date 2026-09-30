# plugin 步骤与运行 Workspace

依据[已批准的 operation 后续决策](../../../decisions/cloud/operation/20260928-plugin-step-and-run-workspace-release.md)。
本文件登记 Workspace 在插件安装后才开放准入、插件不再经 Substrate、运行 Workspace 只由 IssueRun 建删的承诺。
直接证据来自 [cloud#41](https://github.com/ora-space/cloud/pull/41) 的 `integration` 包；真实 Node 插件落盘仍需 M1 联调。

## Workspace Opens Admission Only After The Plugin Step

### 风险

Agent 会话在其 agent 插件安装前启动；或一个损坏的插件让 Space 的所有新 Workspace 都无法就绪。

### 前置状态

Space 有 `desired_state = installed` 的插件，其中一个的 release 无法安装。

### 触发

`create_workspace`、`create_project`、`start` 推进到 `plugin` 步骤；执行结果含成功与失败项；执行整体失败；结果未知。

### 必须成立

- 进入步骤的事务写入 `plugin_input`（期望集合）；集合为空时直接推进。
- 执行输入必须等于 `plugin_input`，目标为当前 generation 的 Node。
- 所有插件都有逐项结果后推进并开放准入；失败项只写入实例的 `install_error`。
- 执行整体失败按 `PLUGIN_EXECUTION_FAILED` 重试为新执行；结果未知保持 `blocked`。

### 禁止结果

插件步骤前开放准入；单项失败阻止就绪；计划 `plugin_ensure`/`plugin_delete` effect。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 步骤表与准入时机 | `integration/agent_run_control_test.go` `TestRunWorkspaceAndPluginStep`：clone 后 plugin 未完成不开放准入，接管结果后 ready；非空插件 start 故障尚需验收 | Partial |
| 单项失败不阻塞 | `TestRunWorkspaceAndPluginStep`：checksum_mismatch 保存在实例，Workspace ready，Space Agent 仍 active；`TestPluginEvidenceMustCoverThePlanExactly` 拒绝重复、缺失及种类不符的结果 | Covered |
| 输入固定 | `TestRunWorkspaceAndPluginStep` 拒绝额外插件输入；尚未直接证明快照后新增 Space 插件不改变输入 | Partial |
| 不再计划插件 effect | `TestRunWorkspaceAndPluginStep` 拒绝新 PlanEffect，控制面也拒绝历史插件 effect 上报；尚缺带在途旧插件 effect 的 0023 升级证据 | Partial |

### 决策依据

operation 后续决策 D1、D2、D3，不变量 1–4。

## Run Workspaces Are Created And Deleted Only By Their IssueRun

### 风险

用户经公开 API 删除运行 Workspace，绕过 Revision 交付；或同一运行创建出两个 Workspace。

### 前置状态

存在一个运行 Workspace（`issue_run_id` 非空）。

### 触发

公开 API 对它 start/stop/delete；IssueRun 派发重复创建；进入 `releasing` 重复删除。

### 必须成立

- 公开 API 返回 `404`，列表中不出现。
- 创建与删除以 `(issue_run_id, kind)` 幂等。
- 仍有未终态执行（包括会话、交付、插件）时 quiesce 失败并按原规则重试。

### 禁止结果

公开 API 改变运行 Workspace；一个运行多个 Workspace。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 公开 API 不可见 | `TestRunWorkspaceAndPluginStep` 覆盖 GET 与列表隐藏；start/stop/delete 共用过滤，尚缺直接断言 | Partial |
| 幂等建删 | `TestRunWorkspaceAndPluginStep` 重复创建只有一行 Workspace/create operation、忙时删除返回 busy；`TestSimulatorServicesEndCommandsWhileRunDeletionIsQueued` 覆盖运行结束后删除与重复删除仅一个 operation | Covered |

### 决策依据

operation 后续决策 D4，不变量 5。
