---
status: approved
---

# loop 与 loopExit 由 Cloud 调度器执行：轮次并入运行聚合，后测循环语义沿用 desktop

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策，
承接[根决策 D4](0-cloud-side-scheduler-owns-workflow-execution.md) 把 `loop` / `loopExit` 列为延后项的去向，
以及 [ora-space/circus#448](https://github.com/ora-space/circus/issues/448) owner 选定的「继续 Go 移植」。
它把这两个节点从「发布门禁拒绝」改为「可发布、可由 Cloud 调度器执行」，并决定轮次状态放在哪里。结论：

- **执行模型**：沿用运行行聚合。轮次进度与已完成轮的成员状态写在 `workflow_runs.node_states[loop节点]` 的
  `loop` 字段里，活动轮成员状态留在顶层；继续用运行行的 `version` 比较交换提交，不新建 scope 表、不改控制面契约。
  恢复只读已提交的运行行。
- **语义**：与 desktop 的 [`while-loop`](../../desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md)
  逐项对齐：后测循环、carried 变量只在进入时求值一次、feedback 在同一完成快照上同时赋值、轮末求值 `until`、
  `maxIterations` 1..100、耗尽默认失败且可显式 `succeed`、外层只能经 Loop 命名 `outputs` 读取轮内产物。
- **loopExit**：只能是 loop body 内无出边的终端节点，只有所在分支被选中并成为本轮就绪节点时才提前结束；
  它跳过 feedback，直接按本轮变量池发布 outputs；串行调度下没有需要排空的兄弟执行，清理是「把本轮未执行成员
  记为既有的 `inactive` 状态」这一步，提交失败则整体失败。
- **发布门禁**：接受 `loop` / `loopExit`，校验容器结构、配置形状与 selector 声明，拒绝时给节点、字段、原因；
  运行时与门禁共用同一个校验器。保存草稿仍然宽松（继承根决策 D2 与 MCP 决策 D4）。

**状态与影响面**：本文为 `approved`，固定以下执行契约；实现与直接验证尚待完成，批准不表示已交付。
核心用例见[轮次执行与早退](../../../test-cases/cloud/workflow/loop-and-loop-exit.md)，证据缺口逐项标为 `Missing`。
涉及 cloud 的发布门禁（`internal/core/workflow_graph.go`）、调度器
（`internal/core/workflow_scheduler.go`）、`workflow_runs.node_states` 投影、前端检查器与画布、运行详情视图；
涉及 specs 的 Cloud 工作流执行契约与核心用例。**兼容边界**：不改数据库 schema，不新增运行状态；`node_states`
新增 `loop` 字段与 `loopExit` 节点记录，复用既有节点状态集合；旧读取方忽略未知字段即可；已发布快照在根决策下不可能含 loop，
所以存量运行的执行行为不变。**非目标**：`iteration`、`aggregator`、`subflow`、嵌套复合节点、loop body 内的
HITL、运行中途 resume / snapshot switch，均按各自后续决策处理。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [根决策](0-cloud-side-scheduler-owns-workflow-execution.md) D2：发布门禁按首期可执行子集拒绝 `loop` / `loopExit` | **修改**：两者进入可执行集合；门禁从「按 kind 拒绝」改为按配置与容器结构校验 |
| 根决策 D4：iteration、loop 延后，理由是把 desktop 的持久 scope 形状搬上 Cloud 是一大块引擎 | **落实一部分**：loop 用运行行内的轮次文档承载，不复制 desktop 的 scope 表；iteration 仍延后 |
| 根决策 D4 / D1 条件④：就绪集合只从已提交状态重算，副本崩溃不丢进度 | **继承并扩展**：轮内节点同样只从已提交的轮次状态重算；在途 agent 会话沿用既有「持有到自身结果」的规则 |
| 根决策不变量 3：同一运行至多一个串行执行者，节点结果提交是带状态的比较交换 | **继承**：轮次文档随同一行的 `version` CAS 提交；不存在第二个写者 |
| 根决策不变量 5：没有已提交结果的节点永不被记为成功 | **继承**：轮内节点未知状态不推进轮次，也不会让 Loop 成功 |
| 根决策 D3 / 重试后续决策：agent 节点失败按冻结策略重试 | **继承**：body 内 agent 节点照常重试；Loop 自身不自动重试，定义/取值类失败直接失败运行 |
| [重试与恢复后续决策](20261021-node-retry-and-resume-contract.md) D7：resume 首期不可用 | **继承**：Loop 失败后仍不提供 resume；不新增检查点 |
| [desktop while-loop 决策](../../desktop/core/workflow/20261013-while-loop-with-durable-round-scopes.md)：后测循环、轮次作用域、feedback 同时赋值、耗尽策略、loopExit 清理 | **继承语义，改存储**：语义与失败边界逐项对齐；作用域持久化改为运行行内的轮次文档 |
| [desktop 迭代决策](../../desktop/core/workflow/20260915-iteration-composite-runtime.md)：iteration 与 loop 是不同节点 | **继承**：本文不实现 iteration，也不把 loop 当 foreach 用 |

## 决策

### D1：执行模型——轮次文档写进运行行，活动轮的成员状态留在扁平 `node_states`

`workflow_runs.node_states` 以节点 id 为键、值是节点状态对象。Loop 的状态在同一条记录里多一个 `loop`
字段承载轮次；**活动轮**的成员状态仍以各自的节点 id 留在同一张扁平表里，完成的轮次归档进 `loop.completed`。
这样同一份文档同时满足三件事：轮次进度可恢复、运行视图仍按节点 id 读状态、既有会话/重试/分支路径不需要
认识容器。

```json
{
  "loop-1": {
    "status": "running",
    "displayName": "循环 1",
    "startedAt": "2026-10-25T02:00:00Z",
    "loop": {
      "round": 2,
      "maxIterations": 5,
      "carried": {"draft": "第一轮产出"},
      "completed": [
        {"index": 1, "carried": {"draft": ""}, "states": {"entry": {"status": "succeeded"}, "writer": {"status": "succeeded", "output": {"output": "第一轮产出"}}}}
      ]
    }
  },
  "entry": {"status": "succeeded", "startedAt": "…", "finishedAt": "…"},
  "writer": {"status": "running", "startedAt": "…"}
}
```

- `status` 沿用既有节点状态：`running` / `retrying` / `succeeded` / `failed` / `cancelled` / `inactive`；
  前端对尚无已提交状态的节点仍显示 `idle`。`inactive` 已用于 condition 未选中分支，表示该节点本轮不会执行，
  不是新增状态，也不表示失败或取消。Loop 自身不重试，`retrying` 只用于轮内 agent；`loop` 只描述
  「这一轮走到哪里」，不扩展 `workflow_runs.status` 的公开运行状态机。
- 活动轮成员状态复用 `workflowNodeState` 的形状，键是 body 节点 id，因此 body 内 agent 的会话、重试、
  `workflow_node_id` 与控制面绑定与普通节点完全一致；body 内 condition 的分支选择也存在同一个键上。
- `carried` 是「进入当前轮时」的 carried 值。一轮结束时把该轮成员状态快照进 `completed[]`：继续下一轮时
  从扁平表删除这些成员，下一轮从空状态开始；Loop 结束（成功或耗尽）时保留最后一轮的成员状态，同时归档，
  运行视图因此能看到最后执行的一轮。
- Loop 成功时，`output` 的每个键是 `loopConfig.outputs[].name`，节点 id 加输出名构成外层可引用的
  `{loopId}.{name}` 根；Loop 失败时写 `error` 与 `output.reason`（稳定错误码，见 D6）。
- **成员归一**：Go 解码后的有效容器统一取 `data.containerId ?? parentId`，非空 `containerId` 优先，
  缺失或空值时回落到 React Flow 的 `parentId`。门禁、调度器、变量可见性只用这个解析入口；两个键冲突时
  显式 `containerId` 优先，不产生第二份成员集合。不回写、不迁移存量草稿；iteration 使用同一规则。

**行内状态的体积预算**：只限制轮数不能限制 JSON 字节数。本文为模型 X 固定以下边界；MiB 为
`1024 * 1024` 字节，体积按实际写入的紧凑 UTF-8 JSON 编码计算，重复保存的值也重复计入：

| 对象 | 硬上限 | 超限行为 |
|---|---|---|
| Loop 轮数 | 100；配置仍为 `1..100` | 发布拒绝 `loop_max_iterations_invalid` |
| 一轮归档（carried 与所有成员状态合计）、一组最终 outputs | 各 1 MiB | Loop 失败，`output.reason = loop_state_size_exceeded`；不追加该轮归档或发布部分 outputs |
| 含复合节点的运行的完整 `node_states`（顶层成员、所有容器归档、重复输出全部计入） | 8 MiB，其中预留 64 KiB 给失败记录 | 普通状态提交不得超过 `8 MiB - 64 KiB`；超限使运行失败，`output.reason = workflow_state_size_exceeded` |

100 轮沿用既有 Loop 边界；单轮与最终 outputs 各 1 MiB 为文本与结构化产物留出空间。模型 X 每次成员状态
提交都会整行 CAS 重写 `node_states`，归档与重复输出越多，写放大和 PostgreSQL WAL 成本越高，因此完整状态
初始取保守的 8 MiB。以后放宽上限兼容，收紧却会让已发布的图失败；调整上限须以落地顺序第 3 步的近上限
测量为依据。两层字节预算共同生效，因此达到 100 轮不保证仍可写入任意大小的每轮产物；需要更大产物时应传引用，
不能通过提高轮数绕过总预算。

预算检查发生在每次成员状态提交、轮次归档和输出提交之前，不能等整个容器完成后才检查；进入 Loop 时的
carried 初值也必须满足单轮预算。超限保留上一次已提交状态及既有归档，用预留空间记录容器、轮次、
实测字节数、上限和稳定失败原因，走既有有界资源收尾；不把超大值原样放进错误信息，不截断后记成功。
取消、失败记录与旧归档不因为体积检查被丢弃。此预算限制新引入的复合运行，不回写历史快照或历史运行。

**为什么不是新表**：desktop 的 scope 表服务于它自己的 SQLite 仓储接口（`start_loop_round` /
`settle_iteration_round` / `list_node_runs_in_scope`），Cloud 的执行聚合本来就是「一行 + 版本 CAS + 重算」，
按行内文档实现可以让轮次推进和运行终态在同一个事务里提交，恢复也不需要跨表重建；活动轮留在扁平表里还让
会话路由不必为容器新增一套查找。代价是查询能力弱、文档随轮数增长；本文同时限制轮数和序列化体积，
只在运行详情读取时展开，先接受这个代价。若将来需要按轮查询或更强的恢复能力，再单独决策模型 Y。

### D2：发布门禁接受 loop / loopExit，校验容器结构与配置形状

门禁按下面的规则拒绝，错误码沿用三要素风格（`code` / `nodeId` / `field`）。**运行时使用同一个校验器**，
所以能发布的图就是能跑的图（根决策 D2 的既有原则）。

**容器结构**：

| 规则 | 拒绝码 |
|---|---|
| loop 的 body 必须恰好有一个 `start`（D1 的有效容器指向该 loop） | `loop_body_start_count` |
| body 成员的有效容器必须等于该 loop id；`loopExit` 只能在 body 内 | `loop_member_container_mismatch` / `loop_exit_outside_body` |
| body 内不允许 `loop` / `iteration` / `subflow` / `output`（v1 不嵌套复合节点） | `loop_nested_composite_unsupported` / `loop_body_output_unsupported` |
| 跨容器连边拒绝（一端在 body、另一端在外层） | `loop_cross_container_edge` |
| `loopExit` 必须无出边 | `loop_exit_has_outgoing_edge` |
| body 必须从 body start 可达，图必须无环 | `loop_body_unreachable` / `graph_cycle` |

**配置形状**（`data.loopConfig`，除 `maxIterationsBehavior` 外全部必填）：

| 字段 | 规则 | 拒绝码 |
|---|---|---|
| `maxIterations` | 整数 1..100 | `loop_max_iterations_invalid` |
| `maxIterationsBehavior` | 缺省或 `fail` / `succeed` | `loop_max_iterations_behavior_invalid` |
| `variables[]` | 数组；`name` 合法且不重复；`valueType` 在变量类型表内 | `loop_variable_invalid` |
| `variables[].initial` | `{kind:"constant",value}` 的值必须匹配 `valueType`（file 类按 desktop 规则规范化）；`{kind:"variable",selector}` 的 selector 已声明 | `loop_initial_invalid` / `loop_selector_undeclared` |
| `variables[].feedback` | selector 已声明、指向本轮可见根 | `loop_feedback_invalid` / `loop_selector_undeclared` |
| `until` | `logic` 是 `and` / `or`；`conditions` 复用 condition 的比较校验（算子、selector 声明） | `loop_until_invalid` / `condition_*` |
| `outputs[]` | 数组；`name` 合法且不重复；selector 已声明 | `loop_output_invalid` / `loop_selector_undeclared` |

selector 的「已声明」沿用根决策的变量根声明表，并受 D3 的轮内可见性约束：Loop 可见的外层祖先与
`sys.*` / globals、本 body 节点声明的根、以及 `{loopId}.{variableName}` 形式的 carried 根。
body 内节点引用 carried 值时写 `[loopId, variableName]`；存在于全图的声明不等于在该作用域可见。

**兼容**：保存草稿不校验；已发布快照不重新过门禁；`schemaVersion` 不在本文新增（desktop 的 loop 图使用
顶层 schemaVersion 2，Cloud 的文档读取本来就接受，本文不改）。

### D3：轮次语义与 desktop 对齐

- **后测循环**：进入 Loop 先解析 carried 初值、开第 1 轮；首轮总会执行。
- **初值一次**：`initial.kind = constant` 用常量（按 `valueType` 规范化）；`kind = variable` 用外层 selector
  在进入时求值。求值失败、类型不匹配则在进入 Loop 时失败，不发起任何轮次。
- **轮内变量池**：Loop 可见的外层祖先与 `sys.*` / globals 的值 + `{loopId}.{variableName}` 的 carried 值 +
  本 body 节点声明的根。外层可见祖先是 Loop 在所属外层图中的传递前驱；这些值在进入 body 时已确定。
  body 内节点（包括 aggregator）的节点前驱**只在本 body 的图里计算**：读取本 body 产物时，producer 必须是
  消费节点在 body 图里的传递前驱；不能因另一个分支恰好先提交就获得读取权。外层值只能经上述可见祖先，或
  经 `{loopId}.{variableName}` 的 carried 根进入；`sys.*` / globals 与 carried 根无需 body 前驱边。
  不得直接选择另一个容器的成员、外层非祖先或其它轮的状态。aggregator 的非前驱候选在发布时以
  `aggregator_selector_not_upstream` 拒绝，不能留到运行期当成未赋值跳过；aggregator 后续决策必须使用同一边界。
  Loop 在轮末求值 feedback / until / outputs 时可读取本轮完成快照中的可见 body 根；未选中分支的根仍未赋值，
  不用其它轮的旧值补齐。body 内的 condition 也只按本轮已提交状态选分支。
- **轮末 feedback**：本轮所有节点结算后，从同一份完成快照解出全部 feedback selector，逐个按 `valueType`
  规范化；缺值或类型不符则 Loop 失败，不提交半组 feedback。赋值同时发生，交换两个变量不会受顺序影响。
- **until**：feedback 校验通过后，用本轮完成快照求值 `until`。至少有一个非空 case 时才可能命中；命中即成功
  （包括最后一轮）；`until` 为空或未命中则继续或耗尽。
- **耗尽**：`round == maxIterations` 且未命中时，`maxIterationsBehavior` 缺省 `fail`：Loop 失败，
  `output.reason = "loop_max_iterations_reached"`，`error` 带上限与 until 输入的简短观测（与 desktop 的
  `LimitReached` 文案同义）。`succeed`：按本轮完成快照解析 outputs，Loop 成功，`reason = "max_iterations"`。
  `succeed` 只忽略「没等到条件」，不忽略执行、取值或类型错误。
- **outputs**：只在成功时解析，selector 指向本轮完成快照的可见根；缺值即失败，不导出部分输出。
- **失败与取消**：body 内任一节点失败即 Loop 失败（原因取该节点）；运行取消时 Loop 记为 `cancelled`，
  本轮在途的 agent 会话按既有取消路径收尾。
- **重试**：Loop 自身不自动重试（定义类失败重试不会改变结果）；body 内 agent 仍按各自冻结的重试策略重试。

### D4：loopExit 只在被选中时早退，先定意图再收尾

- `loopExit` 是 body 内的终端节点：没有出边，只有入边。它没有配置，成功输出 `{}`。
- 一轮内，当投影把 `loopExit` 解析为就绪（即它的入边来自本轮已成功的分支）时，调度器**优先**派发它；
  这一轮剩下的未执行成员记为既有的 `inactive`，不再执行，也不伪造成功输出。该状态与 condition 未选中分支
  相同，随节点状态和轮次归档提交；只表示本轮被跳过，不改变其它轮的归档。
  串行调度下早退提交前没有其它在途节点；若 body 内 agent 会话仍在途，必须先按既有串行规则结算，
  不得把它作为未执行成员直接改为 `inactive`。
- 早退成功时：按本轮完成快照解析 outputs（不计算 feedback），Loop 成功，`reason = "loop_exit"`。
  outputs 缺值则失败（fail closed）。
- 迟到的 body 结果不能复活已结束的轮次或 Loop：轮次文档已终结时，后续提交按运行版本 CAS 变成空操作，
  与根决策不变量 3 一致。

### D5：运行视图与 API 形状

- `GET /workflows/{wid}/runs/{rid}` 的 `nodeStates` 原样带出 `loop` 字段；OpenAPI 里 `nodeStates` 是自由对象，
  不需要新增 schema 字段。前端运行详情在 Loop 节点卡片上展示轮次摘要（第几轮 / 上限 / 终态原因），
  展开后从 `loop.completed[index]` 按轮显示成员的名称、状态、输出、错误及会话入口；活动轮从顶层成员状态
  读取，最后一轮既已归档又留在顶层时按轮去重，不显示成两次执行。取消和体积超限失败同样保留可见的轮次与原因。
  `inactive` 仍以节点 `status: "inactive"` 经 API 返回，前端沿用现有未激活的淡化展示；它不是运行终态。
  既有前端已有该状态的类型与样式分支，无需新增状态兼容逻辑；不认识 `loop` 的节点状态读取方仍可忽略该字段，
  读取顶层保留的成员状态。
  其它未知状态沿用既有中性兜底样式，不能推断成成功或失败。
- 运行列表的 `nodeStates` 仍是摘要投影，不展开轮次，避免列表响应随轮数膨胀。

### D6：失败码

Loop 失败时 `output.reason` 取稳定错误码，`error` 是给人读的说明：

| 场景 | `reason` |
|---|---|
| body 节点失败 | 该节点自己的 reason（沿用既有传递） |
| 耗尽且策略为 fail | `loop_max_iterations_reached` |
| feedback / outputs / until 取值缺失 | `loop_value_unset` |
| 类型不匹配 | `loop_variable_type_mismatch` |
| loopExit 的 outputs 缺失 | `loop_exit_output_unset` |
| 轮次文档与配置不符（恢复时发现） | `loop_invalid_round` |
| 单轮状态或最终 outputs 超过 D1 的 1 MiB 预算 | `loop_state_size_exceeded` |
| 整体 `node_states` 超过 D1 的可用字节预算 | `workflow_state_size_exceeded` |

既有语义失败与 desktop 的 `NodeFailureKind` 对应；字节预算失败是 Cloud 模型 X 新增的容量边界。
Cloud 的公开契约是 `reason` 字符串而不是枚举（既有做法），不新增公开运行状态。

## 不变量

1. 一个 Loop 的轮数在 `1..maxIterations` 内，绝不发起第 N+1 轮。
2. feedback 只从一份完成快照同时赋值；缺值或类型错误不提交半组值。
3. 外层节点只能通过 Loop 的命名 outputs 读取轮内产物，不能引用 body 节点的根。
4. `loopExit` 只在被实际分支选中、且是本轮就绪节点时触发；它没有出边。
5. `maxIterationsBehavior=succeed` 只忽略未命中的终止条件，不掩盖执行、取值、类型或 outputs 错误。
6. 轮次推进只依据已提交的轮内状态；没有提交结果的节点永不被记为成功，迟到的提交不改变终态。
7. 发布门禁与运行时共用同一份结构与配置校验；保存草稿不受影响。
8. 成员只由归一后的有效容器决定；状态提交同时受轮数与字节预算限制，超限不会变成截断后的成功。
9. body 节点的前驱关系只在本 body 计算；外层非祖先、其它容器与其它轮的提交顺序不改变其可见根集合。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| **新建 `workflow_execution_scopes` / `workflow_node_runs` 表**（贴近 desktop 的仓储） | 查询与恢复能力更强，未来 iteration 并行轮次更自然；但要做迁移、仓储、事务与调度改造，并放弃单行 CAS 的简单性。当前 loop 串行、轮数有界，先不付这个成本 |
| **发布时把 loop 展开成固定轮次** | 调度简单；但 `until`、feedback 和 loopExit 都是运行时决策，展开要么丢弃语义，要么把条件求值搬进发布期，做不到 |
| **每个 Loop 或每轮创建一个嵌套 WorkflowRun** | 复用运行实体；但会引入父子运行的所有权、取消传播、Workspace 归属与状态聚合，远超 loop 本身，更适合留给 subflow 决策 |
| **把 loop 交给 desktop 的 Rust 引擎执行** | 语义天然一致；但根决策把跨机器执行排除在 desktop 范围外，#448 已选 Go 移植，且 Cloud 的 Workspace/会话边界不同 |
| **继续拒绝 loop，只补 UI 占位** | 改动最小；但用户要的是节点可用，占位会让作者以为在编辑一个能跑的工作流，和根决策「拒绝原因可定位」的初衷相反 |

## 本决策未解决的问题

- **嵌套复合节点**：本文与 desktop 一致地拒绝 body 内的 loop / iteration / subflow；将来要支持需要单独决策。
- **loop body 内的 HITL**：`human` 节点和 `agentConfig.interactive` 仍按 [human 节点后续决策](20261021-human-node-run-input.md)
  的契约推进；本文不实现，
  门禁继续拒绝 `interactive: true`（既有行为）。
- **iteration / aggregator**：共用轮次与变量池模型，但语义不同，分别由后续决策承接。
- **运行中途 resume（从失败轮重开）**：继承重试决策 D7，resume 仍不可用；本文只保证恢复从已提交轮次继续。
- **按轮查询/分页**：轮次文档只在运行详情整体读取；如出现大轮数需求再考虑拆表。

## 落地顺序

1. **ADR 评审通过**（本文改 `approved`）并登记核心用例：while-loop 的结构、feedback 同时赋值、耗尽两策略、
   outputs、loopExit、恢复与体积超限稳定失败且不截断；无直接证据的义务标 `Missing`。
2. **门禁与校验**：`loop` / `loopExit` 进入可执行集合，容器结构与配置校验落地，带表驱动测试。
   完成条件：合法 loop 图可发布；上面每一条拒绝都有直接测试。
3. **调度器轮次引擎**：轮次初始化、轮内投影、feedback/until/outputs、耗尽与 loopExit、取消与恢复，
   带 PostgreSQL 集成测试与并发/恢复测试。完成条件包括每次提交前执行字节预算检查，超限稳定失败且保留已提交归档；
   并在真实 PostgreSQL 上给出接近 `8 MiB - 64 KiB` 可用上限时整行 CAS 提交的耗时与 WAL 测量，
   记录实际状态字节数、提交次数、数据库与测量条件。测量结果写在实现运行里，作为以后调整上限的依据。
4. **编辑器**：loop 检查器的变量/feedback/until/outputs、`loopExit` 节点与连线路由、desktop 图导入导出。
5. **真实栈端到端**：发布 → 启动 → 多轮 → 三种终态 → 运行视图；由落地运行把直接证据登记进核心用例，并将 ADR 转为 `implemented`。

第 2、3 步不依赖编辑器，可先合并；第 4 步只影响新图的作者体验，不改变冻结快照的执行；第 5 步完成前
不把 ADR 标记为 `implemented`。
