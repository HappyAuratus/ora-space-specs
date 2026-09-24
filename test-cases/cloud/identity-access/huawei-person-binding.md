# 华为双标识与人员目录添加

决策依据：[IDaaS UUID 与 globalUserId 关联](../../../decisions/cloud/identity-access/20260923-huawei-global-id-alias.md)。

## Verified Huawei Identifiers Bind One User Across Employee Number Changes

- 风险：用可变化工号授权导致成员丢失或错误转移；两个已存在身份自动合并导致越权。
- 前置状态：管理员按天舟 `globalUserId` 预添加人员，人员尚未登录；另有两名已登录人员。
- 触发：工号改变后首次 IDaaS 登录；让某个已绑定 uuid 携带另一人的 globalUserId。
- 必须成立：首次登录与预添加同一用户；工号不参与匹配；冲突返回 409 且原绑定不变。
- 禁止结果：为新工号建立第二用户、按姓名合并、冲突迁移成员关系。
- 决策依据：本 ADR D1。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 两种 IDaaS 字段经 Gateway 验证并传至 Cloud | `idaas.TestExchangeUsesClientSecretPostAndReturnsCorporateIdentity`、`integration.TestGatewayIDaaSLoginIdentityAndSessionLifetime` | `Covered` |
| 预添加和工号变化后首次访问（含内部准入）保持同一用户 | `integration.TestHuaweiGlobalIdentityBindsPreAddedMemberAcrossEmployeeNumberChange` | `Covered` |
| 已有 UUID 与 globalUserId 冲突拒绝合并 | `integration.TestHuaweiIdentityConflictAndInactiveEmployeeAreRejected` | `Covered` |

## Directory Lookup Never Grants Access Without A Fresh Employment Check

- 风险：浏览器伪造人员、复用过期搜索结果或读取天舟敏感字段，给离职人员开通权限。
- 前置状态：管理员使用内网部署的人员搜索；天舟可返回在职与非在职人员。
- 触发：搜索、选择人员添加；天舟不可用或所选人员状态改变。
- 必须成立：服务端带部署密钥调用固定天舟地址；公开结果只有姓名、工号、部门和稳定 ID；添加时重查在职状态，失败不写成员。
- 禁止结果：浏览器看到密钥、邮箱或手机号；目录失败仍创建成员。
- 决策依据：本 ADR D2。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 固定天舟地址、机器认证头、精简和在职过滤 | `router.TestTianzhouSearchSendsMachineHeadersAndProjectsOnlyEmployedPeople` | `Covered` |
| 上游失败安全拒绝及非在职拒绝 | `router.TestTianzhouSearchFailsClosedOnUpstreamFailure`、`integration.TestHuaweiIdentityConflictAndInactiveEmployeeAreRejected` | `Covered` |
| 搜索结果和添加之间重新调用天舟 | `integration.TestHuaweiMemberAddRechecksDirectoryAfterSearch` | `Covered` |
