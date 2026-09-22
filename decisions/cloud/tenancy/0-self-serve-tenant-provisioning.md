---
status: implemented
---

# 已登录用户自助创建租户，租户成为协作空间的隐式容器

Cloud 新增公开接口 `POST /api/v1/tenants`：任何已通过 Gateway 验证身份的用户都可以为自己创建一个租户。
请求体只描述第一个协作空间（`name`、`slug`）；租户借用该名称，调用者在同一事务里成为租户 `admin` 与该
空间 `owner`。产品界面不展示租户，用户只看见并管理协作空间（产品术语 Workspace）；租户继续作为隔离、
成员与凭据引用的边界存在于后端。`cloudctl bootstrap` 保留为部署路径，两条路径共用一个事务形态。

本决策已实现并验证：`provisionTenant`/`createTenant`、路由与 OpenAPI、前端 `/onboarding` 页面已随
ora-space/cloud#14 合入 main（含集成测试、race、`task frontend:check` 与真机端到端验证）；核心测试用例登记于
`test-cases/cloud/tenancy/self-serve-provisioning.md`。本决策不改变已发布的 `/api/v1/tenants/{tid}/*`
契约、`user_identities` 的 JIT 语义、tenant/space 的角色模型，也不承担合并或拆分既有租户的迁移义务。
涉及模块：`internal/core`（`tenant.go`、`public.go`）、`internal/api/router`、`internal/contract`、
`frontend/src/features/onboarding`。

## 问题与现有边界

- 租户是所有资源的外键根：`tenant_memberships` 是 project owner 与 `credential_refs` 的 FK 目标，
  `idempotency_records` 以 `(tenant_id,user_id,key)` 为主键，公开 API 路径全部以 `/tenants/{tid}` 开头。
- 协作空间（`collab_workspaces`）在租户之下，是前端唯一展示的组织单位；每个 project 必须属于一个空间。
- 至今租户只能由部署者执行 `cloudctl bootstrap` 创建；普通用户首次经 Gateway 访问 `/api/v1/me` 后只得到
  `users` 记录，不属于任何租户，`GET /api/v1/me/tenants` 为空。
- 前端接入真实 Gateway 登录后，首次登录的用户必须有一条不依赖运维介入的路径拿到自己的第一个空间，
  否则登录后无事可做；同时不能因此把租户暴露成第二个需要用户理解的组织概念。

## 决策

### D1：`POST /api/v1/tenants` 由已验证身份直接授权，不要求任何租户成员资格

规则：请求只需通过公开 API 已有的双凭据校验（gateway service JWT + caller 绑定的 user JWT）。Cloud 先按
`(source, subject)` JIT 建立或读取用户并要求其 `active`，然后直接创建租户；不检查 `membership`，因为此时
没有可检查的租户。停用用户仍以 `403 user_disabled` 拒绝。

理由：这是唯一一个在租户存在之前发生的公开写操作。把授权前提收缩为“已验证且启用的身份”，既不放宽任何
既有路由的成员检查，也不引入第二套准入规则。外部组、邀请、配额或实例级“禁止自助创建”开关不在本决策
内；若需要，它们是对该接口的附加策略，而不是替代授权来源。

### D2：一个事务创建租户、管理员、第一个空间与 owner；租户借用空间名称

规则：`provisionTenant` 在一个事务内依次写入 `tenants`、`tenant_memberships(admin,active)`、
`collab_workspaces`、`collab_workspace_members(owner,active)`。租户 `name` 等于请求的空间 `name`（1–128
字节，去首尾空白后非空），空间 `slug` 经小写归一后必须匹配 `^[a-z0-9][a-z0-9-]{0,63}$`。响应 `201` 返回
`{tenant: {id,name,status,role}, space: Space}`。

理由：租户没有自己的名字来源，借用空间名让运维视图（`/me/tenants`、成员列表）与用户看到的名称一致；
把四条写入放在一个事务里，保证“租户必有管理员、空间必有 owner”这两条已有不变量在自助路径上同样成立。
`cloudctl bootstrap` 改为调用同一个 `provisionTenant`，只是租户名独立给出、空间固定为 `Default`/`default`。

| 路径 | 租户名 | 第一个空间 | 谁成为 admin/owner |
| --- | --- | --- | --- |
| `cloudctl bootstrap` | `-name` | `Default` / `default` | `-source`/`-subject` 指定的身份 |
| `POST /api/v1/tenants` | 请求 `name` | 请求 `name` / 请求 `slug` | 调用者 |

### D3：幂等键按 user+key 跨租户匹配，记录挂在新建租户名下

规则：接口要求 `Idempotency-Key`（缺失为 `400 idempotency_key_required`）。查找时使用
`idempotency_records WHERE user_id=$uid AND key=$key`，命中且请求 hash 相同则原样重放 `201` 响应，hash
不同为 `409 idempotency_conflict`；未命中则创建租户后把记录以新租户的 `tenant_id` 写入。

理由：`idempotency_records` 的主键与 FK 都要求租户已存在，而该接口恰恰在租户存在之前执行。按 user+key
匹配复用了现有表和 hash 语义，无需迁移；代价是同一用户在不同租户下复用同一 key 会互相看见对方的重放，
但 key 由客户端生成为 UUID，且 hash 校验仍会把不同请求判为冲突，因此不会误建或误重放。

### D4：自助租户没有 `default` 空间；租户级项目创建对其返回 404

规则：`defaultSpace` 仍只识别 slug 为 `default` 的空间。自助租户的第一个空间使用用户选择的 slug，因此
遗留的租户级 `POST /tenants/{tid}/projects` 对自助租户以 `404 not_found` 结束；项目应通过
`POST /tenants/{tid}/spaces/{sid}/projects` 创建。前端只使用空间级路径。

理由：把用户自选的 slug 强制改成 `default` 会让 URL 与用户输入不一致；改写 `defaultSpace` 为“最早的
空间”则会让 migration 0006 的回填语义变得模糊。租户级项目路径是空间引入前的兼容入口，明确它对自助租户
不可用比让它隐式选择空间更安全。是否在后续 ADR 中废弃该入口另行决定。

### D5：前端把“无租户”视为引导态，而不是错误态

规则：登录后 `GET /api/v1/me/tenants` 为空的用户进入 `/onboarding`，页面只有名称、slug（自动从名称
派生、可改）与 URL 预览；创建成功后进入 `/w/{slug}/issues`。已有租户但没有存活空间的用户也走同一页（改用
`POST /tenants/{tid}/spaces` 在已有租户下补建第一个空间）；前端为多租户成员解析“当前租户”时取列表第一个元素，
`me/tenants` 因此按租户创建时间升序（`id` 为决胜负列）返回，使“第一个”是确定的最早创建者，而不是随机的一个；
分页 cursor 仍是排他租户 UUID。已有租户的用户不经过该页；他们创建更多空间继续使用 `POST /tenants/{tid}/spaces`。
前端在任何界面都不显示租户名称或让用户切换租户，`tenantId` 只用于拼接 API 路径；工作区路由统一在保留前缀
`/w/` 之下，slug 永不与顶层路由冲突。

理由：这是本决策的产品含义——租户对用户是隐式的。若未来需要“一个用户多个租户”的可见模型，那是新的
决策，不是本 ADR 的延伸。

## 不变量

1. 任一时刻，`tenants` 中每个 `active` 租户至少有一个 `active` 的 `admin` 成员；自助路径在创建事务内即满足。
2. `POST /api/v1/tenants` 成功后，调用者同时是该租户 `admin` 与第一个空间 `owner`，且该空间 `tenant_id` 指向新租户。
3. 同一用户以相同 `Idempotency-Key` 与相同请求体重复调用，得到与首次完全相同的响应，不创建第二个租户；
   相同 key 不同请求体为 409。
4. 自助创建不改变任何既有租户、成员或空间；新租户对其他用户不可见，其他租户对新用户仍为 403。
5. slug 的唯一性范围仍是租户内；不同租户可以使用相同 slug。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 首次访问 `/api/v1/me` 时自动创建租户 | 把读操作变成写操作，且用户还没有机会给空间命名；也无法用幂等键约束。 |
| 所有登录用户自动加入同一个“默认租户” | 需要实例级配置决定该租户，破坏“租户是隔离边界”的前提；多人共享一个租户会让所有人互相可见。 |
| 去掉租户层，让空间直接成为根 | `credential_refs`、`projects`、`idempotency_records` 的 FK 与全部公开路径都以租户为根，是一次大迁移；与本次“接入真实登录”的目标无关。 |
| 新增 `tenant_provisioning` 幂等表 | 需要迁移；现有表按 user+key 匹配已能给出相同的重放与冲突语义。 |
| 前端在无租户时提示联系管理员 | 让自助注册的产品目标依赖运维，首次登录用户被卡死。 |

## 风险与未决项

- 自助创建没有速率或数量限制；一个用户可以无限创建租户。当前 Gateway 对登录有限流，但对 `/api/v1/*`
  没有。是否按用户限制租户数量是后续策略决策。
- 自助租户的 `default` 空间缺失使遗留租户级项目路径不可用（D4）；该路径的去留另行决定。
- 租户名与空间名在创建时相同，之后空间改名不会同步租户名；由于租户名不对用户展示，暂不处理。

## 落地与范围

实现已随 ora-space/cloud#14 合入 main；核心用例见 `test-cases/cloud/tenancy/self-serve-provisioning.md`
（三个：事务原子性与授权前提、user+key 跨租户幂等、成员租户列表创建时间升序分页），均已有直接证据。与本
ADR 同批落地的 Gateway SSE 中继与本地开发登录 provider 属于 identity-access 已批准 ADR 的范畴（其 D9/D10），
不在此展开。
