# Extension Pack 核心测试用例

本目录是插件集合包领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/plugin/extension-pack) 镜像。

架构边界和不变量来自
[插件集合包决策](../../../../decisions/desktop/plugin/extension-pack/0-extension-pack.md)。

## 当前状态

根决策已 `approved`；D0-D4 全链路能力建设已完成（安装编排、所有权日志、重启对账、
所有权感知卸载、事务性回滚、前端呈现），并通过 release qualification 验证真实
`python-extension-pack` 内容可完全通过现有机制表达和安装——无需任何 pack-id 特例代码。

已登记用例：

- 清单、索引与可见性（D1 阶段）：见 [manifest-and-visibility.md](manifest-and-visibility.md)
- 安装编排（D2 阶段）：见 [pack-install-orchestration.md](pack-install-orchestration.md)
- 安装所有权日志（D3-A 阶段）：见 [pack-ownership.md](pack-ownership.md)
- 重启对账（D3-B 阶段）：见 [pack-reconciliation.md](pack-reconciliation.md)
- 所有权感知卸载（D3-C 阶段）：见 [pack-uninstall.md](pack-uninstall.md)
- 安装事务回滚（D3-D 阶段）：见 [pack-rollback.md](pack-rollback.md)
- 前端呈现（D4 阶段）：见 [pack-frontend-presentation.md](pack-frontend-presentation.md)

Release qualification（D5 阶段）：`python-extension-pack` 的真实内容已通过
`e2e/tests/plugin_marketplace.rs` `marketplace_plugin_full_lifecycle_walkthrough` 和
`crates/backend/src/plugin/operations/install_tests.rs` 中全部 pack 集成测试验证——
涵盖 sync → discover → install → ownership → uninstall 完整路径，使用真实 Git 源检
出、真实 .orax 制品、真实 SHA-256 校验与真实单插件安装链。无 pack-id 特例代码。

## 尚未登记的用例

- 真实 marketplace 仓库推送后在线安装（需要 `ora-space/marketplace` 仓库提交实际内容）。

## 尚未登记的用例（随落地阶段补充）

- pack 级卸载与所有权语义（D3）。
- `PackInstalled` 聚合结果的前端呈现与聚合进度展示（D4）。
- `python-extension-pack` 真实内容源下的按 agent 过滤完整安装（D5）。
