# 代理与内部凭据

## Cloud Only Receives Gateway-Issued Short-Lived Credentials

- 风险：若浏览器提供的 `Authorization`、`X-Ora-User-Token`、`Cookie` 或 `X-Forwarded-*` 被原样转发，或
  内部 JWT 按 session 期限签发、用同一把密钥签两种 kind，Cloud 的 service/user 双凭据边界与 5 分钟上限
  就被 Gateway 绕过。
- 前置状态：Cloud 只信任 Gateway 的 service 公钥（`kind=service, role=gateway`）与 user 公钥
  （`kind=user`）；浏览器持有有效 session Cookie。
- 触发：带伪造内部 header 与转发 header 的请求；不带 Cookie 但带 `Authorization` 的请求；用 Cloud 验证器
  校验 Gateway 签出的两种凭据，并互换 kind 校验；时间前进超过凭据期限。
- 必须成立：伪造 header 被剥离，请求按 session 身份成功；无 Cookie 的请求为 `401 unauthenticated`；
  service JWT `role=gateway`、user JWT `caller` 等于 service `sub`、`source`/`sub`/`displayName` 来自
  session；两种凭据不能互换用途；期限等于配置值且过期后 Cloud 拒绝；一把私钥或重复 `kid` 在启动时被拒。
- 禁止结果：任何浏览器 header 到达 Cloud；凭据期限超过 5 分钟；service 与 user 共用私钥。
- 决策依据：ADR D1、D7，不变量 11、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 伪造 `Authorization`/`X-Ora-User-Token`/`X-Forwarded-*` 不影响结果；无 Cookie 为 `401` | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| Cloud 验证器接受 Gateway 签发的两种凭据，caller 绑定成立，kind 不可互换，过期后拒绝 | `gateway.TestIssuedCredentialsSatisfyCloudVerification` | `Covered` |
| 同一私钥、重复 `kid`、超过 5 分钟的期限或空 service subject 使 Issuer 构造失败 | `gateway.TestNewIssuerRejectsSharedKeysAndLongLifetimes`、`gateway.TestConfigDefaultsAndSessionLifetimeBounds` | `Covered` |
| 私钥只接受 PKCS#8 Ed25519 PEM | `gateway.TestLoadPrivateKeyAcceptsOnlyPKCS8Ed25519` | `Covered` |
| 内部 JWT 不出现在 session 表、Cookie、重定向、响应 body 或日志 | 没有日志捕获或响应扫描测试 | `Missing` |

## Internal Routes And Cross-Site Mutations Never Reach Cloud

- 风险：Gateway 若代理 `/internal/v1/*`，Controller 内部契约就暴露给浏览器；改变状态的请求若不要求
  同源证明，持有 session Cookie 的浏览器会被第三方页面驱动执行 mutation。
- 前置状态：浏览器持有有效 session Cookie。
- 触发：`POST /internal/v1/access`；不带 `Origin`、带外来 `Origin`、只带 `Sec-Fetch-Site: same-origin`
  的 `POST /api/v1/...`；`GET /api/v1/...` 不带任何同源 header。
- 必须成立：内部路由为 `404 not_found`；无同源证明或外来 origin 的 mutation 为 `403 origin_forbidden`
  且不到达 Cloud；`Sec-Fetch-Site: same-origin` 被接受；`GET` 无需同源证明。
- 禁止结果：内部 API 响应经 Gateway 返回；跨站 mutation 到达 Cloud；同源 mutation 被误拒。
- 决策依据：ADR D1、D8，不变量 11。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| `/internal/v1/*` 为 `404`；跨站或无证明 mutation 为 `403`；`Sec-Fetch-Site` 替代；`GET` 直接转发 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| `Origin` 精确匹配规则（scheme、host、子域、`null`） | `gateway.TestSameOriginRequiresExactOriginOrSameOriginFetchMetadata` | `Covered` |
| 请求体与代理响应大小上限生效 | 没有针对 64 KiB 请求体与 8 MiB 响应上限的测试 | `Missing` |

## Cloud Authorization Decisions Pass Through Unchanged

- 风险：有效 session 若被 Gateway 解释为“已授权”，或 Cloud 的 401/403 被改写，disabled user 与被移除的
  membership 就能继续访问。
- 前置状态：浏览器持有有效 session；Cloud 中该用户随后被置为 `disabled`。
- 触发：经 Gateway 请求 `/api/v1/me`。
- 必须成立：Cloud 基于当前数据库状态返回 `403 user_disabled`，Gateway 原样透传；session 行不被吊销。
- 禁止结果：Gateway 缓存授权结果；把 Cloud 拒绝改写为成功或吊销 session。
- 决策依据：ADR D1，不变量 15。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| disabled user 经 Gateway 得到 Cloud 的 `403 user_disabled` | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| membership 移除或 owner 不匹配的拒绝透传 | 只有 disabled user 路径，membership/owner 路径由 Cloud 自身测试覆盖但未经 Gateway 验证 | `Partial` |

## Cloud Outage Is A Stable Failure That Keeps Sessions

- 风险：Cloud 不可达时若 Gateway 吊销 session、返回不稳定错误或泄露上游细节，用户会在 Cloud 恢复后被
  迫重新登录，或看到内部信息。
- 前置状态：浏览器持有有效 session；Gateway 的 upstream 不可达。
- 触发：经该 Gateway 请求 `/api/v1/me`；再用同一 Cookie 经 upstream 正常的 Gateway 请求。
- 必须成立：故障 Gateway 返回 `502 upstream_unavailable`；同一 session 在健康 Gateway 上仍然 `200`。
- 禁止结果：session 被吊销或清除；上游错误文本进入响应。
- 决策依据：ADR D8，失败与恢复语义表。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| upstream 拒绝连接时返回 `502 upstream_unavailable`，session 在其他副本仍有效 | `integration.TestGatewayProviderAndUpstreamFailuresKeepFactsStraight` | `Covered` |
| Gateway 签名失败返回 `503 credential_unavailable` 且不发请求给 Cloud | 没有签名失败的故障注入 | `Missing` |
