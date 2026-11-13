# 模型接入：部署级连接、临时令牌、强制模型与连续运行

依据 [模型接入根决策](../../../decisions/cloud/model-proxy/0-in-process-model-proxy-with-deployment-connections.md)（`approved`）
和它的后续决策 [长请求的空闲超时与总上限](../../../decisions/cloud/model-proxy/20261113-in-flight-model-calls-bounded-by-idle-and-total-limits.md)
（`implemented`，下称“后续决策”）。没有特别注明时，D 编号与不变量编号指根决策；最后三条用例的编号指后续决策。

证据读取的是 cloud `80e9c9f1` 的 `v2-server/`（路径相对该目录）；"Model Access Survives A Project Running Past Twenty-Four Hours"
一条读取的是 cloud `0ca0a9a0`（含 cloud#358）。根决策把 v2-server 现有的做法定为承诺，所以多数义务
已有测试；用量计量、限流和个人连接不在承诺里，这里没有对应的用例。后续决策的三条用例读取 cloud `4183eb9b`
（ora-space/cloud#367 的合并提交）。

## Upstream Keys Stay In The Cloud Process

- 风险：上游的 Key 或上游地址出现在 API 应答、日志或下发给 Node 的消息里，被用户或 Agent 拿到。
- 前置状态：部署的环境变量配置了上游地址、Key 和模型。
- 触发：读取模型目录接口；Agent 经代理发请求，上游返回错误；环境变量只配了一部分。
- 必须成立：模型目录只返回标识和显示名；日志和错误信息不含 Key；配置不完整时 Cloud 拒绝启动，报错不含已配置的秘密。
- 禁止结果：Key、上游地址或上游的模型名出现在目录接口的应答里；Key 出现在日志里。
- 验证义务与证据：**Partial** — `internal/server/catalog_test.go` 的 `TestExecutionCatalogHTTP`；
  `internal/nodeservices/provider_test.go` 的 `TestRemoteProviderLogsStatusWithoutSecrets`；
  `internal/executionruntime/runtime_test.go` 的 `TestModelsFromEnvironmentRejectsPartialConfigurationWithoutSecrets`。
  已覆盖：目录不泄露上游绑定，日志不含秘密，不完整的配置被拒绝且不带出秘密。仍缺：没有检查写给 Node 的配置和消息里
  不含 Key。
- 决策依据：D1，不变量 1。

## A Session Token Is Issued Only For An Admitted Session Of The Same Project

- 风险：没有被 Cloud 登记的执行申请到临时令牌；一个项目的 Node 用到另一个项目的令牌；不带证书也能申请。
- 前置状态：两个项目各有自己的本机服务；其中一个项目有一个已登记的 Agent 会话。
- 触发：不带客户端证书申请授权；为没有登记的执行申请；用别的执行标识续期；把一个项目的临时令牌拿到另一个项目的
  代理上用；同一个执行申请第二次后用第一次的令牌。
- 必须成立：只有带本项目证书、为已登记的执行申请才成功；别的执行不能续期；令牌在另一个项目的服务上无效；
  重新申请后旧令牌作废。
- 禁止结果：未登记的执行拿到令牌；跨项目使用成功；旧令牌仍然可用。
- 验证义务与证据：**Covered** — `internal/nodeservices/service_test.go` 的 `TestServicesRequireIdentityAdmissionAndLiveGrant`、
  `TestGrantReplacementExpiryIsolationAndPackageSnapshot`。
- 决策依据：D2、D3，不变量 3。

## An Ended Session Stops Its Requests And An Expired Token Starts None

- 风险：会话结束或执行中断之后，Agent 还能用临时令牌发模型请求，或者结束前发出的请求继续消耗上游；过期的令牌还能
  发起新请求。令牌过期本身不取消进行中的请求，那一条见下文“An Admitted Call Is Not Cut By Its Token's Expiry”。
- 前置状态：一个会话持有有效的临时令牌，并有一个正在进行的模型请求。
- 触发：会话结束；令牌过期后发新请求和续期；Cloud 重启后用重启前的令牌。
- 必须成立：会话结束时进行中的上游请求被取消；过期的令牌不能发新请求，也不能续期；重启前的令牌在重启后无效。
- 禁止结果：会话结束后上游请求继续；过期或重启前的令牌被接受。
- 验证义务与证据：**Partial** — `internal/nodeservices/service_test.go` 的 `TestEndCancelsActiveProvider`、
  `TestGrantReplacementExpiryIsolationAndPackageSnapshot`、`TestRemoteProviderCancelsAndDiscardsErrorBody`。
  已覆盖：结束会话取消进行中的上游请求，过期的令牌不能使用和续期，取消会传到上游。仍缺：没有测试直接证明 Cloud 重启后
  重启前的令牌无效（令牌只在内存里，这是结构性事实）。
- 决策依据：D3，不变量 5、6。

## The Proxy Forces The Frozen Model And Forwards Nothing From The Caller

- 风险：Agent 在请求里换模型、夹带自己的认证头，或把临时令牌带到上游；上游的错误正文泄露给 Agent。
- 前置状态：一个会话冻结了某个模型绑定。
- 触发：Agent 的请求体写了别的模型名，并带上额外的请求头；用 OpenAI 协议的令牌访问 Anthropic 协议的路径；上游返回
  带正文的错误；访问代理路径以外的路径。
- 必须成立：发往上游的模型名是冻结的那一个，认证头是上游的 Key，调用方的头和临时令牌不转发；令牌只在自己的协议路径上
  有效；上游的错误正文被丢弃，Agent 只得到固定的错误码。
- 禁止结果：上游收到调用方选择的模型或认证头；上游的错误正文到达 Agent。
- 验证义务与证据：**Covered** — `internal/nodeservices/service_test.go` 的 `TestRemoteProviderForcesModelAndDoesNotLeakCredentials`、
  `TestRemoteProviderCancelsAndDiscardsErrorBody`；`internal/nodeservices/anthropic_test.go` 的
  `TestGrantTokenWorksOnlyOnItsProtocolRoute`、`TestModelRoutesReadTokensOnlyFromTheirHeader`、
  `TestRemoteMessagesProviderUsesItsKeyAndProtocolHeaders`。
- 决策依据：D4，不变量 2、4、7。

## Model Access Survives A Project Running Past Twenty-Four Hours

- 风险：本机服务的证书到期后，Node 申请和续期授权失败，进行中的会话被取消，新会话起不来。
- 前置状态：一个项目的本机服务已经连续运行超过 24 小时。
- 触发：进行中的会话续期；新会话申请授权；Agent 发模型请求。
- 必须成立：三者都照常成功。
- 禁止结果：因证书过期而失败。
- 验证义务与证据：**Covered** — `internal/nodeservices/tls_test.go` 的 `TestServicesKeepWorkingLongAfterStart`：把本机服务的
  签发时刻放到 25 小时前和 3 年前，分别验证 Node 申请授权后续期、Agent 凭临时令牌经代理发模型请求（另有新会话 clone），
  各路径互不依赖；证书在这段时间内到期，对应路径就会因 TLS 校验失败。
- 决策依据：D2。

## An Admitted Call Is Not Cut By Its Token's Expiry

- 风险：用有效令牌发起的长请求，在令牌到期时被切断——无论 Node 是否已经续期；或者在别的执行签发授权、续期被拒时，
  顺带被清理掉。
- 前置状态：一个会话持有临时令牌，并有一个正在进行的模型请求。
- 触发：请求进行中 Node 续期，然后越过续期前的到期时刻；请求进行中令牌过期，随后该令牌续期被拒、另一个执行签发授权
  （清理过期授权）、用过期令牌发新请求；最后会话结束。
- 必须成立：进行中的请求跑完并正常收尾；过期的令牌不能续期、不能发新请求；这个授权在最后一个请求结束后被移除；会话结束
  仍会取消它进行中的请求。
- 禁止结果：请求因令牌到期、过期续期被拒或过期清理而中断；过期的令牌发起了新请求；过期授权的请求不能再被吊销。
- 验证义务与证据：**Covered** — `internal/nodeservices/limits_test.go` 的
  `TestRenewedGrantKeepsInFlightCallPastOriginalExpiry`（续期后请求跑过原到期时刻）、
  `TestExpiredGrantKeepsRunningCallsUntilRevoked`（过期、续期被拒、别的执行触发清理后请求仍收尾，授权随后移除；过期后
  `End` 仍取消）；`internal/nodeservices/service_test.go` 的 `TestEndCancelsActiveProvider`（吊销取消进行中的请求）。
- 决策依据：后续决策 D1，不变量 1、2；根决策 D3。

## Upstream Calls Are Bounded By Idle And Total Limits From Deployment Configuration

- 风险：长推理或长输出因为固定的总时限被切断；或者卡住的上游无限占用连接和额度；或者部署配置写错后静默回退到默认值。
- 前置状态：部署配置了（或没有配置）空闲超时和总上限；上游可能延迟发送响应头、输出后停住、或一直在输出。
- 触发：上游在空闲超时之内发出响应头；上游晚于空闲超时才发响应头；上游输出几段后停住；上游持续输出超过总上限；
  `V2_MODEL_IDLE_TIMEOUT`、`V2_MODEL_MAX_DURATION` 未设置、设为合法值、设为非法值或空闲超时大于总上限。
- 必须成立：等待响应头只受空闲超时约束；连续超过空闲超时没有数据的请求被停止；持续超过总上限的请求被停止；未设置时
  取默认值（2 分钟、30 分钟），合法值应用到两种协议的上游，非法值使配置被拒绝。
- 禁止结果：存在独立于空闲超时的固定响应头时限或固定总时限；持续输出的请求被空闲超时切断；非法配置被接受。
- 验证义务与证据：**Covered** — `internal/nodeservices/limits_test.go` 的 `TestRemoteProviderHeaderWaitBoundedOnlyByIdleLimit`、
  `TestRemoteProviderIdleTimeoutAbortsResponse`、`TestRemoteProviderMaxDurationAbortsActiveStream`（流一直不空闲，只有总上限
  能结束它）、`TestRemoteProviderLimitsValidation`；`internal/executionruntime/runtime_test.go` 的
  `TestModelsFromEnvironmentAppliesRequestLimits`。
- 决策依据：后续决策 D2，不变量 3、5。

## A Call Stopped After Its Header Reaches The Agent As A Failure

- 风险：响应头发出后请求被停止，流却像正常结束一样收尾，Agent 把半截内容当成完整回答并带着它继续请求；或者为了中止
  响应而打断了不经 Agent 路由的进程内调用方。
- 前置状态：Agent 经模型路由的请求已经收到响应头和部分内容；生产装配中路由与 provider 之间有请求记录的包装。
- 触发：空闲超时、总上限或吊销（`End`）在响应头之后停止请求；经请求记录包装时被空闲超时停止；纯模型轮次在收到
  `[DONE]` 后结束上游连接。
- 必须成立：Agent 读到的是失败的响应（连接中止），而不是正常结束的流；经请求记录包装时同样中止，并记为中断；进程内的
  纯模型轮次照常完成。
- 禁止结果：被停止的流以正常结束收尾；请求记录把它记为成功；进程内调用方因中止而崩溃。
- 验证义务与证据：**Covered** — `internal/nodeservices/limits_test.go` 的 `TestRemoteProviderIdleTimeoutAbortsResponse`
  （直接与经 `Unwrap` 包装两种）、`TestRemoteProviderMaxDurationAbortsActiveStream`、`TestRevocationAfterHeaderAbortsRemoteResponse`；
  `internal/nodeservices/metrics_route_test.go` 的 `TestCutOffThroughRequestMetricsAbortsAndRecords`（真实路由加
  `requestmetrics.Wrap`，断言中止并记为 `interrupted`）；`internal/requestmetrics/recorder_test.go` 的
  `TestRecorderRecordsCutOffAsInterrupted`；`internal/modelturn/compatibility_test.go` 的
  `TestRemoteProviderCompletionStopsOpenStreamAndJoins`（纯模型轮次在 `[DONE]` 后结束上游，不受中止影响）。Agent 收到中止后
  的重发与显示不在本用例，由 circus #1132 跟进。
- 决策依据：后续决策 D3，不变量 4。
