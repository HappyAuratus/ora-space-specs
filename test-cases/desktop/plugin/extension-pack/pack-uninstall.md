# Extension Pack：所有权感知卸载核心用例

对应决策：[Pack 卸载所有权决策](../../../../decisions/desktop/plugin/extension-pack/20260916-pack-uninstall-ownership.md)
的不变量 1–7。

前置状态：pack 已安装且所有权日志已记录；对账反映安装树的当前事实。

## Packing uninstall removes managed members and clears the journal

- **风险**：卸载若不按所有权回收，要么删除用户资产，要么留下孤儿包目录与失效日志。
- **触发**：对包含两个 managed 成员的 pack（均为 `ExpectedAndPresent`）执行 pack 卸载。
- **必须成立**：
  - 两个成员的包目录均被删除；
  - 卸载响应携带 pack 的 canonical id；
  - 所有权日志（根记录与成员关系）全部清除。
- **禁止结果**：任一成员目录残留；日志行或根记录残留。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 两个 managed 成员被删除且日志清空 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_uninstall_removes_every_managed_member_and_clears_the_journal` | Covered |
- **决策依据**：卸载决策 D1、D2、D3。

## A pre-existing member survives the pack uninstall

- **风险**：`pre_existing` 成员若被 pack 卸载回收，用户在 pack 之前安装的插件将被破坏。
- **触发**：预安装一个成员后安装 pack，再执行 pack 卸载。
- **必须成立**：
  - 预安装成员的包目录保持原版本不变；
  - managed 成员正常移除；
  - 关系释放后日志清空。
- **禁止结果**：预安装成员被删除或版本被改动。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 预装成员保留、managed 成员移除、日志清空 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_uninstall_preserves_a_pre_existing_member` | Covered |
- **决策依据**：卸载决策 D2、不变量 2。

## An independently upgraded managed member survives the pack uninstall

- **风险**：`managed_by_pack` 成员被用户独立升级后若仍被回收，用户主动选择的版本将被
  降级删除。
- **触发**：pack 安装后独立升级一个成员到新版本，再执行 pack 卸载。
- **必须成立**：
  - 升级后的成员包目录保持不变（分类为 `VersionChanged` → 保留）；
  - 未漂移的 managed 成员正常移除；
  - 日志清空。
- **禁止结果**：独立升级的成员被删除或降级。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 升级成员保留、未漂移成员移除、日志清空 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_uninstall_preserves_an_independently_upgraded_member` | Covered |
- **决策依据**：对账决策 D2 的边界在卸载中的落地、卸载决策不变量 3。

## An already-missing managed member does not fail the uninstall

- **风险**：成员已被外部删除时，若卸载对其报错或重装，pack 卸载将无法完成。
- **触发**：安装 pack 后外部删除一个成员的包目录，再执行 pack 卸载。
- **必须成立**：
  - 缺失成员不产生文件操作，关系直接释放；
  - 其余成员照常移除，卸载成功完成。
- **禁止结果**：缺失成员导致卸载失败；缺失成员触发重装。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 缺失成员不阻塞卸载且关系释放 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_uninstall_completes_when_a_managed_member_is_already_missing` | Covered |
- **决策依据**：卸载决策 D1（already_missing 桶）、不变量 4。

## A mixed pack removes only the eligible member

- **风险**：混合场景下若计划或执行错位，preserve 桶可能被误删，remove 桶可能被误留。
- **触发**：pack 同时含 `managed + ExpectedAndPresent`、`pre_existing`、
  `managed + VersionChanged` 三种成员，执行 pack 卸载。
- **必须成立**：
  - 仅 managed 且未漂移的成员被移除；
  - 预装成员与独立升级成员全部保留；
  - 日志清空。
- **禁止结果**：任何桶错位导致的误删或误留。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 三成员混合场景只移除 eligible 成员 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_uninstall_removes_only_the_eligible_member_in_a_mixed_pack` | Covered |
- **决策依据**：卸载决策 D1、不变量 2、3。

## A failed member uninstall keeps the journal and a retry completes

- **风险**：成员卸载中途失败若清空日志，剩余关系的历史事实丢失；若继续处理后续成员，
  失败定位与重试语义都会模糊。
- **触发**：计划生成后外部删除第二个成员的包目录，使陈旧计划的执行在该成员处失败；
  随后重试卸载。
- **必须成立**：
  - 首个成员在失败前已被移除并保持移除；
  - 失败成员的日志行保留，pack 根记录保留；
  - 重试基于存活日志重新生成计划：失败成员此时为 `Missing` → 关系释放、日志清空。
- **禁止结果**：失败时删除根记录；重试从头重装已移除的成员。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 失败保留日志 + 重试完成卸载 | 无直接测试（此前的证据条目引用了不存在的测试名，按证据规则如实降级） | Missing |
- **决策依据**：卸载决策 D3、不变量 5、6。
