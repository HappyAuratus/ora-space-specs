# Extension Pack：安装所有权日志核心用例

对应决策：[Pack 安装所有权决策](../../../../decisions/desktop/plugin/extension-pack/20260916-pack-installation-ownership.md)
的不变量 1–6，及其继承的根决策不变量 5、7。

前置状态：marketplace source 已同步，pack 清单与成员清单均在源检出中；成员制品真实存在。

## Pack ownership journal records created and pre-existing members

- **风险**：如果所有权只记"成员属于 pack"，卸载时会误删用户预先安装的插件；如果每次
  pack 安装都重写全部关系，被跳过成员的真实来历会丢失。
- **触发**：预安装其中一个成员后执行 pack 安装编排，然后读取所有权日志。
- **必须成立**：
  - 本次运行创建的成员记录为 `managed_by_pack`，`version_at_install` 为落盘版本；
  - 被跳过的成员记录为 `pre_existing`，版本为其当前版本；
  - pack 行携带安装来源的 canonical URL，`pack_id` 为 canonical id。
- **禁止结果**：跳过成员被记为 `managed_by_pack`；已有关系被重写或丢失；pack 因所有权
  日志而出现在发现列表或已安装列表中。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 全新成员记录为 `managed_by_pack` 且带落盘版本 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_install_installs_every_applicable_member_in_declaration_order` | Covered |
  | 预装成员记录为 `pre_existing`，创建成员记录为 `managed_by_pack` | `crates/backend/src/plugin/operations/install_tests.rs` `pack_install_records_created_members_as_managed_and_skips_as_pre_existing` | Covered |
  | 日志不使 pack 进入已安装模型（pack 无包目录、无清单） | 设计约束：日志表只含关系行；根决策不变量 2 的既有测试继续通过 | Covered |
- **决策依据**：所有权决策 D1、D2。

## The ownership journal survives a restart

- **风险**：所有权若只存在于运行内存，重启后 pack 与成员的关系消失，卸载与回滚将失去
  依据。
- **触发**：完成一次 pack 安装并记录所有权后，关闭插件宿主，用同一数据库重新打开，直接
  读取所有权日志。
- **必须成立**：
  - 重新打开后的日志与重启前完全一致（成员、版本、所有权、来源）；
  - 无需重新安装即读到关系。
- **禁止结果**：重启后记录丢失、成员顺序或所有权发生变化。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 重启后日志逐字段一致 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_ownership_survives_a_restart` | Covered |
- **决策依据**：所有权决策 D3、不变量 5。
