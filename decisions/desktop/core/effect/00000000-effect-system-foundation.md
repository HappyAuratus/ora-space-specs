---
status: accepted
---

# Effect 系统基础：以 Generic Target 组织声明式收敛

Effect 系统负责把一个作用域内的 Desired State 安全地收敛到外部环境，并持续证明相关
Consumer 已经可以消费该状态。系统以 Generic Target 作为调度与就绪边界，同时把 Consumer、
Target 和可修改的外部 Resource 建模为三个正交概念；任何 Agent、远程环境或其他运行时都只
是 Consumer 的一种实现，不能成为 Effect Core 的领域边界。

## 决策

Effect Core 使用以下基础关系：

```text
EffectScope
├── DesiredState(generation)
│   └── DesiredEffect *
├── EffectTarget * ────────────────→ Consumer
│   ├── TargetProjection
│   ├── TargetStatus
│   ├── ReconcileRequest
│   └── Condition *
└── EffectResource *
    ├── TargetResourceBinding * ───→ EffectTarget
    ├── ResourceProjection
    ├── ManagedItem *
    ├── ResourceStatus
    └── EffectOperation *
```

这三个核心概念的含义不得混用：

- **Consumer** 是消费 Effect 结果、并能对协调和就绪作出确认的外部运行时；
- **Effect Target** 是一个 Consumer 在一个 Effect Scope 内的完整收敛实例；
- **Effect Resource** 是可以被独立定位、观察、加锁、修改和恢复的外部资源。

一个 Target 可以依赖零个、一个或多个 Resource；一个 Resource 也可以被多个 Target 共享。
Target 负责回答“某个 Consumer 是否已准备好消费当前 generation”，Resource 负责回答“某个
外部资源是否已安全物化到当前 generation”。Target 和 Resource 都是一等领域对象，任何一方
都不能代替另一方。

## 适用边界

Effect Consumer 必须满足两个条件：

1. Ora 对它存在需要持续收敛的 Desired State；
2. 它能够通过契约表达协调要求，并对应用结果或就绪状态作出确认。

Agent 插件、Workspace 对应的远程容器环境、需要由 Ora 配置和维持的遥测出口都可以成为
Consumer。只被动接收 Effect 事件、日志或指标的观察者不是 Effect Consumer；它们应订阅
Audit 或 Watch 流，而不创建 Target。这个限制防止“Consumer”退化成任何下游订阅者的泛称。

第一阶段的 Effect Scope 是 Workspace。`EffectScope` 仍作为明确领域概念存在，以承载隔离、
generation 和 ownership 规则；在没有真实需求前，不开放任意字符串类型的 Scope。

## 核心值对象

以下值对象在所有模型和接口中使用强类型，不以裸字符串互换。

### EffectScope

`EffectScope` 是 Desired State、Target、Resource 和 ownership 的隔离根。

```text
EffectScope {
  identity: EffectScopeId,
  lifecycle: Active | Retiring
}
```

Scope 进入 Retiring 后不再接受新的 Desired Effect 或 Target，只允许完成未终结 Operation、
清理 Managed State 和释放 Resource。Scope 只有在这些义务全部结束后才能被移除。

### EffectScopeId

标识 Desired State、generation、Target、Resource 和 ownership 的隔离范围。首个合法形态为：

```text
EffectScopeId = Workspace(WorkspaceId)
```

不同 Scope 的 Target、Resource、Managed State 和 Operation 不得互相引用或修改，即使它们的
Consumer、路径、远程地址或 Source 相同。

### Generation

`Generation` 是一个 Scope 的完整 Desired State 的单调版本。它表示“要评估的整份声明”，
不属于某个 Effect kind、Consumer 或 Resource。

运行状态、观察结果、重试和 Condition 变化不推进 generation。Desired Effect 的增加、删除
或语义变化在同一个原子提交中推进一次 generation。

### EffectKind

`EffectKind` 是 Desired Effect 的稳定、带命名空间的种类身份，例如 Skill 或 MCP。它决定
definition 的验证器与 planner，但不决定 Consumer 类型或 Resource 类型。

新的 Effect kind 通过新的强类型 definition 和 planner 加入；Core 不使用未经验证的任意
JSON 分支业务逻辑。

### ConsumerIdentity

`ConsumerIdentity` 是跨进程重启和声明更新保持稳定的 Consumer 身份，由强类型
`ConsumerKind` 和该种类自己的稳定 identity 组成。显示名称、进程 ID、连接 ID和当前版本都
不能充当 ConsumerIdentity。

### ManagedIdentity

`ManagedIdentity` 是 Ora 对一个外部 Managed Item 的稳定 ownership 身份。它不随 Source
Revision、目标原生名称、路径或内容 fingerprint 变化。

### Fingerprint 与 Digest

- `Digest` 证明一份不可变输入或规范化 projection 的内容身份；
- `Fingerprint` 证明某个外部 Resource 或其中一个 item 的已观察状态。

二者不能代替 ownership，也不能混用。

## Source 与 Desired State 模型

### EffectSource

`EffectSource` 表示一个稳定的 Effect 来源，而不是某次安装或某个外部文件。

```text
EffectSource {
  identity: EffectSourceIdentity,
  kind: EffectKind,
  lifecycle: Active | Retired,
  publication: Unpublished | Published(EffectRevisionId)
}
```

`EffectSourceIdentity` 在版本升级时保持不变。Published Revision 是当前可供新 Desired State
选择的版本；Retired Source 可以继续被历史 Desired、Managed State 或未完成 Operation 引用。

### EffectRevision

`EffectRevision` 是 Source 的不可变、可寻址版本。

```text
EffectRevision {
  identity: EffectRevisionId,
  source: EffectSourceIdentity,
  revision_key: SourceRevisionKey,
  definition: ValidatedEffectDefinition,
  digest: Digest,
  availability: Available | Unavailable(StableReason)
}
```

`SourceRevisionKey` 是来源定义的精确、不可变版本键，可以是 package version、content revision
或其他经过验证的身份，不要求具有可排序的语义版本格式。Revision 发布后，其 definition、
revision key 和 digest 不可修改。`definition` 是 Effect-kind-specific 强类型值，不包含
Consumer-specific 输出格式。Secret 只以 `SecretRef` 出现，不能进入日志、Audit payload、
Digest 明文输入或普通物化描述。

### DesiredState

`DesiredState` 是一个 Scope 的完整期望集合。

```text
DesiredState {
  scope: EffectScopeId,
  generation: Generation,
  effects: Map<DesiredEffectIdentity, DesiredEffect>
}
```

替换 Desired State 使用 expected-generation 进行并发控制。规范化后的集合没有语义变化时是
no-op，不产生新 generation。

### DesiredEffect

`DesiredEffect` 表示 Scope 希望持续存在的一项 Effect 意图。

```text
DesiredEffect {
  identity: DesiredEffectIdentity,
  revision: EffectRevisionId,
  parameters: ValidatedEffectParameters,
  audience: TargetSelector
}
```

`identity` 在 Revision 和参数更新时保持稳定。`parameters` 只包含该 Effect kind 的规范化输入
和安全引用，不包含 Consumer 或 Resource 的输出格式。

### TargetSelector

`TargetSelector` 定义一个 Desired Effect 应投影到哪些 Target。

```text
TargetSelector {
  required_capabilities: CapabilityRequirement,
  include: TargetInclusion,
  exclude: Set<ConsumerIdentity>
}

TargetInclusion = AllEligible | Only(Set<ConsumerIdentity>)
```

“Eligible”由 Consumer Revision 的 Capability Set 确定。Selector 只决定投影范围，不表示目标
已经接受、应用或准备好消费该 Effect。

## Consumer 与 Target 模型

### Consumer

`Consumer` 表示一种具有稳定身份的外部运行时。

```text
Consumer {
  identity: ConsumerIdentity,
  adapter: ConsumerAdapterIdentity,
  current_revision: ConsumerRevisionId,
  lifecycle: Declared | Retiring
}
```

Consumer 的暂时断连或进程退出不是 Retiring。暂时可用性属于观察事实，通过 Condition 和
reconcile 结果表达；Retiring 表示 Ora 已确认这个 Consumer 不再产生新的 Target，并需要安全
清理现有 Target。

### ConsumerRevision

`ConsumerRevision` 是 Consumer 声明的不可变能力快照。

```text
ConsumerRevision {
  identity: ConsumerRevisionId,
  consumer: ConsumerIdentity,
  capabilities: CapabilitySet,
  declaration_digest: Digest
}
```

Capability Set 至少表达：

- 接受哪些 Effect kind 及其协议版本；
- 支持哪些 materialization contract；
- 能提供哪些协调和 readiness contract；
- 哪些 Target/Resource declaration 规则属于本 Revision。

能力变化创建新的 ConsumerRevision，并唤醒所有引用旧 Revision 的活动 Target。Core 不为
Agent、容器或遥测定义专用 capability 字段；具体 capability 由版本化强类型契约解释。

### EffectTarget

`EffectTarget` 是 Generic Target，是调度和 readiness 的最小边界。

```text
EffectTarget {
  identity: EffectTargetId,
  scope: EffectScopeId,
  consumer: ConsumerIdentity,
  consumer_revision: ConsumerRevisionId,
  lifecycle: Active | Retiring
}
```

一个 `(scope, consumer)` 最多对应一个活动 Target。Target identity 不包含 Consumer 类型专用
信息；Agent Session、容器实例、遥测连接等运行时细节由 Consumer Adapter 管理。

Target 可以在当前 generation 没有任何适用 Desired Effect，也可以没有 Resource binding。
它仍是合法 Target，并可通过一次空 projection 确认 Current。任何持久外部 mutation 则必须
归属一个 Resource。

### TargetDeclaration

`TargetDeclaration` 是 Consumer 针对一个 Scope 给出的完整、可替换声明。

```text
TargetDeclaration {
  target: EffectTargetId,
  consumer_revision: ConsumerRevisionId,
  bindings: Set<TargetResourceBinding>,
  digest: Digest
}
```

声明是 snapshot，不是增量事件。相同 Revision 和 Scope 产生的规范化声明必须确定性一致。
声明变化只改变 Target 与 Resource 的当前拓扑，不直接证明任何外部状态已经改变。

### TargetResourceBinding

`TargetResourceBinding` 表示一个 Target 如何消费一个 Resource。

```text
TargetResourceBinding {
  target: EffectTargetId,
  resource: EffectResourceId,
  accepts: CapabilityRequirement,
  coordination: CoordinationRequirement
}
```

`CoordinationRequirement` 使用通用语义：

```text
CoordinationRequirement =
  | Uninterrupted
  | QuiesceBeforeMutation(CoordinationContract)
```

Agent 的等待 Session 空闲和重启、远程环境的 drain/redeploy、遥测出口的 flush/reconnect 都由
各自 Consumer Adapter 实现同一个协调契约。Core 只持久化协调阶段和 receipt，不包含这些
领域专用状态。

### TargetProjection

`TargetProjection` 是 planner 根据某个 generation 的 Desired State、Target Selector 和精确
Consumer Revision 得到的完整目标视图。

```text
TargetProjection {
  target: EffectTargetId,
  generation: Generation,
  consumer_revision: ConsumerRevisionId,
  desired_effects: Set<DesiredEffectIdentity>,
  resource_requirements: Set<ResourceRequirement>,
  digest: Digest
}
```

Projection 必须是确定性的完整 snapshot。缺失 capability、unsupported effect 或无效声明不
能被静默过滤；planner 必须返回结构化 Condition。Projection 不等于 Operation plan，也不
授权外部 mutation。

### ResourceRequirement

`ResourceRequirement` 是一个 TargetProjection 对 Resource 提出的完整物化需求。

```text
ResourceRequirement {
  target: EffectTargetId,
  resource: EffectResourceId,
  desired_effects: Set<DesiredEffectIdentity>,
  materialization_contract: MaterializationContract,
  digest: Digest
}
```

Requirement 只描述该 Target 的贡献，不能直接作为 Resource 的最终 Desired State。共享 Resource
必须先合并所有活动 Requirement，再形成唯一 ResourceProjection。

## Resource 与物化状态模型

### EffectResource

`EffectResource` 是外部 mutation、ownership、观察、并发控制和恢复的边界。

```text
EffectResource {
  identity: EffectResourceId,
  scope: EffectScopeId,
  resource_key: ResourceKey,
  adapter: ResourceAdapterIdentity,
  descriptor: VersionedResourceDescriptor,
  format: MaterializationFormat,
  lifecycle: Active | Retiring
}
```

`ResourceKey` 是 adapter 规范化后的物理身份。同一 Scope 内相同 ResourceKey 只能对应一个活动
Resource，从而保证只有一个 writer 和一个 ownership ledger。文件目录、单个配置文档、远程
容器配置集、远程订阅或其他可独立校验的对象都可以是 Resource。

“Surface”是 filesystem 等 adapter 可以使用的 Resource 类型，而不是 Effect Core 的调度
根。Resource descriptor 必须由对应 adapter 按版本解析和验证；Core 不读取其中的专用字段。

### ResourceProjection

`ResourceProjection` 是一个 Resource 在某个 generation 应达到的完整状态。

```text
ResourceProjection {
  resource: EffectResourceId,
  generation: Generation,
  contributors: Set<EffectTargetId>,
  items: Map<ManagedIdentity, ResolvedMaterialization>,
  digest: Digest
}
```

它由所有活动 TargetProjection 对该 Resource 的 requirement 合并产生，而不是由任一 Target
单独拥有。合并必须检测 native identity、format、ownership 和 materialization contract
冲突。这个规则保证共享 Resource 上的一个 Target 不能删除另一个 Target 仍然需要的内容。

### ResolvedMaterialization

`ResolvedMaterialization` 是一个 Desired Effect 在精确 Resource 和 Consumer capability 下的规范化
物化结果。

```text
ResolvedMaterialization {
  managed_identity: ManagedIdentity,
  desired_effect: DesiredEffectIdentity,
  revision: EffectRevisionId,
  native_identity: NativeResourceIdentity,
  contract: MaterializationContract,
  input_digest: Digest,
  input: VersionedMaterializationInput
}
```

它不包含当前外部状态，也不授权 mutation。`input` 必须是对应 Resource Adapter 验证过的
版本化类型；未知版本不能被猜测执行。相同 Scope、generation、Consumer Revision 和
Resource descriptor 必须产生相同结果。

### ManagedItem

`ManagedItem` 是 Ora 对 Resource 内一个外部 item 的 ownership ledger。

```text
ManagedItem {
  identity: ManagedIdentity,
  resource: EffectResourceId,
  desired_effect: DesiredEffectIdentity,
  applied_revision: EffectRevisionId,
  native_identity: NativeResourceIdentity,
  fingerprint: Fingerprint,
  applied_generation: Generation
}
```

只有 ManagedItem 才授权 Ora 更新或删除对应外部 item。路径、native name、Source identity、内容
相同或 Target 声明都不能单独证明 ownership。共享 Resource 的 ledger 归 Resource，不复制到
每个 Target。

### ResourceObservation

`ResourceObservation` 是 Resource Adapter 对某一时刻外部事实的规范化快照。

```text
ResourceObservation {
  resource: EffectResourceId,
  items: Map<NativeResourceIdentity, ObservedItem>,
  fingerprint: Fingerprint
}

ObservedItem {
  native_identity: NativeResourceIdentity,
  fingerprint: Fingerprint,
  ownership_evidence: NoOwnershipEvidence | Claims(ManagedIdentity)
}
```

Observation 是事实，不是 ownership。没有与 Managed ledger 精确匹配的 ObservedItem 被分类为
`PreservedItem`；它对 Ora 不可变。若 PreservedItem 与 projection 冲突，reconcile 必须阻塞，
不能接管、覆盖或删除。

### PreservedItem

`PreservedItem` 是没有精确 Managed ledger 匹配的外部状态。

```text
PreservedItem {
  resource: EffectResourceId,
  native_identity: NativeResourceIdentity,
  fingerprint: Fingerprint
}
```

PreservedItem 是从 Observation 和 Managed ledger 比较得到的领域分类，不建立 Ora ownership。
即使其名称、内容或 Source 与某个 Desired Effect 相同，也必须保持不变。

## 收敛状态模型

### TargetStatus

`TargetStatus` 表示一个 Target 的联合收敛水位。

```text
TargetStatus {
  target: EffectTargetId,
  progress: TargetProgress,
  phase: TargetPhase,
  version: StatusVersion
}

TargetProgress {
  desired: Generation,
  observed: Generation,
  applied: Generation,
  ready: Generation
}

TargetPhase =
  | Pending
  | Reconciling(ReconcileStage)
  | Current
  | CurrentWithIssues
  | Retiring
  | RecoveryRequired(EffectOperationId)

ReconcileStage = Planning | Coordinating | Applying | Verifying | Activating
```

generation 必须满足：

```text
ready <= applied <= observed <= desired
```

- `observed`：planner 已使用该 generation 的完整 Desired State 和 Consumer Revision 完成评估；
- `applied`：TargetProjection 依赖的全部 ResourceProjection 已验证完成；
- `ready`：Consumer Adapter 已确认 Consumer 可以安全消费该 generation。

Target phase 只表达通用收敛位置。等待 Agent idle、容器 drain 或遥测 flush 等具体原因通过
Condition 和 Coordination Receipt 表达，不扩展 TargetPhase。

### ResourceStatus

`ResourceStatus` 表示一个 Resource 的独立物化水位。

```text
ResourceStatus {
  resource: EffectResourceId,
  desired: Generation,
  observed: Generation,
  applied: Generation,
  phase: Pending | Reconciling | Current | Retiring |
         RecoveryRequired(EffectOperationId),
  version: StatusVersion
}
```

Resource 没有 `ready`：Resource 只负责物化，只有 Consumer/Target 才能证明可以消费。Target 的
`applied` 只有在它依赖的全部 Resource 对相应 projection 完成后才能推进。

### EffectCondition

`EffectCondition` 表示当前尚未解除、可供控制流和用户界面解释的结构化事实。

```text
EffectCondition {
  identity: ConditionId,
  owner: Target(EffectTargetId) | Resource(EffectResourceId),
  subject: ConditionSubject,
  code: StableConditionCode,
  impact: Blocking | NonBlocking,
  retry: OnChange | Backoff(RetryPolicy) | Manual,
  generation: Unscoped | At(Generation),
  safe_details: SafeConditionDetails
}
```

`ConditionSubject` 是强类型 tagged union，至少允许 Consumer、Target、Desired Effect、Resource、
Managed Item、Operation 和 Artifact。`safe_details` 只能包含公开稳定参数，不得包含 Secret、
配置原文、敏感路径或任意 error chain。

Blocking Condition 阻止相关水位前进；NonBlocking Condition 可以使 Target 进入
`CurrentWithIssues`。message 只用于解释，控制流只依赖 code、impact 和 retry。

### ReconcileRequest

`ReconcileRequest` 是“某个 Target 至少需要再评估一次”的 durable、level-triggered 事实。

```text
ReconcileRequest {
  target: EffectTargetId,
  requested_generation: Generation,
  state: ReconcileRequestState,
  wake_reasons: Set<WakeReason>
}

ReconcileRequestState =
  | Pending
  | Claimed(ReconcileClaim)
  | RetryScheduled { attempt: RetryAttempt, not_before: LocalTime }
  | Blocked { conditions: NonEmptySet<ConditionId>, resume: ResumeTrigger }

ReconcileClaim {
  token: FencingToken,
  worker: WorkerIdentity,
  lease_until: LocalTime
}
```

每个 Target 最多一个 request。连续 wakeup 合并为最大的 requested generation；wake reason 只
用于诊断，不能成为正确性来源。worker claim 后必须重新读取当前事实，不能执行 wakeup 中的
旧 payload。

### ResourceClaim

`ResourceClaim` 是修改 Resource 的独占、带 fencing token 的租约。

```text
ResourceClaim {
  resource: EffectResourceId,
  target_claim: FencingToken,
  resource_fence: FencingToken,
  worker: WorkerIdentity,
  lease_until: LocalTime
}
```

Target claim 只授权推进该 Target，不授权修改共享 Resource。任何 Resource mutation 都必须先
获得对应 ResourceClaim，并在持久化 Operation 前重新验证 ResourceProjection、Status、
Managed ledger 和 Observation。一次需要多个 Resource 的 mutation 按 Resource identity 的
稳定顺序取得 claim，避免死锁。

### ReconcileAttempt

`ReconcileAttempt` 是一次可恢复的 Target 收敛编排，封装精确输入和跨 Resource 的执行顺序。

```text
ReconcileAttempt {
  identity: ReconcileAttemptId,
  target: EffectTargetId,
  generation: Generation,
  consumer_revision: ConsumerRevisionId,
  target_projection: ProjectionDigest,
  resource_projections: Set<ProjectionDigest>,
  coordination: CoordinationPlan,
  operations: OrderedSet<EffectOperationId>,
  phase: Prepared | Coordinated | Applied | Verified |
         Activated | Finalized | RecoveryRequired
}
```

`Prepared` 后输入 revision、projection digest、协调参与者和 operation 顺序不可修改。更高
generation 到来时保留新 request；当前 Attempt 完成或恢复后重新规划，不能篡改正在执行的
Attempt。

### CoordinationPlan 与 CoordinationReceipt

共享 Resource mutation 前，所有声明需要协调且会受该 mutation 影响的 Target 都是参与者。

```text
CoordinationPlan {
  resources: NonEmptySet<EffectResourceId>,
  participants: Map<EffectTargetId, CoordinationRequirement>
}

CoordinationReceipt {
  target: EffectTargetId,
  contract: CoordinationContract,
  state: SafeToMutate | Reactivated,
  proof: AdapterReceipt
}
```

只有全部参与者都取得 `SafeToMutate` receipt 后才能修改共享 Resource。应用完成后，所有已被
协调的参与者都必须进入 Reactivated 或保留可恢复的失败事实。一个参与者的 receipt 只证明
本次协调，不自动推进它的整体 Target `ready`；该 Target 仍需验证自己的完整 projection。

### EffectOperation

`EffectOperation` 是一次 Resource mutation 的不可变 journal，也是崩溃恢复的授权依据。

```text
EffectOperation {
  identity: EffectOperationId,
  attempt: ReconcileAttemptId,
  resource: EffectResourceId,
  generation: Generation,
  mutation: Create | Update | Replace | Delete,
  expected: ExactPreviousState,
  planned: ExactPlannedState,
  payload: VersionedAdapterPlan,
  phase: Prepared | Applied | Finalized | RecoveryRequired
}
```

Operation 必须在任何外部副作用前进入 Prepared。Adapter 重试时只能接受三种可证明状态：当前
等于 expected、当前等于 planned，或进入 RecoveryRequired；不能依据路径、名称或相似内容
猜测。

Finalize 原子提交 Managed ledger transition、Operation phase、相关 Status/Condition 和必要
Audit event。Finalized 表示业务状态已提交，不表示临时 Artifact 已清理。

### OperationArtifact

`OperationArtifact` 是 Operation 为安全 apply、恢复或补偿拥有的临时外部资源。

```text
OperationArtifact {
  identity: ArtifactId,
  operation: EffectOperationId,
  role: ArtifactRole,
  locator: VersionedResourceLocator,
  expected_fingerprint: Fingerprint,
  state: Reserved | Retained | PendingCleanup | CleanupFailed
}
```

Artifact 不是 Managed Item。只有精确 Operation、locator 和 fingerprint 同时匹配时才能使用或
删除；不得通过扫描名称前缀清理没有 durable ownership 的外部资源。

### ReadinessReceipt

`ReadinessReceipt` 是 Consumer Adapter 对 Target readiness 的可验证确认。

```text
ReadinessReceipt {
  target: EffectTargetId,
  generation: Generation,
  consumer_revision: ConsumerRevisionId,
  projection: ProjectionDigest,
  proof: AdapterReceipt
}
```

receipt 必须精确匹配当前 Target、generation、Consumer Revision 和 projection。缺失、重复、
旧 generation 或无法验证的 receipt 都不能推进 `ready`。

### EffectAuditEvent

`EffectAuditEvent` 是追加式、可裁剪的业务历史。

```text
EffectAuditEvent {
  identity: AuditEventId,
  scope: Global | Scoped(EffectScopeId),
  subject: AuditSubject,
  kind: AuditEventKind,
  generation: NotApplicable | At(Generation),
  initiator: User | System | Consumer(ConsumerIdentity),
  payload: VersionedSafeAuditPayload
}
```

Audit 不参与 ownership、调度、恢复或当前状态推导，也不是 event-sourcing log。删除历史 Audit
不得改变系统行为。

## Planner 与 Adapter 接缝

Effect Core 对外提供少量深 interface，把 Consumer 与 Resource 的变化隔离在 adapter 接缝：

```text
EffectKindPlanner
└── project(desired, target, consumer_revision) -> TargetProjection | Conditions

ResourcePlanner
└── merge(resource, target_requirements, managed, observed)
    -> ResourceProjection + MutationPlan | Conditions

ConsumerAdapter
├── coordinate(target, coordination_plan) -> CoordinationReceipt
├── reactivate(target, coordination_plan) -> CoordinationReceipt
└── verify_ready(target, projection) -> ReadinessReceipt

ResourceAdapter
├── observe(resource) -> ResourceObservation
├── apply(operation) -> ApplyReceipt
├── verify(operation) -> VerificationReceipt
└── cleanup(artifact) -> CleanupReceipt

EffectReconciler
└── reconcile(claim) -> ReconcileOutcome
```

Planner 必须是确定性的纯逻辑；Adapter 承担外部协议和副作用；Reconciler 负责 durable 状态机、
claim、顺序和原子提交。调用方不直接拼接 Target、Resource、Status、Condition、Operation 或
ledger 的局部更新。

Consumer-specific 模块可以有自己的内部模型，例如 Agent Session、容器 Deployment 或遥测
Checkpoint，但这些类型不得进入 Effect Core interface。Resource-specific descriptor 和
operation payload 必须版本化，并只由对应 Adapter 解释。

## 收敛流程

一次 Target reconcile 遵循以下顺序：

```text
claim Target request
  → 读取当前 Desired State、Consumer Revision 与 Target declaration
  → 计算完整 TargetProjection
  → 合并所有相关 Target requirement，计算 ResourceProjection
  → 观察 Resource，并与 Managed ledger 一起规划
  → 获取所需 ResourceClaim 并重新验证全部输入
  → 持久化 immutable ReconcileAttempt 与 Prepared Operations
  → 协调所有受影响 Consumer Target
  → 应用并验证 Resource Operations
  → 原子更新 Managed ledger 与 applied watermarks
  → 重新激活参与者并验证当前 Target readiness
  → 推进 ready generation
  → 完成或保留更高 generation 的 request
```

任何外部调用期间都不得长期持有数据库事务。claim/lease 解决 worker ownership，Operation
journal 解决外部副作用恢复；两者不能互相替代。

## 必须保持的不变量

1. 一个 Scope 的状态、ownership 和 mutation 不得越过 Scope 隔离边界。
2. 一个 `(Scope, Consumer)` 最多有一个活动 EffectTarget。
3. Consumer、Target 和 Resource 是不同身份；任一 identity 都不能由另外两者推导代替。
4. Target 与 Resource 是多对多关系，Target readiness 必须覆盖其完整 projection。
5. ResourceProjection 必须合并全部活动贡献者，任一 Target 不能删除其他 Target 仍需要的项。
6. Target claim 不授权 Resource mutation；共享 Resource 在任一时刻最多有一个有效 writer。
7. 修改共享 Resource 前，必须满足所有受影响 binding 的协调要求。
8. `ready <= applied <= observed <= desired`，任何水位都只能由对应证据推进。
9. Desired、Managed、Observed 和 Preserved State 含义独立，Observed State 不能证明 ownership。
10. 只有 ManagedItem 或 durable Operation/Artifact authority 才允许更新或删除外部资源。
11. Prepared Attempt 和 Operation 不可因更高 generation 到来而改变；新工作通过 request 合并。
12. Capability Revision 变化必须重新投影全部相关 Target，不能沿用过期 capability 的 readiness。
13. Blocking Condition 阻止对应水位推进；NonBlocking Condition 不得被伪装成成功无问题。
14. Target、Resource 和 Operation 的状态转换必须通过领域方法原子执行，不能由调用方逐字段写入。
15. 所有外部 mutation 都必须可验证、可重试或明确进入 RecoveryRequired，不能猜测完成状态。
16. Consumer-specific 和 Resource-specific 数据只能存在于版本化契约及 Adapter 内，不能污染
    Effect Core 的通用状态机。

## 结果

Effect Core 的扩展单位是新的 EffectKindPlanner、ConsumerAdapter 或 ResourceAdapter，而不是
为每种 Consumer 复制 Target、Status、Request、Condition 和 Operation 模型。Agent 插件、
远程环境及需要主动收敛的遥测运行时共享同一套 generation、调度、ownership、恢复与可观测性
语义，同时保留各自专用协调协议。

该设计会同时维护 Target 级 readiness 和 Resource 级 materialization 状态，这是有意的：
前者无法证明共享外部资源的安全写入，后者也无法证明依赖多个 Resource 的 Consumer 已整体
就绪。两层状态通过 Projection、Binding 和 Receipt 连接，而不互相冒充。
