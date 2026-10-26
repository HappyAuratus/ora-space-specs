---
status: approved
---

# 变量聚合由 Cloud 调度器执行：按声明顺序透传第一个已赋值的候选

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`approved`）的后续决策。根决策
D4 把 `aggregator` 列为「随所属特性一起」的延后项，发布门禁拒绝它；本文承接该项的去向，并沿用
[ora-space/circus#448](https://github.com/ora-space/circus/issues/448) owner 选定的「继续 Go 移植」。
结论：

- **wire 形状**沿用 desktop：`data.aggregatorConfig.variables` 是**有序、非空**的根选择器列表
  （`string[][]`，每项 `[nodeId, root]`，不允许嵌套路径）。
- **执行语义**沿用 desktop `aggregator.rs` 的 V1 contract：节点按声明顺序取**第一个已赋值**的候选，
  原样透传为 `{aggregatorId}.output`；**已赋值 = 键存在**，`null`、`false`、`0`、`""`、空列表、空对象都算；
  全部未赋值是稳定失败 `aggregator_no_match`，不是空成功。
- **发布门禁**接受 `aggregator`，校验列表形状、根选择器、重复、根声明与传递前驱；容器边界按 D1 判断，
  `sys.*` 与 globals 免前驱校验。保存草稿仍然宽松。
- **运行时**不读分支决策，只读已提交的节点状态与环境变量（`sys.*`、globals）。节点自身不自动重试。

**状态与影响面**：本文为 `approved`；核心验证义务见[变量聚合核心用例](../../../test-cases/cloud/workflow/variable-aggregator.md)，
直接证据按义务标记，不以决策获批代替实现验证。涉及 Cloud 的发布门禁、调度器，以及前端的节点类型、
调色板、检查器与导入。**兼容边界**：
不改数据库 schema、不改 OpenAPI、不改运行状态机；`node_states` 里只是多一个 succeeded/failed 的节点状态。
**非目标**：`iteration`、`subflow` 的执行、嵌套复合节点、静态类型系统（见 D3）。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [根决策](0-cloud-side-scheduler-owns-workflow-execution.md) D2：发布门禁按首期可执行子集拒绝 `aggregator` | **修改**：`aggregator` 进入可执行集合，门禁改为按配置校验，拒绝时给节点、字段、原因 |
| 根决策 D4：aggregator 延后，「随所属特性一起」处置 | **落实**：本文给出首期可执行契约；iteration/subflow 仍延后 |
| 根决策不变量 3：节点结果提交是带状态的比较交换，迟到回执是空操作 | **继承**：aggregator 是无副作用的同步节点，结果随同一行的版本 CAS 提交 |
| Cloud 根决策 D1 / D4：调度器只从已提交状态重算 | **继承**：候选的「已赋值」只由 `node_states` 与环境变量决定，不引入新的调度器状态 |
| [desktop 根决策](../../desktop/core/workflow/0-node-runtime-orchestration.md) D6：Aggregator 候选只能有一个已赋值值，多值冲突失败 | **修改**：Cloud 的 aggregator 按候选声明顺序透传第一个已赋值值；多个候选同时赋值也不冲突。这是对 D6 规则的显式变更，选择与 desktop V1 执行契约一致的确定性优先级语义 |
| desktop 根决策 D6：节点产物写入变量池，区域不能绕过 owner 与类型校验 | **继承 owner 边界**：Cloud 声明 `{aggregator}.output` 一个根；静态类型校验的差异见 D3 |
| desktop `workflow_run/engine/aggregator.rs` V1 contract：有序、非空、根选择器、first-assigned、no-match 失败 | **继承**：逐项一致；本文是 Cloud 侧的实现决策 |
| [loop 与 loopExit 后续决策](20261025-loop-and-loop-exit-execution.md)：loop body 是独立作用域，有效容器统一解析 | **交叉约束**：body 内前驱只在本 body 图里计算；外层值只经可见祖先或 carried 根读取，不越过容器边界（D1） |

## 决策

### D1：wire 形状与发布门禁

`data.aggregatorConfig.variables` 是根选择器列表：

```json
{"variables": [["writer", "output"], ["reviewer", "output"]]}
```

门禁逐项校验并拒绝下列情况（`code` / `nodeId` / `field`）：

| 拒绝原因 | 条件 |
|---|---|
| `aggregator_config_missing` | 没有 `aggregatorConfig` 或它不是对象 |
| `aggregator_variables_not_a_list` | `variables` 缺失或不是列表 |
| `aggregator_variables_empty` | `variables` 是空列表 |
| `aggregator_selector_unsupported` | 某项不是「节点 id + 根」两段非空字符串，或带嵌套路径 |
| `aggregator_selector_duplicate` | 两项指向同一个 `{node}.{root}` |
| `aggregator_selector_undeclared` | 选择器指向的根不在变量根声明表里（含 `sys.*`、globals 与各节点声明的根） |
| `aggregator_selector_not_upstream` | 根已声明，但 producer 不是当前作用域内的传递前驱，也不属于下述外层可见祖先或 carried 根；`sys.*` 与 globals 豁免前驱校验 |

**前驱与容器边界**：前驱是图上沿有向边可到达 aggregator 的**严格传递前驱**，不包括节点自身。
发布校验依据图结构，不依据节点排列、执行先后或运行时恰好已经存在的值；未被实际分支选中的节点仍可以是
图上的前驱，其根在运行时未赋值时按 D2 跳过。

| aggregator 所在位置 / 候选来源 | 发布时的允许条件 |
|---|---|
| 外层 aggregator 选择外层节点 | producer 必须在外层图上是 aggregator 的严格传递前驱；Loop 在外层图上是一个节点，只能选择其已声明的命名 outputs |
| loop body 内选择本 body 节点 | 只在同一 body 的节点与边构成的图里求严格传递前驱；不能借外层路径补成前驱 |
| loop body 内选择外层节点 | producer 必须在外层图上是所属 Loop 的严格传递前驱，即进入 Loop 时已可见的祖先；Loop 旁边没有该前驱关系的兄弟分支不允许 |
| loop body 内选择 `[loopId, variableName]` | 只允许所属 Loop 已声明的 carried 根；它是本轮作用域的输入，不靠跨容器连边建立前驱关系，也不借此读取该 Loop 的最终 outputs |
| 选择 `sys.*` 或 globals | 免前驱校验，仍须通过根声明校验 |

成员归属沿用 Loop 的统一解析：非空 `data.containerId` 优先，缺失或为空时回落到 `parentId`；两者冲突时
使用显式 `containerId`。门禁、前驱计算与运行时可见性使用同一归属。外层节点不能直接选 body 成员的根；
一个 body 不能直接选另一个容器的内部成员，也不能选外层非祖先节点；这些根即使全局已声明，仍以
`aggregator_selector_not_upstream` 拒绝。容器之间的产物只能通过容器已声明的命名 outputs、按外层前驱规则传递。

运行时只解析进入 Loop 时的可见祖先、globals、当前轮 carried 与本轮已提交的可见前驱值，不从其他轮归档
或其他 body 的扁平状态中补值。这样同一图的候选优先级不会被兄弟分支的调度顺序改变；如果只校验根声明，
兄弟分支可能先于 aggregator 提交，也可能尚未执行，first-assigned 的结果就会随调度先后变化。

保存草稿不校验；发布是「这份图将成为运行执行的冻结文档」的时刻（根决策 D2）。**安装、模型与授权不是
聚合节点的门禁**：它没有执行器槽位、不建会话，也不需要 Workspace。

### D2：执行语义——first-assigned 原样透传

- 调度器在节点就绪时按声明顺序扫描候选，读取其 `{node}.{root}`；**第一个存在的键**胜出，值原样写入
  `{aggregator}.output`。
- **已赋值是键存在，不是真值**：`null`、`false`、`0`、`""`、`[]`、`{}` 都会被透传。把它们当作「未赋值」
  会让一个明确产出了空值的分支消失，而作者恰恰用它表达「这一轮没有结果」。
- **没有候选被赋值**：节点失败，`output.reason = aggregator_no_match`，运行失败；失败信息列出扫描过的候选，
  便于作者定位。绝不输出空对象并记为成功。
- **多个候选同时赋值**：仍选声明顺序中的第一个。例如候选为 `[["writer","output"],["reviewer","output"]]`，
  两者都有值时始终透传 writer 的值。这一行为修改 desktop 根决策 D6 的「多值冲突失败」，不是把既有决策
  当作笔误忽略；理由是聚合的优先级由作者显式声明，且与 desktop V1 的实际执行契约一致。
- **不读分支决策**：节点不知道 Condition 存在。哪条分支跑过只体现在已提交状态上；未走的分支没有状态，
  其根自然「未赋值」。
- **环境变量可见**：`sys.workflow_id`、`sys.timestamp` 与 globals 与节点状态一起参与解析，因此候选可以选择
  全局变量；这与 condition 的取值口径一致。
- **无自动重试**：aggregator 是同步、无副作用的节点；no-match 是定义/产物问题，重试不会改变结果。运行级
  失败与取消沿用既有路径，节点没有 in-flight 状态。

### D3：保留静态类型相等这一项差异

| 差异 | desktop 行为 | Cloud 现状 | 处置 |
|---|---|---|---|
| 静态类型相等 | parse 期要求所有候选的声明类型相同，并用该类型声明输出 | 无类型系统；校验结构、根声明与 D1 的前驱/可见性，运行时按键存在取值 | 接受差异：不在发布期保证候选类型相同；实际值原样透传，下游仍按自身取值契约校验。将来引入静态声明集时再补类型相等 |

这一差异只放宽类型承诺，不放宽 producer 的前驱与作用域边界，也不把未知值或缺键伪造成已赋值。

### D4：视图与 API

- 节点状态仍是既有 `workflowNodeState` 形状：`status` / `displayName` / `startedAt` / `finishedAt` /
  `output` / `error`；成功时 `output` 只有一个键 `output`，失败原因沿用 D2 的 `output.reason`。
- 运行详情与历史不需要新字段；OpenAPI 无变化，无数据库迁移。
- 前端检查器编辑的就是 `aggregatorConfig.variables`；顺序即优先级，面板不自动重排。

## 不变量

1. 候选列表在发布时非空；顺序就是优先级，调度器绝不重排。
2. 「已赋值」只按键存在判断，与值的真假无关。
3. 零个已赋值候选 = `aggregator_no_match` 失败，绝不空成功。
4. 节点只声明一个根 `output`；不透传多个候选，也不合并值。
5. 发布门禁与运行时读同一份 `variables`；运行时对无法解析的文档报 `aggregator_invalid_config`，不猜测。
6. aggregator 是同步无副作用节点：不建会话、不重试、不持有运行状态。
7. 节点候选只读当前作用域的严格传递前驱、进入容器时的可见祖先或所属容器的 carried 根；`sys.*` 与 globals
   免前驱校验。非前驱、跨 body 与反向读取 body 的配置在发布时被拒，不由运行期先后决定是否可读。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 多个候选同时赋值即冲突失败（desktop 根决策 D6） | 能暴露意外的多源赋值，但不能表达作者指定的候选优先级；本文显式修改这项决策，选择与 desktop V1 契约一致的 first-assigned，多个已赋值候选按声明顺序确定结果 |
| 在 Cloud 建一套静态类型推断来对齐 desktop 的 parse 期校验 | 需要变量声明集、节点输出类型与 globals 类型的完整推导，超出聚合节点本身；当前不引入类型系统（D3） |
| 只校验根声明，允许非前驱候选在运行时缺值就跳过 | 校验成本较低，但兄弟分支可能已经提交或尚未执行，同一图的优先候选会受调度顺序影响；本文在发布时拒绝，使用 D1 的前驱与作用域规则 |
| 把聚合做成 Junction 的一种模式 | `junction` 是 Cloud 独有的结构性合流占位，desktop 没有执行语义；聚合有确定的取值语义，混在一起会让两类概念都失去边界 |
| 运行时遇到空值就视为未赋值 | 会让明确产出的 `null`/`false`/`""` 消失，改变作者表达；desktop 的键存在判定是更准确的语义 |

## 本决策未解决的问题

- **静态类型系统**：Cloud 何时引入变量的声明集与类型相等，由后续决策给出；届时补上 D3 的差异。
- **iteration 内的聚合**：iteration 尚未实现；本文只保证聚合节点在任意已提交作用域内按同一规则工作。
- **检查器的变量选择器**：当前是 `node.root` 文本输入，选择器化属于编辑器体验，不改变契约。

## 落地顺序

1. **发布门禁与执行**：完成条件是合法配置可发布，非法配置与非前驱候选按 D1 定位拒绝；
   first-assigned、空值计为已赋值、no-match 失败及容器隔离均有直接门禁/调度器证据。
2. **编辑器**：完成条件是节点可创建、编辑、导入，有序候选在保存和发布后保持原顺序，运行视图显示产物与失败原因。
3. **真实栈端到端与落地**：完成条件是发布 → 启动 → 分支汇合 → 运行详情的真实链路验证通过，
   核心用例逐项登记直接证据；完成后才将 ADR 标记为 `implemented`。
