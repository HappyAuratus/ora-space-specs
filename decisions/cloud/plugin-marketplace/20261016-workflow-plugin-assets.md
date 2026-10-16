---
status: proposed
---

# Workflow 插件资产由 Cloud 自行取包、逐文档导入为可编辑副本

`kind = "workflow"` 的插件包是「工作流文档的投递载体」，不是插件代码。Cloud 在**资产导入步骤**
里自己下载该插件版本的 `.orax`、校验 sha256、限定大小，把 `assets/workflows/` 下的每份 `*.json`
导入为该租户的一份普通工作流并发布首个快照。导入产物是**用户数据**（可编辑、可改名、可删除），
来源身份另存为一条独立记录，只服务于幂等、对账与展示，不构成所有权。

这明确扩展了根决策「Cloud 不下载包、Node 不自行解析市场」的边界：扩展**仅限资产导入**，安装落盘
仍只由 Node 完成，Cloud 不把包写进任何 Workspace，取到的字节用完即弃。四种证据——安装成功、接管
成功、发布可用、执行可用——是四个可以分别判断的事实，任何一条都不能推导出另一条。

本 ADR 处于 `proposed`：方案、边界与不变量都需要 reviewer 在实现之前纠偏。资产导入与运行契约的
接缝写在 [workflow 后续决策](../workflow/20261016-asset-import-and-published-snapshots.md)；包内容
格式由 [workflow 插件类别根决策](../../desktop/plugin/category/workflow/0-workflow-package-format.md)
拥有，本文不重复。本 ADR 不承担既有租户的回填迁移义务：只对生效后新触发的导入负责。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [根决策](0-plugin-marketplace.md) D1：`space_plugins` 选择状态与 `workspace_plugin_instances` 执行实例两级分离，聚合规则 | 继承，不变：导入结果不参与插件实例聚合，也不改变 `installed` 的含义（D5） |
| 根决策 D2：fan-out 到 live workspace，受 `one_project_operation` 串行 | 继承，不变：资产导入不 fan-out、不建 operation、不占项目独占（D3） |
| 根决策 D3：目录快照由 Cloud git 同步，读取永远不出网 | 继承，不变：导入只消费目录快照已记录的身份与 `url`/`sha256`，不新增出网点（D3） |
| 根决策 D4：`plugin_ensure` 载荷与 `installed=true` 证据 | 继承，不变；资产导入的证据不是插件安装的证据（D5） |
| 根决策不变量 2：外部副作用前权威状态与 operation 已落库；网络/下载绝不在数据库事务内 | 继承，并扩展：下载、校验与解析都在事务外，导入结果落库在事务内（D3） |
| 根决策不变量 4：sha256 校验必选 | 继承，不变：导入前先校验，且先于任何解析（D3） |
| [管理员选择决策](20260927-admin-selection-and-idle-maintenance.md) D1：只有当前管理员可以改变期望 | 继承，不变：导入由管理员改变期望触发，不新增入口（D6） |
| 管理员选择决策不变量 5：共享汇总不暴露受限日志，Node 结果才证明实例安装结局 | 继承，不变：导入结果与安装结局分别可见（D5） |
| [Node 执行安装决策](20260928-node-executes-plugin-installs.md) 标题与不变量 3：安装与移除由 Node 执行，实例状态只由 Node 执行结果写入 | 继承，不变：**安装落盘仍然只由 Node 完成**，本文其余部分不触碰它（D3） |
| Node 执行安装决策 D1：执行结果逐项、失败码有界 | 形式沿用（逐文档结果同样有界失败码），语义不同：那是安装落盘，这是资产导入（D4） |
| Node 执行安装决策不变量 1：载荷由 Cloud 从目录快照拼装，Node 不自行解析市场 | **修改**：Cloud 自己取包，但**仅用于资产导入**；仍然「Node 不自行解析市场」，且 Cloud 不把包交给 Node（D3） |
| [desktop plugin registry 决策](../../desktop/plugin/registry/20260831-source-derived-namespace.md)：`workflows`、`agents` 的 namespace 恒为 `local` 且插件不向它们贡献任何行 | 继承，不变：导入副本是 `local` 用户数据，插件身份只存在于来源记录中，不进 workflow 行的身份（D1） |
| 根决策 v1 边界：pack 禁装（目录可列出） | 继承，不变：pack 携带文档的选择语义属于 pack 自己的决策，不在本文（「本决策未解决的问题」） |

## 决策

### D1：资产模式 A —— 导入为可编辑副本，来源另存

每份文档导入为当前租户的一份普通 workflow（沿用编辑器同一套创建规则）与一个已发布快照；导入产物
与用户在编辑器里新建、发布的工作流在存储与运行契约上没有区别。

来源记录至少包含：

```text
source_identity = (source_namespace, identifier, package_version, package_sha256, document_path)
```

- `(source_namespace, identifier)` 是 canonical Plugin ID，`document_path` 是包内路径
  （`assets/workflows/<name>.json`）。canonical 身份用显式列对保存，不塞进 JSON。
- 记录还保存触发它的 space、导入结果与时间；用户对副本改名、编辑、归档都不改写这条记录。
- 来源不构成所有权：副本可编辑、可改名、可归档；任何地方都不能用「它来自某个插件」阻止用户改动，
  也不能因为插件卸载、来源删除或市场下架而回收副本。

选择模式 A 同时让 registry 决策继续成立：插件不以插件身份向 `workflows` 贡献行，导入副本的身份是
`local` 用户数据，来源只在独立记录里。这与 desktop 已实现的语义一致
（`ora-plugin-manifest::PluginKind::Workflow`：包是文档投递载体，宿主把文档按用户数据导入工作流库，
被导入的工作流比投递它的包活得久；卸载插件从不移除它）。

### D2：归属保持 tenant 级，空间只记入来源

- 不给 `workflows` 加 space 列，也不把 workflow 变成空间私有资源：它继续是租户级、编辑器与 `@`
  共用同一份。
- 「哪个空间的期望触发了这次导入」只是来源记录的一段事实，不决定可见性。
- 本决定依赖[一租户一空间](../tenancy/20260923-one-space-per-tenant.md)不变量。引入多空间时必须
  重新定义跨空间可见性：同一份副本不能同时是「空间 A 的模板」和「空间 B 不可见」，来源里的 space
  字段无法承担这个语义。
- 名称唯一性沿用现有 tenant 级规则（`workflows(tenant_id, name) WHERE deleted_at IS NULL`），
  因此跨空间同名文档按 D4 逐文档失败，而不是并存两份。

### D3：触发点与取包 —— Cloud 在资产导入步骤自行下载 `.orax`

**触发**：space 的期望状态出现某个 `kind = "workflow"` 插件时，由 Cloud 侧的资产导入步骤执行。
触发**不依赖 Node 安装结果**：没有 live workspace 的新空间也要拿到文档，不能出现「空间已聚合
`installed` 却没有任何模板」。这也意味着导入不与安装 fan-out 配对，不需要 `one_project_operation`
的独占。

**取包**：载荷与 effect 一样来自目录快照，不新增出网点：

- workflow kind 是平台无关的通用发布资产，永远只需 `universal{url, sha256}`。`workflow` 不允许
  `[[targets]]` / `[artifact]`（`may_ship_targeted_artifact()` 对它为假），因此导入不需要任何
  target 选择与主机兼容性判断。
- 边界：sha256 必选且先于任何解析；只接受 `assets/workflows/` 直接子层的 `*.json`（README、
  子目录、符号链接忽略），份数 1..=256，与 desktop 的 `MAX_WORKFLOWS_PER_PACKAGE` 一致。
- 大小上界：压缩包 ≤ 16 MiB、解压后总量 ≤ 64 MiB、归档条目数 ≤ 512。超限以
  `asset_package_too_large` / `asset_package_expansion_too_large` 拒绝整个包。包体只承载 JSON
  文本，上界要在「覆盖真实包」与「限制 Cloud 从远端取字节的风险面」之间取；解压上界才是真正的
  控制（防 zip bomb），压缩上界只是提前失败。这三个数值是可评审的取值，不是从既有常量推导的。
- 下载、校验、解压与解析全部在数据库事务外；只有逐文档结果在事务内落库（根决策不变量 2）。

**明确扩展与明确不扩展**：本文扩展「Cloud 不下载包」，仅限资产导入这一处。不扩展的是「安装落盘
只由 Node 完成」与「Node 不自行解析市场」：Cloud 取到的字节只用于解析文档，不写入任何 Workspace，
也不经过 Node。

### D4：逐文档导入、有界失败码与幂等升级

一份文档一个结果：

| 结果 | 内容 |
| --- | --- |
| `imported` | `workflow_id`、`snapshot_version` |
| `failed` | 有界错误码：`document_invalid_json`、`document_without_name`、`document_not_executable_graph`、`name_conflict` |

`snapshot_version` 是 Cloud 发布规则分配的下一个整数，不是文档里写的版本，所以没有「版本不可寻址」
这一类失败（见 [workflow 后续决策](../workflow/20261016-asset-import-and-published-snapshots.md) D1）。

- **逐文档失败，不整包失败**：一份坏 JSON 或非法图只让该文档失败，同包其他文档照常导入。这与
  skill 插件的整包校验相反，理由不同：skill 贡献集合是插件身份的一部分，而这里每份文档都是彼此
  独立、且最终属于用户的资产；让一份坏文档连带丢弃其余可用工作流，损失的是用户数据。包完整性由
  sha256 保证，不再由「每份文档都合法」承担。
- **名称冲突不改名、不覆盖**：按现有 tenant 级唯一规则让该文档失败（`name_conflict`）。自动改名
  会让同一份来源在不同租户导入出不同名字，来源记录无法复核。
- **幂等与升级只增不改**：`(canonical Plugin ID, package version, document_path)` 是来源身份；
  同一来源身份最多导入一次，重试、进程重启、重复安装都不重复创建。包升级时，旧版本已导入的来源
  路径**不再导入**（用户的副本可能已被编辑，覆盖就是数据损失），只导入新版本新增的路径；新版本
  删掉的路径不删副本。包内版本提升导致的文档内容变化，只能由用户在副本上显式操作。
- 导入结果必须独立持久化并可单独读取（按来源身份与文档路径），不能只作为插件安装结果的一个字段。

### D5：四种证据分离

| 证据 | 由谁产生 | 能证明什么 |
| --- | --- | --- |
| 安装成功 | Node 执行结果 → 实例行与 space 聚合（既有） | 包已落到某个运行 Workspace；不证明有任何模板 |
| 接管成功 | 本文的逐文档导入结果 | 该文档已成为本租户的一份 workflow；不证明当下仍可用、也不证明能跑 |
| 发布可用 | 该 workflow 未被归档且有快照，并进入 `@` 目录投影 | 现在可以被引用；不证明运行能启动 |
| 执行可用 | 既有运行状态与启动门禁 | 运行真的起来了 |

四者互相不能推导：`installed` 不宣称有模板；导入成功不宣称可执行；导入失败不改变插件安装状态；
用户事后归档副本会让「发布可用」失效而不回退「接管成功」（后者是历史事实）。因此导入结果与安装
结果必须分别展示，不能合成一个「装好了」的布尔值。

### D6：预置只做管理员显式安装

首期只有一条入口：管理员把 workflow 插件显式安装到某个 space（沿用既有管理员门控与期望状态）。
市场源登记、目录可列出、`marketplace_visible` 都不等于预置；新租户不会被自动改成安装了任何包。
官方示例与默认模板入口留二期。

## 不变量

1. 导入副本是用户数据：可编辑、可改名、可归档；来源记录既不阻止改动，也不暗示所有权。
2. 卸载插件、包升级与来源消失都不删除或改写已导入的副本与既有快照。
3. 下载、校验、解压与解析都在数据库事务外；导入结果落库在事务内。
4. sha256 校验先于任何解析，且下载与解压都有上界。
5. 同一来源身份（canonical Plugin ID + package version + document path）最多导入一次。
6. 插件身份不出现在 `workflows` 行的身份里；tenant 级名称唯一性不变，冲突逐文档失败。
7. 资产导入不改变插件安装状态；插件安装状态也不宣称任何资产可用。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 模式 B：引用包内文档，不复制 | 包升级或卸载会改变用户资产，用户编辑无处安放；与 desktop 已实现的「用户数据」语义分叉；`workflows` 行会变成由插件身份拥有，破坏不变量 6 |
| 每个 Workspace 的安装都触发导入 | 同一租户会重复建文档（#73 的范围已排除）；没有 live workspace 的空空间拿不到模板 |
| Cloud 把包写入 Workspace，或让 Node 解析文档后上报 | 与「安装落盘只由 Node 完成」「Node 不自行解析市场」冲突；同一份包在两处解析，格式必然分叉；空空间没有 Node |
| 按包升级覆盖副本 | 破坏用户编辑，也与「已发布快照不可变」矛盾 |
| 名称冲突时自动改名 | 同一来源在不同租户导入出不同名字，来源记录无法复核；用户也看不到冲突这件事 |
| 一份文档非法就整包不导入 | 用用户数据为包的完整性买单；sha256 已经承担了包完整性 |
| 导入结果只写在插件安装结果里 | 无 live workspace 的空间没有安装结果，模板却已经存在；四证据合成一个布尔值就再也分不开 |

## 本决策未解决的问题

- pack（extension-pack）是否也能携带 workflow 文档、以及成员插件如何被选中：属于 pack 自己的
  决策，本文不预先规定。
- 官方示例包的制包与默认预置入口（#76）属于二期。
- 「用户副本与来源的差异」如何对账展示（用户改过哪一份、来源已升级到哪个版本）：需要 README 与
  界面决策，超出本次契约。

## 落地

- 阶段 1（#73）：来源记录与逐文档结果落库、资产导入步骤（下载/校验/解压/逐文档导入）、幂等与
  升级只增不改。完成条件是真实 registry `.orax` 加 HTTP 与 PostgreSQL 的证据，不以 Desktop 本地
  SQLite import 或 `Installed(version)` 单独验收。
- 阶段 2（#76）：预置入口与官方示例包。
- 两个阶段都不改动插件安装与移除的既有路径；插件实例的状态仍只由 Node 执行结果的接管写入。
- 核心测试用例在本 ADR 转为 `approved` 后按最终决策补充。
