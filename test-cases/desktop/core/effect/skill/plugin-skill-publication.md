# Plugin Skill Source 发布与收敛

## Plugin Skill Publication Must Converge Through Effect Ownership

### 风险

如果 Catalog 同步直接把插件目录复制到 Workspace，可能覆盖用户内容、丢失 revision 证据，或让插件
更新绕过通用 Effect 的 generation、ownership、journal 和恢复协议。

### 前置状态

- Plugin Skill Catalog projection 已提供 package root、插件版本、manifest digest 和 package
  fingerprint，并已发布对应 immutable Effect revision。
- Workspace Scope 与支持 Skill materialization 的 Consumer/Resource 已存在。

### 触发

Effect worker 为当前 generation 规划并应用 Plugin Skill Source revision。

### 必须成立

- publication 生成 plugin-kind Effect Source 和 immutable revision，而不是直接修改 Workspace。
- 新 Source 进入现有 Workspace Desired State，并推进 Scope generation。
- materialization 从发布的 package root 复制完整 Skill package，写入前仍受 Resource ownership 与
  adapter journal 约束。
- 同一 revision 已收敛后重放为 no-op；Source retirement 通过下一 generation 清理受管内容。

### 禁止结果

- Catalog transaction 内直接写 Workspace filesystem。
- 只复制 `SKILL.md` 而丢失同 package 的 scripts、references 或 assets。
- 依据同名目录推断 Ora ownership 并覆盖未管理内容。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| Plugin Skill publication 为现有 Workspace 添加 Desired 并推进 generation | `ora_db::effect_repository_tests::source_publication_changes_each_complete_scope_generation_once` | `Covered` |
| 完整 Target generation 能从 Plugin Skill revision 物化并 finalize | `ora_db::effect_repository_tests::reconciler_materializes_and_finalizes_one_complete_target_generation` | `Covered` |
| 已收敛 generation 重放不新增 operation | `ora_db::effect_repository_tests::reconciler_materializes_and_finalizes_one_complete_target_generation` | `Covered` |
| Source retirement 清理受管结果且不碰未拥有内容 | repository round-trip 测试覆盖 Desired retirement；完整 filesystem ownership 风险由 Effect 核心用例覆盖 | `Partial` |
| 完整 package 中非 manifest 文件也随 Plugin Skill materialization 保留 | 当前 adapter 的目录 fingerprint/copy 测试提供分层证据，缺少 Plugin Skill 入口的直接端到端 fixture | `Partial` |

### 决策依据

- Skill Effect 根决策 D1–D4：Catalog projection 与 publication 分离，revision 绑定完整 package，
  publication 通过 generation 和 ownership 协议收敛。
- Skill Effect 根决策不变量 1–7：Source、revision、Desired、materialization 与 ownership 边界必须
  保持独立。
- Effect 系统基础决策不变量 5–8、10–11：Observed State 不授予 ownership，Resource mutation 必须
  持有 fenced authority，并以完整 projection 和 durable journal 推进。
