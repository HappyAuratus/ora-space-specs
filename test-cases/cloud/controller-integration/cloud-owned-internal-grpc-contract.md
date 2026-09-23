# Cloud 内部控制契约验收

依据[已实现的根决策](../../../decisions/cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)。
证据核对 ora-space/cloud#24 栈顶 `9b33596ddee1e59e90f622181fb88e88afa95cdf`（cloud#20–#24 尚未合并，契约本身
由 cloud#19 合入主干）；下列 PostgreSQL 测试已在该提交上以 `REQUIRE_POSTGRES=1` 本地重跑通过。

代表性证据写作 `包.测试函数`：`integration.*` 位于 `integration/control_grpc_test.go` 与
`integration/clone_api_test.go`，使用真实 PostgreSQL、真实 gRPC 服务端与拦截器；`controlgrpc.*` 与 `core.*`
是对应包的单元测试。Controller 侧客户端、适配器与跨仓库链路的证据见
[API 边界核心用例](../../controller/api-boundary/controller-hosted-client-api.md#generated-cloud-contracts-preserve-the-boundary)、
[持久协调双适配器核心用例](../../controller/persistence/coordination-store-adapters.md)与
[minicloud 云端形态核心用例](../../minicloud/runtime/cloud-hosted-clone-loop.md)。

## Contract Changes Stay Additive And Generated Code Never Drifts

- **风险**：契约被破坏性修改而锁定旧 commit 的消费方失效，或生成代码与 proto 不一致。
- **前置状态**：`proto/ora/cloud/internal/v1/` 为唯一契约源，`internal/controlpb` 为提交的生成物。
- **触发**：修改 proto 后提交 PR。
- **必须成立**：`STANDARD` lint 通过；相对目标分支无 `FILE` 级破坏性变更；重新生成后无 diff。
- **禁止结果**：手改生成物、在 `v1` 内删除或改号字段、契约出现租户／用户字段。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| lint、breaking 与生成漂移 | `.github/workflows/backend.yml` 在每个 PR 上执行 `buf lint`、以目标分支为基线的 `buf breaking` 与 `buf generate` 后的 diff 检查；本地对应 `task proto:lint`／`proto:breaking`／`proto:check` | Covered |
| 契约不含租户字段 | 目前只依赖评审；没有对 proto 字段集合的自动检查 | Missing |

决策依据：D1，不变量 1、8。

## Declared Controller Identity Holds The Lease And Stale Epochs Are Fenced

- **风险**：无身份的调用被当作某个 Controller，两个 Controller 同时持有租约，或陈旧 worker 继续写入。
- **前置状态**：真实 gRPC 服务端与 PostgreSQL；两个声明不同 `ControllerId` 的调用方。
- **触发**：不带或带空白 `x-ora-controller-id` 调用；A 取得租约后 B 获取／续期；A 用错误 epoch 续期；A 释放后 B 获取；A 以旧 epoch 续期或领取。
- **必须成立**：缺失或空白身份 `INVALID_ARGUMENT`；单一持有者，他人 `LEASE_HELD`；续期保持 epoch 并延长过期；交接后 epoch 递增；陈旧 epoch 的续期、领取与写入均 `STALE_CONTROLLER`。
- **禁止结果**：无身份调用成功、两个持有者并存、交接不递增 epoch、陈旧 epoch 写入成功。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 身份声明与单持有者租约 | `integration.TestControlGRPCLeaseHolderAndFencing` 覆盖缺失／空白身份、`LEASE_HELD`、续期保持 epoch、释放后交接 epoch 递增、旧持有者续期被拒 | Covered |
| 写入与领取的 epoch fencing | `integration.TestControlGRPCCloneLoopWithSubmissionReplay` 以 `epoch + 1` 领取被拒为 `STALE_CONTROLLER`；尚缺以陈旧 epoch 调用 `RecordDispatch`／`TakeOverNodeEvent` 的直接断言 | Partial |

决策依据：D2、D4、D7，不变量 6、10。

## Submission Replay Returns The Original Response

- **风险**：回复丢失后的重传被当作新提交重复生效，或被当作陈旧写入拒绝，Controller 无法区分"已提交"与"从未提交"。
- **前置状态**：已登记一次写提交并记录响应。
- **触发**：同一 `submission_id` 同内容重传；同一 `submission_id` 改变内容重传。
- **必须成立**：同身份同内容返回记录的响应且不重复生效；同身份不同内容 `ABORTED` + `CONFLICT`，原记录不变。
- **禁止结果**：重传产生第二次状态变化、覆盖原提交记录、把已提交的重传报为冲突。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 同身份回放与冲突 | `integration.TestControlGRPCCloneLoopWithSubmissionReplay` 覆盖 `RecordDispatch`／`TakeOverNodeEvent` 同身份重传返回原记录、改变内容被 `ABORTED` 拒绝；但这些方法本身也按业务身份幂等，尚缺"业务规则已会拒绝、但提交记录回放原成功响应"的隔离断言 | Partial |
| 提交记录有界 | ADR 明确当前不清理，留作未决 | Missing |

决策依据：D3，不变量 3。

## Dispatch Registration And Event Takeover Commit Before Delivery And Ack

- **风险**：同一请求出现第二个执行、伪造或变更的事件被确认，或查询结果被当作 Ack 依据。
- **前置状态**：Cloud 已接受一个 clone 请求；Controller 持有租约。
- **触发**：领取、登记派发、以不同执行再次登记、接管事件、以同序号不同内容接管、提交冲突的查询结果与相同的查询结果。
- **必须成立**：领取是纯读，只返回 `queued` 请求；登记后请求不再可领取；同一请求第二个执行冲突；接管保存结果与唯一收据并使请求进入终态；同序号不同内容冲突；冲突结果不覆盖原事实；查询结果不增加收据。
- **禁止结果**：领取改变归属、两个执行、覆盖原结果、为查询结果写入收据。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 领取、登记与单执行 | `integration.TestControlGRPCCloneLoopWithSubmissionReplay` 覆盖空队列领取、登记后不可再领取、新提交身份下的第二个执行仍冲突 | Covered |
| 接管、收据与查询结果 | 同一测试覆盖接管后请求 `succeeded`、收据恰为一条、伪造事件与冲突结果被拒、相同查询结果幂等 | Covered |

决策依据：D4，不变量 4、5。

## Recovery Reads Locate The Original Execution Without A Lease

- **风险**：替换 worker 在取得租约前无法定位原执行而另建执行，或已完成执行仍被当作待协调。
- **前置状态**：一个已登记、尚无结果的执行；另一个未持有租约的 Controller。
- **触发**：未持租约者调用 `ListPendingDispatches`／`GetDispatch`；执行完成后再次列出；查询不存在的执行。
- **必须成立**：未持租约者可读到待协调执行与原输入；完成后不再列出；缺失为 `NOT_FOUND`；读取仍要求声明身份。
- **禁止结果**：恢复读取要求租约、已完成执行仍列为待协调、缺失被伪装为空结果。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 无租约恢复读取 | `integration.TestControlGRPCCloneLoopWithSubmissionReplay` 以 `controller-b` 读取待协调执行与完成后的结果，缺少身份被拒，完成后列表为空，缺失为 `NOT_FOUND` | Covered |

决策依据：D4，不变量 6。

## Watch Signals Only Accelerate Claiming

- **风险**：陈旧 Controller 收到信号、信号早于接受事务提交、慢订阅者阻塞接受，或关停时流悬挂。
- **前置状态**：Controller 持有租约并建立 `Watch`。
- **触发**：以陈旧 epoch 建立流；接受新 clone 请求；幂等重复提交；订阅者不读取；Cloud 关停。
- **必须成立**：陈旧 epoch `STALE_CONTROLLER`；接受提交后发送带 `operation_id` 的 `WorkAvailable`；重复提交不发信号；缓冲有界且满时丢弃；`Drain` 后流正常结束，之后的 `Watch` 为 `UNAVAILABLE`。
- **禁止结果**：信号改变归属、阻塞接受事务、关停后仍接受订阅。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 资格、时机与排空 | `integration.TestControlGRPCWatchDeliversWorkAndDrain` 覆盖陈旧 epoch、提交后 `WorkAvailable`、`Drain` 后 EOF 与关停后拒绝订阅；`integration.TestPublicClonesAcceptOnceAndReflectExecutionFacts` 覆盖重复提交不发信号 | Covered |
| 有界、不阻塞 | `core.TestControlHubDropsOnFullAndDrainsOnce` 覆盖缓冲上限 16、满时丢弃、重复 `Drain` 只生效一次 | Covered |

决策依据：D5，不变量 7。

## Control Errors Carry One Consistent Classification

- **风险**：状态码与 `ErrorDetail` 不一致，业务冲突被误判为资格失效，或数据库细节泄露给调用方。
- **前置状态**：`core.Fault` 的各类裁决与非业务错误。
- **触发**：把每类裁决转换为 gRPC 状态。
- **必须成立**：映射与 ADR D6 表一致；`FAILED_PRECONDITION` 只用于租约类裁决；非业务错误为 `UNAVAILABLE` 且消息不含内部细节。
- **禁止结果**：同一裁决在不同路径得到不同状态码、业务冲突返回 `FAILED_PRECONDITION`。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 映射表 | `controlgrpc.TestFaultMapping` 覆盖全部分支及 `UNAVAILABLE` 消息脱敏；集成测试以 `expectStatus` 同时断言状态码与 detail | Covered |

决策依据：D6，不变量 9。

## Public Clone Intake Mirrors The Transitional Contract

- **风险**：公开入口与过渡 JSON 接口语义漂移，接受无法派发的请求使其永远 pending，或跨用户泄露。
- **前置状态**：真实 Cloud 公开 API、租户与两个成员。
- **触发**：无幂等键提交；提交、跨幂等键重复、同身份改输入、复用幂等键改 body；提交各类非法来源；列表与单项读取；另一成员读取；Controller 登记与接管成功／失败结果。
- **必须成立**：缺少幂等键 `400`；`202` 返回 pending 视图；`requestId` 跨键返回原请求，改输入 `409`；非法来源 `400`；仅提交者可见；视图随登记与结果变化，失败原因为过渡 DTO 的 camelCase 值。
- **禁止结果**：同一 `requestId` 产生两个请求、非法来源进入队列、其他成员读到请求、暴露收据。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接受、幂等与校验 | `integration.TestPublicClonesAcceptOnceAndReflectExecutionFacts` | Covered |
| 可见性与视图投影 | 同一测试覆盖另一成员不可见、登记后视图含 `executionId`／`nodeId`、成功与失败状态投影 | Covered |

决策依据：D8，不变量 2、8。
