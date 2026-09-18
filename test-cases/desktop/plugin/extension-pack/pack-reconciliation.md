# Extension Pack：重启对账核心用例

对应决策：[Pack 重启对账决策](../../../../decisions/desktop/plugin/extension-pack/20260916-pack-restart-reconciliation.md)
的不变量 1–5。

前置状态：pack 已安装且所有权日志已记录；随后安装树发生漂移（外部删除、独立升级）。

## A fresh pack install reconciles every member as present at the recorded version

- **风险**：对账若不能确认"日志 == 现实"的正常状态，D3-C 的卸载回收就没有可信基线。
- **触发**：完成一次 pack 安装编排并记录所有权，随后对账。
- **必须成立**：每个日志成员分类为 `ExpectedAndPresent`，携带记录版本与日志所有权；
  对账结果携带 pack 的 canonical id。
- **禁止结果**：未漂移的成员被分类为漂移；对账改写任何状态。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 全新安装后两个成员均为 `ExpectedAndPresent`（含所有权） | `crates/backend/src/plugin/operations/install_tests.rs` `reconcile_reports_expected_present_after_a_fresh_pack_install` | Covered |
- **决策依据**：对账决策 D1、D3。

## An externally deleted member reconciles as missing with the journal untouched

- **风险**：外部删除是真实漂移；对账若改写日志，所有权事实就被对账行为本身破坏。
- **触发**：安装 pack 后从安装树中删除一个成员的包目录，再对账。
- **必须成立**：
  - 被删除成员分类为 `Missing`，携带日志中的 `version_at_install` 与所有权；
  - 其余成员分类不受影响；
  - 日志在对账后保持原样（成员数、版本、所有权均不变）。
- **禁止结果**：对账把缺失成员从日志中移除，或触发任何重装。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 缺失分类 + 日志只读 | `crates/backend/src/plugin/operations/install_tests.rs` `reconcile_identifies_a_missing_member_without_touching_the_ledger` | Covered |
- **决策依据**：对账决策 D1、D3、不变量 1。

## An independently upgraded member reconciles as version-changed and the journal keeps its facts

- **风险**：独立升级若改写 `version_at_install`，历史事实丢失；若把升级后的
  `PreExisting` 成员转为 pack 管理，D3-C 卸载会误收用户自有的插件。
- **触发**：pack 安装后，两个成员（一个 `managed_by_pack`、一个 `pre_existing`）分别被
  独立升级到新版本，再对账。
- **必须成立**：
  - 两个成员都分类为 `VersionChanged`，携带记录版本、当前版本与日志所有权；
  - 日志的 `version_at_install` 与所有权在对账后保持不变；
  - `pre_existing` 不因版本变化转为 `managed_by_pack`。
- **禁止结果**：对账更新日志；所有权随版本漂移改变归属。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 两种所有权的版本变化分类 + 日志保持历史事实 | `crates/backend/src/plugin/operations/install_tests.rs` `reconcile_identifies_an_independently_upgraded_member_and_keeps_the_journal_facts` | Covered |
- **决策依据**：对账决策 D2、不变量 3、4。

## Reconciliation never invents a relationship for an unrecorded pack

- **风险**：若对账从 marketplace 清单为未安装过的 pack 发明关系，registry 的后续更新会
  污染旧安装关系的解释，所有权日志的"只记录真实安装行为"承诺被破坏。
- **触发**：对从未记录过安装的 pack id 执行对账。
- **必须成立**：返回空，不产生任何分类。
- **禁止结果**：从当前清单推导出成员关系。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 未记录的 pack 对账为空 | `crates/backend/src/plugin/operations/install_tests.rs` `reconcile_returns_none_for_an_unrecorded_pack` | Covered |
- **决策依据**：对账决策 D3、不变量 5。
