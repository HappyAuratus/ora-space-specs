# Cloud 单进程：崩溃即中断、凭据不下发、持久化与连续运行

依据 [Cloud 单进程根决策](../../../decisions/cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `80e9c9f1` 的 `v2-server/`（路径相对该目录）；"A Deployment Keeps Working Past Twenty-Four Hours" 一条读取的是
cloud `0ca0a9a0`（含 cloud#358）。第一版是单机、Node 为本地子进程；多机和隔离不在
根决策的承诺里，所以这里没有对应的用例。

## Cloud Restart Interrupts Every Unfinished Execution Before Serving

- 风险：Cloud 重启后，重启前没跑完的执行仍显示为进行中，页面一直等；或者新请求在恢复完成前进来，读到半恢复的状态。
- 前置状态：重启前有正在进行的模型或 Agent 轮次、任务运行、工作流运行、定时触发和任务拆分。
- 触发：Cloud 进程崩溃或被停止后重新启动。
- 必须成立：开始服务请求之前，所有重启前未结束的执行都已进入中断终态，原因写明是重启；已产生的部分输出保留；被中断的会话不再接受追加。
- 禁止结果：重启后仍有处于进行中的旧执行；中断结果丢失已产生的部分输出；恢复写入失败时照样接纳新的工作。
- 验证义务与证据：**Partial** — `internal/executionruntime/runtime_test.go` 的 `TestRuntimeStartupInterruptsOldLiveSession`；
  `internal/store/chat_test.go` 的 `TestInterruptEndedChatSessionsAfterRecovery`；`internal/taskrun/taskrun_test.go` 的
  `TestInterruptEndsRunsLeftByThePreviousProcess`；`internal/scheduler/workflow_test.go` 的 `TestWorkflowRunFailedByRestartIsInterrupted`；
  `internal/store/workflow_engine_test.go` 的 `TestInterruptWorkflowRunsFailsOnlyLiveWork`；`internal/taskdecomp/taskdecomp_test.go` 的
  `TestRecoverEndsWhatTheLastProcessLeft`；`internal/scheduler/scheduler_test.go` 的 `TestCoordinatorRecoveryGateAndConcurrentTicks`。
  已覆盖：执行、对话、任务运行、工作流运行、定时触发和任务拆分各自在重启后进入中断，保留部分输出，恢复失败时不接纳新工作。
  仍缺：没有测试证明 HTTP 监听在所有恢复完成之后才开始接受请求，只有启动顺序的结构性事实。
- 决策依据：D3，不变量 2。

## Interrupted Executions Are Never Replayed By Infrastructure

- 风险：恢复逻辑自动重发被中断的模型请求、重跑 Agent 轮次或重复派发定时任务，产生重复的副作用。
- 前置状态：重启前有已派发但未结束的执行，以及已记录但未派发的定时触发。
- 触发：Cloud 重启后执行恢复。
- 必须成立：恢复只读取原尝试的结果并写入终态；是否重做由业务层或用户决定。
- 禁止结果：恢复期间发出新的模型请求、新的 Agent 轮次或重复派发同一触发。
- 验证义务与证据：**Covered** — `internal/scheduler/recovery_test.go` 的 `TestStartupRecoveryReadsOriginalAttemptWithoutReplay`；
  `internal/scheduler/scheduler_test.go` 的 `TestCoordinatorRestartRecoversWithoutReplayingAndDoesNotBlockOtherTasks`；
  `internal/executionruntime/runtime_test.go` 的 `TestRuntimeStartupInterruptsOldLiveSession`（被中断的会话再追加返回 `ErrSessionEnded`）。
- 决策依据：D3，不变量 4。

## No Request Takes A Global Serializing Lock

- 风险：重新引入一把所有事务都要取的锁，只读吞吐不再随并发增加（#1023 测得 v1 只读吞吐锁死在约 170–200 req/s）。
- 前置状态：多个用户并发读写不同的项目和会话。
- 触发：并发的只读请求；并发写不同业务对象的请求。
- 必须成立：只读请求不取锁；写请求只在同一业务对象上互斥。
- 禁止结果：存在所有请求或所有写事务共用的 advisory 锁；不同项目的写请求互相等待。
- 验证义务与证据：**Missing** — 没有并发测试。现有代码只按业务对象取锁（例如 `internal/store/plugins.go` 按空间加插件标识
  取 `pg_advisory_xact_lock`），但这只是结构性事实，不是测试证据。
- 决策依据：D5，不变量 6。

## Schema Upgrades Apply Versioned Migrations In Order

- 风险：升级时靠启动建表，漏掉列或约束的变更；新建的库和升级过的库 schema 不一致。
- 前置状态：一个用旧版本建好并有数据的库。
- 触发：用新版本启动 Cloud。
- 必须成立：按编号依次执行尚未应用的迁移；升级后的 schema 与新建库一致；已有数据保留。
- 禁止结果：迁移被跳过或乱序；升级后与新建库的 schema 不同。
- 验证义务与证据：**Missing** — 现在启动时建表（D6 保证清单第 5 项），由 I1 #1051 承接。
- 决策依据：D5。

## A Node Exit Ends Its Executions Without Resuming Elsewhere

- 风险：Node 进程退出或连接断开后，执行一直挂着；或者旧 Node 还没停就在同一个工作目录上起了新的。
- 前置状态：一个项目的 Node 上有正在进行的 Agent 轮次。
- 触发：Node 进程退出；Cloud 和 Node 之间的连接断开。
- 必须成立：这个 Node 上未结束的执行结束，不续跑；确认旧 Node 已停止之前不重建；其他项目的执行不受影响。
- 禁止结果：执行停在进行中；没有确认旧 Node 已停止就在同一工作目录上重建。
- 验证义务与证据：**Partial** — `internal/agentruntime/driver_ipc_test.go` 的 `TestDriverIPCDisconnectRequiresStopProofBeforeRebuild`。
  已覆盖：连接断开时当前轮次结束，确认旧 Node 停止之前不重建。与 D3 的出入：当前实现把它记为失败（`failed`，原因
  `agent_transport_closed`），不是中断终态。
- 决策依据：D3，不变量 3。

## Cloud Never Hands Model Keys Or Repository Tokens To The Node

- 风险：模型 Key 或外部仓库令牌出现在下发给 Node 的配置、消息或请求头里；以后做隔离时发现凭据早就在 Node 一侧。
- 前置状态：部署配置了模型上游的 Key；项目绑定了带令牌的私有外部仓库。
- 触发：Agent 经 Cloud 的模型代理发请求，并在请求里夹带自己的头；Cloud 拉取外部仓库；检查 Cloud 写给 Node 的配置和消息。
- 必须成立：Node 只拿到按会话签发的临时令牌；模型 Key 只由 Cloud 进程内的代理加到上游请求上；调用方的请求头不转发给
  上游；外部仓库令牌只在 Cloud 进程和它启动的 Git 子进程里。
- 禁止结果：下发给 Node 的配置或消息里出现模型 Key 或外部仓库令牌；调用方的头被转发到上游；上游地址可以携带凭据。
- 验证义务与证据：**Partial** — `internal/nodeservices/service_test.go` 的 `TestRemoteProviderForcesModelAndDoesNotLeakCredentials`、
  `TestCloseJoinsProviderAndRemovesOnlyCredentials`；`internal/nodeservices/provider_test.go` 的 `TestRemoteProviderLogsStatusWithoutSecrets`；
  `internal/executionruntime/runtime_test.go` 的 `TestModelsFromEnvironmentRejectsPartialConfigurationWithoutSecrets`。
  已覆盖：代理用后端的 Key 访问上游、不转发调用方的头、拒绝带凭据的上游地址、日志不含秘密。仍缺：没有检查写给 Node 的
  配置里不含 Key；外部仓库令牌还没有实现（I7 #1057）。
- 决策依据：D4，不变量 5。

## A Deployment Keeps Working Past Twenty-Four Hours

- 风险：Cloud 内部签发的凭据有固定有效期，部署连续运行一段时间后 clone 和模型访问突然失效，只有重启才恢复。
- 前置状态：一个项目的 Node 和本机服务已经连续运行超过 24 小时。
- 触发：新建会话（clone）；进行中的会话续期模型授权；Agent 发模型请求。
- 必须成立：三者都照常成功。
- 禁止结果：因证书或其他内部凭据过期而失败。
- 验证义务与证据：**Covered** — `internal/nodeservices/tls_test.go` 的 `TestServicesKeepWorkingLongAfterStart`：把本机服务的
  签发时刻放到 25 小时前和 3 年前，分别验证新会话用系统 git clone、Node 申请并续期模型授权、Agent 经代理发模型请求，三条
  路径互不依赖；任一凭据在这段时间内到期，对应路径就会因 TLS 校验失败。v2-server 内部有固定有效期的凭据只有本机服务的
  CA 与证书（`internal/nodeservices/tls.go`，有效期远长于任何一次不重启的运行）和模型授权令牌（`internal/nodeservices/grant.go`，
  短期、靠续期延长，续期路径在上述测试里）。
- 决策依据：D2，不变量 7。
