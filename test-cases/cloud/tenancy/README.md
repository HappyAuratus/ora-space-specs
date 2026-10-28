# Tenancy 核心测试用例

本目录是 Cloud tenancy 领域的 reviewer 风险导航图，镜像
`specs/decisions/cloud/tenancy/`。它记录自助租户创建、租户内隔离与成员可见性需要长期守住的系统承诺，
以及这些承诺是否已有直接、可信的可执行证据；它不是测试函数目录。

架构边界和不变量来自
[已登录用户自助创建租户，租户成为协作空间的隐式容器](../../../decisions/cloud/tenancy/0-self-serve-tenant-provisioning.md)。

## 小队定向加入

[定向加入与撤权用例](targeted-joining-and-revocation.md)依据
[approved 后续 ADR](../../../decisions/cloud/tenancy/20261027-squad-targeted-joining-and-inviter-revocation.md)，
新增义务均为 Missing。既有邀请、租户隔离测试的 Covered 不表示新原子编组、空操作、来源及签发者失权规则已覆盖。

## 阅读与维护规则

- 核心用例按业务风险组织，不按 Go package、单元测试或集成测试组织。
- 代表性证据写作 `包.测试函数`：`integration.*` 是 `integration/` 下使用真实 HTTP 与 PostgreSQL 的验收
  测试。
- 只有在测试失败会直接说明某项验证义务不成立时，才把该测试列为代表性证据。
- 文件路径与用例标题共同构成稳定引用；集成测试的 doc comment 引用它们。
- 改变租户创建事务、幂等键匹配范围、角色写入或 `me/tenants` 排序时，必须检查相关核心用例及其证据状态。
