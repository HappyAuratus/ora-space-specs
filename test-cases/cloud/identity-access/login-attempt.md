# 登录尝试与 provider 交换

## Login Start Requires Same-Origin Proof And Writes One Bound Attempt

- 风险：login start 会无条件写入一行 attempt。若它能被跨站导航、`<img>`、预取或无同源证明的请求触发，
  任何页面都能让受害者浏览器发起登录 CSRF，或无界灌入 `gateway_login_attempts`；若 `return_to` 未在
  写入前规范化，callback 会把用户重定向到攻击者控制的 origin。
- 前置状态：Gateway 已配置 public origin、至少一个 provider 与限流阈值；数据库中没有该浏览器的 attempt。
- 触发：浏览器向 `POST /auth/login` 提交 `{provider, returnTo}`，分别带正确 `Origin`、错误/缺失 `Origin`、
  各种形式的 `returnTo`、未知 provider、未知字段，或超过限流阈值连续发起。
- 必须成立：只有同源、JSON、限流内的请求写入一行 attempt 并返回 `authorizationUrl`；attempt Cookie 为
  `SameSite=Lax`、`HttpOnly`、`Path=/auth/callback`、`Max-Age` 等于 attempt 有效期；`returnTo` 被规范化为
  同源相对路径或在写入前被拒绝；authorize URL 携带 `state`、S256 `code_challenge` 和固定 `redirect_uri`
  且不请求 scope。
- 禁止结果：`GET /auth/login` 存在；无同源证明或被限流的请求写入 attempt；绝对、`//`、`/\` 或带反斜杠/
  控制字符的 `returnTo` 被保存；attempt secret 或 `state` 明文进入数据库。
- 决策依据：ADR D4、D8，不变量 6、13。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 无 `Origin`、外来 `Origin` 或 `GET` 的 start 被拒绝且不写 attempt；`Sec-Fetch-Site: same-origin` 可替代缺失的 `Origin` | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume`、`gateway.TestSameOriginRequiresExactOriginOrSameOriginFetchMetadata` | `Covered` |
| 绝对、`//`、`/\`、反斜杠、控制字符与超长 `returnTo` 在写入前失败 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume`、`gateway.TestNormalizeReturnToRejectsEveryOffOriginForm` | `Covered` |
| 超过限流阈值的 start 返回 `429` 且 attempt 行数不增加 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight`、`gateway.TestRateLimiterBurstRefillAndBoundedKeys` | `Covered` |
| attempt Cookie 的 `SameSite=Lax`、`Path`、`HttpOnly`、`Max-Age` 与前缀策略 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume`（startLogin）、`gateway.TestCookiePolicyEnforcesPrefixesScopeAndDatabaseBoundedMaxAge` | `Covered` |
| authorize URL 绑定 `state`、S256 challenge、固定 callback，不含 scope 或 secret | `github.TestAuthorizationURLBindsStateChallengeAndCallbackWithoutScopes` | `Covered` |
| 未知 provider 与未知 JSON 字段被拒绝 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| 数据库只出现 secret/state 的定长 digest，attempt secret 明文不出现 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |

## Callback Consumes An Attempt At Most Once

- 风险：同一 attempt 被重放、被错误 `state` 或缺失 Cookie 的请求完成，或两个并发 callback 各自创建一个
  session，会让攻击者用注入的 code 登录受害者浏览器，或让一次授权产生多份凭据。
- 前置状态：一个未消费、未过期的 attempt，浏览器持有对应 attempt Cookie，provider 已为该 attempt 签发
  code。
- 触发：分别以缺失 Cookie、错误 `state`、正确参数、再次重放正确参数，以及两个同时到达的正确 callback
  请求 `GET /auth/callback/github`。
- 必须成立：缺失 Cookie 与错误 `state` 统一返回 `401 login_failed`；一次失败的 callback 清除 attempt
  Cookie，使同一 attempt 之后的合法回跳也失败，用户必须重新 start；正确 callback `303` 到 attempt 中保存
  的 `return_to`；重放返回 `401`；并发 callback 恰好一个 `303`、另一个 `401`，数据库中只多出一个有效
  session；`consumed_at` 与 session 在同一事务提交。
- 禁止结果：failure 语义区分“attempt 不存在”与“state 错误”；重放或并发产生第二个 session；`Location`
  取自请求参数而非 attempt 行；provider 调用发生在事务或行锁内。
- 决策依据：ADR D4、D8，不变量 6、14。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 缺失 attempt Cookie、错误 `state`、重放均为统一 `401`，且不产生 session | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| 失败 callback 清除 attempt Cookie 并使原 attempt 不再可用 | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| 成功 callback 重定向到 attempt 保存的 `return_to`，attempt 被标记消费 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| provider 允许 code 重用时，两个并发 callback 只有一个创建 session | `integration.TestGatewayLoginAttemptReplayStateAndConcurrentConsume` | `Covered` |
| provider exchange 成功但 session 事务失败时没有 session、返回失败而非推断成功 | 没有数据库提交失败的故障注入 | `Missing` |
| provider 调用不在数据库事务或行锁内 | 结构上由 `Login.Callback` 的调用顺序保证，没有直接检测锁持有的测试 | `Missing` |

## Provider Rejection Never Creates A Session

- 风险：provider 拒绝（错误 code、verifier 不匹配、用户文档非法）若被当作成功，或 provider 超时被当作
  拒绝而丢弃可重试的 attempt，会产生无效身份的 session 或错误的用户提示。
- 前置状态：有效 attempt 与 Cookie；provider 会校验 `code_verifier` 与 challenge、拒绝未签发的 code。
- 触发：callback 携带 provider 从未签发的 code；callback 落到一个 PKCE 派生密钥不同的副本；provider
  返回错误对象、非 bearer token、非 200 用户响应、缺失/非数字/非正 `id`、超过 64 KiB 或非法 JSON；
  provider 超时或 context 取消。
- 必须成立：上述拒绝均为 `401 login_failed`，attempt 保持未消费，没有新 session；超时与取消是基础设施
  错误（`503 login_unavailable`），与拒绝区分；access token 与完整用户文档不离开适配器。
- 禁止结果：任一拒绝路径创建 session；超时被归类为 `login_failed`；provider 错误文本、token 或 code 回显
  给浏览器。
- 决策依据：ADR D3、D4、D8，不变量 5、6。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 未签发的 code 与 PKCE verifier 不匹配都得到 `401`，attempt 未消费，session 数不变 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight` | `Covered` |
| GitHub 错误对象（含 HTTP 200）、非 bearer、用户端点非 200、非法/缺失 `id`、超大或非法 JSON 均为 `ErrProviderRejected` | `github.TestExchangeFallsBackToLoginAndRejectsBadProviderAnswers` | `Covered` |
| 超时与取消是基础设施错误而非 provider 拒绝 | `github.TestExchangeReportsInfrastructureFailuresDistinctly` | `Covered` |
| 经过完整 HTTP 路径时 provider 超时返回 `503 login_unavailable` 且 attempt 保持可用 | 只有适配器层证据，没有通过 Gateway HTTP 的超时验收 | `Partial` |
| exchange 表单携带 `code`、`code_verifier`、`client_secret`、`redirect_uri`，用户读取使用换得的 token | `github.TestExchangeUsesVerifierReadsNumericIDAndDiscardsToken` | `Covered` |

## GitHub Identity Is Keyed By Numeric ID Within Cloud Field Limits

- 风险：用 login 或 email 做身份键会在改名后分裂或错误合并 Ora 用户；`displayName` 超过 Cloud 的 200
  字节上限会让 Cloud 返回 `401 invalid_identity`，把一次成功的外部登录变成无法解释的未认证。
- 前置状态：Cloud 中不存在该 GitHub 用户；provider 返回数字 `id`、`login`、`name`（可能超长或为空）。
- 触发：首次登录；provider 改变 `login`/`name` 后再次登录；provider 返回 255 字符或多字节超长 `name`。
- 必须成立：`user_identities` 中出现 `(github.com, <数字 id>)` 且仅一行；改名后仍映射同一 Cloud user；
  `displayName` 在离开 Authenticator 前按字节截断在 rune 边界；`name` 为空时使用 `login`。
- 禁止结果：以 login/email 作为 `subject`；超长 name 到达 Cloud；截断切断 UTF-8 序列；`users` 出现第二行。
- 决策依据：ADR D3，不变量 3、4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 首次登录建立 `github.com/<id>` identity 并 JIT 创建用户 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| 改名后再次登录不产生第二个用户 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight` | `Covered` |
| 超长多字节 name 被截断到 200 字节的 rune 边界，登录与 `/api/v1/me` 成功 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight`、`gateway.TestNormalizeMatchesCloudByteLimits` | `Covered` |
| `source`/`subject` 超过 128/512 字节或为空时拒绝而非截断 | `gateway.TestNormalizeMatchesCloudByteLimits` | `Covered` |
| `name` 为空时回退到 `login`；数字 `id` 以十进制字符串作为 subject | `github.TestExchangeFallsBackToLoginAndRejectsBadProviderAnswers`、`github.TestExchangeUsesVerifierReadsNumericIDAndDiscardsToken` | `Covered` |
