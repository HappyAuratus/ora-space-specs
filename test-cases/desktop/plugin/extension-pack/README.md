# Extension Pack 核心测试用例

本目录是插件集合包领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/plugin/extension-pack) 镜像。

架构边界和不变量来自
[插件集合包决策](../../../../decisions/desktop/plugin/extension-pack/0-extension-pack.md)。

## 当前状态

根决策尚处于 `proposed`。按 ADR 生命周期，核心测试用例在该决策评审通过并标记为 `approved`
后，依据最终同意的不变量补充；在此之前本目录不登记任何用例。

预计首批核心用例将覆盖的风险面（以决策评审结论为准）：

- preflight 未通过时不发生任何下载，`installed/` 不发生变化。
- 成员安装失败后停止后续成员，已安装成员保留，不自动回滚。
- 已安装成员被跳过且不算失败，其既有版本不被改动或降级。
- `marketplace_visible = false` 的条目不进入派生索引，但仍可按 identifier 解析与安装。
- `agents` 引用按已安装的 agent 插件过滤；引用未命中不是错误，只影响成员取舍。
- pack 条目在旧宿主上整条消失、隐藏成员在旧宿主上仍然可见的兼容行为。
