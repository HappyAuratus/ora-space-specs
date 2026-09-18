# Extension Pack：安装事务回滚核心用例

对应决策：[Pack 安装事务性回滚决策](../../../../decisions/desktop/plugin/extension-pack/20260916-pack-install-transactional-rollback.md)
的不变量 1–7。

前置状态：pack 清单与成员清单均在源检出中；成员制品按测试意图构造（正常或损坏）。

## Rollback removes created members when a later member fails

- **风险**：成员失败若不回滚，失败的 pack 安装会永久留下半套成员，破坏"失败安装可安全
  撤销"的承诺。
- **触发**：两个成员按声明顺序安装，第二个成员的制品损坏导致其提取失败。
- **必须成立**：
  - 第一个成员（已落盘）被回滚删除；
  - 失败成员未落盘；
  - 结果为 `Ok(PackInstalled)`：`members` 为空、`failed` 携带原始失败与空回滚诊断。
- **禁止结果**：已落盘成员残留；失败被上报为 `Err` 丢失成员级信息。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 后失败场景的回滚删除创建成员且结果如实 | `crates/backend/src/plugin/operations/install_tests.rs` `rollback_removes_created_members_when_a_later_member_fails` | Covered |
- **决策依据**：回滚决策 D1、D3、D4。

## The rollback never touches pre-existing members and mints no phantom journal facts

- **风险**：回滚若作用于预存成员或跳过成员，用户资产会被误删；失败运行若伪造
  `pre_existing` 日志关系，所有权日志就不再反映真实安装行为。
- **触发**：预安装一个成员，pack 含三个成员（创建、预存、失败），失败成员制品损坏。
- **必须成立**：
  - 仅创建成员被回滚；预存成员保持安装且版本不变；
  - 失败运行不写日志：不存在为跳过成员伪造的 `pre_existing` 关系。
- **禁止结果**：预存成员被删除；日志出现本次失败运行伪造的关系。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 预存成员保留、创建成员回滚、日志保持运行前事实 | `crates/backend/src/plugin/operations/install_tests.rs` `rollback_never_touches_pre_existing_members` | Covered |
- **决策依据**：回滚决策 D1、D3、不变量 2。

## The rollback runs in reverse creation order

- **风险**：非逆序回滚（按创建顺序）会在依赖关系尚未撤销时先撤销基础成员；顺序错误还会
  让残余集合的归因失真。
- **触发**：三个成员按声明顺序安装，第三个成员制品损坏失败；观察前两个成员的回滚结果。
- **必须成立**：
  - 逆序回滚（第二创建者 → 第一创建者）后两个成员均被删除；
  - 回滚诊断为空（两次回滚均成功）。
- **禁止结果**：任一创建成员残留；回滚诊断非空。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 逆序回滚后全部创建成员被删除 | `crates/backend/src/plugin/operations/install_tests.rs` `rollback_runs_in_reverse_order_and_clears_every_created_member` | Covered |
- **决策依据**：回滚决策 D2。

## A failed rollback journals residual members and surfaces both failures

- **风险**：回滚自身失败若被吞掉，残余成员将变成无人认领的孤儿；若覆盖原始失败，用户
  将看不到真正的失败原因。
- **触发**：使第一个创建成员的回滚失败（其数据目录被外部进程占用），第三个成员安装
  失败触发回滚。
- **必须成立**：
  - 回滚失败的成员保持安装，并以 `managed_by_pack` 立即写入日志（恢复证据）；
  - 结果中 `failed` 携带原始失败成员与错误码，`rollback_failures` 携带回滚失败的成员；
  - 回滚成功的成员已删除且不出现在日志中。
- **禁止结果**：回滚失败覆盖原始失败；残余成员既不安装也不入账。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 残余成员保持安装、日志记录、双失败结构化呈现 | `crates/backend/src/plugin/operations/install_tests.rs` `a_failed_rollback_keeps_residual_members_and_journals_them` | Covered |
- **决策依据**：回滚决策 D2、D3、D4、不变量 4、5、6。

## A retry after a complete rollback installs normally

- **风险**：完全回滚后若日志残留任何关系，重试会跳过成员或伪造所有权。
- **触发**：失败并完全回滚后，修复制品并重试安装。
- **必须成立**：
  - 重试安装全部成员成功；
  - 所有权日志记录普通 `managed_by_pack` 关系，无幽灵事实。
- **禁止结果**：重试跳过已回滚成员；日志出现回滚前失败尝试的痕迹。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 重试成功且日志干净 | `crates/backend/src/plugin/operations/install_tests.rs` `retry_after_a_complete_rollback_installs_normally` | Covered |
- **决策依据**：回滚决策 D5、不变量 5。

## A partial-rollback residual is reconciled after a restart and removable by uninstall

- **风险**：部分回滚失败的残余成员若在重启后不可见或不可回收，恢复证据就失去了意义。
- **触发**：回滚失败产生残余成员并写入日志后，重启宿主，对账，然后执行 pack 卸载。
- **必须成立**：
  - 重启后对账将残余成员如实识别为 `ExpectedAndPresent`（`managed_by_pack`）；
  - pack 卸载将其回收（版本未漂移），日志清空。
- **禁止结果**：重启后残余成员从日志消失；卸载无法回收残余成员。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 重启对账识别残余 + 卸载回收 | `crates/backend/src/plugin/operations/install_tests.rs` `partial_rollback_residual_is_reconciled_after_a_restart` | Covered |
- **决策依据**：回滚决策 D5、不变量 5；对账决策 D1。

## A failed rollback never deletes prior journal relations

- **风险**：失败运行的回滚若删除运行前已存在的日志关系，历史安装的真实归属将被破坏。
- **触发**：run 1 安装并记录两个成员；外部删除其中一个成员；run 2 重装该成员并在另一
  个（损坏的）成员上失败，触发回滚。
- **必须成立**：
  - run 2 回滚仅移除本次重建的成员包；
  - 日志保持 run 1 的全部关系（成员、版本、所有权）逐字段不变；
  - 被外部删除的成员在对账中如实呈现为 `Missing`。
- **禁止结果**：run 2 的失败或回滚删除/改写 run 1 的日志关系。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | run 2 失败后日志与 run 1 记录逐字段一致 | `crates/backend/src/plugin/operations/install_tests.rs` `a_failed_rollback_never_deletes_prior_journal_relations` | Covered |
- **决策依据**：回滚决策 D1、D3、不变量 2、5。
