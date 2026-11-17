# Cloud 单进程：崩溃即中断、凭据不下发、持久化与连续运行

依据 [Cloud 单进程根决策](../../../decisions/cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `80e9c9f1` 的 `v2-server/`（路径相对该目录）；"A Deployment Keeps Working Past Twenty-Four Hours" 一条读取的是
cloud `0ca0a9a0`（含 cloud#358），"Every Tenant Route Authorizes Through One Entry" 一条读取的是 cloud `b03ac2b4`（含 cloud#363），
"Offline Deployment Serves Same-Origin Frontend And Preserves Data Without Demo Fixtures" 一条读取的是 cloud `85bc9a5f`（含 cloud#362）。
第一版是单机、Node 为本地子进程；多机和隔离不在根决策的承诺里，所以这里没有对应的用例。

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

## Every Tenant Route Authorizes Through One Entry

- 风险：某条租户 API 漏判成员身份，或各处判定条件不一致，一个团队的人读写到另一个团队的数据；被移出团队的人
  已排好的定时任务、任务运行仍以他的身份继续执行；只有 owner 能做的操作被普通成员做成。
- 前置状态：两个团队；一个用户只在其中一个团队里，团队里另有 owner 和普通成员；有人排了定时任务后被移出团队。
- 触发：非成员、其他团队的成员、未登录的调用方和普通成员逐条请求所有 `/tenants/{tid}` 路由；新增一条路由；被移出
  的人的定时任务到点触发、任务运行或执行会话开始。
- 必须成立：所有租户路由都经同一个判定函数和同一个路由中间件；非成员（含用户停用、删除，团队停用）一律 `404`，
  未登录 `401`，普通成员调 owner 路由 `403`；owner 路由只在路由表上声明；后台工作开始执行时用同一个判定，不通过
  就不开始；已开始的工作照常收尾。
- 禁止结果：有租户路由不经判定就进入 handler；绕过路由表注册路由；非成员得到 `404` 以外的结果；被移出的人的定时
  任务继续触发、执行准入继续放行；业务查询按调用者之外的条件读到别的团队的数据。
- 验证义务与证据：**Partial** — `internal/server/routes_test.go` 的 `TestEveryTenantRouteRequiresTenantAccess`、
  `TestRoutesAreRegisteredOnlyThroughTheRouteTable`、`TestRouterRefusesTenantRoutesWithoutTenantAuthorization`；
  `internal/store/access_test.go` 的 `TestTenantAccessGrantsRolesOnlyToActiveMembersOfActiveTenants`、
  `TestScheduledWorkStopsWhenTheCreatorLosesTenantAccess`；`internal/store/execution_access_test.go` 的
  `TestExecutionRuntimeWritesSurviveAccessRevocation`、`TestExecutionAdmissionChecksProjectTenant`；
  `internal/taskrun/taskrun_test.go` 的 `TestRevokedStarterLeavesTheIdleRunInPlace`。
  已覆盖：遍历路由表的全部 118 条租户路由，未登录 `401`、非成员和格式错误的租户 `404`（带 `no-store`），7 条 owner
  路由对普通成员 `403`，且 owner 路由清单固定在测试里；在路由表之外注册路由或把租户路由注册成别的级别，测试或启动
  失败；判定函数对无 membership、用户停用、用户删除、团队停用一律拒绝；被移出的创建者不再被定时扫描、触发被拒；
  撤权后执行会话创建和追加被拒、已开始的执行仍能写完；撤权后任务运行不替换已有的运行。
  仍缺：业务查询只按 `tenant_id` 过滤没有逐条的测试，现在只有项目、执行准入等少数代表性用例证明拿别的团队的
  资源 id 读不到数据。
- 决策依据：D4 第 3 条，D7 第 5 条。

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

## Offline Deployment Serves Same-Origin Frontend And Preserves Data Without Demo Fixtures

- 风险：部署启动清空或覆盖已有数据、混入演示账号与演示数据；离线包搬移后资源路径失效或夹带敏感文件；同源前端吞掉 `/api` 等后端路由或放过路径穿越；关闭官方市场后仍展示旧官方快照或向外同步。
- 前置状态：主机构建或解压了包含服务端、前端产物、相对路径资源清单和启动器的离线目录，且 `V2_DATA_DIR` 已有历史数据；部署显式设置 `V2_PLUGIN_MARKETPLACE_URL=""`。
- 触发：校验未携带 demo/cleanup 标志的启动预检、整体搬移离线目录、请求页面/后端前缀/越权路径、关闭官方市场后读取和同步插件目录。
- 必须成立：部署启动不修改已有数据并跳过演示准备（当前构建不支持 IDaaS 登录时必须在接触数据目录与数据库前拒绝启动）；离线打包只保留白名单交付物，搬移后相对路径与 SHA-256 校验通过并拒绝篡改或越界路径；同源前端只服务静态文件与无扩展名 HTML 导航，`/api`、`/auth`、`/internal` 等后端前缀交回后端且符号链接与路径穿越返回 `404`；显式关闭官方市场后目录不返回官方条目、拒绝官方同步与包解析，且重新启用后保留原快照。
- 禁止结果：部署模式清空数据或注入演示数据；离线包夹带 `users.json`、私有数据目录或包外路径；`/api` 请求落到 `index.html` 或穿越读出包外文件；关闭官方市场后仍可安装官方插件。
- 验证义务与证据：**Partial** — `internal/server/frontend_test.go` 的 `TestFrontendBoundary`、`TestFrontendPreservesRealAPIRouting`；
  `internal/preparedassets/assets_test.go` 的 `TestResourceRelocationAndLegacyAbsolutePath`；
  `internal/store/plugins_test.go` 的 `TestDisabledOfficialCatalogRetainsSnapshotWithoutOfferingInstall`；
  `scripts/package_offline_test.py` 的 `PackageBoundary.test_allowlist_relocation_and_corruption`；
  `deployment_test.go` 的 `TestDeploymentPreflightPreservesExistingData`、`TestOptionalIntegrationUnsetAndEmpty`。
  已覆盖：同源前端静态边界与后端路由隔离、相对路径资源搬移与摘要拒绝、关闭官方市场隐藏目录/拒绝同步并保留旧快照、离线打包白名单与搬移校验、预检失败不破坏已有数据。
  仍缺：部署模式不跑 seed/demo（在 I2 #1052 接入 IDaaS 登录并能于真实数据库启动 `V2_LOGIN=idaas` 前标为 Missing）。
- 决策依据：D2，D6 第 8 条。
