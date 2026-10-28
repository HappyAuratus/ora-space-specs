# 工作流节点会话读取

依据[工作流节点会话读取](../../../decisions/cloud/workflow/20261026-workflow-node-session-read.md)
（`implemented`，D1–D4）。这些用例覆盖按（运行，节点，尝试）有界分页读取节点会话记录、终态回执按已提交证据
权威排除、旧尝试不冒领最新输出、输出与结束原因分开、授权下探到节点，以及按所选尝试终态证据与历史回执判定
的空态。调度串行与恢复见 [调度串行与恢复](scheduler-serialism-and-recovery.md)，单个尝试内的会话结束见
[工作流会话结束与有界等待](session-end-after-initial-turn.md)，重试隔离见
[工作流节点重试与恢复](retry-and-resume.md)。

实现由运行 [ora-space/circus#720](https://github.com/ora-space/circus/issues/720) 经
[cloud#195](https://github.com/ora-space/cloud/pull/195) 合入 main（merge `63034cfcad687845ee9fdc498449f481e6eb0f9e`），
源码链接固定到该提交。cloud 侧证据来自真实 PostgreSQL 的读取用例（按 router 的入口驱动 `Store.Public`）、
路由契约测试与前端渲染测试；前端测试只证明节点详情的渲染，不代替 Cloud 在真实 PostgreSQL 上的义务。
2026-10-28 在 cloud `63034cf` 上完成了真实栈验收（运行 `a4bb5182…` 的两个 agent 节点、运行 `6deea532…` 的一次
真实重试）：按尝试读到用户输入、工具活动与助手回答，跨节点与跨尝试的执行 ID、会话 ID 互不混用，`output` 与
`sessionEndReason` 分开呈现，`attempt` 越界、`limit=201` 与「`before` 不带 `attempt`」分别得到 404 / 400；
证据与截图记在 #720。真实栈没有再现 `Gap` / `truncated` / 历史开头丢失三种 `incomplete` 判据，也没有覆盖跨运行
越权，这些仍只由真实 PostgreSQL 用例证明。
`output` / `sessionEndReason` 的语义由 cloud#182（已于 2026-10-27 合并）定义，读取端只消费。

本文的 `Partial` 表示已有直接证据只覆盖义务的一部分；每行写明尚缺范围。

## An Attempt-Scoped Read Never Crosses Attempts

- 风险：按（运行，节点）读最新 `agent_session` 行会让第 2 次尝试继承第 1 次的失败结果或记录；翻页期间节点
  重试，下一页按「最新」重新解析会落到新尝试，把新尝试记录接到旧页后面；请求旧尝试时读到新尝试的状态。
- 前置状态：同一（运行，节点）已有第 1 次尝试的执行行、历史回执与结果；第 2 次尝试已启动，或将在翻页期间
  启动。
- 触发：`?attempt=1`、缺省 `attempt`、带 `before` 的翻页请求；翻页期间节点重试。
- 必须成立：服务端按（运行，节点，尝试）解析唯一执行行；`attempt` 越界或不属于该节点 404；缺省读当前/最新
  尝试；`before` 只与 `attempt` 配对，带 `before` 不带 `attempt` 返回 400；翻页请求必须携带首次响应返回的
  `execution.attempt` 与 `before`，服务端钉在该尝试上解析，重试后仍读到原尝试的记录；`execution.attempt` 由
  服务端解析后回显。
- 禁止结果：第 2 次尝试继承第 1 次的结果或记录；下一页落到新尝试；`before` 不带 `attempt` 被静默当作最新
  尝试；旧尝试详情显示新尝试的记录。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `attempt` 越界或不属于该节点 404，缺省读最新尝试 | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadScopesToTheRequestedAttempt` 在同一节点的两次尝试上读 `attempt=1`（执行行、记录与 `result` 都属第 1 次）、缺省（落到第 2 次）、`attempt=3`（404）与 `attempt=0`（`invalid_pagination`）；真实栈（#720，run `a4bb5182…`）缺省读落在唯一尝试、`attempt=2` 返回 404 | Covered |
| `before` 不带 `attempt` 返回 400 | 同一用例的 `before=2` 请求返回 `invalid_pagination`；[路由契约测试][router] 的 `TestNodeSessionQueryAllowlist` 只放行 `limit` / `before` / `attempt` 三个查询参数；真实栈（#720）实测该组合返回 400 `invalid_pagination` | Covered |
| 翻页期间重试，下一页仍读原尝试的记录 | 同一用例先读 `attempt=1` 的一页，再启动第 2 次尝试，然后按 `attempt=1&before=3` 翻页，仍返回 1、2 且 `hasMore` 为假 | Covered |

- 决策依据：D1，不变量 1、4，重试决策 D4，会话结束决策不变量 2。

## A Pagination Cursor Never Skips A Record

- 风险：`nextBefore` 指向未考察记录导致某条记录被永久跳过；整页形状不合法时没有游标、更早的历史取不到；
  `hasMore` 靠本页是否填满导致字节触顶时误判无更多；单页字节无上限形成无界响应。
- 前置状态：一次尝试有 5 条历史回执（外层序号 1、2、3、4、5），或存在使单页字节先触顶的长记录，或本页窗口
  含形状不合法的历史回执（含整页都不合法）。
- 触发：`limit=2` 的首页；字节先触顶的一页；带 `before=<nextBefore>` 的下一页；窗口含形状不合法行的一页。
- 必须成立：取页由新到旧，**响应内 `records` 按外层 `sequence` 升序返回**（与 `thread_read.go`、
  `task_conversation_messages.go` 同向）；`nextBefore` 固定为本页**最早一条考察过**的历史回执外层
  `sequence`（取页窗口升序后的第一行，含形状不合法、只计入 `incomplete` 的行）；本页窗口内没有形状不合法行时
  它就是本页最早一条记录，有（或全是）形状不合法行时它仍指向最早考察过的那一行；下一页 `before=<nextBefore>` 取
  `sequence < before`，每条记录都被后续页面覆盖且每页只考察一次；`hasMore` 由独立探测（是否存在
  `sequence < nextBefore` 的历史回执）决定，条数触顶与字节触顶两种情况都成立；单页受条数（≤ 200）与字节
  （`maxPageBytes`，4 MiB）双上限，任一先触顶处停止，字节预算按回执 `event` 的字节数在加入下一条前判定，
  跨过预算的那一条整条计入，单页正文上限为 `maxPageBytes` 加单条回执 `event` 的字节数上限；`nextBefore`
  仅当存在更早历史回执时返回；终态回执被跳过，不计入 `limit`、不产生 `hasMore`、不参与 `nextBefore`。
- 禁止结果：某条记录被跳过；整页形状不合法时没有 `nextBefore`、更早历史读不到；同一行被反复读回；字节触顶时
  `hasMore` 误报为无；单页超过 200 条；单页正文超过 `maxPageBytes` 加单条回执 `event` 的字节数上限；分页
  折叠或截断记录正文。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 响应按序号升序，`nextBefore` 指向本页最早考察过的序号，逐页覆盖全部记录 | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadPagesEveryRecordWithoutSkipping`：5 条记录 `limit=2` 逐页得到 4,5 / 2,3 / 1，每页 `hasMore` 时 `nextBefore` 等于本页第一条记录，末页不再带 `nextBefore`；真实栈（#720，run `a4bb5182…`）7 条记录 `limit=2` 得到 6,7 → `before=6` 取到 1–5 且 `hasMore` 为假 | Covered |
| 本页含形状不合法行时仍返回 `nextBefore`，下一页取到更早的历史回执，不重复读回同一行 | `TestWorkflowNodeSessionReadReportsUnshapedHistoryWithoutStrandingTheRest`：记录 1、3 合法、2 形状不合法，首页 `limit=1` 返回 3；窗口落在 2 的那一页返回空 `records`、`incomplete`、`nextBefore=2`、`hasMore`，下一页取到 1 | Covered |
| 整页形状不合法时仍返回 `nextBefore`，更早的历史仍能取到 | 同一用例：窗口只有那一条不合法行（整页都不合法）时仍返回 `nextBefore=2`，更早的 1 由 `before=2` 取到 | Covered |
| 字节触顶时 `hasMore` 仍由独立探测决定 | `TestWorkflowNodeSessionReadStopsAPageAtItsByteBudget`：20 条 250 KiB 记录在 `limit=200` 下被 4 MiB 预算提前截断，`hasMore` 为真且 `nextBefore` 指向本页最早记录；下一页取到剩余记录 | Covered |
| 单页受条数与字节双上限，跨过预算的那一条整条计入、不多返回半条 | 同一用例证明跨过预算的那一条整条返回（首条正文长度等于存储值）且该页不标 `incomplete`；条数上限由 `router.go` 的 `maxLimit = core.NodeSessionPageMax` 实现，路由测试只证明 200 被接受，真实栈（#720）实测 `limit=201` 返回 400 `invalid_pagination`。尚缺精确页字节上限的直接断言 | Partial |
| 终态回执不参与分页计数与游标 | `TestWorkflowNodeSessionReadExcludesTheTerminalReceiptBySequenceNotShape`：终态回执不投影，`records` 为 1、2，`hasMore` 假、无 `nextBefore`、`state` 为 `available` | Covered |

- 决策依据：D1，不变量 6。

## The Old Attempt Never Claims The Current Output

- 风险：请求旧尝试时把节点当前的 `status` / `output` / `sessionEndReason` / `error` 当作所选尝试的结果，旧尝试
  详情显示新尝试的成功状态或回答。
- 前置状态：第 1 次尝试已终态（`result` 非空，如可重试失败），第 2 次尝试 `running` 或已成功；节点当前
  `output` / `sessionEndReason` 属于第 2 次尝试。
- 触发：`?attempt=1` 读取。
- 必须成立：`execution` 块只含所选尝试的执行证据（`attempt`、`executionId`、`result`、`records`）；`node` 块是
  节点当前状态（`status`、`attempt`、`output`、`sessionEndReason`、`error`）；仅当
  `execution.attempt == node.attempt` 时 `node.output` / `node.sessionEndReason` 才是所选尝试的最终回答与结束
  原因；请求旧尝试时它们仍描述节点当前状态，不冒充旧尝试结果；旧尝试的终态证据是它自己的 `execution.result`
  （`reason` 即那次结束原因），最终回答由前端按 `records` 里 `turnEnded` 前最后一条 `agent_message_chunk`
  重放得到。
- 禁止结果：旧尝试详情显示新尝试的成功状态或回答；旧尝试的 `output` 被清空或被当前 `output` 覆盖。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `execution` 与 `node` 分开，旧尝试不冒领最新 `output` | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadNeverPresentsTheCurrentAttemptAsAnOldOne`：读 `attempt=1` 时 `node` 块是第 2 次的 `succeeded`、`output` 与 `sessionEndReason=idle_timeout`，`execution.result.reason` 是第 1 次自己的 `agent_failed`；真实栈（#720，run `6deea532…`）读第 1 次尝试得到同样的分离：`node` 块是第 2 次的 `succeeded` / `idle_timeout`，`execution.result.reason=agent_failed` | Covered |
| `execution.attempt == node.attempt` 时 `node.output` 才是所选尝试的回答 | 同一用例的缺省读取（当前尝试）断言 `execution.attempt == node.attempt`，此时 `node.output` 是第 2 次自己的回答 | Covered |
| 旧尝试的最终回答从 `records` 重放、不借用当前 `output` | [视图测试][view] 的 `shows the end reason only when the node state describes the attempt being read` 证明节点状态不属于所选尝试时结束原因不呈现，[面板测试][panel] 证明回答逐行来自 `records`。尚缺「两次尝试、旧尝试的最终回答取自第 1 次记录而非第 2 次 `node.output`」的直接渲染断言 | Partial |

- 决策依据：D1，不变量 4，重试决策 D3。

## Terminal Receipts Are Excluded By Committed Evidence Not By Shape

- 风险：用「是 JSON 对象且有 `record`」证明某行是历史回执，正常结束的会话在终态回执处解析失败或被误报成历史
  缺口；`result` 为空时把不存在的终态回执当成历史回执。
- 前置状态：一次会话既有历史回执又有终态回执（如 `"session-terminal-event"` / `"late-original-terminal"`）；
  终态 `event` 允许是任意非空字符串，可能恰好也是含 `record` 的 JSON 对象。
- 触发：读取该尝试的 `records`。
- 必须成立：`result` 非空时，终态回执就是 `sequence = node_executions.last_event_sequence` 那一行，读取端按
  序号排除它，其余 `sequence < last_event_sequence` 的行才是历史回执；`result` 为空时该执行没有终态回执
  （终态回执只由 `nodeExecutionResult` 与 `result` 在同一事务写入），所有回执都是历史回执；内容形状
  （`event IS JSON OBJECT` 且 `event->>'record'` 可解析为 HistoryRecord）只用于**校验**历史回执、不用于**证明**
  来源；形状不合法的历史回执标 `incomplete`，不静默丢弃。
- 禁止结果：终态回执被投影进 `records` 或解析失败；用内容形状判定终态回执；形状不合法的历史回执被静默丢弃。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `result` 非空时终态回执按 `last_event_sequence` 排除 | `TestWorkflowNodeSessionReadExcludesTheTerminalReceiptBySequenceNotShape`（终态序号 3 被排除，`records` 为 1、2）与 `TestWorkflowNodeSessionReadScopesToTheRequestedAttempt`（第 1 次已结算，`records` 为 1、2、3，不含序号 4 的终态回执） | Covered |
| 含 `record` 形状的终态 `event` 仍按序号排除、不被当历史 | `ExcludesTheTerminalReceiptBySequenceNotShape` 的终态 `event` 本身就是含 `record` 的 JSON；读取仍按 `last_event_sequence` 排除它，且未把该页标 `incomplete` | Covered |
| `result` 为空时全部回执作为历史回执投影 | `PagesEveryRecordWithoutSkipping` 与 `ReportsAGapOnlyOnThePageThatObservesIt` 的尝试都未结算（无 `result`、无终态回执），回执 1..5 / 1,2,4,5 全部作为历史回执返回 | Covered |
| 形状不合法的历史回执标 `incomplete` 而非静默丢弃 | `ReportsUnshapedHistoryWithoutStrandingTheRest`：不合法行不进 `records`，该页标 `incomplete` 并给出仍能翻到更早历史的 `nextBefore` | Covered |

- 决策依据：D1，不变量 5、8。

## Authorization Scopes To The Workflow Run Node Without A SessionId Bypass

- 风险：直读 `execution_id` / `session_id` / 产物地址绕过（运行，节点，尝试）作用域；跨节点、跨运行、跨租户读；
  归档工作流的历史仍可读。
- 前置状态：请求者是活跃成员、非成员或其他租户；workflow 归档或未归档；`nodeId` 属于或不属于该运行。
- 触发：`GET` 该接口，`nodeId` 属于其他运行或不存在；尝试直读 execution_id / session_id；归档 workflow。
- 必须成立：授权与 `workflowModelRead`（`session_model_statistics.go`）同一套——`sessionReadIdentity` + 活跃租户
  成员；workflow 归档门禁（归档工作流连同运行历史 404）；运行必须属于该租户、该 workflow 且未删除；节点必须
  属于该运行（从冻结图解析 `nodeId`，不存在或不属于该运行 404）；不提供按 execution_id / session_id / 对象键
  直读的路由；未知字段按边界拒绝。
- 禁止结果：成员跨节点/跨运行读；sessionId 直读绕过作用域；归档工作流历史仍可读；非成员读。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 节点不属于该运行或不存在时 404，不泄露其他运行数据 | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadStaysOnTheWorkflowRunNode`：冻结图里没有的 `nodeId` 与拿 `executionId` 当节点都 404；节点从运行的冻结快照解析，作用域不会落到别的运行 | Covered |
| 归档工作流连同运行历史 404 | 同一用例把 workflow 标记 `deleted_at`（cloud 的归档/删除门禁：`workflow()` 只认 `deleted_at IS NULL`）后读取 404 | Covered |
| 活跃成员可读，非成员 403 / 其他租户 404 | 同一用例：活跃非 admin 成员读到 200 与记录 1、2；租户外调用者得到 `membership_required`；同一节点在另一租户下 404 | Covered |
| 无 execution_id / session_id / 对象键直读路由 | [路由契约测试][router] 的 `TestNodeSessionQueryAllowlist` 断言该路径只有一条无 action 的公开 GET、路径不含 `:sessionId` / `:executionId` / `:attempt`，查询白名单拒绝 `sessionId`、`executionId`、`path`、`after` | Covered |

- 决策依据：D2，不变量 2、3。

## Output And Session End Reason Are Presented Separately

- 风险：结束原因占用 `output` 槽位，下游 Output 节点或 Issue `run.message` 拿到 `reason` 而非回答；或结束原因
  丢失；或把失败中的部分文本误发布为最终 `output`。
- 前置状态：一个成功结算的 agent 节点（成功回合，有最终回答与结束原因）；一个失败/取消结算的 agent 节点（有
  结束原因、`error` 与部分会话记录，但没有已提交的最终 `output`）。
- 触发：读取这些节点详情。
- 必须成立：`output` 只对成功结算的节点是最终可见回答（成功回合才写 `{agent}.output`、落 `output` 根，不因
  读取被清空）；`sessionEndReason` 独立呈现结束原因（Node 报告的 `user_ended` / `idle_timeout` / `cancelled` /
  `agent_failed` / `interrupted`；Cloud 会话期结算的诊断 `node_unavailable` /
  `session_turn_deadline_exceeded` / `session_start_deadline_exceeded` / `session_end_deadline_exceeded` / 取消
  结算的 `cancelled`，以及 `agent_failed` + `detail` 提升的 `agent_plugin_unavailable`；Cloud 启动期结算的
  `workflow_run_project_required` / `workspace_unavailable` / `agent_plugin_unavailable` /
  `agent_plugin_version_unfrozen` / `agent_prompt_template_invalid` / `workflow_executor_unconfigured`）。
  `sessionEndReason` 不是闭集：Cloud 增加新诊断不需要改读取端，读取端只呈现、不新增也不改写取值；
  失败/取消节点没有已提交的最终 `output`，只有
  `sessionEndReason`、`error` 与部分会话记录；`error` 只在失败节点携带公开失败原因；Cloud 对记录业务字段的
  解释限于最终回答归集（成功回合 `turnEnded` 边界前最后一条完整的 `agent_message_chunk` 或其 `visibleText`
  投影），其余记录按原样透传由前端渲染。
- 禁止结果：输出与结束原因混在一个字段；失败/取消节点的部分文本被误发布为最终 `output`；成功节点的 `output`
  被清空；Cloud 泛化解析全部记录业务字段。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 成功节点读到最终回答与独立 `sessionEndReason` | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadNeverPresentsTheCurrentAttemptAsAnOldOne`：成功结算的第 2 次尝试 `node.output` 是最终回答（`{"output":"Inspected the files"}`），`node.sessionEndReason` 是独立的 `idle_timeout`；真实栈（#720，run `a4bb5182…`）的两个节点上 `node.output.output` 与 `node.sessionEndReason=idle_timeout` 也各自独立 | Covered |
| 失败/取消节点不发布最终 `output`，只有 `sessionEndReason`、`error` 与部分会话记录 | `TestWorkflowNodeSessionReadReportsACloudSettlementAsUnavailableNotInFlight`：`node_unavailable` 结算的节点 `status=failed`、`sessionEndReason` 与 `error` 都是 `node_unavailable`、`output` 为空；真实栈（#720，run `6deea532…`）的第 1 次尝试 `agent_failed`：`output` 为空、`sessionEndReason` 与 `error` 都是 `agent_failed`，读取该尝试得 `state=unavailable`、0 条记录。尚缺取消结算与「失败节点带部分会话记录」各一例 | Partial |
| Cloud 自造的启动期与会话期原因原样呈现为 `sessionEndReason`，读取端不新增或改写取值 | 同一用例覆盖会话期的 `node_unavailable`；真实栈（#720）覆盖会话期 `agent_failed` 与 `idle_timeout` 的原样呈现。尚缺启动期原因（如 `workflow_run_project_required`、`workspace_unavailable`）与各 deadline / 取消结算的读取用例 | Partial |
| 读取接口不修改 `node_states`（`output` 不被清空或改写） | `TestWorkflowNodeSessionReadIsReadOnly` 读取前后比较 `workflow_runs.node_states` 的完整 JSON | Covered |
| Cloud 只解释最终回答，其余记录原样透传 | 记录投影只挑 `sequence`、`record` 与 canonEvent 的可选字段；用例读回的记录内容（回答文本、`gap` 类型、`truncated` 标记、外层序号）与写入一致。尚缺 `historySequence` / `visibleText` / `visibleTextTruncated` 与存储回执逐字段相等的断言 | Partial |

- 决策依据：D3，不变量 7、8，根决策变量根，#673 owner 结论。

## Empty States Follow The Attempt Terminal Evidence And Result Empty Is Not In Flight

- 风险：把 `result` 为空当作在途，把「Node 消失 / 会话超时 / 已取消」的零历史尝试误标成「等待首条」；把未派发、
  等待首条、终态无历史、不适用混为一谈。
- 前置状态：各空态场景——(a) 所选尝试没有 `agent_session` 执行行；(b) 有执行行、`result` 为空、
  `node.attempt == execution.attempt` 且 `node.status` 为 `running`、零历史回执；(c) 有执行行、零历史回执、终态
  证据成立（`result` 非空；或 `node.attempt == execution.attempt` 且 `node.status` 终态）——包括 Cloud 因
  `node_unavailable` / `session_turn_deadline_exceeded` / `session_end_deadline_exceeded` /
  `session_start_deadline_exceeded` / 取消结算的失败或取消节点（`result` 为空但 `node.status` 终态）；(d) 有执行
  行、`result` 为空、零历史回执、无法证明终态（如 `node.attempt > execution.attempt` 却 `result` 为空）；(e) 非
  agent 节点。
- 触发：读取这些场景。
- 必须成立：未派发 →「未建立会话」；在途零历史 →「等待首条记录」；终态零历史 →「历史不可用」（沿用 #296 的
  `unavailable`）；证据缺失 →「未知」；非 agent 节点 →「不适用」；有结果、缺记录 → 分别展示结果与「历史不完整
  ／不可用」；这些状态位由读取接口返回，节点状态不新增字段（仍是既有的 `status` / `output` /
  `sessionEndReason` / `error` 等）。
- 禁止结果：把 `result` 为空的 Cloud 终态路径标成「在途」；把零回执的在途会话标成「历史不可用」；用空列表掩盖
  数据缺失。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| Cloud 终态结算（无 `result`）判「终态、无历史」，不判「在途」 | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadReportsACloudSettlementAsUnavailableNotInFlight`：Node 心跳失效后 Cloud 不写 `result` 就结算，读取 `state=unavailable`、`result` 为空、`node.status=failed`。尚缺各 deadline 与取消结算的零历史用例 | Partial |
| 在途零历史判「等待首条记录」 | `TestWorkflowNodeSessionReadReportsAnInFlightAttemptAwaitingItsFirstRecord`：已派发、零回执的在途尝试 `state=awaiting_first_record`，无记录，`incomplete` 与 `hasMore` 都为假 | Covered |
| 未派发、非 agent 节点、证据缺失分别判「未建立会话」「不适用」「未知」 | `TestWorkflowNodeSessionReadDistinguishesNotApplicableAndNotStarted`（未派发 `not_started`、`start` 节点 `not_applicable`）与 `TestWorkflowNodeSessionReadReportsContradictoryEvidenceAsUnknown`（`node.attempt > execution.attempt` 却无 `result` 判 `unknown`） | Covered |
| 有结果缺记录分别呈现结果与「不完整/不可用」 | `TestWorkflowNodeSessionReadReportsAGapOnlyOnThePageThatObservesIt`、`...ReportsANodeGapRecordAsIncomplete`、`...ReportsATruncatedRecordAsIncomplete`、`...ReportsAHistoryMissingItsBeginning` 各自证明页标 `incomplete` 且记录不被丢弃，但这些尝试都未结算（`result` 为空）。尚缺「`result` 非空 + `Gap` / 截断 / 形状不合法」的直接用例 | Partial |

- 决策依据：D4，不变量 9，cloud `workflow_run_session.go` 的终态路径，#296 D4 的空态词汇。

## Incomplete Is A Per-Page Observation Until All Pages Are Read

- 风险：单页没有看到缺口就断言整次历史完整；从尚未读齐的页推定旧尝试的最终回答；`incomplete` 与 `hasMore`
  混为一谈。
- 前置状态：一次尝试的历史在某页存在 `Gap` / 截断 / 形状校验失败；或较早页有缺口而当前页没有；或所选尝试的
  最早历史回执序号大于 1（开头已丢失，翻页看不到）。
- 触发：逐页读取全部 `records`。
- 必须成立：`incomplete` 表示「本页观测到历史无法完整呈现」（本页投影时出现 `Gap` / 截断 / 形状校验失败，或
  所选尝试的最早历史回执序号大于 1），不是整次尝试的完整性结论；Cloud 不从单页断言整次完整；前端在读齐
  全部页且各页都无 `incomplete` 前保持「完整性未知」，任一页有即整体不完整；`hasMore` 表示「还有更早一页
  可读」，与 `incomplete` 独立；响应内 `records` 按外层 `sequence` 升序，旧尝试的最终回答只能从读齐的
  `records` 重放，不从尚未读齐的页提前推定。
- 禁止结果：首页无缺口即宣称整次历史完整；旧尝试最终回答从第一页推断；`incomplete` 与 `hasMore` 语义混淆。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `incomplete` 只在观测到缺口的页为真，无缺口的页不为真 | `TestWorkflowNodeSessionReadReportsAGapOnlyOnThePageThatObservesIt` 逐页断言：缺口之上的页与缺口之下的页 `incomplete` 为假，跨越缺口的页为真；`ReportsANodeGapRecordAsIncomplete` 与 `ReportsATruncatedRecordAsIncomplete` 也证明不含该记录的页不为真 | Covered |
| 读齐全部页且各页无 `incomplete` 前，前端不宣称整次完整 | [视图测试][view] 的 `keeps a gap any page observed` 证明多页合并保留任一页的缺口，[面板测试][panel] 只渲染已加载页并给出「可能不完整」告示。尚缺「只读首页时完整性未知」的直接断言：当前面板没有单独表达完整性未知的文案 | Partial |
| 旧尝试最终回答不从尚未读齐的页提前推定 | 视图只从已加载页重建对话，`hasMore` 时提供「加载更早的记录」，面板渲染的回答逐行来自 `records`。尚缺「未读齐时不渲染最终回答」的直接断言 | Partial |

- 决策依据：D1、D4，不变量 6。

## The Read Is Read-Only And Never Opens A Session

- 风险：读取放出了命令、隐式启用 HITL 或开启新对话，推进了节点状态或会话。
- 前置状态：一个运行中的 agent 节点会话。
- 触发：反复 `GET` 该接口（含带 `attempt` / `before` / `limit` 的翻页）。
- 必须成立：接口是纯只读：不推进节点、不放出任何命令、不隐式启用 HITL、不开启新对话；`node_states`、
  `node_executions`、`node_event_receipts` 的持久化形状与值不变；翻页幂等。
- 禁止结果：读取后节点状态 / 执行行 / 回执改变；隐式注入命令或开启新对话；读取推进了会话结束。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 读取前后节点状态、执行行与回执不变 | [读取 DB 用例][db] 的 `TestWorkflowNodeSessionReadIsReadOnly`：两次成功读取、两次等价重复与两个失败请求前后，完整比较 `workflow_runs.node_states`、`node_executions` 与 `node_event_receipts` 的 JSON | Covered |
| 反复翻页幂等，不产生命令或执行工作项 | 同一用例同时比较 `thread_commands` 与 `execution_work` 的计数 | Covered |
| 不隐式开启新对话或 HITL | 同一用例证明读取没有新增命令或工作项（隐式开启新对话或 HITL 都会留下它们）；[路由契约测试][router] 证明该路径只有一条无 action 的公开 GET | Covered |

- 决策依据：D2，不变量 2，根决策 D2 的 `interactive` 门禁，#296 D4 的「只读不隐式注入」。

[db]: https://github.com/ora-space/cloud/blob/63034cfcad687845ee9fdc498449f481e6eb0f9e/internal/core/workflow_node_session_db_test.go
[router]: https://github.com/ora-space/cloud/blob/63034cfcad687845ee9fdc498449f481e6eb0f9e/internal/api/router/router_test.go
[view]: https://github.com/ora-space/cloud/blob/63034cfcad687845ee9fdc498449f481e6eb0f9e/frontend/src/features/workflows/run/run-node-session-view.test.ts
[panel]: https://github.com/ora-space/cloud/blob/63034cfcad687845ee9fdc498449f481e6eb0f9e/frontend/src/features/workflows/run/run-node-session.test.tsx
