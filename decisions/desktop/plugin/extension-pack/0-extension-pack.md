---
status: implemented
---

# 插件集合包：Extension Pack 是安装编排条目，不是运行时插件

Ora 的 marketplace 新增一种条目形态：Extension Pack（集合包）。pack 用一份 `kind = "pack"` 的
release 清单声明一组成员插件（`[[pack.members]]`）与每个成员的 agent 适配范围，自身不携带任何
可下载资产。用户安装一个 pack 时，宿主在 preflight 通过后按声明顺序逐个安装适用成员，复用既有
单插件安装链路；pack 自身永远不会成为已安装插件、不生成 `.orax`、不进入发现与更新生命周期，
但 pack 安装会单独持久化一份所有权日志，卸载 pack 走所有权感知的成员回收。同时新增 release
形态可选字段 `marketplace_visible`（缺省 `true`）：声明为 `false` 的条目不进入派生索引、不在
marketplace 前端独立展示，但仍保留在源仓库中、可按 identifier 解析、可被 pack 安装，安装后是
真实 Installed Plugin。

本 ADR 已随 Extension Pack V1 能力落地为 `implemented`：正文固定当前稳定边界，其中被后继决策
修改的行为（成员失败回滚、pack 卸载、所有权持久化）以指向对应日期决策的方式标注，设计演进链
由 `20260916-*` 系列决策承载。涉及模块为 `ora-plugin-manifest`（清单 schema）、
`ora-plugin-registry`（索引构建）、`crates/backend`（安装编排与契约）、`ora-plugin-manager`
（成员安装复用）、`ora-contracts` 与 `packages/app-shell`（聚合结果与前端呈现）。

兼容性边界：pack 条目在旧宿主上整条消失（未知 kind 硬拒绝的既定行为），隐藏成员在旧宿主上
仍然可见（未知字段忽略的既定保护），两者的不对称是接受的代价而非需要迁移的状态（见 D8）。
本决策不承担插件依赖求解与安装前成员预览——见"本决策未解决的问题"；pack 级卸载已由
[Pack 卸载所有权决策](20260916-pack-uninstall-ownership.md) 承担。

## 术语与边界

- **Listing（marketplace 条目）**是 marketplace 源仓库 `registry/**/orax.toml` 中的一份
  release 清单；**派生索引（Registry Index）**是宿主扫描全部源后写出的展示缓存
  `registry_index.json`；**Installed Plugin**是 `installed/<namespace>/<identifier>/<version>/`
  下真实落盘的包。三者是不同层，本文显式区分。
- **Extension Pack**是 `kind = "pack"` 的 listing；**成员**是它声明的插件 identifier；
  **通用成员**是未声明 `agents` 的成员；**agent 专属成员**是声明了 `agents` 的成员；
  **隐藏条目**是声明 `marketplace_visible = false` 的 listing。
- **Agent Plugin**是 `kind = "agent"` 的已安装插件；其运行期身份是完整 canonical Plugin ID
  `<namespace>/<identifier>`（来源派生 Namespace 决策 D8），会话把它持久化在 `agent_cli`。

一个 pack 条目与其隐藏成员的示例：

```toml
# registry/o/ora-space.python-extension-pack/orax.toml
resolver = 1
identifier = "ora-space.python-extension-pack"
title = "Python Extension Pack"
kind = "pack"
version = "0.1.0"
description = "Python 开发所需的技能、MCP 与 agent 专属工具集合"

[[pack.members]]
identifier = "ora-space.python-core"                  # 通用成员

[[pack.members]]
identifier = "ora-space.claude-python-tools"
agents = ["ora-space.claude"]                         # agent 专属成员

# registry/o/ora-space.python-core/orax.toml（隐藏成员，节选）
# resolver = 1 / identifier = "ora-space.python-core" / kind = "skill"
marketplace_visible = false
url = "…"
sha256 = "…"
```

## 与相邻决策的边界

本文固定 Extension Pack 能力本身的系统承诺，沿用既有分工而不复制权威规则：

- `orax.toml` 字段级 schema（`pack` 拼写、`[[pack.members]]` 的类型策略与双形态约束、
  `marketplace_visible` 的类型策略）的权威定义属于
  [插件清单根决策](../manifest/0-versioned-dual-form-manifest.md)，需以该链的日期决策登记；
  本文 D1、D3 只固定能力视角的规则。
- 索引构建与缓存机制的权威定义属于
  [插件分发根决策](../registry/0-registry-distribution.md)，可见性过滤需以该链的日期决策登记；
  本文 D3 只固定过滤语义与"resolve 不受影响"的承诺。
- [来源派生 Namespace 决策](../registry/20260831-source-derived-namespace.md)不变：namespace
  由宿主派生并持久化、agent 身份是 canonical Plugin ID（D8）；本文在其之上引用。

## 决策

### D1：pack 是 `kind = "pack"` 的 release 清单，不携带发布资产

pack 条目自述身份与成员集合，不承载任何运行时贡献物与发布资产：禁止顶层 `url`/`sha256`、
`[[targets]]` 与 installed `[artifact]`；`[[pack.members]]` 对 pack 必需且独占，对其余 kind
禁止；`[workbench]`/`[webview]` 对 pack 禁止；`kind = "pack"` 只允许出现在 release 形态，
installed 形态出现即报错。pack 条目按既有目录约定携带 README 与 logo，仍被索引展示。

分发规则已把"无发布资产的条目仍被索引与展示、只是本机不可安装"作为合法状态。pack 是该状态的
显式化：它天然可被扫描、解析、展示，但不属于 `Installer` 的可安装对象。不新增第二套条目文件
（如 `pack.toml`），因为索引扫描、README/logo 约定、解析入口与安装 resolve 只有同一条流水线，
pack 复用全部既有路径，只需在 kind 判定处分叉。

"这个条目不可安装、不可运行"必须能从 schema 一眼判定，而不是靠"恰好没写 url 加恰好写了
members"的组合推断。让非法组合在解析期失败（pack 带 `url`、非 pack 带 `[[pack.members]]`），
保持了清单根决策 D4"清单不能是半个两种 kind"的承诺。

### D2：成员用裸 `identifier` 声明，并解析在 pack 所属的同一个 Marketplace Source 内

`[[pack.members]]` 的 `identifier` 是 `PluginName` 形态的裸标识符；成员的完整 Plugin ID 由宿主
组装为 `<pack 的 namespace>/<member identifier>`，成员解析永远发生在 pack 自己的源检出内。
V1 不支持跨源成员。

namespace 由宿主从 source canonical URL 派生并在首次绑定时持久化，作者无法预知第三方源派生出
的 namespace（来源派生 Namespace 决策 D2/D3 的同一逻辑）：让 pack 作者书写完整 canonical ID，
等于让不可验证的自述参与引用。裸 identifier 还使 pack 在官方源与 fork 源之间原样可用——这正是
"fork marketplace 仓库后作为自己的源使用"的工作流所要求的。跨源引用需要成员级依赖求解与多源
顺序语义，超出 V1 边界。

### D3：`marketplace_visible = false` 的条目不进入派生索引，但保持可解析、可安装

`marketplace_visible` 是 release 形态的展示意图声明：`false` 的唯一语义是"该条目不作为独立
listing 进入 marketplace 前端的浏览、搜索与详情入口"。它不表示 disabled、不可安装或不可解析——
条目仍在源仓库 registry 中，仍可按 identifier 解析，仍可被 pack 安装，安装后是真实 Installed
Plugin。缺省 `true`；installed 形态出现该字段直接报错（与 `[[targets]]` 不允许出现在 installed
形态同一先例）：可见性是 marketplace listing 的属性，已落盘的包没有展示面。

过滤发生在索引构建期、生成索引条目之前，而不是投影或前端：隐藏条目不可能通过任何派生路径泄漏
给前端，每一层消费者都不需要理解和转发一个可见性标志。索引 schema 不因此新增字段，缓存版本
不变；旧宿主（或升级前）写出的缓存可能短暂包含后来变为隐藏的条目，到下一次同步自然清除——为
一个无害的临时可见性去作废整份缓存、让市场在用户手动刷新前显示为空，不值得。

字段放在清单而不是 registry 侧的旁路元数据里，是因为索引构建与安装 resolve 解析的是同一份
`orax.toml`；旁路元数据会让"列表里看不到"与"安装时读到的声明"成为两个可漂移的事实。命名上
否决 `listed`（会被误读为"是否存在于 registry"——隐藏条目恰恰存在于 registry）与
`hidden = true`（否定式默认要求读者先计算"未写即可见"）。

### D4：`agents` 把成员标记为 agent 专属，按已安装的 agent 插件过滤

成员的 `agents` 字段（可选）声明该成员仅在被引用的 agent 插件已安装时才属于安装集合；缺省
表示全 agent 通用。每一项是 agent 的裸 `identifier`（如 `ora-space.claude`），也接受完整
canonical ID；匹配规则：存在任一已安装的 `kind = "agent"` 插件，其 `identifier` 等于该引用，
或其 canonical ID 等于该引用，则该成员入选。引用拼写在清单解析期校验；引用未命中已安装 agent
不是错误，只影响成员取舍。

Ora 没有全局"当前使用的 agent"概念——agent 绑定在 session 上（`sessions.agent_cli`），而
marketplace 安装发生在设置页。因此"根据当前使用的 Agent 安装"在宿主侧落地为"按已安装的
agent 插件过滤"：装了 Claude Code 就装 claude 专属成员；之后装了 Codex，再次安装 pack 会自然
补齐 codex 专属成员。

这个匹配是一次性的展示期判定，引用值不进入任何持久化状态、协议字段或安装目录，因此不与"agent
标识使用 canonical Plugin ID"（来源派生 Namespace 决策 D8）冲突——D8 管身份与持久化，本条管
listing 期过滤。跨源同 identifier 的 agent 都已安装时，过滤结果同为"入选"，布尔语义下不存在
歧义；不使用完整 canonical ID 作为唯一拼写，是因为 namespace 由宿主派生、作者不可预知，与 D2
的理由相同。

### D5：安装一个 pack = preflight + 顺序安装成员，已安装成员跳过

安装入口复用既有 `installPlugin` 契约（`pluginId` = pack 的 canonical ID）。宿主 resolve 出
清单后发现 `kind = "pack"`，转入编排而不是 `Installer`：

1. **preflight**（下载任何字节之前）：成员 `identifier` 去重校验、自引用校验（成员 identifier
   等于 pack identifier）、逐成员在 pack 所属源内解析清单（成员不存在即失败）、成员清单
   `kind = "pack"` 即失败（V1 禁止嵌套）、按 D4 选出适用成员（无通用成员且无命中的 agent 专属
   成员即失败）、对入选成员逐个执行纯函数的 release 选择（当前 host triple 不兼容即失败）。
2. **顺序安装**：按声明顺序对每个入选成员执行既有单插件安装（下载 → SHA-256 → staging → 包
   校验 → 原子提交，全部既有校验不变）。成员已安装（任意版本）则记为 `skipped` 并继续，不报
   错，既有版本不被改动或降级。
3. **结果**：返回聚合 outcome（见 D6）；进度事件沿用现有 `plugin-install-progress`，以 pack 的
   pluginId 发出，进度按"已完成成员数 + 当前成员内部进度，除以入选成员总数"聚合。

顺序而非并行安装：复用单插件安装的全部原子性与错误定位，进度与失败一一对应；V1 成员数量小，
并行的收益不值得引入并发编排。

### D6：preflight 失败零下载；成员失败停止运行并回滚本次创建的成员，结果逐成员可辨

preflight 任一失败即整包失败，返回 typed 错误且不产生任何下载与落盘。成员重复、自引用、成员
不存在、嵌套 pack、无适用成员复用或新增封闭的 public error 变体；入选成员与当前 host 不兼容
复用既有 host 不兼容语义，但发生在 preflight 而非首个成员安装中途。

preflight 通过后的成员安装中，任一成员失败：剩余成员不再尝试，**本次运行创建的成员**按逆创建
顺序回滚（复用所有权感知的既有单插件卸载链路）；本次运行之前已存在的成员与成员关系不受影响。
契约 `InstallOutcome` 新增 `PackInstalled { members, skipped, failed }`：`members` 按声明顺序
携带已安装成员的 `{ plugin_id, outcome }`（复用既有单插件 outcome 枚举，含命令别名冲突报告），
`skipped` 列出已安装被跳过的成员，`failed` 携带首个失败成员、其错误码与回滚失败诊断。用 `Ok`
outcome 如实上报失败，而不是把失败伪装成整包成功。本节的回滚语义由
[Pack 安装事务性回滚决策](20260916-pack-install-transactional-rollback.md) 取代了最初
"停止且不回滚"的方案，该决策保留了回滚只撤销 `created_this_run` 的边界。

### D7：pack 不进入已安装模型；安装关系单独持久化，卸载按所有权回收

pack 不落盘、不产生 `InstalledPlugin`、不参与发现/更新流程；marketplace 前端对 pack 卡片永远
呈现安装入口。pack 条目的 `version` 是成员集合的内容版本，纯展示；成员变化后重装 pack 即补齐
新成员，已装成员跳过。成员被单独卸载后再次安装 pack 会重装该成员——这是期望行为：pack 表达的
是"这套集合应该存在"，不是"曾经装过"。

pack 与成员的生命周期有意分离：成员是真实插件，拥有完整的卸载/更新/数据目录生命周期；pack
自身不伪造包目录、贡献物类型与发现规则。pack 的"已安装"状态由单独持久化的所有权日志承载
（[Pack 安装所有权决策](20260916-pack-installation-ownership.md)）：日志记录每个被 pack 点名
的成员是本次安装创建的（`managed_by_pack`）还是点名时已存在（`pre_existing`）及建立关系时的
版本，记录本身不是 InstalledPlugin。卸载 pack 依据日志与重启对账计算计划，只回收 pack 创建
且未被用户改变的成员（[Pack 卸载所有权决策](20260916-pack-uninstall-ownership.md)）；重启
对账只识别漂移、不改写日志
（[Pack 重启对账决策](20260916-pack-restart-reconciliation.md)）。本节的持久化与卸载语义由
20260916 系列决策在"pack 不进入已安装模型"的边界之上补充，取代了最初"不参与卸载生命周期"的
表述。

### D8：旧宿主对新条目的行为是确定的

- **pack 条目在旧宿主上整条消失**：未知 kind 是清单根决策 D4 的显式硬错误，索引构建按既有
  规则跳过并记 warning。这是封闭 kind 设计的既定代价；pack 消费者需要新版宿主。
- **隐藏成员在旧宿主上仍然可见**：`marketplace_visible` 是未知字段，被旧宿主忽略，条目照常
  进入索引。这是清单根决策 D2"未知字段忽略"对增量 schema 的既定保护。

两者都是既有决策在新条目上的自然投影，本决策不引入版本探测、不承担对旧宿主的迁移义务。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| pack 作为普通插件打包成 `.orax` 并安装 | pack 无运行时贡献物；它会进入发现/校验/生命周期却没有任何可运行内容，且"安装包"与"成员包"两个事实来源要在 installer 内互相解释 |
| 独立 `pack.toml` 与独立 listing 流水线 | 索引扫描、README/logo 约定、resolve 路径、前端 listing 模型都要第二套；两个条目模型漂移后"能上架"与"能安装"再次分叉 |
| 成员/agent 引用使用完整 canonical ID | namespace 由宿主从 source URL 派生，作者不可预知；fork 源部署下所有引用都要改写，与清单不声明 namespace 的既定身份边界冲突 |
| `marketplace_visible` 放进 registry 侧旁路元数据，由投影或前端过滤 | 索引与 resolve 读取两个事实来源，可见性声明与安装声明可漂移；每层消费者都要记得尊重标志，遗漏即泄漏 |
| 安装 pack 时安装全部声明的 agent 专属成员 | 给未安装对应 agent 的用户塞入不可用的插件，与"按 agent 适配安装"的需求直接相悖 |
| 成员并行安装 | 单插件安装的原子性与错误定位被并发编排稀释；V1 成员量小，收益不成比例 |
| 成员失败自动回滚全部已装成员（含既有成员） | 最初的边界；"卸掉用户既有的可用插件"是破坏性行为。最终采用的折中见 [Pack 安装事务性回滚决策](20260916-pack-install-transactional-rollback.md)：只回滚本次运行创建的成员，既有成员关系不受影响 |
| 通用 plugin-to-plugin 依赖求解器 | 版本区间、拓扑排序、冲突求解是依赖系统问题；当前需求只是"一次装多个 + 按过滤取舍"，用编排即可满足，求解器是不受支持的复杂度 |

## 不变量

1. pack 条目不携带任何可下载资产；安装 pack 不下载 pack 自身的任何字节。
2. pack 永远不会以 `InstalledPlugin`、安装目录或包内清单的形式存在；所有权日志记录不是
   InstalledPlugin。
3. preflight 未通过时，不发生任何下载，`installed/` 不发生变化。
4. 成员只在 pack 所属的 Marketplace Source 内解析；V1 不存在跨源成员。
5. 已安装的成员在 pack 安装中被跳过且不算失败；其既有版本不被改动或降级。
6. 任一成员安装失败后，不再尝试后续成员；本次运行创建的成员按逆创建顺序回滚，本次运行之前的
   成员与所有权关系保持不变。
7. pack 安装的结果必须逐成员可辨（已装/跳过/失败），不存在整体布尔成功。
8. `agents` 引用不进入持久化状态、协议字段或安装目录；它只在安装时与已安装 agent 插件求值。
9. `marketplace_visible = false` 的条目不出现在任何派生索引文件中，但始终可按 identifier 在
   其所属源内解析与安装。
10. 成员安装复用单插件安装的全部校验（SHA-256、staging、包校验、原子提交），不引入新的落盘
    路径。
11. pack 安装的所有权关系单独持久化：`managed_by_pack` 与 `pre_existing` 连同
    `version_at_install` 一起记录；marketplace 清单的内容变化不重写历史日志。
12. pack 卸载只回收 `managed_by_pack` 且对账为 `ExpectedAndPresent` 的成员；`pre_existing` 与
    独立升级（`VersionChanged`）的成员永远保留，`Missing` 成员不阻塞其余成员的回收。

## 本决策未解决的问题

- pack 详情页与"安装前成员预览"：需要把成员的 listing 元数据经新契约暴露给前端，或为隐藏
  条目增加另一条读取路径；留待 V1 之后的 ADR。
- 嵌套 pack、跨源成员、成员版本区间约束：需要依赖求解语义，明确不在 V1。
- `agents` 引用未来是否扩展"agent 能力标签"（如 `any`、`none`）以表达反选语义。
- pack 级卸载与成员分组展示已由 20260916 系列决策（所有权、对账、卸载、前端呈现）解决，
  不再是本决策的未决项。

## 落地顺序

1. **清单与索引层**：`ora-plugin-manifest` 登记 `kind = "pack"`、`[[pack.members]]`、
   `marketplace_visible`（清单链日期决策）；`ora-plugin-registry` 索引构建过滤隐藏条目
   （registry 链日期决策）。完成条件：pack/隐藏条目的解析、索引、resolve 行为有单元测试。
2. **编排与契约层**：`crates/backend` 安装编排 + preflight + `PackInstalled` outcome；
   `ora-contracts` 与 TS 类型再生成。完成条件：preflight 失败零下载、成员失败停止运行并回滚
   本次创建成员的集成测试。
3. **前端层**：marketplace 卡片对 pack 的呈现、聚合进度、聚合结果反馈。完成条件：安装 pack
   的端到端用户路径可用。
4. **内容层**：marketplace 仓库提交 `python-extension-pack` 及其成员条目（含
   `marketplace_visible = false` 的隐藏成员）与 README。完成条件：真实源下完成一次按 agent
   过滤的完整安装。
