# 租户即空间及私有加入

决策依据：[每个协作空间对应一个租户](../../../decisions/cloud/tenancy/20260923-one-space-per-tenant.md)。

## A User Sees Every Joined Tenant Space And One Space Per Tenant

- 风险：仍取首个租户或沿用空间级成员表，会隐藏空间或产生两份权限来源。
- 前置状态：用户创建两个租户，另一用户尚未加入。
- 触发：列出 `/me/spaces`、在前端切换，尝试在已有租户内再建空间；改名和重复 slug。
- 必须成立：每个租户恰有一个空间；列表跨租户完整；slug 全局唯一、不可改且不复用；租户与空间名称一起更新。
- 禁止结果：第二空间写入、跨租户 slug 重复、未知 slug 借用首个租户权限。
- 决策依据：本 ADR D1、D2。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 单租户单空间、全局 slug 冲突和同名改名 | `integration.TestTenantOwnsOneVisibleSpaceAndSharesItsName`、`integration.TestSelfServeTenantProvisioning` | `Covered` |
| 多租户空间列表与路由选中的租户 ID | `integration.TestTenantOwnsOneVisibleSpaceAndSharesItsName`、前端 `spaces.test.tsx`、`app-sidebar.test.tsx` | `Covered` |
| 已归档 slug 不复用 | `integration.TestArchivedSpaceSlugRemainsReserved` | `Covered` |

## Tenant Membership Is The Only Role Source And Keeps One Admin

- 风险：成员停用后仍可访问；并发降级可能使租户无管理员。
- 前置状态：两个管理员和一个普通成员；已存在共享项目。
- 触发：并发降级两位管理员；停用普通成员后使用旧凭据读取资源。
- 必须成立：只有 `admin`、`member` 两级；恰好一位管理员仍有效；停用立即拒绝新访问与执行准入。
- 禁止结果：`owner` 权限残留、两个管理员都降级、已停用成员获得新 ticket。
- 决策依据：本 ADR D1。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 并发降级保留一位管理员 | `integration.TestTenantMemberLastAdminRace` | `Covered` |
| 停用后读取与准入立即被拒，直接 PUT 不能新建或恢复成员 | `integration.TestInvitationRedeemsOnceAndRemovalImmediatelyRevokesAccess`、`integration.TestRemainingPublicContractsAndMembershipRevocation` | `Covered` |

## Invitation Redemption Is Single Use And Join Requests Require Approval

后续 approved 的[加入来源与链接预检契约](../../../decisions/cloud/tenancy/20261027-member-entry-provenance-and-link-preview.md)
修改已有成员的消费行为：空操作不消费、不发加入通知、不写来源。下表保留原有邀请/审批义务的证据范围，
不证明新增行为；[新增核心用例](member-entry-provenance-and-link-preview.md)独立登记，状态均为 Missing。

- 风险：泄漏令牌、撤销或到期仍可加入、并发兑换获得两个成员资格、申请绕过管理员。
- 前置状态：管理员创建 7 天邀请和 30 天申请链接；两个已登录普通用户。
- 触发：兑换、重放、并发兑换、到期、撤销；两个用户申请后分别批准和拒绝。
- 必须成立：数据库仅有令牌摘要；邀请只能一次授予普通成员；申请链接可多人使用但审批前无权限；撤销和到期拒绝新操作。
- 禁止结果：原文令牌入库、同一邀请产生第二次真实准入、待审批者读取租户资源；幂等重放和已有成员空操作不算新的准入。
- 决策依据：本 ADR D3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 单次、幂等、并发、到期与撤销 | `integration.TestInvitationRedeemsOnceAndRemovalImmediatelyRevokesAccess`、`integration.TestInvitationExpiryRevocationAndConcurrentRedemption` | `Covered` |
| 多人申请、批准前拒绝、批准与驳回、撤销 | `integration.TestJoinRequestNeedsAdminApprovalAndLinkCanBeRevoked` | `Covered` |
| 前端创建私有链接、兑换与待审批状态 | `members-page.test.tsx`、`join-page.test.tsx`、`onboarding-page.test.tsx` | `Covered` |
