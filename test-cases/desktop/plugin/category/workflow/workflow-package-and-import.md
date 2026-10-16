# Workflow 插件包与文档导入

## A Workflow Package Contributes Exactly Its Asset Directory Documents

### 风险

如果贡献集合取决于枚举顺序、文件系统细节或额外文件，同一个 Plugin ID 与版本会在不同机器上导入出
不同的工作流集合；UI 也无法说明「这个包带来了什么」。

### 前置状态

- installed package 的 `orax.toml` 声明 `kind = "workflow"`。
- 包内 `assets/workflows/` 下混有 `*.json` 文档、`README.md`、子目录与符号链接，或该目录缺失、
  为空、文件数超过上限。

### 触发

`PluginManager::discover` 读取最高 installed 版本并验证 Workflow contribution。

### 必须成立

- `kind = "workflow"` 由 release 与 installed manifest schema 接受，且拒绝 `[workbench]`、
  `[webview]` 与 `[artifact]`/`[[targets]]`。
- 贡献集合恰好是 `assets/workflows/` **直接子层**的普通 `*.json`（扩展名比较忽略 ASCII 大小写），
  按文件名稳定排序。
- `README`、其他扩展名、子目录与符号链接既不参与集合也不导致失败；符号链接不被跟随。
- `assets/workflows/` 缺失、不是目录、零份文档或超过 256 份时，产生定位到 `workflow` 字段的
  discovery issue。
- 包内没有其他 kind 的专属 section 时，workflow 包不因缺少 section 而失败。

### 禁止结果

- 递归发现或接受嵌套目录里的 `*.json`。
- 因一个多余文件而拒绝一个本来有效的包。
- 因文件枚举顺序不同而得到不同的文档集合或顺序。
- 把符号链接的目标读进贡献集合。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| release 与 installed manifest 都接受 workflow kind，且拒绝其他 kind 的专属 section 与 target 形态 | `ora_plugin_manifest::tests::parses_workflow_kind_installed_manifest`; `ora_plugin_manifest::tests::workflow_kind_rejects_workbench_and_webview_sections`; `ora_plugin_manifest::tests::workflow_kind_rejects_artifact_section` | `Partial` |
| 一份或多份直接子层文档被解析为贡献，并按文件名稳定排序 | `ora_plugin_manager::tests::discovers_workflow_package_documents`; `ora_plugin_manager::workflow::tests::accepts_one_or_more_workflow_documents` | `Covered` |
| README、子目录与其他扩展名被忽略而不导致失败 | `ora_plugin_manager::workflow::tests::ignores_files_that_are_not_workflow_documents` | `Covered` |
| 缺失、空目录与零份文档的包被拒绝 | `ora_plugin_manager::tests::rejects_workflow_package_without_documents`; `ora_plugin_manager::workflow::tests::rejects_missing_empty_and_documentless_asset_directories` | `Covered` |
| 超过文档数上限的包被拒绝 | `ora_plugin_manager::workflow::tests::rejects_more_documents_than_the_budget` | `Covered` |
| 符号链接被忽略且不可越过包根 | 实现按不跟随链接的文件类型判断，尚无符号链接 fixture 的直接测试 | `Partial` |

「都接受」这一行标为 `Partial`：release 形态目前只有 section 拒绝与 installed 接受两处直接证据，
还没有一份 release 形态的 workflow 清单解析测试。

### 决策依据

- Workflow 插件类别根决策 D1：workflow 是无进程、无专属 section、无 target 形态的 kind。
- Workflow 插件类别根决策 D2：贡献集合由浅层固定目录确定，多余条目忽略、零份文档拒绝。
- Workflow 插件类别根决策不变量 2。

## Discovering A Workflow Package Must Not Parse Its Documents

### 风险

如果发现阶段读取并解析每份文档，插件发现会变成读取用户文档的热路径，而且一份坏文档会连带影响整个
包的发现结果，让「这个包装没装上」与「其中一份文档是坏的」两件事混在一起。

### 前置状态

- 一个 `kind = "workflow"` 的包，其中至少一份文档不是合法 JSON 或不构成可执行图。
- 插件发现是所有 kind 共用的路径。

### 触发

发现该包，随后请求导入它的文档。

### 必须成立

- 发现只枚举文档路径并返回描述符，不读取内容、不解析 JSON。
- 文档内容错误属于导入结果，不属于 discovery issue：包本身仍然被发现为有效贡献。
- 读取一份文档失败（数据目录不可用）是内部错误，而不是该文档的内容错误。

### 禁止结果

- 在发现阶段解析文档并据此拒绝整包。
- 把文档内容错误报成 discovery issue 或安装失败。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 发现阶段不解析文档内容，坏文档不影响包的发现结果 | `ora_plugin_manager::tests::discovers_workflow_package_documents` 用内容为 `{}` 的文档（不构成工作流文档）证明枚举阶段不解析内容；尚无一份「有效包内混入非法文档」后仍导入其余文档的显式断言 | `Partial` |
| 文档内容错误在导入阶段逐文档报告 | `ora_application::workflow::import::tests::reports_one_bad_document_without_stopping_the_batch`; `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin` | `Covered` |

### 决策依据

- Workflow 插件类别根决策 D3。
- Workflow 插件类别根决策不变量 3。

## Importing One Document Must Never Cost Another Its Workflow

### 风险

如果一份文档失败会让整批导入回滚，用户会因为一个坏 JSON 丢失同包其他可用的工作流；如果失败被静默
跳过，用户又会以为模板齐了。

### 前置状态

- 一个包携带多份文档，其中至少一份是无效 JSON、缺少名称、图不可执行、与既有工作流同名，或带有
  不可寻址的版本候选（例如声明 `draft`、`a/b` 或空白）。
- 工作流库的 tenant 级名称唯一性规则与发布门禁都生效。

### 触发

导入该包的全部文档。

### 必须成立

- 返回与输入同序的逐文档结果：成功带 `workflow_id` 与发布版本，失败带原因（无效 JSON、未声明
  名称、图不可执行、名称冲突，以及创建或发布环节自身的失败）。
- 一份失败不改变其余文档的结果，也不移除投递它的包。
- 一份成功的文档产生一份工作流与一个已发布快照；名称冲突不改名、不覆盖。
- 发布版本按 文档自述 `version` → 文件名词干 → 工作流标题（词干为空时）的优先级取；不可寻址的
  候选不被采用、也不被清洗，三个来源都不可用时按自动版本发布，文档仍然导入。
- 存储的图是文档原文。

### 禁止结果

- 整批回滚或整批拒绝。
- 静默跳过失败文档而不报告。
- 名称冲突时自动改名或覆盖既有工作流。
- 只建 workflow 不发布快照。
- 改写文档内容后再存储。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 每份文档各建一份工作流并发布一个快照 | `ora_application::workflow::import::tests::imports_each_document_and_publishes_a_snapshot`; `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin` | `Covered` |
| 一份坏文档独立失败而不影响同批其余文档 | `ora_application::workflow::import::tests::reports_one_bad_document_without_stopping_the_batch` | `Covered` |
| 未声明名称与不可执行图被拒绝 | `ora_application::workflow::import::tests::refuses_documents_without_a_name`; `ora_application::workflow::import::tests::refuses_documents_whose_graph_cannot_execute` | `Covered` |
| 名称冲突逐文档失败，不改名也不覆盖 | `ora_application::workflow::import::tests::refuses_a_duplicate_workflow_name` | `Covered` |
| 发布版本按优先级派生，不可寻址的候选不被采用、也不被清洗 | `ora_application::workflow::import::tests::derives_publish_versions_in_precedence_order` | `Covered` |
| 安装结局与导入结局分别返回，导入失败不移除包 | `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin` 同一次响应里同时给出 `outcome` 与逐文档结果 | `Covered` |

「回落自动版本」那一支的直接证据在发布处理器一侧：`ora_application::workflow::tests::publish_uses_the_injected_clock_for_automatic_versions` 与 `…::publish_retries_automatic_versions_that_collide_at_the_same_clock_value`。导入侧只断言派生函数对不可寻址候选返回 `None`，还没有一条端到端的「词干不可寻址 → 文档导入成功且拿到自动版本」测试。

### 决策依据

- Workflow 插件类别根决策 D4。
- Workflow 插件类别根决策不变量 4。
- [cloud/plugin-marketplace 后续决策](../../../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md) D5：
  安装成功、接管成功、发布可用、执行可用是四个可以分别判断的事实。

## An Imported Workflow Outlives The Package That Delivered It

### 风险

如果副本随包卸载被删除、或被包升级覆盖，用户对副本的编辑会静默丢失；用户也会因为怕丢数据而不敢改
导入进来的工作流。

### 前置状态

- 已经从某个 workflow 包导入了一份工作流，用户可能编辑、改名或归档它。
- 随后显式卸载该插件，或安装该插件的新版本。

### 触发

卸载插件；或以更高版本重新安装同一插件。

### 必须成立

- 卸载插件不删除、不改写已导入的工作流与它的既有快照。
- 市场安装与升级路径不读取包内文档，因此不会覆盖用户副本。
- 副本可编辑、可改名、可归档；没有任何路径把它当作插件拥有的只读对象。
- 副本的身份与命名空间与插件无关（`local` 用户数据），插件身份不出现在工作流行的身份里。

### 禁止结果

- 卸载时删除或归档导入的工作流。
- 升级时按包内内容覆盖或新建同名副本。
- 把导入的工作流标成插件拥有、只读，或随插件卸载隐藏。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 卸载只清理插件 Skill 投影与 agent 声明，不触碰工作流 | 实现路径为 `PluginHost::uninstall`（`remove_plugin_skills` 与 `replace_agent_effect_declaration`），尚无「导入后卸载仍可读」的直接测试 | `Partial` |
| 升级/市场安装路径不导入文档，因此不会覆盖用户副本 | 文档只在本地 `.orax` 导入路径被读取（`PluginHost::import` → `workflow_import.handle`），尚无升级路径的直接测试 | `Partial` |
| 导入副本是 `local` 命名空间的普通工作流，插件身份不进入它的身份 | `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin` 断言插件 id 为 `local/ora.workflows` | `Covered` |

### 决策依据

- Workflow 插件类别根决策 D5、D6 与不变量 5、6。
- [registry 决策](../../../../../decisions/desktop/plugin/registry/20260831-source-derived-namespace.md)：
  `workflows` 与 `agents` 的 namespace 恒为 `local`，插件不向它们贡献任何行。
- [cloud/plugin-marketplace 后续决策](../../../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md) D1、D5。
