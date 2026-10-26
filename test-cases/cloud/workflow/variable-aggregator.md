# 变量聚合的发布门禁与有序透传

依据[变量聚合执行决策](../../../decisions/cloud/workflow/20261025-variable-aggregator-execution.md)
（`approved`，D1–D4）。用例按发布确定性、取值语义与容器隔离组织；每项义务独立登记直接证据，
下列义务均为 `Missing`。desktop 参考实现与未合并的候选代码不作为 Cloud 已履行这些义务的证据。
运行整体的串行与恢复约束另见[调度串行与恢复](scheduler-serialism-and-recovery.md)。

## An Assigned Candidate Is Forwarded Without Transformation

- 风险：聚合节点被当作结构占位，执行成功却没有 `output`；或把值转成字符串、合并多个根，改变下游看到的产物。
- 前置状态：图包含合法的非空有序候选列表；候选 producer 是 aggregator 的图上传递前驱，已提交的状态含所选根。
- 触发：发布图并运行，调度器推进到 aggregator 就绪。
- 必须成立：首个已赋值候选的值原样写为节点状态的 `output.output`，只声明 `{aggregatorId}.output` 一个根；
  节点成功后下游从该根读到同一 JSON 值。聚合节点自身不创建 Agent 会话或执行工作项。
- 禁止结果：只推进状态而不写产物；字符串化、包裹或合并候选值；多写候选根；为聚合创建 Agent 会话。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 首个已赋值根原样成为唯一输出，下游读到同一 JSON 值 | 真实 PostgreSQL 调度器用例：前驱提交结构化值，推进后读回 aggregator 与下游结果，并断言没有聚合节点的会话/执行工作项 | Missing |
| 发布到运行详情的真实链路保留同一值和成功状态 | 真实栈：发布、启动、经过前驱与聚合、读取运行详情，核对冻结配置与最终产物 | Missing |

- 决策依据：D1、D2、D4，不变量 4、6。

## An Unassigned Candidate Is Skipped After Its Branch Settles

- 风险：聚合等待未被选中的分支永远赋值；或缺键被提前写成空值，使后面的有效候选无法胜出。
- 前置状态：两个候选均已声明且是图上传递前驱；前序分支已结算，排在前面的候选未执行或其根缺键，后一个候选有值。
- 触发：调度器按既有就绪规则推进到 aggregator；分别使用未选中分支和已成功但根缺失的前驱。
- 必须成立：按声明顺序跳过缺键，输出后一个已赋值候选；未选中分支不造成额外等待，聚合无需读取 Condition 的分支决策。
- 禁止结果：为缺键补 `null` 并选中它；把跳过的分支当作在途节点；越过既有就绪门槛提前求值。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未选中分支没有根时，后一个已赋值候选胜出且运行可以结束 | 真实 PostgreSQL：Condition 选中后一个候选的分支，汇合后读回输出与终态；前一个候选没有提交值 | Missing |
| 成功前驱的声明根缺键时同样跳过，不伪造空值 | 真实 PostgreSQL：前驱成功但没有写所选根，后一个根存在，断言后者输出 | Missing |

- 决策依据：D1 的图上前驱与实际分支区分、D2，不变量 1、2。

## Null False And Empty Values Count As Assigned

- 风险：用真值判断代替键存在判断，明确产出的空值被当作缺失，悄悄改选后一个候选。
- 前置状态：首个候选的根存在，值分别为 `null`、`false`、`0`、`""`、`[]`、`{}`；后一个候选也存在且值不同。
- 触发：对这些值分别执行同一有序候选配置。
- 必须成立：首个候选始终胜出，JSON 值与类型均保留；显式 `null` 与根缺键可区分，只有缺键才扫描后续候选。
- 禁止结果：空值触发 fallback；`null` 输出键被省略；空数组或空对象被字符串化；零与 `false` 被互换。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 键存在的六类假值/空值均胜出，键缺失才跳过 | 直接取值测试与真实 PostgreSQL 调度器用例：逐类对照首个根缺键，后一个候选设为不同哨兵值，断言持久化输出的键、值与类型 | Missing |

- 决策依据：D2，不变量 2。

## No Assigned Candidate Fails The Run With A Stable Reason

- 风险：全部候选缺键仍被记为成功；或聚合自动重试、无限等待，掩盖缺失的分支产物。
- 前置状态：候选配置合法，前序分支已结算且 aggregator 已就绪；所有候选的根均未赋值。
- 触发：调度器推进聚合节点，并在结果提交后再次推进。
- 必须成立：聚合节点失败，`output.reason = "aggregator_no_match"`，失败信息可定位扫描过的候选；
  运行走既有失败结算路径，不派发聚合的下游，不为聚合自动重试或创建会话；重复推进不能把终态改为成功。
- 禁止结果：空对象成功、用 `null` 伪造命中、永久等待、启动新尝试，或下游继续消费不存在的聚合产物。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 全部缺键形成节点失败和稳定 `aggregator_no_match`，可定位候选 | 真实 PostgreSQL：合法已发布图的全部候选缺键，读回失败节点的 reason、error 与运行失败结算 | Missing |
| 失败后不执行下游、不重试，重复推进不改写终态 | 同一真实 PostgreSQL 场景重复推进，断言无下游派发、无聚合会话/重试工作项，节点与运行终态保持失败 | Missing |

- 决策依据：D2，不变量 3、6。

## Declaration Order Wins When Multiple Candidates Are Assigned

- 风险：以完成先后、节点 id 或图存储顺序选择候选；或沿用多值冲突失败，使作者声明的优先级失效。
- 前置状态：两个不同值的候选均已赋值且均是传递前驱；候选列表为 `[writer.output, reviewer.output]`，
  producer 的 id 排序、节点数组顺序与提交顺序分别可以与声明顺序不同。
- 触发：发布并运行；仅交换候选列表顺序后再次发布运行。
- 必须成立：原配置始终输出 writer 的值，交换候选后输出 reviewer 的值；多个已赋值候选不冲突。
  编辑器保存、导入与发布冻结都保持作者声明的顺序。
- 禁止结果：按最快完成或字典序取值；合并两个值；多值时报冲突；检查器自动排序。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 两个候选同时赋值时仅声明顺序决定输出，不受提交/id/图数组顺序影响 | 真实 PostgreSQL：两前驱都已提交且值不同，对照候选交换与不同状态/图排列，断言输出始终跟随候选列表 | Missing |
| 编辑、导入、保存与发布冻结不重排候选 | 前端往返用例与真实发布读回：候选顺序刻意不同于节点 id 顺序，核对检查器配置、草稿和快照 | Missing |

- 决策依据：D2、D4，继承与修改中的 desktop D6 修改项，不变量 1、4。

## Publication Rejects Candidates That Are Not Upstream

- 风险：已声明的兄弟分支根被放行，是否先提交决定聚合是否命中；检查节点存在却不检查前驱，产生不确定结果。
- 前置状态：图与候选的形状、根声明均合法；分别构造直接前驱、经过中间节点的传递前驱、并行兄弟、后继、
  aggregator 自身，以及 `sys.*` 与 globals 候选。
- 触发：发布上述配置。
- 必须成立：直接和传递前驱可选；兄弟、后继和自身被 `aggregator_selector_not_upstream` 拒绝，带聚合节点
  `nodeId` 和候选所在 `field`；图中声明了该根不构成前驱证明。已声明的 `sys.*` 与 globals 豁免前驱校验，
  未声明的环境根仍被 `aggregator_selector_undeclared` 拒绝。
- 禁止结果：兄弟根因存在于声明表而被放行；只接受直接入邻居而拒绝传递前驱；对环境根要求图连边；以运行时缺键跳过代替发布拒绝。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 已声明的非前驱拒绝，直接/传递前驱允许 | 发布门禁直接测试与真实发布读回：兄弟、后继、自身被命名错误拒绝且不产生发布快照，合法前驱成功冻结 | Missing |
| `sys.*` / globals 只免前驱校验，不免根声明校验 | 发布门禁测试：无 producer 连边时已声明环境根通过，未声明环境根仍按声明错误拒绝 | Missing |

- 决策依据：D1，不变量 7，替代方案中拒绝运行期跳过非前驱的取舍。

## A Loop Body Aggregator Uses Only Its Visible Current Round Scope

- 风险：全局扁平状态让 body 读到其他轮、其他容器或外层兄弟分支；对容器外值全部放行或全部拒绝，破坏有效的 carried/祖先输入。
- 前置状态：Loop body 内有 aggregator；本 body 的图上前驱、外层 Loop 的传递祖先、所属 Loop 的 carried 根
  均已声明；同时存在本 body 非前驱、其他 body 成员、外层非祖先和已完成轮归档。同一图还含外层 aggregator。
- 触发：分别发布合法与越界候选；执行两轮，第一轮本 body 前驱有值，第二轮该前驱未赋值；用 `containerId`、
  `parentId` 回落和两者冲突的图核对有效容器规则。
- 必须成立：body 内前驱仅在本 body 图中计算；外层可见值只来自进入所属 Loop 时已可见的祖先、所属 Loop 的
  carried 根与环境根。其他 body、外层非祖先、本 body 非前驱以及外层直接选 body 成员都在发布时被
  `aggregator_selector_not_upstream` 拒绝；外层只能按前驱规则选 Loop 的命名 outputs。运行时仅读当前轮，
  第二轮不得因第一轮归档中的旧值而命中。有效容器以非空 `data.containerId` 优先，缺失/空值才回落 `parentId`。
- 禁止结果：跨容器连边补足本 body 前驱；用 Loop 最终 outputs 冒充本轮 carried 根；从其他 body 或旧轮归档补值；
  门禁与执行使用不同成员集合；外层绕过 Loop outputs 直接读 body 内部状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 本 body 直接/传递前驱允许，body 内非前驱与其他容器成员拒绝 | 发布门禁容器图对照：仅改变 producer 归属或边，核对 `aggregator_selector_not_upstream`、节点和字段 | Missing |
| 外层祖先、carried 与环境根可用，外层非祖先拒绝 | 发布与真实 PostgreSQL 轮次用例：祖先在进入 Loop 前已提交、carried 随轮变化；以外层兄弟替换候选时发布失败 | Missing |
| 外层只经 Loop 已声明的命名 outputs 读取轮内产物 | 发布门禁对照：Loop 是外层 aggregator 的前驱，选择 Loop outputs 可发布，直接选其 body 成员被拒 | Missing |
| 第二轮不从第一轮归档或其他 body 的扁平状态补值 | 真实 PostgreSQL：保留旧轮/其他 body 的不同哨兵值，本轮根缺失，聚合按后续候选 fallback 或形成 `aggregator_no_match` | Missing |
| 门禁与执行共享同一有效容器归属 | 直接容器归一测试与真实 PostgreSQL 轮次用例：`containerId` 优先、缺失/空值回落、冲突时显式值胜出，前驱与取值集合一致 | Missing |

- 决策依据：D1 的前驱与容器边界、D2，不变量 7；与 Loop 的独立轮次作用域及有效容器约定共同约束。

## Invalid Aggregator Configuration Is Rejected At Publication

- 风险：非法候选进入不可执行的冻结快照；拒绝信息无法定位，或草稿保存被发布门禁阻断。
- 前置状态：聚合配置缺失/非对象，候选列表缺失/非列表/为空，选择器不是两个非空字符串或带嵌套路径，
  候选重复，或根未声明；另有一份形状与前驱关系均合法的对照配置。
- 触发：分别保存草稿和发布；运行时读取无法解析的聚合配置。
- 必须成立：保存草稿保持宽松；发布按 D1 对应错误码拒绝并给出 `nodeId`、`field`，不冻结非法快照，合法配置可发布。
  运行时无法解析配置时按 `aggregator_invalid_config` 失败，不猜测配置、不降级成空成功。
- 禁止结果：用默认空列表补齐非法配置；裁掉嵌套路径后放行；重复根被静默去重；草稿无法保存；运行遇到坏配置仍成功。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 缺失/非对象配置、缺失/非列表/空列表分别按对应码拒绝 | 发布门禁直接测试：`aggregator_config_missing`、`aggregator_variables_not_a_list`、`aggregator_variables_empty`，同时断言节点与字段 | Missing |
| 非法/嵌套选择器、重复项、未声明根分别按对应码拒绝 | 发布门禁直接测试：`aggregator_selector_unsupported`、`aggregator_selector_duplicate`、`aggregator_selector_undeclared`，同时断言节点与字段 | Missing |
| 草稿可保存，非法发布不生成快照，合法发布成功 | 真实发布链路：保存同一非法草稿成功，发布被拒且没有新快照；纠正后发布成功并保持候选顺序 | Missing |
| 运行时无法解析的配置稳定失败 | 调度器直接测试：坏配置返回 `aggregator_invalid_config`，不提交成功状态或伪造输出 | Missing |

- 决策依据：D1，不变量 5；非前驱错误单独见上面的发布前驱用例。
