# plugin 步骤的派发

依据[已批准的 Controller plugin 根决策](../../../decisions/controller/plugin/0-plugin-installs-as-node-executions.md)。
证据预期来自 desktop `apps/ora-controller` 以假 Cloud（`test-server`）与假 Node 的测试；实现尚未开始，全部为 `Missing`。

## Plugin Step Dispatches Cloud's Input To A Capable Node

### 风险

Controller 自行决定安装内容，或把插件执行发给不支持的旧 Node，得到无法解析的拒绝。

### 前置状态

Cloud 的 operation 快照处于 `plugin` 步骤，带 `plugin_input`。

### 触发

快照中无执行、有未结果执行、有已接管结果、执行整体失败、结果未知；目标 Node 未声明 `plugin_install`；快照无 `plugin_input`。

### 必须成立

- 登记的执行输入与 `plugin_input` 逐字段相同，目标为当前 generation 的 Node。
- 结果已接管后 `AdvanceOperation`，单项失败不影响推进。
- 执行整体失败 `DeferOperation(retry_wait, PLUGIN_EXECUTION_FAILED)`；未知 `DeferOperation(blocked, PLUGIN_RESULT_UNKNOWN)`。
- Node 未声明能力时 `blocked`/`EXTERNAL_FAILURE`，不派发；无 `plugin_input` 时直接推进。
- 不再计划或执行插件 Substrate effect。

### 禁止结果

修改或拼装安装载荷；向无能力的 Node 派发；调用 Substrate 插件 effect。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 五种快照情况 | 假 Cloud + 假 Node 测试：逐一断言 Controller 的 gRPC 调用序列 | Missing |
| 能力门槛 | 测试：Node 握手不含 `plugin_install` 时无派发并 `blocked` | Missing |

### 决策依据

Controller plugin 根决策 D1、D2，不变量 1–3。
