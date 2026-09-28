# 协作空间与租户成员的一致性

决策依据：[单空间租户](../../../decisions/cloud/tenancy/20260923-one-space-per-tenant.md)。此前的独立空间成员与 `owner` 角色已被此决策取代。

## Creating A Tenant Grants Its Creator The Sole Initial Admin Role

- 风险：创建者未成为管理员，或空间有第二份独立角色，造成权限漂移。
- 前置状态：已登录用户不属于目标租户。
- 触发：`POST /api/v1/tenants` 创建；读取唯一空间及成员。
- 必须成立：一个事务创建租户、唯一空间与一条 active admin `tenant_memberships`；无 `collab_workspace_members` 表。
- 禁止结果：空间无管理员、`owner` 角色残留、同租户出现第二个空间。
- 验证义务与证据：**Covered** — `integration.TestSelfServeTenantProvisioning`、`integration.TestTenantOwnsOneVisibleSpaceAndSharesItsName`。

## The Last Admin Cannot Be Demoted Or Disabled

- 风险：成员写入或并发操作移走最后有效管理员。
- 前置状态：租户有一位或两位管理员。
- 触发：`PUT /tenants/:tid/members/:uid` 降级或停用；两位管理员并发降级。
- 必须成立：最后一位管理员的更新返回 `409 last_admin`；并发时恰一位保留。
- 禁止结果：有效管理员计数为零。
- 验证义务与证据：**Covered** — `integration.TestTenantMemberLastAdminRace`、`integration.TestRemainingPublicContractsAndMembershipRevocation`。

## A Tenant Member Reads Its Sole Space Without A Second Join

- 风险：租户成员加入后仍被旧空间级成员逻辑拒绝。
- 前置状态：已有空间及共享项目，管理员添加普通成员。
- 触发：成员读取空间、项目和运行时 workspace；未加入租户的用户尝试同样操作。
- 必须成立：有效成员可以读取，非成员被拒；停用后立即拒绝新访问。
- 禁止结果：依赖已删除的空间成员表、跨租户可见。
- 验证义务与证据：**Covered** — `integration.TestIdentityConcurrencyMembershipAndIsolation`、`integration.TestInvitationRedeemsOnceAndRemovalImmediatelyRevokesAccess`。
