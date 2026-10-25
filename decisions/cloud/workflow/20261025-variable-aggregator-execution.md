---
status: proposed
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
- **发布门禁**接受 `aggregator`，校验列表形状、根选择器、重复与根声明；保存草稿仍然宽松。
- **运行时**不读分支决策，只读已提交的节点状态与环境变量（`sys.*`、globals）。节点自身不自动重试。

**状态与影响面**：本文为 `proposed`，需 owner 评审后改为 `approved`，核心用例在批准后登记；实现由
[ora-space/circus#637](https://github.com/ora-space/circus/issues/637) 承接（cloud 分支 `feat/637-aggregator`）。
涉及 cloud 的发布门禁（`internal/core/workflow_graph.go`）、调度器（`internal/core/workflow_scheduler.go`）、
新增的 `internal/core/workflow_aggregator_graph.go`，以及前端的节点类型、调色板、检查器与导入。**兼容边界**：
不改数据库 schema、不改 OpenAPI、不改运行状态机；`node_states` 里只是多一个 succeeded/failed 的节点状态。
**非目标**：`iteration`、`subflow`、静态类型系统、跨容器 producer 可达性校验（见 D3）。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [根决策](0-cloud-side-scheduler-owns-workflow-execution.md) D2：发布门禁按首期可执行子集拒绝 `aggregator` | **修改**：`aggregator` 进入可执行集合，门禁改为按配置校验，拒绝时给节点、字段、原因 |
| 根决策 D4：aggregator 延后，「随所属特性一起」处置 | **落实**：本文给出首期可执行契约；iteration/subflow 仍延后 |
| 根决策不变量 3：节点结果提交是带状态的比较交换，迟到回执是空操作 | **继承**：aggregator 是无副作用的同步节点，结果随同一行的版本 CAS 提交 |
| 根决策 D5 / 条件求值：调度器只从已提交状态重算 | **继承**：候选的「已赋值」只由 `node_states` 与环境变量决定，不引入新的调度器状态 |
| [desktop 根决策](../../desktop/core/workflow/0-node-runtime-orchestration.md) D6：Aggregator 候选只能有一个已赋值值，多值冲突失败 | **修改**：Cloud 的 aggregator 按候选声明顺序透传第一个已赋值值；多个候选同时赋值也不冲突。这是对 D6 规则的显式变更，选择与 desktop V1 执行契约一致的确定性优先级语义 |
| desktop 根决策 D6：节点产物写入变量池，区域不能绕过 owner 与类型校验 | **继承 owner 边界**：Cloud 声明 `{aggregator}.output` 一个根；静态类型校验的差异见 D3 |
| desktop `workflow_run/engine/aggregator.rs` V1 contract：有序、非空、根选择器、first-assigned、no-match 失败 | **继承**：逐项一致；本文是 Cloud 侧的实现决策 |
| while-Loop 后续决策（另案：circus#631、specs PR #224，`proposed`）：loop body 是独立作用域 | **交叉约束**：aggregator 可以在 loop body 内；它只看到本轮作用域可见的已提交状态，语义不变 |

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

### D3：与 desktop 的两处已知差异

| 差异 | desktop 行为 | Cloud 现状 | 处置 |
|---|---|---|---|
| 静态类型相等 | parse 期要求所有候选的声明类型相同，并用该类型声明输出 | 无类型系统；只校验结构、根声明与运行期取值 | 接受差异，记录在案；输出按实际值透传，运行期行为不受影响。若将来 Cloud 引入变量的静态声明集，再补类型相等 |
| producer 必须是传递前驱 | parse 期拒绝非前驱（globals 豁免） | 未校验；非前驱的根永远「未赋值」，运行期等价于跳过 | 暂不补。补它需要图可达性校验与 globals 豁免的同一份实现；在引入前，门禁接受、运行时行为可解释（不会伪造值） |

两处差异都朝着「更宽松地接受、运行期按已提交状态解释」的方向，而不是静默改写值；这也是本文选择把它们
写进 ADR 而不是绕过的原因。

### D4：视图与 API

- 节点状态仍是既有 `workflowNodeState` 形状：`status` / `displayName` / `startedAt` / `finishedAt` /
  `output` / `error`；`output` 只有一个键 `output`。
- 运行详情与历史不需要新字段；OpenAPI 无变化，无数据库迁移。
- 前端检查器编辑的就是 `aggregatorConfig.variables`；顺序即优先级，面板不自动重排。

## 不变量

1. 候选列表在发布时非空；顺序就是优先级，调度器绝不重排。
2. 「已赋值」只按键存在判断，与值的真假无关。
3. 零个已赋值候选 = `aggregator_no_match` 失败，绝不空成功。
4. 节点只声明一个根 `output`；不透传多个候选，也不合并值。
5. 发布门禁与运行时读同一份 `variables`；运行时对无法解析的文档报 `aggregator_invalid_config`，不猜测。
6. aggregator 是同步无副作用节点：不建会话、不重试、不持有运行状态。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 多个候选同时赋值即冲突失败（desktop 根决策 D6） | 能暴露意外的多源赋值，但不能表达作者指定的候选优先级；本文显式修改这项决策，选择与 desktop V1 契约一致的 first-assigned，多个已赋值候选按声明顺序确定结果 |
| 在 Cloud 建一套静态类型推断来对齐 desktop 的 parse 期校验 | 需要变量声明集、节点输出类型与 globals 类型的完整推导，超出聚合节点本身；当前不引入类型系统（D3） |
| 要求候选必须声明在传递前驱上 | 需要图可达性实现；非前驱的根在运行期恒为未赋值，行为可解释，暂缓（D3） |
| 把聚合做成 Junction 的一种模式 | `junction` 是 Cloud 独有的结构性合流占位，desktop 没有执行语义；聚合有确定的取值语义，混在一起会让两类概念都失去边界 |
| 运行时遇到空值就视为未赋值 | 会让明确产出的 `null`/`false`/`""` 消失，改变作者表达；desktop 的键存在判定是更准确的语义 |

## 本决策未解决的问题

- **静态类型系统**：Cloud 何时引入变量的声明集与类型相等，由后续决策给出；届时补上 D3 第一项。
- **producer 前驱校验**：与上图可达性校验一起，由后续决策承接；在此之前门禁与运行时的差异按 D3 记录。
- **iteration 内的聚合**：iteration 尚未实现；本文只保证聚合节点在任意已提交作用域内按同一规则工作。
- **检查器的变量选择器**：当前是 `node.root` 文本输入，选择器化属于编辑器体验，不改变契约。

## 落地顺序

1. **发布门禁与执行**：配置校验、first-assigned、no-match 失败，表驱动测试与
   PostgreSQL 调度器用例。
2. **编辑器**：节点类型、调色板、检查器、导入与文案。
3. **真实栈端到端**：发布 → 启动 → 分支汇合 → 运行详情；证据写进运行 issue。
4. ADR 评审通过后补核心用例（命中、跳过未赋值、null 计为已赋值、no-match、顺序优先）；真实栈 E2E 完成前
   不把 ADR 标记为 `implemented`。
