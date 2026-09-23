# 协作 Space 成员、角色与归档

决策依据：[协作 Space 是租户内的共享、可见性与授权边界](../../../decisions/cloud/workspace/0-collab-space-boundary.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/space_test.go`、
`integration/project_space_test.go`、`integration/cloud_test.go`）验证。

## Creating a space grants the creator the sole owner role

- 风险：创建事务只插 Space 不插 owner，或角色写错，导致 Space 出现后立刻违反 owner 不变量。
- 前置状态：active tenant member（alice）登录。
- 触发：`POST /spaces` 创建 Space，随后直接查询成员表。
- 必须成立：创建返回 200 且 slug 归一化；`collab_workspace_members` 中
  `(workspace_id, alice)` 恰一行，role=owner、status=active。
- 禁止结果：owner 行缺失、重复，或成员角色不是 owner。
- 验证义务与证据：**Covered** — `integration TestSpaceLifecycleMembershipAndOwnerInvariants`
  断言数据库 owner 行计数恰为 1。

## The last owner cannot be demoted or disabled

- 风险：成员管理路径在更新后才发现失去最后 owner，留下无 owner 的 Space。
- 前置状态：Space 只有 alice 一个 owner；bob 为 admin。
- 触发：bob 与 alice 先后对 alice 执行 `PUT /spaces/:sid/members/:uid` 降级或禁用。
- 必须成立：两次都返回 `409 space_last_owner`，成员行不变。
- 禁止结果：owner 降级成功、错误码不是 space_last_owner、或降级后 owner 数为 0。
- 验证义务与证据：**Covered** — `integration TestSpaceLifecycleMembershipAndOwnerInvariants`
  覆盖 admin 与 owner 两个角色的降级尝试。

## Concurrent demotion of two owners leaves exactly one owner

- 风险：并发检查-后-写使两个 owner 同时被降级，最后 owner 保护被绕过。
- 前置状态：Space 有两个 owner（alice、bob），成员 version 均为 1。
- 触发：两个 goroutine 同时用同一 version 把两位 owner 降级为 member。
- 必须成立：恰一个 200、一个 409；owner 计数恒为 1。
- 禁止结果：两个都成功、或 owner 计数为 0/2。
- 验证义务与证据：**Covered** — `integration TestSpaceMemberLastOwnerRace` 断言
  {200, 409} 恰好各一且 owner 计数为 1。

## Slug is normalized, immutable, and unique per tenant

- 风险：slug 大小写漂移或可被修改，使前端路由、分享链接与 CLI 配置失去稳定锚点。
- 前置状态：tenant 已有 slug=`collab-space` 的 Space。
- 触发：以 `Collab-Space`、重复 slug、`Bad Slug!` 分别创建；以携带 slug 的 PATCH 更新。
- 必须成立：创建归一化为小写 `collab-space`；重复返回 `409 space_slug_conflict`；格式错误
  返回 `400 invalid_slug`；PATCH 字段白名单不含 slug。
- 禁止结果：大小写 slug 入库、重复 slug 成功、PATCH 可改 slug。
- 验证义务与证据：**Covered**（归一化与唯一性）— `integration
  TestSpaceLifecycleMembershipAndOwnerInvariants` 断言归一化结果与 409/400；**Partial**
  （不可变性）— 由路由字段白名单与严格解码保证，尚无直接断言 PATCH 携带 slug 被拒的测试。

## Members and admins cannot archive; owners archive and the space disappears from all reads

- 风险：归档权限过宽，或归档后仍有读取路径泄漏内容。
- 前置状态：alice 为 member、bob 为 owner 的 Space。
- 触发：alice `DELETE /spaces/:sid`；bob 归档；随后列表与单查。
- 必须成立：alice 返回 403；bob 返回 200 且 `archivedAt` 置值；归档后 `GET /spaces` 列表不含
  该 Space，成员读取返回 404。
- 禁止结果：member 归档成功、归档 Space 出现在列表或可被读取。
- 验证义务与证据：**Covered** — `integration TestSpaceLifecycleMembershipAndOwnerInvariants`
  覆盖 403、200、列表排除与读取 404。

## Archiving never alters the space's projects

- 风险：归档联动改动 Project 行，破坏 Project 独立 lifecycle 状态机。
- 前置状态：Space 内含 active Project。
- 触发：owner 归档 Space，随后检查 Project 行。
- 必须成立：Project 行（lifecycle、version、deleted_at）不变，运行时编排不受影响。
- 禁止结果：归档写 projects 表、触发 Project 删除或停止。
- 验证义务与证据：**Partial** — 归档事务只更新 `collab_workspaces`（代码事实）并有手动
  验证手册（test_2.md S14）记录；尚无自动化测试直接断言含 Project 的 Space 归档后 Project
  行逐字段不变。

## Every tenant member joins the default space

- 风险：成员加入 tenant 后未获得默认 Space 成员身份，共享可见性残缺；或映射与 0007 回填
  不一致（admin 应为 owner）。
- 前置状态：alice 在默认 Space 创建了 Project；new-user 尚未加入 tenant。
- 触发：admin 将 new-user 以 member 角色加入 tenant；new-user 读取该 Project 与运行时
  workspace。
- 必须成立：加入 tenant 后（无显式 Space 操作）读取 Project 与 workspace 返回 200；tenant
  级 Project 列表含该共享 Project；未加入 tenant 的用户 `POST /projects` 返回 403。
- 禁止结果：读取 404/403、列表为空、非 tenant 成员可创建 Project。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  断言加入后双读取 200、列表恰 1 项、非成员创建 403。

## Space lists contain only spaces the caller joined

- 风险：属于 tenant 即暴露全部 Space，破坏 Space 可见性边界。
- 前置状态：alice 加入 default 与 collab 两个 Space；carol 为 tenant 成员但只加入 default。
- 触发：两人分别 `GET /spaces`。
- 必须成立：alice 列表恰含已加入的两个 Space 且带角色；carol 列表不含未加入的 Space。
- 禁止结果：未加入的 Space 出现在列表中。
- 验证义务与证据：**Covered** — `integration TestSpaceLifecycleMembershipAndOwnerInvariants`
  断言 alice 列表恰 2 项且含目标 Space。
