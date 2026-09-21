# Hook 插件核心测试用例

本目录是 Hook 插件领域的 reviewer 风险导航图，与
[决策目录](../../../../../decisions/desktop/plugin/category/hook) 镜像。

架构边界和不变量来自
[Hook 插件根决策](../../../../../decisions/desktop/plugin/category/hook/0-hook-plugin-pack.md)。

## 当前状态

根决策为 `approved`，实现尚未开始，因此本目录已登记的核心用例暂无敌对证据，全部验证义务标记为
`Missing` 或 `Partial`。实现落地后须通过生产接口取得直接证据并更新证据表，不以用例文档或生成产物
检查代替行为验证。

## 用例索引

| 用例 | 覆盖的承诺 |
| --- | --- |
| [Hook Declarations Must Resolve To A Contained Executable With Declared Lifecycle Commands](hook-lifecycle-execution.md#hook-declarations-must-resolve-to-a-contained-executable-with-declared-lifecycle-commands) | 声明收敛为可执行文件与生命周期能力，包内路径校验与旧字段拒绝 |
| [Hook Lifecycle Commands Must Only Run On Explicitly Authorized Single-Plugin Operations](hook-lifecycle-execution.md#hook-lifecycle-commands-must-only-run-on-explicitly-authorized-single-plugin-operations) | 授权闸门、触发点绑定单插件操作、pack 与回滚不执行 |
| [Hook Lifecycle Execution Must Use The Installed Package Executable In The Minimal Environment](hook-lifecycle-execution.md#hook-lifecycle-execution-must-use-the-installed-package-executable-in-the-minimal-environment) | 执行前复校、最小执行环境、副作用归属 |
| [Hook Lifecycle Failure Must Stay Visible Without Changing Install State Or Blocking Other Flows](hook-lifecycle-execution.md#hook-lifecycle-failure-must-stay-visible-without-changing-install-state-or-blocking-other-flows) | 失败隔离、可重试、超时与截断、记录不持久化 |

## 证据维护

每项义务须通过生产接口取得直接证据，标记为 `Covered`、`Partial` 或 `Missing`；不能仅凭生成类型、
代码存在或依赖组件测试通过升级证据状态。执行结果的记录与前端披露分别验证，权限披露需包含真实
安装路径下的用户可见观察，而不是只验证后端返回值。
