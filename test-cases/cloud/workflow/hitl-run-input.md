# 人工节点的运行级输入与有界等待

依据[人工节点运行级输入契约](../../../decisions/cloud/workflow/20261021-human-node-run-input.md)
（`approved`，D1–D9）。这些用例覆盖 `human` 节点进入有界等待、输入校验与追加、幂等、无绑定运行的权限、
409/410 判定顺序、取消竞争、超时结算、运行详情的读取形状、Issue 活动与恢复重算；agent 节点的重试等待见
[节点重试与恢复](retry-and-resume.md)，调度器整体行为见[调度串行与恢复](scheduler-serialism-and-recovery.md)。

产品实现尚未开始，cloud 侧义务全部为 `Missing`，随实现逐项登记直接测试。实现本文前发布门禁继续拒绝
`human` 节点与 `interactive: true`，公开契约也不含 `awaiting_input`（D9）；这些用例约束的是实现合并后、
且 `awaiting_input` 重新进入公开枚举时的行为。

## A Ready Human Node Enters A Durable Bounded Wait

- 风险：运行停在 `running` 却没有任何可输入的对象；等待只存在进程内或不在扫描范围，输入、取消与超时
  都无人消费；等待运行按 `created_at` 挤占在途运行批次；`human` 节点被会话证据路径当成「已派发未回报
  的 agent」而按启动超时失败；下游与兄弟节点在等待期间提前派发；等待列只写一半。
- 前置状态：运行处于 `running`，图中的一个 `human` 节点就绪，其冻结快照里的 `humanConfig.inputSchema`
  合法；同一运行还有该节点的下游节点与兄弟 agent 节点。
- 触发：调度 pass 发现该 `human` 节点就绪；等待中的运行数超过单批扫描上限；两个副本并发 pass。
- 必须成立：同一个事务里把该节点置为 `running`、写入成对的 `awaiting_input_at` 与
  `awaiting_input_node_id`、把运行置为 `awaiting_input` 且版本 +1、为关联 IssueRun 追加一条活动；等待
  运行由独立的**有界**扫描覆盖，超出上限的等待留到下一轮，`pending` / `running` 批次不被等待运行挤占；
  每个等待运行的 pass 在读取任何节点执行证据之前按「已记录取消 → 已接受输入 → 已过截止 → 原样返回」
  处理；`workflowRunningAgentNode`、`workflowAgentSessionOutcome` 及其调用方读取证据时结合图节点 kind，
  `human` 节点不进入会话证据路径；等待期间不派发该运行的其他节点；并发 pass 由运行级排他事务与版本
  CAS 串行化，同一个等待只进入一次。
- 禁止结果：`human` 节点被创建 AgentSession、执行工作项或按会话启动超时失败；等待运行永远不被扫描；
  等待运行挤占在途批次；下游或兄弟节点在等待期间就绪；两个等待列只写一列；两个副本各写一次等待或产生
  两个节点状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `human` 节点就绪时同一事务写入节点 `running`、两列等待、运行 `awaiting_input` 与版本 +1 | 真实 PostgreSQL：调度 pass 后检查 `node_states`、`workflow_runs` 与 Issue 活动 | Missing |
| 等待运行由独立有界扫描覆盖，超出上限的等待留到下一轮，在途批次不被挤占 | 真实 PostgreSQL：等待数超过批次上限时 pass，检查本次实际前进的运行集合 | Missing |
| 等待分支先于节点证据，`human` 节点不被会话证据路径判为失败 | 真实 PostgreSQL：写入等待后连续 pass，检查无启动超时失败、无执行工作项与 AgentSession | Missing |
| 等待期间下游节点与兄弟 agent 节点不被派发 | 真实 PostgreSQL：含下游与兄弟 agent 的图，等待期间无新执行工作项 | Missing |
| 并发 pass 只进入一次等待，等待列与版本只前进一次 | 真实 PostgreSQL：并发调用调度 pass，检查等待列、运行版本与 Issue 活动数量 | Missing |

- 决策依据：D2，不变量 3、4，根决策 D1 与 D2。

## The Publish Gate And Run Admission Require A Supported Human Input Schema

- 风险：门禁继续拒绝全部 `human`，或放行缺失、非法、含不受支持关键字的 schema，冻结一份实现会忽略的
  契约；历史快照带着「没有校验」进入等待；输入期校验读当前文档而不是冻结快照，创建运行后被编辑的工作流
  悄悄改变该运行的校验规则。
- 前置状态：草稿 `human` 节点的 `humanConfig.inputSchema` 分别为缺失、`null`、空对象、合法子集、含
  `oneOf` / `$ref` / `format` 等不支持关键字、`required` 指向未声明属性；已发布的历史快照；创建运行后
  重新发布同一工作流。
- 触发：发布草稿；对历史快照创建运行；提交输入。
- 必须成立：缺失（含 `null`、空对象）以 `human_input_schema_missing` 拒绝；结构非法以
  `human_input_schema_invalid` 拒绝；出现子集外关键字以 `human_input_schema_unsupported` 拒绝并指出节点与
  字段；保存草稿仍宽松；历史快照不重新过发布门禁，但运行准入要求每个 `human` 节点有合法 schema，缺失或
  非法时以同一命名原因拒绝创建运行、不回填、不猜测；输入期校验使用运行创建时冻结的图快照，创建后编辑
  或重新发布不改变该运行的校验规则。
- 禁止结果：缺失 schema 被当作「接受任意 JSON」放行；不受支持关键字被静默忽略；历史快照无校验进入等待；
  运行中途改用工作流当前文档的 schema。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 缺失、非法与不受支持关键字各返回命名错误并定位节点与字段 | 发布门禁表驱动测试：三类错误码分别带 `nodeId` 与 `field` | Missing |
| 受支持子集的合法 schema 发布成功，保存草稿仍宽松 | 真实 PostgreSQL：含合法与非法 schema 的草稿发布与保存 | Missing |
| 历史快照在运行准入处按同一子集校验，缺失或非法拒绝创建运行 | 真实 PostgreSQL：直接写入旧形状快照后创建运行，检查命名原因 | Missing |
| 输入期校验使用冻结快照，创建运行后的编辑与重新发布不影响该运行 | 真实 PostgreSQL：创建运行后改工作流 schema 再提交输入，观察判定不变 | Missing |

- 决策依据：D5，不变量 8，根决策 D2 的不变量 1，重试决策 D8 的差异说明。

## An Input Request Appends One Row And The Scheduler Consumes It Exactly Once

- 风险：请求事务直接写节点或运行状态，绕过调度器；同一等待接受多行输入；重复提交或幂等重放产生第二行、
  返回不同响应；调度器崩溃后重复消费或丢失已提交输入；`node_states` 被塞进输入历史，恢复重算时出现
  两个事实来源。
- 前置状态：运行处于 `awaiting_input`，请求 `nodeId` 匹配，该等待还没有输入行；`Idempotency-Key` 分别为
  未使用与已使用；调度器在输入提交前、提交后但消费前崩溃。
- 触发：`POST …/runs/:rid/inputs`；同一 `Idempotency-Key` 加同一请求重放；不同幂等键再次提交；提交后一次
  或多次 pass；进程重启后的首个 pass。
- 必须成立：请求只追加一条输入行并返回 202 与
  `{"resource": {"id", "runId", "nodeId", "actorId", "input", "recordedAt"}}`，不写运行状态、不写
  `node_states`、不推进图；同一 `Idempotency-Key` 加同一请求由平台幂等记录返回原响应，不产生第二行；不同
  幂等键得到 `409 workflow_input_already_recorded`；唯一约束 `(workflow_run_id, node_id)` 成立；下一次重算
  在同一事务里把节点置 `succeeded`、`output` 为通过校验的输入对象、写 `finished_at`、清空两列，把运行放回
  `running`（或图结束时的 `succeeded`）；一次提交只消费一次，重启后读到的仍是同一条已提交行。
- 禁止结果：202 已接受但永不消费；请求事务推进图或写节点终态；同一等待出现第二行输入；两次消费或消费
  丢失；输入行被更新、删除或覆盖；`node_states` 保存输入历史。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 请求只追加输入行并返回 202，运行时状态与图不变 | 真实 PostgreSQL：请求后检查输入行、`workflow_runs` 与 `node_states` | Missing |
| 同一幂等键重放返回原响应，不同幂等键返回 `409 workflow_input_already_recorded` | 真实 PostgreSQL：两次相同请求与一次新键请求，检查输入行数 | Missing |
| 下一次重算消费输入并把节点结算为 `succeeded`，输出取自输入对象 | 真实 PostgreSQL：提交输入后 pass，检查节点终态、`output` 与等待列 | Missing |
| 崩溃或重启后同一输入只消费一次，不丢失也不重复 | 真实 PostgreSQL：模拟消费事务提交前后重启，检查节点版本与输出 | Missing |

- 决策依据：D1、D2、D6，不变量 1、5、7。

## Only The Initiator Or A Tenant Admin Controls A Waiting Run

- 风险：`start → human → output` 这类 human-only 图不冻结任何 `run_model_bindings` 行，若沿用「无绑定运行
  回退到成员规则」，输入会对任意活跃成员开放；权限取自请求里的 actor 字段；未授权的调用者从 409/410/400
  的差异推断运行是否在等待；等待中的取消与输入使用不同谓词，形成「不能完成、但能作废」的旁路。
- 前置状态：human-only 运行（没有冻结绑定）；调用者分别为运行发起人 `created_by`、当前租户
  `status='active'` 的管理员、普通活跃成员、非活跃管理员、已被撤销管理员权限的成员；同一租户还有另一
  运行的发起人。
- 触发：各类调用者提交输入；等待中的运行请求取消；作为对照，`pending` / `running` 的运行请求取消。
- 必须成立：输入与等待中的取消只允许 `uid = created_by` 或当前租户中 `status='active'` 且
  `role='admin'` 的成员，**与是否冻结模型绑定无关**；其他活跃成员得到
  `403 workflow_run_input_forbidden`；`awaiting_input` 进入可取消状态集合，并沿用同一谓词；授权检查先于
  状态、节点匹配与输入校验，未授权调用者得不到 409/410/400 的任何差异；`pending` / `running` 的取消规则
  保持既有行为不变。
- 禁止结果：无绑定运行按成员规则放行；非活跃或已撤权的管理员放行；授权依赖请求体中的 actor 字段；先做
  状态判定再授权，泄露运行是否在等待；等待中的取消沿用更宽的成员规则。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 发起人与当前租户管理员可提交输入，其他活跃成员返回 403 | 真实 PostgreSQL：human-only 运行上四类调用者各提交一次 | Missing |
| 撤销管理员权限或转为非活跃后立即失去输入与等待中取消的权限 | 真实 PostgreSQL：提交前修改角色或状态，检查 403 与无输入行 | Missing |
| 未授权调用者先得到 403，不能从 409/410/400 差异推断运行状态 | 真实 PostgreSQL：同一调用者分别对等待、已取消、窗口外运行请求，响应一致 | Missing |
| `pending` / `running` 的取消仍按既有谓词，不受本节收窄影响 | 真实 PostgreSQL：无绑定运行在 `running` 下由有效成员按既有规则取消成功 | Missing |

- 决策依据：D4，不变量 9，个人模型发起者决策的「无模型绑定运行保持原规则」。

## Input Acceptance Returns 409 And 410 In A Fixed Order

- 风险：窗口外返回 202 却永远不会被消费；取消后返回 410 暗示输入还有机会生效；判定顺序不一致，同一状态
  得到不同错误码；窗口判定用应用进程时钟，多副本与调度器看到不同的截止时间；判定与写入分离，出现
  TOCTOU 后仍写入输入行。
- 前置状态：运行分别为 `awaiting_input`、已被超时结算的 `failed`、已取消的 `cancelled`；请求 `nodeId`
  分别匹配与不匹配；`cancel_requested_at` 分别已写与未写；该等待分别已有与没有输入行；数据库时钟分别在
  截止时间前后。
- 触发：上述组合下提交输入；取消与输入在同一时刻并发提交。
- 必须成立：授权通过后在写入输入行的同一事务里按固定顺序判定——运行 `awaiting_input`（409
  `run_not_awaiting_input`）→ `nodeId` 等于 `awaiting_input_node_id`（409）→ `cancel_requested_at IS NULL`
  （409）→ 尚未记录输入行（409 `workflow_input_already_recorded`）→ 数据库时钟早于
  `awaiting_input_at + workflow.hitl_wait_timeout`（410 `workflow_run_input_window_closed`）；410 只在运行
  仍等待、节点匹配、无取消请求、无已接受输入且窗口已过时可达；超时或取消结算后晚到的输入先命中状态检查，
  得到 409 而不是 410；判定使用数据库时钟，任一步不满足都不写输入行。
- 禁止结果：窗口外返回 202；取消请求已记录但未结算时返回 410；顺序颠倒导致已结算运行返回 410；使用应用
  时钟；判定与写入不在同一事务，竞争下写入窗口外或取消后的输入行。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 五步判定按固定顺序返回对应 409/410，且不写输入行 | 真实 PostgreSQL：逐组合驱动请求，检查错误码与输入行数 | Missing |
| 410 只在运行仍等待且其它条件满足时可达 | 真实 PostgreSQL：取消、超时与已接受输入三种前置下的窗口外请求 | Missing |
| 已结算运行的晚到输入返回 409 `run_not_awaiting_input` | 真实 PostgreSQL：超时与取消结算后分别提交输入 | Missing |
| 判定与写入在同一事务内，使用数据库时钟 | 真实 PostgreSQL：并发提交与截止时刻边界，检查无越界输入行 | Missing |

- 决策依据：D1、D3、D4，不变量 1、2。

## Cancellation Overrides A Waiting Input And Clears The Wait

- 风险：`awaiting_input` 仍被当作不可取消，等待只能拖到超时；取消要等到超时结算才生效；已提交但未消费的
  输入把运行从终态复活或覆盖取消结果；等待列在终态残留半截事实。
- 前置状态：运行处于 `awaiting_input`，分别没有输入行与有已提交但未消费的输入行；`cancel_requested_at`
  为空。
- 触发：请求取消后 pass；取消与输入并发提交；取消后再次提交输入。
- 必须成立：`awaiting_input` 进入可取消状态集合，等待中的取消使用 D4 谓词；取消请求记录后，调度器在下一次
  重算把节点与运行结算为 `cancelled`（没有会话可停止），同时清空两个等待列；未消费的输入行保留为历史，
  但不产生节点成功；取消先提交时，之后的输入先命中 D1 的取消检查得到 409；输入先提交但尚未消费时取消
  到达，取消仍决定运行终态；终态不被输入复活。
- 禁止结果：取消被等待拖延；取消与超时同时写终态；已提交输入在取消后仍把节点置为 `succeeded`；终止后
  运行回到 `running`；等待列残留。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 等待中取消立即结算 `cancelled` 并清空等待列，未消费输入保留为历史 | 真实 PostgreSQL：等待中请求取消后 pass，检查运行、节点、输入行与等待列 | Missing |
| 已提交未消费的输入不能让取消后的运行复活或节点成功 | 真实 PostgreSQL：先提交输入再取消，pass 后检查节点为 `cancelled` | Missing |
| 取消与输入并发只产生一种可解释结果 | 真实 PostgreSQL：并发提交取消与输入，检查终态与输入行二者择一 | Missing |

- 决策依据：D4，不变量 2、6，根决策 D6。

## An Unanswered Wait Fails The Run When The Window Expires

- 风险：没有人处理时运行永久停在 `awaiting_input`，Workspace 无限期保留；超时被静默当成「已批准」并继续
  执行下游分支；已接受的输入被超时覆盖；截止判定受调度周期、进程时钟或副本数量影响。
- 前置状态：运行处于 `awaiting_input`，`awaiting_input_at` 已提交；部署配置分别为默认 24 小时与调整后的
  值；分别没有输入行与有截止前提交的输入行。
- 触发：数据库时钟分别在截止时间前后 pass；调度器停机后重启再 pass；等待期间调整
  `workflow.hitl_wait_timeout`。
- 必须成立：窗口从 `awaiting_input_at` 起算，用数据库时钟与当时配置比较，与调度周期、进程时钟和副本数量
  无关；到期且没有已接受输入时，节点以 `hitl_wait_timeout` 失败（节点 `error` 记同名原因）、运行进入
  `failed` 且 `error` 为 `human input wait timed out`、Workspace 按既有终态路径释放；存在截止前提交的
  已接受输入时消费输入而不是判超时，即使重算发生在窗口之后；已记录的取消请求优先，结算 `cancelled` 而不写
  超时失败；离开等待的每条路径同时清空两个等待列。
- 禁止结果：永久等待；超时后跳过节点继续执行下游；已接受输入被超时覆盖；用应用时钟或调度周期判断到期；
  终态残留等待列。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 到期且无输入时节点与运行按命名原因失败，Workspace 释放 | 真实 PostgreSQL：调整截止前后 pass，检查节点、运行与释放声明 | Missing |
| 截止前提交的输入优先于超时，窗口后重算仍消费输入 | 真实 PostgreSQL：截止前提交、截止后 pass，检查节点为 `succeeded` | Missing |
| 取消优先于超时，不写超时失败 | 真实 PostgreSQL：同时存在取消请求与过期窗口，检查终态为 `cancelled` | Missing |
| 到期判定只依赖 `awaiting_input_at`、当前配置与数据库时钟 | 真实 PostgreSQL：调整周期、进程时钟与副本数后比对判定 | Missing |

- 决策依据：D3，不变量 2、6，根决策 D7。

## The Run Detail Is The Authoritative Read Of Wait And Inputs

- 风险：运行页只能轮询列表猜等待状态；详情缺截止时间，直接运行（没有 Issue 活动）无处可读；把 `deadline`
  另存第三列后与配置或算法漂移；`inputs` 顺序不稳定导致页面跳变；列表行泄露输入内容。
- 前置状态：运行分别为等待中、已消费输入、从未有输入；等待期间调整 `workflow.hitl_wait_timeout`；同一
  运行有多条输入行（不同节点，首期 human 节点至多一次，列表顺序仍须稳定）。
- 触发：`GET …/workflows/:wfid/runs/:rid` 与列表读取；等待中修改配置后再次读取。
- 必须成立：详情在现有投影上新增 `awaitingInput` 与 `inputs` 两个只读字段；`awaitingInput` 在不在等待时为
  `null`，等待时为 `{"nodeId", "startedAt", "deadline"}`；`startedAt` 就是 `awaiting_input_at`，`deadline`
  按当前配置与数据库时钟算法实时计算，不另存第三列，部署调整配置后未结算等待的截止时间一起移动；`inputs`
  按 `recordedAt` 升序、同一时刻按 `id` 升序，每项
  `{"id", "nodeId", "actorId", "input", "recordedAt"}`，没有输入行时为空数组；直接运行也能从详情读到等待与
  输入；列表行不含这两个字段，只通过 `status = "awaiting_input"` 表达等待；节点最终输出仍只从
  `nodeStates[nodeId].output` 读取。
- 禁止结果：新增 `deadline` 存储列造成漂移；列表行返回 `inputs`；顺序不稳定；没有 IssueRun 时读不到截止
  时间；`awaitingInput` 与等待列不一致。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 详情在等待与非等待时分别返回对象与 `null`，字段齐全 | 真实 PostgreSQL + HTTP：等待、消费、取消、超时四种状态各读一次 | Missing |
| `deadline` 按当前配置实时计算，调整配置后未结算等待随之移动 | 真实 PostgreSQL：等待中修改 `workflow.hitl_wait_timeout` 后重新读取 | Missing |
| `inputs` 顺序稳定、空数组，且列表行不含等待与输入字段 | 真实 PostgreSQL + HTTP：多条输入行与列表读取对比 | Missing |

- 决策依据：D1 的读取形状、D8，不变量 4。

## The Waiting Transition Writes One Issue Activity And No Thread Turn

- 风险：等待时没有活动，参与者只能靠轮询运行页；把输入记成线程回合，伪造一条不存在的人类对话并污染
  Thread 语义与统计；每次 pass 重复追加活动；IssueRun 状态被等待改写；把活动里的截止时间快照当成权威值。
- 前置状态：WorkflowRun 关联了 workflow IssueRun；随后进入等待、消费输入、取消或超时；等待期间调整
  `workflow.hitl_wait_timeout`。
- 触发：进入等待的同一次事务；消费输入后的 pass；等待中的多次 pass；配置调整后重新读取运行详情与活动。
- 必须成立：进入等待的同一次事务追加**一条** Issue 系统活动，动作名 `run.awaiting_input`，正文包含节点
  id、节点显示名与进入等待时按当时配置算出的截止时间快照；后续 pass 不重复追加；输入被消费时不新开线程
  回合、不写 `thread_commands`、不改变 IssueRun 的 `status`，IssueRun 按既有投影保持 `running`；直接运行
  没有活动，只从运行详情读取等待与截止时间；配置调整后活动里的截止时间不更新，运行详情的
  `awaitingInput.deadline` 是权威值。
- 禁止结果：输入成为线程回合或写 `thread_commands`；同一等待出现多条活动；IssueRun 状态被等待或输入改写；
  把活动快照当成截止时间的最终依据。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 进入等待只追加一条 `run.awaiting_input`，后续 pass 不重复 | 真实 PostgreSQL：等待后连续 pass，统计活动数量与正文字段 | Missing |
| 消费输入不新开线程回合、不写 `thread_commands`、不改 IssueRun 状态 | 真实 PostgreSQL：消费前后对比 IssueRun、Thread 与 `thread_commands` | Missing |
| 活动截止时间是进入等待时的快照，运行详情在配置调整后是权威值 | 真实 PostgreSQL：等待中调整配置，对比活动正文与详情 `deadline` | Missing |

- 决策依据：D8、D1，根决策 D3。

## Recovery And Multiple Replicas Recompute The Wait From Committed State

- 风险：输入的消费、取消与超时依赖进程内定时器或内存队列，崩溃或换副本后等待永久卡住；重启后重复消费
  同一输入或丢失已提交输入；两个副本各自结算出不同终态；等待列与运行状态在恢复后分叉。
- 前置状态：已提交的输入行、已记录的取消请求、成对非空的等待列；调度器重启或第二副本开始 pass。
- 触发：重启后的首个 pass；并发 pass；消费事务提交前崩溃；等待运行与在途运行混合的批次。
- 必须成立：等待分支的所有判定只读已提交行、配置与数据库时钟，不依赖进程内状态；固定顺序为「已记录取消
  → 已接受输入 → 已过截止 → 原样返回」，任何一步都不落到节点会话证据路径；一次 pass 至多结算一次；重启
  后读到的仍是同一条已提交输入，消费一次且只消费一次；多副本由运行级排他事务与版本 CAS 串行化；消费、
  取消与超时各自在事务里同时清空两个等待列并把运行放到 `running` / `succeeded` / `cancelled` / `failed`；
  不同结算路径不会同时生效。
- 禁止结果：依赖进程定时器或内存队列；重启后重复消费或丢失输入；两个副本写出两种终态；等待列非空但运行
  已在终态，或运行 `awaiting_input` 却没有等待列。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 重启后按已提交输入与取消请求继续，消费一次且不丢失 | 真实 PostgreSQL：新建调度器实例对同一等待 pass，检查节点与输入行 | Missing |
| 并发 pass 只产生一种结算结果 | 真实 PostgreSQL：两个实例并发 pass，检查节点版本、运行终态与活动 | Missing |
| 每条离开等待的路径同时清空两列，状态与等待列始终一致 | 真实 PostgreSQL：消费、取消、超时各驱动一次后检查等待列不变式 | Missing |

- 决策依据：D2、D3、D4，不变量 2、3、4、5，根决策的条件②（无状态调度）。
