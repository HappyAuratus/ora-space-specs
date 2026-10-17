# Cloud 校验后登记的 bundle Revision

依据[已实现的 Cloud Revision 根决策](../../../decisions/cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)。
证据来自 `ora-space/cloud`：`integration` 包以真实 gRPC、PostgreSQL 与 RustFS 运行（CI 设 `REQUIRE_S3=1`），
`internal/objectstore` 与 `internal/config` 是不访问网络的单元测试。交付执行的 gRPC 接管顺序另见
[Agent 运行执行与上传授权](../controller-integration/agent-run-executions-and-thread.md)，本文件只登记 Revision 自身的承诺。

## Revision Is Registered Only After Cloud Verifies The Stored Objects

### 风险

Cloud 凭 Node 的“已上传”登记 Revision，运行 Workspace 随后被释放，而对象并不存在或内容不符，Agent 的成果永久丢失。

### 前置状态

运行处于 `delivering`，交付执行已登记；对象存储已配置。

### 触发

Node 报告 `delivered` 或 `unchanged`，声明对象键、大小与 SHA-256。

### 必须成立

- Cloud 在事务外以自己的凭据 `HEAD` 每个声明的对象，大小与 SHA-256 都一致才在事务内登记 Revision 并结算交付。
- 对象缺失或大小、摘要不符时接管原 Node 事件，记 Cloud 自己的 `failed{verification_failed}` 结论，不登记 Revision，
  按 IssueRun D5 重试新交付；原 Node 结果保留不被改写。
- 对象存储暂时不可用（网络错误或其他 HTTP 状态）时不接管、不 ACK，可重放。
- 外部校验之后、提交之前，租约与运行状态被重新检查。

### 禁止结果

未校验就登记；校验失败仍登记；Node 在自己的结果中报告 `verification_failed`；暂时故障被当作确定性失败。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 校验通过才登记 | `integration/revision_test.go::TestRevisionS3VerificationAndAtomicTakeover`：真实 RustFS 上 `delivered`/`unchanged` 登记，对象缺失、大小或摘要不符都不登记，hook 或提交失败整体回滚；`integration/agent_run_delivery_test.go::TestRevisionDeliveredIsVerifiedRegisteredAndReleasesTheRun` | Covered |
| 校验失败是 Cloud 的结论 | `integration/agent_run_delivery_test.go::TestRevisionVerificationFailureIsCloudsOwnVerdict`、`::TestNodeMayNotReportCloudsOwnVerificationVerdict` | Covered |
| 暂时故障可重放 | `integration/revision_test.go::TestRevisionNetworkFailureRemainsReplayable`；`integration/agent_run_delivery_test.go::TestDeliveryWithAnUnreachableObjectStoreIsRefusedTransiently` | Covered |
| 外部 I/O 后重新围栏 | `integration/revision_test.go::TestRevisionLeaseAndStateAreRecheckedAfterExternalVerification` | Covered |
| 每个运行至多一个 Revision，重放只接受相同事件 | `integration/agent_run_delivery_test.go::TestRevisionRegistrationIsUniquePerRunAndRollsBackAConflictingAttempt`、`::TestDeliveryReplayMatrix` | Covered |

### 决策依据

Revision D4，不变量 3。

## Declarations Contradicting The Delivery Input Are Refused Before Takeover

### 风险

指向别的 ref、基础提交或对象键的声明被接管，成为该执行不可信的持久结果。

### 前置状态

交付工作项的固定输入含 `revisionRef`、`baseCommit` 与两个对象键。

### 触发

Node 报告的声明与输入不一致，或形状不合法。

### 必须成立

- 在访问对象存储之前拒绝：Node、基础提交、ref、最终提交、bundle 键与形状的问题为 `ABORTED`；会话记录键与形状
  的问题为 `INVALID_ARGUMENT`。
- 拒绝不写收据、结果、验证结论或 Revision，不推进序号，运行保持 `delivering`。

### 禁止结果

为矛盾声明访问对象存储；接管矛盾声明并记 `verification_failed`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接管前拒绝且不留记录 | `integration/agent_run_delivery_test.go::TestRevisionResultContradictingItsInputNeverReachesTheObjectStore`：ref、基础提交、bundle 键不符为 `ABORTED`，会话记录键不符为 `INVALID_ARGUMENT`，均不访问对象存储、不写记录 | Covered |

### 决策依据

Revision D4 第 1 步，不变量 3a。

## Upload Grants Write Only The Fixed Keys And Never Overwrite

### 风险

Node 用授权写到别的键、覆盖已验证的对象，或授权被持久化后泄漏。

### 前置状态

交付工作项的固定输入含两个对象键。

### 触发

Controller 调用 `GrantRevisionUpload`，Node 用授权 PUT；交付失败后重试。

### 必须成立

- 授权只对单个键、`PUT` 与有效期有效；带摘要的请求把 `x-amz-checksum-sha256` 签入，所有授权签入 `If-None-Match: *`。
- 有效授权不能覆盖已存在的对象；错误摘要与过期授权被 S3 拒绝。
- 每次重试使用新工作项的新对象键。
- 签发不改变执行输入，不持久化授权；已有结果或不再交付的执行不签发。

### 禁止结果

授权可写其他键；对象被第二张授权覆盖；授权出现在数据库中。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 签名绑定键与条件头 | `internal/objectstore/presign_test.go::TestPresignPUTBindsTheKeyAndStaysLocal`、`::TestPresignPUTChecksumBindsHeaderAndRejectsInvalidDigest`。缺口：没有真实 S3 上“A 键授权写 B 键被拒”的用例 | Partial |
| 只能首次创建 | `integration/revision_immutability_test.go::TestRevisionStoredObjectsRejectLiveGrantOverwrite`；`internal/simulator/revision_upload_test.go::TestRevisionUploadReconcilesAnAmbiguousCreatedObject`：重复 PUT 的 412 仍交 Cloud 核对 | Covered |
| S3 校验摘要与有效期 | `integration/revision_test.go::TestS3RejectsIncorrectChecksumAndExpiredGrant` | Covered |
| 重试使用新键 | `integration/revision_test.go::TestFailedDeliveryRetryUsesNewWorkKeysAndAtomicHook` | Covered |
| 签发条件与不持久化 | `integration/agent_run_delivery_test.go::TestGrantRevisionUploadSignsOneGrantPerObjectKey`：执行不存在为 `NOT_FOUND`，已有结果或已放弃为 `ABORTED`，签发不改执行输入、不写数据库、不访问对象存储 | Covered |

### 决策依据

Revision D2、D3，不变量 1、2、4。

## Unconfigured Object Storage Skips Delivery Once

### 风险

未配置对象存储的环境里，运行永远卡在 `delivering`，或被当作“已保存”。

### 前置状态

Cloud 启动时没有启用 `object_store`。

### 触发

运行的会话结束，进入 `delivering`。

### 必须成立

- 不放出交付工作项，在同一事务中以 `deliveryState = skipped` 结算并进入释放，每个运行只记录一次，不登记 Revision。
- 健康检查报告 `objectStore: unconfigured`，不暴露配置内容；启用时静态配置非法拒绝启动。

### 禁止结果

运行停在 `delivering`；跳过与业务转移不在同一事务；`skipped` 被重复记录。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 只跳过一次且与业务同事务 | `integration/agent_hook_atomicity_test.go::TestUnconfiguredRevisionStorageSkipsOnceAndRollsBackWithBusiness` | Covered |
| 健康检查与配置校验 | `integration/endpoints_test.go::TestHealthReportsOptionalObjectStoreWithoutDisclosingConfiguration`；`internal/config/object_store_test.go::TestObjectStoreUsesMountedCredentialsAndRejectsInvalidDeployment` | Covered |

### 决策依据

Revision D1，不变量 6。

## Public Run Reads Expose Only Revision Metadata

### 风险

公开 API 泄漏对象键、ref、摘要或可下载的 URL。

### 前置状态

运行已登记 Revision。

### 触发

读取单个运行或运行列表。

### 必须成立

- 运行带 `revision{id, baseCommit, finalCommit, changed, bundleSize, historySize, priorRevisionId, createdAt}`，未登记时为
  `null`；`result` 带 `deliveryState` 与 `revisionId`。
- 对象键、ref、摘要与 URL 都不出现。

### 禁止结果

公开响应中出现对象位置或签名信息。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 投影字段与不泄漏 | `integration/agent_run_delivery_test.go::TestRevisionDeliveredIsVerifiedRegisteredAndReleasesTheRun`：单个与列表读取的投影，且不含对象键、ref 与摘要；前端 `run-delivery.test.tsx` 覆盖保存结果的展示 | Covered |

### 决策依据

Revision D5。
