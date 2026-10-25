# Iteration 的 foreach 执行、作用域与按轮账本

依据[Cloud iteration 执行契约](../../../decisions/cloud/workflow/20261025-iteration-foreach-execution.md)
（`approved`，D1–D6）。按业务风险登记发布门禁、轮次执行、持久化、取消与容量边界。
下列义务尚无已核对的 Cloud 直接证据，均标为 `Missing`；desktop 参考实现和未合并实现不能替代这些证据。
调度器共同义务见[调度串行与恢复](scheduler-serialism-and-recovery.md)。

## The Publish Gate Uses One Membership And A Deterministic Iterator Source

- 风险：两种容器键产生不同成员集合；并行兄弟节点偶然先提交值，被用作不确定的迭代源。
- 前置状态：图分别使用 `parentId`、非空 `data.containerId`、二者冲突或空 `containerId`；迭代源分别
  指向传递前驱、外层非前驱、自身、body 成员、其他容器成员及已声明 globals / `sys.*`。
- 触发：保存草稿、发布图，并在不同调度顺序下启动合法图。
- 必须成立：非空 `containerId` 优先，其余回落 `parentId`，所有读取共用同一成员集合且不回写草稿；
  来源根必须已声明，其节点必须是 iteration 在外层图上的传递前驱，globals / `sys.*` 仅豁免前驱检查。
  非法来源返回 `iteration_iterator_undeclared` 并定位节点和字段；D2 其他配置与区域错误也明确拒绝。
- 禁止结果：外层非前驱因为恰好有状态就被放行；成员或自身提供迭代数组；发布自动截断配置或改写草稿。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 成员解析在缺失、空值与冲突时一致 | 门禁及调度器使用同一组图的归属断言 | Missing |
| 传递前驱与已声明环境根合法，其他来源按命名错误拒绝 | 发布门禁表驱动测试，含非前驱已经有值的反例 | Missing |
| D2 九个拒绝码定位节点与字段，合法草稿仍可保存 | 配置 / 区域门禁测试与 PostgreSQL 保存 / 发布测试 | Missing |

- 决策依据：D1、D2，不变量 1、10。

## An Empty Iterator Succeeds Without Starting Any Member

- 风险：空数组误报无入口失败，或创建一次空轮 / Agent 会话。
- 前置状态：合法 iteration 区域，迭代源为 `[]`。
- 触发：执行至该 iteration。
- 必须成立：立即成功，`output=[]`、`entries=[]`、`failed_count=0`；持久化的 `items`、`ledger`、
  `output`、`completed` 均为空数组，`round=0`；不创建任何成员执行或归档。
- 禁止结果：启动成员、生成第 0 轮账本条目、把数组编码为 null 或以缺收集值为由失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空数组输出与持久化形状正确，零成员执行 | PostgreSQL 调度器测试，检查 node_states、执行工作项与会话计数 | Missing |

- 决策依据：D3–D5，不变量 2。

## An Oversized Iterator Fails Before The First Round

- 风险：先执行到上限才失败，或截断输入后以成功结束。
- 前置状态：合法 `maxIterations=M`，来源分别为长度 M、M+1 的数组；配置缺省、100、101 或非法类型。
- 触发：发布配置并启动已发布的合法图。
- 必须成立：缺省 50，配置只能为 `1..100` 整数；配置非法以 `iteration_max_iterations_invalid` 拒绝。
  N>M 在启动边界以 `iteration_over_limit` 失败，错误含 N、M 和调整指引，任何成员均未启动。
- 禁止结果：处理前 M 项后成功或失败；`continue` 吸收项数超限；自动把 101 改成 100。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 缺省、硬上限与非法配置在发布时区分 | 门禁边界测试 | Missing |
| N=M 正常运行，N=M+1 在两种策略下都零轮失败 | PostgreSQL 测试，检查账本 / 归档为空且没有成员工作项 | Missing |

- 决策依据：D2、D6，不变量 5。

## Every Round Sees Only Its Own Item Index And Regional Predecessors

- 风险：本轮读到上一轮的变量、其他容器成员或非祖先产物；未选中的分支被当成执行成功。
- 前置状态：至少两项的迭代源，body 含 condition、两条分支和 aggregator；外层含可见祖先及并行非祖先。
- 触发：两轮选择不同分支；区域内外分别引用 item/index、成员根、外层来源及容器最终暴露根。
- 必须成立：第 r 轮只有 `items[r]` / r；body 节点的传递前驱只在本区域计算，aggregator 对区域内候选
  检查传递前驱，外部只允许 iteration 可见祖先、globals / `sys.*` 及本轮 item/index。item/index 不向
  外层暴露，最终三个暴露根不注入 body；未选中成员写为既有 `inactive`，不输出值，不创建会话。
- 禁止结果：外层非祖先、其他容器或上一轮状态成为本轮前驱 / 值来源；已运行成员被改为 inactive；
  外层直接引用 item/index 或 body 根。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 区域前驱与显式导入边界在门禁和取值时一致 | 门禁 + 变量池测试，包含 body aggregator 与外层非祖先反例 | Missing |
| 两轮绑定独立，condition 选择与 inactive 按轮归档 | PostgreSQL 测试，逐轮检查顶层 / completed 状态、输出及会话计数 | Missing |
| item/index 轮内私有、外层只读最终暴露根 | 发布可见性测试与真实运行读回 | Missing |

- 决策依据：D2、D3、D5，不变量 3、10、11。

## A Collect Target Must Have Succeeded In The Current Round

- 风险：收集目标本轮没有运行，却收集到上一轮残留值或把缺值当成成功的 null。
- 前置状态：第 0 轮成功运行收集目标，第 1 轮 condition 绕开该目标；两种 errorStrategy 分别运行。
- 触发：第 1 轮 body 结算并收集输出。
- 必须成立：只收集本轮 `succeeded` 目标的根；未运行目标使本轮失败，说明为
  `collect target did not run this round`，账本失败条目 `output=null`，不进入成功 output 数组。
  fail 停止，continue 记录失败后推进下一轮。
- 禁止结果：读取第 0 轮的值、为 inactive 目标合成输出、无错误地跳过该输入项。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 相反分支选择时收集缺失稳定记失败，策略边界正确 | PostgreSQL 两轮 / 三轮测试，检查每轮状态与账本及后续是否启动 | Missing |

- 决策依据：D4、D6，不变量 4、6。

## Fail Stops At The First Failed Round While Continue Keeps Its Ledger

- 风险：continue 掩盖容器自身失败，或 fail 丢失失败轮证据 / 继续启动后续项。
- 前置状态：三项数组，第 1 轮成员失败；分别采用缺省、显式 fail、continue。
- 触发：执行失败轮；另以非数组来源、暴露写入失败和字节超限触发容器自身失败。
- 必须成立：fail（含缺省）先提交失败轮账本和归档，再使容器与 run 失败，后续项不执行；continue 记录
  失败轮后执行后续项，最终成功并保留错误。容器自身失败在两种策略下都终止 run，不伪装成可吸收的轮失败。
- 禁止结果：自动重跑失败轮；遗漏失败账本；以 continue 吸收非数组、暴露写入或预算失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| fail 停止、continue 完成，失败轮两者均留档且无自动重跑 | PostgreSQL 成员失败测试，检查每轮派发次数与运行终态 | Missing |
| 容器自身失败不被策略吸收 | 非数组、暴露提交失败与容量失败注入测试 | Missing |

- 决策依据：D4、D6，不变量 6、7。

## Exposed Variables Keep Their Types And Input Order

- 风险：切换策略改变下游变量类型，或 output / entries 因完成顺序、假值而错位。
- 前置状态：成功、失败混合输入，成功值包含 null、false、空字符串及对象；两种策略有合法完成的运行。
- 触发：读取已完成 iteration 的三个根与中途失败运行的内部账本。
- 必须成立：成功终态 output 为仅成功项的有序数组，entries 为按输入位置排列的信封数组，failed_count
  为失败条目数；信封四个键齐全，成功 error=null，失败 output=null，以 status 区分成功 null 与失败。
  fail 提前结束的内部账本只含已结算前缀，不伪造后续条目。
- 禁止结果：把成功 null / false / 空串过滤掉；把失败项混进 output；按策略切换 array 与其他类型。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 类型固定、假值保真、过滤与完整账本的顺序一致 | PostgreSQL 输出读回及变量根声明测试 | Missing |
| fail 终态账本为连续已结算前缀 | 首轮 / 中间轮失败测试，断言无未执行条目 | Missing |

- 决策依据：D4、D5，不变量 4。

## Recovery Uses One Committed Round Document Without Replaying Settled Work

- 风险：恢复重新读取迭代源或清空账本，从第 0 轮重跑；半次 CAS 形成轮号与归档不一致。
- 前置状态：第 0 轮已归档，第 1 轮分别处于未派发、已有结果、在途会话；输入来源之后发生变化。
- 触发：换调度器实例接手，重复 / 并发 pass，轮结算前后模拟 CAS 失效；另加载不一致的轮次文档。
- 必须成立：只读已冻结 items 与 D5 wire 字段；账本、派生值、归档、清空顶层及轮号推进原子提交。
  已结算轮和活动轮已提交结果不重派，在途会话持有到自身结果；无下一轮时保留最后启动的 round，
  空数组 round=0。非法轮次文档以 `iteration_invalid_round` 失败，不能按“未开始”重新执行。
- 禁止结果：改变 items、重复 append 同轮账本、归档缺项、CAS 失败仍推进、换副本重建在途会话。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| wire 编解码与 round / ledger / completed 的关系符合 D5 | 持久化 round-trip 测试，含运行中、成功、失败、取消和空数组 | Missing |
| 接手不重跑已结算轮 / 已提交成员或在途会话 | PostgreSQL 恢复及并发测试，检查状态版本与工作项 / 会话计数 | Missing |
| CAS 原子性与非法轮次稳定失败 | 提交故障测试与不一致轮次文档测试 | Missing |

- 决策依据：D5，不变量 7、11，根决策 D1 / D4。

## Cancellation Preserves History And Never Starts Another Round

- 风险：取消只改容器状态但下一轮仍被派发；归档被清空，取消不能定位活动项。
- 前置状态：至少两轮，取消分别发生在活动轮有在途会话及两轮交界处。
- 触发：请求取消并重复推进调度器，等待既有有界收尾。
- 必须成立：当前轮按既有路径停止，会话和 Workspace 按运行取消路径收尾；容器最终为 cancelled，
  已提交 completed / ledger 不变，活动轮状态保留可读；请求被提交后不启动新轮。
- 禁止结果：新建后续轮会话；伪造未完成轮的成功账本；取消后丢掉已结算记录或无界等待。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 轮中与轮边界取消均不启动新轮，历史可读且有界收尾 | PostgreSQL 取消竞争测试与真实栈取消 / 资源读回 | Missing |

- 决策依据：D5，不变量 7。

## Byte Limits Fail Stably Without Truncating Values Or Losing History

- 风险：仅限制轮数仍写爆运行行；错误说明复制超大值挤掉失败预留；continue 把容量失败当成成功。
- 前置状态：紧凑 UTF-8 JSON 编码的输入项、collect 值、单轮归档分别在 1 MiB 边界内外；完整
  node_states 接近 `8 MiB - 64 KiB`，包括输入、当前成员、归档、账本与输出的重复值。
- 触发：启动检查、成员提交、轮结算和最终暴露提交；两种策略分别触发超限。
- 必须成立：每项输入 / collect / 轮归档各不超过 1 MiB，普通提交不使用 64 KiB 失败预留，总量不超过
  8 MiB。单项超限以 `iteration_state_size_exceeded`、整体超限以 `workflow_state_size_exceeded` 终止，
  保留上次已提交状态，错误只含有界的容器 / 轮次 / 字节数 / 上限，既有账本与取消 / 失败记录不丢失。
- 禁止结果：逻辑去重后计费、漏算 UTF-8 或重复值、截断值 / 删除历史后记成功、continue 吸收预算失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 每个检查点与两类字节边界按实际编码生效 | 预算单元测试 + PostgreSQL 边界测试，含多字节文本与重复输出 | Missing |
| 超限保留既有提交，并在失败预留内稳定结算 | PostgreSQL 失败注入测试，检查 run / node 状态、账本、诊断大小与无后续工作项 | Missing |
| 接近总预算时整行 CAS 的耗时与 WAL 成本可复核 | 实现运行中的可重现测量，记录体积、负载、方法与结果，可和 loop 共用 | Missing |

- 决策依据：D5、D6，不变量 9，落地顺序第 3 步。

## RunDetails Show Each Round Once And Preserve Member Attribution

- 风险：最后一轮同时来自顶层和归档而显示两次；continue 失败、inactive 或取消活动轮失去归因。
- 前置状态：成功、fail、continue、取消与预算失败运行，含已归档轮和未结算活动轮。
- 触发：读取运行详情和运行列表，展开 iteration 的逐项账本及逐轮成员。
- 必须成立：按 `(iteration id, index)` 唯一展示轮次；`completed[index=round]` 存在时只展示归档，
  否则顶层成员显示为未结算活动轮。能定位 item / index、成员输出、错误、会话和 inactive“未到达”；
  列表只给摘要，旧读取方仍能读取既有状态字段并忽略新增 iteration 文档。
- 禁止结果：显示第 N+1 轮、把最后一轮算两遍、把未选中成员显示为成功、丢掉取消或容量失败的活动轮。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 活动 / 最终轮去重，成员和错误归因完整 | 前端投影 / 详情测试与真实栈逐轮展开读回 | Missing |
| API 兼容且列表不展开归档 | API 响应与前端解析测试 | Missing |

- 决策依据：D3、D5，不变量 11。
