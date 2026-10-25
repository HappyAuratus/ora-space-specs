---
status: proposed
---

# Iteration 在 Cloud 执行：区域成员、foreach 轮次与按轮账本

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`approved`）的后续决策，承接
[ora-space/circus#641](https://github.com/ora-space/circus/issues/641)：把 desktop 可执行的 `iteration`
（foreach）节点迁到 Cloud。根决策 D4 把它列为延后项，Cloud 现在仍拒绝该 kind。

结论：

- **成员模型统一**：decode 时取 `effectiveContainer(node) = data.containerId ?? parentId`。iteration
  的存量草稿用 `parentId`、loop 用 `containerId`；归一后门禁与调度器只认一个键，存量草稿继续可读。
- **wire** 沿用 Cloud 编辑器已有的 `data.iterationConfig`：`iteratorSelector`、`collectSelector`、
  `errorStrategy`（`fail` 默认 / `continue`）、`maxIterations`（默认 50，≥ 1）。
- **轮次**：数组长度即轮数，第 r 轮把 `items[r]` 绑定到 `{iterationId}.item`、r 绑定到
  `{iterationId}.index`，然后驱动区域内节点一轮；全部轮结算后节点完成。
- **暴露变量**：`output`、`entries`、`failed_count`，三者类型固定，不随 `errorStrategy` 改变。
- **上限**：`N > maxIterations` 在**启动边界失败**，一轮都不执行，绝不静默截断；空数组立即成功。
- **失败**：区域内失败可被 `continue` 吸收进账本；iteration 自身的失败（迭代源非数组、超限、暴露写入
  失败）总是传播为 run 失败。

**状态与影响面**：`proposed`；实现由 circus#641 承接。涉及 `internal/core` 的门禁、调度器、运行状态与
前端检查器；**不改数据库 schema、不改 OpenAPI**（沿用 `workflow_runs.node_states` 单行 CAS）。
**非目标**：并行迭代、`maxIterations` 的界面软警告、子流程画布、变量选择器化。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D2：发布门禁拒绝首期子集之外的 kind | **修改**：`iteration` 进入可执行集合，门禁改为按配置与区域校验 |
| 根决策 D4：iteration、loop 等复合运行时延后 | **落实**：iteration 的首期可执行契约；loop 另案（specs PR #224） |
| 根决策 D5：调度器只从已提交状态重算，节点结果随版本 CAS 提交 | **继承**：每轮 body 节点状态、轮终态与账本在同一次提交里写入 |
| 0号决策 D3 六条区域边界校验（desktop） | **继承**：Cloud 复用 loop 的容器校验；第六条读作「iteration 必须声明 `1 ≤ maxIterations`」 |
| desktop `iteration.rs` 与 `20260915-iteration-composite-runtime.md`：数组长度即轮数、item/index 绑定、三暴露变量、fail/continue、超限启动失败、空数组成功 | **继承**：逐项一致，Cloud 只改成员表达与持久化编码（D1、D5） |
| 根决策 D6：变量根声明与下游可见性 | **继承**：iteration 声明三个暴露变量；`item` / `index` 只对区域内可见 |

## 决策

### D1：成员归属的归一

decode 工作流图时，把每个节点的有效容器定为：

```
effectiveContainer(node) = data.containerId  (存在且非空)
                          ?? parentId         (React Flow 的父节点)
```

- **为什么**：Cloud 前端两条路径已经固定——iteration 成员持久化 `parentId`（`use-workflow-draft.ts`
  序列化保留），loop 成员用 `data.containerId`（调色板工厂创建）。Go 侧若各写一套成员集合，门禁、
  调度器和变量可见性会出现三份不一致的判断。
- **兼容边界**：只读归一，不回写草稿、不改 wire；`parentId` 仍是 iteration 的持久化形式。
- **区域规则**（沿用 loop 的校验器）：成员不能是 `loop` / `iteration` / `subflow`；`output` 不能出现在
  区域内；成员不能跨区域；区域必须非空。

### D2：配置与门禁

`iterationConfig` 是必填对象，逐项校验（`code` / `nodeId` / `field`）：

| 拒绝原因 | 条件 |
|---|---|
| `iteration_config_missing` | 没有配置或不是对象 |
| `iteration_iterator_unsupported` | `iteratorSelector` 不是「节点 + 根」两段，或带嵌套路径 |
| `iteration_iterator_undeclared` | 迭代源根未声明，或不是区域外的可见变量（含 globals / `sys.*`） |
| `iteration_collect_unsupported` | `collectSelector` 不是两段根选择器 |
| `iteration_collect_outside_region` | 收集目标不是区域成员声明的根 |
| `iteration_error_strategy_unknown` | `errorStrategy` 不是 `fail` / `continue` |
| `iteration_max_iterations_invalid` | `maxIterations` 不是 ≥ 1 的整数 |
| `iteration_body_empty` | 区域没有成员 |
| `iteration_body_unreachable` | 区域内没有入口成员（区域内无前驱的成员），或存在从入口不可达的成员 |

`maxIterations` 缺省 50；**不设上限**（desktop 只要求 ≥ 1），Cloud 的 while-Loop 另案的上限不与本节点共用。
保存草稿不校验；发布是冻结执行文档的时刻（根决策 D2）。

### D3：轮次与作用域

- 迭代源解析为数组后得到 `N`；`N = 0` 是合法输入，节点立即成功，`output`/`entries` 为空数组、
  `failed_count = 0`。
- 第 r 轮（`0 ≤ r < N`）把 `{iter}.item = items[r]`、`{iter}.index = r` 注入**本轮**作用域，然后驱动
  区域 body 从入口成员开始的一轮执行；一轮内 body 的节点状态按既有 `node_states` 提交。
- 轮次串行：第 r 轮全部结算（成功或失败）后才开始第 r+1 轮。
- `item` / `index` 只对区域内可见；区域外不可见（轮内私有）。区域外可见三个暴露变量。
- body 入口成员：区域内没有来自其他成员入边的成员。没有入口或存在入口不可达成员时发布被拒（D2）。

### D4：暴露变量与账本

| 变量 | 类型 | 含义 |
|---|---|---|
| `{iter}.output` | `array` | 只含**成功轮**的收集值，保持输入顺序 |
| `{iter}.entries` | `array[object]` | 完整账本：每轮 `{item, status, output, error}`，与输入数组位置对齐 |
| `{iter}.failed_count` | `number` | 失败轮数 |

- `output` / `entries` / `failed_count` 的类型不随 `errorStrategy` 改变：策略只决定「失败轮之后是否继续」，
  不改变下游变量的类型。
- 每轮的收集值来自 `collectSelector` 指向的根；该根本轮未运行时，该轮记为失败，错误为
  `collect target did not run this round`。
- 账本以轮次为键、连续且不可变更；`entries` 按轮次投影，`output` 过滤失败轮。

### D5：持久化与取消

- 复用 `workflow_runs.node_states` 单行 CAS：运行中的当前轮 body 状态与 iteration 状态（轮次、账本摘要、
  item/index 当前值）一起提交；已完成的轮归档在 `iteration.completed` 中，最后一轮保留在顶层。
- **不新增表**、不改 OpenAPI；`iteration` 状态的 wire 形状由实现定义并在文档中登记。
- 取消：run 取消时停止当前轮、把 iteration 标记为 cancelled，不再启动后续轮；已提交的轮保持原样。
- 恢复：调度器只依据已提交状态重算，崩溃后从未结算的轮继续；已结算的轮不重跑。

### D6：失败与上限

- `N > maxIterations`：节点在启动边界失败，不执行任何轮，错误信息包含数组长度、上限值与调整指引。
  **绝不截断后成功**：那会让作者以为 N 项都处理了。
- `errorStrategy = fail`（默认）：首个失败轮使 iteration 失败，run 失败，后续轮不执行。
- `errorStrategy = continue`：失败轮记入账本并继续；iteration 正常完成，失败留在 `entries` 与
  `failed_count`。
- iteration **自身**的失败（迭代源不是数组、暴露变量写入失败、超限）总是传播为 run 失败，不被自身策略吸收。
- 无自动重试：失败轮不重跑。

## 不变量

1. 成员只由 `effectiveContainer` 决定，两个持久化键不会产生两套成员集合。
2. 轮数等于数组长度；空数组立即成功且账本为空。
3. 第 r 轮的 `item` / `index` 只在该轮区域内可见。
4. `output` 只含成功轮，`entries` 与输入位置对齐，`failed_count` 等于失败轮数；三者类型固定。
5. `N > maxIterations` 时零轮执行并失败，绝不截断。
6. 区域内失败按策略处理；iteration 自身失败总是 run 失败。
7. 已结算的轮不可变更、不重跑；取消后不再启动新轮。
8. 不新增表、不改 OpenAPI、不改已发布图的 wire 形状。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 只认 `parentId`（desktop 的做法） | Cloud loop 的成员已经是 `containerId`，只认 `parentId` 会让 loop 区域在 Go 侧变空；统一归一代价更小 |
| 只认 `containerId` 并迁移 iteration 草稿 | 迁移存量草稿是兼容边界变更，收益只是少一个 fallback；归一读法同时兼容两者 |
| 超限时截断到上限并成功 | 正确性灾难：作者以为 500 项全处理、实际只跑 50 项；与仓库 fail-loud 哲学冲突 |
| 三模式下拉（Dify） | 会让 `output` 的类型随配置变化，下游引用被迫重校验；desktop 已用「策略 × 读哪个变量」解耦 |
| 把失败轮从 `entries` 中省略 | 丢失失败归因；账本必须与输入位置对齐 |
| 新增 `iteration_runs` 表 | 违反根决策「单行 CAS、不新增表」；账本随 `node_states` 提交足够 |

## 本决策未解决的问题

- **并行迭代**：轮次串行；并行完成序与轮次序的编码留给后续决策（账本以轮次为键已为之留路）。
- **启动边界软警告**：迭代源来自 Start 输入时，部署界面可提前提示超限；权威检查仍在节点启动边界。
- **类型系统**：`collectSelector` 的声明类型相等校验依赖变量类型系统，当前不引入。
- **检查器体验**：selector 仍是文本输入；选择器化不改变契约。

## 落地顺序

1. **成员归一与门禁**（cloud#641）：`effectiveContainer`、区域规则、配置校验与错误码、表驱动测试。
2. **轮次执行**：item/index 注入、body 波次、收集、账本、fail/continue、超限启动失败、空数组；PostgreSQL
   调度器用例覆盖四种终态。
3. **暴露变量与运行视图**：三个根声明、可见性、运行详情展示。
4. **编辑器**：已有检查器接入新校验错误；真实栈 E2E 完成前不改 ADR 状态。
