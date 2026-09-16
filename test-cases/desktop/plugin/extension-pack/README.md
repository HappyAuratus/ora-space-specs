# Extension Pack 核心测试用例

本目录是插件集合包领域的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/plugin/extension-pack) 镜像。

架构边界和不变量来自
[插件集合包决策](../../../../decisions/desktop/plugin/extension-pack/0-extension-pack.md)。

## 当前状态

根决策已 `approved`；安装编排（D2）、安装所有权日志（D3-A）、重启对账（D3-B）、所有权
感知卸载（D3-C）与安装事务回滚（D3-D）已落地。已登记用例：

- 清单、索引与可见性（D1 阶段）：见 [manifest-and-visibility.md](manifest-and-visibility.md)
- 安装编排（D2 阶段）：见 [pack-install-orchestration.md](pack-install-orchestration.md)
- 安装所有权日志（D3-A 阶段）：见 [pack-ownership.md](pack-ownership.md)
- 重启对账（D3-B 阶段）：见 [pack-reconciliation.md](pack-reconciliation.md)
- 所有权感知卸载（D3-C 阶段）：见 [pack-uninstall.md](pack-uninstall.md)
- 安装事务回滚（D3-D 阶段）：见 [pack-rollback.md](pack-rollback.md)

后续阶段（D4 前端呈现、D5 真实内容接入）的用例随其落地阶段登记。

## 尚未登记的用例（随落地阶段补充）

- pack 级卸载与所有权语义（D3）。
- `PackInstalled` 聚合结果的前端呈现与聚合进度展示（D4）。
- `python-extension-pack` 真实内容源下的按 agent 过滤完整安装（D5）。
