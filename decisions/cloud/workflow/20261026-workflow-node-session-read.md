---
status: proposed
---

# 工作流节点会话读取：按尝试有界读取节点会话记录，输出与结束原因按 cloud#182 口径分开呈现

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策，只固定
「运行中和终态运行如何读到某一个 agent 节点本次尝试的完整会话」这一接缝。结论：Cloud 新增一个公开只读接口，
按（WorkflowRun，节点，尝试）把 `node_executions` 的会话证据与 `node_event_receipts` 的会话记录有界、分页地
呈现为节点会话历史；节点状态把「结束原因」与「输出」分开，其中 `output` 是最终可见回答、结束原因独立成
`sessionEndReason`——这套分离由 cloud#182（承接 ora-space/circus#676，当前为 Draft 候选、未合并）实现，本文
继承它的口径、依赖它先合并，不重新定义。读取沿用 `workflowModelRead` 的授权模板下探到节点，不新增 sessionId
或产物地址直读；只把含 `record` 的历史回执投影进会话记录，终态回执由独立 `result` 表示；运行中零历史回执、
终态无历史、非 agent 节点各有真实空态，分页与截断都有显式状态，不用空列表或静默截断掩盖数据缺失。

**状态与影响面**：本文为 `proposed`，待 owner 评审。涉及 cloud 的公开读取路由与 OpenAPI 新增接口，以及前端
节点详情的会话渲染。**兼容边界**：只新增接口，不改 `node_executions` / `node_event_receipts` 的持久化形状与
既有约束，不新增迁移；`nodeStates` 的 `output` / `sessionEndReason` 分离由 cloud#182 候选实现，本文只消费这
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
| 根决策「实现与 approved 方案的差异」已知缺口（Agent 回答没写进 `output` 根） | 由 cloud#182 候选 / ora-space/circus#673、#676 修复（待合并）；本文读取端消费修复后的 `output`，不重复修复 |
| [会话结束后续决策](20261019-workflow-session-end-after-initial-turn.md) D1–D5：成功回合后 Cloud 按节点唯一放出 `end_session`，结束原因取 `idle_timeout`；节点终态仍只来自会话结果 | 继承结束原因词汇：`user_ended` / `idle_timeout` / `cancelled` / `agent_failed` / `interrupted` 及 Cloud 自己的诊断是结束原因的闭集；本文把它们作为 `sessionEndReason` 呈现，不新增原因 |
| [节点重试](20261021-node-retry-and-resume-contract.md) D4：每次尝试有独立的 `node_executions` 行，按尝试唯一的会话证据与结束命令 | 继承，并成为读取的作用域：一次读取只落到一个尝试的执行行，重试后能区分每次尝试，切节点/切运行/刷新不串用会话 |
| 节点重试 D1：`attempt` 是 Cloud 内部路由字段，不来自线上请求 | 继承：读取按（运行，节点，尝试）解析执行行，尝试序号来自已提交节点状态，不接受调用方指定 execution_id |
| cloud#182（`fix/676-agent-output-template`，Draft 候选 `b327040`，未合并）：成功结算时把初始回合最终的完整助手回答写进 `output`（`{"output": text}`），截断则失败；结束原因另存 `sessionEndReason`，不占用 `output`；用 `event IS JSON OBJECT` 保护终态回执的解析 | 继承，不改写，并作为实现前置：cloud#182 先合并，本文读取端消费 `output` / `sessionEndReason`；历史回执筛选与 cloud#182 同一保护一致（见 D1） |
| [运行拥有成果决策](20261025-run-owned-git-delivery-and-history.md) D4：完整归档的读取继承原运行授权，本文只交付保存与可验证引用，不增加完整节点日志 UI | 边界切分：节点会话读取/UI 由本文承接，不属于 #296；`not_started` / `unavailable` 的空态词汇沿用 #296 D4 |
| `workflowModelRead`（`session_model_statistics.go`）：sessionReadIdentity + 成员 + workflow 归档门禁 + run∈workflow/tenant | 继承授权模板，扩展到节点级作用域；不引入 sessionId 直读 |
| `SessionHistory`（`session_history.go`）的读取边界：整次历史 32 MiB / 10 万条上限、超限返回 413 | 继承「整次读取必须有界」的原则；但节点会话读取用「分页 + 单页字节上限」保证每次响应有界，不复用它的 413/截断态（见 D1） |

## 决策

### D1：公开读取按（运行，节点，尝试）有界、分页地呈现该次尝试的会话记录

新增只读接口 `GET /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/nodes/:nodeId/session`，参数：

- `?attempt=N`（可选）：读第 N 次尝试；缺省读该节点当前/最新尝试；越界或不属于该节点 404。
- `?before=<sequence>`（可选）：取该尝试中外层 `sequence < before` 的一页历史回执，按降序；缺省从最新往回取。**只与 `attempt` 配对使用**：带了 `before` 却不带 `attempt` 时 400——`before` 只有钉在具体尝试上才有意义。
- `?limit=N`（可选）：一页条数，缺省 100、上限 200；与字节上限共同限界（见下）。

响应固定几块：

- **身份与结果**：`nodeId`、`attempt`（服务端解析后回显，翻页必须原样带回）、`executionId`、节点 `status`、
  `output`（最终可见回答，见 D3）、`sessionEndReason`（结束原因，见 D3）、`error`（仅失败节点）、`result`
  （`node_executions.result` 的 `{outcome, reason, detail}`，存在时）。
- **会话记录** `records`：一页历史回执，按外层 `sequence` 降序。每条保留外层 `sequence`、内层
  `historySequence`（`HistoryLine.seq`，可选，见下）、`turnId`、`truncated`、`visibleText` 与
  `visibleTextTruncated`（有界文本投影，见下），以及 `record`——`event.record` 这个 JSON 字符串按 JSON
  对象透传（不是再转义的字符串），前端沿用既有 `HistoryRecord` 反序列化。
- **分页与完整性状态**：`nextBefore`（本页最早一条历史回执的外层 `sequence`，仅 `hasMore` 为真时存在）、
  `hasMore`（是否还有更早的历史回执）、`incomplete`（会话历史本身不完整：显式 `Gap` 或 `truncated`）。

**`node_event_receipts.event` 是异质列，读取必须按「历史回执」筛选（核心边界）**：同一表同时保存两类回执：

- **历史回执**：`takeOverThreadEvents` 写入的 canonEvent JSON 对象 `{record, sequence, truncated, turnId,
  historySequence?, visibleText?, visibleTextTruncated?}`，其中 `record` 是 HistoryRecord 的 JSON 字符串。
- **终态回执**：`nodeExecutionResult` → `recordNodeReceipt` 写入的**任意非空字符串**，只要求
  `sequence >= 1 && event != ""`，不一定是 JSON；Cloud 测试用合法的 `"ended"`（
  `workflow_run_session_end_db_test.go`）、`"session-terminal-event"`（`integration/node_execution_receipts_test.go`）。

因此读取只把「历史回执」投影进 `records`：判据是 `event IS JSON OBJECT` 且 `event.record` 能解析为
HistoryRecord；终态回执不进 `records`，由独立的 `result` 字段表示（它本来就是 `node_executions.result`
的 ACK，不是会话历史）。这条筛选与 cloud#182 的 `workflowOutputReceiptsSQL` 同一条保护一致（
`CASE WHEN event IS JSON OBJECT THEN event::jsonb ELSE '{}'::jsonb END`）。分页、计数、游标、空态全部作用在
「历史回执」这个筛选集合上：终态回执被跳过，不计入 `limit`、不产生 `hasMore`、不参与 `nextBefore`。

**为什么必须有界分页、且每页要按字节限界**：接管路径只限制单批 64 条、单条 256 KiB，从不限制一次会话的
条数或总字节；若公开 GET 一次返回全部记录，长会话就形成无界响应。分页把每次 GET 都限定在单页内。但
`limit=200` × 单条允许的 256 KiB 理论上单页可到 50 MiB，所以单页还要一个字节预算 `maxPageBytes`（取 4 MiB，
约 16 条最大记录或 100 条 40 KiB 记录，足够一屏，也远小于 `SessionHistory` 的 32 MiB 全量上限）：一页在
条数或字节任一先触顶处停止，下一条历史回执记为 `nextBefore`、`hasMore=true`。这样单次响应无论单条多大都
有界。**不需要**「整次会话 32 MiB 总上限」：分页已经逐页限界，客户端按 `nextBefore` 逐段展开即可；
`SessionHistory` 的 413 是「一次全量读取超限」的失败信号，与这里的分页读取语义不同，不复用它的截断/413 态。
`incomplete` 只表达「会话历史本身有 Gap/截断」，与分页的 `hasMore` 是两回事。

**翻页游标必须绑定尝试，重试时才不会串会话**：首次请求缺省 `attempt` 时读「当前/最新」尝试，但
`nextBefore` 只是外层序号，若翻页期间节点重试出了新尝试，下一页若再按「当前/最新」解析就会落到新尝试、把
新尝试的记录接到旧页后面。因此：响应始终回显解析后的 `attempt`，翻页请求必须携带首次响应返回的 `attempt`
与 `before`；服务端按（运行，节点，该 `attempt`）解析出该次尝试的执行行，再在其历史回执中取
`sequence < before` 的一页。该执行行与它的回执在重试后仍保留，所以翻页始终钉在第一次那次尝试上，不变量 1
成立；服务端校验 `attempt` 属于该节点/该运行，不合法即 404，不靠前端清空旧视图兜底。

**两个序号必须分清**：外层 `node_event_receipts.sequence` 是回执的插入顺序，每个执行内连续、单调，用作
分页游标与响应排序；内层 `historySequence`（即 `HistoryLine.seq`，Node 可选上报，范围 0..4294967295）是
会话回放的时间线顺序，前端 `adaptArchive` 按它重放、`adaptThread` 按公开条目序号重放。两者独立，写入
顺序不等于时间线（工具调用可后结算）。响应里两个序号都保留：游标与排序用外层，回放顺序用内层，前端不
在读取端猜映射。`visibleText` 是 cloud#182 已用的有界文本投影（≤ 64 KiB，独立于完整记录），供列表与
最终回答归集使用，不是完整记录的替代。

**长内容可展开**：分页不折叠、不截断记录正文；单条仍受既有 256 KiB 回执校验约束。前端决定折叠粒度，
Cloud 只保证分页与完整性状态诚实。

### D2：授权沿用 `workflowModelRead` 模板，作用域下探到节点，不绕过 sessionId

读取的授权与 `workflowModelRead`（`session_model_statistics.go`）同一套，多一步节点作用域：

- `sessionReadIdentity` + 活跃租户成员（非 admin 也可读，与模型读取一致）；
- `workflow` 归档门禁：归档工作流连它的运行历史一起 404；
- 运行必须属于该租户、该工作流且未删除；
- **节点必须属于该运行**：从该运行的冻结图解析 `nodeId`，不存在或不属于该运行即 404。

**不做的事**：不提供「按 execution_id / session_id / 对象键直读」的路由，不让调用方跳过（运行，节点，
尝试）这一层作用域。会话记录与产物地址不是授权主体，主体仍是租户成员 + workflow + 运行 + 节点。读取是
纯只读：不放出任何命令、不隐式启用 HITL、不开启新对话（根决策 D2 的 `interactive: true` 门禁与 #296 D4
的「只读不隐式注入」都不受影响）。

### D3：输出是最终可见回答，结束原因独立成 `sessionEndReason`，读取端分别呈现

节点状态的「结束原因」与「输出」分离由 cloud#182 候选（Draft，未合并）实现，本文继承它的口径、不改写：

- **`output`**：agent 节点的最终可见回答，写进 `{"output": text}`，落到节点声明的 `output` 根，供下游
  Output 节点绑定与 Issue `run.message` 使用（根决策变量根 + #673 owner 结论）。**不因本文清空**。文本是
  初始回合在 `turnEnded` 边界前、最后一次 `agent_message_chunk` 的完整回答；被截断的记录不作为成功输出，
  cloud#182 把节点判 `agent_output_truncated` 失败，不发布裁剪预览。
- **`sessionEndReason`**：会话结束原因的独立字段（`user_ended` / `idle_timeout` / `cancelled` /
  `agent_failed` / `interrupted`，或 Cloud 自己的诊断），是执行证据，不是变量根。
- **`error`**：维持原义，只在失败节点携带公开失败原因。

**「Cloud 不解释记录业务字段」的窄例外**：既有边界是为了不让 Cloud 重写一份 ACP 会话解析。cloud#182 为了
归集 `output` 已经对记录做了一次**窄而有限**的解释——只读初始回合内、`turnEnded` 边界前最后一条完整的
`agent_message_chunk`（或其 `visibleText` 投影），用来取最终回答；这之外 Cloud 仍不解释记录的其余业务字段。
本文的读取（D1）继承同一条窄例外：`output` 由 cloud#182 归集，完整会话记录仍按原样透传，由前端用既有
`HistoryRecord` 渲染「用户输入 → 助手结果 → 工具活动」。不把这条窄例外扩大成通用记录解析器。

### D4：运行中零历史回执、终态无历史、非 agent 节点各有真实空态

会话读取不得用空列表掩盖数据缺失，空态要能区分，且判据分在途/终态。**判据用「历史回执」而不是
`node_event_receipts` 行数**：正常结束的会话一定有终态回执（`"ended"` 等），若按表行数判，任何结束的会话
都不满足「零回执」；只有「历史回执」为零才表示没有可回放的会话内容。

| 状态 | 判据 | 呈现 |
|---|---|---|
| 尚未派发 | 该节点该尝试没有 `agent_session` 执行行（启动失败、从未派发或尝试等待中） | 明确「未建立会话」，不渲染空对话 |
| **运行中、零历史回执** | 有执行行、运行仍 `running`、无任何历史回执（终态回执也还没到） | 明确「等待首条记录」——会话在途，首条事件还没到，不是历史丢失 |
| **终态、无历史** | 有执行行、运行已终态、无任何历史回执（只有 `"ended"` 这类终态回执） | 明确「历史不可用」（沿用 #296 的 `unavailable`） |
| 有结果、缺记录 | 有 `result` 但历史回执不齐（显式 `Gap` 或 `truncated`） | 展示结果与「历史不完整／不可用」，分别说明，不当作完整 |
| 非 agent 节点 | `output` / `condition` 等没有会话 | 明确「不适用」，不渲染空会话 |
| 正常会话 | 有执行行、有历史回执、有/无终态结果 | 展示身份、记录、结果与分页状态 |

**为什么「运行中零历史回执」不能直接判「历史不可用」**：会话在途时零历史回执只是首条事件尚未上报（Node
可能还在跑），把它标成「不可用」会把暂时空白误报成历史丢失。判据必须带上运行是否终态：在途是「等待首条」，
终态才可能「无历史」。「有结果、缺记录」同样要有可验证判据（显式 `Gap` 或 `truncated`），不能因为某页没读满
就推断历史完整。这些状态位由读取接口返回，节点状态本身继续只有 `status` / `output` / `sessionEndReason` /
`error`。`not_started` 与 `unavailable` 是不同事实（#296 D4 已这样区分），第二期释放后回看复用同一套状态。

## 不变量

1. 一次读取只落到（运行，节点，尝试）唯一执行行；重试、切节点、切运行或刷新都不会串用另一个尝试的会话，翻页也不串。
2. 读取是纯只读：不推进节点、不放出命令、不隐式启用 HITL 或新对话。
3. 授权主体是租户成员 + workflow + 运行 + 节点，不接受 execution_id / session_id / 产物地址直读。
4. 会话记录只投影「历史回执」（`event IS JSON OBJECT` 且含可解析 `record`），终态回执由独立 `result` 表示，不进 `records`。
5. 会话读取有界、分页：单页受条数与字节双上限约束，游标按外层历史回执序号稳定排序并绑定尝试；`hasMore` 与 `incomplete` 都是显式状态，不静默截断。
6. `output` 是最终可见回答（落 `output` 根），结束原因只在 `sessionEndReason`；两者由读取端分别呈现，`output` 不被清空。
7. Cloud 对记录业务字段的解释限于最终回答归集的窄例外，完整会话记录按外层回执序号、携带内层回放序号原样透传。
8. 未建立会话、等待首条记录、历史不可用、不适用是不同的空态，判据区分在途与终态、以历史回执为准，不用空列表掩盖数据缺失。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 一次返回整次会话、不分页 | 接管只限制单批与单条，不限制总量；长会话形成无界公开响应，违背「整次读取有界」原则 |
| 把 agent 的 `output` 清空，实际结果只进会话读取 | 违背根决策变量根（Agent 声明 `output`）与 #673 owner 结论（最终回答写进 `{agent}.output` 供下游和 Issue 使用），也与 cloud#182 候选冲突；要改输出语义须先请 owner 改前序决策 |
| Cloud 通用解析 `Update` 记录，把整段对话摘要进 `output` | 把「不解释记录业务字段」的窄例外扩大成通用解析器，记录形状变化时要维护 Go 侧解析；`output` 根只承载最终回答，完整对话属于会话读取 |
| 把 `node_event_receipts` 每一行都当作 HistoryRecord 解码进 `records` | 终态回执是任意非空字符串（如 `"ended"`），不是含 `record` 的 JSON；正常结束的会话会在终态回执处解析失败，或被误报成历史缺口 |
| 翻页游标只带外层序号、不绑定尝试 | 首次缺省 `attempt` 读「最新」后，翻页期间若节点重试，下一页按「最新」重新解析会落到新尝试，把新尝试记录接到旧页后面，破坏不变量 1 |
| 按「运行，节点」读最新 `agent_session` 行（不带尝试） | 重试后第 2 次尝试会继承第 1 次的失败结果或记录，正是 `workflowAgentSessionOutcome` 注释拒绝的错误 |
| 复用 `SessionHistory`（`session_history.go`）读 WorkflowRun 会话 | 它只接受 `issue_runs.executor_type=agent` 且依赖 revisions 登记，WorkflowRun 没有 Thread/Revision 登记，不能把 workflow 运行伪装成 Agent IssueRun |
| 按 execution_id / session_id 提供直读路由 | 把产物地址当成授权主体，切节点/切运行/换租户都可能绕过（运行，节点，尝试）作用域；与 `workflowModelRead` 的既有授权方向相反 |
| 只在前端加标签页，不动读取契约 | 现场核实：会话记录只存不读（没有公共读取把 `node_event_receipts` 呈现为节点会话），前端拿不到数据；这是执行证据读取、持久化与界面之间的缺口，必须新增读取契约 |
| 「运行中零回执」直接显示「历史不可用」 | 在途零历史回执只是首条事件未到，误报成历史丢失；判据必须区分在途与终态 |

## 本决策未解决的问题

- **Workspace 释放后的会话回看口径**：本文的读取源是数据库里的 `node_event_receipts`（回执是数据库行，不随
  Workspace 释放删除），所以终态运行释放 Workspace 后，**已落库的历史回执仍可按本文读取**；这与 #296 的
  「释放后回看已保存会话」指的不是同一份数据——后者是 Node 归档到对象存储的完整会话文件与文件证据、diff，
  受保留期约束。两者边界由 #296 的成果契约最终定，本文只消费数据库回执，不定义归档保留期。
- **文件变更与 diff**：文件路径、变更类型、增删统计与可打开 diff 需要执行期采集与按写入归属的证据，并行节点
  共用工作区时时间窗 diff 也不能当独占写入证明——这些由 #296 承接，本文不读文件。
- **非 agent 节点（`output` / `condition`）的「结果」是否也需要会话式读取**：它们没有会话记录，本期只给
  「不适用」空态；若将来要展示它们的完整求值过程，另开决策。
- **旧终态运行缺失 `sessionEndReason` 的呈现**：cloud#182 候选不回填旧终态运行（「old terminal runs are not
  backfilled」），读取端对旧运行显示未知；若 owner 要求历史运行也能看到结束原因，按 #296 的归档回看一起决定，
  不在本期补造。

## 落地顺序

- **阶段 1（ADR 评审）**：本 ADR 以 `proposed` 提交 specs PR，只含正文，不碰 `test-cases/`（`proposed`
  阶段不编写核心测试用例）。完成条件是 PR 已开、owner 开始评审，改动文件只有这一份 ADR。
- **阶段 2（批准后补用例并合并）**：owner 批准后，在同一 PR 补 `test-cases/cloud/workflow/` 下的核心
  用例，状态改为 `approved` 再合并。用例至少覆盖：按尝试读取不串会话、翻页游标绑定尝试、只投影历史回执
  （终态回执不进 `records`）、单页条数与字节双上限、显式不静默截断、授权拒绝跨节点/跨运行/直读、输出与
  结束原因分开呈现、各空态真实呈现、读取纯只读不隐式开启新对话。完成条件是每条不变量都有对应用例或明确的
  `Missing`，`decisions` 与 `test-cases` 目录镜像一致，PR 由 owner 合并。
- **阶段 3（Cloud 实现，待开运行）**：新增公开读取路由、历史回执筛选与分页、授权、前端节点详情接入会话渲染
  （消费 cloud#182 候选的 `output` / `sessionEndReason`）。完成条件是真实 PostgreSQL 的读取测试覆盖上述路径、
  OpenAPI 与前端生成物已同步、`task check` 全绿。
- **阶段 4（状态同步）**：实现合并后由落地运行把本文改为 `implemented`，核对偏离、精简正文，并按最终实现
  登记核心用例证据状态。
- **保持不变**：会话写入与结束/结算路径、`node_executions` / `node_event_receipts` 持久化形状与约束、
  `output` / `sessionEndReason` 的语义（由 cloud#182 定义）、非 agent 节点的 `output`、取消与释放的既有结算，
  以及 Controller 线上契约。
