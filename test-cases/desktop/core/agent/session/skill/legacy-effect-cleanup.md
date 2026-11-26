# 升级时清理旧受管 Skill 目录与 0025 预备份边界

决策依据：[Skill 按存储路径以会话清单交给 Agent](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)
（`implemented`）D9，代表性证据固定于 [desktop#827](https://github.com/ora-space/desktop/pull/827) 合并提交
[`9f3ed63f`](https://github.com/ora-space/desktop/commit/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf)。

## Upgrade Cleanup Must Delete Only Exact Ledger Matches And Be Idempotent

### 风险

升级会删除 Effect 表，账本是判断目录归属的唯一依据。如果按目录名、路径或内容相似度删除，会删掉用户或其他工具放入的
内容，或用户改过的副本。如果迁移先于清理执行，账本已不存在，旧副本永远留在原处，与清单重复甚至冲突。如果一次删除
失败就阻止启动，Windows 上的句柄占用会让用户无法使用应用。

### 前置状态

升级前的 Effect 账本中有若干受管条目，对应的 Workspace 状态分别为：

1. 目录存在，`.ora-managed.json` 标记（`schema_version`、`resource`、`managed_identity`）和当前 fingerprint 都与账本一致；
2. 目录存在，标记一致但内容被用户修改，fingerprint 漂移；
3. 目录存在，没有标记、标记损坏、或 `resource` / `managed_identity` 与账本不一致；
4. 目录已不存在，或 Workspace 根目录已被删除，或数据库文件不存在；
5. 目标目录或任一祖先目录是符号链接或 Windows junction；
6. 目录完全匹配，但删除时文件被占用。

另外，同一发现目录下有不在账本中的用户目录，以及一个未终结 Operation 留下的 staging 或 backup Artifact。

### 触发

首次启动移除 Effect 的版本（以及开发期 `cargo xtask reconcile-migrations`）。另做一次：清理完成后、迁移执行前再次触发清理，然后再执行 `0025` 迁移。

### 必须成立

- 只有状态 1 的目录被删除。
- 状态 2–5 的目录、不在账本中的用户目录、Artifact，以及变空的父目录（如 `.claude/skills`）都保持不变。
- 状态 6 的校验或删除失败时，记录 Workspace 相对路径并继续，启动不被阻止，后续有效条目照常清理。
- 清理在删除 Effect 表的迁移 `0025` 之前执行；`0025` 从 `effect_revisions` 回填 `skills.package_root` 并删尽全部 `effect_*` 表、触发器与索引，迁移完成后不再清理。
- 中断后再次启动时清理重做，已删除的目录被跳过，最终结果与一次完整执行相同。

### 禁止结果

- 删除标记、native identity、resource 或 fingerprint 任一项与账本不一致的目录。
- 沿目标或祖先符号链接或 junction 删除。
- 删除 Artifact、用户目录或父目录。
- 迁移先于清理执行。
- 因单个目录清理失败而阻止启动。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 只删除标记、native identity、resource 与 fingerprint 都与账本一致的目录，其他情况全部保留 | [`deletes_exact_managed_skills_and_preserves_siblings_parent_and_artifacts`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L170)、[`preserves_drifted_or_unverified_directories_and_continues_on_errors`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L239)、[`preserves_managed_skill_when_marker_resource_differs`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L474) | `Covered` |
| 目标或祖先链接、junction 与缺失数据库/根目录被跳过 | [`skips_missing_database_without_creating_file`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L356)、[`skips_windows_junction_targets_without_deleting_link_or_target`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L369)、[`skips_unix_symlink_targets_without_deleting_link_or_target`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L427)、[`preserves_managed_skill_when_ancestor_directory_is_a_link`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L514) | `Covered` |
| 单个目录校验或删除失败时记录并继续，不阻止启动与后续条目清理 | [`preserves_drifted_or_unverified_directories_and_continues_on_errors`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L239) 已证明校验失败（漂移、缺/错/坏 marker）不阻断后续有效条目；尚无直接模拟操作系统层 `remove_dir_all` 失败继续的单元测试 | `Partial` |
| 清理先于删除 Effect 表的迁移（`Backend::open` 与 `xtask` 双路径），并回填 `skills.package_root`、删尽 `effect_*` 对象 | [`opens_pre_0025_database_backs_up_and_cleans_legacy_managed_skills`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/bootstrap.rs#L727)、[`reconcile_migrations_cleans_legacy_managed_skills_before_dropping_ledger`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/xtask/src/reconcile_migrations.rs#L52)、[`backfills_plugin_package_root_and_drops_populated_effect_ledger`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/migration/schema/schema_v0025.rs#L125)、[`drops_all_effect_tables_triggers_and_indexes_from_sqlite_master`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/migration/schema/schema_v0025.rs#L204)、[`fresh_database_migrates_and_inserts_workspaces_cleanly`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/migration/schema/schema_v0025.rs#L236) | `Covered` |
| 重复执行与迁移后执行幂等，结果与一次完整执行相同 | [`deletes_exact_managed_skills_and_preserves_siblings_parent_and_artifacts`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L170)（迁移前连跑两次 + 迁移后空操作） | `Covered` |

### 决策依据

- Skill 会话清单决策 D9 与“风险与迁移”中的 Effect 表顺序。
- 不变量 1、11。

## Pre-0025 Migration Must Back Up Atomically And Reject Downgrade

### 风险

schema `0025` 会永久删除 Effect 账本表，旧版 Desktop 无法直接在升级后的库上继续运行。如果迁移前没有原子备份，或者
沿用过时的备份文件（例如回退旧版写新数据后再升级），用户回退旧版时会丢失数据或拿到空/坏文件；如果备份失败还继续
跑 `0025`，就失去了唯一恢复副本；如果允许旧版尝试把 `0025` 降级，会破坏已升级库并卡死后续重开。

### 前置状态

- 存在一个已初始化（含 `migrations` 表）且仍存在 `effect_managed_items` 与 `effect_resources` 表的 SQLite 库（包含 WAL 未截断写入、已有旧备份或 0 字节残余备份、以及表仍在但 `migrations` 已有 `version='0025'` 行的情形）。
- 另有一个全新库、未建库的空文件，以及已完成 `0025` 的库。
- 另有备份失败条件（例如 `.pre-0025.bak.tmp` 被非空目录占用）。

### 触发

启动 `Backend::open` 或调用 `backup_database_before_0025`；以及对已升级到 `0025` 的库执行降级后再次由新版打开。

### 必须成立

- 仅当 `effect_*` 表仍存在时执行 `VACUUM INTO` 临时文件并原子替换 `ora.sqlite3.pre-0025.bak`（不依赖 `migrations` 的 `0025` 版本行）；每次处于 pre-0025 状态的启动都刷新备份，替换旧备份或 0 字节文件并清掉残留 `.tmp`。
- 全新库、缺失/空数据库文件与已完成 `0025` 的库不生成也不修改备份文件。
- 清除残留 `.tmp`、`VACUUM INTO` 或最终原子重命名失败时，已有有效备份不被破坏，`Backend::open` 返回 `PreMigrationBackupFailed` 错误且不执行 `0025` 迁移。若 `VACUUM INTO` 或最终重命名失败，宿主尽力删除临时文件，但清除旧 `.tmp` 失败时可能残留该临时文件。
- `0025` 持久化降级语句通过 `RAISE(ABORT, ...)` 明确拒绝降级并回滚，保留 `0025` 迁移记录，新版仍可正常重开。

### 禁止结果

- 沿用旧版回退前的过时备份或 0 字节备份，或因迁移编号变化漏掉备份。
- 备份失败后仍执行 `0025` 删表迁移，或损坏既有的有效备份文件。
- 允许把 `0025` 降级回 pre-0025 并使库无法被新版重开。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 按 `effect_*` 表存在判断触发 `VACUUM INTO` 与原子替换（每次 pre-0025 启动刷新、替换 0 字节、跳过全新/已完成库） | [`refreshes_self_contained_pre_0025_backup_and_skips_fresh_or_post_0025_databases`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L582)、[`keys_pre_0025_backup_on_legacy_effect_tables_instead_of_recorded_version`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L717)、[`clears_stale_temp_file_and_replaces_existing_backup`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L789) | `Covered` |
| 清除旧 `.tmp`、`VACUUM INTO` 或原子重命名失败时保留已有备份并中止迁移；`VACUUM INTO` 或重命名失败时尽力清理 `.tmp`（清除旧 `.tmp` 失败时可能残留） | [`preserves_existing_backup_when_stale_temp_cleanup_or_vacuum_fails`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L817)、[`removes_temp_file_when_final_backup_rename_fails`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/legacy_effect_cleanup/tests.rs#L867) | `Covered` |
| 备份失败时 `Backend::open` 报错且不执行 `0025` 迁移 | [`fails_open_without_migrating_when_pre_0025_backup_fails`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/backend/src/bootstrap.rs#L637) | `Covered` |
| `0025` 迁移不可逆，旧版降级被拒绝并回滚，新版仍可正常重开 | [`downgrade_from_0025_is_rejected_and_newer_build_still_reopens_cleanly`](https://github.com/ora-space/desktop/blob/9f3ed63fa3f36701d4d2f8ca25414ea018d779cf/crates/db/src/migration/schema/schema_v0025.rs#L262) | `Covered` |

### 决策依据

- Skill 会话清单决策 D9：升级前自动备份与 `0025` 不可逆边界。
- 不变量 11。
