# 交付业务结算与原执行未受理证明

依据[交付截止结算与未受理证明 ADR](../../../decisions/cloud/issue-run/20261026-delivery-execution-settlement-and-non-acceptance.md)
（`approved`，D1–D5）。按跨 Cloud、Controller、Node 的责任和风险登记义务；本文不建立第二份协议。
本批先登记验证要求，尚未核定最终合并实现的直接证据，全部保持 `Missing`。候选测试通过不等于
部署、物理释放或原任务恢复通过；后续逐项登记固定版本与证明边界。

## Business Settlement Closes Dispatch Without Inventing Execution Facts

- 风险：用 Cloud 超时伪造 Node 终态，或业务已结束仍不断领取、登记交付执行。
- 前置状态：分别存在未登记 work、已登记但响应丢失的 execution、已受理但回报丢失的 execution；
  产物达到绝对截止或 Revision 按既有 D5 放弃。
- 触发：Cloud 执行业务结算，并让旧 claim、缓存输入、新领取、新登记和新 permit 请求跨越提交时点。
- 必须成立：取得权威锁后按数据库时间判断，独立结算来源/时间与派发资格封闭同事务生效；失败全部
  回滚。原身份保持，未登记 work 留作审计但不再反复占据队头；已经发出的 permit 仍可能有外部责任。
- 禁止结果：写伪造 result、sequence、receipt 或 Revision；以新 execution 重试未知结果；仅因业务
  结算就宣告 idle、终止或可删数据；回滚后留下半份结算。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 截止与登记/permit 竞争原子串行，旧 claim 不能绕过 | 真实 PostgreSQL 竞争与事务故障测试，检查完整允许结果、回滚及权威时刻 | Missing |
| 三类派发/受理前置均不被伪造为 Node 终态 | 控制面与真实执行侧联调，核对原身份、result/sequence/receipt/Revision 和未知责任 | Missing |
| 未登记 work 不再领取且不挡住其他合法 work | 实际领取路径测试，核对拒绝后的队列推进与无新 execution | Missing |

- 决策依据：D1，不变量 1、2、5。

## Acceptance And A Durable Input Fence Cannot Both Commit

- 风险：Node 查无后先回复证明、再写围栏，使迟到受理仍能执行。
- 前置状态：原实例拥有完整 ledger；同一交付 execution 尚未受理，旧 permit 或派发已在途。
- 触发：真实受理与未受理证明并发；分别控制受理先提交、证明先提交，并在持久化时注入失败。
- 必须成立：完整 ledger 检查、不可撤销围栏和证明同一持久事务提交；受理先成功则拒绝证明，证明
  先成功则拒绝受理。只有持久提交成功后才回复证明。
- 禁止结果：双方都成功、仅内存围栏、磁盘失败仍回复证明，或通过重连/新 epoch/迟到旧 permit 受理。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 两种提交顺序及真实并发只允许一个赢家 | Node 生产受理/证明路径和真实持久库的竞争测试，直接检查执行副作用与 ledger | Missing |
| 围栏跨持久化失败、重启、重连与 epoch 仍正确 | 写失败和进程重启测试，证明失败不产生成功回复、已提交围栏阻挡迟到输入 | Missing |

- 决策依据：D2，不变量 3、4。

## Non Acceptance Proofs Bind The Original Instance And Frozen Input

- 风险：新实例空 ledger、错绑工作项或变化的输入被当成旧执行未受理证明。
- 前置状态：Cloud 持有原冻结身份；分别改变 tenant、Workspace、sandbox/generation、Node/instance/
  incarnation、execution、work、Cloud operation、Node operation 或执行输入。
- 触发：经真实协议请求、重放并提交证明；另使用缺少原 ledger 的替换实例。
- 必须成立：Node 和 Cloud 各自校验所属边界；原证明只按原身份重放，同名 execution 的不同输入或
  operation 拒绝；任何缺失/不匹配都保留未知责任，不能由当前身份补猜历史。
- 禁止结果：跨租户/代次/实例接管、Controller 改绑原身份、换输入复用证明或新空库证明旧实例未受理。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 完整身份和冻结输入在 Node/Controller/Cloud 逐跳一致 | 实际协议往返及逐字段错绑拒绝矩阵，核对拒绝无责任清除或持久事实覆盖 | Missing |
| 新实例缺证据时等待，原持久证明可安全重放 | 原实例/替换实例与重启场景，核对 ledger、证明和 Cloud 接管后的原身份 | Missing |

- 决策依据：D2、D3、D5，不变量 4。

## Missing Replies And Host Stop Evidence Do Not Prove Non Acceptance

- 风险：将 Unknown、sequence=0、host NotStarted 或进程尚未启动误作业务 execution 未受理。
- 前置状态：分别存在 Accepted、Running、Completed、恢复未知责任、不可完整读取 ledger，以及
  已受理但 host 报 NotStarted 的 execution；另含 host UnknownIdentity。
- 触发：请求未受理证明、断线重连和执行状态查询。
- 必须成立：上述情况不产生新未受理证明；已受理执行继续按原事件/查询对账，证据不足保持未知。
  host 对 ProcessRun 的停止证据不替代业务 execution 的原子 ledger 检查。
- 禁止结果：从查询为空、缺回报、超时或 host 停止证据推导未受理，直接移除 pending/unresolved。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 已受理与 ledger 不完整的各状态拒绝新证明 | Node 真实持久状态测试，覆盖已完成记录、恢复责任及读取失败 | Missing |
| Unknown 与 host 两类停止应答不进入未受理证明链 | 生产 Controller/Node 接缝测试，直接断言未知责任保留且无成功证明 | Missing |

- 决策依据：D2、D3，继承表的 Node 停止确认证据，不变量 1、4。

## Only Cloud Commit Releases The Matching Unfinished Responsibility

- 风险：Controller 在 Cloud 持久接管前清除责任，或把专门证明包装为普通终态 ACK。
- 前置状态：Node 已持久产生原证明，Cloud 接管分别成功、失败、响应丢失或租约已过期。
- 触发：Controller 提交证明，重试、重连或由新租约持有者接手。
- 必须成立：Cloud 在当前服务身份/租约下核对原身份、业务结算及证明，独立持久保存；真实提交后才
  解除对应 execution 的责任，失败/未知响应不能凭本地猜测解除；重放只确认原证明。
- 禁止结果：创建 Completed/Failed 事件、推进 sequence、发普通 event ACK、登记 artifact/Revision，
  或清掉其他 execution 的责任。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| Cloud 接管独立持久且受原身份、服务身份和租约约束 | 真实 PostgreSQL/gRPC 成功与拒绝矩阵，检查事实、序列和业务对象无额外写入 | Missing |
| 只有真实提交后才清除对应 pending/unresolved | 实际 Controller 会话与 Fleet 观察路径，注入接管失败/响应丢失/重启/租约切换 | Missing |

- 决策依据：D3，不变量 1、4、5。

## Execution Fences Do Not Replace Global Quiesce Or Cross Delivery Families

- 风险：一个产物到期或一份证明关闭整个 Workspace 输入，误伤尚有效 Revision，或漏掉其他活动责任。
- 前置状态：同 Workspace 同时存在已结算产物执行、仍有效 Revision 以及其他执行/票据或 unresolved。
- 触发：封闭一个 execution 输入、提交其证明并请求 quiesce；另交换两种交付族的角色。
- 必须成立：各交付族的窗口、授权和结果独立；未受理证明只解除对应 execution。全局 quiesce 仍
  关闭新派发，检查所有剩余 execution/票据、准入及控制代次。
- 禁止结果：局部围栏取消其他合法交付、单份证明使 Workspace 自动 idle、绕过其他未完成责任。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 产物与 Revision 的资格/期限互不越界 | 两族并存的实际 Cloud 与 Node 路径测试，分别先截止并检查另一族仍合法 | Missing |
| 局部解除后全局 quiesce 仍检查其余责任 | Controller 生产 idle/quiesce 路径，含其他交付族、票据、unresolved 和过期代次 | Missing |

- 决策依据：D1–D3，不变量 5、7。

## Late Facts And Conflicting Proofs Never Rewrite Settled History

- 风险：截止后直接 ACK/丢弃真实终态，或矛盾事实重放变成功而解除保护。
- 前置状态：业务已结算，分别有迟到真实终态、已保存未受理证明、同 Issue 后创建 Run；对象登记
  窗口已到期或原 Revision 尚未登记。
- 触发：原身份提交终态或证明，包括相同 submission 重放、新 submission 重放和相互矛盾事实。
- 必须成立：迟到真实事实沿既有 receipt-only/后创建 Run 守卫处理，真实接管前不 ACK；原 Run 结果
  和交付窗口保持。矛盾持久审计，不覆盖原事实，不因重放放行 idle。
- 禁止结果：伪造 Revision、延长对象登记窗口、覆盖新 Run 成果、以行政结算代替接管，或冲突重放
  成功后清除未知责任。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 迟到事实遵守原身份、回执与后创建 Run 守卫 | 真实 Cloud 事务/协议测试，检查旧结果、新成果、登记窗口及 ACK 顺序 | Missing |
| 矛盾事实和证明重放保持审计与保护 | 双向冲突、相同/不同 submission 回归，直接核对持久冲突、拒绝响应与 idle 阻挡 | Missing |

- 决策依据：D4，不变量 1、4、6。

## Legacy Migration Preserves History And Does Not Invent Origin Evidence

- 风险：升级重写历史执行事实，或用迁移时刻和当前实例冒充旧截止/旧执行身份。
- 前置状态：存量已结算产物有持久截止证据，其他已结算 Run 仍有未知交付；另含真实终态和缺失原实例。
- 触发：新库迁移、旧库追加升级与重复执行迁移。
- 必须成立：有产物截止证据的独立登记来源；其余按存量已结算首次观测记录迁移时刻。旧 result、
  sequence、receipt、Run 和 Revision 不变；无可靠原实例的责任仍未知，不具备新证明资格。
- 禁止结果：补造原截止/实例、改写原业务结果、重复迁移重复结算、升级后错误放行未知资源。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 新库/存量/重放保留历史并区分结算来源 | 真实 PostgreSQL 前一 schema 升级及重复迁移，对比完整历史和来源/时刻 | Missing |
| 缺失原实例不获得证明资格或物理释放资格 | 升级后的控制读取和资格判定，核对未知责任及原身份缺失边界 | Missing |

- 决策依据：D5，不变量 1、4、6。

## Schema And Capability Upgrades Fail Closed Across Mixed Versions

- 风险：旧 Node 忽略新围栏，或新 Controller 对未协商节点发请求并以兼容 fallback 宣告 idle。
- 前置状态：旧/新 Node、Controller 和 Cloud 的混合版本，Node 含已持久围栏的新 schema。
- 触发：升级、重启、尝试旧程序打开新库及 Hello 能力协商；读取既有封闭响应。
- 必须成立：显式 schema 升级保留围栏，旧程序拒绝不认识的 schema；仅协商成功才请求证明，旧版本
  保守保留未知。新增字段由拥有者契约定义/生成，不自动混入旧封闭投影。
- 禁止结果：降级忽略围栏、向旧 Node 发陌生请求、无 B 能力却清除责任、手写不一致 wire。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 持久 schema 升级及旧程序拒绝保护围栏 | 真实 Node 数据库升级、重启、旧版本打开新库的直接结果 | Missing |
| 协商矩阵与旧封闭响应保持兼容且不放宽责任 | 新旧协议端点组合、生成漂移和实际控制会话测试，检查未知责任保留 | Missing |

- 决策依据：D5，不变量 3、4。

## Original Task Recovery Requires Saved Results And Separate Physical Evidence

- 风险：用业务 done、备份存在或副本任务成功宣称原任务恢复，或提前删除未交付成果。
- 前置状态：原任务被失败交付暂停，原 Node 永久不可达或责任未知，原 Workspace/数据仍保留。
- 触发：在批准实现、明确部署版本与既有授权下进行原任务恢复。
- 必须成立：业务可有界结束，物理未知仍需原证明或真实终止加 late-ensure 围栏；终止后保留数据，
  删除另行确认。原任务完成真实 Claude 两轮执行、两轮 Revision、最终字节/SHA-256、后续消息和
  必要物理释放的直接验收，保留旧失败历史。
- 禁止结果：以副本任务或 CI 替代原任务、以业务结算授权删除、未保全就丢弃成果或修改旧结果。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 永久不可达时业务结算与物理责任/删除授权分离 | 真实部署故障与两阶段受控处置，核对原身份、终止/围栏、保留及独立删除证据 | Missing |
| 原任务可续作且两轮成果真实保存 | 原任务真实 Agent 与 Git Server 内容读回、摘要、后续消息和资源释放证据 | Missing |

- 决策依据：D3、D5、验收与落地，不变量 5、6。
