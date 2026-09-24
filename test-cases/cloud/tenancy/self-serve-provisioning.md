# 自助租户创建

## A Signed-In Member Provisions Exactly One Tenant And Its First Space

- 风险：首次登录的用户若没有不依赖运维的路径拿到第一个空间，产品无法自助起步；若创建路径不原子或授权
  前提错误，会出现无管理员的租户、无唯一空间的租户，或未验证身份建租户。
- 前置状态：用户经 Gateway 验证身份（`(source, subject)` JIT 建立 user），不属于任何租户；数据库无该
  `Idempotency-Key` 的记录。
- 触发：`POST /api/v1/tenants {name, slug}`（合法与非法输入、缺失 key）；随后 `GET /api/v1/me/tenants`。
- 必须成立：合法请求在一个事务里创建租户、`admin` membership 与唯一空间，响应
  `201 {tenant, space}`；非法 name/slug 或缺失 key 为 400 且不写任何行；停用用户为 `403 user_disabled`；
  成员可见列表包含新租户；`cloudctl bootstrap` 与该路径共用同一事务形态。
- 禁止结果：只写了一部分行的"半租户"；未验证身份或停用身份创建租户；新租户对其他用户可见。
- 决策依据：ADR D1、D2，不变量 1、2、4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 一个事务创建租户+admin+唯一空间；非法输入与缺失 key 在写库前拒绝且不产生任何行 | `integration.TestSelfServeTenantProvisioning` | `Covered` |
| 经 Gateway 的完整 HTTP 路径（真实 Cloud router + PostgreSQL）创建租户并进入空间 | `integration.TestGatewayRelaysEventStreamsBeyondTimeouts`（租户创建后订阅事件流） | `Partial` |

## Idempotency Key Matches Per User Across Tenants

- 风险：该接口在租户存在之前执行，`idempotency_records` 的主键要求租户已存在；若按不存在的租户查找，
  重放会绕过幂等；若重放写入第二个租户，同一请求产生两份资源。
- 前置状态：用户已成功以某 key 创建过一个租户；数据库保存该 `(user_id, key)` 记录。
- 触发：同 key 同请求体重放；同 key 不同请求体；无记录的新 key。
- 必须成立：同 key 同体重放原样返回首次 `201` 响应，不创建第二个租户；同 key 不同体为
  `409 idempotency_conflict`；新 key 正常创建，记录挂在新建租户名下。
- 禁止结果：重放创建第二个租户；hash 不同仍重放；冲突时部分写入。
- 决策依据：ADR D3，不变量 3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 同 key 同体重放、同 key 异体 409、重放/冲突不产生第二租户 | `integration.TestSelfServeTenantProvisioning` | `Covered` |

## The Member's Tenant List Pages In Ascending Creation Order

- 风险：兼容接口若分页错乱会跳过或重复租户；空间切换器则必须使用新的 `/me/spaces` 全量列表。
- 前置状态：同一成员先后创建（或加入）多个租户。
- 触发：`GET /api/v1/me/tenants`（全量与 `limit`/`after` 分页）。
- 必须成立：按 `created_at` 升序（`id` 为决胜负列）返回；`after` 是排他
  租户 UUID cursor，翻页不重不漏；对外契约（cursor 形态、`limit` 1–100）不变。
- 禁止结果：顺序随 UUID 随机；翻页跳过或重复行；cursor 暴露 `created_at`。
- 决策依据：ADR D5，不变量 4。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 多租户成员按创建时间升序列出，最早者在首位；分页 cursor 逐页推进不重不漏 | `integration.TestSelfServeTenantProvisioning` | `Covered` |
