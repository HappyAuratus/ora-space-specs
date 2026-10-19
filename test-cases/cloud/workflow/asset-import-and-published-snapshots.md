# 资产导入与已发布快照：导入即发布、门禁先于落库、快照不可变

依据[资产导入与已发布快照决策](../../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)
（`implemented`），以及它继承的[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)。
导入已随 [cloud#77](https://github.com/ora-space/cloud/pull/77) 落地：逐文档导入在同一个
事务里创建 workflow 与首个已发布快照，走与编辑器发布相同的 `validateWorkflowGraph`，失败文档不
留下任一行。下表把导入侧的直接证据与既有发布/运行规则分开标注。

## An Imported Document Leaves A Published Snapshot Runs Can Freeze

- 风险：导入只建 workflow 不发布，系统里出现「能看到、点不动」的资产；或导入把文档自述版本写进
  快照，导致版本不可寻址、并发导入撞号。
- 前置状态：包内文档是合法 JSON 且有名称；租户里没有同名 workflow；导入步骤在事务外解析、事务内
  落库。
- 触发：导入一份文档；同一 workflow 连续导入或发布多次；文档自述 `version` 或包内文件名与 Cloud
  分配结果不一致；导入后直接运行该 workflow。
- 必须成立：一次成功导入产出一个 workflow 加一个**已发布**快照；快照版本是 Cloud 分配的
  `MAX(version)+1`（租户内按 workflow 唯一），文档自述版本与文件名不参与；发布门禁先于落库，失败
  不留下 workflow 或快照；导入产物与编辑器发布的工作流在运行契约上等价，运行只认 `snapshot_id`；
  后续编辑活稿不改写既有快照。
- 禁止结果：导入后没有快照、按文件名词干或文档版本派生版本号、并发导入得到同号快照、门禁失败仍
  留下 workflow 行、运行读取活稿而不是快照。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 导入成功必然留下 workflow 与已发布快照 | 真实 HTTP + PostgreSQL：导入后两份行都存在，运行能按该 `snapshot_id` 创建 | Partial |
| 版本由 Cloud 分配，文档版本与文件名不参与 | 真实 PostgreSQL：自述版本冲突的文档导入后仍是下一个整数 | Covered |
| 快照 append-only，发布与编辑不改写既有快照 | 真实 PostgreSQL：连续发布得到递增版本，历史版本内容逐字节不变，活稿编辑不影响它们 | Covered |
| 运行按 `snapshot_id` 读取，不读活稿 | 真实 HTTP + PostgreSQL：改了活稿后运行仍执行所选版本 | Partial |

- 决策依据：D1（一份文档 = 一份 workflow 加一个已发布快照、快照版本由 Cloud 分配）、D2（快照
  append-only）、不变量 1、2。
- 直接证据：**Covered（第二、三行）、Partial（第一、四行）** — cloud#77 的
  `TestWorkflowAssetImportFromInstalledPlugin` 导入后同时断言文档行里的 `workflowId`/`snapshotVersion`
  与 `GET .../workflows/{id}/snapshots` 里那一条已发布快照；`release.json` 自述 `"version":7`，快照
  仍由 Cloud 分配为 `v1`，升级到 `v1.1.0` 也不给已导入路径追加版本。快照的 append-only 与版本分配
  仍由 [`integration/workflow_snapshot_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_snapshot_test.go)
  的 `TestWorkflowSnapshotPublishServices` 与迁移
  [`0028_workflow_snapshots.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0028_workflow_snapshots.sql)
  的 `(tenant_id, workflow_id, version)` 唯一索引证明；运行侧由
  `TestWorkflowRunValidatesAgainstTheVersionItRuns` 与 `TestWorkflowRunSnapshotIsScopedToItsWorkflow` 证明。
- Partial 的原因：第一行还没有「用导入产生的 `snapshot_id` 创建一次运行」的用例；第四行的运行侧规则
  已有直接证据，但没有针对导入快照重复同一断言。

## Import Refuses Graphs Cloud Cannot Execute

- 风险：包内文档只满足「能存进编辑器」而不满足发布门禁，导入后成为一份跑不起来的资产；#76 的
  官方示例包若含首期不执行的节点或未声明槽位，会在用户点运行时才暴露。
- 前置状态：首期只执行 `start` / `agent` / `condition` / `output`；agent 节点的 `skills`、`mcps`、
  `interactive`、`outputContract`、`retry`、`promptInactivity` 非 null 即拒绝；槽位必须在图里声明
  （`default` 除外）。
- 触发：导入含 `tool` 等首期不执行节点的文档；导入 agent 节点带上述任一字段的文档；导入引用未声明
  槽位的文档；同包其他文档合法。
- 必须成立：导入调用与编辑器发布同源的 `validateWorkflowGraph`；拒绝是逐文档的
  `document_not_executable_graph`，带门禁的问题列表；失败文档不产生 workflow 与快照；同包其他文档
  照常导入；首期可执行的文档正常发布。
- 禁止结果：导入绕过门禁先落库、把不可执行图降级成「导入成功但不可运行」、因一份门禁失败丢弃同包
  其他文档、错误原因不可定位到节点/槽位/字段。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 门禁按首期子集拒绝并点名节点、字段、槽位 | 真实 HTTP：各类不可执行图发布被拒，问题列表逐项可定位 | Covered |
| 导入调用同一门禁，失败不留下 workflow 或快照 | 真实 HTTP + PostgreSQL：导入不可执行文档后两表都无对应行 | Covered |
| 失败逐文档，不影响同包其他文档 | 真实 HTTP + PostgreSQL：混入不可执行文档的包仍有其他文档导入成功 | Covered |

- 决策依据：D1（门禁的连带后果与包制作约束、#76 只能在首期子集内制作）、D4（同一份图 schema、
  没有豁免）；[插件资产决策](../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)
  D4（`document_not_executable_graph` 覆盖门禁全部拒绝并带问题列表）。
- 直接证据：**Covered（三行）** — cloud#77 的 `parseWorkflowAssetDocument` 直接调用
  `validateWorkflowGraph`，`TestWorkflowAssetImportDocumentFailuresAreIndependent` 导入含
  `nodes`/`edges` 但没有 `start` 的文档后得到 `document_not_executable_graph` 与问题列表，并断言
  `workflows`/`workflow_snapshots` 里没有它的行；同包的 `ok.json` 照常导入并发布。门禁本身的节点、
  字段、槽位矩阵仍由 [`integration/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_admission_test.go)
  的 `TestWorkflowPublishValidatesTheGraphAndSavingDoesNot` 与
  `TestWorkflowPublishRefusesUnusedAgentConfiguration`，以及
  [`internal/core/workflow_graph_test.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph_test.go)
  的 `TestValidateWorkflowGraph` 覆盖。

## Provider Uninstall Or Upgrade Never Rewrites Imported Snapshots

- 风险：包升级或卸载改写用户副本与既有快照，已冻结或已开始的运行读到不同的图；或升级用新文档覆盖
  用户编辑过的副本。
- 前置状态：某文档已成功导入并产生快照；用户可能已编辑或归档副本；该 workflow 可能已有引用快照的
  运行；提供方插件可能被升级或卸载。
- 触发：升级插件到含同名文档的新版本；卸载提供方插件；删除或归档副本；读取已冻结运行的图。
- 必须成立：升级只导入从未成功导入过的路径，成功导入过的路径不改写副本与快照；卸载只表示「不再有
  来源提供更新」，不删除副本、不改写快照；已冻结运行按 `snapshot_id` 读取，与副本其后编辑、归档、
  来源删除无关；`workflow_snapshots` 没有 UPDATE 路径。
- 禁止结果：升级覆盖用户副本或追加改写既有快照、卸载时回收副本、运行因来源消失而失效、来源状态
  进入 `workflow_runs`。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 升级不触碰已成功导入的路径 | 真实 HTTP + PostgreSQL：升级后副本的图与快照逐字节不变 | Covered |
| 卸载不删除副本、不改写快照 | 真实 PostgreSQL：卸载后 workflow 与快照仍在，仍可发布、仍可运行 | Partial |
| 已冻结运行与副本后续变化无关 | 真实 PostgreSQL：编辑或归档副本后，既有运行仍读到冻结的图 | Partial |
| `workflow_snapshots` 只有 INSERT | 仓库级检查：不存在更新快照的 SQL 或写入路径 | Partial |

- 决策依据：D2（资产生命周期不改写快照）、D3（来源不进入运行契约）、不变量 2、3。
- 直接证据：**Covered（第一行）、Partial（第二至四行）** — cloud#77 的
  `TestWorkflowAssetImportFromInstalledPlugin` 让包从 `1.0.0` 升到 `1.1.0`：同一路径的 `workflowId`
  不变、该 workflow 的快照数仍为 1，升级前后分别读取活稿与 `v1` 快照的 `graph::text` 并逐字节
  比较，两者都没有变化，新版本只新增 `hotfix.json`；卸载插件后 live workflow 仍在，
  `workflow_snapshots` 计数与卸载前相同。更广的快照不可变证据是 `TestWorkflowSnapshotPublishServices`
  与 [`0028_workflow_snapshots.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0028_workflow_snapshots.sql)
  （表上没有 `updated_at`，发布只 INSERT）；运行侧由 `TestWorkflowRunValidatesAgainstTheVersionItRuns`
  证明执行的是所选快照。
- Partial 的原因：卸载后的测试只断言 workflow 行仍在、快照数不变，没有真的再发布或启动一次运行，
  所以「仍可发布、仍可运行」仍是缺口；没有测试断言「归档副本后既有运行仍可读图」，也没有对
  「仓库里不存在 UPDATE 快照路径」的自动检查。

## Imported Workflows Carry No Author Or Publisher

- 风险：导入把触发安装的管理员写成作者，或伪造一个系统身份；作者随后被当成权限、可见性或运行的
  输入，而副本比任何一位管理员活得久。
- 前置状态：`workflows` 与 `workflow_snapshots` 只有租户归属；不同管理员可能先后触发导入；触发者
  可能已停用或离开。
- 触发：管理员 A 触发导入；管理员 B 再次触发或升级；管理员 A 停用后读取、运行该副本；读取快照与
  运行身份。
- 必须成立：两张表不为导入新增作者/发布者列，也不写入伪造的系统身份；触发者信息只留在来源记录；
  作者不参与权限判定、`@` 目录可见性与运行身份；副本在触发者离开后照常可读、可编辑、可运行。
- 禁止结果：把管理员写成作者、伪造系统身份、按作者过滤可见性或运行、触发者停用后副本不可用。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 导入不写作者/发布者，表结构也没有该列 | 真实 PostgreSQL：导入后两张表的列与行都不含作者信息 | Covered |
| 触发者只记在来源记录 | 真实 PostgreSQL：来源记录有触发 space 与时间，workflow 行没有 | Covered |
| 作者不是权限、可见性或运行的输入 | 真实 HTTP：不同管理员与触发者停用后，副本的读取、目录投影与运行结果一致 | Partial |

- 决策依据：D1（导入的 workflow 与快照不记创建者或发布者）、不变量 7；[个人槽位映射决策](../../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)
  确立的运行身份范围。
- 直接证据：**Covered（第一、二行）、Partial（第三行）** — cloud#77 的
  `TestWorkflowAssetImportFromInstalledPlugin` 在导入后查询 `information_schema.columns`，断言
  `workflows`、`workflow_snapshots` 没有任何 author/creator/publisher 列，并确认来源记录行带有触发
  space、租户与 `attempted_at`。[`0027_workflows.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0027_workflows.sql)
  与 [`0028_workflow_snapshots.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0028_workflow_snapshots.sql)
  的列定义支持同一结论。
- Partial 的原因：表里没有作者列，导入也就无从把作者写进权限、可见性或运行输入；但没有一条用例
  让不同管理员先后触发后，再对 `@` 目录投影与运行结果做横向断言。
