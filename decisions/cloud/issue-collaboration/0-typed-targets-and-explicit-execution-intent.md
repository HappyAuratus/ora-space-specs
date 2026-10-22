---
status: implemented
---

# Issue 协作：`@` 只选择类型化目标，执行只来自显式意图

> 历史决策：D9 中「活稿与所选版本不一致时返回 `409 workflow_version_fields_changed`」已由
> [Confirm 只按所选发布版本自身的 schema 校验](20261020-confirm-validates-against-the-chosen-version.md) 修改，
> 该部分当前系统承诺以后续决策为准；本决策其余内容与状态 `implemented` 不变。
>
> 历史决策：D8 规则 1 对保留键 `snapshotId` 不再适用（作者声明不生效，键归平台）；规则 2、规则 3 也被修改
> （全部已发布版本都不可选时不注入选择器，选项只列未占用保留键的版本）；规则 7 中「作者声明该键就跳过旧名
> 折叠」的 guard 随之失效。见 [snapshotId 是保留字段](20261021-snapshot-id-is-a-reserved-field.md)。该改写已随
> cloud#123 落地，D8 的上述规则不再按原文执行。

Issue 是多人与多种能力协作的地方。Cloud 的 Issues 模块只拥有 Issue 领域的状态（Issue、评论、交互、运行记录、
活动与 Timeline、上下文引用），Agent、Team、Workflow、运行时、通知、实时推送都经消费侧端口接入。`@` 是**选择协作
目标**，不是执行触发：目标类型决定交互模式，人只被提及，Agent/Team 必须带明确任务才产生运行，Workflow 先渲染表单，
只有用户显式 Confirm 才产生运行。一条评论可以对应多个交互，一个交互至多对应一个初始运行。Timeline 是评论与活动按
每个 Issue 共用的 `seq` 排序的高层投影，与执行日志严格分开。

本决策为 `implemented`：它把 Cloud 在 Wave 3 期间冻结并已上线的协作契约补登为本模块的根决策，原文是 cloud 仓库的
`docs/migrations/multica-issue-board/12-collaboration-architecture.md`（§6.4、§8、§37、§38，随
ora-space/circus#79 删除）。正文只保留当前系统承诺；实现过程、阶段记录和已被取代的草案不再保留。涉及 cloud 的
公开 API（`/collaboration/*`、`/issues/{iid}/comments|interactions|timeline|context-refs`）、`internal/core`
的协作端口与 `issue_interactions`/`issue_runs`/`issue_activities` 的写入规则，以及前端的 `@` 选择与 Workflow
表单。本决策不改变任何既有迁移，不承担数据迁移。

Agent 运行在运行 Workspace 里的编排、Thread 与交付由 [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)
与 [Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md) 规定；Workflow 的发布、启动门禁、执行与
`snapshotId` 命名由 [工作流执行根决策](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md) 规定。本决策
只规定 Issue 一侧如何选择目标、记录交互、产生运行意图和展示结果。

## 背景

Issue 看板从 Multica 迁入时只有人类评论和单一用户负责人。要让 Issue 成为“执行可观察”的协作空间，需要同时回答：
谁可以被 `@`、`@` 之后发生什么、一条评论和若干次运行如何对应、Workflow 这种需要结构化输入的能力怎样被安全调用，
以及在 Agent/Team/Workflow 模块尚未存在或各自演进时，Issues 如何不替它们做设计。Multica 的做法（执行者恒为 agent
外键、评论正文正则触发、`comment` 与执行一一对应）在这些问题上都会把外部模块的形状固化进 Issue 表。

## 决策

### D1：Issues 只拥有 Issue 领域状态，外部能力经端口接入

Issues 拥有：Issue、状态目录、评论、交互（`issue_interactions`）、运行记录（`issue_runs`）、活动与 Timeline 投影、
Issue 级上下文引用（`issue_context_refs`）、运行的触发来源。Agent、Team、Workflow 的领域模型，运行时、Node、沙盒、
模型提供方，通知投递、实时连接，PR/Git 提供方，执行日志存储都不归 Issues，即使对应模块尚不存在。

外部能力都经消费侧端口接入。每个端口有三种状态：`Unavailable`（契约存在、能力不可用，返回明确的不可用结果）、
`Simulator`（确定性的开发/演示适配器）、`Real`（真实模块）。适配器只在装配处（`core.NewStore`、`cmd/*`）选择；
Issue 业务代码不分支判断“哪个后端存在”。

| 端口 | 方向 | 职责 | 当前实现 |
|---|---|---|---|
| `CollaborationDirectory` | 出 | 列出和解析可选的非人类目标，返回统一的 `CollaborationTargetSummary` | 已实现：Space Agent 名册在前、Workflow 目录在后（`DirectoryChain`）；Team 只有开发夹具 |
| `InteractionDescriptor` | 出 | 目标被选中后进入哪种模式、是否需要任务、表单的 `formRef` | 已实现（值类型，随目标投影返回） |
| `FormDescriptorProvider` | 出 | 按 `formRef` 解析面向 Issues 的 `FormDescriptor` | 已实现：由 Workflow 的 Start 节点投影 |
| `InputAssistProvider` | 出 | 为表单提供只读建议 | 端口已实现；只有确定性的开发夹具，真实提供方未定 |
| `ContextBuilder` | 出 | 由 Issue 材料、任务或表单值、上下文引用构造调用时上下文 | 已实现：确定性构造器 |
| `ExecutionDispatcher` | 出 | 把已准备好的运行交给外部执行者 | 已实现；Agent 运行改走 IssueRun 编排 |
| `ExecutionObserver` | 入 | 把外部执行的生命周期映射回运行状态和 Timeline | 已实现；不是 HTTP 路由 |
| `ActorResolver`、`CollaborationTargetResolver`、`WorkflowResolver`、`NotificationSink`、`RealtimePublisher`、`ProjectContextResolver`、`PullRequestResolver`、`ExecutionLogProvider` | 出 | 作者/负责人解析、目标校验、Workflow 领域调用、通知、实时事件、项目上下文、PR、执行日志 | 只有概念，没有代码；接入时沿用同一三态模型 |

开发夹具（`internal/collab`）只在开发/演示装配中启用，生产默认关闭；它们不建任何领域表或 `sim_*` 表，删除夹具装配
不影响 Issue 核心。

**为什么**：端口让 Issues 只描述“需要什么”，Agent/Team/Workflow 模块可以各自落地而不改 Issue 表和 UI 契约；三态
模型让“能力缺席”成为可展示的明确状态，而不是散落在业务代码里的条件分支。

### D2：作者与目标是两种引用，Workflow 不是参与者

- `ActorRef ∈ {user, agent, team, system}`：作者、负责人、活动发起者。
- `CollaborationTargetRef ∈ {user, agent, team, workflow}`：`@` 的对象、调用对象。

两者都以 `type + id` 存储，不建跨模块外键，经对应端口校验。**Workflow 只出现在目标一侧**：它是带输入结构的可调用
能力，不是协作者。因此 Workflow 运行的活动以 `system` 为发起者，在 `details` 里记录 `{runId, executorType, executorId}`；
Workflow 的人类可读输出是 `run.message` 活动，不是评论。Agent/Team 的回复是 `author_type ∈ agent|team` 的评论，
参与同一 Timeline 排序。客户端不能指定作者类型冒充 Agent 或 Team。

运行的执行者是一对多态引用 `executor_type ∈ {agent, team, workflow}` + `executor_id`，没有外键；`issue_runs`
只允许不透明的外部引用（`external_execution_id`、`execution_context_ref`、`workflow_invocation_ref`），不保存运行时、
沙盒、Node、模型、提供方等执行系统内部标识。

### D3：`@` 选择目标，模式由服务端按目标决定

| 目标类型 | 模式 | 是否产生 IssueRun |
|---|---|---|
| `user` | Mention：提及，持久化类型化目标 | 否，禁止 |
| `agent` | Task：必须带非空任务 | 是，评论提交时产生一次初始运行 |
| `team` | Task：与 Agent 相同 | 是 |
| `workflow` | Form：渲染表单，经 Review 后 Confirm | 只在 Confirm 时产生 |

- 模式由服务端从目标类型和目录返回的 `InteractionDescriptor` 推出，客户端从不发送 `mode`；目标都经
  `CollaborationDirectory` 服务端解析，客户端不能虚构目标或能力。
- 评论正文里的 `@xxx` 或 `mention://` 只用于显示，不参与路由：没有 `targets` 项的评论不触发任何交互。
- Task 模式下评论正文和交互的 `task` 是两个字段。UI 可以用正文初始化任务，但任务是独立的执行意图。
- `GET /collaboration/targets` 是面向 Issues 的只读目标投影：人类目标来自真实成员关系，其余来自目录。它不是 Agent、
  Team 或 Workflow 的领域 API，不携带领域字段，也不提供写操作。
- 指派负责人（`assignee`）只改变责任，不启动执行（Assign ≠ Execute）。执行只来自显式动作：手动创建运行、`@`
  Agent/Team 并给出任务、Workflow Confirm。

### D4：评论、交互、运行三者分离

```
一条 Comment      → 0..N 个 Interaction（每个被选中的目标一个）
一个 Interaction  → 0..1 个初始 IssueRun
因此一条 Comment  → 0..N 个 IssueRun
```

- 每个目标一行 `issue_interactions`（`target_type`、`target_id`、`mode`、`task`、`input`、`run_id`）。**不引入
  `comment.run_id`**：一对一列表达不了提及（0 个运行）和一条评论多个目标（多个运行）。
- 交互没有状态列。`run_id` 既是确认标记也是初始运行引用：`mode='form'` 且 `run_id IS NULL` 表示已选中未确认；
  `run_id IS NOT NULL` 表示已确认；执行进度由 `issue_runs.status` 承载。
- 运行的来源记在 `trigger_evidence_kind` + `trigger_evidence_ref_id`：评论触发为 `comment` + 评论 id，Confirm 触发为
  `interaction` + 交互 id。
- 同一 `(issue_id, executor_type, executor_id)` 在 `queued|dispatched` 状态下至多一个运行（部分唯一索引
  `issue_run_pending_uniq`），冲突返回 `409 pending_run_exists`。一条评论里的不同目标因 `executor_id` 不同各得一个
  运行；同一目标重复出现不产生两个并发的待执行运行。

### D5：上下文分三层，运行输入是快照

| 概念 | 是什么 | 是否持久化 |
|---|---|---|
| `IssueContextRef` | Issue 上稳定的显式引用，由用户经 context-refs API 增删 | 是，`issue_context_refs` |
| `ContextBuilder` | 调用时构造上下文的唯一接缝 | 否 |
| `IssueRun.input` | 这次运行实际收到的输入快照 | 是 |

- Task 模式和 Form 模式共用同一个 `ContextBuilder` 与 `ContextInput`（Task 填 `Task`，Form 填 `InteractionValues`），
  不分叉出 Workflow/Agent 专用的构造器。确定性构造器只使用 Issue 标题、描述、最近评论、显式上下文引用、任务或表单值。
- 运行始终带上 Issue 的全部 `issue_context_refs`；Confirm 请求里的 `contextRefs` 是追加在其后的额外引用，只进入这次
  运行的输入快照，不写成 Issue 级引用。
- `FormDescriptor` 从不作为运行输入持久化。

### D6：Timeline 是按 `seq` 排序的高层投影

- Timeline = 评论 + 活动，二者共用每个 Issue 一个 `seq` 命名空间，`GET /issues/{iid}/timeline` 按 `seq, id` 排序；
  `created_at` 只用于显示。
- 外部执行经 `ExecutionObserver` 映射：`started → running` + `run.started`；`progress` → `run.progress`（不改状态）；
  `message` → Agent/Team 为回复评论、Workflow 为 `run.message` 系统活动；`completed`/`failed`/`cancelled` → 对应终态
  + 同名活动。运行入队写 `run.enqueued`。
- Timeline 只放粗粒度、人可读的条目。逐节点日志、stdout/stderr、工具调试输出属于执行模块，只经
  `ExecutionLogProvider` 引用，不进 Timeline。

### D7：Workflow 表单是面向 Issues 的渲染描述，不是 Workflow schema

`FormDescriptor = {formRef, title?, description?, fields[]}`，`fields` 的数组顺序就是显示顺序。字段为
`{key, label, type, required, description?, defaultValue?, placeholder?, options?}`，类型限于 `text`、`textarea`、
`number`、`boolean`、`select`、`multi_select`。描述必须满足：`key` 唯一；`select`/`multi_select` 有且仅有非空
`options`；`defaultValue` 与类型一致且在选项内；字段数不超过 100。违反时整个描述无效（`500 invalid_form_descriptor`），
不渲染半合法的表单。

- 它只回答“前端该画哪些控件、做哪些最基本的校验”。不含 JSON Schema、`$ref`、表达式、条件显示、计算字段、嵌套
  或重复组；Workflow 内部用什么表达 schema 由适配器自行映射。
- `formRef` 对 Issues 不透明，经目标投影的 `InteractionDescriptor` 下发；当前就是 Workflow id。描述按需读取：
  `GET /collaboration/forms/{formRef}[?issueId=]` 投影 Workflow 的**活稿**；已选定版本的调用方读
  `GET /workflows/{wfid}/snapshots/{snapshotId}/form`，它与 Confirm 校验用的是同一个投影函数，响应额外回显
  `snapshotId`。读取描述不是启动门禁，不创建任何东西。
- 前端只按描述渲染控件，不按 Workflow id 分支、不写死任何字段；未知字段类型显示为错误状态，不做猜测。
- 表单值是以字段 `key` 为键的扁平对象，值为 JSON 标量或字符串数组。Issue 领域不为任何 Workflow 新增列。
- 失败语义：能力未装配时目标投影不提供 Workflow；提供方暂时失败 `503 form_descriptor_unavailable`；`formRef` 未知
  `404 form_descriptor_not_found`；提供方返回非法描述 `500 invalid_form_descriptor`。

### D8：发起表单的字段目录

带 Issue 上下文（`?issueId=`，或 Assist、Confirm 时的所属 Issue）读取描述时，平台在作者的 Start 变量之前注入一组
字段；不带 Issue 上下文时一个都不注入，描述与没有这些字段时逐字节相同。

| 字段 | key | 类型 | 默认必填 | 默认值 | 注入条件 |
|---|---|---|---|---|---|
| 仓库地址 | `repository` | `text` | 是 | Issue 所属 Project 的 `repository_url` | 总是 |
| 分支 | `branch` | `text` | 是 | 同一 Project 的 `default_branch` | 总是 |
| 运行版本 | `snapshotId` | `select` | 否 | 最新发布的快照 | 至少有一个已发布快照 |
| 提示词 | `prompt` | `textarea` | 否 | Start 节点的初始提示词；留空表示用 Workflow 自己的 | 总是 |
| 补充上下文引用 | `context_refs` | `multi_select` | 否 | 不选 | Issue 有 Project 或父 Issue |
| 每个全局变量一项 | `<global.name>` | 按 `valueType` | 否 | 变量声明的值 | 图里声明了全局变量 |

顺序为：平台字段 → 作者的 Start 变量 → 全局变量；超过 100 个字段时后面的部分不再追加。规则：

1. **作者声明优先**：Start 节点自己声明了某个平台 key 时，该 key 的控件、标签和必填性归作者；平台只在默认值能通过
   作者控件校验时补默认值。全局变量名必含 `.`，与 Start 变量不会冲突。
2. **无可选项就不注入**：没有已发布快照就没有 `snapshotId`，Issue 不能引用任何东西就没有 `context_refs`。这是唯一
   合法的渲染，不是降级。
3. **`snapshotId` 的值是快照 id**，版本号只出现在选项标签（`v3 · <name>`）。选项列出全部已发布版本，与读哪个投影无关。
4. **`context_refs` 只能追加**：选项只有 Issue 自己的 `project_ref` 和 `parent_issue_id`，值为 `refType:refId`，
   提交时转为 Confirm 的 `contextRefs` 并与 Assist 应用的引用去重。
5. **作者可以按 key 声明 `launchFields`**：图信封里的 `launchFields: [{key, enabled?, required?}]`，`key` 限于
   `repository`、`branch`、`snapshotId`、`prompt`、`context_refs`。逐 key 覆盖，未提及的 key 保持平台默认；无法识别
   的条目被丢弃。声明不能让平台注入它没有可选项的字段；对作者自己 Start 节点声明的 key 无效。声明读自活稿
   `workflows.graph`，保存即生效，不随快照或运行保存。被撤下的 key 在 Assist 和 Confirm 中都是未知 key（`400
   invalid_field_value`）。
6. **保留 key**：`repository`、`branch`、`snapshotId`、`prompt`、`context_refs` 是平台 key。执行方必须显式映射它们，
   不能假定 `issue_interactions.input` 的每个 key 都是 Start 变量。
7. **旧名兼容**：按[工作流执行根决策](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md)，版本字段由
   `version` 改名为 `snapshotId`。已发布快照里写着 `version` 的 `launchFields` 声明按 `snapshotId` 读取；Confirm 收到
   旧 key `version` 时，只要作者的 Start 节点没有声明同名变量，就把它当作版本选择并在校验和持久化前折叠到
   `snapshotId`（已带 `snapshotId` 时以后者为准）。Assist 不接受旧 key。历史交互的 `input` 不改写。

### D9：选择、草稿、Assist 都不执行，Confirm 是唯一执行边界

```
选择 Workflow → 加载描述 → 填写（草稿）→ 可选 AI Assist → 用户应用建议 → Review → Confirm → IssueRun
```

- **选择只在客户端暂存**：不写评论、不写交互、不建运行；放弃表单不留任何痕迹。表单值是前端草稿，服务端不保存未确认的值。
- **AI Assist**：`POST /issues/{iid}/collaboration/assist`，按 `targetId` 无状态调用；只校验当前值的形状，不要求必填
  完整。它只返回建议：按字段的部分补丁（未提及的字段保留用户的值；只保留描述上存在的 key）、建议的上下文引用、
  可选的解释。Assist 不写评论、交互、运行、上下文引用，不改 Issue 或运行状态，不发通知，不调用外部系统。未装配
  提供方时为 `503 assist_unavailable`。应用建议是用户动作，建议的上下文引用应用后只进入这次 Confirm 的载荷。
- **Review**：Confirm 前有独立的 Review 步骤，展示目标、生效的值、选中的上下文引用以及哪些值来自建议。任何字段变化
  都不是执行触发。
- **Confirm**：客户端先提交带 Workflow 目标的评论（产生 `mode='form'`、`run_id IS NULL` 的交互），随即调用
  `POST /issues/{iid}/interactions/{ixid}/confirm`。服务端在一个事务内：按租户和 Issue 定位交互（他人的或不存在的
  一律 404）；非 Form 交互 `409 interaction_not_confirmable`；按工作流执行根决策的启动门禁解析版本；活稿表单与该
  版本对提交有影响的属性（key 集合、控件类型、必填性、选项集合）不一致时 `409 workflow_version_fields_changed` 并列出
  字段；按该版本的描述完整校验（`400 required_field_missing` / `invalid_field_value`）；构造运行输入并冻结版本；入队
  运行；以 `UPDATE … SET input, run_id WHERE run_id IS NULL` 比较交换认领交互，0 行即 `409 interaction_already_confirmed`。
  外部派发在提交之后进行。
- Confirm 必须带 `Idempotency-Key`：同一 key 同一请求重放原响应，不产生第二个运行；同一 key 不同请求
  `409 idempotency_conflict`。两个标签页同时 Confirm 由事务串行化和比较交换保证只有一个初始运行，`issue_run_pending_uniq`
  是第二层保护。

## 不变量

1. `@` 人类目标不产生 IssueRun。
2. 运行只由显式动作产生：手动创建、`@` Agent/Team 带任务提交、Workflow Confirm；指派负责人、选择目标、编辑表单、
   AI Assist 都不产生运行。
3. 交互模式和目标能力由服务端解析，客户端不能设置 `mode`、虚构目标或伪造描述；前端描述只是渲染提示，不是授权来源。
4. 一个交互至多一个初始运行；`run_id` 由 `NULL` 变为非空只发生在 Confirm（或 Task 模式的评论提交）中，且只发生一次。
5. Confirm 总是用服务端当前解析的描述重新校验提交的值。
6. AI Assist 没有任何写副作用。
7. Workflow 不是 `ActorRef`；Workflow 运行的活动以 `system` 为发起者并在 `details` 中记录 Workflow 身份。
8. Timeline 只按 `seq` 排序，只包含评论和活动，不包含执行日志。
9. `FormDescriptor` 不持久化；确认后的表单值存于 `issue_interactions.input`，运行实际收到的输入存于 `issue_runs.input`。
10. Issue 领域表不出现任何 Workflow 专属列，也不出现 Agent/Team/Workflow 的外键。
11. `ExecutionObserver` 不暴露为 HTTP 路由，客户端不能伪造执行结果。
12. 开发夹具不在生产默认装配中，不建领域表。

## 为什么不是这些替代方案

| 替代方案 | 否决理由 |
|---|---|
| 照搬 Multica：`issue_runs` 用 `agent_id` 外键，Team/Workflow 再加可空列 | 把外部模块的表结构耦合进 Issue 表，每加一类执行者就改一次列；多态 `executor_type + executor_id` 让 UI 与存储契约保持 `{type, id}` 不变（D2） |
| 正文正则或 `mention://` 作为路由依据 | 正文可编辑、可伪造，同一文本无法区分“提及”和“委托”；路由只认类型化的 `targets`（D3） |
| `comment.run_id` 一对一 | 表达不了提及（0 个运行）和多目标（多个运行），并把交互模型冻结在评论表里（D4） |
| 交互加 `draft/ready/confirmed/executing` 状态列 | 与 `run_id IS NULL` 和 `issue_runs.status` 重复，两套状态机会漂移（D4） |
| 把 Workflow 当作 `ActorRef`，输出写成评论 | 评论没有承载 Workflow 来源的字段，需要迁移；也把一个能力当成了对话参与者。活动的 `details` 天然承载来源（D2） |
| 在目标投影里内嵌完整 `FormDescriptor` | 每次打开选择器都要解析所有 Workflow 的表单，真实 Workflow 模块可能很慢；`formRef` + 单独读取把代价推迟到选中时（D7） |
| 服务端保存表单草稿 | 每次输入都要带版本的持久化写入，而用户可能放弃；交互本身是否存在已足够，丢失的只是未确认的值（D9） |
| 描述带版本令牌做陈旧检测 | 当前由 Confirm 按所选发布版本重新校验（D9，后续 [Confirm 只按所选发布版本自身的 schema 校验](20261020-confirm-validates-against-the-chosen-version.md) 已删除活稿比较与显式字段差异报告）替代；需要更早发现时再由提供方给出令牌 |
| 平台字段目录写死在代码里，对所有 Workflow 一样 | 不同 Workflow 对版本、提示词、仓库的需要不同；逐 key 覆盖的 `launchFields` 让作者决定，又不会让新加入的平台字段对已保存的 Workflow 不可见（D8） |
| 为 Workflow 单独建 `WorkflowContextBuilder`、`WorkflowDispatcher` | 把一个抽象分叉成平行系统；Workflow 差异只在适配器层（D5） |

## 本决策未解决的问题

- **隐式续聊（ConversationTarget）**：不带 `@` 的后续评论是否继续发给当前执行者，作用域（每 Issue、每用户、每会话）
  和持久化都未定；当前只有显式 `@` 才产生交互。
- **通知**：Mention 只持久化和展示，不投递；通知模块接入 `NotificationSink` 时需要能深链回 Issue 与上下文。
- **真实 AI Assist 提供方**：Cloud 没有自己的模型调用方，归属未定；当前只有确定性夹具。
- **Team 的真实语义**：Team 是否有自己的执行身份、是否分派给成员，留给 Team 模块的决策；当前 Issues 只把 Team 当作
  能接任务的目标，生产中没有 Team 目标。
- **同一 Issue 同一执行者的并发运行**：如果真实执行者需要，另做前向迁移和决策。
- **表单能力扩展**：上下文引用选择器字段类型、条件显示、计算字段、重复组都需要新的决策，不是本决策的扩展。
- **Workflow 作为对话参与者写评论**：需要先有评论来源的承载字段。

## 落地状态

全部决策已在 cloud main 实现并有集成测试，直接证据见[核心用例](../../../test-cases/cloud/issue-collaboration/collaboration-interactions.md)。
迁移 `0010_issue_collaboration.sql`（`issue_runs`、`issue_activities`、`issue_run_pending_uniq`）、
`0011_issue_interactions.sql` 与 `0012_issue_interaction_input.sql` 承载持久化部分；三个迁移文件的注释仍引用已删除的
cloud 文档章节号，迁移不可改写，以本决策为准。
