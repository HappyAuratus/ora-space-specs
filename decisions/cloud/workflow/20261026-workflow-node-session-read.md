---
status: proposed
---

# 工作流节点会话读取：按尝试读本节点的用户输入、助手结果与工具活动，结束原因与输出分开呈现

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`implemented`）的后续决策，只固定
「运行中和终态运行如何读到某一个 agent 节点本次尝试的会话」这一接缝。结论：Cloud 新增一个公开读取，按
（WorkflowRun，节点，尝试）把 `node_executions` 的会话证据与 `node_event_receipts` 的会话记录按序列呈现为
节点会话历史；节点状态把「结束原因」从 `output` 槽位拆成独立 `reason` 字段，agent 节点的实际结果由会话
读取呈现而不是塞进 `output` 的 `{"reason":...}`。读取沿用 `workflowModelRead` 的授权模板，不新增 sessionId
或产物地址直读；没有会话、没有记录、非 agent 节点各有真实空态，不用空列表掩盖数据缺失。

**状态与影响面**：本文为 `proposed`，待 owner 评审。涉及 cloud 的公开读取路由与 `workflow_runs.node_states`
投影、OpenAPI 的 `nodeStates` 与新增读取接口，以及前端节点详情的会话渲染。**兼容边界**：新增字段一律可选，
旧读取方忽略未知字段即可；`nodeStates` 新增 `reason` 字段，agent 节点的 `output` 从 `{"reason":...}` 改为
空，其余节点类型的 `output` 不变；不改变 `node_executions` / `node_event_receipts` 的持久化形状与既有约束，
不新增迁移。**行为变化**：agent 节点详情的「输出」不再显示 `{"reason":"idle_timeout"}`，改由结束原因字段与
会话读取分别呈现；这是修复缺陷，不是回归。**非目标**：不新增会话写入、不改变会话的结束与结算路径、不新增
取消或 HITL 入口、不读取已释放 Workspace 的文件与 diff（那是 #296 的成果契约，本文只读会话证据）；不把
Cloud 升级为解释 `ora-history` 记录业务字段的读取端——记录仍按原样透传，由前端沿用既有 `HistoryRecord`
反序列化渲染。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D4：节点终态只来自会话执行结果（`node_executions.result`）；崩溃恢复只按已提交结果重算 | 继承，并成为读取前提：会话证据与记录都挂在已提交的执行行上，读取不推进节点、不重算 |
| 根决策 D4 / D2：agent 节点首期串行，发布门禁拒绝 `interactive: true`，同一 run 的 agent 会话是单回合 | 继承：读取端看到的是一次单回合会话的输入与结果，不出现用户后续轮次 |
| 根决策 D7：首期只保证资源释放，成果交付延后 | 继承：本文只读会话证据，不触碰成果保存、文件归属与释放顺序 |
| [会话结束后续决策](20261019-workflow-session-end-after-initial-turn.md) D1–D5：成功回合后 Cloud 按节点唯一放出 `end_session`，结束原因取 `idle_timeout`；节点终态仍只来自会话结果 | 继承结束原因词汇：`user_ended` / `idle_timeout` / `cancelled` / `agent_failed` / `interrupted` 以及 Cloud 自己的诊断是结束原因的闭集；本文只把它们从 `output` 拆到 `reason`，不新增原因 |
| [节点重试](20261021-node-retry-and-resume-contract.md) D4：每次尝试有独立的 `node_executions` 行，按尝试唯一的会话证据与结束命令 | 继承，并成为读取的作用域：一次读取只落到一个尝试的执行行，重试后能区分每次尝试，切节点/切运行/刷新不串用会话 |
| 节点重试 D1：`attempt` 是 Cloud 内部路由字段，不来自线上请求 | 继承：读取按（运行，节点，尝试）解析执行行，尝试序号来自已提交节点状态，不接受调用方指定 execution_id |
| [运行拥有成果决策](20261025-run-owned-git-delivery-and-history.md) D4：完整归档的读取继承原运行授权，本文只交付保存与可验证引用，不增加完整节点日志 UI | 边界切分：节点会话读取/UI 由本文承接，不属于 #296；`not_started` / `unavailable` 的空态词汇沿用 #296 D4，第二期的释放后回看复用同一套状态 |
| `workflowModelRead`（`session_model_statistics.go`）：sessionReadIdentity + 成员 + workflow 归档门禁 + run∈workflow/tenant | 继承授权模板，扩展到节点级作用域；不引入 sessionId 直读 |

## 决策

### D1：公开读取按（运行，节点，尝试）呈现该次尝试的会话记录

新增只读接口，把已存在的执行关联呈现为节点会话历史：

- 路径 `GET /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/nodes/:nodeId/session`，可选
  `?attempt=N`。缺省 `attempt` 时读该节点**当前（最新）尝试**；指定时读那一次尝试，越界 404。
- 响应固定四块：身份（`nodeId`、`attempt`、`executionId`、节点 `status` 与 `reason`）、按 `sequence`
  升序的会话记录（`records`，每条带 `type` 判别符与原始记录 JSON）、终态结果（`result`，即
  `node_executions.result` 的 `{outcome, reason, detail}`，存在时）、以及会话是否建立/记录是否齐备的
  状态位（见 D4）。
- 数据来源是现成的身份链：`workflow_runs.id` → `node_executions`（`operation_id`、
  `workflow_node_id`、`workflow_node_attempt`、`kind='agent_session'`）→ `node_event_receipts`
  （`execution_id`、`sequence`、`event`）。读取端按这条链解析，**不接受调用方提供的
  execution_id、session_id 或产物地址**（D2）。

**为什么记录按原样透传而不是 Cloud 解释业务字段**：会话记录是 `ora-history` 的 `HistoryRecord`，
`Update` 变体里是装配好的 ACP `SessionUpdate`（已结算的 user/assistant 消息、thought、tool call、plan），
`TurnEnded` 结束一轮，`Meta`/`AgentSwitched`/`HandoffDelivered`/`Gap` 覆盖 ACP 没有的状态。Cloud 接管
已经只校验 `type` 判别与 `stop_reason`，不解释记录业务字段（会话结束决策 D1 与 `workflow_run_session.go`
注释）。要渲染「用户输入 / 助手结果 / 工具活动」，前端已经拥有 `HistoryRecord` 的反序列化与 `ChatView`
（desktop `run-node-session-chat.tsx` 即按会话历史渲染）；Cloud 把记录按序列原样返回，前端沿用既有
渲染器，就不需要在 Go 里复制一份 ACP 会话解析、也不需要在记录形状变化时同步两个解析实现。Cloud 只做
三件它已经会做的事：按 `sequence` 排序、按（运行，节点，尝试）圈定范围、把已经校验过的终态
`result` 单独给出。

**为什么尝试必须显式**：一个节点重试后，第 1 次尝试的失败结果与记录仍留在自己的执行行上（重试决策
D4），而节点状态只指向当前尝试。按「运行，节点」读最新的 `agent_session` 行会让第 2 次尝试继承第 1 次的
结果或记录——正是 `workflowAgentSessionOutcome` 注释里点名拒绝的错误。因此读取按（运行，节点，尝试）
落到唯一执行行，缺省尝试取节点状态的当前尝试序号，与调度器读证据同一条规则。

**长内容可展开**：记录按 `sequence` 是稳定的全序（`HistoryLine.seq` 在会话里赋号，写入顺序不等于时间线，
读取端按 `seq` 排序），接口不折叠、不分页截断记录正文；前端决定折叠粒度。Cloud 对记录长度设与既有回执
一致的上限，超出按既有 `receipt` 校验拒绝，不在这里放宽。

### D2：授权沿用 `workflowModelRead` 模板，作用域下探到节点，不绕过 sessionId

读取的授权与 `workflowModelRead`（`session_model_statistics.go`）同一套，多一步节点作用域：

- `sessionReadIdentity` + 活跃租户成员（非 admin 也可读，与模型读取一致）；
- `workflow` 归档门禁：归档工作流连它的运行历史一起 404；
- 运行必须属于该租户、该工作流且未删除；
- **节点必须属于该运行**：从该运行的冻结图解析 `nodeId`，不存在或不属于该运行即 404，不把它当成
  另一条运行的节点。

**不做的事**：不提供「按 execution_id / session_id / 对象键直读」的路由，不让调用方跳过（运行，节点，
尝试）这一层作用域。会话记录与产物地址不是授权主体，主体仍是租户成员 + workflow + 运行 + 节点；
这样切节点、切运行或换租户都拿不到别人的会话。读取是纯只读：不放出任何命令、不隐式启用 HITL、
不开启新对话（根决策 D2 的 `interactive: true` 门禁与 #296 D4 的「只读不隐式注入」都不受影响）。

### D3：节点状态把结束原因拆成独立 `reason` 字段，agent 节点不再把 reason 写进 `output`

现在 agent 节点结算时无条件 `state.Output = Object{"reason": reason}`（`applyWorkflowAgentTerminal`、
`cancelWorkflowAgentNode`、`failWorkflowAgentNode` 及 `workflowAgentSessionOutcome` 的各失败分支），于是
详情页只看到 `{"reason":"idle_timeout"}`——结束原因占据了输出槽位，正常终止也看不到实际助手结果。本文
把这两件事分开：

- **`reason`**（新增字段）：agent 节点的会话结束原因。闭集是 Node 上报的会话结束原因
  `user_ended` / `idle_timeout` / `cancelled` / `agent_failed` / `interrupted`，加上 Cloud 自己的诊断
  `node_unavailable` / `session_start_deadline_exceeded` / `session_turn_deadline_exceeded` /
  `session_end_deadline_exceeded` / `workflow_run_project_required` / `workflow_executor_unconfigured` /
  `agent_plugin_version_unfrozen` / `agent_plugin_unavailable` / `workspace_unavailable`。非 agent 节点
  不携带 `reason`。
- **`output`**：agent 节点不再写 `{"reason":...}`，改为空/省略。agent 节点的实际结果是它的会话对话
  （D1 的 `records`），由前端在会话视图渲染，不塞进节点状态的输出槽位——Cloud 不解释记录业务字段，
  自然也不从记录里摘「最后一句助手文本」冒充输出。`output` / `condition` 等非 agent 节点的 `output`
  维持原义不变。
- **`error`** 维持原义：仍只在 failed 节点携带（可被提升的）公开失败原因，如
  `agent_plugin_unavailable`。它与 `reason` 的差别是：`reason` 是「会话怎么结束的」（原始结束原因，
  成功节点也有），`error` 是「公开侧读到的失败原因」（只有失败节点有，且可能从 `detail` 提升）。

**为什么不是把助手文本摘进 `output`**：那需要 Cloud 解析 `Update` 记录找出「最后一条 assistant 消息」，
把「不解释记录业务字段」的既有边界打破，还要定义「哪一条算输出」的规则并在记录形状变化时维护。会话
读取已经给出完整记录，前端渲染器天然展示「用户输入 → 助手结果 → 工具活动」，比一个摘要字段更诚实、
更少歧义。结束原因单独成字段，满足「结束原因与实际结果分开呈现」。

**兼容**：`reason` 与「agent 节点 `output` 变空」都是新增/收窄，不是对既有字段语义的重新解释；旧读取方
忽略 `reason`、仍渲染 `output` 时最多看到空，不会读出错义。历史终态运行的 `node_states` 不补算
`reason`——它们没有需要迁移的数据，读取端对缺失 `reason` 的旧运行显示为未知，不回填。

### D4：没有会话、没有记录、非 agent 节点各有真实空态

会话读取与节点状态不得用空列表掩盖数据缺失，空态要能区分：

| 状态 | 判据 | 呈现 |
|---|---|---|
| 尚未派发 | 该节点该尝试没有 `agent_session` 执行行（启动失败、从未派发或尝试等待中） | 明确「未建立会话」，不渲染空对话 |
| 已派发、无记录 | 有执行行但 `node_event_receipts` 为空（Node 未上报任何记录即消失） | 明确「历史不可用」（沿用 #296 的 `unavailable` 词汇），不当作「空对话」 |
| 有结果、缺记录 | 有 `result` 但记录不齐 | 展示结果与「历史不可用」，分别说明 |
| 非 agent 节点 | `output` / `condition` 等没有会话 | 明确「不适用 / 不支持」，不渲染空会话 |
| 正常会话 | 有执行行、有记录、有/无终态结果 | 展示身份、记录与结果 |

这些状态位由读取接口返回（D1 的状态位），节点状态本身继续只有 `status` / `reason` / `error` /
`output`。判据读的是已提交执行行与回执，读取不替运行做任何判定，也不把「读不到」当成「未建立」——
`not_started` 与 `unavailable` 是不同事实（#296 D4 已经这样区分）。第二期（释放后回看已保存会话）复用
同一套状态，只是 `unavailable` 时多了「已过保留期」这一来源。

## 不变量

1. 一次读取只落到（运行，节点，尝试）唯一执行行；重试、切节点、切运行或刷新都不会串用另一个尝试的会话。
2. 读取是纯只读：不推进节点、不放出命令、不隐式启用 HITL 或新对话。
3. 授权主体是租户成员 + workflow + 运行 + 节点，不接受 execution_id / session_id / 产物地址直读。
4. agent 节点的结束原因只出现在 `reason` 字段，不写进 `output`；`output` 槽位对 agent 节点为空。
5. Cloud 不解释会话记录的业务字段，记录按 `sequence` 原样透传，由前端沿用 `HistoryRecord` 渲染。
6. 未建立会话、历史不可用、不适用是三种不同的空态，不用空列表掩盖数据缺失。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| Cloud 解析 `Update` 记录，把最后一条助手文本摘要进节点 `output` | 打破「Cloud 不解释记录业务字段」的既有边界，还要定义「哪一条算输出」并在 `ora-history` 形状变化时同步维护 Go 解析；会话读取已给出完整记录，前端渲染更诚实 |
| 按「运行，节点」读最新 `agent_session` 行（不带尝试） | 重试后第 2 次尝试会继承第 1 次的失败结果或记录，正是 `workflowAgentSessionOutcome` 注释拒绝的错误 |
| 复用 `SessionHistory`（`session_history.go`）读 WorkflowRun 会话 | 它只接受 `issue_runs.executor_type=agent` 且依赖 revisions 登记，WorkflowRun 没有 Thread/Revision 登记，不能把 workflow 运行伪装成 Agent IssueRun |
| 按 execution_id / session_id 提供直读路由 | 把产物地址当成授权主体，切节点/切运行/换租户都可能绕过（运行，节点，尝试）作用域；与 `workflowModelRead` 的既有授权方向相反 |
| 只在前端加标签页，不动读取契约 | 现场核实：会话记录只存不读（没有公共读取把 `node_event_receipts` 呈现为节点会话），前端拿不到数据；这是执行证据读取、持久化与界面之间的缺口，必须新增读取契约 |
| 把结束原因继续留在 `output`，只在前端换文案 | 正常终止原因（`idle_timeout`）仍会替代实际助手结果，数据结构与语义都错，前端无法在数据缺失时诚实呈现 |

## 本决策未解决的问题

- **释放后回看已保存会话与文件证据**：本文只读运行生命周期内的会话证据；Workspace 释放后能否回看、按
  什么保留契约，属于 #296 的成果契约（第二期）。本文的空态词汇与 #296 对齐，但不在本期定义保留窗口。
- **文件变更与 diff**：文件路径、变更类型、增删统计与可打开 diff 需要执行期采集与按写入归属的证据，
  并行节点共用工作区时时间窗 diff 也不能当独占写入证明——这些由 #296 承接，本文不读文件。
- **非 agent 节点（`output` / `condition`）的「结果」是否也需要会话式读取**：它们没有会话记录，本期只给
  「不适用」空态；若将来要展示它们的完整求值过程，另开决策。
- **旧终态运行缺失 `reason` 的呈现**：不回填历史 `node_states`，读取端对旧运行显示未知；若 owner 要求
  历史运行也能看到结束原因，需要按 #296 的归档回看一起决定，不在本期补造。

## 落地顺序

- **阶段 1（ADR 评审）**：本 ADR 以 `proposed` 提交 specs PR，只含正文，不碰 `test-cases/`（`proposed`
  阶段不编写核心测试用例）。完成条件是 PR 已开、owner 开始评审，改动文件只有这一份 ADR。
- **阶段 2（批准后补用例并合并）**：owner 批准后，在同一 PR 补 `test-cases/cloud/workflow/` 下的核心
  用例，状态改为 `approved` 再合并。用例至少覆盖：按尝试读取不串会话、授权拒绝跨节点/跨运行/直读、
  结束原因与输出分离、三种空态各自真实呈现、读取纯只读不隐式开启新对话。完成条件是每条不变量都有
  对应用例或明确的 `Missing`，`decisions` 与 `test-cases` 目录镜像一致，PR 由 owner 合并。
- **阶段 3（Cloud 实现，待开运行）**：新增公开读取路由与授权、`nodeStates` 的 `reason` 字段与
  `output` 收窄、前端节点详情接入会话渲染。完成条件是真实 PostgreSQL 的读取测试覆盖上述路径、
  OpenAPI 与前端生成物已同步、`task check` 全绿。
- **阶段 4（状态同步）**：实现合并后由落地运行把本文改为 `implemented`，核对偏离、精简正文，并按最终
  实现登记核心用例证据状态。
- **保持不变**：会话写入与结束/结算路径、`node_executions` / `node_event_receipts` 持久化形状与约束、
  非 agent 节点的 `output` 语义、取消与释放的既有结算，以及 Controller 线上契约。
