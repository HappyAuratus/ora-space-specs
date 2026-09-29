---
status: approved
---

# Controller 把 operation 的 plugin 步骤派发为 Node 执行，不再调用 Substrate 插件 effect

云端 Controller 推进处于 `plugin` 步骤的 operation 时，不再 `PlanEffect(PluginEnsure/PluginDelete)` 并调用
Substrate，而是像 clone 步骤一样：用 operation 快照中的 `request.plugins` 登记一个 `InstallPlugins` 或
`RemovePlugins` 执行，派发给该 Workspace 当前 generation 的 Node，接管结果后推进或 defer。Controller 不
解释插件载荷，也不决定安装哪些插件。

本 ADR 处于 `approved`（已评审，待实现），是 `controller/plugin` 的根决策（本目录范围：“Plugin 期望状态、Node 安装计划和
运行状态汇总”；第一版只承担安装计划的派发，期望状态与汇总在 Cloud）。继承
[Controller 驱动 Workspace 沙盒](../node-management/0-controller-drives-workspace-sandboxes.md) 的 operation
推进与 clone 步骤规则；步骤语义见 [Cloud operation 后续决策](../../cloud/operation/20260928-plugin-step-and-run-workspace-release.md)，
Node 语义见 [Node plugin-runtime 根决策](../../node/plugin-runtime/0-node-installs-plugins-into-workspace-home.md)。

## 决策

### D1：plugin 步骤与 clone 步骤同构

| 情况 | Controller 行为 |
|---|---|
| 快照中该 operation 没有插件执行，或最近一次已失败 | `RecordDispatch` 新执行（输入 = `request.plugins`，目标 = 当前 Node），然后派发 |
| 有尚无结果的执行 | 查询 Node 状态；有结果则接管，未知则按恢复规则等待或重发原执行 |
| 结果已接管、全部插件有逐项结果 | `AdvanceOperation`（单项失败不阻止推进） |
| 执行失败（`plugins_failed`、Node 不可达超过期限） | `DeferOperation(retry_wait, PLUGIN_EXECUTION_FAILED)` |
| 执行结果未知 | `DeferOperation(blocked, PLUGIN_RESULT_UNKNOWN)`，不自动重试 |

- 目标 Node 没有会话时按 `NODE_UNAVAILABLE` defer，与 clone 步骤一致。
- 执行输入取自快照的 `plugin_input`，已登记的执行取自 `plugin_executions`；快照没有 `plugin_input` 时直接推进。
- Controller 删除 `substrate` 客户端中插件 effect 的请求与证据映射；遇到历史快照中已计划的插件 effect，
  只做对账读取，不再执行。

### D2：Node 必须声明插件能力

只有握手能力包含 `plugin_install` 的 Node 才能被派发插件执行；缺少能力时 defer 为 `blocked`，原因
`EXTERNAL_FAILURE`，避免把执行发给旧版本 Node 后得到无法解析的拒绝。

## 不变量

1. 插件执行的输入等于 Cloud operation 快照中的 `request.plugins`。
2. 插件执行只派发给声明了插件能力、且属于该 Workspace 当前 generation 的 Node。
3. Controller 不计划或执行插件 Substrate effect。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保留 Substrate effect 路径作为回退 | 兼容旧部署；但 Sandbox Server 从未实现它，保留只会留下一条无法验证的分支 |
| Controller 按插件逐个派发 | 结果粒度自然；步骤推进要跟踪多个执行（见 Cloud operation 后续决策） |

## 落地

替换 `steps.rs` 的 `plugin` 步骤实现与 `substrate.rs` 的插件映射；以假 Cloud 与假 Node 的测试覆盖表中五种
情况与能力缺失。

## 运行控制接线（随实现同步）

插件执行继承当前 clone 的 runtime control 围栏：云端使用 `ControlledPlugins` 携带原始插件命令和
Cloud 为该 execution 签发的短期许可；接受与开始落盘前分别校验。未携带许可的插件命令仅用于未启用
runtime control 的本地入口，不能绕过已持久化的围栏。插件执行计入未完成责任，阻止提前报告 idle。

Node 在同一进程内按已接受责任完成安装；本地未围栏执行重启后清理临时目录并重跑。云端 Node 重启后，
旧 incarnation 的许可不授权新的安装变更，因此该执行以 `plugins_failed{interrupted}` 收尾，Cloud
按既有失败重试规则登记新执行并签发新许可。此处收窄原先无条件“重启重跑”的承诺，防止自动恢复绕过
后来引入的运行控制关闭与 incarnation 隔离。
