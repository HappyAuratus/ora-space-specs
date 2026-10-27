---
status: approved
---

# 工作流节点会话读取：按尝试有界读取节点会话记录，输出与结束原因按 cloud#182 口径分开呈现

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策，只固定
「运行中和终态运行如何读到某一个 agent 节点本次尝试的完整会话」这一接缝。结论：Cloud 新增一个公开只读接口，
按（WorkflowRun，节点，尝试）把 `node_executions` 的会话证据与 `node_event_receipts` 的会话记录有界、分页地
呈现为节点会话历史；节点状态把「结束原因」与「输出」分开，其中 `output` 是最终可见回答、结束原因独立成
`sessionEndReason`——这套分离由 cloud#182（承接 ora-space/circus#676，已于 2026-10-27 合并，merge commit
`f433499b`）实现，本文继承它的口径、不重新定义。读取沿用 `workflowModelRead` 的授权模板下探到节点，不新增 sessionId
或产物地址直读；响应把「所选尝试的执行证据」与「节点当前状态」分开，旧尝试不冒领最新输出；终态回执按已提交
终态证据的序号权威排除、不由内容形状判定；运行中零历史回执、终态无历史、非 agent 节点各有真实空态，分页与
截断都有显式状态，不用空列表或静默截断掩盖数据缺失。

**状态与影响面**：本文为 `approved`，方向已经 owner 批准（ora-space/circus#691）；cloud#182 已合并，读取实现由
后续实现运行落地。涉及 cloud 的公开读取路由与 OpenAPI 新增接口，以及前端
节点详情的会话渲染。**兼容边界**：只新增接口，不改 `node_executions` / `node_event_receipts` 的持久化形状与
既有约束，不新增迁移；`nodeStates` 的 `output` / `sessionEndReason` 分离由 cloud#182 实现，本文只消费这
两个字段、不重新定义。**行为变化**：agent 节点详情从「只显示 `{"reason":...}`」变为「输出与结束原因分开、完整
会话按尝试分页可读」——这是修复缺陷，不是回归。**非目标**：不新增会话写入、不改变会话的结束与结算路径、
不新增取消或 HITL 入口、不读取 Workspace 里的文件与 diff（那是 #296 的成果契约）；不把 Cloud 升级为解释
`ora-history` 记录业务字段的通用读取端——除最终回答的窄例外（见 D3）外，记录仍按原样透传，由前端沿用既有
`HistoryRecord` 反序列化渲染。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D4：节点终态只来自会话执行结果（`node_executions.result`）；崩溃恢复只按已提交结果重算 | 继承，并成为读取前提：会话证据与记录都挂在已提交的执行行上，读取不推进节点、不重算 |
| 根决策 D4 / D2：agent 节点首期串行，发布门禁拒绝 `interactive: true`，同一 run 的 agent 会话是单回合 | 继承：读取端看到的是一次单回合会话的输入与结果，不出现用户后续轮次 |
| 根决策「变量根的声明范围」：Agent 声明 `output` 根，Output 节点从这个根取值；节点产出首期落在节点状态里供详情展示 | 继承，并成为 D3 的前提：`output` 是最终可见回答，不因本文被清空 |
| 根决策「实现与 approved 方案的差异」已知缺口（Agent 回答没写进 `output` 根） | 已由 cloud#182 修复（ora-space/circus#673、#676，2026-10-27 合并）；本文读取端消费修复后的 `output`，不重复修复 |
| [会话结束后续决策](20261019-workflow-session-end-after-initial-turn.md) D1–D5：成功回合后 Cloud 按节点唯一放出 `end_session`，结束原因取 `idle_timeout`；节点终态仍只来自会话结果 | 继承结束原因词汇：`user_ended` / `idle_timeout` / `cancelled` / `agent_failed` / `interrupted` 及 Cloud 自己的诊断是结束原因的闭集；本文把它们作为 `sessionEndReason` 呈现，不新增原因 |
| [节点重试](20261021-node-retry-and-resume-contract.md) D4：每次尝试有独立的 `node_executions` 行，按尝试唯一的会话证据与结束命令 | 继承，并成为读取的作用域：一次读取只落到一个尝试的执行行，重试后能区分每次尝试，切节点/切运行/刷新不串用会话 |
| 节点重试 D1：`attempt` 是 Cloud 内部路由字段，不来自线上请求 | 继承：读取按（运行，节点，尝试）解析执行行，尝试序号来自已提交节点状态，不接受调用方指定 execution_id |
| 节点重试 D3：下一次启动把节点状态改为 `running`、`attempt+1`、清掉旧错误，最终被新回答覆盖 | 继承：`node_states` 里 `status` / `output` / `sessionEndReason` / `error` 只有当前尝试一份；本文把「节点当前状态」与「所选尝试执行证据」分开呈现（见 D1），旧尝试不冒领最新 `output` |
| cloud#182（`fix/676-agent-output-template`，已于 2026-10-27 合并，head `42b1b91`，merge commit `f433499b`）：成功结算时把初始回合最终的完整助手回答写进 `output`（`{"output": text}`），截断则失败；结束原因另存 `sessionEndReason`，不占用 `output`；用 `event IS JSON OBJECT` 保护终态回执的解析 | 继承，不改写；实现前置已满足（cloud#182 先合并），本文读取端消费 `output` / `sessionEndReason`；历史回执的筛选与 cloud#182 同一方向，但判别以终态序号为准（见 D1） |
| [运行拥有成果决策](20261025-run-owned-git-delivery-and-history.md) D4：完整归档的读取继承原运行授权，本文只交付保存与可验证引用，不增加完整节点日志 UI | 边界切分：节点会话读取/UI 由本文承接，不属于 #296；`not_started` / `unavailable` 的空态词汇沿用 #296 D4 |
| `workflowModelRead`（`session_model_statistics.go`）：sessionReadIdentity + 成员 + workflow 归档门禁 + run∈workflow/tenant | 继承授权模板，扩展到节点级作用域；不引入 sessionId 直读 |
| `SessionHistory`（`session_history.go`）的读取边界：整次历史 32 MiB / 10 万条上限、超限返回 413 | 继承「整次读取必须有界」的原则；但节点会话读取用「分页 + 单页字节上限」保证每次响应有界，不复用它的 413/截断态（见 D1） |

## 决策

### D1：公开读取按（运行，节点，尝试）有界、分页地呈现该次尝试的会话记录

新增只读接口 `GET /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/nodes/:nodeId/session`，参数：

- `?attempt=N`（可选）：读第 N 次尝试；缺省读该节点当前/最新尝试；越界或不属于该节点 404。
- `?before=<sequence>`（可选）：取该尝试中外层 `sequence < before` 的一页历史回执，按降序；缺省从最新往回取。**只与 `attempt` 配对使用**：带了 `before` 却不带 `attempt` 时 400——`before` 只有钉在具体尝试上才有意义。
- `?limit=N`（可选）：一页条数，缺省 100、上限 200；与字节上限共同限界（见下）。

响应固定两块，把「所选尝试的执行证据」与「节点当前状态」分开，**旧尝试不冒领最新输出**：

- **`execution`（所选尝试的执行证据，本次读取的主体）**：`attempt`（服务端解析后回显，翻页必须原样带回）、
  `executionId`、`result`（该执行行 `node_executions.result` 的 `{outcome, reason, detail}`，存在时）、`records`
  （该尝试的历史回执，见下）。
- **`node`（节点当前状态，来自 `workflow_runs.node_states`）**：`status`、`attempt`（当前尝试号）、`output`、
  `sessionEndReason`、`error`。`output` / `sessionEndReason` / `error` 是**当前尝试**的状态——节点重试 D3 会把
  它们改为新尝试的 `running`、清掉旧错误，最终被新回答覆盖。所以只有当 `execution.attempt == node.attempt`
  时，`node.output` / `node.sessionEndReason` 才是所选尝试的最终回答与结束原因；请求旧尝试时它们仍描述节点
  当前状态，**不冒充旧尝试的结果**——旧尝试的终态证据是它自己的 `execution.result`（`reason` 即那次结束原因），
  最终回答由前端按 `records` 里 `turnEnded` 前最后一条 `agent_message_chunk` 重放得到。

`records` 每条保留外层 `sequence`、内层 `historySequence`（`HistoryLine.seq`，可选）、`turnId`、`truncated`、
`visibleText` 与 `visibleTextTruncated`（有界文本投影），以及 `record`——`event.record` 这个 JSON 字符串按 JSON
对象透传（不是再转义的字符串），前端沿用既有 `HistoryRecord` 反序列化。分页与完整性状态见下。

**历史回执与终态回执要按「终态证据」权威区分，不能只看内容形状（核心边界）**：`node_event_receipts` 同表存
两类回执——历史回执（`takeOverThreadEvents` 写入的 canonEvent JSON 对象 `{record, sequence, truncated, turnId,
historySequence?, visibleText?, visibleTextTruncated?}`，`record` 是 HistoryRecord 的 JSON 字符串）与终态回执
（`nodeExecutionResult` → `recordNodeReceipt` 写入的任意非空字符串，如 `"ended"`、
`"session-terminal-event"`）。终态 `event` 既然允许是任意字符串，就可能恰好也是一个含 `record` 的 JSON 对象，
所以「是 JSON 对象且有 `record`」不能作为「这是历史回执」的来源证明。权威判据来自已提交的终态证据：
`nodeExecutionResult` 在同一事务里写入 `result` 与终态回执，`recordNodeReceipt` 强制终态回执取该执行的
`last_event_sequence`（最后一个序号）。因此：

- 当所选执行的 `result` 非空时，终态回执就是 `sequence = node_executions.last_event_sequence` 那一行，读取端
  **按序号排除它**，其余 `sequence < last_event_sequence` 的行才是历史回执；
- 当 `result` 为空时还没有终态回执——终态回执只由 `nodeExecutionResult` 与 `result` 在同一事务写入，Cloud 按
  自身判定结算（`node_unavailable` / 会话超时 / 取消）不写 `result`、也就不写终态回执；该执行的所有回执都是
  历史回执。`result` 为空只表示「Node 未提交结果」，不代表在途（见 D4）；
- 内容形状只用于**校验**历史回执（它们应当 `event IS JSON OBJECT` 且 `event->>'record'` 能解析为
  HistoryRecord），不用于**证明**某行是终态回执；排除终态序号后，本页投影时若仍有回执形状不合法，是数据
  缺口，该页标 `incomplete`，不静默丢弃。

分页、计数、游标、空态都作用在「历史回执」这个集合上：终态回执被跳过，不计入 `limit`、不产生 `hasMore`、
不参与 `nextBefore`。

**为什么必须有界分页、且每页要按字节限界**：接管路径只限制单批 64 条、单条 256 KiB，从不限制一次会话的
条数或总字节；若公开 GET 一次返回全部记录，长会话就形成无界响应。分页把每次 GET 都限定在单页内。但
`limit=200` × 单条允许的 256 KiB 理论上单页可到 50 MiB，所以单页还要一个字节预算 `maxPageBytes`（取 4 MiB，
约 16 条最大记录或 100 条 40 KiB 记录，足够一屏，也远小于 `SessionHistory` 的 32 MiB 全量上限）：一页在
条数或字节任一先触顶处停止。这样单次响应无论单条多大都有界。**不需要**「整次会话 32 MiB 总上限」：分页
已经逐页限界，客户端按游标逐段展开即可；`SessionHistory` 的 413 是「一次全量读取超限」的失败信号，与这里的
分页读取语义不同，不复用它的截断/413 态。`incomplete` 与分页的 `hasMore` 是两回事：`hasMore` 表示「还有更早
一页可读」，`incomplete` 表示「本页观测到历史无法完整呈现」。**`incomplete` 是本页观测，不是整次尝试的完整性
结论**——较早页面的 `Gap` / 截断 / 形状校验失败无法由当前页排除，所以 Cloud 不从单页断言整次完整，前端在读齐
全部页且各页都无 `incomplete` 之前应保持「完整性未知」；任一页有 `incomplete` 即整体不完整。同理，旧尝试的
最终回答只能从读齐的 `records` 重放得到，不从尚未读齐的页提前推定。

**分页游标规则固定，不能漏记录**：`nextBefore` 固定为本页**最早一条已返回**历史回执的外层 `sequence`；下一页
请求 `before=<nextBefore>` 取 `sequence < before` 的历史回执。这样每条记录都被后续页面覆盖（记录序号
5,4,3,2,1，首页 `limit=2` 返回 5、4 → `nextBefore=4` → 下页 `sequence<4` 返回 3、2，不漏）。`hasMore` 由独立
探测决定——是否存在 `sequence < nextBefore` 的历史回执——不靠本页是否填满，条数触顶与字节触顶两种情况都成立；
`nextBefore` 仅当存在更早历史回执时返回。若 `nextBefore` 指到下一条未返回记录，`sequence < before` 会把它
永久跳过，故必须指向已返回的序号。

**翻页游标必须绑定尝试，重试时才不会串会话**：首次请求缺省 `attempt` 时读「当前/最新」尝试，但 `nextBefore`
只是外层序号，若翻页期间节点重试出了新尝试，下一页若再按「当前/最新」解析就会落到新尝试、把新尝试的记录
接到旧页后面。因此：翻页请求必须携带首次响应返回的 `execution.attempt` 与 `before`；服务端按（运行，节点，该
`attempt`）解析出该次尝试的执行行，再在其历史回执中取 `sequence < before` 的一页。该执行行与它的回执在重试后
仍保留，所以翻页始终钉在第一次那次尝试上，不变量 1 成立；服务端校验 `attempt` 属于该节点/该运行，不合法即
404，不靠前端清空旧视图兜底。

**两个序号必须分清**：外层 `node_event_receipts.sequence` 是回执的插入顺序，每个执行内连续、单调，用作分页
游标与响应排序；内层 `historySequence`（即 `HistoryLine.seq`，Node 可选上报，范围 0..4294967295）是会话回放
的时间线顺序，前端 `adaptArchive` 按它重放、`adaptThread` 按公开条目序号重放。两者独立，写入顺序不等于
时间线（工具调用可后结算）。响应里两个序号都保留：游标与排序用外层，回放顺序用内层，前端不在读取端猜映射。
`visibleText` 是 cloud#182 已用的有界文本投影（≤ 64 KiB，独立于完整记录），供列表与最终回答归集使用，不是
完整记录的替代。

**长内容可展开**：分页不折叠、不截断记录正文；单条仍受既有 256 KiB 回执校验约束。前端决定折叠粒度，Cloud
只保证分页与完整性状态诚实。

### D2：授权沿用 `workflowModelRead` 模板，作用域下探到节点，不绕过 sessionId

读取的授权与 `workflowModelRead`（`session_model_statistics.go`）同一套，多一步节点作用域：

- `sessionReadIdentity` + 活跃租户成员（非 admin 也可读，与模型读取一致）；
- `workflow` 归档门禁：归档工作流连它的运行历史一起 404；
- 运行必须属于该租户、该工作流且未删除；
- **节点必须属于该运行**：从该运行的冻结图解析 `nodeId`，不存在或不属于该运行即 404。

**不做的事**：不提供「按 execution_id / session_id / 对象键直读」的路由，不让调用方跳过（运行，节点，尝试）
这一层作用域。会话记录与产物地址不是授权主体，主体仍是租户成员 + workflow + 运行 + 节点。读取是纯只读：
不放出任何命令、不隐式启用 HITL、不开启新对话（根决策 D2 的 `interactive: true` 门禁与 #296 D4 的「只读不
隐式注入」都不受影响）。

### D3：输出是最终可见回答，结束原因独立成 `sessionEndReason`，读取端分别呈现

节点状态的「结束原因」与「输出」分离由 cloud#182（已于 2026-10-27 合并）实现，本文继承它的口径、不改写：

- **`output`**：agent 节点的最终可见回答，写进 `{"output": text}`，落到节点声明的 `output` 根，供下游 Output
  节点绑定与 Issue `run.message` 使用（根决策变量根 + #673 owner 结论）。**不因本文清空**。文本是初始回合在
  `turnEnded` 边界前、最后一次 `agent_message_chunk` 的完整回答；被截断的记录不作为成功输出，cloud#182 把
  节点判 `agent_output_truncated` 失败，不发布裁剪预览。
- **`sessionEndReason`**：会话结束原因的独立字段，是执行证据，不是变量根。取值有两类来源：Node 报告的
  `user_ended` / `idle_timeout` / `cancelled` / `agent_failed` / `interrupted`，以及 Cloud 提到公开面的诊断
  （`agent_plugin_unavailable` 由 `agent_failed` + `detail` 提升，`node_unavailable`、
  `session_turn_deadline_exceeded`、`session_start_deadline_exceeded`、`session_end_deadline_exceeded` 与取消
  结算的 `cancelled` 由 Cloud 自己给出，沿用 #296 D4 与取消路径的词汇）。Cloud 不因读取新增或改写取值。
- **`error`**：维持原义，只在失败节点携带公开失败原因。

**「Cloud 不解释记录业务字段」的窄例外**：既有边界是为了不让 Cloud 重写一份 ACP 会话解析。cloud#182 为了
归集 `output` 已经对记录做了一次**窄而有限**的解释——只读初始回合内、`turnEnded` 边界前最后一条完整的
`agent_message_chunk`（或其 `visibleText` 投影），用来取最终回答；这之外 Cloud 仍不解释记录的其余业务字段。
本文的读取（D1）继承同一条窄例外：`output` 由 cloud#182 归集，完整会话记录仍按原样透传，由前端用既有
`HistoryRecord` 渲染「用户输入 → 助手结果 → 工具活动」。不把这条窄例外扩大成通用记录解析器。

### D4：空态按所选尝试的终态证据判定，`result` 为空不单独判在途，非 agent 节点各有真实空态

会话读取不得用空列表掩盖数据缺失，空态要能区分。**判据用「所选尝试」的终态证据与历史回执，不用整个 run 的
状态**：重试后第 1 次尝试可能已有终态 `result`、第 2 次仍在运行，若按整个 run 是否 `running` 判，会把旧尝试
的「无历史」误标成「在途等待」。同样，「零回执」指「零历史回执」——正常结束的会话一定有终态回执（`"ended"`
等），按表行数判会把任何已结束会话都排除在「零回执」之外。

**`result` 为空不等于该次尝试在途**：Cloud 会在 `node_executions.result IS NULL` 时按自己的判定结算节点——
Node 不可达结算 `failed`（`node_unavailable`）、会话 turn 超时结算 `failed`
（`session_turn_deadline_exceeded`）、`end_session` 确认超时结算 `failed`（`session_end_deadline_exceeded`）、
从未派发超时结算 `failed`（`session_start_deadline_exceeded`）、取消结算 `cancelled`——这些路径都不伪造执行
结果，`result` 仍为空。所以 `result` 为空只表示「Node 未上报结果」，不能单独证明「仍在运行」。

**该次尝试的终态证据**取两者的并集：

- 首选 `execution.result`（Node 上报，或重试前那次可重试失败）——非空即该次尝试已结算；
- `result` 为空时，看该尝试是否仍是节点当前尝试：`node.attempt == execution.attempt` 时，`node.status` 是
  Cloud 结算的权威信号（`succeeded` / `failed` / `cancelled` 即已终态，`node.error` 是那次终态原因；`running`
  即在途）；`node.attempt > execution.attempt` 时该尝试已被重试替代、必然已终态（重试只在 `result` 非空时发生），
  若此时 `result` 反而为空属数据异常，按「未知」处理。

据此判空态：

| 状态 | 判据 | 呈现 |
|---|---|---|
| 尚未派发 | 所选尝试没有 `agent_session` 执行行（启动失败、从未派发或尝试等待中） | 明确「未建立会话」，不渲染空对话 |
| **在途、零历史回执** | 有执行行、`result` 为空、`node.attempt == execution.attempt` 且 `node.status` 为 `running`、无历史回执 | 明确「等待首条记录」——会话在途，首条事件还没到，不是历史丢失 |
| **终态、无历史** | 有执行行、零历史回执，且终态证据成立（`result` 非空；或 `node.attempt == execution.attempt` 且 `node.status` 终态） | 明确「历史不可用」（沿用 #296 的 `unavailable`）——该次会话已终止，只是没留下可回放的历史 |
| **终态证据缺失（未知）** | 有执行行、`result` 为空、零历史回执、且无法按上述证明已终态（如 `node.attempt > execution.attempt` 却 `result` 为空的数据异常） | 明确「未知」，不当作在途也不当作完整 |
| 有结果、缺记录 | 终态证据成立、历史回执不齐（本页或已读页观测到显式 `Gap` / `truncated` / 形状校验失败） | 展示结果与「历史不完整／不可用」，分别说明，不当作完整 |
| 非 agent 节点 | `output` / `condition` 等没有会话 | 明确「不适用」，不渲染空会话 |
| 正常会话 | 有执行行、有历史回执、有/无终态证据 | 展示身份、记录、结果与分页状态 |

**为什么判据要落在所选尝试、且 `result` 与节点状态并取**：`result` 为空不代表在途——Cloud 有多个不写 `result`
的终态路径（Node 不可达、会话超时、`end_session` 超时、从未派发超时、取消），只看 `result` 会把「Node 消失 /
会话超时 / 已取消」的零历史尝试误标成「等待首条」。只有 `result` 为空且节点仍停在该尝试且 `status == running`
时才是在途；否则一旦有终态证据就判终态，证据都缺才标「未知」。这比整个 run 的 `running` 更精确，也与节点重试
后「旧尝试已结算、新尝试在途」的多尝试并存事实一致。「有结果、缺记录」要有可验证判据（显式 `Gap` 或
`truncated` 或形状校验失败），不能因为某页没读满就推断历史完整。这些状态位由读取接口返回，节点状态本身继续
只有 `status` / `output` / `sessionEndReason` / `error`。`not_started` 与 `unavailable` 是不同事实（#296 D4 已
这样区分），第二期释放后回看复用同一套状态。

## 不变量

1. 一次读取只落到（运行，节点，尝试）唯一执行行；重试、切节点、切运行或刷新都不会串用另一个尝试的会话，翻页也不串。
2. 读取是纯只读：不推进节点、不放出命令、不隐式启用 HITL 或新对话。
3. 授权主体是租户成员 + workflow + 运行 + 节点，不接受 execution_id / session_id / 产物地址直读。
4. 响应把「所选尝试的执行证据」（`execution`）与「节点当前状态」（`node`）分开；旧尝试不冒领最新 `output` / `sessionEndReason`。
5. 历史回执与终态回执按已提交终态证据的序号权威区分：终态序号由 `result` 与 `last_event_sequence` 决定，内容形状只用于校验历史回执、不用于证明来源。
6. 会话读取有界、分页：单页受条数与字节双上限约束，`nextBefore` 指向本页最早已返回序号、`hasMore` 独立探测，游标绑定尝试，不漏记录、不静默截断。
7. `output` 是最终可见回答（落 `output` 根），结束原因只在 `sessionEndReason`；两者由读取端分别呈现，`output` 不被清空。
8. Cloud 对记录业务字段的解释限于最终回答归集的窄例外，完整会话记录按外层回执序号、携带内层回放序号原样透传。
9. 未建立会话、等待首条记录、历史不可用、终态证据缺失（未知）、不适用是不同的空态，判据以所选尝试的终态证据（`result` 加上节点当前状态）与历史回执为准，`result` 为空不单独判在途，不用空列表掩盖数据缺失。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 一次返回整次会话、不分页 | 接管只限制单批与单条，不限制总量；长会话形成无界公开响应，违背「整次读取有界」原则 |
| 把 agent 的 `output` 清空，实际结果只进会话读取 | 违背根决策变量根（Agent 声明 `output`）与 #673 owner 结论（最终回答写进 `{agent}.output` 供下游和 Issue 使用），也与 cloud#182 冲突；要改输出语义须先请 owner 改前序决策 |
| Cloud 通用解析 `Update` 记录，把整段对话摘要进 `output` | 把「不解释记录业务字段」的窄例外扩大成通用解析器，记录形状变化时要维护 Go 侧解析；`output` 根只承载最终回答，完整对话属于会话读取 |
| 把 `node_event_receipts` 每一行都当作 HistoryRecord 解码进 `records` | 终态回执是任意非空字符串（如 `"ended"`），不是含 `record` 的 JSON；正常结束的会话会在终态回执处解析失败，或被误报成历史缺口 |
| 用「是 JSON 对象且有 `record`」证明某行是历史回执 | 终态 `event` 允许是任意字符串，可能恰好也是含 `record` 的 JSON 对象；来源判据必须用已提交终态证据的序号，不能把内容形状当来源证明 |
| `nextBefore` 指向下一条未返回记录 | `before=<nextBefore>` 取 `sequence<before` 会把那条记录永久跳过；必须指向本页最早已返回序号 |
| 响应把节点当前 `status` / `output` / `sessionEndReason` 与所选旧尝试的执行证据放在同一组 | 这些字段只有当前尝试一份，重试后被新回答覆盖；旧尝试详情会冒领最新成功状态与回答 |
| 按整个 run 是否 `running` 判空态 | 多尝试并存时（旧尝试已结算、新尝试在途）会把旧尝试「无历史」误标成「在途等待」；判据应落在所选尝试的终态证据 |
| 把 `result` 为空当作在途 | Cloud 有多个不写 `result` 的终态路径（Node 不可达、会话 turn/`end_session`/派发超时、取消），会把「Node 消失 / 会话超时 / 已取消」的零历史尝试误标成「等待首条」；在途必须同时满足 `result` 为空且节点停在该尝试且 `status == running` |
| 把 `incomplete` 当作整次尝试的完整性结论 | `incomplete` 是本页观测，较早页面的缺口无法由当前页排除；应把它限定为单页信号，前端读齐全部页前保持完整性未知 |
| 翻页游标只带外层序号、不绑定尝试 | 首次缺省 `attempt` 读「最新」后，翻页期间若节点重试，下一页按「最新」重新解析会落到新尝试，把新尝试记录接到旧页后面，破坏不变量 1 |
| 按「运行，节点」读最新 `agent_session` 行（不带尝试） | 重试后第 2 次尝试会继承第 1 次的失败结果或记录，正是 `workflowAgentSessionOutcome` 注释拒绝的错误 |
| 复用 `SessionHistory`（`session_history.go`）读 WorkflowRun 会话 | 它只接受 `issue_runs.executor_type=agent` 且依赖 revisions 登记，WorkflowRun 没有 Thread/Revision 登记，不能把 workflow 运行伪装成 Agent IssueRun |
| 按 execution_id / session_id 提供直读路由 | 把产物地址当成授权主体，切节点/切运行/换租户都可能绕过（运行，节点，尝试）作用域；与 `workflowModelRead` 的既有授权方向相反 |
| 只在前端加标签页，不动读取契约 | 现场核实：会话记录只存不读（没有公共读取把 `node_event_receipts` 呈现为节点会话），前端拿不到数据；这是执行证据读取、持久化与界面之间的缺口，必须新增读取契约 |

## 本决策未解决的问题

- **Workspace 释放后的会话回看口径**：本文的读取源是数据库里的 `node_event_receipts`（回执是数据库行，不随
  Workspace 释放删除），所以终态运行释放 Workspace 后，**已落库的历史回执仍可按本文读取**；这与 #296 的
  「释放后回看已保存会话」指的不是同一份数据——后者是 Node 归档到对象存储的完整会话文件与文件证据、diff，
  受保留期约束。两者边界由 #296 的成果契约最终定，本文只消费数据库回执，不定义归档保留期。
- **文件变更与 diff**：文件路径、变更类型、增删统计与可打开 diff 需要执行期采集与按写入归属的证据，并行节点
  共用工作区时时间窗 diff 也不能当独占写入证明——这些由 #296 承接，本文不读文件。
- **非 agent 节点（`output` / `condition`）的「结果」是否也需要会话式读取**：它们没有会话记录，本期只给
  「不适用」空态；若将来要展示它们的完整求值过程，另开决策。
- **旧终态运行缺失 `sessionEndReason` 的呈现**：cloud#182 不回填旧终态运行（「old terminal runs are not
  backfilled」），读取端对旧运行显示未知；若 owner 要求历史运行也能看到结束原因，按 #296 的归档回看一起决定，
  不在本期补造。

## 落地顺序

- **阶段 1（ADR 评审）**：本 ADR 以 `proposed` 提交 specs PR，只含正文，不碰 `test-cases/`（`proposed`
  阶段不编写核心测试用例）。完成条件是 PR 已开、owner 开始评审，改动文件只有这一份 ADR。
- **阶段 2（批准后补用例并合并）**：owner 批准后，在同一 PR 补 `test-cases/cloud/workflow/` 下的核心用例，
  状态改为 `approved` 再合并。用例至少覆盖：按尝试读取不串会话、翻页游标绑定尝试、旧尝试不冒领最新输出、
  终态序号权威排除、`nextBefore` 不漏记录、单页条数与字节双上限、显式不静默截断、授权拒绝跨节点/跨运行/
  直读、输出与结束原因分开呈现、各空态按所选尝试的终态证据（`result` 与节点状态并取）真实呈现、`result` 为空
  不误判在途（Node 不可达/会话超时/取消等 Cloud 终态路径）、`incomplete` 为本页观测且读齐前完整性未知、读取
  纯只读不隐式开启新对话。完成条件是每条不变量都有对应用例或明确的 `Missing`，`decisions` 与 `test-cases`
  目录镜像一致，PR 由 owner 合并。
- **阶段 3（Cloud 实现，待开运行）**：新增公开读取路由、历史回执筛选（终态序号排除）与分页、授权、前端节点
  详情接入会话渲染（消费 cloud#182 的 `output` / `sessionEndReason`）。完成条件是真实 PostgreSQL 的读取
  测试覆盖上述路径、OpenAPI 与前端生成物已同步、`task check` 全绿。
- **阶段 4（状态同步）**：实现合并后由落地运行把本文改为 `implemented`，核对偏离、精简正文，并按最终实现
  登记核心用例证据状态。
- **保持不变**：会话写入与结束/结算路径、`node_executions` / `node_event_receipts` 持久化形状与约束、
  `output` / `sessionEndReason` 的语义（由 cloud#182 定义）、非 agent 节点的 `output`、取消与释放的既有结算，
  以及 Controller 线上契约。
