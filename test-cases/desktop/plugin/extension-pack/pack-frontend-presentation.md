# Extension Pack：前端呈现核心用例

对应决策：[Pack 前端呈现决策](../../../../decisions/desktop/plugin/extension-pack/20260916-pack-frontend-presentation.md)
的不变量 1–5，及其继承的根决策 D7 与所有权决策 D1。

前置状态：pack 已通过所有权感知编排安装（或安装失败），宿主设置页渲染插件市场与已装
集合包列表。

## The marketplace projects pack cards with their members and an install entry

- **风险**：pack 卡片若因"无发布资产"被禁用安装，D7 的"安装入口永远存在"即被破坏；若
  成员清单缺失，用户无法理解集合包的内容。
- **触发**：渲染包含 pack 列表的 marketplace 发现网格。
- **必须成立**：
  - pack 卡片携带 `Pack` 徽标与成员清单（来自 `packMembers` 展示字段）；
  - 兼容性投影为可安装（安装走编排），安装按钮可用；
  - 隐藏成员不出现在任何卡片或成员清单中。
- **禁止结果**：pack 卡片呈现为不可安装；隐藏成员出现在卡片上。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 兼容性投影为可安装 | `e2e/tests/plugin_marketplace.rs` `marketplace_plugin_full_lifecycle_walkthrough`（断言 `Compatible`） | Covered |
  | 成员清单与徽标渲染 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` pack 徽标与成员行断言 | Covered |
- **决策依据**：呈现决策 D1、不变量 3。

## Pack install feedback reports members, skips, failures and rollback diagnostics

- **风险**：pack 安装若只用通用成功提示，用户无法知道哪些成员已装、哪些被跳过、是否
  发生回滚残留——回滚失败残余会成为无人认领的安装。
- **触发**：安装 pack 后观察安装反馈；分别覆盖全部成功、部分跳过、失败+回滚残留。
- **必须成立**：
  - 反馈逐成员呈现：已安装成员数、跳过的已存在成员数、失败成员及其错误码；
  - 回滚失败的残余成员作为补充诊断随原始失败一起呈现，不覆盖原始失败；
  - 反馈文案全部来自 `PackInstalled` 契约事实。
- **禁止结果**：回滚失败覆盖原始失败；反馈丢失任一桶的信息。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 逐成员反馈构造（members/skipped/failed/rollback） | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` `renders structured feedback for a pack install outcome` | Covered |
- **决策依据**：呈现决策 D2、不变量 1。

## The installed-packs section presents journal members with their reconciled states

- **风险**：pack 若被伪造成 InstalledPlugin 混入单插件列表，生命周期操作（配置、更新、
  运行状态）会对不存在的包目录操作；若从当前 marketplace 推导已装 pack，registry 更新
  会污染历史关系。
- **触发**：记录存在 pack 安装时渲染已装集合包区。
- **必须成立**：
  - 已装集合包区独立于单插件已安装列表，数据来自所有权日志 + 对账投影；
  - 逐成员呈现 `ExpectedAndPresent` / `VersionChanged` / `Missing` 状态与所有权；
  - pack 不出现在单插件已安装列表中。
- **禁止结果**：pack 伪装成 InstalledPlugin；集合包区从 installed-plugin 目录取数。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 已装集合包区按日志 + 对账渲染 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` `renders the installed packs section from the ownership journal` | Covered |
- **决策依据**：呈现决策 D4、不变量 5。

## The pack uninstall confirmation presents the backend plan verbatim

- **风险**：卸载确认若由前端重算取舍，所有权边界即被破坏；呈现与执行的计划不一致会让
  用户确认一个未发生的操作。
- **触发**：对已记录 pack 发起卸载，打开确认对话框。
- **必须成立**：
  - 确认框按后端计划逐桶呈现 `remove` / `preserve`（附结构化原因）/ `already_missing`；
  - 确认后执行 `uninstallPlugin(pack_id)`，计划与执行来自同一次后端计算。
- **禁止结果**：前端自行增删任何桶；确认框呈现的计划与执行不一致。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 确认框逐桶呈现后端计划 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` `renders the backend plan buckets in the uninstall confirmation` | Covered |
- **决策依据**：呈现决策 D3、不变量 4。
