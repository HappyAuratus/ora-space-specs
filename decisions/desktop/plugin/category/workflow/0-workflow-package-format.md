---
status: implemented
---

# Workflow 插件：以静态包投递工作流文档，导入为可编辑的用户数据

`kind = "workflow"` 的插件是一个不启动进程的静态插件包。插件用 `orax.toml` 声明
`kind = "workflow"`，并在固定的 `assets/workflows/` 布局中携带一份或多份 `*.json` 工作流文档。
宿主在插件发现阶段只证明包内有形状正确的文档树（有哪些文件、共几份），在用户触发导入时逐份解析并
导入工作流库：一份文档对应一份工作流与一个已发布快照。导入产物是**用户数据**——可编辑、可改名、
可删除，比投递它的包活得久；显式卸载插件从不移除它。

该决策由 `ora-plugin-manifest`、`ora-plugin-manager` 与 `crates/backend` 的 workflow 导入用例
落地（提交 `a9c6742e`）。本文固定 workflow 包的布局、发现与导入语义；Cloud 侧按同一份格式导入
资产（由 [cloud/plugin-marketplace 后续决策](../../../../cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)
拥有），它自己的工作流运行契约由 [cloud/workflow 根决策](../../../../cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)
及其[后续决策](../../../../cloud/workflow/20261016-asset-import-and-published-snapshots.md)拥有。

## 术语与边界

- **Workflow 插件**是 `orax.toml` 中 `kind = "workflow"` 的 `.orax` 插件包；它的身份是完整
  canonical Plugin ID `<namespace>/<identifier>`，版本由清单声明。
- **Workflow 文档**是 `assets/workflows/` 直接子层的一份 `*.json` 文件。
- **导入副本**是导入后落进工作流库的那份工作流（含它的首个已发布快照）。副本与包之间没有所有权
  关系：包只提供内容和它的来源身份。
- 编辑器自己的导出/导入（`.reactflow.json` 文件、目录导入）不属于本决策；这里只规定插件包布局与
  由插件触发的导入。

## 决策

### D1：`kind = "workflow"` 是无进程、无专属 section 的 kind

release 与 installed 两种 `orax.toml` schema 都接受 `kind = "workflow"`（installed 形态可省略
`resolver`，按当前 resolver 1 解析）；身份、版本、描述等通用字段仍按[插件清单规则](../../manifest/0-versioned-dual-form-manifest.md)
校验。

Workflow kind 拒绝其他 kind 的专属 section：`[workbench]` 与 `[webview]` 都报
`NotAllowedForKind`；也不允许 `[[targets]]` 与 `[artifact]`（`PluginKind::may_ship_targeted_artifact()`
对 workflow 为假）。原因是工作流文档是平台无关的文本，包内没有任何需要按主机判断兼容性的二进制，
所以 workflow 包永远只有 universal 发布形态，消费方不需要做 target 选择。

至此 resolver 1 的 kind 集合是八种：`workbench`、`agent`、`webview`、`skill`、`mcp`、`hook`、
`pack`（由 extension-pack 决策加入）、`workflow`（本文）。[清单根决策](../../manifest/0-versioned-dual-form-manifest.md)
D4 与 [registry 根决策](../../registry/0-registry-distribution.md)正文的枚举仍只列前六种，以本文与
extension-pack 决策的修改项为准。

### D2：包内容固定为 `assets/workflows/` 下的浅层 `*.json`

```text
<plugin-package>/
├── orax.toml
└── assets/
    └── workflows/
        ├── <document-a>.json
        └── <document-b>.json
```

规则如下：

1. `assets/workflows/` 必须存在且是目录。
2. 只取该目录**直接子层**的普通 `*.json` 文件；扩展名比较忽略 ASCII 大小写。
3. `README`、其他扩展名的文件、子目录与符号链接被**忽略**而不是拒绝；符号链接在读取类型时不
   跟随，因此既不能被导入，也不能绕过包根。
4. 份数必须在 1 到 256（`MAX_WORKFLOWS_PER_PACKAGE`）之间：零份文档的包与超过上限的包都按
   `workflow` 字段报错。
5. 文档按文件名稳定排序后汇报，保证同一个 Plugin ID 与版本始终对应确定的文档集合。

第 3 条与第 4 条的取舍不同：契约是「这个目录里的每份文档都会被导入」，一个多余文件不是拒绝一个
本来有效的包的理由；但一份文档都没有的包没有任何投递意义。

### D3：发现只证明文档树，解析发生在导入一份文档时

插件发现阶段只枚举文档路径并返回描述符，不读取内容。逐份解析发生在导入时，因此一份无法解析的
文档只让**它自己**失败，而不是让同包其他可用工作流一起消失。

把解析提前到发现阶段会把所有插件共用的短路径变成读取用户文档的 I/O 与解析热路径，并且一份坏
文档会连带影响整个包的发现结果——这与「文档彼此独立的用户资产」这一性质相反。

### D4：一份文档 = 一份工作流 + 一个已发布快照，逐文档独立

导入逐份进行，返回与输入同序的逐文档结果：成功给出 `workflow_id` 与发布版本，失败给出该文档的
原因（无效 JSON、未声明名称、图不可执行、名称冲突，以及创建或发布环节自身的失败）。一份文档
失败不中断其余文档。

导入复用编辑器同一套创建与发布处理器，而不是另写一套写入路径，因此：

- **名称冲突沿用工作流库的既有规则**：不改名、不覆盖，这份文档失败。
- **图必须可执行**：解析与图校验由引擎负责，一份永远跑不起来的文档在这里就被拒绝，而不是存下来
  等到运行才发现。
- **发布版本优先级**：文档自述的 `version` → 包内文件名的词干（去掉 `.reactflow.json` 或 `.json`，
  忽略大小写、最长后缀优先）→ 工作流标题（仅当词干为空，例如文件名为 `.json`）。每个候选都必须是
  可寻址的版本：非空、≤128 字节、不含控制字符与路径分隔符、不是 `.`/`..`，也不是保留的 `draft`。
  不可寻址的候选**不被采用，也不被清洗**——静默改写会以作者没写过的版本号发布。三个来源都不可用时
  这份文档仍然导入：发布处理器为它生成自动版本（`is_automatic_version`）。因此「版本不可寻址」不是
  一种文档失败原因，编辑器写下的草稿标记 `draft` 也只被跳过。
- **存储的图是文档原文**：编辑器写在图旁边的元数据不需要投影出去，导出文件因此可以原样往返。

### D5：导入副本是用户数据，生命周期与包解耦

> 历史决策：D5 的市场安装与升级触发范围、来源幂等，以及 D4 的市场同名冲突结果已由
> [市场导入、依赖报告与卸载提示](20261109-marketplace-workflow-import.md)修改。
> 市场触发与卸载呈现以该后续决策为准；本地导入、包格式与本决策其余已实现规则保持不变。

- 副本可编辑、可改名、可归档；没有任何路径把它当作插件拥有的只读对象（与
  [插件 Skill 的只读投影](../skill/0-static-skill-package.md)相反，工作流文档不是 Catalog 投影）。
- 显式卸载插件不移除、不改写导入的工作流。文档只在**本地 `.orax` 导入**这一条路径上被读取并导入；
  市场安装与升级不读取文档，因此升级不会覆盖用户副本（Cloud 侧另有「升级只增不改」的导入规则，
  由 [cloud/plugin-marketplace 后续决策](../../../../cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)拥有）。
- 包安装失败不会创建任何工作流，一份文档导入失败也不会移除投递它的包：安装结局与导入结局分开记录
  （`ImportedPlugin.outcome` 与逐文档结果各自返回）。
- 副本能否运行与它的来源无关：它和编辑器发布的工作流共享同一份图 schema 与运行契约。

### D6：插件不以插件身份向工作流库贡献行

`workflows` 表的 namespace 恒为 `local`，插件不向它贡献任何行（[registry 决策](../../registry/20260831-source-derived-namespace.md)）。
导入副本正是这条不变量的体现：它的身份是用户数据，插件身份只出现在来源记录里，不进入工作流的
身份，也不构成所有权。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 在 `orax.toml` 里用 `[[workflows]]` 声明每份文档 | 产生清单与文件树两份贡献清单，必须再定义缺失、重名与路径漂移的优先级；固定目录已经完整表达集合 |
| 递归扫描包内任意 `*.json` | `references` 或 vendored 内容会被误当作用户资产被导入，作者也无法从浅层布局判断包的公开内容 |
| 一份文档非法就拒绝整包 | Skill 的整包校验成立是因为贡献集合是插件身份的一部分；工作流文档是彼此独立的用户资产，用用户数据为包的完整性买单不划算，包完整性已由 sha256 承担 |
| 发现阶段就读取并解析文档内容 | 把共用短路径变成读取用户文档的热路径，并让一份坏文档影响整个包的发现结果 |
| 导入时自动改名避开名称冲突 | 用户看不到冲突，且同一个包在不同环境导入出不同名字，来源无法复核 |
| 让导入副本随包升级被覆盖 | 用户对副本的编辑会被静默丢弃，是数据损失；也违背「导入副本比较包的寿命更长」 |
| 把导入的工作流标记为插件拥有（只读） | 与「用户数据」语义冲突，并让工作流库出现插件身份的行，破坏 D6 |

## 不变量

1. `kind = "workflow"` 的包不启动进程、无专属 section、无 target 形态。
2. 一个包贡献的文档集合由 `assets/workflows/` 直接子层的 `*.json` 文件名集合确定；被忽略的条目
   不参与，也不影响结果。
3. 发现阶段不解析文档内容；解析在导入一份文档时发生。
4. 导入逐文档独立：一份文档失败不改变同批其余文档的结果。
5. 导入的副本是用户数据；卸载插件或升级包都不删除、不改写它。
6. 插件不以插件身份向工作流库贡献行，副本的身份与命名空间与插件无关。

## 当前未解决的问题

- 编辑器自己的导出与目录导入协议不属于 workflow 插件语义，本文不统一两者的失败与命名规则。
- 来源记录、幂等与对账展示由 Cloud 侧决策拥有；本文只固定包内格式与宿主侧的导入语义。
- pack 是否能携带 workflow 文档、以及成员插件如何被选中，属于 pack 自己的领域决策。
