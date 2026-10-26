# Loop 轮次执行、早退与容量边界

依据 [loop 与 loopExit 执行决策](../../../decisions/cloud/workflow/20261025-loop-and-loop-exit-execution.md)
（`approved`）。本文登记 Cloud 的验证义务；`approved` 表示契约已确定，不表示执行能力已验收。
证据只登记能直接判定相应义务的已合并实现测试或真实栈记录；未合并代码和普通 DAG 的间接测试不作为 Loop 的证据。
以下尚无直接证据的义务均为 `Missing`。

## Only Structurally Valid Loop Bodies Can Be Published

- 风险：草稿中的容器歧义、跨容器边或非法配置进入冻结快照，运行时执行了不同的成员集合，或无法定位失败原因。
- 前置状态：草稿包含 loop、body start、普通成员与 loopExit；成员可能通过 `data.containerId` 或 `parentId` 表达归属。
- 触发：保存草稿、发布合法或非法图，以及运行时校验冻结图。
- 必须成立：保存未完成草稿仍被允许；发布与运行时共用校验器，合法图可通过；有效容器按非空 `containerId`
  优先、否则回落 `parentId`，冲突时不形成两份成员集合且不回写草稿。body 恰有一个 start、成员可达且图无环，
  不含嵌套复合节点或 output，不允许跨容器边，loopExit 只能在 body 内且没有出边。轮数、变量、initial、feedback、
  until 与 outputs 均满足 D2，拒绝包含稳定 `code`、`nodeId`、`field`。
- 禁止结果：发布和执行解析出不同成员；静默修图；越界轮数、缺失 feedback、未声明 selector 被接受；错误只有泛化文案。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 两种成员键及冲突优先级在门禁与执行一致，草稿不被改写 | 合法图的发布与执行测试，比较保存前后的原图与实际 body 集合 | Missing |
| 非法结构被逐类拒绝，错误可定位 | 结构门禁测试覆盖 D2 的 start 数量、成员归属、嵌套、output、跨容器、loopExit 出边、不可达与环 | Missing |
| 配置与 selector 校验一致，合法图可发布、非法草稿仍可保存 | 发布与运行时校验测试覆盖 D2 配置边界，并核对拒绝三要素与草稿保存 | Missing |

- 决策依据：D1 成员归一、D2，不变量 7、8。

## Body Selectors Respect Local Predecessors And Visible Outer Ancestors

- 风险：body 节点读取并行兄弟、外层非祖先或其它容器的值，同一张图因提交顺序不同而得到不同结果。
- 前置状态：Loop 外有可见祖先、非祖先节点与另一个容器；body 内有分支和 aggregator，并声明 carried 变量。
- 触发：发布各类 selector 配置；改变兄弟分支的执行顺序；进入下一轮。
- 必须成立：body 内节点（含 aggregator）的 producer 前驱只在本 body 图里计算；本 body 产物只能由其传递后继读取。
  外层仅导入 Loop 在外层图中的可见祖先，以及 `sys.*` / globals；carried 经 `{loopId}.{variableName}` 读取，
  无需 body 前驱边。不得直接读取其它容器、外层非祖先或其它轮；aggregator 非前驱候选在发布时以
  `aggregator_selector_not_upstream` 拒绝。Loop 轮末求值可读取本轮完成快照中可见的 body 根，未选中分支仍未赋值。
- 禁止结果：把非前驱候选留到运行期跳过；全图有声明就允许跨区域读取；用上一轮的同名根补齐未赋值分支。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 合法 body 前驱、外层可见祖先、carried 与环境根可读取 | 同一合法图的发布与多轮执行，分别断言各类根的取值 | Missing |
| 其它容器、外层非祖先与 body 非前驱被拒 | 发布门禁测试含 aggregator 并行兄弟候选，并断言 `aggregator_selector_not_upstream` | Missing |
| 每轮变量池独立，未赋值分支不继承旧值 | 两轮选择不同分支，断言旧轮产物不参与本轮解析 | Missing |

- 决策依据：D2 selector 声明、D3 轮内变量池，不变量 3、9；aggregator 后续决策的前驱约束。

## Feedback Uses One Completed Snapshot And Initial Values Are Evaluated Once

- 风险：逐个更新 carried 使交换变量依赖声明顺序；重新求 initial 覆盖前一轮 feedback；某个值无效时只提交半组。
- 前置状态：至少两个 carried 变量，其 feedback 相互引用；initial 含常量与外层 selector。
- 触发：进入 Loop、完成一轮并进入下一轮；initial 或某项 feedback 缺值、类型不符。
- 必须成立：initial 只在进入时求值与规范化一次；失败时不开任何轮次。轮末先从同一完成快照解析并规范化全部
  feedback，全部有效后同时赋值；交换两个变量得到旧值互换，声明顺序不影响结果。缺值或类型错误稳定失败，
  保留已提交事实，不提交半组 carried，也不开始下一轮。
- 禁止结果：initial 每轮重算；第二个变量读取刚更新的第一个值；类型失败后仍推进轮次；无效输入被当成空值成功。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 初值只求一次、首轮与后续轮分别使用 initial 和已提交 feedback | 多轮 PostgreSQL 测试，读取每轮 carried 并核对求值边界 | Missing |
| 交换同时赋值，缺值或类型错误不会提交半组 | 交换及失败路径测试，断言 carried、轮号和失败码 `loop_value_unset` / `loop_variable_type_mismatch` | Missing |

- 决策依据：D3 初值一次、轮末 feedback、D6，不变量 2、6。

## Until Is Tested After The Body And Before Starting Another Round

- 风险：把后测循环执行成前测循环；条件命中时多执行一轮；最后一轮命中被误判为耗尽。
- 前置状态：Loop 配置有效，首轮或最后允许轮的本轮快照能使命中条件；另有空 until 和未命中配置。
- 触发：进入 Loop；body 结算、feedback 校验完成后的轮末推进。
- 必须成立：首轮总会执行；feedback 校验通过后，在本轮完成快照上求值 until；命中（含最后一轮）即成功，
  不再开下一轮。空 until 不命中；轮号不超过 `maxIterations`，未提交的 body 结果不参与判定。
- 禁止结果：跳过首轮；先递增轮次再判条件；命中却报 `loop_max_iterations_reached`；产生第 N+1 轮。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 首轮必执行，轮末命中成功且最后一轮命中优先于耗尽 | 轮次推进测试核对成员执行次数、归档和 Loop 终态 | Missing |
| 空或未命中条件受轮数边界约束，只读已提交快照 | 空 until、未命中及未提交结果测试，核对实际派发轮号 | Missing |

- 决策依据：D3 后测循环、until，不变量 1、6。

## Exhausted Loops Follow The Frozen Fail Or Succeed Policy

- 风险：耗尽时静默成功，或 `succeed` 把执行、取值与类型错误一并掩盖。
- 前置状态：until 在允许轮数内未命中；快照分别未设策略、设为 `fail` 或 `succeed`。
- 触发：完成最后允许轮；另在 `succeed` 图中注入 body、feedback 或 outputs 错误。
- 必须成立：缺省和 `fail` 以 `loop_max_iterations_reached` 失败；`succeed` 解析最后一轮 outputs 并成功，
  `reason = "max_iterations"`。两种策略都不启动额外轮次，保留最后一轮状态与归档；`succeed` 只忽略 until 未命中。
- 禁止结果：默认成功；使用上一轮输出；`succeed` 掩盖其它失败；超过快照冻结的上限。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 缺省与显式 fail 稳定失败，succeed 导出最后轮并成功 | 两类耗尽的 PostgreSQL 测试，核对轮号、终态、输出与 reason | Missing |
| succeed 不吸收执行、取值和类型错误 | 同一策略下的各类失败测试，核对真实错误未被改成耗尽成功 | Missing |

- 决策依据：D3 耗尽、D6，不变量 1、5。

## Named Outputs Are Published Atomically From The Final Round

- 风险：外层直接读取 body 产物，或某项输出缺值时只发布一部分，从而把失败轮当成成功。
- 前置状态：Loop 声明多个命名 outputs；外层后继经 `{loopId}.{name}` 消费，另有直接选择 body 根的非法图。
- 触发：until 命中、耗尽 succeed 或 loopExit 成功；某个输出 selector 未赋值。
- 必须成立：成功输出全部取自对应的本轮完成快照，只以 Loop 的命名 outputs 对外公开；任一项缺值则失败且无部分
  outputs，loopExit 使用 `loop_exit_output_unset`，其它缺值使用 `loop_value_unset`。外层不能直接引用 body 根。
- 禁止结果：混用多轮输出；失败后可见半组成功 outputs；隐式声明或伪造 `{loopId}.output`；泄漏 body 成员根。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 三种成功路径均从本轮原子导出命名 outputs | PostgreSQL 执行测试，断言外层后继实际读取到的值与终轮一致 | Missing |
| 缺值不部分发布，外层直接读取 body 根被拒 | 输出缺值的状态提交测试与 selector 门禁测试，核对错误码与可见根 | Missing |

- 决策依据：D1 output 形状、D3 outputs、D4、D6，不变量 3、5。

## Only A Selected Loop Exit Can Skip Remaining Body Members

- 风险：未选中的 loopExit 触发早退；早退后仍派发成员或计算 feedback；未执行成员被伪造为成功。
- 前置状态：body 包含条件分支、loopExit、尚未执行的成员与命名 outputs；另有仍在途的 agent 会话。
- 触发：loopExit 所在分支被选中并就绪，或分支未选中；早退状态提交失败；终结后迟到的 body 回调到达。
- 必须成立：仅本轮实际选中且就绪的 loopExit 优先执行，成功输出 `{}`；先确定早退，再把本轮未执行成员提交为既有
  `inactive`。沿用串行约束，早退提交前结算在途会话；不运行剩余成员，不求 feedback，直接解析本轮 outputs，
  成功 reason 为 `loop_exit`。未选中的 loopExit 不影响正常轮次；提交或 outputs 失败均 fail closed；迟到结果受 CAS
  约束，不能改变终态。
- 禁止结果：把在途会话改为 inactive；反馈覆盖 carried 后再早退；未选中分支触发；提交失败仍宣布成功；迟到回调复活 Loop。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 选中才早退，跳过 feedback、剩余成员为 inactive 且不再派发 | 条件分支执行测试，使用若被求值就会失败的 feedback，并核对 outputs、归档和派发集合 | Missing |
| 在途会话先结算、早退提交失败不成功、迟到回调不改终态 | 真实 PostgreSQL / 执行侧故障测试，断言会话、版本与终态 | Missing |

- 决策依据：D4、D6，不变量 4、6。

## Recovery Continues From Committed Rounds Without Replaying Members

- 风险：重启丢失 carried、重复推进一轮，或把上一轮同节点 id 的回执结算成当前轮结果。
- 前置状态：运行行保存 completed 归档、活动轮 round/carried/成员状态；某个 agent 已派发但尚未提交结果。
- 触发：成员提交后或轮次切换边界崩溃，重启或另一副本接手；重放重复、迟到回执；恢复时轮次与配置不一致。
- 必须成立：只读已提交运行行，恢复 round/carried/归档及活动成员；轮次推进与归档受同一版本 CAS 保护，已提交成员
  不重派、已完成轮不重跑。下一轮只从空成员状态开始；终轮保留与归档一致。在途执行继续持有到自身结果或既有有界失败，
  未知不等于成功；重复或迟到回调不改变当前轮、归档和终态；无效轮次以 `loop_invalid_round` 失败。
- 禁止结果：依赖进程内轮号；重复归档；上一轮结果污染下一轮；恢复重置 carried；把恢复接手当成用户 resume。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| 重启与另一副本从已提交轮次继续，不重派已完成成员 | PostgreSQL 中间态及真实调度副本接手测试，核对轮号、归档与派发次数 | Missing |
| 并发推进与重复、迟到回执不重复归档或污染当前轮 | 并发 CAS 与跨轮回执重放测试，比较 node_states、版本和执行记录 | Missing |
| 在途执行仍由自身结果结算，无效轮次稳定失败 | 真实会话恢复及轮次不一致测试，核对持有边界与 `loop_invalid_round` | Missing |

- 决策依据：D1、D3、D4、D6，不变量 6；根决策 D1 条件②、D4。失败后的用户 resume 仍不在本决策范围内。

## State Budgets Fail Stably Without Truncating Committed History

- 风险：轮数虽有限但行内 JSON 无限增长；超限时丢归档、截断输出后记成功，或因错误本身过大而无法记录失败。
- 前置状态：含复合节点的运行；构造紧贴单轮、最终 outputs 及完整 node_states 上限的状态，含多字节文本与重复产物。
- 触发：初始化 carried、成员状态提交、轮次归档、最终 outputs 提交或取消；分别达到和超过可用上限。
- 必须成立：按实际紧凑 UTF-8 JSON 字节计量，重复保存重复计入。单轮归档（carried 与全部成员状态）及最终 outputs
  各至多 1 MiB，initial carried 同样受单轮预算约束；完整 node_states 至多 8 MiB，其中 64 KiB 留给失败记录，普通
  提交至多 `8 MiB - 64 KiB`。每次提交前检查，超限拒绝该次超大状态、保留上次已提交归档，以
  `loop_state_size_exceeded` 或 `workflow_state_size_exceeded` 稳定失败并走有界收尾；错误只记容器、轮次、实测大小、
  上限与原因，不复制超大值。失败和取消记录不得被丢弃，不能输出部分结果或截断成功。
- 禁止结果：按字符数或压缩后字节计量；直到容器完成才查预算；消耗失败预留；删除旧归档腾空间；大输出仍记 succeeded。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| initial、成员提交、归档与 outputs 均执行边界检查，UTF-8 与重复值计量准确 | 边界测试分别到达和超过 1 MiB、`8 MiB - 64 KiB`，包含多字节文本及多容器累计状态 | Missing |
| 超限保留已提交事实、记录稳定失败、资源有界收尾，不截断成功 | PostgreSQL 与真实 agent 大产物测试，比较拒绝前后归档、reason、错误大小、派发和资源终态 | Missing |
| 接近预算时仍能提交失败与取消记录 | 预留空间故障测试，断言最终序列化状态不超过 8 MiB，失败和取消原因可读取 | Missing |
| 有调整上限所需的整行 CAS 耗时与 WAL 基线 | 真实 PostgreSQL 在接近可用上限时重复提交，记录状态字节数、提交次数、耗时、WAL 和数据库条件；结果保存在实现运行 | Missing |

- 决策依据：D1 体积预算、D6，不变量 8，落地顺序第 3 步。性能测量用于评估后续上限调整，不替代正确性证据。

## Run Details Show Each Round And Inactive Member Once

- 风险：终轮同时归档并留在顶层造成重复执行展示；早退成员被显示成失败；读取方误以为 inactive 是新增运行状态。
- 前置状态：多轮运行有 completed 归档与顶层活动成员；终轮已归档仍保留顶层，另有早退、取消与容量失败运行。
- 触发：通过运行详情 API 读取，在前端展开 Loop 与成员；用不识别 loop 字段的已有节点状态读取方读取。
- 必须成立：API 的 nodeStates 返回 loop 文档，节点复用 `status: "inactive"`；现有前端沿用淡化的未激活样式，
  inactive 不是运行终态。Loop 摘要含轮号、上限与终态原因；每轮成员名称、状态、输出、错误与会话入口可查看；
  活动轮和保留的终轮按轮去重。既有节点状态读取方仍可忽略 loop 字段并读取顶层成员；列表仍用摘要，不展开全部归档。
- 禁止结果：同一轮成员显示成两次执行；未执行成员被显示成功或失败；取消与超限原因丢失；列表响应携带完整轮次归档。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| API 与列表保持相应形状，inactive 无需新状态兼容，既有节点状态读取方仍可读 | 运行详情和列表契约测试及既有节点状态读取方兼容测试 | Missing |
| 多轮、终轮去重及早退、取消、超限原因在视图可定位 | 浏览器或组件交互测试，核对每轮成员、错误、输出和会话入口 | Missing |

- 决策依据：D1 状态与归档、D4、D5。

## Cancellation Settles Active Loop Work Before The Run Becomes Terminal

- 风险：取消后仍开始新轮或新会话，提前写终态导致资源泄漏，或取消清理抹掉已完成轮。
- 前置状态：Loop 正在轮内执行 agent，已存在完成轮归档与可能的重试状态。
- 触发：请求取消运行，并与结果回调、轮次推进竞争。
- 必须成立：沿用既有有界取消和资源收尾路径，Loop 标记 cancelled；不再开始新轮或派发新成员，会话与资源完成有界
  收尾后运行才终态；保留已提交轮次、成员状态与取消原因。迟到结果不得复活运行或转成成功。
- 禁止结果：取消请求直接伪装成清理完成；终态后继续派发；删除已提交归档；把 cancelled 改成 succeeded。
- 验证义务与证据：

| 义务 | 所需直接证据 | 证据状态 |
| --- | --- | --- |
| Loop 取消停止派发并有界清理，归档与原因保留 | 真实执行侧运行中取消，核对会话、资源终态、nodeStates 与后续派发集合 | Missing |
| 取消与轮次推进、结果回调竞争不复活运行 | PostgreSQL 并发取消与迟到回执测试，断言允许终态及版本变化 | Missing |

- 决策依据：D1 超限/取消记录保留、D3 失败与取消、D5；根决策 D4、D6。
