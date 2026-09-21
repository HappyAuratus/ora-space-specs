# 浏览器会话

## Session Validity Is Decided Only By PostgreSQL State And Time

- 风险：若某个副本用本地时钟、内存缓存或 Cookie 自身内容判断 session 有效，多副本会对同一 token 得出
  不同结论，过期或吊销无法在所有副本同时生效；若读取续期，绝对期限会被活跃访问无限延长。
- 前置状态：两个 Gateway 副本共享同一 PostgreSQL schema 与 Cloud；一个浏览器在副本 A 完成登录。
- 触发：用同一 Cookie 访问副本 B；多次读取后比较 `expires_at`；把 `expires_at` 改到过去后再访问两个副本。
- 必须成立：副本 B 直接解析副本 A 创建的 session；数据库只保存 token 的 SHA-256 digest；Cookie 的
  `Max-Age` 不晚于数据库 `expires_at`；多次读取后 `expires_at` 不变；到期后两个副本都返回
  `401 unauthenticated`。
- 禁止结果：原始 token 出现在数据库；任一副本在数据库判定过期后仍接受 token；读取滑动延长期限。
- 决策依据：ADR D5、D6，不变量 5、7、8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 副本 B 解析副本 A 的 session | `integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| 数据库只出现 token digest，原始 token 片段不出现 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| 多次访问后 `expires_at` 不变；到期后所有副本拒绝 | `integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| Cookie `Max-Age` 等于数据库期限且不为负 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries`、`gateway.TestCookiePolicyEnforcesPrefixesScopeAndDatabaseBoundedMaxAge` | `Covered` |
| session TTL 配置零、负数或超过 90 天时启动失败；GitHub 缺省 30 天，IDaaS 未显式配置时缺省 12 小时 | `gateway.TestConfigDefaultsAndSessionLifetimeBounds`、`gateway.TestIDaaSConfigIsSelectedIndependentlyAndDefaultsToTwelveHours`、`gateway.TestLoadConfigReadsFileAndEnvironmentOverrides`、`integration.TestGatewayIDaaSLoginIdentityAndSessionLifetime` | `Covered` |
| 生产 `__Host-` 前缀与 `Secure`，开发 loopback 才允许降级 | `gateway.TestCookiePolicyEnforcesPrefixesScopeAndDatabaseBoundedMaxAge`、`gateway.TestPublicOriginRequiresHTTPSOutsideLoopbackDevelopment` | `Covered` |

## Revocation Is Immediate On Every Replica And Idempotent

- 风险：logout 或按 identity 吊销若只在发起副本生效、不幂等、或泄露 token 是否存在，就无法用于泄露处置；
  设备间吊销若互相影响，会让单设备退出变成全体退出。
- 前置状态：同一 identity 在两个设备各有一个 session；两个副本共享数据库。
- 触发：设备 A 在副本 A logout；设备 A 的 Cookie 访问副本 B；设备 B 继续访问；对 identity 调用
  `RevokeIdentity` 两次；再次 logout。
- 必须成立：logout 需要同源证明并返回 `204`；此后该 token 在所有副本都是 `401`；设备 B 不受影响；
  `RevokeIdentity` 第一次吊销全部活跃 session、第二次返回 0 且不报错；吊销行带 `revoked_at` 与有限
  `revoked_reason`；已过期的行不获得吊销标记。
- 禁止结果：跨站 logout 成功；吊销只在一个副本生效；重复吊销报错或改写已有吊销事实；未知 token 的
  logout 与已知的返回不同。
- 决策依据：ADR D5、D8，不变量 7、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| logout 无同源证明被拒；成功后同一 token 在另一副本被拒；再次 logout 幂等 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries`、`integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| 设备 A 退出不影响设备 B | `integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| `RevokeIdentity` 吊销全部活跃 session 且幂等，只计活跃行 | `integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| 吊销行持久化 `revoked_reason=logout` | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries` | `Covered` |
| 已过期 session 的 logout 不写入吊销标记 | 当前测试未直接对过期行调用 logout 并检查 `revoked_at` | `Missing` |

## IDaaS Logout Revokes Only The Local Cloud Session

- 风险：Cloud logout 若调用 IDaaS/W3 全局退出，会意外结束同一浏览器中的其他华为系统；若只清 Cookie 而
  不吊销 PostgreSQL session，复制的 Cookie 仍可在其他 Gateway 副本使用。
- 前置状态：用户通过 IDaaS/W3 建立一个有效的 12 小时 Cloud Browser Session。
- 触发：同源 `POST /auth/logout`，随后再次访问 Cloud 页面。
- 必须成立：Gateway 在 PostgreSQL 中吊销当前 session、清除本地 Cookie，所有副本立即拒绝旧 token；不调用
  IDaaS logout、revoke 或 refresh。前端清除当前用户查询缓存后再次进入自动登录；若 W3 SSO 仍有效，IDaaS
  可以无感完成新一次授权，但必须创建全新的 Cloud session。
- 禁止结果：logout 只清浏览器 Cookie；保存/刷新 provider token；Cloud 退出影响其他 W3 应用。
- 决策依据：ADR D5、D8，不变量 5、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| logout 吊销数据库 session、跨副本立即生效且幂等 | `integration.TestGatewayLoginProxyLogoutAndCredentialBoundaries`、`integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| 前端 logout 调用真实 `/auth/logout` 并返回自动登录页，不保存本地 token | `app-sidebar.test.tsx`、`login-page.test.tsx`、`api-client.test.ts` | `Covered` |
| Gateway 不调用 IDaaS logout/revoke/refresh | adapter 未实现这些端点；尚无“无外部调用”的显式 spy 断言 | `Partial` |

## Cleanup Is Bounded And Preserves Live Rows

- 风险：无界 `DELETE` 会长时间锁住生产表；清理误删活跃 session 会把用户踢出；不清理则 attempt 与
  session 无限增长。
- 前置状态：数据库中混有活跃、已过期、已吊销的 session 与过期/未过期的 attempt。
- 触发：以不同 `retention` 与 `batch` 多次调用 Cleanup，直到返回 0。
- 必须成立：在保留窗口内的行不被删除；每次调用每表最多删除 `batch` 行；重复调用直至排空；活跃 session 与
  未过期 attempt 始终保留。
- 禁止结果：单次删除超过批上限；活跃行被删除；清理错误中断清理循环。
- 决策依据：ADR D6。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 保留窗口内不删除；每表每批不超过上限；循环排空后只剩活跃 session 与未过期 attempt | `integration.TestGatewaySessionsAcrossReplicasExpiryRevocationAndCleanup` | `Covered` |
| 清理 goroutine 由生命周期拥有：取消后退出，失败只记录并在下一轮重试 | `RunCleanup` 没有直接测试 | `Missing` |

## Schema Constraints Reject Invalid Session And Attempt Rows

- 风险：应用层 bug 或误操作写入超过 90 天的 session、非定长 digest、无 reason 的吊销或绝对 `return_to`，
  会让数据库承载与 ADR 相悖的事实；升级路径若破坏既有数据，会阻断部署。
- 前置状态：schema 已迁移到 `0005_gateway_auth.sql`；或数据库处于 `0003` 并带有业务数据。
- 触发：直接插入各类非法行；从 `0003` 运行 `Migrate` 并 `CheckSchema`。
- 必须成立：超过 90 天、早于创建、短 digest、无 reason 的吊销、未知 reason、超长 `source`/`display_name`、
  超过 1 小时的 attempt、绝对/`//`/`/\` 的 `return_to` 均被约束拒绝；`token_hash` 唯一；恰好 90 天被接受；
  升级后 `CheckSchema` 通过且既有数据保留。
- 禁止结果：任一非法行被接受；迁移不可重复执行；升级改写既有业务行。
- 决策依据：ADR D5、D6，不变量 8。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 上述每类非法行被拒绝，恰好 90 天被接受，`token_hash` 唯一 | `integration.TestGatewaySchemaConstraints` | `Covered` |
| 从 `0003` 升级到最新 schema 后 `CheckSchema` 通过且既有数据保留；`Migrate` 可重复执行 | `integration.TestMigrateUpgradesPreviousSchemaAndData`、`integration.setup`（两次 `Migrate`） | `Covered` |
| Gateway 运行时角色无法执行 DDL 或读取 Cloud 业务表 | 属于部署配置，仓库内没有角色权限测试 | `Missing` |
