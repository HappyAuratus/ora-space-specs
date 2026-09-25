# Controller 消费 Watch 信号流验收

依据[已批准的后续决策](../../../decisions/controller/api-boundary/20260924-controller-consumes-watch-signals.md)。
决策尚未实现，下列义务的证据均为 `Missing`；表中的"所需直接证据"描述实现时应补的测试，落地后按实际测试名
更新并核对提交。行为测试以内存假 Cloud（由 `ora-controller-proto` 测试专用 feature 生成的服务端桩托管）为
对端、以 `ControllerRuntime` 驱动云端持久模式；状态迁移另以对流类型泛型的纯逻辑单元测试逐一覆盖。Cloud 服务端
一侧的信号、排空与保活证据见
[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md#watch-signals-only-accelerate-claiming)。

## Claiming Follows The Watch Stream State

- **风险**：流在线时仍按 `claim_interval_ms` 空转查询，流建立前已接受的工作因无人发信号而滞留，信号被丢弃后
  积压无人领取，或断流后不再领取。
- **前置状态**：云端模式 Controller 持有租约；假 Cloud 可入队工作、发送 `WorkAvailable`、以错误状态结束流，
  并记录全部调用。
- **触发**：`claim_interval_ms` 取远大于测试时长的值后发送 `WorkAvailable`；在 Controller 启动前入队工作；
  流在线且无信号时经过续约节拍；流以错误状态结束。
- **必须成立**：流建立后立即领取一次；`WorkAvailable` 立即触发领取；在线时领取节拍不领取，续约后兜底领取一次；
  流以错误结束后进入无流，按 `claim_interval_ms` 领取并在领取与续约节拍上重开流。
- **禁止结果**：在线时按 `claim_interval_ms` 领取、订阅前已接受的工作等到下一次续约才登记、断流后停止领取、
  未持有租约时打开流或领取。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 信号与流建立触发领取 | 以假 Cloud 为对端：`claim_interval_ms` 为 1 小时时，`WorkAvailable` 后迅速完成派发登记；启动前入队的工作在 `Watch` 打开后立即登记 | Missing |
| 在线时只按信号与兜底领取 | 状态机单元测试：在线状态下领取节拍不产生领取、续约节拍产生一次兜底领取、`NodeAssignment` 不产生领取 | Missing |
| 断流回退与重开 | 以假 Cloud 为对端：流以错误状态结束后按 `claim_interval_ms` 领取，假 Cloud 重新接受后流重开；状态机单元测试覆盖无流状态两个节拍上的打开与领取 | Missing |
| 无租约不打开、不领取 | 状态机单元测试：未持有租约时任一状态、任一节拍都不打开流、不领取 | Missing |

决策依据：D1、D2、D6，不变量 2、3。

## Drain Pauses Claiming Until A Watch Is Reestablished

- **风险**：Cloud 关停期间持续发出注定失败的领取，排空时释放租约或打断 Node 会话，Cloud 恢复后 Controller
  不再领取，或 `Watch` 被明确拒绝时无限期停止领取。
- **前置状态**：云端模式 Controller 持有租约且流在线；假 Cloud 可发送 `Drain` 并正常结束流、以指定状态拒绝后续
  `Watch`、之后重新接受。
- **触发**：发送 `Drain` 后正常结束流；只正常结束流而不发送 `Drain`；排空期间以 `UNAVAILABLE` 拒绝重开；
  排空期间在队列中积压工作后重新接受 `Watch`；排空期间以 `UNIMPLEMENTED` 等其他错误拒绝重开；排空期间丢弃
  epoch。
- **必须成立**：收到 `Drain` 或正常结束后不领取、继续续约、不释放租约，Node 会话照常；`UNAVAILABLE`、超时与
  资格失效保持排空；重开成功后立即领取并登记积压；以其他错误拒绝时转入无流并恢复周期领取。
- **禁止结果**：排空期间发出 `ClaimWork` 或 `RecordDispatch`、排空时调用 `ReleaseLease`、Cloud 恢复后仍不领取、
  `Watch` 被明确拒绝后永久不领取。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 排空期间不领取、不释放租约 | 以假 Cloud 为对端：`Drain` 后若干个领取节拍内调用序列只有续约与重开，没有 `ClaimWork` 与 `ReleaseLease`；等待时长以节拍倍数表达 | Missing |
| 恢复后立即领取积压 | 以假 Cloud 为对端：重新接受 `Watch` 后排空期间积压的工作全部登记 | Missing |
| 按打开结果决定去向 | 状态机单元测试：排空状态下打开成功、`UNAVAILABLE`、超时、资格失效与其他错误分别到达 D3 表中的状态；正常结束与 `Drain` 进入同一状态；以假 Cloud 为对端，排空后以 `UNIMPLEMENTED` 拒绝时恢复周期领取 | Missing |

决策依据：D3，不变量 3、4。

## One Claim Trigger Registers The Whole Backlog In Bounded Batches

- **风险**：信号被丢弃后积压只能按兜底节拍逐项排空，长队列推迟续约直到租约过期，或队头无法登记时反复告警。
- **前置状态**：假 Cloud 队列中有超过一批上限（16 项）的工作；另有一项按 Node 命令校验失败的工作。
- **触发**：只发送一个 `WorkAvailable`；让无法登记的工作位于队头。
- **必须成立**：一次触发连续领取并登记直到队列为空；满批后先处理关停与续约再继续；某一步失败即停止并等待下一次
  触发，同一队头只告警一次。
- **禁止结果**：每次触发只登记一项、满批时跳过续约、对同一队头重复告警或跳过它领取后续工作。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 连续登记到队列为空 | 以假 Cloud 为对端：入队 17 项以上、只发一个信号，全部登记 | Missing |
| 满批让出与失败即停 | 批量领取的单元测试：满批时报告仍有积压；队头失败时停止且只告警一次 | Missing |

决策依据：D4，不变量 1、5。

## Watch Lives Only Under The Held Epoch

- **风险**：陈旧 Controller 在打开 `Watch` 被判资格失效后继续领取或写入，或关停时流悬挂、租约未释放。
- **前置状态**：云端模式 Controller；假 Cloud 可以带 `STALE_CONTROLLER` 的 `FAILED_PRECONDITION` 拒绝 `Watch`，
  并能观察流被取消。
- **触发**：打开 `Watch` 时被判陈旧；持有租约且流在线时关停 Controller。
- **必须成立**：打开时被判陈旧即丢弃 epoch、关闭流，之后在重新取得租约前不领取、不写入；关停时先关闭流再以当前
  epoch 释放租约。
- **禁止结果**：丢弃 epoch 后仍发出 `ClaimWork`、`RecordDispatch` 或其他写入；关停后假 Cloud 仍持有订阅；
  以旧 epoch 释放租约。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 资格失效后停止领取与写入 | 以假 Cloud 为对端：打开 `Watch` 被判陈旧后的调用序列不含领取与写入；同时补上[持久协调双适配器核心用例](../persistence/coordination-store-adapters.md#remote-writes-retransmit-lost-replies-under-one-submission-identity)中"资格失效后停止写入"的缺口 | Missing |
| 关停关闭流并释放租约 | 以假 Cloud 为对端：关停后假 Cloud 观察到流被取消，并收到带正确 epoch 的 `ReleaseLease` | Missing |

决策依据：D1，不变量 2；持久协调后续决策 D1。

## Idle Cloud Connections Stay Alive And Lost Ones Fail Over

- **风险**：空闲连接被 Cloud 以 `GOAWAY` 反复断开，或经 NAT 静默失联的连接让 Controller 一直以为流在线。
- **前置状态**：Cloud 监听已放宽 PING 执行策略；Controller 的 Cloud 通道启用 30 秒 / 10 秒、空闲也发送的保活。
- **触发**：流在线且空闲超过 grpc-go 默认策略下的断开时间；对端停止回应 PING。
- **必须成立**：空闲连接不被 `GOAWAY`；对端不回应时连接关闭，流以错误结束并进入无流。
- **禁止结果**：保活触发 `too_many_pings`；失联连接上的流一直被当作在线。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空闲不被断开 | Cloud 侧由[监听保活用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md#controller-keepalive-pings-never-trigger-goaway)证明策略；Controller 侧以真实 Cloud 空闲超过 2 分钟的手工验证 | Missing |
| 失联后回退 | 以真实 Cloud 或可停止回应 PING 的对端验证流以错误结束并进入无流 | Missing |

决策依据：D5，不变量 6。
