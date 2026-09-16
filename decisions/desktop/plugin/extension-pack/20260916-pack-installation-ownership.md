---
status: implemented
---

# Pack 安装所有权：持久化"本次安装是否创建了成员"

Extension Pack 的安装编排（根决策 D5/D6）落地后，一次 pack 安装的真实结果只存在于
`PackInstalled` 返回值中，重启即失。本决策为 pack 安装增加一份持久化的所有权日志：
记录每个被 pack 点名的成员是**由这次安装创建的**（`managed_by_pack`）还是**在 pack
点名时就已经存在**（`pre_existing`），以及建立关系时的成员版本。它是 D3 卸载所有权、
回滚与重启对账的持久化基座；记录本身不是 InstalledPlugin——pack 依旧没有包目录、没有
清单、不进入发现与更新生命周期（继承根决策 D7）。

承担模块：`ora-db`（`pack_installation` / `pack_installation_member` 表与
`SqlitePackInstallationRepository`）、`crates/backend`（安装编排写入日志）。

## 继承与修改

| 根决策的约定 | 本决策 |
| --- | --- |
| pack 不进入已安装模型：无包目录、无清单、不参与发现/更新生命周期 | 继承，不变；所有权日志是独立的关系表，不伪造包目录，也不把 pack 变成 InstalledPlugin |
| `PackInstalled` 逐成员可辨（已装/跳过/失败） | 继承，并把同一信息在安装运行结束后持久化 |
| 已安装成员跳过且不算失败；既有版本不被改动或降级 | 继承；跳过成员按"本次未触碰"如实记录，首次跳过记为 `pre_existing` |
| 成员失败停止且不回滚；失败成员可辨 | 继承；失败成员不写入日志，后续 pack 安装补齐它时再扩展记录 |
| 重装 pack 等价于补齐缺失成员 | 继承；补齐的成员按本次行为记录为 `managed_by_pack` |

## 决策

### D1 所有权记录"安装行为"，而非"成员归属"

`ownership` 只有两个值：`managed_by_pack` 表示这条关系对应的安装行为创建了该成员；
`pre_existing` 表示 pack 点名时成员已存在、安装行为从未触碰它。成员被用户独立升级、
替换或删除后，记录不随之改写——`version_at_install` 与实际版本的偏差正是 D3 后续
阶段对账的输入，而不是需要立即修复的错误。

### D2 日志在安装运行结束时写入，失败成员缺席

每次 pack 安装编排运行（无论全部完成还是中途失败）结束时写入一次：本次创建的成员按
落盘版本记为 `managed_by_pack`；被跳过的成员若已有记录则保持原样（不刷新版本、不改变
所有权），首次被跳过则按当前版本记为 `pre_existing`；运行中失败的成员不写日志。这样
"补齐缺失成员"的下一次安装自然扩展记录，且失败不会把未完成的状态伪装成事实。

### D3 关系是重启持久的、按成员稳定可读的

`pack_id`（canonical `namespace/identifier`）与 `member_id` 构成关系主键；成员行按
identifier 排序读取，与写入顺序无关。pack 行额外记录安装来源的 canonical URL，供后续
阶段把成员解析引导回 pack 自己的源。记录只在对应 pack 的关系被显式移除时删除。

## 不变量

1. 所有权日志不使 pack 成为 InstalledPlugin：无包目录、无清单、不进入发现列表。
2. `managed_by_pack` 仅表示"该关系对应的安装行为创建了此成员"，不表示独占所有权。
3. 被跳过的成员不改变既有记录；首次跳过以 `pre_existing` 和当前版本入账。
4. 运行中失败的成员不写入日志。
5. `version_at_install` 记录关系建立时的成员版本，独立升级造成的偏差留给对账阶段识别。
6. 成员关系按 `(pack_id, member_id)` 唯一，读取按 identifier 排序。

## 核心测试用例

[Pack ownership journal records created and pre-existing members](../../../../test-cases/desktop/plugin/extension-pack/pack-ownership.md#pack-ownership-journal-records-created-and-pre-existing-members)、
[The ownership journal survives a restart](../../../../test-cases/desktop/plugin/extension-pack/pack-ownership.md#the-ownership-journal-survives-a-restart)。
