---
status: implemented
---

# Pack 卸载所有权：只回收 Pack 创建且未被用户改变的成员

[Pack 安装所有权决策](20260916-pack-installation-ownership.md) 与
[重启对账决策](20260916-pack-restart-reconciliation.md) 提供了卸载所需的全部判定依据：
日志中的所有权是历史事实，对账分类是当前事实。本决策将两者组合为所有权感知的 pack
卸载：先计算计划，再执行，最后释放日志——pack 卸载可回收的成员，当且仅当它是
pack 创建（`managed_by_pack`）且仍处于创建时的版本（`ExpectedAndPresent`）。

承担模块：`crates/backend`（卸载计划与执行编排）。成员删除本身复用既有单插件卸载链
（stop lifecycle → supervisor 挂起 → `lifecycle.uninstall_plugin` → 数据目录处置），
不引入第二套删除机制。

## 继承与修改

| 前序决策的约定 | 本决策 |
| --- | --- |
| 对账三分类（ExpectedAndPresent / VersionChanged / Missing），只读 | 继承；分类驱动卸载计划，对账行为保持只读 |
| `managed_by_pack` 表示"该关系对应的安装行为创建了成员" | 继承并落地：只有它且版本未漂移的成员可被回收 |
| `version_at_install` 是历史事实，对账不更新 | 继承；卸载也不改写它，被保留成员的日志行随关系释放一起删除 |
| 成员卸载复用单插件链路（stop → supervisor 挂起 → uninstall → 清理） | 继承；pack 卸载逐成员调用同一链路，supervisor 按成员挂起 |

## 决策

### D1 卸载计划先于执行

对账完成后立即生成封闭计划：`remove`（managed 且版本未变）、`preserve`（带原因：
`pre_existing` 或 `version_changed`）、`already_missing`（managed 但已不存在）。计划不落
盘；执行只消费计划。这使卸载行为可测试、可解释，也让 D4 前端可以直接呈现"将删除/
将保留"。

### D2 成员回收复用单插件卸载链

被移除成员逐个走完整单插件卸载（生命周期停止、supervisor 按成员挂起、包目录删除、
数据目录按处置参数清理）。数据目录处置沿用调用方对 pack 卸载的处置参数。

### D3 日志逐关系释放；根记录在全部关系释放后删除

- 成功移除的 managed 成员：日志行删除（关系已履行完毕）；
- 保留成员（`pre_existing`、`version_changed`）：日志行删除（pack 卸载主动解除关系，
  被保留成员此后与 pack 无生命周期关联）；
- already-missing 成员：日志行删除（关系释放，磁盘上本就无物可删）；
- 卸载失败的成员：日志行**保留**，运行立即停止——重试从该成员继续；
- 根 `pack_installation` 行仅在成员关系全部释放后删除。

这意味着"存在日志行"不表示"pack 安装完整成功"，也不表示"卸载已完成"——它只表示
仍有未释放的关系。

## 不变量

1. 卸载计划只读生成；执行只消费计划。
2. `pre_existing` 成员在任何对账状态下都被保留。
3. `managed_by_pack` 且版本未漂移才可回收；版本漂移即保留。
4. `Missing` 的 managed 成员不产生文件操作。
5. 成员卸载失败立即停止运行；失败成员的日志行保留，pack 根记录保留。
6. 根记录只在全部关系释放后删除；重试基于存活的日志重新生成计划。
7. 成员删除只走既有单插件卸载链，supervisor 按成员挂起。

## 核心测试用例

[Packing uninstall removes managed members and clears the journal](../../../../test-cases/desktop/plugin/extension-pack/pack-uninstall.md#packing-uninstall-removes-managed-members-and-clears-the-journal)、
[A pre-existing member survives the pack uninstall](../../../../test-cases/desktop/plugin/extension-pack/pack-uninstall.md#a-pre-existing-member-survives-the-pack-uninstall)、
[An independently upgraded managed member survives the pack uninstall](../../../../test-cases/desktop/plugin/extension-pack/pack-uninstall.md#an-independently-upgraded-managed-member-survives-the-pack-uninstall)、
[An already-missing managed member does not fail the uninstall](../../../../test-cases/desktop/plugin/extension-pack/pack-uninstall.md#an-already-missing-managed-member-does-not-fail-the-uninstall)、
[A mixed pack removes only the eligible member](../../../../test-cases/desktop/plugin/extension-pack/pack-uninstall.md#a-mixed-pack-removes-only-the-eligible-member)、
[A failed member uninstall keeps the journal and a retry completes](../../../../test-cases/desktop/plugin/extension-pack/pack-uninstall.md#a-failed-member-uninstall-keeps-the-journal-and-a-retry-completes)。
