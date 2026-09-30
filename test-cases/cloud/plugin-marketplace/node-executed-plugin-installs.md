# 插件由 Node 执行安装

依据[已批准的 plugin-marketplace 后续决策](../../../decisions/cloud/plugin-marketplace/20260928-node-executes-plugin-installs.md)。
既有 fan-out 与聚合用例见 [plugin-marketplace.md](plugin-marketplace.md)。以下证据来自
[cloud#41](https://github.com/ora-space/cloud/pull/41)，真实插件目录的落盘仍需 M1 联调。

## Plugin Instance State Is Written Only From Node Execution Results

### 风险

实例被标为 `installed` 但 Node 上并没有该版本；或 Space Agent 行与 agent 插件的选择不一致。

### 前置状态

插件 operation 或 Workspace 的 plugin 步骤已登记执行。

### 触发

接管结果：`installed{version}` 与计划版本相同/不同、`failed{reason}`、`removed`；agent 类插件的 Space 聚合变为 `installed` 或 `removed`。

### 必须成立

- 实例状态只在接管执行结果的事务中写入；`installed` 的版本不等于计划版本时以 `invalid_plugin_evidence` 拒绝整个结果。
- 失败原因为有界错误码，写入 `install_error`。
- agent 插件聚合为 `installed` 时同一事务创建或恢复 `space_agents` 行；移除时置为 `retired`，不删除。

### 禁止结果

版本不符的结果被接受；Agent 行因某个 Workspace 实例失败而消失。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 版本证据校验 | `integration/plugins_effect_test.go` 的版本不符结果测试；`agent_run_boundary_test.go` `TestPluginEvidenceMustCoverThePlanExactly` 验证集合完整且种类一致，拒绝时实例不变 | Covered |
| Agent 行跟随 Space 选择 | `TestRunWorkspaceAndPluginStep` 覆盖 installed、单实例失败及 retired；`TestAgentSelectionSurvivesCatalogRemoval` 覆盖空 Space 选中和目录消失后移除 | Covered |
| 新 Workspace 补装 | `TestRunWorkspaceAndPluginStep`：选中 agent 后新 Workspace 固定 plugin 输入含该插件；尚需多插件完整集合与真实 Node 目录证据 | Partial |
| 插件执行运行许可 | `TestPluginExecutionReceivesAFencedRuntimePermit`：当前执行取得许可，旧 epoch 与已终态拒绝 | Covered |

### 决策依据

plugin-marketplace 后续决策 D1、D2、D3，不变量 1–4。
