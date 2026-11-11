# Cloud 单进程：崩溃即中断、安全底线、持久化与演示路径

依据 [Cloud 单进程根决策](../../../decisions/cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `ad324c77` 的 `v2-server/`（路径相对该目录）。执行宿主、Node 容器、Git server、企业登录都还没有实现，
相关义务保持 **Missing**，由 A2–A6 approved 后各自的核心用例细化；本文只登记根决策自身的承诺。

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
- 决策依据：D3，不变量 3。

## Interrupted Executions Are Never Replayed By Infrastructure

- 风险：恢复逻辑自动重发被中断的模型请求、重跑 Agent 轮次或重复派发定时任务，产生重复的副作用。
- 前置状态：重启前有已派发但未结束的执行，以及已记录但未派发的定时触发。
- 触发：Cloud 重启后执行恢复。
- 必须成立：恢复只读取原尝试的结果并写入终态；是否重做由业务层或用户决定。
- 禁止结果：恢复期间发出新的模型请求、新的 Agent 轮次或重复派发同一触发。
- 验证义务与证据：**Covered** — `internal/scheduler/recovery_test.go` 的 `TestStartupRecoveryReadsOriginalAttemptWithoutReplay`；
  `internal/scheduler/scheduler_test.go` 的 `TestCoordinatorRestartRecoversWithoutReplayingAndDoesNotBlockOtherTasks`；
  `internal/executionruntime/runtime_test.go` 的 `TestRuntimeStartupInterruptsOldLiveSession`（被中断的会话再追加返回 `ErrSessionEnded`）。
- 决策依据：D3，不变量 5。

## Losing A Node Host Or Machine Interrupts Only That Machine's Executions

- 风险：Node 进程退出、执行宿主断开或机器失联后，执行一直挂着，或者在别的机器上被续跑；一台机器的故障波及其他机器上的执行。
- 前置状态：多台机器各有正在执行的项目。
- 触发：Node 进程退出；宿主和 Cloud 之间的连接断开；机器失联超过宽限期。
- 必须成立：受影响机器上未结束的执行进入中断；其他机器上的执行不受影响；项目下次使用时可以在别的机器上重建 Node。
- 禁止结果：执行停在进行中；执行在别的机器上从中途续跑；没有确认旧 Node 已停止就在同一工作目录上重建。
- 验证义务与证据：**Partial** — `internal/agentruntime/driver_ipc_test.go` 的 `TestDriverIPCDisconnectRequiresStopProofBeforeRebuild`。
  已覆盖：单机下 Node 连接断开时当前轮次结束，确认旧 Node 停止之前不重建。与 D3 的出入：当前实现把它记为失败（`failed`，原因
  `agent_transport_closed`），不是中断终态。仍缺：执行宿主和多机都还没有实现，宿主断开、机器失联和跨机器重建都没有证据（I4 #1054）。
- 决策依据：D3，不变量 2、4。

## Agent Code Cannot Reach The Cloud Database, Other Projects Or Credentials

- 风险：Agent 执行的任意代码连上 PostgreSQL、读到别的项目的仓库或会话、拿到外部仓库凭据，或访问内网其他机器。
- 前置状态：Node 跑在执行宿主管理的容器里；`restricted` 与 `open` 两种出网模式。
- 触发：Agent 在容器内访问数据库地址、宿主本机端口、其他机器、其他项目的 Git 路径和 Cloud 的凭据接口。
- 必须成立：两种出网模式下都只能访问 Cloud 入口和管理员白名单（`open` 另外允许公网）；项目令牌只能访问本项目的数据。
- 禁止结果：任一模式下连上数据库、宿主本机端口或其他机器；用本项目的令牌读到其他项目的数据。
- 验证义务与证据：**Missing** — 现在 Node 是和后端同一用户的本地子进程，没有隔离（D6 保证清单第 3 项）。容器与网络规则由
  A3 #1047、I5 #1055 承接，令牌范围由 A2、A4、A5 承接。
- 决策依据：D4 第 1、4 条，不变量 6。

## Model Keys And Repository Credentials Never Enter The Node

- 风险：模型 Key 或外部仓库令牌出现在 Node 的环境变量、配置文件、请求头或日志里，被 Agent 读走。
- 前置状态：部署配置了模型上游的 Key；项目绑定了带凭据的外部仓库。
- 触发：Agent 经 Cloud 的模型代理发请求，并在请求里夹带自己的头；Cloud 同步或推回外部仓库。
- 必须成立：Node 只持有按会话签发、可吊销的临时令牌；模型 Key 只由 Cloud 进程内的代理加到上游请求上；调用方的请求头不转发给上游；
  外部仓库凭据只在 Cloud 进程内使用。
- 禁止结果：Node 的环境、文件或日志里出现模型 Key 或外部仓库凭据；调用方的头被转发到上游；上游地址可以携带凭据。
- 验证义务与证据：**Partial** — `internal/nodeservices/service_test.go` 的 `TestRemoteProviderForcesModelAndDoesNotLeakCredentials`、
  `TestCloseJoinsProviderAndRemovesOnlyCredentials`；`internal/nodeservices/provider_test.go` 的 `TestRemoteProviderLogsStatusWithoutSecrets`；
  `internal/executionruntime/runtime_test.go` 的 `TestModelsFromEnvironmentRejectsPartialConfigurationWithoutSecrets`。
  已覆盖：代理用后端的 Key 访问上游、不转发调用方的头、拒绝带凭据的上游地址、日志不含秘密。仍缺：没有扫描 Node 进程的环境和
  数据目录确认其中没有 Key；外部仓库凭据还没有实现（A4 #1048、I7 #1057）。
- 决策依据：D4 第 2、3 条，不变量 7。

## Cross-Machine Links Use Scoped Revocable Tokens Over Server TLS

- 风险：执行宿主或 Node 不经认证就能连上 Cloud；令牌泄露后无法吊销；一台机器的令牌能冒充别的机器或别的项目。
- 前置状态：管理员为每台机器签发了宿主令牌；Cloud 为每个项目的 Node 签发了项目令牌。
- 触发：用缺失、过期、已吊销或属于别的机器、别的项目的令牌连接；不经 TLS 连接。
- 必须成立：跨机器的连接都经 Cloud 的 HTTPS 入口；令牌缺失、已吊销或范围不符时拒绝；吊销之后，已建立的连接也被断开。
- 禁止结果：明文连接被接受；吊销后的令牌仍可用；一个项目的令牌访问到别的项目。
- 验证义务与证据：**Missing** — 现在只有本机 Unix socket 和 24 小时不轮换的自签证书；由 A2 #1046、I4 #1054 承接。
- 决策依据：D2，不变量 8。

## No Request Takes A Global Serializing Lock

- 风险：重新引入一把所有事务都要取的锁，只读吞吐不再随并发增加（#1023 测得 v1 只读吞吐锁死在约 170–200 req/s）。
- 前置状态：多个用户并发读写不同的项目和会话。
- 触发：并发的只读请求；并发写不同业务对象的请求。
- 必须成立：只读请求不取锁；写请求只在同一业务对象上互斥。
- 禁止结果：存在所有请求或所有写事务共用的 advisory 锁；不同项目的写请求互相等待。
- 验证义务与证据：**Missing** — 没有并发测试。现有代码只按业务对象取锁（例如 `internal/store/plugins.go` 按空间加插件标识
  取 `pg_advisory_xact_lock`），但这只是结构性事实，不是测试证据。
- 决策依据：D5，不变量 9。

## Schema Upgrades Apply Versioned Migrations In Order

- 风险：升级时靠启动建表，漏掉列或约束的变更；新建的库和升级过的库 schema 不一致。
- 前置状态：一个用旧版本建好并有数据的库。
- 触发：用新版本启动 Cloud。
- 必须成立：按编号依次执行尚未应用的迁移；升级后的 schema 与新建库一致；已有数据保留。
- 禁止结果：迁移被跳过或乱序；升级后与新建库的 schema 不同。
- 验证义务与证据：**Missing** — 现在启动时建表（D6 保证清单第 7 项），由 I1 #1051 承接。
- 决策依据：D5。

## The Demo Starts With One Command While New Paths Are Developed

- 风险：新路径（执行宿主、容器、Git server、企业登录）的开发改变了演示路径的行为，`task v2:demo` 起不来或演示场景失败。
- 前置状态：演示结束之前；新路径的代码已经合入，但默认关闭。
- 触发：在干净的环境里运行 `task v2:demo`。
- 必须成立：一条命令起来，走本地 Node 子进程和本地 mirror 路径，演示场景照常可用。
- 禁止结果：要额外的配置或命令；默认走了新路径。
- 验证义务与证据：**Missing** — 没有自动测试，现在靠人工跑演示确认；新路径的实现 PR 要在描述里写明对演示路径的验证。
- 决策依据：D7，不变量 10。
