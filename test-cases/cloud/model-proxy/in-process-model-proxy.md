# 模型接入：部署级连接、临时令牌、强制模型与连续运行

依据 [模型接入根决策](../../../decisions/cloud/model-proxy/0-in-process-model-proxy-with-deployment-connections.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `80e9c9f1` 的 `v2-server/`（路径相对该目录）；"Model Access Survives A Project Running Past Twenty-Four Hours"
一条读取的是 cloud `0ca0a9a0`（含 cloud#358）。根决策把 v2-server 现有的做法定为承诺，所以多数义务
已有测试；用量计量、限流和个人连接不在承诺里，这里没有对应的用例。

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

## An Ended Or Expired Token Stops New And In-Flight Requests

- 风险：会话结束或执行中断之后，Agent 还能用临时令牌发模型请求，或者结束前发出的请求继续消耗上游。
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
