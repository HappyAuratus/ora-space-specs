---
status: implemented
---

# Pack 安装事务性回滚：只撤销本次创建的成员

[Pack 安装编排决策](20260916-pack-installation-ownership.md) 的成员失败语义是"停止且不
回滚"。本决策将该行为升级为事务性回滚：一次 pack 安装在成员失败后，立即按**逆创建顺序**
撤销**本次运行创建的成员**，然后如实报告失败。回滚复用所有权感知卸载的既有单插件卸载
链路，不引入第三套删除机制。

承担模块：`crates/backend`（`PackRunLedger` 事务账本、`rollback_created_members`）。

## 继承与修改

| [Pack 安装编排决策](0-extension-pack.md) 与 [所有权决策](20260916-pack-installation-ownership.md) 的约定 | 本决策 |
| --- | --- |
| 成员失败停止运行，失败可辨 | 继承；停止后先回滚再报告 |
| 已安装成员跳过且不算失败 | 继承；跳过成员（含历史 `managed_by_pack` 成员）永远不进入回滚集合 |
| 所有权日志记录创建 vs 预存 | 继承；回滚只作用于本次创建的成员，日志的历史关系不被删除 |
| 日志在安装运行结束时写入 | **修改**：失败的运行如果回滚未完全成功，必须立即写入残余证据日志；回滚完全成功的运行不写日志，日志保持运行前的事实 |

## 决策

### D1 回滚集合 = 本次运行创建的成员

回滚的输入是本次运行账本（`PackRunLedger.installed`，按创建顺序）中实际落盘的成员。
`PreExisting`/跳过成员、历史 `managed_by_pack` 成员、用户独立安装的成员一律不进入回滚
集合——回滚不可能触碰用户资产或历史事实。

### D2 逆创建顺序撤销，首个失败即停止

撤销按创建顺序的**严格逆序**执行（最后创建的成员最先撤销），复用所有权感知卸载的
单插件卸载链（supervisor 不需要挂起——回滚的成员从未被 reconcile 进 supervisor）。首个
回滚失败即停止：该成员与其余未尝试的成员保持安装状态，作为残余证据记录。

### D3 日志语义：完全回滚不写、残余必须写

- 回滚完全成功：日志保持运行前的事实（本次运行没有持久化任何关系，也没有删除任何
  既有关系）——不写日志；
- 回滚部分失败：残余成员（回滚失败 + 未尝试）必须立即以 `managed_by_pack` 写入日志，
  作为后续卸载/重试的恢复证据；跳过成员不写入（失败的运行不得伪造
  `pre_existing` 关系）。

### D4 失败报告：原始失败为主，回滚失败为辅

原始成员安装失败是 primary failure；回滚失败作为结构化补充诊断
（`PackInstallFailure.rollback_failures`，按创建顺序）随同一个 `Ok(PackInstalled)` 结果
返回，不覆盖原始失败原因。残余成员通过 `members` 如实呈现为已安装。

### D5 重试与重启

- 完全回滚后重试：日志保持运行前事实，重试等价于一次全新安装；
- 部分回滚失败后重试：preflight 识别残余成员已安装并跳过（不重复安装、不覆盖历史
  `version_at_install`），剩余成员照常安装；
- 重启后对账（D3-B）如实呈现残余成员（`ExpectedAndPresent`），后续 pack 卸载可回收
  它（`managed_by_pack` 且版本未漂移）。

## 不变量

1. 回滚集合仅含本次运行创建且仍安装的成员；逆创建顺序执行。
2. `PreExisting`、跳过成员、历史 `managed_by_pack` 成员永远不被回滚。
3. 回滚复用单插件卸载链；不新增删除机制。
4. 首个回滚失败即停止；未尝试成员保持安装并计入残余。
5. 残余成员必须立即以 `managed_by_pack` 持久化到日志；回滚完全成功的失败运行不写日志。
6. 原始安装失败始终是 primary failure；回滚失败是补充诊断。
7. 重试以文件系统与日志的真实状态为输入：残余成员被跳过（reuse），不重复删除用户资产。

## 核心测试用例

[Rollback removes created members when a later member fails](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#rollback-removes-created-members-when-a-later-member-fails)、
[The rollback never touches pre-existing members and mints no phantom journal facts](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#the-rollback-never-touches-pre-existing-members-and-mints-no-phantom-journal-facts)、
[The rollback runs in reverse creation order](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#the-rollback-runs-in-reverse-creation-order)、
[A failed rollback journals residual members and surfaces both failures](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#a-failed-rollback-journals-residual-members-and-surfaces-both-failures)、
[A retry after a complete rollback installs normally](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#a-retry-after-a-complete-rollback-installs-normally)、
[A partial-rollback residual is reconciled after a restart and removable by uninstall](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#a-partial-rollback-residual-is-reconciled-after-a-restart-and-removable-by-uninstall)、
[A failed rollback never deletes prior journal relations](../../../../test-cases/desktop/plugin/extension-pack/pack-rollback.md#a-failed-rollback-never-deletes-prior-journal-relations)。
