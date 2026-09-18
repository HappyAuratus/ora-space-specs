---
status: implemented
---

# Pack 前端呈现：投影后端事实，不重算所有权

Extension Pack 的所有权日志、重启对账与卸载计划全部由后端持有（D3-A/B/C）。本决策将
它们投影为前端呈现，并钉住两条边界：pack 卡片**永远呈现安装入口**（根决策 D7）；前端
**不重算所有权**——成员的归属、版本事实与卸载取舍全部消费后端投影。

承担模块：`crates/contracts`（`PackInstallationStatus` / `PackUninstallPlan` 查询契约）、
`crates/backend`（journal + reconcile 的只读投影）、`packages/app-shell`（安装反馈、已装
集合包列表、卸载确认对话框）。

## 继承与修改

| 前序决策的约定 | 本决策 |
| --- | --- |
| pack 卡片永远呈现安装入口（根决策 D7） | 继承并落地：pack 兼容性投影为可安装（安装走编排而非制品下载） |
| 所有权日志是唯一所有权来源（D3-A） | 继承；`AvailablePlugin.packMembers` 仅为展示数据 |
| 对账只读、识别不修复（D3-B） | 继承；前端呈现的对账状态是同一投影 |
| 卸载计划先于执行（D3-C） | 继承；确认对话框呈现计划后执行 |

## 决策

### D1 pack 卡片的身份与成员呈现

pack 卡片携带 `Pack` 徽标与成员清单（来自索引的 `pack_members` 展示字段）。兼容性投影
为可安装：pack 的安装入口永远可用，重装等价于补齐缺失成员（根决策 D7）。

### D2 安装反馈逐成员呈现

`PackInstalled` 结果按成员呈现：已安装成员数、跳过的已存在成员数、失败成员（含其错误
码）、以及回滚失败的残余成员。原始失败为主、回滚失败为辅的排序与后端一致。

### D3 卸载确认呈现计划而非结果

pack 卸载前通过 `packUninstallPlan` 查询后端计划，按 `remove` / `preserve`（附原因：
`pre_existing` / `version_changed`）/ `already_missing` 分桶呈现；确认后执行卸载并刷新
所有权日志投影。前端不自行计算任何成员取舍。

### D4 已装集合包区独立于已安装插件列表

已安装集合包列表来自所有权日志 + 对账投影（`listPackInstallations`），逐成员呈现
`ExpectedAndPresent` / `VersionChanged` / `Missing` 状态与所有权。pack 不伪装成
`InstalledPlugin`，因此该列表与单插件列表物理分离。

### D5 pack 级聚合状态暂不引入

"安装成功/部分安装/损坏"的 pack 级徽标可由日志 + 对账动态推导；在产品明确需要前不
定义持久化聚合状态，也不在前端静态枚举。

## 不变量

1. 前端不重算所有权：成员归属、版本事实与卸载取舍均来自后端投影。
2. 隐藏成员不出现在任何前端呈现中（由索引过滤保证，见 D1 决策）。
3. pack 卡片的安装入口永远存在；重装即补齐缺失成员。
4. 卸载确认呈现的桶与后端计划逐桶一致。
5. 已装集合包区与单插件已安装列表物理分离。

## 核心测试用例

[The marketplace projects pack cards with their members and an install entry](../../../../test-cases/desktop/plugin/extension-pack/pack-frontend-presentation.md#the-marketplace-projects-pack-cards-with-their-members-and-an-install-entry)、
[Pack install feedback reports members, skips, failures and rollback diagnostics](../../../../test-cases/desktop/plugin/extension-pack/pack-frontend-presentation.md#pack-install-feedback-reports-members-skips-failures-and-rollback-diagnostics)、
[The installed-packs section presents journal members with their reconciled states](../../../../test-cases/desktop/plugin/extension-pack/pack-frontend-presentation.md#the-installed-packs-section-presents-journal-members-with-their-reconciled-states)、
[The pack uninstall confirmation presents the backend plan verbatim](../../../../test-cases/desktop/plugin/extension-pack/pack-frontend-presentation.md#the-pack-uninstall-confirmation-presents-the-backend-plan-verbatim)。
