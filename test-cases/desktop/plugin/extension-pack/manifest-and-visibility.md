# Extension Pack：清单、索引与可见性核心用例

对应决策：[插件集合包决策](../../../../decisions/desktop/plugin/extension-pack/0-extension-pack.md)
的 D1（pack 清单）、D3（可见性过滤）、D8（旧宿主行为投影到本宿主的确定性）与不变量
1、2、9。

前置状态：宿主已配置至少一个 marketplace source 并完成一次同步。

## Pack listings are indexed as display entries without a release of their own

- **风险**：pack 若被当作普通插件清单处理，会因缺少发布资产被静默丢弃，或反而被索引成
  可直接安装的条目，使 D2 编排层在错误的入口上建立。
- **触发**：向 registry 提交一份 `kind = "pack"`、携带 `[[pack.members]]` 且不声明
  `url`/`sha256`/`[[targets]]` 的清单，执行同步与索引重建。
- **必须成立**：
  - pack 条目进入派生索引，`kind` 为 `pack`，携带标题、描述与 logo/README 约定；
  - 其 `release_targets` 为 `None`（无可下载 release），宿主兼容性投影为不可直接安装；
  - 成员以裸 `identifier` 加可选 `agents` 引用编译成功，保持声明顺序。
- **禁止结果**：pack 因"无发布资产"被跳过、被记为坏文件，或被投影为可直接安装条目。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | pack 清单解析并携带按声明顺序的成员（含裸 identifier 与两种 `agents` 拼写） | `crates/plugin-manifest/src/pack.rs` `pack::tests::compiles_members_in_declaration_order_with_both_agent_spellings` | Covered |
  | pack 进入索引、kind 为 `pack`、`release_targets` 为 `None` | `crates/plugin-registry/src/index.rs` `index::tests::indexes_a_pack_listing_without_release_targets` | Covered |
  | pack 在 marketplace 发现列表中可见（经生产 `list_available`） | `e2e/tests/plugin_marketplace.rs` `marketplace_plugin_full_lifecycle_walkthrough` | Covered |
- **决策依据**：D1（pack 是 orchestration 条目，不携带发布资产）、不变量 1、2。

## Hidden listings stay out of discovery but remain resolvable by id

- **风险**：`marketplace_visible = false` 若在投影或前端层过滤，隐藏条目可能经任何派生路径
  泄漏回发现列表；若过滤错误地作用于 resolve，pack 安装将无法找到成员，迫使 installer 增加
  旁路。
- **触发**：向 registry 提交一份 `marketplace_visible = false` 的清单（携带真实可安装
  release），执行同步与索引重建，随后按其完整 id 解析清单。
- **必须成立**：
  - 过滤发生在索引构建期、生成索引条目之前，隐藏条目不出现在派生索引与任何发现列表中；
  - 同一条目按 `namespace/identifier` 仍可从其所属源检出中 resolve 出完整清单，
    且清单自述 `marketplace_visible == false`、release 元数据完整；
  - 任意 kind（非 pack）同样可携带该字段。
- **禁止结果**：隐藏条目出现在索引文件、发现列表或任何投影结果中；resolve 按 id 找不到
  隐藏条目；过滤产生 skipped 告警（隐藏不是坏文件）。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 索引构建期过滤，隐藏条目不进索引 | `crates/plugin-registry/src/index.rs` `index::tests::excludes_hidden_listings_from_the_index_but_keeps_them_resolvable` | Covered |
  | 隐藏条目按 id 可 resolve，清单自述不可见且 release 完整 | 同上 | Covered |
  | 隐藏条目经生产 `list_available` 不可见（真实 Git 源同步后） | `e2e/tests/plugin_marketplace.rs` `marketplace_plugin_full_lifecycle_walkthrough` | Covered |
- **决策依据**：D3（可见性只影响 discovery，不影响 addressability）、不变量 9。

## Pack visibility semantics survive restart and rebuild

- **风险**：可见性与成员声明若是运行期易变状态（内存过滤、按次构建的旁路），重启或重建后
  行为漂移，pack 成员在升级路径上时隐时现。
- **触发**：完成一次同步后重启宿主（同一数据目录），不执行任何重新同步，直接读取发现列表。
- **必须成立**：
  - 持久化的源配置与派生索引直接给出与重启前一致的语义：pack 可见、隐藏成员不可见；
  - 索引 schema 不因可见性新增字段，缓存版本不变。
- **禁止结果**：重启后隐藏条目重新可见，或 pack 从发现列表消失，或缓存因可见性被作废。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 重启后语义一致（真实 Backend 重开同一数据目录） | `e2e/tests/plugin_marketplace.rs` `marketplace_plugin_full_lifecycle_walkthrough` | Covered |
  | 索引 schema 与缓存版本不变 | `crates/plugin-registry/src/index.rs` `INDEX_VERSION` 未变更，既有 `load_round_trips_written_index` 等 30 项既有测试全部通过 | Covered |
- **决策依据**：D3（"索引 schema 不因此新增字段，缓存版本不变"）。

## Pack manifests reject illegal combinations at parse time

- **风险**：pack 与普通插件的结构边界若靠组合推断（"恰好没写 url 加恰好写了 members"），
  半个两种 kind 的清单会流入索引与编排层，错误在安装期才暴露。
- **触发**：解析下列清单：pack 携带 `url`/`sha256`/`[[targets]]`；非 pack 携带
  `[[pack.members]]`；`kind = "pack"` 出现在 installed 形态；`marketplace_visible` 出现在
  installed 形态；成员 identifier 或 `agents` 引用拼写非法；pack 缺少 `pack` 表。
- **必须成立**：
  - 每种组合在解析期即失败，错误携带结构化字段路径（数组含下标）；
  - `marketplace_visible` 缺省 `true`，且任意 release kind 均可携带；
  - installed 形态硬拒绝 pack（pack 永远不会成为已安装包）。
- **禁止结果**：任一非法组合解析成功；错误只有无定位的文本；pack 在 installed 形态被静默
  当作普通 kind 接受。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | pack 禁携带 release（universal 与 targets 两种声明） | `crates/plugin-manifest/src/pack.rs` `pack::tests::rejects_a_release_declaration_on_a_pack_listing` | Covered |
  | 非 pack 禁携带 `[[pack.members]]`；pack 缺表即失败 | `crates/plugin-manifest/src/pack.rs` `rejects_pack_members_on_a_non_pack_listing`、`requires_the_pack_table_for_pack_kind`、`rejects_a_pack_listing_without_the_pack_table` | Covered |
  | installed 形态拒绝 pack 与 `marketplace_visible` | `crates/plugin-manifest/src/pack.rs` `rejects_a_pack_manifest_in_the_installed_form`、`rejects_marketplace_visible_on_an_installed_manifest` | Covered |
  | 成员 identifier 与 `agents` 引用拼写校验 | `crates/plugin-manifest/src/pack.rs` `rejects_a_malformed_member_identifier`、`rejects_an_agent_reference_that_is_neither_bare_nor_canonical` | Covered |
  | 解析不产生 I/O（纯函数，未知字段忽略不受影响） | `crates/plugin-manifest/src/tests.rs` `ignores_unknown_manifest_fields_in_both_forms` | Covered |
- **决策依据**：D1（`[[pack.members]]` 必需且独占）、D3（installed 形态报错）、D4（引用
  拼写在解析期校验）、清单根决策"未知字段忽略"的保护。
