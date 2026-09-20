---
status: proposed
---

# 外部身份由 Gateway 归一，Cloud 保持授权权威

Ora 的公开认证入口由独立 Gateway 承担。Gateway 通过可插拔的 Authenticator 验证外部身份，把结果归一为
稳定的 `source`、`subject` 和非权威的 `displayName`，再使用现有的短期内部 service/user JWT 调用 Cloud。
Cloud 不直接接入 GitHub、密码或其他身份协议，也不持有浏览器会话；它继续以 PostgreSQL 中的
`users`、`user_identities`、membership 和资源 owner 关系作为用户与授权的唯一权威。

首个落地适配器使用 GitHub OAuth Authorization Code Flow。浏览器会话由 Gateway 存入 PostgreSQL，所有
会话都有非空的绝对过期时间；MVP 默认有效 30 天，配置值不得超过 90 天，且会话始终可以提前退出或吊销。
外部 provider token、OAuth code、PKCE verifier 和浏览器 session token 明文均不持久化，也不进入 Cloud。

本文处于 `proposed`，用于评审认证所有权、持久化、安全与扩展边界；不表示接口、表结构或部署已经上线。
本次决策只覆盖后端，不定义前端页面、UI 路由和交互样式，也不改变 Cloud 已发布的公开 API、用户 JIT
创建语义或现有 tenant/owner 授权规则。

## 问题与现有边界

Cloud 已经接受两类相互独立的内部凭据：`Authorization` 中的 service JWT 证明调用服务，
`X-Ora-User-Token` 中的 user JWT 证明最终用户；公开 API 只允许 `role=gateway` 的服务调用，并要求
`user.caller == service.sub`。内部 JWT 的有效期有严格上限，不能被改造成浏览器长期会话。

Cloud 的 `user_identities(source, subject)` 已经表达“外部身份映射到 Ora 用户”的关系，并通过联合唯一约束
保证并发首次登录只创建一个用户。这个模型刻意不依赖协议名、用户名、展示名或 email，但当前系统没有
负责完成外部登录、保存浏览器会话、签发内部 JWT 和安全转发公开 API 的生产 Gateway。

因此需要同时解决以下问题：

- GitHub OAuth 的 redirect、callback、`state`、PKCE、code exchange 和身份读取由谁负责；
- 浏览器如何保持登录，而不延长或暴露 Cloud 的内部 JWT；
- 会话如何在 PostgreSQL 中保存、过期、吊销和清理，并确保 Cookie 不超过服务端绝对期限；
- 后续增加其他 OAuth/OIDC、企业 SSO 或 passkey 时，如何避免复制会话和 Cloud 转发逻辑；
- Gateway、Cloud、外部 provider、浏览器和 PostgreSQL 之间各自信任什么，失败时保留什么事实。

## 术语与信任边界

- **Authenticator**：验证一种外部认证证明并返回 `VerifiedIdentity` 的适配器。只有适配器能解释 provider
  code、token、issuer 和 provider 用户响应；它不能授予 Ora tenant、role 或 owner 权限。
- **VerifiedIdentity**：Gateway 内部的短生命周期值，至少包含稳定的 `source`、稳定的 `subject` 和可选
  `displayName`。它表示外部认证已成功，不表示 Cloud 用户启用、membership 有效或资源可访问。
- **Login Attempt**：一次有时限、一次性的外部登录尝试，用于绑定 provider、`state`、PKCE、浏览器和
  same-origin `return_to`。
- **Browser Session**：Gateway 接受浏览器请求时使用的长期、不透明凭据。其权威状态保存在 PostgreSQL，
  与内部 JWT 的生命周期分离。
- **Internal Credentials**：Gateway 为一次或一小段上游调用签发的 service JWT 和 user JWT。Cloud 只信任
  经过现有密钥、issuer、audience、kind、role 和 caller 校验的内部凭据。
- **Absolute Session Lifetime**：从 session 创建到 `expires_at` 的固定期限。MVP 默认 30 天且最多 90 天，
  请求活动不会滑动延长该时间；session 也会因提前退出、管理员吊销、Cookie 丢失或浏览器策略而失效。

## 决策

### D1：Gateway 是公开认证边界，Cloud 不直接实现外部认证协议

新增独立的 Gateway 进程作为浏览器可访问的认证与反向代理入口，即 RFC 10017 §6.1 描述的
Backend-For-Frontend 模式：浏览器不持有任何 OAuth 或内部凭据，所有 provider 交互和 Cloud 请求都经由
Gateway。建议在 Cloud 代码仓库中使用 `cmd/gateway` 只做配置、依赖装配、生命周期与退出行为，认证编排、
provider 适配、会话策略和 HTTP 边界放在独立的 `internal/` 包中。Gateway 可以与 Cloud 一起部署，也可以
独立扩缩容；进程边界不能因同仓库部署而消失。

Gateway 只承担以下职责：

1. 发起和完成外部登录；
2. 创建、解析、吊销浏览器会话；
3. 为已通过会话校验的请求签发短期内部凭据；
4. 移除浏览器提交的内部认证 header，并把允许的 `/api/v1/*` 请求转发给固定配置的 Cloud upstream；
5. 对修改状态的浏览器请求执行 same-origin/CSRF 校验，并返回稳定、脱敏的认证错误。

Gateway 不得转发 `/internal/v1/*`，不得接受请求参数选择任意 upstream，也不得把客户端提供的
`Authorization` 或 `X-Ora-User-Token` 原样传给 Cloud。Cloud 继续负责用户启用状态、identity 映射、JIT
用户创建、tenant membership、role、owner 和所有业务访问判定；有效 Gateway session 不能绕过 Cloud
拒绝一个 disabled user 或无权限用户。

生产环境可以在受信 ingress 终止 TLS，Gateway 本身不必直接监听 HTTPS，但浏览器看到的公开 origin 和
OAuth callback 必须是 HTTPS。Gateway 使用显式配置的 public base URL 和 callback URL，不能从不受信的
`Host`、`Forwarded` 或 `X-Forwarded-*` header 推导 redirect URI。仅本地开发可以在 provider 允许时使用
loopback HTTP，并启用明确的 development 配置；生产配置不得降级 Cookie 和 origin 校验。

### D2：认证方式通过 `VerifiedIdentity` 收敛，会话与 Cloud 转发保持 provider-neutral

Gateway 的认证编排依赖小型 Authenticator 接口，而不是依赖 GitHub SDK 类型。每个适配器只负责生成
provider-specific 授权请求、交换 provider 凭据和读取外部身份；编排层向它提供已生成的 `state`、PKCE
参数和固定 callback URL。适配器成功后只能返回统一结果：

```text
VerifiedIdentity {
  source: string
  subject: string
  displayName?: string
}
```

登录编排统一拥有 Login Attempt、`state`、PKCE、`return_to`、失败映射和 session 创建；Session Service
只接受已经验证的 `VerifiedIdentity`，不解释 OAuth/OIDC/provider token；Cloud Proxy 只接受已解析的
Session，不知道身份来自 GitHub 还是未来的其他方式。HTTP handler 只做输入限制与协议翻译，不直接执行
SQL 或拼装 JWT claim。

增加新的认证方式时，只能新增一个产生 `VerifiedIdentity` 的适配器及其配置，不复制 session 表、Cookie
策略、内部 JWT 签发或 Cloud proxy。若未来认证方式需要账户关联、MFA、组织准入或高风险操作的再次认证，
必须另行决策；Authenticator 不能通过伪造新的 `source/subject` 自动合并已有 Ora 用户。

### D3：GitHub OAuth 是首个适配器，并使用 GitHub 数字用户 ID 作为稳定 subject

MVP 使用 GitHub OAuth App 的 Authorization Code Flow，并启用随机 `state` 与 PKCE `S256`。GitHub 在
authorize 请求携带 `code_challenge` 后要求 token exchange 携带匹配的 `code_verifier`，因此 PKCE 在
GitHub 上是被 provider 强制校验的保护，不是可被静默忽略的装饰参数。Gateway 只请求读取基本身份所需的
最小 scope，不请求 email、repository 或组织权限。callback 使用 code 换取短期可用的 GitHub access
token，调用 GitHub authenticated-user API 读取身份后立即丢弃该 token。

GitHub 身份归一规则固定为：

| 字段 | 取值 | 理由 |
| --- | --- | --- |
| `source` | `github.com` | 为 provider 身份空间提供稳定、可读的命名；不使用 OAuth App client ID。以 host 命名意味着未来的 GitHub Enterprise Server 实例是另一个 `source`，其数字 ID 不会与 `github.com` 混淆 |
| `subject` | GitHub 响应中数字 `id` 的十进制字符串 | 数字 ID 不随 login 改名；login、name 或 email 都不能作为身份键 |
| `displayName` | 非空 `name`，否则使用 `login`；规范化后不超过 200 个字符，超出按 Unicode 字符截断 | 只用于展示，可以变化，不能参与查找、合并或授权 |

字段长度以 Cloud 现有 identity contract 为准：`source` 1–128、`subject` 1–512、`displayName` 0–200
（`internal/core/store.go` 的 `identity()` 对超出值返回 `401 invalid_identity`）。GitHub 的 `name` 最长
255 个字符，因此 Gateway 必须在产生 `VerifiedIdentity` 前完成截断，而不是把长度错误留给 Cloud 变成
用户不可理解的未认证响应。Gateway 必须校验 GitHub 响应完整性和字段长度；缺失或非法的稳定 ID 使登录
失败。GitHub token、OAuth code、完整 provider 响应和 email 不写入数据库、日志、错误、JWT 或浏览器。
若未来需要 GitHub 组织准入，它是独立的授权策略，不能把组织成员关系塞进 `source` 或 `subject`。

### D4：Login Attempt 使用 PostgreSQL 保存一次性、短期的登录关联事实

登录由浏览器向 Gateway 发起一次 `POST` start 请求触发，该请求与其他状态改变请求一样接受 D8 的
`Origin` 校验，并受限流约束；不提供可被 `<img>`、预取或第三方页面顶层导航触发的 `GET` start。原因是
start 会无条件写入一行 attempt：没有 Origin 校验与限流，任何页面都能让受害者浏览器或脚本无界灌入
attempt 表，也能替受害者发起登录 CSRF。

Gateway 为每次登录生成至少 256 bit 的随机 attempt secret 和独立随机 `state`。浏览器只在 attempt Cookie
中持有 attempt secret；PostgreSQL 只保存它与 `state` 的 SHA-256 digest。PKCE verifier 由 attempt secret
和仅 Gateway 可读的密钥通过带域分离的 HMAC 确定性派生，数据库不保存 verifier 明文。

attempt Cookie 使用 `HttpOnly`、`SameSite=Lax`、`Path` 限定为 callback 路径、`Max-Age` 等于 attempt
有效期，生产 HTTPS 下必须 `Secure`。这里必须是 `Lax` 而不是 `Strict`：provider 回跳是来自 `github.com`
的跨站顶层 `GET` 导航，`Strict` Cookie 不会随之发送，callback 将永远无法匹配 attempt。限定 `Path` 使
该 secret 不随普通业务请求或代理请求发送。callback 无论成功还是失败都必须清除 attempt Cookie。

Login Attempt 至少保存以下字段：

| 字段 | 约束与语义 |
| --- | --- |
| `id` | Gateway 生成的 UUID 主键 |
| `secret_hash` | 32-byte digest，唯一 |
| `provider` | 受配置注册表限制的适配器名称 |
| `state_hash` | 32-byte digest，唯一 |
| `return_to` | 已验证的 same-origin 相对路径：以单个 `/` 开头且第二个字符不是 `/` 或 `\`，不允许 scheme、host、protocol-relative URL 或反斜杠伪装的 host |
| `created_at` / `expires_at` | `timestamptz`；有效性使用 PostgreSQL 权威时间判定 |
| `consumed_at` | `timestamptz NULL`；非空后永远不能再次创建 session |

callback 必须同时匹配 attempt Cookie、`state`、provider、未过期和未消费状态。对 GitHub 的 HTTP 调用不得
发生在数据库事务或行锁内。provider exchange 成功后，Gateway 开启短事务锁定 attempt，再次验证其未
消费，并在同一事务中写入 `consumed_at` 与新 session；这两个写入只能一起提交。

若 provider 已消费 code 但数据库事务失败，本次登录明确失败，用户重新发起登录；系统不得推断 session
已经创建。重复、过期、已消费、错误 state 或错误 attempt secret 对外使用同一种安全失败语义，避免暴露
attempt 是否存在。Login Attempt 的有效期必须是分钟级配置，并在短期审计窗口后有界清理；attempt 表的
行数由有效期、有界清理和 start 限流共同约束，不存在无界增长路径。

### D5：Browser Session 存在 PostgreSQL，数据库只保存不透明 token 的 digest

Gateway Session Service 使用 PostgreSQL 作为唯一权威存储，不提供内存、SQLite 或仅 Cookie 签名的并行
实现。创建 session 时生成至少 256 bit 的随机 token，将明文只返回一次写入 HttpOnly Cookie，数据库只
保存 SHA-256 digest。每次请求对收到的 token 计算 digest，并按唯一索引解析 session。

`gateway_sessions` 至少保存以下字段：

| 字段 | 约束与语义 |
| --- | --- |
| `id` | Gateway 生成的 UUID 主键，用于审计，不作为浏览器凭据 |
| `token_hash` | 32-byte digest，唯一；数据库永远没有原始 token |
| `source` / `subject` | 归一后的稳定外部身份，字段长度与 Cloud identity contract 一致 |
| `display_name` | 可空、不超过 200 个字符的非权威快照，与 D3 的规范化结果和 Cloud 上限一致 |
| `created_at` | PostgreSQL 权威时间 |
| `expires_at` | `timestamptz NOT NULL`；必须晚于 `created_at`，表示不可滑动延长的绝对期限 |
| `revoked_at` / `revoked_reason` | 可空；一旦吊销不可恢复，reason 使用稳定、脱敏的有限值 |

同一个 `source/subject` 可以拥有多个并发 session，以支持多设备和分别退出。Session 不外键关联 Cloud
`users`：外部身份可能在首次 `/api/v1/me` 前已登录，且 Gateway 不能读取或写入 Cloud 业务表。身份是否
存在、用户是否 disabled 及 membership 是否有效，仍由每次 Cloud 请求基于当前业务状态判断。

Session Store 在消费边界至少提供 Create、Resolve、Revoke 和 RevokeIdentity 能力。Create 只把 raw token
返回给调用者一次；Resolve 只有在 `revoked_at IS NULL` 且 `expires_at > clock_timestamp()` 时成功；不存在、
过期和已吊销对外统一表现为未认证。Revoke 与 RevokeIdentity 幂等。吊销保证后续新进入 Gateway 的请求
失败，不撤回已经通过 admission 并正在 Cloud 执行的请求。

MVP 的 session TTL 是正数配置，默认 30 天，硬上限 90 天；缺失配置使用默认值，零、负数或超过上限的
配置使 Gateway 启动失败。创建 session 时由 PostgreSQL 权威时间确定 `created_at` 与 `expires_at`，读取
session 不进行滑动续期。Cookie 使用 `HttpOnly`、`SameSite=Lax`、`Path=/`、无 `Domain`，在生产 HTTPS 下
必须使用 `Secure`，并使用 `__Host-` 名称前缀：浏览器对该前缀强制要求 `Secure`、`Path=/` 且无 `Domain`，
把上述属性从约定变成浏览器端强制，也阻止子域或同站其他 origin 覆盖该 Cookie。它应是持久 Cookie，且
`Expires`/`Max-Age` 不得晚于数据库 `expires_at`。浏览器可以更早清除 Cookie。退出只吊销当前 session 并
清除 Cookie；“退出所有设备”和管理员吊销可以复用 RevokeIdentity，但其公开管理 API 不在本 ADR 范围。
仅本地开发的 loopback HTTP 配置可以省略 `__Host-` 前缀与 `Secure`，且该降级必须由显式 development
配置开启。

同一 identity 的 session 数量不设上限：每次成功登录都创建新 session，重复登录只会累积直到各自到期或
被吊销。MVP 接受这一点以保持登录路径简单；数据库增长由绝对期限与 D6 的有界清理约束。

### D6：Gateway 与 Cloud 可共用 PostgreSQL 集群，但数据库权限和表所有权必须分离

MVP 可以复用 Cloud 已有 PostgreSQL 集群、连接配置方式和迁移工具，以减少新的运行依赖。新增
`gateway_login_attempts` 与 `gateway_sessions` 表必须通过 `internal/core/migrations` 下新的有序 SQL
forward migration 创建，并由单独授权的 `cloudctl migrate` 身份执行；Gateway 启动时只验证 migration
状态，绝不执行 DDL 或 AutoMigrate。

运行时使用独立的 Gateway 数据库角色，只授予上述认证表所需的 `SELECT`、`INSERT`、`UPDATE`、`DELETE`
权限。Gateway 不得查询 `users`、`user_identities`、membership、tenant、project、workspace 或 operation
表；Cloud 运行时也不依赖 Gateway session 表完成业务授权。共享集群是部署复用，不是共享数据所有权。

表约束至少在数据库层保证 digest 长度、identity 字段边界、`expires_at > created_at`、
`expires_at <= created_at + interval '90 days'`、吊销 reason 与时间的一致性，并为 token lookup、identity
revoke、attempt state lookup 和有界清理建立索引。时间字段使用 `timestamptz`；有效性、过期和清理边界
使用数据库时间，不依赖多 Gateway 实例的本地时钟。

过期或吊销 session 可以保留一个有限审计窗口后批量清理，已消费/过期 attempt 也必须有界清理。清理任务
由 Gateway 生命周期拥有，可取消、限制每批行数并允许重试；不得删除尚未过期且未吊销的活跃 session，
也不得用一次无界 `DELETE` 阻塞生产表。

### D7：有限期浏览器会话只换取短期内部凭据，不改变 Cloud 的 JWT 安全上限

每次代理已认证请求时，Gateway 先从 PostgreSQL Resolve 当前 session，再用两把按用途分离的 Ed25519
私钥签发：

- 用 service 签名密钥签发 `kind=service`、`role=gateway` 的 service JWT，其 `sub` 是稳定的 Gateway
  服务/实例身份；
- 用 user 签名密钥签发 `kind=user` 的 user JWT，其 `source`、`sub` 和 `displayName` 来自 session，
  `caller` 必须等于本次 service JWT 的 `sub`。

必须是两把密钥而不是一把：Cloud 的验证器按 `kid` 登记的 key purpose（`kind=service` 或 `kind=user`）
选择公钥，`docs/authentication.md` 中的 `gateway-service-2026` 与 `user-identity-2026` 就是这两个用途的
公钥。一把私钥同时签发两种 kind 会让 key-purpose 绑定失去意义，也无法分别轮换。

两种 JWT 的 issuer、audience、`kid`、kind、role、caller 和最长有效期继续服从 Cloud 现有验证规则，期限
保持分钟级且不得按 browser session 的较长期限签发。内部 JWT 只存在于 Gateway 到固定 Cloud upstream
的请求中，不得写入 session 表、浏览器 Cookie、重定向 URL、响应 body 或日志。

Gateway 的两把签名私钥和 PKCE 派生密钥是三个独立 secret，支持分别轮换，不存入 PostgreSQL。Cloud 只
配置对应公钥及允许的用途。Gateway 多副本共享 session 数据，但每个副本使用的 service subject 和受信
key 必须满足 Cloud caller binding；不能拿一个实例生成的 user JWT 与另一个 caller 不同的 service JWT
混用。

### D8：浏览器安全依赖 same-origin、CSRF 防护和严格 redirect，而不只依赖 Cookie 属性

Gateway 的认证路由与代理路由默认同源，不开放 credentialed cross-origin CORS。对会改变状态的浏览器
请求（包括 login start、logout 和代理的 `/api/v1/*` mutation），Gateway 必须校验 `Origin` 与配置的
public origin 完全一致，并继续要求现有的 JSON/content-type 边界；不匹配的 `Origin` 按稳定错误拒绝。
`Origin` 缺失时，`Sec-Fetch-Site: same-origin` 可作为等价的同源证据被接受；两者都缺失则拒绝。这样既
不依赖单一 header，也不放行任何没有浏览器同源证据的跨站请求。OAuth callback 是跨站顶层导航的例外，
但必须完整验证 Login Attempt、`state`、PKCE 和 callback URL。

所有 `return_to` 必须在写入 attempt 前按 D4 的规则规范化为同源相对路径，并在回调后再次从可信 attempt
读取；请求参数不能直接成为 `Location`。认证错误不得回显 provider token、code、state、attempt secret、
session token、SQL 或上游响应。结构化日志可以记录 request ID、provider、阶段、稳定错误码和 session
UUID，不能记录上述秘密、完整 Cookie 或不必要的个人信息。

login start 和 callback 是未认证入口，必须有限流：至少按客户端来源限制单位时间内的 start 次数，且
限流拒绝不写入 attempt。限流阈值是部署配置，本 ADR 只要求它存在并在被触发时返回稳定错误。

Gateway 必须设置请求体、header、callback 参数、provider 响应和代理响应的大小/时间上限，向外部 provider
发起的 HTTP 请求使用可取消 context、连接与总超时，并在所有路径关闭 response body。provider 暂时失败
不能创建 session；Cloud 暂时失败不能使有效 session 被吊销，浏览器可以在 Cloud 恢复后重试业务请求。

## 失败与恢复语义

| 失败点 | 对外结果 | 持久事实与恢复 |
| --- | --- | --- |
| 创建 attempt 失败 | 不跳转 provider | 不存在可用 attempt；可安全重试 start |
| provider 拒绝、state/PKCE 不匹配或 callback 超时 | 登录失败 | 不创建 session；attempt 过期或标记消费后清理 |
| provider exchange 成功、session 事务失败 | 登录失败 | code 可能已不可重用，但没有 session；必须重新开始登录 |
| session Cookie 不存在、未知、过期或已吊销 | 统一未认证 | 不泄露 token 是否曾存在；可重新登录 |
| Gateway 签名失败 | 请求不发给 Cloud | session 保持有效；修复密钥配置后重试 |
| Cloud 网络或 5xx 失败 | 返回稳定 upstream failure | 不吊销 session，不推断业务请求成功；业务幂等仍由 Cloud contract 决定 |
| 用户在 Cloud 被禁用或 membership 被移除 | Cloud 返回当前授权结果 | session 可以存在，但不能绕过当前业务状态；重新登录也不能恢复权限 |
| logout 与业务请求并发 | logout 幂等成功 | 已 admission 的请求可完成，后续 Resolve 必须失败 |

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| Cloud 直接实现 GitHub OAuth | 少一个部署进程和一次代理 | 把外部协议、Cookie、重定向和业务授权耦合到权威服务，后续认证方式会扩大 Cloud 攻击面并复制边界逻辑 |
| 浏览器直接持有 GitHub token 调 Cloud | Gateway 更薄 | Cloud 将被迫信任 provider token 并联网验证；token 暴露给浏览器和业务 API，无法保持现有 service/user 双凭据边界 |
| 把内部 user JWT 做成永久 token | 无需 session 查询 | 破坏 Cloud 的短期凭据上限，用户禁用、logout 和泄露处置无法及时生效，也会把内部凭据暴露给浏览器 |
| 使用纯签名/加密 Cookie，不存 session | 无数据库读取，部署简单 | 无法可靠 logout、按身份全量吊销或观察泄露影响；长期凭据只能等待密钥全局轮换 |
| 将 session 外键关联 `users` | 可在数据库中直接绑定 Ora user | Gateway 必须跨越业务表所有权，首次 JIT 创建前无法建 session，并把外部认证成功错误解释为 Cloud 授权成功 |
| 用 GitHub login 或 email 作为 `subject` | 人类可读 | login 可改名，email 可缺失、变化或复用；会错误合并或分裂 durable identity |
| 持久化 GitHub access token | 可减少下次 provider 调用 | MVP 只需要登录时读取身份；长期保存扩大泄露、加密、scope、撤销和轮换责任，且 Cloud 请求不需要它 |
| 每种 provider 单独建 session 表和 handler | 单个适配器局部直观 | 会复制安全策略、清理、Cookie、JWT 和 proxy 行为，使不同认证方式产生不一致的权限与失败语义 |
| 单独部署 Redis session | 低延迟、天然 TTL | 当前系统的唯一权威持久化是 PostgreSQL，MVP 不需要新增运维与恢复边界；PostgreSQL 已能表达有限 TTL、即时吊销和清理审计 |

## 不变量

1. 只有 Gateway 解释外部认证协议；Cloud 不接收或验证 GitHub token、OAuth code、密码或浏览器 Cookie。
2. Authenticator 成功只产生 `VerifiedIdentity`，不能授予 Ora tenant、role、membership、owner 或业务权限。
3. 外部身份只由稳定的 `(source, subject)` 查找；`displayName`、login、name 和 email 永不参与身份合并或授权。
4. GitHub 的 `subject` 必须来自数字用户 ID，`source` 必须为 `github.com`；`VerifiedIdentity` 的字段长度
   在离开 Authenticator 前已满足 Cloud 的 128/512/200 上限。
5. 原始 session token、attempt secret、OAuth code、provider access token 和 PKCE verifier 不得持久化、记录或返回给 Cloud。
6. Login Attempt 必须有时限、绑定浏览器与 provider、验证 `state`/PKCE，并且最多成功创建一个 session。
   login start 只接受经过同源校验和限流的 `POST`；attempt Cookie 只发送到 callback 路径，并在 callback
   结束时清除。
7. Browser Session 的有效性只由 PostgreSQL 当前状态和权威时间决定；未知、过期和已吊销统一为未认证。
8. 每个 session 的 `expires_at` 必须非空、晚于 `created_at` 且不超过创建时间 90 天；Resolve 不得滑动延长期限。
9. 同一 identity 可以有多个 session；session 不外键关联 Cloud user，也不证明该用户当前可访问任何资源。
10. Gateway 运行时只访问自己拥有的认证表，不读写 Cloud 业务表，不在启动时执行 DDL。
11. Gateway 必须移除浏览器提供的内部认证 header，只向固定 Cloud upstream 代理允许的公开 API，绝不代理内部 API。
12. Gateway 到 Cloud 的 service/user JWT 始终短期有效并满足现有 issuer、audience、kind、role、caller 与 key-purpose 校验；
    service JWT 与 user JWT 由两把按 kind 分离的私钥签发。
13. 生产公开 origin 和 OAuth callback 必须使用 HTTPS；redirect URI 来自显式配置，不能由不受信请求 header 决定。
14. 外部 HTTP 调用不得发生在数据库事务或行锁内；ambiguous failure 不得被推断为登录或业务操作成功。
15. 有效 session 不能覆盖 Cloud 对 disabled user、membership、tenant 和 owner 的当前授权判定。

## 验收场景

- 首次 GitHub 登录用数字 ID 建立 `github.com/<id>` identity；GitHub login 或 name 改变后再次登录仍映射到
  同一个 Cloud user，user JWT 只携带最新的非权威展示名。Cloud 现有 JIT 语义只在创建用户时写入
  `users.display_name`，不因后续登录更新它；是否同步展示名是独立的 Cloud 决策，不属于本 ADR。
- GitHub `name` 超过 200 个字符时，Gateway 截断后登录成功，Cloud 不会返回 `invalid_identity`。
- 两个并发 callback 使用同一个 attempt 时，最多一个事务消费 attempt 并创建 session；另一个返回统一
  登录失败，数据库没有第二个有效 session。
- session 表只出现 token digest；使用数据库内容、日志或 Cloud 请求都无法还原浏览器 session token 或
  GitHub access token。
- 未过期 session 在 Gateway 重启和多副本切换后仍可 Resolve；到达数据库 `expires_at` 或 logout 后，所有
  副本的后续请求都拒绝该 token，访问活动不能延长绝对期限。
- 一个 identity 的两个设备 session 可以独立使用；退出设备 A 不影响设备 B，RevokeIdentity 后两者都不能
  新进入 Cloud。
- 浏览器伪造 `Authorization`、`X-Ora-User-Token`、upstream、caller、source 或 subject 时，Gateway 丢弃或
  拒绝这些输入，Cloud 只看到 Gateway 根据 PostgreSQL session 生成的内部凭据。
- session 仍有效但 Cloud user 已 disabled、membership 已删除或 owner 不匹配时，Cloud 基于当前数据库
  状态拒绝访问；Gateway 不把这类响应改写为认证成功。
- 错误 state、错误 attempt Cookie、过期 attempt、重复 callback、绝对或 `//`、`/\` 开头的 `return_to`、
  跨源 mutation、`GET` start、无同源证据的 start 和非 HTTPS 生产配置都在产生 session 或调用 Cloud 前
  失败。
- 超过限流阈值的 start 请求返回稳定错误且不写入 attempt 行；同一浏览器完成 callback 后 attempt Cookie
  已被清除，重放同一 callback URL 得到统一登录失败。
- provider 超时发生在数据库事务之外；Cloud 5xx 不吊销 session；数据库 commit 失败后即使 provider code
  已消费也不会出现推断创建的 session。
- migration 由 `cloudctl migrate` 执行；Gateway 的受限数据库角色无法执行 DDL 或读取 Cloud 业务表。

## 风险与代价

- 每个已认证请求增加一次 PostgreSQL session lookup。MVP 接受这项成本以换取即时 logout 和吊销；只有在
  保持数据库为权威、证明失效上限并另行决策后才能增加缓存。
- 30 天默认绝对期限仍给 Cookie 被盗留下较长暴露窗口。HttpOnly、Secure、same-origin/CSRF、digest-only
  存储、90 天硬上限和吊销降低但不能消除风险；正式支持高价值操作前应增加空闲期限、再认证和会话管理。
- Gateway 成为公开可用性的关键路径，需要独立 readiness、限流、超时、日志脱敏和密钥轮换演练。共享
  PostgreSQL 集群降低运维成本，但必须通过数据库角色防止横向读取业务数据。
- 同一 identity 的 session 数量无上限，反复登录只增不减，直到各自到期或被吊销。表体积由 30/90 天
  期限、start 限流和有界清理约束；若观察到异常累积，按 identity 限制并发 session 数是后续决策。
- GitHub 是首个外部可用性依赖。GitHub 故障会阻止新登录，但不得使已有 session 或 Cloud 内部授权失效。

## 本决策未解决的问题

- 前端登录页面、按钮、回调展示、路由恢复和用户会话管理 UI。
- 账户主动关联/解绑、多个 external identity 合并、误绑定恢复和 identity 管理审计。
- GitHub organization/team 准入、企业 OIDC/SAML、MFA、passkey、step-up authentication 和风险评分。
- 管理员 session 列表与吊销 API、用户“退出所有设备”公开接口和完整安全事件审计产品。
- session token 自动轮换、空闲过期、续期策略、设备信息以及高价值操作的重新认证策略。
- ingress、域名、证书和 secret manager 的具体产品选择；本文只固定公开 HTTPS 与可信配置边界。

## 落地顺序

1. **冻结契约**：评审本文并确认 Gateway/Cloud 所有权、GitHub identity 映射、30/90 天 session 期限和安全
   不变量。状态批准后，在镜像目录补充核心测试用例与预期证据。
2. **持久化基础**：新增 forward migration、数据库约束、最小权限角色说明和 PostgreSQL-backed Login
   Attempt/Session Store；验证 fresh database 与上一 schema 升级、并发消费、吊销和有界清理。
3. **认证模块**：实现 provider-neutral 编排与 GitHub 适配器，以 fake provider 覆盖状态机，再用受控边界
   测试验证真实 HTTP/timeout/cancellation 行为；此阶段仍不改变 Cloud API。
4. **Gateway 边界**：新增命令、认证路由、Cookie/CSRF/redirect 防护、内部 JWT 签发和 allowlisted proxy；
   验证伪造 header、内部路由、错误 upstream 和敏感数据不会越界。
5. **端到端接入**：配置 HTTPS ingress、GitHub OAuth App、Gateway key 与 Cloud trusted public key，验证登录、
   JIT user、disabled user、membership removal、logout、重启和多副本行为后再开放前端入口。

完成上述阶段不会改变 Cloud 现有公开 API、`user_identities` 唯一性、短期 JWT 上限或 tenant/owner 授权；
变化只发生在 Cloud 之前新增一个生产级外部认证与浏览器会话边界。

## 参考

- Cloud 当前认证契约：`docs/authentication.md`
- Cloud identity schema：`internal/core/migrations/0001_core.sql`
- OAuth 2.0 Security Best Current Practice：[RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)
- OAuth 2.0 for Browser-Based Applications：[RFC 10017](https://www.rfc-editor.org/rfc/rfc10017)
- GitHub OAuth web application flow：<https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps>
