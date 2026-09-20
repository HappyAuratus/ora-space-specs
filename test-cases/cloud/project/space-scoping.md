# Project Space 作用域与角色门控

决策依据：[Project 归属 Space](../../../decisions/cloud/project/0-project-space-scoping.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/project_space_test.go`、
`integration/cloud_test.go`）与前端 Vitest 验证。

## Members create and read projects inside their space

- 风险：成员无法在 Space 内创建或读取 Project，协作边界形同虚设；或响应缺失归属信息。
- 前置状态：bob 为 Space member。
- 触发：bob `POST /spaces/:sid/projects`；随后列出 Space 项目。
- 必须成立：创建返回 202（既有异步语义）；列表恰含该项目且响应携带 `spaceId`。
- 禁止结果：member 创建被拒、列表缺失、响应不含 spaceId。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言
  member 创建 202、列表 1 项且 spaceId 与 Space 一致。

## Non-members cannot read a project or its runtime workspace

- 风险：仅凭 Project UUID 绕过 Space 边界读取内容或运行时状态。
- 前置状态：carol 是 tenant 成员但非该 Space 成员。
- 触发：carol `GET /projects/:pid` 与 `GET /workspaces/:wid`。
- 必须成立：均返回 404（与全 API 统一策略一致），不返回任何资源内容。
- 禁止结果：200、或返回 403 以外码泄露存在性。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates`（carol
  读取 404）与 `TestIdentityConcurrencyMembershipAndIsolation`（成员加入后 200，形成对照）。

## Members update business content and stale versions conflict

- 风险：业务写被错误门控，或多人编辑丢失覆盖；乐观锁语义在 Space 化后失效。
- 前置状态：bob 为 member，Project version=v。
- 触发：bob 以 version=v 修改 name；随后以同一 version 再次修改。
- 必须成立：第一次 200 且 name 生效；第二次 `409 version_conflict`。
- 禁止结果：member 修改被拒、陈旧版本覆盖成功。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言
  member PATCH 200 与陈旧 PATCH 409。

## Members cannot delete projects; owners delete through the lifecycle state machine

- 风险：删除权限过宽；或 Space 化后删除绕过既有 lifecycle 编排变成直接物理删除。
- 前置状态：Project lifecycle=active；bob 为 member。
- 触发：bob `DELETE /projects/:pid`；随后 owner 删除并推进编排。
- 必须成立：bob 返回 `403 space_role_required`；owner 返回 202 且 resource lifecycle 进入
  deleting；编排完成后 `deleted_at` 置值。
- 禁止结果：member 删除成功、owner 删除不走状态机、或删除被静默降级为软删。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言 403、
  deleting 状态、drain 后 deleted_at 置值。

## Project lists include only projects from spaces the caller joined

- 风险：列表泄漏未加入 Space 的 Project；或继续按 owner 过滤导致共享项目不可见。
- 前置状态：alice 在默认 Space 创建 Project；new-user 加入 tenant（自动获得默认 Space 成员）。
- 触发：new-user `GET /projects`（tenant 级）。
- 必须成立：列表恰含该共享 Project；加入前不可见。
- 禁止结果：按 owner 过滤导致列表为空、或未加入时可见。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  断言加入后列表恰 1 项。

## Cross-tenant project-space bindings are rejected at SQL level

- 风险：复合外键缺失或可被绕过，出现 Project 与 Space 跨 tenant 归属。
- 前置状态：两个 tenant 各有默认 Space。
- 触发：直接 SQL 插入 `space_id` 来自其他 tenant 的 Project。
- 必须成立：外键拒绝，插入失败。
- 禁止结果：跨 tenant 行入库。
- 验证义务与证据：**Partial** — 复合外键 `(space_id, tenant_id)` 由 migration 0007 建立且
  0007 的回填必须先于 NOT NULL 收紧（迁移成功即证明无 orphan）；尚无直接注入跨 tenant 行的
  违反性测试。

## The UI hides role-gated actions from members

- 风险：前端角色判断与后端矩阵不一致，member 看到并尝试被拒的操作。
- 前置状态：member 登录的浏览器会话。
- 触发：渲染 Project 详情、成员页与设置页。
- 必须成立：member 看不到删除入口与成员管理控件；owner 可见归档 danger zone。
- 禁止结果：member 视图出现删除/成员管理入口。
- 验证义务与证据：**Covered** — 前端 `project-detail-page.test.tsx`
  （hides the delete action from members）、`members-page.test.tsx`
  （hides management controls from members）、`general-settings-page.test.tsx`
  （hides the danger zone from non-owners）。
