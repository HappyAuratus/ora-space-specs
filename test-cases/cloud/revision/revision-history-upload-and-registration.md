# Cloud 的会话记录上传授权与 Revision 登记

依据[已实现的 Cloud Revision 根决策](../../../decisions/cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)。
根决策中 bundle 的对象键、校验、上传授权与下载授权已被 [Cloud Revision Git 交付决策](../../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md)
取代：Revision 经交付 ref 核验后登记，对应用例见 [Revision 经 Git Server 交付与续作](git-native-delivery-refs-and-resume.md)。
本文件继续登记仍然成立的义务：会话记录（JSONL）的上传授权、每个运行至多一个 Revision，以及对象存储的健康检查与配置校验。
证据来自 `ora-space/cloud`：`integration` 包以真实 gRPC、PostgreSQL 与 RustFS 运行（CI 设 `REQUIRE_S3=1`），
`internal/objectstore` 与 `internal/config` 是不访问网络的单元测试。交付执行的 gRPC 接管顺序另见
[Agent 运行执行与上传授权](../controller-integration/agent-run-executions-and-thread.md)。

## A Run Registers At Most One Revision And Replays Only The Same Event

### 风险

同一运行登记多个 Revision，或重放被当作新的交付结果接受，成果的归属不唯一。

### 前置状态

运行处于 `delivering`，交付执行已登记。

### 触发

Node 重放同一事件；另一次尝试带着不同的内容报告结果。

### 必须成立

- 每个运行至多一个 Revision；冲突的尝试整体回滚。
- 重放只接受与原事件相同的内容，其余被拒绝。

### 禁止结果

同一运行出现第二个 Revision；重放改写已登记的结果。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 每个运行至多一个 Revision，重放只接受相同事件 | `integration/agent_run_delivery_test.go::TestRevisionRegistrationIsUniquePerRunAndRollsBackAConflictingAttempt`、`::TestDeliveryReplayMatrix` | Covered |

### 决策依据

Revision D4；Git 交付决策 D3。

## Upload Grants Write Only The Session History Key And Never Overwrite

### 风险

Node 用授权写到别的键、覆盖已验证的会话记录，或授权被持久化后泄漏。

### 前置状态

交付工作项的固定输入含会话记录的对象键。

### 触发

Controller 调用 `GrantRevisionUpload`，Node 用授权 PUT；交付失败后重试。

### 必须成立

- 授权只对单个键、`PUT` 与有效期有效；带摘要的请求把 `x-amz-checksum-sha256` 签入，所有授权签入 `If-None-Match: *`。
- 有效授权不能覆盖已存在的对象；错误摘要与过期授权被 S3 拒绝。
- 每次重试使用新工作项的新会话记录键。
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
| 签发条件与不持久化 | `integration/agent_run_delivery_test.go::TestGrantRevisionUploadSignsOnlyTheHistoryKey`：只签发会话记录键，其他键的校验和被拒绝；执行不存在为 `NOT_FOUND`，已有结果或已放弃为 `ABORTED`，签发不改执行输入、不写数据库、不访问对象存储 | Covered |

### 决策依据

Revision D2、D3，不变量 1、2、4；Git 交付决策 D6。

## Object Storage Health And Configuration Are Reported Without Disclosure

### 风险

未配置对象存储的环境里健康检查泄露配置内容，或启用时静态配置非法却仍然启动。

### 前置状态

Cloud 启动时没有启用 `object_store`，或启用时配置非法。

### 触发

读取健康检查；以非法配置启动。

### 必须成立

- 健康检查报告 `objectStore: unconfigured`，不暴露配置内容；启用时静态配置非法拒绝启动。
- 未配置时交付以 `skipped` 结算的义务见 [Revision 经 Git Server 交付与续作](git-native-delivery-refs-and-resume.md)。

### 禁止结果

健康检查暴露配置内容；非法配置被接受。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 健康检查与配置校验 | `integration/endpoints_test.go::TestHealthReportsOptionalObjectStoreWithoutDisclosingConfiguration`；`internal/config/object_store_test.go::TestObjectStoreUsesMountedCredentialsAndRejectsInvalidDeployment` | Covered |

### 决策依据

Revision D1，不变量 6；Git 交付决策 D2。
