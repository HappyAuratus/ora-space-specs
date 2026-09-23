---
status: implemented
---

# Project 归属 Space：以成员身份取代个人所有作为可见与授权边界

`projects.space_id` 成为必填字段，复合外键 `(space_id, tenant_id) → collab_workspaces(id, tenant_id)`
在 SQL 级杜绝跨租户归属。Project 的可见性与授权从 `owner_user_id` 个人所有改为 Space 成员身份：
active Space member 可以读取与修改业务内容，删除与敏感操作要求 `admin/owner`。实体路由从
Project 自身反查 Space 并校验成员身份，绝不信任客户端提供的 spaceId。`owner_user_id` 降级为
创建者审计字段，其既有复合外键骨架（operations/effects/credential_refs 的引用）全部不动，
`immutable_ownership` 触发器与乐观版本控制不变。

本决策已实现并全量通过集成测试（`integration/project_space_test.go`、
`TestIdentityConcurrencyMembershipAndIsolation`），`status: implemented` 表示规则与实现同步。

## 行为变化与不变边界

| 约定 | 本决策 |
| --- | --- |
| `owner_user_id` 作为可见性与授权边界 | 修改：降级为创建者审计字段，不再参与访问判定 |
| `projects` 的 tenant/owner 复合 FK 骨架与 `immutable_ownership` 触发器 | 继承，不变 |
| Project 删除走 lifecycle 状态机（quiesce→terminate→cleanup→storage_delete） | 继承，不变；新增删除前 role ∈ {admin, owner} 校验 |
| 乐观版本 + `409 version_conflict` / `428 version_required` | 继承，不变 |
| 运行时 `workspaces` 的 project 归属 | 继承，不变 |
| tenant membership 前置条件 | 继承，不变 |

## D1：归属与 SQL 级约束

`projects.space_id` 为 NOT NULL，复合外键引用 `collab_workspaces(id, tenant_id)`，配合
`UNIQUE(id, tenant_id)` 在 SQL 级保证 Project 与其 Space 同 tenant；`project_space_list
(space_id, id)` 索引支持 Space 作用域列表。`UNIQUE(id, tenant_id, owner_user_id)` 及被
credential_refs/operations/external_effects 引用的复合 FK 全部保留，创建 Project 时
`owner_user_id` 仍写当前用户，保持执行域外键骨架不破坏。

## D2：授权模型与授权链

- 读取与业务写（`PATCH` name、启动/停止运行时等日常操作）：active Space membership 即可。
- 删除与敏感操作：role ∈ {admin, owner}，否则 `403 space_role_required`；通过后仍走既有
  lifecycle 状态机，不改变删除编排。
- 授权链固定为：`projectId` → 加载 Project（tenant + 未删除）→ 校验
  `spaceMember(project.space_id, uid)` → 必要时校验 role。entity API 不接收客户端 spaceId；
  集合 API 的 spaceId 来自路由参数，且创建/列表都先做成员校验。
- 运行时 workspace 的非管理读取跟随其 project 的 Space membership
  （`workspace()` 非 admin 路径改为空间成员校验）；`administrative-stop` 仍要求 tenant admin。
- 执行准入（`/internal/v1/access`、`/internal/v1/admissions`）随本决策从 owner 专属放宽为
  Space 成员，见
  [执行访问准入跟随 Space 成员身份](../execution-admission/0-access-follows-space-membership.md)。

## D3：Space 作用域集合与跨 Space 禁止

- `GET/POST /spaces/:sid/projects`：创建要求该 Space membership，列表按 `space_id` 过滤且
  排除已删除。
- tenant 级 `GET /projects` 从按 owner 过滤改为 membership join：只返回调用者已加入且未归档
  Space 的 Project。
- legacy `POST /tenants/:tid/projects` 保留，回退到 tenant 的默认 Space，旧客户端不受影响。
- 跨 Space 引用是硬性禁止：SQL 层由复合外键保证同 tenant；core 层所有 Project 查询都携带
  Space 范围或经 `projectInSpace` 推导；只检查目标 ID 存在性的查询不被允许。未来 Issue →
  Project 的归属校验必须复用同一不变量。

## 不变量

1. `projects.space_id` 恒非空，且与 tenant_id 一致（SQL 级）。
2. 任何 Project 读取或变更都通过其 Space 的 active membership；不能仅凭 Project UUID 跨 Space 访问。
3. member 不可删除 Project 或做敏感配置变更（403）。
4. 授权永远从持久化的 `project.space_id` 推导，客户端提供的空间标识不被信任。
5. Project 删除与 Space 归档相互独立，互不触发。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 只在 HTTP middleware 校验 Space | 内部调用与未来 handler 可绕过，安全边界不能只存在于路由层 |
| 信任客户端传入的 spaceId | 身份与范围字段由服务端权威，客户端可伪造跨 Space 引用 |
| 保留 owner 可见性 + Space 附加共享 | 双重可见性边界使成员可见性难以推理 |
| `space_id` 保持可空 | 出现未归属 Project 窗口，跨 Space 校验失去 SQL 保障 |
| 删除也开放给 member | 触发 lifecycle 状态机的破坏性操作需要更高角色 |

## 落地与验收

实现位于 `internal/core/public.go`（`projectInSpace`、Space 作用域读写）、
`internal/core/store.go`（`project`/`workspace` 授权改造）。核心行为由
`integration/project_space_test.go` 的 `TestProjectSpaceScopingAndRoleGates`、
`TestSpaceMemberLastOwnerRace` 与 `TestIdentityConcurrencyMembershipAndIsolation` 直接验证，
前端角色门控由 `project-detail-page.test.tsx`、`members-page.test.tsx` 覆盖。核心用例见
[Project Space 作用域](../../../test-cases/cloud/project/space-scoping.md)。
