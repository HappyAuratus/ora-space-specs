---
status: implemented
---

# 会话与交付执行输入增加前序 Revision，新增 `GrantRevisionDownload` 签发只读下载授权

为支持 [IssueRun 续接决策](../issue-run/20261010-resume-run-from-latest-issue-revision.md)，Internal Control gRPC
只增不改地扩展三处：`AgentSessionSpec` 与 `DeliverRevisionSpec` 增加 `prior_revision`；`RevisionUnchanged`
的适用条件放宽到“最终提交等于前序最终提交”；`AgentRunService` 新增 `GrantRevisionDownload`，为带前序的会话
执行签发前序 bundle 的预签名 GET。Controller 在 Node 与 Cloud 之间中继下载授权，方式与上传授权相同，并只把
带前序的会话派给声明了恢复能力的 Node。

本 ADR 处于 `implemented`，继承 [Agent 运行执行与上传授权决策](20260928-agent-run-executions-thread-and-upload-grants.md)，
描述 Cloud 的 proto、`internal/controlgrpc` 映射与授权签发，以及 desktop Controller 的当前实现。兼容性边界：
`v1` 已发布的消息、字段与方法不变，`buf breaking` 检查通过；不带 `prior_revision` 的执行与之前完全相同。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| D1：`AgentSession{agent_plugin_id, agent_plugin_version, checkout_execution_id, git_identity, initial_turn}` | 修改：增加 `prior_revision`（字段 6） |
| D1：`DeliverRevision{session_execution_id, …, history_key}` | 修改：增加 `prior_revision`（字段 7），不带 bundle |
| D1：`RevisionUnchanged` 表示最终提交等于基础提交 | 修改：或者等于输入中前序的最终提交 |
| D4：授权只用于上传，不持久化、不进日志 | 继承；下载授权遵守同样的规则（D2） |
| 根 D6：错误只有七个枚举值 | 继承，不新增 |
| 会话结束原因的封闭集合 | 继承，不新增；恢复失败用 `AGENT_FAILED` 与 `detail`（D3） |

## 决策

### D1：执行输入中的前序 Revision

- 新消息 `PriorRevision{revision_id = 1, final_commit = 2, bundle = 3}`，`bundle` 是 `StoredObject`（对象键、大小、
  SHA-256）。
- `AgentSessionSpec.prior_revision = 6`，带 `bundle`：它是 Cloud 登记时校验过的对象，可能属于另一个 Revision
  （续接决策 D5 的沿用），Node 按它校验下载结果。字段 7 由个人模型决策另行增加，与本文无关。
- `DeliverRevisionSpec.prior_revision = 7`，与该运行会话输入中的是同一个 Revision，不带 `bundle`：Node 只需要
  它的最终提交来判定 D4 的 `unchanged`。
- 两者都在 Cloud 放出工作项时固定；`RecordDispatch` 要求 Controller 转交的整份输入与工作项中的相同，否则
  `dispatch_conflict`，规则同前序决策 D1。

为什么把 bundle 的对象信息放进会话输入、而不放下载链接：对象键与摘要不会过期，可以随工作项长期等待领取；
链接会过期，所以与上传一样另行申请（前序决策“上传授权在工作项中一起返回”的否决理由同样适用）。

### D2：`GrantRevisionDownload`

- `AgentRunService.GrantRevisionDownload(epoch = 1, execution_id = 2)` 返回 `grants = 1`，恰好一张
  `DownloadGrant{object_key = 1, url = 2, method = 3, headers = 4, expires_at = 5}`：`method` 恒为 `GET`；
  `headers` 须原样携带，目前为空；`expires_at` 是实际过期时间。
- 只对已登记（经过 `RecordDispatch`）、尚无结果、未被强停、输入带 `prior_revision.bundle` 的会话执行签发，
  对象键只能是该 bundle 的键。执行所在 Workspace、运行与 Node 的有效性检查与上传授权相同。错误映射：执行
  不存在或不是会话执行为 `NOT_FOUND`；其他拒绝为 `ABORTED`/`CONFLICT`（`dispatch_conflict`，对象存储未配置时
  为 `object_store_unconfigured`）；租约失效为 `FAILED_PRECONDITION`，Controller 会重试，所以拒绝签发不使用它。
- 不是写提交，不携带 `submission_id`，不在 Cloud 留下记录；每次调用签发新授权。有效期沿用对象存储配置的
  `upload_grant_ttl`。
- 链接使用对象存储的公开地址（沙盒可达），不带条件头；Node 自己校验大小与摘要。
- 授权是短期的持有者凭据：Cloud、Controller、Node 都只在内存中持有，不写日志、不进错误信息。

为什么不复用 `GrantRevisionUpload`：上传授权只对交付执行签发，键来自交付输入，且带不可覆盖的写条件；把读
混进去会让“谁能拿到什么权限”不再由执行种类一眼决定。

### D3：恢复失败的会话结果

恢复失败时 Node 报告 `AgentSessionEnded{reason = AGENT_FAILED, detail}`，`detail` 为
`prior_revision_unavailable` 或 `prior_revision_base_unavailable`。Cloud 在会话结束结算中按 IssueRun 续接决策
D3 处理。不新增结束原因：运行状态的推导（`agent_failed` → `failed`）不变，旧 Controller 与 Cloud 都能接受
这个结果。

### D4：`RevisionUnchanged` 的适用条件

交付输入带 `prior_revision` 时，`RevisionUnchanged.final_commit` 可以等于 `prior_revision.final_commit`
（且不等于基础提交），表示续接后没有新提交；其余字段不变，仍只带会话记录。Node 协议的编解码对 `unchanged`
只校验提交格式，不再自行要求“最终提交等于基础提交”；合法性由持有输入的一方判断：Node 冻结交付计划时、
Controller 接受结果时与 Cloud 登记时，其他情况仍要求最终提交等于基础提交。

### D5：Controller 的中继与派发

- Node 新能力 `revision_restore`。会话输入带 `prior_revision` 时，Controller 只在目标 Node 声明了该能力时登记
  派发（同时带模型绑定的会话还要求 `model_proxy`）；否则与能力缺失的其他工作项一样留在 Cloud。
- Node 请求下载授权的消息不占事件序号，只在握手声明了 `revision_restore` 的连接上接受。Controller 先按本地
  派发记录预检：执行不属于该 Node、不是会话、已有结果或没有前序时直接转告不可签发，不调用 Cloud。
- 预检通过后调用 `GrantRevisionDownload`，只接受针对该会话 bundle 键的 GET，把结果原样转交给发出请求的连接。
  Cloud 以 `NOT_FOUND` 或 `ABORTED` 拒绝签发时转告 Node 不可签发，Node 据此失败（`prior_revision_unavailable`）；
  Cloud 不可用或租约失效时每 500 毫秒重试，同一执行的重复请求合并。Cloud 返回的键或方法不符时按契约错误
  丢弃，不转告拒绝，由 Node 重新请求。
- Controller 不缓存、不持久化下载授权，连接重建后由 Node 重新请求。

## 不变量

1. `v1` 已发布的消息、字段与方法不变；不带前序的执行与之前相同。
2. 下载授权只针对一个会话执行输入中固定的那个 bundle 对象，只读，不持久化，不进日志。
3. `RevisionUnchanged` 的最终提交等于基础提交，或等于交付输入中前序的最终提交。
4. 带前序的会话只派给声明了 `revision_restore` 的 Node。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 下载链接直接写进会话输入 | 少一次调用；链接会过期，而工作项可能长期等待领取 |
| Node 直接持有对象存储凭据读取 | 无需中继；违背 Revision D1“凭据只在 Cloud” |
| 新增结束原因 `RESTORE_FAILED` | 语义更直白；需要改状态推导，旧组件会拒绝未知枚举值 |
| 交付输入不带前序，由 Cloud 在登记时判定“空运行” | Node 不用改判定；Node 会白白上传一个与前序相同的 bundle |

## 本决策未解决的问题

- 下载授权的单独有效期配置（目前与上传共用）。
- Controller 遇到能力不足（`revision_restore`、`model_proxy`）等无法登记的工作项时停止本批领取，而 Cloud 总是
  返回最早的未登记工作项，造成 ClaimWork 队头阻塞；随 ClaimWork 的整体修复处理。

## 验证

核心用例见 [前序 Revision 与下载授权的 gRPC 契约](../../../test-cases/cloud/controller-integration/revision-restore-contract.md)。
Cloud：`integration/agent_run_resume_test.go` 以 PostgreSQL 覆盖前序在放出工作项时固定、交付输入不带 bundle、
只对带前序的未结束会话签发且键只能是固定的 bundle、放宽后的 `RevisionUnchanged` 校验；
`internal/controlgrpc` 的线上形状测试断言字段 6、7；`internal/objectstore` 测试断言 GET 走公开端点且不带写条件；
CI 运行 `buf breaking`。desktop：`apps/ora-controller/tests/workspaces/restores.rs` 覆盖能力检查、中继不进日志、
拒绝转告、Cloud 不可用时重试与重连后等待 Node 重新请求；`crates/node-protocol` 覆盖前序在两种输入中的往返。
Cloud 集成测试尚未断言拒绝的具体错误码，Controller 尚无租约失效后重试的直接测试。
