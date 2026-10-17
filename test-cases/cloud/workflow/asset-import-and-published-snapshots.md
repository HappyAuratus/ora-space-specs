# 资产导入与已发布快照：导入即发布、门禁先于落库、快照不可变

依据[资产导入与已发布快照决策](../../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)
（`approved`），以及它继承的[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)。
资产导入本身尚未实现（落地阶段 1 是 ora-space/circus#73），因此导入侧义务今天都是 `Missing`；
能引用直接证据的是 Cloud 既有的发布与快照规则，它们证明导入将要复用的那部分契约成立。

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
| 导入成功必然留下 workflow 与已发布快照 | 真实 HTTP + PostgreSQL：导入后两份行都存在，运行能按该 `snapshot_id` 创建 | Missing |
| 版本由 Cloud 分配，文档版本与文件名不参与 | 真实 PostgreSQL：自述版本冲突的文档导入后仍是下一个整数 | Missing |
| 快照 append-only，发布与编辑不改写既有快照 | 真实 PostgreSQL：连续发布得到递增版本，历史版本内容逐字节不变，活稿编辑不影响它们 | Covered |
| 运行按 `snapshot_id` 读取，不读活稿 | 真实 HTTP + PostgreSQL：改了活稿后运行仍执行所选版本 | Partial |

- 决策依据：D1（一份文档 = 一份 workflow 加一个已发布快照、快照版本由 Cloud 分配）、D2（快照
  append-only）、不变量 1、2。
- 直接证据：**Covered（第三行）、Partial（第四行）** — [`integration/workflow_snapshot_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_snapshot_test.go)
  的 `TestWorkflowSnapshotPublishServices` 断言首发布为 `v1`、再发布为 `v2`/`v3`，每版内容不被后续
  发布改写，发布不提升活稿版本，归档后历史被隐藏；版本分配在
  [`internal/core/workflow.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow.go)
  里是 `COALESCE(MAX(version),0)+1`，迁移
  [`0028_workflow_snapshots.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0028_workflow_snapshots.sql)
  用 `(tenant_id, workflow_id, version)` 唯一索引兜住并发撞号。第四行：`TestWorkflowRunValidatesAgainstTheVersionItRuns`
  与 `TestWorkflowRunSnapshotIsScopedToItsWorkflow` 证明运行按所选版本校验与执行；`TestWorkflowSnapshotRestore`
  证明恢复是「把快照写回活稿」，不是改动快照。
- 仍缺：导入路径不存在，前两行 `Missing`。第四行的 `Partial` 指运行侧规则已有直接证据，但没有一份
  用例针对「导入产生的快照」执行同样的断言。

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
| 导入调用同一门禁，失败不留下 workflow 或快照 | 真实 HTTP + PostgreSQL：导入不可执行文档后两表都无对应行 | Missing |
| 失败逐文档，不影响同包其他文档 | 真实 HTTP + PostgreSQL：混入不可执行文档的包仍有其他文档导入成功 | Missing |

- 决策依据：D1（门禁的连带后果与包制作约束、#76 只能在首期子集内制作）、D4（同一份图 schema、
  没有豁免）；[插件资产决策](../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)
  D4（`document_not_executable_graph` 覆盖门禁全部拒绝并带问题列表）。
- 直接证据：**Covered（第一行）** — [`integration/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_admission_test.go)
  的 `TestWorkflowPublishValidatesTheGraphAndSavingDoesNot`（`tool` 节点被拒、问题点名 id 与 kind、
  `workflow_snapshots` 计数为 0）与 `TestWorkflowPublishRefusesUnusedAgentConfiguration`（`retry` /
  `skills` 被逐项点名，改回首期配置后发布成功），加上
  [`internal/core/workflow_graph_test.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph_test.go)
  的 `TestValidateWorkflowGraph` 与 `TestRequireValidWorkflowGraphIsOneStableFault`。这些证明门禁本身
  成立，不证明导入接线。
- 仍缺：导入调用门禁、错误码映射与「失败不落库」都随导入步骤才有对象，后两行 `Missing`。第一行
  的 `Covered` 只覆盖编辑器发布这条路径；导入复用同一函数是决策，不是今天的测试事实。

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
| 升级不触碰已成功导入的路径 | 真实 HTTP + PostgreSQL：升级后副本的图与快照逐字节不变 | Missing |
| 卸载不删除副本、不改写快照 | 真实 PostgreSQL：卸载后 workflow 与快照仍在，仍可发布、仍可运行 | Missing |
| 已冻结运行与副本后续变化无关 | 真实 PostgreSQL：编辑或归档副本后，既有运行仍读到冻结的图 | Partial |
| `workflow_snapshots` 只有 INSERT | 仓库级检查：不存在更新快照的 SQL 或写入路径 | Partial |

- 决策依据：D2（资产生命周期不改写快照）、D3（来源不进入运行契约）、不变量 2、3。
- 直接证据：**Partial（第三、四行）** — `TestWorkflowSnapshotPublishServices` 断言历史版本不被后续
  发布改写，[`0028_workflow_snapshots.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0028_workflow_snapshots.sql)
  的表上没有 `updated_at`，`publishWorkflow` 只 INSERT；运行侧由
  `TestWorkflowRunValidatesAgainstTheVersionItRuns` 证明执行的是所选快照。没有测试断言「归档副本后
  既有运行仍可读图」，也没有对「仓库里不存在 UPDATE 快照路径」的自动检查，所以两行都是 `Partial`。
- 仍缺：升级与卸载两条义务完全依赖尚不存在的导入步骤，记 `Missing`。

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
| 导入不写作者/发布者，表结构也没有该列 | 真实 PostgreSQL：导入后两张表的列与行都不含作者信息 | Missing |
| 触发者只记在来源记录 | 真实 PostgreSQL：来源记录有触发 space 与时间，workflow 行没有 | Missing |
| 作者不是权限、可见性或运行的输入 | 真实 HTTP：不同管理员与触发者停用后，副本的读取、目录投影与运行结果一致 | Missing |

- 决策依据：D1（导入的 workflow 与快照不记创建者或发布者）、不变量 7；[个人槽位映射决策](../../../decisions/cloud/model-access/20261015-executor-slot-mappings-and-per-slot-run-bindings.md)
  确立的运行身份范围。
- 仍缺：三条都 `Missing`。表结构今天确实没有作者列（[`0027_workflows.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0027_workflows.sql)、
  [`0028_workflow_snapshots.sql`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/migrations/0028_workflow_snapshots.sql)），
  但那是「还没有实现导入」，不是导入遵守了这条约束的测试证据；`@` 目录投影与运行身份里也没有任何
  作者相关的直接用例。
