# 工作流节点会话读取

依据[工作流节点会话读取](../../../decisions/cloud/workflow/20261026-workflow-node-session-read.md)
（`approved`，D1–D4）。这些用例覆盖按（运行，节点，尝试）有界分页读取节点会话记录、终态回执按已提交证据
权威排除、旧尝试不冒领最新输出、输出与结束原因分开、授权下探到节点，以及按所选尝试终态证据与历史回执判定
的空态。调度串行与恢复见 [调度串行与恢复](scheduler-serialism-and-recovery.md)，单个尝试内的会话结束见
[工作流会话结束与有界等待](session-end-after-initial-turn.md)，重试隔离见
[工作流节点重试与恢复](retry-and-resume.md)。

实现由 [ora-space/circus#691](https://github.com/ora-space/circus/issues/691) 承接，尚未合入 main，cloud 侧义务
全部为 `Missing`，随实现逐项登记直接测试；前端节点详情只渲染读取接口返回的字段，不能代替 Cloud 在真实
PostgreSQL 上的证据。`output` / `sessionEndReason` 的语义由 cloud#182（Draft 候选）定义，读取端只消费。

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
| `attempt` 越界或不属于该节点 404，缺省读最新尝试 | 真实 PostgreSQL：同一节点两次尝试，分别读 `attempt=1`、缺省、`attempt=3` | Missing |
| `before` 不带 `attempt` 返回 400 | 接口测试：只带 `before` 的请求 | Missing |
| 翻页期间重试，下一页仍读原尝试的记录 | 真实 PostgreSQL：首页后插入第 2 次尝试执行，带原 `attempt` 翻页，记录仍属第 1 次 | Missing |

- 决策依据：D1，不变量 1、4，重试决策 D4，会话结束决策不变量 2。

## A Pagination Cursor Never Skips A Record

- 风险：`nextBefore` 指向未返回记录导致某条记录被永久跳过；`hasMore` 靠本页是否填满导致字节触顶时误判无更多；
  单页字节无上限形成无界响应。
- 前置状态：一次尝试有 5 条历史回执（外层序号 5、4、3、2、1），或存在使单页字节先触顶的长记录。
- 触发：`limit=2` 的首页；字节先触顶的一页；带 `before=<nextBefore>` 的下一页。
- 必须成立：`nextBefore` 固定为本页最早一条**已返回**历史回执的外层 `sequence`；下一页 `before=<nextBefore>`
  取 `sequence < before`，每条记录都被后续页面覆盖；`hasMore` 由独立探测（是否存在 `sequence < nextBefore`
  的历史回执）决定，条数触顶与字节触顶两种情况都成立；单页受条数（≤ 200）与字节（`maxPageBytes`，4 MiB）
  双上限，任一先触顶处停止；`nextBefore` 仅当存在更早历史回执时返回；终态回执被跳过，不计入 `limit`、
  不产生 `hasMore`、不参与 `nextBefore`。
- 禁止结果：某条记录被跳过；字节触顶时 `hasMore` 误报为无；单页超过 200 条或 4 MiB；分页折叠或截断记录正文。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `nextBefore` 指向本页最早已返回序号，逐页覆盖全部记录 | 真实 PostgreSQL：5 条记录 `limit=2` 逐页，序列 5,4 / 3,2 / 1，不漏 | Missing |
| 字节触顶时 `hasMore` 仍由独立探测决定 | 真实 PostgreSQL：构造单页字节先触顶的数据，`hasMore` 正确 | Missing |
| 单页受条数与字节双上限，超出即停止 | 真实 PostgreSQL：`limit=200` 与超长记录，单页响应在 4 MiB 内 | Missing |
| 终态回执不参与分页计数与游标 | 真实 PostgreSQL：含终态回执的会话，`limit`、`hasMore`、`nextBefore` 只看历史回执 | Missing |

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
| `execution` 与 `node` 分开，旧尝试不冒领最新 `output` | 真实 PostgreSQL：两次尝试，读 `attempt=1`，`node.output` 仍描述当前尝试、`execution.result` 属第 1 次 | Missing |
| `execution.attempt == node.attempt` 时 `node.output` 才是所选尝试的回答 | 真实 PostgreSQL：读当前尝试，两者一致 | Missing |
| 旧尝试的最终回答从 `records` 重放、不借用当前 `output` | 前端两次尝试渲染测试：读 `attempt=1` 渲染的回答来自第 1 次的 `agent_message_chunk`，不是第 2 次的 `node.output` | Missing |

- 决策依据：D1，不变量 4，重试决策 D3。

## Terminal Receipts Are Excluded By Committed Evidence Not By Shape

- 风险：用「是 JSON 对象且有 `record`」证明某行是历史回执，正常结束的会话在终态回执处解析失败或被误报成历史
  缺口；`result` 为空时把不存在的终态回执当成历史回执。
- 前置状态：一次会话既有历史回执又有终态回执（如 `"ended"` / `"session-terminal-event"`）；终态 `event` 允许是
  任意非空字符串，可能恰好也是含 `record` 的 JSON 对象。
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
| `result` 非空时终态回执按 `last_event_sequence` 排除 | 真实 PostgreSQL：正常结束会话，`records` 只含历史回执 | Missing |
| 含 `record` 形状的终态 `event` 仍按序号排除、不被当历史 | 真实 PostgreSQL：终态回执是含 `record` 的 JSON，`records` 不含它 | Missing |
| `result` 为空时全部回执作为历史回执投影 | 真实 PostgreSQL：在途会话或 Cloud 终态结算（无 result）的历史回执 | Missing |
| 形状不合法的历史回执标 `incomplete` 而非静默丢弃 | 真实 PostgreSQL：插入形状不合法的历史回执，响应标 `incomplete` | Missing |

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
| 节点不属于该运行或不存在时 404，不泄露其他运行数据 | 真实 PostgreSQL：跨节点、跨运行请求 | Missing |
| 归档工作流连同运行历史 404 | 真实 PostgreSQL：归档后读该工作流运行 | Missing |
| 活跃成员可读，非成员 403 / 其他租户 404 | 接口测试：四类请求者 | Missing |
| 无 execution_id / session_id / 对象键直读路由 | OpenAPI 漂移检查：路由清单 | Missing |

- 决策依据：D2，不变量 2、3。

## Output And Session End Reason Are Presented Separately

- 风险：结束原因占用 `output` 槽位，下游 Output 节点或 Issue `run.message` 拿到 `reason` 而非回答；或结束原因
  丢失；或把失败中的部分文本误发布为最终 `output`。
- 前置状态：一个成功结算的 agent 节点（成功回合，有最终回答与结束原因）；一个失败/取消结算的 agent 节点（有
  结束原因、`error` 与部分会话记录，但没有已提交的最终 `output`）。
- 触发：读取这些节点详情。
- 必须成立：`output` 只对成功结算的节点是最终可见回答（成功回合才写 `{agent}.output`、落 `output` 根，不因
  读取被清空）；`sessionEndReason` 独立呈现结束原因（`user_ended` / `idle_timeout` / `cancelled` /
  `agent_failed` / `interrupted` 或 Cloud 自己的诊断）；失败/取消节点没有已提交的最终 `output`，只有
  `sessionEndReason`、`error` 与部分会话记录；`error` 只在失败节点携带公开失败原因；Cloud 对记录业务字段的
  解释限于最终回答归集（成功回合 `turnEnded` 边界前最后一条完整的 `agent_message_chunk` 或其 `visibleText`
  投影），其余记录按原样透传由前端渲染。
- 禁止结果：输出与结束原因混在一个字段；失败/取消节点的部分文本被误发布为最终 `output`；成功节点的 `output`
  被清空；Cloud 泛化解析全部记录业务字段。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 成功节点读到最终回答与独立 `sessionEndReason` | 真实 PostgreSQL：成功结算节点，`output` 与 `sessionEndReason` 分开 | Missing |
| 失败/取消节点不发布最终 `output`，只有 `sessionEndReason`、`error` 与部分会话记录 | 真实 PostgreSQL：失败与取消各一例，`output` 为空、原因与错误独立 | Missing |
| 结束原因闭集，Cloud 诊断（`node_unavailable` 等）作为 `sessionEndReason` 呈现 | 真实 PostgreSQL：Cloud 终态结算的节点 | Missing |
| 读取接口不修改 `node_states`（`output` 不被清空或改写） | 真实 PostgreSQL：读取前后 `node_states` 不变 | Missing |
| Cloud 只解释最终回答，其余记录原样透传 | 真实 PostgreSQL：`records[i].record` 的内层字段与回执 `event.record` 解析结果一致，外层 `sequence` / `turnId` / `truncated` 分别与 `node_event_receipts` 对应列一致 | Missing |

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
  ／不可用」；这些状态位由读取接口返回，节点状态本身继续只有 `status` / `output` / `sessionEndReason` /
  `error`。
- 禁止结果：把 `result` 为空的 Cloud 终态路径标成「在途」；把零回执的在途会话标成「历史不可用」；用空列表掩盖
  数据缺失。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| Cloud 终态结算（无 `result`）判「终态、无历史」，不判「在途」 | 真实 PostgreSQL：`node_unavailable`、各 deadline、取消结算的零历史节点 | Missing |
| 在途零历史判「等待首条记录」 | 真实 PostgreSQL：`running` 且零历史回执的节点 | Missing |
| 未派发、非 agent 节点、证据缺失分别判「未建立会话」「不适用」「未知」 | 真实 PostgreSQL：三类场景各一例 | Missing |
| 有结果缺记录分别呈现结果与「不完整/不可用」 | 真实 PostgreSQL：`result` 非空但历史回执有 Gap/截断/形状不合法 | Missing |

- 决策依据：D4，不变量 9，cloud `workflow_run_session.go` 的终态路径，#296 D4 的空态词汇。

## Incomplete Is A Per-Page Observation Until All Pages Are Read

- 风险：单页没有看到缺口就断言整次历史完整；从尚未读齐的页推定旧尝试的最终回答；`incomplete` 与 `hasMore`
  混为一谈。
- 前置状态：一次尝试的历史在某页存在 `Gap` / 截断 / 形状校验失败；或较早页有缺口而当前页没有。
- 触发：逐页读取全部 `records`。
- 必须成立：`incomplete` 表示「本页观测到历史无法完整呈现」（本页投影时出现 `Gap` / 截断 / 形状校验失败），
  不是整次尝试的完整性结论；Cloud 不从单页断言整次完整；前端在读齐全部页且各页都无 `incomplete` 前保持
  「完整性未知」，任一页有即整体不完整；`hasMore` 表示「还有更早一页可读」，与 `incomplete` 独立；旧尝试的
  最终回答只能从读齐的 `records` 重放，不从尚未读齐的页提前推定。
- 禁止结果：首页无缺口即宣称整次历史完整；旧尝试最终回答从第一页推断；`incomplete` 与 `hasMore` 语义混淆。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `incomplete` 只在观测到缺口的页为真，无缺口的页不为真 | 真实 PostgreSQL：缺口在较早页，首页 `incomplete` 为假 | Missing |
| 读齐全部页且各页无 `incomplete` 前，前端不宣称整次完整 | 前端测试：只读首页时完整性未知，读齐后确定 | Missing |
| 旧尝试最终回答不从尚未读齐的页提前推定 | 前端测试：旧尝试 `records` 未读齐时不渲染最终回答 | Missing |

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
| 读取前后节点状态、执行行与回执不变 | 真实 PostgreSQL：读取前后比较 `node_states` 与 `node_event_receipts` | Missing |
| 反复翻页幂等，不产生命令或执行工作项 | 真实 PostgreSQL：翻页后无 `thread_commands` / `execution_work` 新增 | Missing |
| 不隐式开启新对话或 HITL | 契约测试：路由只读，无命令副作用 | Missing |

- 决策依据：D2，不变量 2，根决策 D2 的 `interactive` 门禁，#296 D4 的「只读不隐式注入」。
