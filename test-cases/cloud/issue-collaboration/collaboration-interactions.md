# Issue 协作交互

依据[Issue 协作根决策](../../../decisions/cloud/issue-collaboration/0-typed-targets-and-explicit-execution-intent.md)。
证据来自 `ora-space/cloud`：`integration/collaboration_interaction_test.go`、`integration/workflow_interaction_test.go`、
`integration/workflow_snapshot_form_test.go`（真实 HTTP 与 PostgreSQL，校验 OpenAPI 契约），以及
`internal/core/workflow_collaboration_test.go`、`internal/core/workflow_launch_fields_test.go`、
`internal/collab/wire_test.go`。Agent/Team 的执行由开发夹具 `MockExecutionDispatcher` 驱动，这里只证明 Issue 一侧的
交互、运行意图与 Timeline；Agent 运行的真实编排见 [IssueRun 核心用例](../issue-run/agent-run-orchestration.md)。

## Mentioning A Human Never Creates A Run

### 风险

`@` 一个人被当作执行请求，产生运行或执行副作用。

### 前置状态

Issue 存在，被提及的人是租户成员。

### 触发

提交带 `targets: [{type: "user", id}]` 的评论。

### 必须成立

- 记录一条 `mode='mention'`、`run_id IS NULL` 的交互，Timeline 能展示类型化的被提及者。
- 不产生运行，不派发。

### 禁止结果

为人类目标创建 IssueRun；按正文里的 `@` 文本路由。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 提及只持久化目标、不产生运行 | `TestHumanMentionPersistsTypedTarget` | Covered |
| 正文 `@` 文本不触发交互 | 缺口：没有“正文含 `@Agent` 但无 `targets`”的单独用例 | Missing |

### 决策依据

D3；不变量 1、2。

## Agent And Team Tasks Create One Run Per Target

### 风险

Task 交互缺少任务仍然执行；同一目标被重复入队；多目标评论只产生一个运行。

### 前置状态

Issue 存在，目录能解析 Agent/Team 目标。

### 触发

提交带 Agent 或 Team 目标和 `task` 的评论；或对已有待执行运行的同一执行者再次入队。

### 必须成立

- 每个 Task 目标一条交互和一个初始运行，来源记为 `comment` + 评论 id；任务为空时拒绝。
- 同一 `(issue, executor_type, executor_id)` 已有 `queued|dispatched` 运行时 `409 pending_run_exists`。
- Agent/Team 回复是 `author_type ∈ agent|team` 的评论。

### 禁止结果

无任务的执行；同一执行者在同一 Issue 上两个待执行运行。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Agent 任务产生运行与回复评论 | `TestAgentTaskRunsThroughMockExecution` | Covered |
| Team 任务与 Agent 相同 | `TestTeamTaskRunsThroughMockExecution` | Covered |
| 任务必填、目标必须可解析 | `TestCollaborationTargetValidation` | Covered |
| 待执行运行去重 | `TestPendingRunDedup` | Covered |

### 决策依据

D3、D4；不变量 2、4。

## Clients Cannot Choose Modes, Targets Or Authors

### 风险

客户端自报模式、虚构目标能力、冒充 Agent 作者、确认其他租户或其他 Issue 的交互、伪造执行结果。

### 前置状态

两个租户；Issue 上有各种模式的交互。

### 触发

提交带 `authorType`、未知目标、非 Form 交互的确认、跨租户或跨 Issue 的交互 id。

### 必须成立

- 模式由服务端从目标推出；未知目标 404；作者类型不可由客户端指定（400）。
- 确认只在本租户本 Issue 内定位交互，其余一律 404；非 Form 交互 `409 interaction_not_confirmable`。
- 执行结果只能经 `ExecutionObserver` 进入，没有对应 HTTP 路由。

### 禁止结果

客户端决定模式或作者；跨租户确认成功。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 不能冒充作者 | `TestSecurityCannotImpersonateActorAndFailureLifecycle` | Covered |
| 目标服务端解析 | `TestCollaborationTargetValidation` | Covered |
| 确认的租户、Issue 与模式约束 | `TestWorkflowConfirmScopeAndModeGuards` | Covered |
| Observer 不可经 HTTP 调用 | 缺口：没有断言路由表中不存在 `Observe*` 路由的测试 | Missing |

### 决策依据

D2、D3、D9；不变量 3、11。

## Selecting Or Assisting A Workflow Never Executes It

### 风险

选择 Workflow、填写表单或 AI Assist 产生运行、交互输入或 Issue 级上下文引用。

### 前置状态

Workflow 已发布；Issue 存在。

### 触发

提交带 Workflow 目标的评论；调用 `POST /issues/{iid}/collaboration/assist`。

### 必须成立

- Workflow 目标只产生 `mode='form'`、`run_id IS NULL` 的交互，没有运行。
- Assist 只返回描述上存在的 key 的建议，不写交互输入、运行、上下文引用；未装配提供方时 `503 assist_unavailable`。

### 禁止结果

在 Confirm 之前出现运行；Assist 留下任何写入。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 选择不执行 | `TestWorkflowCommentCreatesFormInteractionWithoutRun` | Covered |
| Assist 无副作用 | `TestWorkflowAssistIsSideEffectFree`：断言无运行、无输入、`issue_context_refs` 为空 | Covered |
| Assist 不可用 | `TestWorkflowAssistUnavailable` | Covered |
| 选择在客户端暂存、放弃不留痕 | 前端行为，缺口：没有端到端证据 | Missing |

### 决策依据

D9；不变量 2、6。

## Confirm Creates Exactly One Run From Revalidated Values

### 风险

前端未校验的值进入运行；双击或重放产生两个运行；用旧版本的 schema 校验用户看到的新表单。

### 前置状态

Issue 上有未确认的 Form 交互；Workflow 已发布，可能已有新草稿。

### 触发

`POST /issues/{iid}/interactions/{ixid}/confirm`，含非法值、重复 key、并发、所选版本与活稿字段不同（含选旧版本）等情形。

### 必须成立

- 按版本描述完整校验，非法值 400，不建运行。
- 合法确认恰好一个运行，交互 `input` 与运行输入快照一致，来源为 `interaction` + 交互 id，版本被冻结。
- 同 key 同请求重放原响应；同 key 不同请求 `409 idempotency_conflict`；再次确认 `409 interaction_already_confirmed`。
- 只按所选版本校验：选中字段与活稿不同的旧版本，提交符合该版本即通过；缺该版本必填字段返回 `400 required_field_missing`，提交该版本未声明的字段返回 `400 invalid_field_value`，都不建运行。
- 有平台版本选择器时，客户端渲染的表单与校验依据同源：未手动切换版本时也按默认选中的发布版本读取，活稿里未发布的字段不出现在默认视图。
  保留键 `snapshotId` 由平台持有，活稿里的同名声明不改变这一点：`@` 表单始终投影平台选择器，Confirm 的版本游标一律读这个键；用启动字段撤下选择器时界面没有选择器，不提交该键执行最新已发布版本，提交了仍按版本游标读取（见 D1）。
- Confirm 解析到的已发布版本如果在自己的冻结图里声明了 `snapshotId`，确认返回 `409 workflow_version_reserved_field`（参数带被解析到的版本与字段名），不建运行、不认领交互；带不带版本游标都一样，也不改选别的版本。

### 禁止结果

两个初始运行；按用户没见过的字段报错。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 重新校验并恰好一个运行 | `TestWorkflowConfirmValidatesAndExecutes` | Covered |
| 幂等与二次确认 | `TestWorkflowConfirmIdempotencyAndDoubleConfirm` | Covered |
| 只按所选版本校验 | 真实 HTTP + PostgreSQL：`TestWorkflowConfirmValidatesAgainstTheChosenVersion` —— 旧版本按自身 schema 通过并记录该版本；缺该版本必填字段被 `required_field_missing` 拒绝、提交它未声明的字段被 `invalid_field_value` 拒绝，两条都断言不建运行且同一交互重新按该版本确认成功 | Covered |
| 客户端默认视图也按所选发布版本渲染 | 前端组件用例：`opens on the default published version by reading that version, so rendering and validation match` 断言默认视图读取 `/snapshots/:id/form` 并按该版本的字段确认 | Covered |
| 并发确认只成功一个 | 真实 HTTP + PostgreSQL：`TestWorkflowConfirmConcurrentInvocationCreatesExactlyOnePair` —— 两个并发确认只产生一个 WorkflowRun 和一个 IssueRun | Covered |
| 声明了保留键的版本不被执行 | 真实 HTTP + PostgreSQL：`TestWorkflowReservedFieldWithholdsStockVersionsAndRefusesThem`——不带游标解析到最新存量版本、带游标显式选中它，两条都返回 `409 workflow_version_reserved_field`，参数带版本与字段，且不建运行、交互仍未认领；`TestWorkflowReservedFieldStaysThePlatformsWhenTheDraftDeclaresIt`——活稿仍声明该键时同样拒绝，可选版本照常执行（cloud#123 未合入 main，按 `Missing` 记） | Missing |

### 决策依据

D4、D9；不变量 4、5；[Confirm 只按所选发布版本自身的 schema 校验](../../../decisions/cloud/issue-collaboration/20261020-confirm-validates-against-the-chosen-version.md)；[snapshotId 是保留字段](../../../decisions/cloud/issue-collaboration/20261021-snapshot-id-is-a-reserved-field.md)（D3）。

## Form Descriptors Are Validated Projections

### 风险

非法描述被渲染成半合法表单；读取描述被当作启动许可；版本表单与校验用的描述不一致。

### 前置状态

Workflow 的 Start 节点声明变量；可能有多个已发布版本。

### 触发

`GET /collaboration/forms/{formRef}`、`GET /workflows/{wfid}/snapshots/{snapshotId}/form`。

### 必须成立

- 描述满足 D7 的约束，违反时 `500 invalid_form_descriptor`；提供方失败 503，未知 `formRef` 404。
- 版本表单按该版本的图投影：不带 `issueId` 时就是工作流自己的 schema，作者声明原样投影；带 `issueId` 时叠加平台的版本选择器，作者的 `snapshotId` 声明不渲染。两者都与活稿各自独立；跨租户、跨 Workflow、未知或归档一律 `404 not_found`。
- 读取版本表单不创建运行，启动门禁不通过的 Workflow 仍可读取表单。

### 禁止结果

渲染无效描述；读取描述产生副作用。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 投影与 404 | `TestFormDescriptorProjection`；单元 `TestFormDescriptorProjection*` | Covered |
| 提供方失败与非法描述 | `TestFormDescriptorUnavailableAndInvalid` | Covered |
| 版本表单按版本投影 | `TestWorkflowSnapshotFormProjectsTheChosenVersion` | Covered |
| 版本表单的作用域 | `TestWorkflowSnapshotFormScoping` | Covered |
| 读取不是门禁 | `TestWorkflowSnapshotFormIsNotARunGate` | Covered |

### 决策依据

D7；不变量 9。

## Launch Fields Follow The Catalogue And The Author's Declaration

### 风险

平台字段覆盖作者自己声明的控件；没有可选项的字段被注入导致描述非法；作者撤下的字段仍被接受；旧名 `version`
的声明或提交被当作未知 key，导致作者撤下的字段重新出现或用户选择的版本丢失。

### 前置状态

Issue 有或没有 Project、父 Issue；Workflow 有或没有已发布快照、`launchFields` 声明、同名 Start 变量。

### 触发

带 `issueId` 读取描述；Assist；Confirm。

### 必须成立

- 无 Issue 上下文时不注入；有时按 D8 的顺序和条件注入，作者声明的 key 归作者，保留键 `snapshotId` 除外：它始终由平台注入，活稿里的同名声明不进入表单，Confirm 把它当版本游标读。
- 版本选择器只列出自身冻结图不声明 `snapshotId` 的已发布版本，默认最新的可选版本；有版本被排除时选择器的说明写明原因；没有可选版本时不注入选择器。
- `launchFields` 逐 key 覆盖；撤下的 key 在 Assist 与 Confirm 中都是未知 key；对作者自己的 key 无效。
- 旧名 `version` 的声明按 `snapshotId` 读取；Confirm 的旧 key 按 D8 规则 7 折叠。
- Confirm 把平台字段的值写进交互输入和运行快照。

### 禁止结果

平台改写作者控件；保留键被作者的同名声明抢走；撤下的字段仍然被接受；旧 key 导致字段复现或版本错选。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 注入条件、顺序与默认值 | `TestFormDescriptorPlatformFieldsForAnIssue`；单元 `TestPlatformFieldsLeadTheForm`、`TestVersionFieldOffersPublishedSnapshots`、`TestContextRefsFieldIsAdditiveOnly`、`TestGlobalsComeAfterTheAuthorsOwnFields` | Covered |
| 作者声明优先 | `TestFormDescriptorPlatformFieldsCollideWithAuthorDeclarations`；单元 `TestAuthorDeclaredKeysWin` | Covered |
| 保留键始终属于平台 | 单元 `TestLaunchKeysCollideLikeTheOlderOnes`（作者声明的 `snapshotId` 被丢弃，选择器照常注入）、`TestWorkflowVersionCursorIsThePlatformsForTheReservedKey`；真实 HTTP：`TestAuthorDeclaredSnapshotIDDoesNotTakeTheKeyBack`、`TestWorkflowReservedFieldStaysThePlatformsWhenTheDraftDeclaresIt`（cloud#123 未合入 main，按 `Missing` 记） | Missing |
| 版本选择器排除声明保留键的存量版本并说明原因 | 单元 `TestVersionFieldExplainsWithheldVersions`；真实 HTTP + PostgreSQL：`TestWorkflowReservedFieldWithholdsStockVersionsAndRefusesThem` 断言选项里只有可选的版本、默认值是最新的可选版本、选择器 `description` 说明被排除的版本，以及不带 `issueId` 的版本表单仍按作者声明投影（cloud#123 未合入 main，按 `Missing` 记） | Missing |
| `launchFields` 覆盖与撤下 | `TestFormDescriptorAuthorDeclaredLaunchFields`；单元 `TestLaunchFields*` | Covered |
| 平台字段进入运行 | `TestWorkflowConfirmCarriesPlatformFields` | Covered |
| 旧名兼容 | `TestWorkflowConfirmReadsThePreRenameVersionKey`；单元 `TestFoldLegacyVersionAnswer`、`TestLaunchFieldsDeclarationAcceptsThePreRenameKey` | Covered |

### 决策依据

D8；不变量 5；[snapshotId 是保留字段](../../../decisions/cloud/issue-collaboration/20261021-snapshot-id-is-a-reserved-field.md)（D1、D2）。

## Timeline Orders Comments And Activities By Seq

### 风险

评论与活动按时间戳交错导致顺序不稳定；Workflow 被当作作者；执行日志混入 Timeline。

### 前置状态

Issue 上有评论、Agent/Team 回复和 Workflow 运行。

### 触发

`GET /issues/{iid}/timeline`。

### 必须成立

- 条目按共享 `seq` 严格递增。
- Agent/Team 回复是评论；Workflow 的进度与消息是 `system` 发起的活动，`details` 带 `executorType='workflow'`；
  没有 `authorType='workflow'` 的条目。

### 禁止结果

`seq` 重复或倒序；以 Workflow 为作者的条目。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Agent 链路的交错顺序 | `TestAgentTaskRunsThroughMockExecution`：评论、`run.enqueued`、`run.started`、回复、`run.completed` 的 `seq` 为 1–5 | Covered |
| Workflow 输出是系统活动、`seq` 严格递增 | `TestWorkflowConfirmValidatesAndExecutes` | Covered |
| 执行日志不进入 Timeline | 缺口：Timeline 只由两张表组成，没有直接测试 | Missing |

### 决策依据

D2、D6；不变量 7、8。

## Development Fixtures Stay Out Of Production Wiring

### 风险

开发夹具在生产装配中生效，出现不存在的 Agent/Team 或模拟执行。

### 前置状态

不同的装配入口。

### 触发

`collab.WireDevelopmentFixtures` 与生产 `core.NewStore`。

### 必须成立

- 夹具装配安装全部开发端口，并把真实目录放在夹具之后组合，而不是替换。
- 生产默认不装配夹具。

### 禁止结果

生产环境出现夹具目标。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 夹具装配与组合 | `TestWireDevelopmentFixturesInstallsAllPorts`、`TestFallbackDirectoryComposesPrimaryAndFixtures`、`TestFallbackDirectoryWithoutPrimary` | Covered |
| 生产默认关闭 | 缺口：开发门禁在 `cmd/server` 里，没有直接测试 | Missing |

### 决策依据

D1；不变量 12。
