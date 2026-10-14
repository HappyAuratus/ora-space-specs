# 空间插件管理员权限与持久维护等待

已实现服务承诺仍由原 ADR 拥有；下文的 UI 与完整联合故障矩阵改由[后续验收 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)承接。保留本文件与英文标题作为稳定引用，Partial/Missing 不因后端 implemented 而自动提高。

依据[已实现后端的插件维护决策](../../../decisions/cloud/plugin-marketplace/20260927-admin-selection-and-idle-maintenance.md)。新增义务按直接证据登记 Partial/Missing；前序固定版本、摘要校验和租户隔离证据继续保留，不替代真实插件下载、文件及进程的完整故障验收。

## Only Current Tenant Admins Change Space Plugin Selection

- 风险：普通成员改变空间插件，从而修改其他成员运行时；空间汇总泄漏受限日志。
- 前置状态：管理员、普通成员、跨租户用户及已停用管理员；空间含多个运行时和固定插件版本。
- 触发：读取目录与选择，安装/移除/更改版本，直接绕过界面调用；重放已接受请求。
- 必须成立：有效成员可读选择；仅当前管理员可改变期望并记录 actor、版本。拒绝请求无 desired/operation/effect；汇总区分保存、等待、成功和失败，内容详情按运行时权限过滤。
- 禁止结果：改成逐运行时私有选择、成员安装成功、接受即显示全部完成、汇总返回他人执行日志。
- 验证义务与证据：**Partial** — 需真实 HTTP/PostgreSQL 角色及租户矩阵、零副作用和前端汇总测试；原成员插件测试不能证明管理员门控。
- 决策依据：D1、D4，不变量 1、5。

- 本次直接证据：**Partial** — [Cloud integration/plugin_pending_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/plugin_pending_test.go) 的管理员门控、忙时保存 pending；`frontend/src/features/plugins/plugins-page.test.tsx` 覆盖页面角色、版本与汇总。
- 仍缺：跨租户与全部选择/移除/角色变化组合尚需联合覆盖。验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。

## Plugin Maintenance Waits For Human Release And Continues From Durable Pending

- 风险：管理员安装抢占会话、卸载绕过活动保护，或 pending 只能靠再次点击才能继续。
- 前置状态：多个 live 运行时，分别有人类会话、活动/未知执行、已停止、空闲；项目普通 operation 槽位被占用。
- 触发：管理员保存插件期望，释放会话或结束任务，正常启动已停止运行时，重启调度器/Controller。
- 必须成立：待办持久保存；维护在无人持有且无活动/未知执行时取得独占资格，项目槽位不足继续等待。停止实例不为插件自启动；条件满足或恢复扫描后自动继续，无需再次点击。维护期间用户获取返回占用。
- 禁止结果：只检查 ready、闲置人类会话被抢、卸载与 Agent 并行、重启丢待办、重新点击制造副作用。
- 验证义务与证据：**Partial** — 需 PostgreSQL 多运行时调度与真实插件文件/进程测试，覆盖安装、升级、卸载与维护/用户获取竞争。
- 决策依据：D2、D3，不变量 2、3。

- 本次直接证据：**Partial** — [Cloud integration/plugin_pending_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/plugin_pending_test.go) 证明管理员门控、持有人等待及释放后重新处理 pending（执行结果由测试替身提供）；plugins_test.go 的显式模拟维护计划和恢复。
- 仍缺：生产 Node 插件执行已接通；真实安装/移除、下载、未知响应及并发装卸矩阵仍缺，模拟器不替代这些边界。验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。

## Changed Plugin Intent And Unknown Results Reconcile Without Duplicate Execution

- 风险：安装/卸载并行、结果丢失后重新执行，或删除实例被等待队列重新恢复。
- 前置状态：原期望待派发或已派发，Node execution 结果未知；实例可在等待期间被删除。
- 触发：幂等重放、管理员按版本更改期望、恢复查询原 execution、删除目标；原管理员离开。
- 必须成立：未派发按新期望计划，已派发先核对原结果再收敛最新期望；同实例不并发安装卸载。删除保留清理引用并退出候选，接受的维护不随管理员离开消失。无真实执行器时能力不开放、不报告成功。
- 禁止结果：换 execution ID 重试未知结果、旧验证结果覆盖新期望、恢复已删除运行时、隐式自动升级。新 Workspace 补装选中插件的批准后续规则见 [Node 插件安装](node-executed-plugin-installs.md)。
- 验证义务与证据：**Partial** — 需持久待办/版本/删除竞争与真实执行器幂等恢复测试；模拟器 effect journal 只作为前序证据。
- 决策依据：D3、D4，不变量 3–5。

- 本次直接证据：**Partial** — `integration/plugin_intent_recovery_test.go` `TestChangedPluginIntentReconcilesUnknownOldResultBeforeRemoval`：真实 PostgreSQL + 显式 Node 执行替身，核对旧期望未知结果、新移除期望等待、稳定执行重放和唯一后续维护。原 effect journal 是前序规则证据，不证明新的 Node 安装路径。
- 仍缺：真实插件旧期望/迟到结果、下载/进程/丢失响应矩阵仍缺；代码或状态本身不是直接验收。验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。
- 失败终态的直接证据：`TestPluginFailedResultReleasesMaintenanceForNewSelection` 覆盖失败后移除、移除先于旧执行失败、维护绑定释放与最新期望收敛；整体仍为 `Partial`，不替代真实插件执行器证据。
