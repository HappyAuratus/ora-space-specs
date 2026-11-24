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
| 扩展名比较忽略 ASCII 大小写（`.JSON` 与 `.json` 等价） | 实现按 `name.to_ascii_lowercase()` 判断后缀，尚无一份大写扩展名的 fixture | `Missing` |
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
| 发现阶段不解析文档内容，坏文档不影响包的发现结果 | `ora_plugin_manager::tests::discovers_workflow_package_documents` 用内容为 `{}` 的文档证明枚举阶段不解析内容；`crates/backend` 的 `retry_workflow_import_reconsiders_only_unrecorded_paths` 在市场包里混入 `{ not json` 与一份有效文档，安装仍然成功，坏文档只在逐文档结果里失败 | `Covered` |
| 文档内容错误在导入阶段逐文档报告 | `ora_application::workflow::import::tests::reports_one_bad_document_without_stopping_the_batch`; `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin`; `crates/backend` 的 `retry_workflow_import_reconsiders_only_unrecorded_paths` | `Covered` |

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
  名称、图不可执行，以及创建或发布环节自身的失败）。本地导入的名称冲突仍逐文档失败；市场自动导入
  的冲突与已处理路径按下面的生命周期用例区分跳过结果。
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
| 本地导入名称冲突逐文档失败，不改名也不覆盖 | `ora_application::workflow::import::tests::refuses_a_duplicate_workflow_name` | `Covered` |
| 发布版本按优先级派生，不可寻址的候选不被采用、也不被清洗 | `ora_application::workflow::import::tests::derives_publish_versions_in_precedence_order` | `Covered` |
| 本地导入的安装结局与导入结局分别返回，导入失败不移除包 | `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin` 同一次响应里同时给出 `outcome` 与逐文档结果 | `Covered` |

「回落自动版本」那一支的直接证据在发布处理器一侧：`ora_application::workflow::tests::publish_uses_the_injected_clock_for_automatic_versions` 与 `…::publish_retries_automatic_versions_that_collide_at_the_same_clock_value`。导入侧只断言派生函数对不可寻址候选返回 `None`，还没有一条端到端的「词干不可寻址 → 文档导入成功且拿到自动版本」测试。

### 决策依据

- Workflow 插件类别根决策 D4。
- Workflow 插件类别根决策不变量 4。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md)
  D2、D4：市场同名冲突持久跳过，安装与逐文档结果分开呈现。
- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D1–D3：市场冲突可恢复并由安装和卡片重试、以及升级首次冲突打开「未导入」报告弹窗。
- [cloud/plugin-marketplace 后续决策](../../../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md) D5：
  安装成功、接管成功、发布可用、执行可用是四个可以分别判断的事实。

## An Imported Workflow Outlives The Package That Delivered It

### 风险

如果副本随包卸载被删除、或被包升级覆盖，用户对副本的编辑会静默丢失。若用「市场不导入」保护副本，
安装后又没有可用工作流；若升级每次重导，改名会产生重复副本，删除后会自动恢复，历史同名工作流还会
反复触发失败提示。

### 前置状态

- 一个市场 workflow 包或包含 workflow 成员的 pack，包内有多份文档。
- 已导入副本可能被用户编辑、改名、归档或删除；来源记录可能为已导入或因同名冲突跳过。
- 旧版已安装包尚无来源记录，工作流库可能有历史本地导入或用户自建的同名文档。
- 新版本既含旧路径（内容可变化），也含新增路径；两个来源可使用相同 identifier 与文档路径。

### 触发

市场首次安装、升级、pack 成员安装；导入后卸载重装；删除副本或解除同名冲突后再次升级、重试或重装。

### 必须成立

- 卸载插件不删除、不改写已导入的工作流与它的既有快照。
- 市场首次安装导入包内文档；升级重判无记录与 `conflict_skipped` 路径，并尊重当前安装周期内已删除的副本；pack 仅对 `InstallOutcome::PackInstalled.members` 中本次实际安装成功的 workflow 成员开启新周期并按各自身份导入。
- 来源键为 `(source_namespace, identifier, document_path)`，不含版本、内容散列或工作流名称。
  同路径的新内容仍跳过，新增路径独立导入，不同来源不共享记录。
- 存活的 `imported` 副本及其来源记录跨升级、改名、归档与卸载重装保留，不重复导入、不覆盖、不改名；
  当前安装周期内，升级和卡片重试尊重用户对副本的删除（且升级里已删除副本的结果与存活副本的 `SkippedProcessed` 可区分，不显示为「此前已处理」）；成功市场重装（含卸载后才删除或 0.6.0 遗留记录）按当前存活状态重新评估已删除副本，并重新检查 `conflict_skipped` 路径（冲突消除后尝试导入，仍冲突时保持 `conflict_skipped` 并返回 `SkippedConflict`）。
- 旧安装首次升级补导入无冲突文档；已有同名工作流（按 `COLLATE NOCASE` 比较）保留，来源记为 `conflict_skipped` 并按首次冲突打开报告弹窗；之后已知冲突（本次操作读到的前状态为 `conflict_skipped` 且仍同名）在升级时仅以中性摘要计数，冲突消除后升级可再次导入，仍不凭名称将历史本地导入或自建工作流认作市场来源。
- 本地 `.orax` 导入继续显式逐文档处理，重复同名仍失败，不使用市场来源记录跳过。
- 副本可编辑、可改名、可归档；没有任何路径把它当作插件拥有的只读对象。
- 副本的身份与命名空间与插件无关（`local` 用户数据），插件身份不出现在工作流行的身份里。

### 禁止结果

- 卸载时删除或归档导入的工作流。
- 升级时按包内内容覆盖或新建同名副本。
- 因存活副本改名或归档而重导旧路径；因包版本变化或卸载而清除存活副本来源记录，或在当前安装周期内的升级/卡片重试中复活用户已删除的副本。
- 把所有 pack 成员的文档记在 pack 身份下，对未实际安装的 pack 成员开启新周期，或把一个来源的记录用于另一个来源。
- 每次升级重复弹窗报告已知同名冲突、把已知冲突或已删除副本显示为「此前已处理」，或仅凭名称给历史工作流补记成功来源。
- 把导入的工作流标成插件拥有、只读，或随插件卸载隐藏。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 卸载只清理插件 Skill 投影与 agent 声明，不触碰工作流 | `crates/backend` 的 `reinstalling_the_package_does_not_import_the_documents_again`（待按新规则改写，收窄为存活副本不重导）走过卸载与重装，重装后工作流库里仍有卸载前导入的存活副本；Skill 投影与 agent 声明的清理范围仍由 `PluginHost::uninstall` 的实现路径保证 | `Partial` |
| 市场首次安装导入全部可用文档；pack 按成员身份导入 | `crates/backend` 的 `marketplace_install_imports_workflow_documents`; `crates/backend` 的 `pack_install_imports_workflow_documents_of_its_members` | `Covered` |
| 升级只增不改：同路径内容变更跳过，新版本新增的路径独立导入 | `crates/backend` 的 `marketplace_upgrade_imports_only_new_documents` | `Covered` |
| 用户对副本的编辑、改名和归档在升级后保持不变 | `crates/backend` 的 `marketplace_upgrade_imports_only_new_documents` 断言升级不重写已交付副本的图；改名与归档仍无直接断言 | `Partial` |
| 来源键隔离不同 namespace 的同名插件，且跨版本稳定 | `crates/db` 的 `commits_the_copy_and_its_record_together` 与 `crates/backend` 的 `marketplace_upgrade_imports_only_new_documents` 证明同一来源路径跨版本只认一次；不同 namespace 使用相同 identifier 的隔离仍无直接断言 | `Partial` |
| 旧安装首次升级补导入；历史本地导入及自建同名保留，首次冲突弹窗而已知冲突在升级保持中性摘要 | 旧测试 `crates/backend` 的 `marketplace_install_reports_a_name_conflict_once` 仍按前序「冲突一次后转为 `SkippedProcessed`」语义断言，待在 Desktop 实现运行改写；「旧安装没有来源记录时的首次升级补导入」与「已知冲突升级中性摘要」尚无直接断言 | `Missing` |
| 存活副本来源记录跨卸载重装、重启保留；当前安装周期内已删除副本不复活且升级可与存活跳过区分 | 旧测试 `crates/backend` 的 `reinstalling_the_package_does_not_import_the_documents_again`、`a_deleted_workflow_is_not_resurrected_by_an_upgrade_or_retry` 与 `crates/db` 的 `a_record_survives_the_workflow_delete_and_a_later_reinstall` 按前序「卸载重装永久不恢复」语义编写，待在 Desktop 实现运行改写；重装恢复已删除副本、升级已删除副本可与 `SkippedProcessed` 区分以及进程重启后读取尚无直接断言 | `Missing` |
| 本地导入不受市场来源记录抑制，同名仍失败 | `crates/backend` 的 `importing_the_same_document_name_twice_refuses_the_second_import` 证明本地导入仍按同名拒绝；与市场来源记录共存的场景仍无直接断言 | `Partial` |
| 导入副本是 `local` 命名空间的普通工作流，插件身份不进入它的身份 | `crates/backend` 的 `imports_workflow_package_documents_alongside_the_plugin` 断言插件 id 为 `local/ora.workflows` | `Covered` |

### 决策依据

- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D1–D2 与不变量 1–4：按安装周期与当前存活/同名状态判定导入、重装恢复已删除副本与保留存活来源。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md)
  D1–D3 与不变量 1–3：市场触发范围与来源键定义；冲突重判、安装周期划分及重装恢复由上述后续决策修改。
- Workflow 插件类别根决策 D5 的用户数据生命周期、D6 与不变量 5、6；市场不导入的旧触发限制由上述后续决策修改。
- [registry 决策](../../../../../decisions/desktop/plugin/registry/20260831-source-derived-namespace.md)：
  `workflows` 与 `agents` 的 namespace 恒为 `local`，插件不向它们贡献任何行。
- [cloud/plugin-marketplace 后续决策](../../../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md) D1、D5。

## Retrying Marketplace Documents Must Preserve Completed Imports

### 风险

插件安装成功不等于文档全部导入成功。创建副本和记录来源之间发生失败，可能留下无法恢复的半成品，
或让重试创建重复副本；把所有失败都记为已处理则会永久丢失本可重试的文档。

### 前置状态

- 单包或 pack 成员的文档混有有效文档、同名冲突、非法文档或暂时无法读写的文档。
- 插件安装可成功，文档创建、发布快照或保存来源时可失败或中断；已有成功与冲突跳过记录。

### 触发

执行市场安装、重装或升级；在文档提交与状态转换边界注入失败、并发同名或中断；安装成功后修复失败条件或解除同名冲突，从已安装的 workflow 插件卡片触发「重试导入未导入的文档」，包括重启后重试；升级也重新处理无记录与 `conflict_skipped` 路径。

### 必须成立

- `InstallPluginResponse` 与 `UpdatePluginResponse` 分别报告安装结局和逐文档结局，pack 结果可定位
  成员插件及文档路径；`SkippedConflict` 携带冲突工作流名及按前状态计算的已知冲突标记。前端未交付非成功样式只用于没有任何 `Imported` 且至少有一份本次新产生的 `Failed`、首次冲突，或安装/卡片重试中的冲突；升级中已知冲突与在装期间已删除副本均按中性摘要呈现，不改变成功样式。
- 文档失败不撤销成功安装，不中断其他文档；未成功安装的成员不创建副本，成功成员不被其他成员的
  文档失败回滚。
- 一份成功文档的工作流、首个已发布快照与来源记录在同一事务中以读到的前状态为条件原子提交（条件 `UPDATE` / `INSERT`）；并发另一操作先改写来源时按最新状态重判，不挂出第二份存活副本；名称检查后若并发写入先占用同名（`COLLATE NOCASE`），回滚创建并按 `SkippedConflict` 报告、保留或提交 `conflict_skipped`，不把同名约束或已有来源记录误报为 `Failed`。重装恢复已删除副本遇非同名失败时，在同一次操作里以读到的前状态为条件删除该 `imported` 记录回到「无记录」，仅当该条件删除自身失败时退回「修复后再次卸载重装」。
- 名称冲突记为 `conflict_skipped`，解除同名后可经卡片重试、升级或重装再次导入；普通失败不写入阻挡重试的终态记录（无记录或 `conflict_skipped` 保留原状态），修复条件后仍可重试。
- 已安装的 workflow 插件卡片（包括 pack 的 workflow 成员卡片）提供「重试导入未导入的文档」动作，
  不要求卸载或等待新版本；动作返回并展示逐文档结果、摘要与「未导入」报告弹窗（有冲突时），对当前安装周期内已删除副本提示「副本已删除，卸载后重装可恢复」。
- 卡片动作与升级都跳过存活的 `imported` 副本，尊重当前安装周期内已删除的副本（升级结果可与存活副本的 `SkippedProcessed` 区分，不显示为「此前已处理」），并重新处理无记录路径及 `conflict_skipped` 路径；不通过覆盖用户数据来恢复。

### 禁止结果

- 把插件安装成功当成全部文档成功，或只在日志里留下文档失败。
- 安装成功后因为插件已存在而永远无法重试失败文档或已解除的同名冲突。
- 创建成功而来源记录丢失导致重试生成重复副本；发布失败留下妨碍重试的同名半成品；因纯 `INSERT` 主键冲突或并发同名把 `conflict_skipped` / `SkippedConflict` 误报为 `Failed`。
- 给普通失败写终态来源记录导致永久跳过、在条件删除可行时保留已删除副本 `imported` 挡住恢复非同名失败重试，或整批回滚已成功文档。
- 在升级中复活当前周期已删除的副本，或把升级已知冲突、升级已删除副本展示为非成功样式或「此前已处理」。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 安装与升级响应逐文档返回，pack 成员可归因 | `crates/backend` 的 `marketplace_install_imports_workflow_documents`; `crates/backend` 的 `marketplace_upgrade_imports_only_new_documents`; `crates/backend` 的 `pack_install_imports_workflow_documents_of_its_members` | `Covered` |
| 前端安装、升级与重试提示按规则区分成功样式、中性摘要与未交付非成功样式，且展示逐文档计数 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` 的 `reports the workflow documents a marketplace install imported` 只断言已导入计数；非成功样式（无 `Imported` 且含新 `Failed`/首次冲突/安装或重试冲突）、升级已知冲突中性计数与升级已删除副本中性摘要均无直接断言 | `Missing` |
| 单包及 pack 文档失败彼此隔离，安装失败不创建对应副本 | `crates/backend` 的 `retry_workflow_import_reconsiders_only_unrecorded_paths` 证明一份非法文档不阻断同批其余文档且安装仍成功；pack 成员安装失败不创建副本仍无文档侧直接断言 | `Partial` |
| 创建、发布和来源记录以读到的前状态为条件原子提交，并发同名回滚为 `SkippedConflict`，恢复非同名失败条件删回无记录 | `crates/db` 的 `commits_the_copy_and_its_record_together`; `crates/db` 的 `a_failed_commit_leaves_neither_a_copy_nor_a_record`; 迁移约束见 `crates/db` 的 `keeps_one_terminal_state_per_source_path`；条件 `UPDATE`/`DELETE`、并发同名回滚报 `SkippedConflict`、并发来源重判以及发布失败/进程中断注入尚无直接断言 | `Partial` |
| 安装成功后可重试普通失败与已解除的 `conflict_skipped`，并跳过存活副本及尊重周期内删除 | `crates/backend` 的 `retry_workflow_import_reconsiders_only_unrecorded_paths` 仅在修复无记录坏文档后重试；`conflict_skipped` 重试与周期内已删除副本跳过、重装恢复非同名失败后的本周期重试及重启后重试尚无直接断言 | `Partial` |
| 已安装 workflow 插件卡片的「重试导入未导入的文档」动作重判无记录与 `conflict_skipped` 路径，展示冲突报告与已删除副本提示 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` 的 `retries failed workflow document imports from the installed card` 与 `crates/backend` 的 `retry_workflow_import_reconsiders_only_unrecorded_paths` 仅覆盖旧文案与无记录失败路径；新按钮文案、重判 `conflict_skipped`、打开「未导入」报告弹窗及「副本已删除，卸载后重装可恢复」提示尚无直接断言 | `Partial` |
| 升级顺带重试无记录失败与 `conflict_skipped` 路径，跳过存活副本并对周期内已删除副本输出可区分的中性结果 | `crates/backend` 的 `marketplace_upgrade_imports_only_new_documents` 覆盖升级跳过存活路径与导入新增路径；「升级重试此前失败」「升级解除冲突后导入」「升级已知冲突中性摘要」与「升级已删除副本区分 `SkippedProcessed` 且不显示此前已处理」尚无直接断言 | `Partial` |

pack 的 workflow 成员卡片与单包卡片是同一个已安装卡片组件，按 `kind = workflow` 触发同一条
`retryWorkflowImport` 路径，因此没有单独针对 pack 的断言。

### 决策依据

- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D1–D3 与不变量 1–5：条件状态转换、冲突与恢复失败重试及结果呈现分层。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md)
  D1–D4 与不变量 3–5；冲突重判、非成功样式触发与卡片按钮范围由上述后续决策修改。
- [Workflow 插件类别根决策](../../../../../decisions/desktop/plugin/category/workflow/0-workflow-package-format.md)
  D4 的逐文档独立与首次发布规则。

## Marketplace Install And Uninstall Must Surface Workflow Dependencies Without Blocking Removal

### 风险

如果市场安装或升级含工作流的插件只弹成功 toast 而不展示工作流引用的 MCP / Skill 就绪状态，用户
直到实际运行才会发现缺插件或未配置；如果卸载被引用的 MCP / Skill 插件毫无提示，用户会误删工作
流依赖；若因被引用而硬阻止卸载，或因加载未完成/单图损坏就给出「无依赖」结论，则会误导操作或锁死
插件管理。

### 前置状态

- 存在携带 `kind = "workflow"` 成员或文档的市场插件，导入的工作流草稿或已发布快照引用了 MCP / Skill，
  或 Agent 节点存在 `"agentConfig": null`。
- 工作流库中部分工作流的草稿或已发布快照可能引用了待卸载的 Skill 或 MCP 插件；另一部分工作流在
  草稿中移除了绑定但已发布快照仍引用，或存在无法拉取/非法的图。

### 触发

1. 从市场卡片、详情页或 Pack 执行安装或升级（含卡片重试导入）并带回 `imported`、`failed` 或触发弹窗的 `SkippedConflict`（安装/重试的任意冲突，或升级的首次冲突）；或仅带回跳过/已知冲突/在装期间已删除副本结果。
2. 从已安装插件管理器、插件详情页、Skills 设置页删除插件 Skill、或 Pack 卸载确认弹窗触发卸载前影响检查。

### 必须成立

- 市场安装、Pack 安装与卡片重试响应含 `imported`、`failed` 或任意 `SkippedConflict` 时，以及升级响应含 `imported`、`failed` 或首次冲突时，弹出报告弹窗：对 `imported` 工作流沿用与本地 `.orax` 共用判定的逐工作流 MCP / Skill 依赖判定与去安装 / 去配置 / 去查看入口（当前页可直接跳转修复并关闭报告）；对 `SkippedConflict` 在独立的「未导入」分组逐项列出 `source_file`、冲突工作流名及恢复方法（Pack 条目同时显示成员身份并指向该 canonical Plugin ID 的已安装卡片）。每当报告弹窗因 `imported`、`failed` 或首次冲突打开时，同批次中的已知冲突也一并列入「未导入」分组并标为「仍未解决」；「未导入」条目不参与 MCP / Skill 依赖判定。安装/重试仅含存活副本 `SkippedProcessed`（及已删除副本），或升级仅含存活副本 `SkippedProcessed`、已知冲突与在装期间已删除副本时，不弹出报告弹窗，保持一行摘要。
- `agentConfig: null` 与 `undefined` 一致视为无插件绑定，既不使依赖报告弹窗崩溃，也不在卸载检查中
  误计为不可读。
- 四个卸载入口均同时扫描工作流的草稿与已发布快照，列出引用待卸载插件的工作流与节点并给出警告，但
  **只警告、不阻止**卸载确认。
- 目录或工作流列表/草稿正在初次加载或后台 stale 刷新时禁用卸载确认按钮，且刷新失败时不复用旧缓存
  图；单个工作流图不可读时只计入不可读提示而不丢失其余已确认依赖；目录整体读取失败时给出明确提示
  并允许用户继续确认。待卸载插件没有 MCP / Skill 目标时跳过逐工作流 `workflow.get` 扫描。

### 禁止结果

- 市场安装或升级含新导入/失败工作流时不展示缺失/未配置依赖，或与本地导入使用两套互相矛盾的依赖判定。
- 安装/卡片重试存在冲突或升级出现首次冲突时不弹出「未导入」报告明细；把「未导入」条目送入依赖判定；或在升级仅含已知冲突与已删除副本时弹出报告弹窗。
- `agentConfig: null` 导致安装报告渲染崩溃。
- 仅扫草稿而漏掉已发布快照中的引用，或在检查/刷新未完成时放开确认按钮。
- 卸载因被工作流引用而硬阻止，或目录/图读取失败时静默汇报「无依赖影响」。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 市场卡片、详情页、升级、Pack 安装与卡片重试在含导入/失败或需弹窗的冲突时弹出报告弹窗（分别展示依赖与「未导入」分组并支持修复跳转） | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` 的 `reports workflow dependencies after a marketplace install`、`reports workflow dependencies after a marketplace update`、`reports workflow dependencies after installing from the detail page`、`reports workflow dependencies after upgrading from the detail page`、`reports workflow dependencies after a Pack install`、`resolves dependencies in-pane when upgrading from the plugin manager`、`opens the MCP configuration editor from a marketplace card report`；`retries failed workflow document imports from the installed card` 在一条 `skipped_processed` 与一条 `failed` 并存时只断言弹窗打开，未断言修复跳转；安装/卡片重试冲突、升级首次冲突、已知冲突在弹窗开时标为「仍未解决」、Pack 成员卡片指向及「未导入」不进依赖判定尚无直接断言 | `Partial` |
| 安装与重试仅含跳过/已删除副本，或升级仅含存活跳过、已知冲突与已删除副本时保持摘要、不弹出报告 | 没有测试断言「无弹窗触发项时保持摘要不弹窗」（含全跳过、升级仅已知冲突与升级仅已删除副本），上述任何入口都没有；也没有测试调用 `marketplaceWorkflowReportInput` 或 `marketplaceReportOutcomes` | `Missing` |
| `agentConfig: null` 按无绑定处理，安装报告与卸载检查口径一致 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` 的 `reports a marketplace install whose Agent node has a null agent config`；`packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts` 的 `reads an explicit null agent config as a workflow with no dependencies` | `Covered` |
| 四个卸载确认入口扫描草稿与已发布快照，列出依赖工作流与节点并只警告不阻止 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` 的 `names the workflows that depend on a plugin before uninstalling it`、`names affected workflows before uninstalling from the detail page`、`names the workflows a Pack uninstall would break`、`names workflows whose published snapshot still binds a removed plugin`；`packages/app-shell/src/features/settings/atoms-settings.test.tsx` 的 `names workflows that depend on a plugin Skill before uninstalling from Skills settings`；`packages/app-shell/src/features/settings/workflow-dependency-impact.test.ts` | `Covered` |
| 检查与刷新期间禁用确认按钮，目录/单图异常与空目标分支行为准确且不阻断卸载 | `packages/app-shell/src/features/settings/plugins-settings.test.tsx` 的 `disables uninstall confirmation until the workflow impact check finishes`、`re-checks and disables uninstall confirmation while cached workflow data refreshes`、`allows uninstall with an explicit warning when workflow catalogs fail to load`、`keeps confirmed dependents visible when another workflow draft fails to load`、`does not reuse cached draft graphs when the refresh fails`、`skips fetching workflow drafts when the removed plugin owns no Skill or MCP targets`；`packages/app-shell/src/features/settings/use-workflow-uninstall-impact.test.ts` 的 `skips the workflow library when the catalogs resolve no bindable target` | `Covered` |

上述前端行为证据均在 `app-shell` 测试层通过类型化 handler 驱动真实设置组件与 hook 验证，未启动
Tauri/Rust 后端或真实远程市场。`retries failed workflow document imports from the installed card` 同时
包含 `skipped_processed` 与 `failed`，只证明弹窗打开；它以及所有其他入口都没有断言「无弹窗触发项时不弹窗」与冲突「未导入」分组，
`marketplaceWorkflowReportInput` / `marketplaceReportOutcomes` 也没有直接调用测试。

### 决策依据

- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D3 与不变量 5：冲突打开「未导入」报告弹窗、已知冲突附加展示、升级静默规则及「未导入」不参与依赖判定。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md)
  D4（安装、升级与重试的逐文档结果及依赖报告；卸载被工作流引用的 MCP / Skill 插件）与不变量 5、7；「全部冲突跳过不弹窗」的前序规则已由上述后续决策修改。

## Uninstalling A Workflow Plugin Keeps Its Copies And Marks Their Uninstalled Source

### 风险

卸载 workflow 插件时如果只保持工作流库原样而没有任何说明，用户既不知道已导入副本是否会被删除，
也无法在工作流库和创建运行入口分辨哪些工作流来自已卸载插件；反过来，如果把来源已卸载当作不可用
并禁用编辑或运行，则会违背「副本属于用户数据且运行不依赖来源包」的契约。

### 前置状态

- 工作流库中包含：① 由市场 workflow 插件导入的副本（可能已被用户改名，或部分已被用户删除）；
  ② 本地 `.orax` 导入或手工自建的工作流；③ 与包内同名的 `conflict_skipped` 原有工作流。
- 部分来源插件已卸载，部分仍在装或随后重装同一 canonical Plugin ID（或装了不同 `namespace` 下的同名
  `identifier`）；或已安装插件列表正在加载/加载失败。

### 触发

1. 在已安装管理器、插件详情页或 Pack 卸载确认弹窗中发起卸载 `kind = "workflow"` 插件（Pack 含
   `remove` 成员与 `preserve` 成员）。
2. 确认卸载后浏览工作流库列表与工作区创建运行菜单，点击编辑或发起运行；随后重装同一 canonical
   Plugin ID。

### 必须成立

- 后端在 `WorkflowSummary` DTO 中提供 `sourcePluginId`，工作流库与创建运行菜单读取同一摘要，不各自
  在前端 join 来源。映射来自 `workflow_document_imports`（`state = 'imported'`），保持工作流
  `namespace` 为 `local`、改名后关联不变、已删除副本不列出，本地导入/自建/`conflict_skipped` 无来源
  插件 ID；本地 `.orax` 安装的 workflow 插件没有市场导入记录，其副本不参与保留列表和来源标记。
- 卸载 workflow 插件（含 Pack 的 `remove` 成员）时，确认弹窗在检查完成前禁用确认按钮；完成后列出
  该插件导入且当前仍存在的工作流，说明卸载会保留它们且可继续编辑和运行。Pack 中的 `preserve`
  成员不触发此列出。工作流库或来源记录读取失败时明确提示列表无法完整确认，检查结束后仍允许继续
  卸载，不把「读不到」当成「没有」。
- 「来源插件已卸载」由导入记录与已安装插件目录共同派生，包括功能上线前已经卸载的来源；工作流库与
  工作区创建运行菜单均对这些工作流显示该标记。标记纯属信息提示，编辑、重命名、导出、删除与打开运行
  弹窗不受阻碍，也不参与运行前或画布上的缺依赖判定。
- 重装同一 canonical Plugin ID 后标记消失，且存活副本不重复导入、不产生重复副本（此前已删除副本的重装恢复见下方重装恢复用例）；不同 `namespace` 的
  同名插件不消除原标记；已安装插件列表加载中或加载失败时不误标「来源插件已卸载」。

### 禁止结果

- 卸载 workflow 插件时删除、归档、改写或禁用已导入的工作流副本，或阻碍编辑与创建运行。
- 在确认列表中列出已被用户删除的工作流、本地自建/本地导入工作流、`conflict_skipped` 既有工作流，
  或 Pack 中属于 `preserve` 桶的成员导入的工作流。
- 凭工作流名称推断并回填旧安装来源，或在重装后为存活副本重复创建工作流副本。
- 把 plugin identity 写入 `workflows.namespace`，或在已安装插件目录失败时误标所有市场导入工作流。
- 将工作流库或来源记录读取失败当作没有保留副本，或因此永久禁用卸载确认。
- 把「来源插件已卸载」作为缺失依赖或禁止运行的理由。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `WorkflowSummary` DTO 提供 `sourcePluginId`（保留 `local`、改名保持、删除排除，本地导入/自建/冲突无来源，本地 `.orax` 安装不参与列表和标记） | [SQLite 仓储的来源推导测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/crates/db/src/repository_tests.rs) `list_workflows_derives_source_plugin_id_from_imported_records` 覆盖改名、软删除、自建和冲突。[真实安装、卸载和重装测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/crates/backend/src/plugin/operations/install_tests.rs) `workflow_summary_tracks_marketplace_source_across_uninstall_and_reinstall` 对照本地 .orax 与市场来源；无来源条目在前端由 [来源状态测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/packages/app-shell/src/state/hooks/use-uninstalled-workflow-sources.test.ts) 验证跳过。 | `Covered` |
| 单插件与 Pack（仅 `remove` 成员）卸载确认在检查中禁用确认，列出存活导入工作流及保留可运行说明；读取失败明确提示且仍允许卸载 | [卸载弹窗测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/packages/app-shell/src/features/settings/plugins-settings.test.tsx) 的 `lists surviving imported workflows when uninstalling a workflow plugin from the manager` 断言等待和改名后的保留清单；同文件覆盖详情页、Pack 仅移除成员、工作流库失败仍可确认；`names the workflows a Pack uninstall would break` 在计划返回后挂起工作流草稿读取，断言确认按钮禁用，完成检查后恢复。 | `Covered` |
| 工作流库与创建运行菜单从同一摘要派生「来源插件已卸载」（含功能上线前已卸载），编辑、重命名、导出、删除与发起运行不受阻碍，且不参与缺依赖判定 | [工作流库测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx) 覆盖进入编辑器、重命名、打开导出弹窗、删除，以及通过内存 handler 执行卸载后的自动刷新；[创建运行菜单测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/packages/app-shell/src/features/workspace/workspace-sidebar.test.tsx) 的 `marks workflows whose source plugin is uninstalled in the create-run menu without blocking runs, and clears on reinstall` 断言能打开创建运行弹窗。缺少提交导出、实际开始调度及画布和运行预检均独立于来源的直接断言。 | `Partial` |
| 重装同一 canonical Plugin ID 后标记消失且存活副本不重导；不同 `namespace` 同名与目录失败/加载中不误判 | [真实安装、卸载和重装测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/crates/backend/src/plugin/operations/install_tests.rs) 断言重装后同一存活副本且保持来源；[来源状态测试](https://github.com/ora-space/desktop/blob/8527b245c58e4adf15c1b83591414846dfb02796/packages/app-shell/src/state/hooks/use-uninstalled-workflow-sources.test.ts) 验证初次加载时无标记，空目录成功返回后显示标记。缓存该目录后刷新失败，测试等待查询进入 `error` 并断言标记清空。保存另一插件的配置后，配置写入缓存，目录仍为 `error`，标记仍为空。再次成功刷新空目录后标记恢复，重装同一 ID 后标记消失。[工作流库测试](https://github.com/ora-space/desktop/blob/6824ff903cb9025f248b6e41473caa25a06819ff/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx) 对照不同 namespace 的同名来源，只有精确重装的标记消失。 | `Covered` |

### 决策依据

- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D1–D2：重装保留存活副本及其来源、消除「来源插件已卸载」标记，并对已删除副本重新评估恢复。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md)
  D2、D4（卸载 workflow 插件的保留提示与「来源插件已卸载」标记）与不变量 1、2、7。
- [Workflow 插件类别根决策](../../../../../decisions/desktop/plugin/category/workflow/0-workflow-package-format.md) D5、D6。

## Clearing A Name Conflict Lets The Marketplace Document Import

### 风险

如果市场工作流首次遇到同名冲突就把来源路径写死为不可重试的终态，用户在工作流库里把同名工作流改名或删除后，无论卡片重试、自动升级还是卸载重装都会永远被「此前已处理」挡住而拿不到工作流；反过来，如果自动升级对一直未处理的已知冲突每次都弹出错误报告，或因纯 `INSERT` 主键碰撞和并发同名把冲突误报为 `Failed`，则会持续打扰用户或误导排查。

### 前置状态

- 本地工作流库已有同名存活工作流（含大小写不同但按 `COLLATE NOCASE` 同名），或存在 0.6.0 遗留的 `conflict_skipped` 来源记录。
- 市场 workflow 插件或 Pack 携带与上述工作流同名的文档；部分批次同时包含可导入文档、普通失败文档或已删除副本。
- 存在并发请求可能在同一文档检查名称后、提交事务前修改同一来源记录或创建同名存活工作流。

### 触发

1. 首次从市场安装遇到同名冲突；随后将已有工作流改名（含仅改大小写或改为其他名字）、改回原名后重试/重装、再删除冲突对象后从卡片点击「重试导入未导入的文档」、自动升级或卸载重装。
2. 升级遇到新增文档首次同名冲突、已有 `conflict_skipped` 已知冲突（单独出现，或与 `Imported`/`Failed`/首次冲突同批出现），以及 0.6.0 遗留 `conflict_skipped` 解除冲突后的首次升级。
3. 名称检查通过后、提交前并发写入抢先占用同名工作流或先改写同一来源状态。

### 必须成立

- 首次同名冲突保留现有工作流、记录 `conflict_skipped`，并返回带工作流名的 `SkippedConflict`。按 #1399 链路将冲突工作流改名后，卡片重试、升级与重装均可重新导入并改写为 `imported`；若在未导入前改回原名再试（包括仅大小写不同），再次触发 `SkippedConflict` 且不覆盖已有工作流；删除原冲突对象后重试、升级或重装均可成功导入。
- 「已知冲突」严格以本次操作读到的前状态为 `conflict_skipped` 且本次仍因同名跳过判定，与操作类型（安装、升级、卡片重试）无关；只要库里当前仍有同名存活工作流（不必是当初那一份），即判定为已知冲突。0.6.0 遗留的 `conflict_skipped` 直接参与重判，无需回填迁移。
- 安装（含 Pack）与卡片重试只要返回 `SkippedConflict`（不论首次还是已知冲突，也不论整批是否全为冲突）即打开报告弹窗，在独立「未导入」分组逐项列出 `source_file`、冲突工作流名与恢复方法，Pack 条目同时显示成员身份并指向该 canonical Plugin ID 的已安装卡片；「未导入」条目不参与 MCP / Skill 依赖判定；批次没有任何 `Imported` 且含冲突时提示不用完整成功样式。
- 升级遇到首次冲突时，按同一规则打开报告弹窗并列入「未导入」分组；每当报告弹窗因 `Imported`、`Failed` 或首次冲突打开时，同批次中的已知冲突也一并列入「未导入」分组并标为「仍未解决」。升级仅含已知冲突（或合并存活 `SkippedProcessed` 与在装期间已删除副本）时，不打开报告弹窗，保持中性一行计数摘要，不使用错误样式，也不显示为「此前已处理」。
- 状态转换以读到的前状态为条件提交：仍冲突时保持 `conflict_skipped` 并返回 `SkippedConflict`，不因重复插入撞主键而产生 `Failed`；名称检查后若并发写入先占用同名，回滚工作流创建并按 `SkippedConflict` 报告及提交/保留 `conflict_skipped`；并发操作先改写同一来源时按最新状态重判，确保同一来源键任一时刻至多关联一个存活副本。

### 禁止结果

- 将 `conflict_skipped` 永久当作 `SkippedProcessed` 跳过，导致改名或删除冲突对象后仍无法通过卡片重试、升级或重装导入。
- 覆盖、改名或删除既有的同名工作流。
- 对自动升级的已知冲突每次弹出错误弹窗，或将升级已知冲突显示为「此前已处理」；在报告弹窗因其他项打开时漏列同批次的已知冲突，或把「未导入」条目计入 MCP / Skill 依赖缺口。
- 因纯 `INSERT` 主键冲突或并发同名竞争把同名冲突报告为 `Failed`，或给同一来源键挂出两份存活副本。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| #1399 四步回归与三条恢复入口：首次冲突、改名（含 `COLLATE NOCASE`）后导入、改回原名再冲突、删除后通过卡片重试/升级/重装导入 | Desktop 实现待补测试（同时改写旧 `marketplace_install_reports_a_name_conflict_once`） | `Missing` |
| 0.6.0 遗留 `conflict_skipped` 直接参与重判，且当前阻挡对象可不同于原冲突工作流；已知冲突标记只按前状态计算、与操作类型无关 | Desktop 实现待补测试 | `Missing` |
| 安装（含 Pack）、卡片重试与升级首次冲突打开「未导入」报告弹窗（含 `source_file`、工作流名、恢复指引、Pack 成员卡片指向与非成功样式）；「未导入」不进依赖判定；弹窗打开时同批次已知冲突标为「仍未解决」 | Desktop 实现待补测试 | `Missing` |
| 升级仅含已知冲突时不弹窗，返回带工作流名的已知 `SkippedConflict` 并以中性一行计数呈现，不用错误样式也不显示「此前已处理」 | Desktop 实现待补测试 | `Missing` |
| 条件状态转换与并发安全：仍冲突不撞主键变 `Failed`；并发同名回滚创建并报告为 `SkippedConflict`；并发来源改写按最新状态重判，同一来源键至多一个存活副本 | Desktop 实现待补测试 | `Missing` |

### 决策依据

- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D1–D3 与不变量 1–5。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md) D1–D4。

## Reinstalling A Workflow Plugin Restores Deleted Copies Without Duplicating Surviving Ones

### 风险

如果重装恢复已删除副本依赖卸载时清理来源记录，那么卸载后才删除的副本、0.6.0 上已经卸载的旧插件以及清理失败或与导入交错的情况都无法恢复；如果恢复时不小心重导存活副本，又会给用户改名或编辑过的工作流制造重复副本；如果恢复已删除副本遇到非同名失败后仍保留已删除 `imported` 记录，用户在卡片上刚看到 `Failed` 再点重试却会被「卸载后重装可恢复」误导且新版本升级也不会导入。

### 前置状态

- 工作流库与来源表存在来自同一市场插件（或 Pack 成员）的两类记录：① `imported` 且关联副本仍存活（`workflows` 中 `id = workflow_id` 且 `is_deleted = 0`，可能已被用户改名/编辑）；② `imported` 但关联副本已删除（不存在或 `is_deleted = 1`，包括在装期间删除、卸载后才删除，以及 0.6.0 上已卸载插件遗留的记录）。
- 重装包内待恢复文档可能遇到库中同名存活工作流（`COLLATE NOCASE`），或遇到非同名导入失败（如无效图或注入失败，及条件删除自身失败的数据库只读兜底）。
- 存在 Pack 安装，其中包含本次实际安装成功的 workflow 成员、已经在装或安装失败的成员。

### 触发

1. 在装期间删除副本后执行自动升级或点击已安装卡片「重试导入未导入的文档」。
2. 卸载（或不经过卸载清理、含卸载后才删除副本和 0.6.0 遗留记录）后成功市场 `install`（含 Pack）重装同一 canonical Plugin ID，包括恢复时遇同名冲突、恢复时遇非同名失败以及修复后再次卡片重试/升级。

### 必须成立

- 市场 `install` 成功开启新的**安装周期**；对于 Pack，**仅** `InstallOutcome::PackInstalled.members` 中本次实际安装成功的 workflow 成员开启新周期，已经在装、安装失败或未安装的成员保持原周期、不触发已删除副本恢复。
- 重装时不依赖卸载时清理，直接按当前副本存活状态重判：
  - `imported` 且副本存活：保持原工作流与来源记录不变，返回 `SkippedProcessed`（即使已被用户改名或编辑），不创建重复副本，且「来源插件已卸载」标记随重装消失。
  - `imported` 且副本已删除：按未交付路径重新尝试导入。成功时在同一事务里以读到的前状态为条件将原记录改写为指向新副本的新 `imported`；遇到同名存活工作流（`COLLATE NOCASE`）时，登记为本周期的 `conflict_skipped`，按安装规则打开「未导入」报告弹窗，之后可从卡片重试或升级恢复。
- 重装恢复已删除副本时遇到非同名失败：在同一次操作里以读到的前状态为条件删除这行 `imported`（副本已删除）记录，使该文档回到「无记录」，之后的卡片重试和升级均按无记录处理并可重试恢复；仅当该条件删除本身失败时（例如失败就来自数据库不可写），记录才保留为已删除副本的 `imported`，退回「修复后再次卸载重装」兜底。
- 在当前安装周期内，自动升级和卡片重试继续尊重用户删除，不自动复活已删除副本：升级返回与存活副本 `SkippedProcessed` 可区分的结果，不显示为「此前已处理」，仅计入中性摘要而不改变成功样式；卡片重试同样可与存活副本区分，并提示「副本已删除，卸载后重装可恢复」。

### 禁止结果

- 在当前安装周期内的自动升级或卡片重试中复活用户已删除的副本，或把升级已删除副本显示为「此前已处理」及非成功样式。
- 重装时为仍存活（含已改名/编辑）的副本新建第二份重复副本，或因未经过卸载清理而导致卸载后才删除的副本、0.6.0 已卸载遗留记录无法在成功重装时恢复。
- 对 Pack 中已经在装、安装失败或未安装的成员开启新安装周期并恢复其已删除副本。
- 重装恢复遇同名时不登记本周期 `conflict_skipped`；或遇非同名失败且条件删除可执行时，仍把 `imported`（副本已删除）留在库里挡住本周期的卡片重试和升级。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 成功重装恢复在装期间删除、卸载后才删除以及 0.6.0 遗留 `imported` 的已删除副本，同时保留存活（含改名/编辑）副本及其来源且不建重复副本 | Desktop 实现待补测试（同时改写旧 `reinstalling_the_package_does_not_import_the_documents_again` 与 `a_record_survives_the_workflow_delete_and_a_later_reinstall`） | `Missing` |
| Pack 安装仅对 `InstallOutcome::PackInstalled.members` 中实际安装的成员开启新周期，已在装或失败成员的已删除副本不被恢复 | Desktop 实现待补测试 | `Missing` |
| 重装恢复已删除副本遇同名（`COLLATE NOCASE`）时记录本周期 `conflict_skipped`、打开「未导入」报告，消除冲突后可经卡片重试或升级导入 | Desktop 实现待补测试 | `Missing` |
| 重装恢复已删除副本遇非同名失败时条件删除旧 `imported` 记录回到「无记录」，随后卡片重试和升级可直接重试；条件删除自身失败时退回「修复后卸载重装」兜底 | Desktop 实现待补测试 | `Missing` |
| 在装期间删除副本在升级里与存活 `SkippedProcessed` 可区分、不显示为「此前已处理」且保持中性摘要不改成功样式；卡片重试提示「副本已删除，卸载后重装可恢复」 | Desktop 实现待补测试（同时核对/改写旧 `a_deleted_workflow_is_not_resurrected_by_an_upgrade_or_retry`） | `Missing` |

### 决策依据

- [市场工作流冲突恢复后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261124-marketplace-workflow-conflict-recovery.md)
  D1–D3 与不变量 1–5。
- [市场自动导入后续决策](../../../../../decisions/desktop/plugin/category/workflow/20261109-marketplace-workflow-import.md) D2、D4。
