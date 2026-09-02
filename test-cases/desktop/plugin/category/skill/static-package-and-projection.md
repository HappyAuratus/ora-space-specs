# Skill 插件静态包与 Catalog 投影

## Invalid Contributed Skill Must Reject The Whole Plugin Snapshot

### 风险

如果插件中的部分非法 Skill 被静默跳过，同一个 Plugin ID/version 会因文件枚举顺序或解析差异形成
不同贡献集合；UI 也无法准确表达“插件已安装但只安装了一部分”。

### 前置状态

- installed package 的 `orax.toml` 声明 `kind = "skill"`。
- `assets/` 缺失、为空，或至少一个直接 Skill 子目录缺少/包含非法 `SKILL.md`。

### 触发

`PluginManager::discover` 读取最高 installed version 并验证 Skill contribution。

### 必须成立

- `kind = "skill"` 由 release 与 installed manifest schema 接受，且拒绝其他 kind 的专属 section。
- `assets/` 必须至少包含一个完整直接子目录 Skill package。
- 任一 contributed Skill 非法时，整个插件不进入 installed snapshot，并产生定位到 `skill` 的 discovery
  issue。
- 不回退到同一插件较低的旧版本，也不返回只包含有效 Skill 的部分 descriptor。

### 禁止结果

- 把嵌套任意深度的 `SKILL.md` 自动暴露为插件贡献。
- 忽略一个坏 Skill 后继续发布同包其他 Skill。
- 因最高版本损坏而静默恢复较低版本。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| release 与 installed manifest 接受 Skill kind，installed resolver 可省略 | `ora_plugin_manifest::tests::parses_skill_kind_marketplace_manifest`; `ora_plugin_manifest::tests::parses_installed_skill_manifest_without_resolver_or_download_fields` | `Covered` |
| Skill kind 拒绝 workbench/webview 专属 section | `ora_plugin_manifest::tests::skill_kind_rejects_workbench_and_webview_sections` | `Covered` |
| 一个或多个完整直接子目录被解析为贡献 | `ora_plugin_manager::tests::discovers_skill_plugin_with_required_skill_assets`; `ora_plugin_manager::skill::tests::accepts_one_or_more_skill_directories` | `Covered` |
| 缺失、空或不完整 assets tree 拒绝整个插件 | `ora_plugin_manager::tests::rejects_skill_plugins_without_complete_skill_assets`; `ora_plugin_manager::skill::tests::rejects_missing_empty_and_incomplete_skill_trees` | `Covered` |
| 任一多个 Skill 中的非法 manifest 都拒绝整包，而非部分接受 | 当前测试覆盖单个不完整 Skill；尚无一个包内“一个有效、一个非法”的直接测试 | `Partial` |
| 最高版本损坏时不回退 | `ora_plugin_manager::tests::does_not_fall_back_when_highest_version_is_broken` 覆盖通用插件发现规则，尚无 Skill 专属 fixture | `Partial` |

### 决策依据

- Skill 插件根决策 D1–D3：贡献集合来自固定浅层目录，且 installed snapshot 采用整包校验。
- Skill 插件根决策不变量 2：不存在部分接受的 Skill 插件。

## Plugin Skill Identity And Ownership Must Survive Catalog Round Trips

### 风险

如果 Catalog 丢失 Plugin owner、package root 或稳定 ID，应用层可能把插件内容当作本地 Skill 修改，
更新/卸载也可能碰到其他插件的同名 Skill。

### 前置状态

- 有效插件 `official/review-pack` 贡献 Skill `review`。
- Catalog 使用现有 `skills` 表持久化投影。

### 触发

后端启动或插件安装/更新同步贡献，随后通过 Skill API 读取、尝试修改，并最终显式卸载插件。

### 必须成立

- Skill ID 是 `plugin:official/review-pack:review`，namespace 是 `official/review-pack`。
- round trip 后 origin 仍包含 Plugin ID 和 immutable package root。
- API 返回 `SkillSource::Plugin`，从插件 package 读取正文，并拒绝用户更新和删除。
- 显式卸载按 Plugin ID 隐藏所有 active Skill 行，不影响其他 namespace。

### 禁止结果

- 用裸 `review` 作为持久 Skill ID。
- 把 marketplace namespace `official` 单独保存为 Skill namespace。
- 通过普通 Skill update/delete 修改插件安装目录。
- 卸载一个插件时按裸 Skill name 删除其他插件的同名贡献。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 启动发现会把 Plugin Skill 写入共享 Catalog，并公开 Plugin source | `ora_backend::bootstrap::tests::opens_with_plugin_skills_written_to_the_existing_database_schema` | `Covered` |
| repository round trip 保留 Plugin origin/package root，移除插件时撤销 Catalog 投影 | `ora_db::repository_tests::plugin_skill_projection_round_trips_and_is_removed_with_its_plugin` | `Covered` |
| Plugin Skill 从 package 读取正文并拒绝用户 mutation | `ora_application::skill::tests::plugin_skills_load_from_their_package_and_reject_user_mutations` | `Covered` |
| 安装、更新、本地 import 和卸载 API 的完整端到端路径都同步 Catalog | 当前有实现路径和分层测试，但没有覆盖四条入口的直接端到端测试 | `Partial` |
| 外部删除或损坏不会被误报为受控卸载 | 当前实现明确不做全量 stale projection sweep，尚无固定该限制的直接测试 | `Missing` |

### 决策依据

- Skill 插件根决策 D5–D7：身份包含 owner，Plugin Skill 只读，受控生命周期负责 Catalog 投影同步。
- Skill 插件根决策不变量 3、4、7：ID、ownership 与显式卸载边界必须稳定。
