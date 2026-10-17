# 前序 Revision 与下载授权的 gRPC 契约

依据[已实现的恢复契约决策](../../../decisions/cloud/controller-integration/20261010-revision-restore-contract.md)。
证据来自 `ora-space/cloud`（`integration` 包的 PostgreSQL 测试、`internal/controlgrpc` 线上形状测试、`internal/objectstore`
单元测试）与 `ora-space/desktop`（`apps/ora-controller` 以替身 Cloud 与 Node 运行的测试、`crates/node-protocol` 往返测试）。

## Prior Revisions Are Fixed In Execution Inputs

### 风险

会话与交付看到不同的前序；或交付输入带上 bundle，Node 可以据此申请不属于它的对象。

### 前置状态

运行续接了前序 Revision。

### 触发

Cloud 放出会话与交付工作项，Controller 转交并 `RecordDispatch`。

### 必须成立

- `AgentSessionSpec.prior_revision`（字段 6）带 bundle；`DeliverRevisionSpec.prior_revision`（字段 7）是同一个 Revision，
  不带 bundle。
- Controller 原样转交两者，`RecordDispatch` 的输入必须与工作项相同。

### 禁止结果

字段号改变；交付输入带 bundle；Controller 改写前序。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 线上形状 | cloud `internal/controlgrpc/session_models_restore_test.go::TestSessionWirePreservesModelBindingAndPriorRevision`；desktop `crates/node-protocol/tests/protocol/revision_restore.rs::prior_revision_round_trips_in_session_and_delivery_inputs` | Covered |
| Controller 原样转交 | desktop `apps/ora-controller/src/cloud/agents/mapping.rs::resumed_sessions_carry_the_prior_revision` | Covered |
| 交付输入不带 bundle | cloud `integration/agent_run_resume_test.go::TestResumedRunWithoutNewCommitsReusesThePriorBundle` | Covered |

### 决策依据

契约 D1，不变量 1。

## Download Grants Cover Only The Session's Prior Bundle

### 风险

下载授权可读其他对象，或被持久化、写入日志后泄漏。

### 前置状态

带前序的会话执行已登记。

### 触发

Node 请求下载授权，Controller 调用 `GrantRevisionDownload`。

### 必须成立

- Cloud 只对已登记、尚无结果、输入带 bundle 的会话执行签发恰好一张 GET 授权，键是该 bundle，走公开端点，不带写条件；
  没有前序或已有结果的会话被拒绝。
- Controller 只转交针对该 bundle 键的 GET；Cloud 拒绝签发时转告 Node 不可签发；Cloud 不可用时重试，连接重建后等
  Node 重新请求；授权不进日志。

### 禁止结果

为其他对象签发；Controller 转交其他键或方法；授权出现在日志中。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 签发范围 | cloud `integration/agent_run_resume_test.go::TestFollowUpRunResumesTheIssueLatestRevision`：恰好一张、键为 bundle、GET、无 `if-none-match`，已有结果的会话被拒；`::TestRunWithoutHistoryStartsFresh`：没有前序的会话被拒；`internal/objectstore/presign_test.go::TestPresignGETReadsThroughThePublicEndpointWithoutWriteConditions`。缺口：未断言拒绝的具体错误码，未覆盖运行已取消或 Node 已替换后的拒绝 | Partial |
| Controller 过滤与中继 | desktop `apps/ora-controller/src/cloud/agents/mapping.rs::download_grants_pass_through_only_for_the_prior_bundle`；`apps/ora-controller/tests/workspaces/restores.rs::download_grants_are_relayed_unlogged_and_refusals_answered` | Covered |
| 重试与重连 | desktop `restores.rs::outages_are_retried_and_reconnects_wait_for_the_node_to_ask_again`。缺口：租约失效后的重试没有直接测试 | Partial |

### 决策依据

契约 D2、D5，不变量 2。

## Resumed Sessions Go Only To Restore Capable Nodes

### 风险

带前序的会话派给不会恢复的 Node，Agent 在全新 clone 上运行并交付出丢了前序成果的 Revision。

### 前置状态

会话工作项带 `prior_revision`，目标 Node 未声明 `revision_restore`。

### 触发

Controller 领取工作项。

### 必须成立

- 不登记派发，工作项留在 Cloud；Node 声明能力后才派发。同时带模型绑定的会话还要求 `model_proxy`。

### 禁止结果

把带前序的会话派给未声明能力的 Node。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能力检查 | desktop `apps/ora-controller/tests/workspaces/restores.rs::resumed_sessions_wait_for_a_restore_capable_handshake`、`::model_backed_resumes_require_both_capabilities_and_preserve_both_inputs` | Covered |

### 决策依据

契约 D5，不变量 4。

## Unchanged Results May Stop At The Prior Final Commit

### 风险

续接后的空运行被三方之一拒绝，交付永远无法结算；或任意提交都被当作 `unchanged`。

### 前置状态

交付输入带 `prior_revision`。

### 触发

Node 报告 `RevisionUnchanged`，最终提交等于前序最终提交。

### 必须成立

- Node 协议对 `unchanged` 只校验提交格式；Controller 与 Cloud 接受等于基础提交或前序最终提交的 `unchanged`，拒绝其他提交。

### 禁止结果

任一方拒绝停在前序最终提交的 `unchanged`；接受既不是基础提交也不是前序最终提交的 `unchanged`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 协议编解码 | desktop `crates/node-protocol/tests/protocol/revision.rs::revision_results_round_trip_and_keep_commits_consistent` | Covered |
| Controller 接受 | desktop `apps/ora-controller/src/cloud/deliveries/mapping.rs::resumed_deliveries_accept_an_unchanged_prior_final_commit` | Covered |
| Cloud 登记 | cloud `integration/agent_run_resume_test.go::TestResumedRunWithoutNewCommitsReusesThePriorBundle`：停在前序为 `unchanged`，既不在基础提交也不在前序时被拒 | Covered |

### 决策依据

契约 D4，不变量 3。
