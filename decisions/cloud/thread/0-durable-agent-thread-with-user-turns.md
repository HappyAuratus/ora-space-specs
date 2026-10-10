---
status: approved
---

# Agent 运行的对话记为 Thread：Node 的会话记录逐条持久流入 Cloud，用户可追加消息，空闲或用户结束后收尾

一次 Agent IssueRun 的 Agent 会话称为 **Thread**。Node 把会话中每条已定型的记录（与 Ora 会话 JSONL
的行完全相同）作为有序、需确认的事件发回；Controller 接管进 Cloud 后才确认。Cloud 按序保存为 Thread
条目，Issue 页右侧面板经 REST 读取、经 SSE 失效提示实时刷新。用户可以在 Thread 中追加消息，Cloud 把它
作为命令经 Controller 交给 Node 排队执行；Agent 完成当前轮次后 Thread 进入空闲，用户结束或空闲期满时
Cloud 请求结束会话，进入 Revision 交付。

本 ADR 处于 `approved`（已评审，待实现），是新建领域 `cloud/thread` 的根决策。运行阶段由
[IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)承担；事件与命令的线上
形状由 [gRPC 后续决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)
与 [Node 协议后续决策](../../node/protocol/20260928-streamed-thread-events-and-session-commands.md)承担；
记录格式由 desktop `ora-history` 所有。本文不定义 token 级增量流、Thread 在运行结束后的续接，以及
Thread 以外的交互式聊天（`chat` 页）。

## 决策

### D1：每个 Agent IssueRun 恰有一个 Thread，条目有 Cloud 分配的连续序号

- 表 `thread_entries`：`(run_id, seq)` 主键，`seq` 从 1 连续递增，由 Cloud 在写入事务中分配；列
  `source`（`node | user | system`）、`kind`、`record jsonb`、`turn_id`、`node_execution_id`、
  `node_sequence`、`created_at`。`(node_execution_id, node_sequence)` 唯一。
- Thread 状态挂在 `issue_runs` 上：`thread_state` 为 `pending | active | idle | ending | ended`。
- 同一 Node 事件重复接管（重放、确认丢失）按 `(node_execution_id, node_sequence)` 幂等，不产生新条目；
  同键不同内容以 `CONFLICT` 拒绝，原条目不变。

为什么 Cloud 重新编号而不直接用 Node 序号：一个 Thread 的条目来自 Node 会话执行和用户两个来源，Node 的
序号只在一次执行内有意义；前端需要一个跨来源、无洞的顺序作为分页游标。

### D2：条目内容就是 Ora 会话记录的行，按定型记录粒度流式

- Node 发回的 `record` 是 `ora-history` 的 `HistoryLine` 原样 JSON（用户消息、Agent 消息、思考、工具
  调用最终快照、`TurnEnded`、`Gap` 等），`kind` 取其类型标签；Cloud 不解析业务字段，只校验是 JSON
  对象、`kind` 在已知集合内、单条不超过 256 KiB。
- 粒度是**定型记录**：一条 Agent 消息在该消息结束时整条到达，工具调用在终态时到达。这比逐 token 粗，
  但每条都能按 D1 持久接管和确认，刷新页面、断线重连、Controller 重启都不会丢或重复。
- 会话结束后上传到对象存储的会话 JSONL（Revision 的一部分）与 Thread 条目来自同一份记录，二者内容一致；
  Thread 是展示和对话用的在线副本，JSONL 是随 Revision 保存的归档。

为什么第一版不做 token 级增量：增量要么走 Node 事件的持久确认路径（每个 token 一次持久化与 Ack，代价
不可接受），要么另建一条至多一次的旁路（需要第二套乱序与断线语义）。定型记录已能支撑“看得到 Agent
在做什么、能回复它”，token 级体验留给后续决策以旁路方式叠加，不改变本条的持久语义。

### D3：首条 prompt 和用户追加消息都是用户轮次，由 Cloud 生成身份

- **首条 prompt**：运行进入 `starting` 时，Cloud 用 IssueRun 输入快照（Issue 标题、描述、任务文本、
  最近评论、上下文引用）按固定模板渲染为文本 content block，写入一条 `source = system`、`kind = user_turn`
  的条目，`turn_id` 由 Cloud 生成。它作为 Agent 会话执行输入的一部分下发，不单独排队。模板在 Cloud，
  使 Node 不需要理解 Issue 领域。
- **追加消息**：`POST /api/v1/tenants/{tid}/issues/{iid}/runs/{rid}/thread/messages`（必须带
  `Idempotency-Key`，body `{content: [{type: "text", text}]}`，文本合计 ≤ 64 KiB）。Thread 状态为
  `pending | active | idle` 时接受；Cloud 写入 `source = user`、`kind = user_turn` 条目（状态 `queued`）
  并在同一事务中经 `enqueueThreadCommand` 放出 Thread 命令 `SubmitUserTurn{turn_id, content}`（命令表与事务边界见
  [controller-integration D6](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)）。`ending | ended` 时返回 `409 thread_closed`。
- Node 执行某个用户轮次时写入的用户消息记录携带同一 `turn_id`；Cloud 接管到它时把对应 `user_turn`
  条目标为 `delivered`。前端据此区分“已发送、排队中”与“Agent 已接收”。
- Agent 正在执行上一轮时，新消息在 Node 上按接收顺序排队（[Node 协议](../../node/protocol/20260928-streamed-thread-events-and-session-commands.md) D4），不打断当前轮次。
- 会话结束时仍未被执行的用户轮次（排队中遇到结束或取消），在 Cloud 接管会话终态的事务中标为
  `discarded`，前端显示为未送达；它们不会在之后的任何会话中执行。

授权：能读取该 Issue 的活动 Space 成员都可以读 Thread 和追加消息，与评论权限一致。追加消息的用户记录在
条目上，但 Agent 执行时的 git 身份始终是触发运行的用户（[git 身份决策](../identity-access/20260928-user-git-commit-identity.md)），
第一版不按消息切换身份。

### D4：空闲期满或用户结束时 Cloud 请求结束会话

| 事件 | Thread 状态 |
|---|---|
| 会话执行已登记、尚无记录 | `pending` |
| 接管到任意 Node 记录 | `active` |
| 接管到 `TurnEnded` 且没有 `queued` 的用户轮次 | `idle`，记录 `idle_since`（数据库时间） |
| 新用户轮次写入 | `active` |
| 用户点击“结束”、`idle` 超过空闲窗口、或运行被取消 | `ending`，放出 `EndSession{reason}` 命令 |
| 会话执行终态被接管 | `ended` |

- 空闲窗口是 Cloud 配置 `issue_runs.thread_idle_timeout`，第一版默认 15 分钟；由 Cloud 的派发循环
  按数据库时间判断，不依赖 Controller 或 Node 的时钟。
- `reason` 为 `user_ended | idle_timeout | cancelled`；Node 对三者都停止 Agent、完成会话记录后以终态结束
  会话执行，区别只影响 IssueRun 的 `status`（[IssueRun D4](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)）。
- Agent 在首轮以非正常原因结束（拒绝、失败、连接丢失）时，Thread 同样进入 `idle`，由用户决定重试
  （追加消息）或结束；不自动结束，因为用户可能想让 Agent 继续。

为什么不在 Agent 首轮 `EndTurn` 时立即结束：用户明确要求在 Thread 中对话；一轮结束后立即释放沙盒，
用户的回复就只能开启一次新运行，并重新 clone 和安装插件。空闲窗口让“快速回复”复用同一沙盒，代价是
每次运行至少多占用一个窗口的沙盒时间。

### D5：公开读取与实时刷新

- `GET .../runs/{rid}/thread?after={seq}&limit={n}` 按 `seq` 升序返回条目与 `threadState`；
  `limit` 上限 500。
- 条目写入事务提交后，Cloud 在 Space 事件流上发布 `issue_run.thread_appended{issueId, runId, lastSeq}`；
  事件只是失效提示，前端据此从自己的游标继续 `GET`，与 Space API 根决策中“事件只带失效信息，权威状态
  经 REST 重取”一致。
- Issue 详情页右侧面板列出该 Issue 的 Agent 运行，选中一个显示其 Thread；输入框在 `pending | active | idle`
  时可用。

## 不变量

1. Thread 条目的 `seq` 在一个 Thread 内连续、不重排；同一 Node 事件至多产生一个条目。
2. Node 的 Thread 事件只有在 Cloud 事务提交后才会被确认；未提交的事件会重放。
3. 用户轮次在 Cloud 持久化后才会下发，`turn_id` 由 Cloud 生成并贯穿到 Node 记录。
4. Thread 进入 `ending` 后不再接受用户轮次；`ended` 由会话执行的终态决定。
5. 空闲期满由数据库时间判断。
6. Cloud 不改写 Node 记录的内容。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 运行结束后只从上传的 JSONL 回放 | 无需新的事件路径；但运行中看不到进展、也无法对话，不满足本轮需求 |
| token 级增量走持久事件路径 | 体验最好；每个 token 一次持久化与确认，吞吐和存储代价不可接受（D2） |
| Thread 条目复用 `issue_comments` | 已有展示；但工具调用、思考等大量记录会淹没评论时间线，且评论有编辑、删除语义，与只追加的会话记录冲突 |
| Controller 直接转发用户消息，不在 Cloud 落库 | 少一次写入；但 Controller 重启会丢消息，也无法向前端表达排队状态 |
| Agent 首轮结束即结束会话 | 沙盒占用最短；用户无法在同一沙盒继续对话（D4） |

## 本决策未解决的问题

- token 级增量显示的旁路设计。
- Thread 条目与 JSONL 的保留期限、Thread 的总大小上限。
- 运行结束后在 Thread 中继续对话（需要从 Revision 续接）。2026-10-10 补充：新运行续接前序 Revision 已由 [IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md) 提出（proposed）；同一个 Thread 在运行结束后继续对话仍未决。
- 权限请求的人工审批：第一版沿用 desktop 运行时的自动批准，Thread 只显示结果。

## 落地顺序

1. 迁移 `thread_entries` 与 `issue_runs.thread_state/idle_since`；接管路径与幂等以 PostgreSQL 集成测试覆盖
   （重复接管、同键不同内容、`seq` 连续、空闲期满放出结束命令）。
2. 公开 API、OpenAPI 与前端生成客户端；SSE 事件。
3. 前端 Issue 详情页右侧 Thread 面板（列表、输入、状态），以 mock 数据的组件测试覆盖状态显示。

保持不变：评论、Timeline 活动与 manual run。开始改变：Agent 运行有可读、可对话的 Thread。
