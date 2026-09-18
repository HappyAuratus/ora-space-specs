---
status: implemented
---

# Pack 重启对账：只识别漂移，不修复状态

[Pack 安装所有权决策](20260916-pack-installation-ownership.md) 落地后，所有权日志与
安装树是两个可能漂移的事实：成员可能被用户独立升级、独立卸载，或被外部手段直接删除。
本决策为重启后的对账定义封闭的分类与只读语义：对账**只识别**，不修复、不改写日志、
不从当前 marketplace 清单重新推导所有权——pack 清单可能已经更新，不能用新版的声明去
解释旧安装关系。

承担模块：`crates/backend`（`pack_reconcile`：对账分类）。

## 继承与修改

| [Pack 安装所有权决策](20260916-pack-installation-ownership.md) 的约定 | 本决策 |
| --- | --- |
| 所有权日志记录 `managed_by_pack` / `pre_existing` 与 `version_at_install` | 继承；对账的比对基线永远是日志，而不是当前 marketplace 清单 |
| `version_at_install` 记录关系建立时的成员版本，独立升级偏差留给对账识别 | 继承并落地：对账将偏差分类为 `VersionChanged`，日志保持历史事实不改写 |
| 日志在安装运行结束时写入，失败成员缺席 | 继承；对账只读，不把"日志成员数不完整"解释为安装成功或失败 |

## 决策

### D1 成员级封闭分类

每个日志成员对账为恰好一种状态：`ExpectedAndPresent`（已安装版本与
`version_at_install` 一致）、`VersionChanged`（已安装但版本不同——独立升级或替换）、
`Missing`（未安装——外部删除或操作失败后未补齐）。没有第四种状态，也没有包级的
"安装成功"判定：日志本身就是 ownership ledger，不承载"pack 当前是否完整"的断言。

### D2 所有权与历史版本在对账中不可变

`PreExisting` 成员即使后来被升级，分类也是 `VersionChanged` 而所有权仍是
`pre_existing`——版本变化不改变来历。`managed_by_pack` 成员独立升级只改变分类，绝不
改写 `version_at_install`：该字段表达历史事实（"pack 当时创建的是哪个版本"）。这一
边界是 D3-C 卸载规则的前提：`managed_by_pack` 且版本未变才可回收，已独立变化默认
保留。

### D3 对账是纯读操作

对账读取所有权日志与一次安装树发现快照，不写日志、不触碰安装目录、不访问
marketplace。pack 无日志时返回空——对账不为从未发生过的安装发明关系。

## 不变量

1. 对账只读：日志、安装目录、marketplace 检出均不被修改。
2. 每个日志成员恰有一种分类；分类集合封闭。
3. 所有权永远来自日志，独立升级不改变所有权归属。
4. `version_at_install` 是历史事实，对账不更新它。
5. 对账不从 marketplace 清单推导任何关系。

## 核心测试用例

[A fresh pack install reconciles every member as present at the recorded version](../../../../test-cases/desktop/plugin/extension-pack/pack-reconciliation.md#a-fresh-pack-install-reconciles-every-member-as-present-at-the-recorded-version)、
[An externally deleted member reconciles as missing with the journal untouched](../../../../test-cases/desktop/plugin/extension-pack/pack-reconciliation.md#an-externally-deleted-member-reconciles-as-missing-with-the-journal-untouched)、
[An independently upgraded member reconciles as version-changed and the journal keeps its facts](../../../../test-cases/desktop/plugin/extension-pack/pack-reconciliation.md#an-independently-upgraded-member-reconciles-as-version-changed-and-the-journal-keeps-its-facts)、
[Reconciliation never invents a relationship for an unrecorded pack](../../../../test-cases/desktop/plugin/extension-pack/pack-reconciliation.md#reconciliation-never-invents-a-relationship-for-an-unrecorded-pack)。
