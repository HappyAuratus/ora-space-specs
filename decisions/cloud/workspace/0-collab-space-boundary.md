---
status: implemented
---

# 协作 Space 是租户内的共享、可见性与授权边界

> 后续已实现决策：[运行时使用权限](20260927-runtime-workspace-use-permissions.md)增加创建者/管理员使用权限，状态为 `implemented`。服务规则以其继承表为准，UI 与完整故障验收另由后续 ADR 承接；本文保留原状态。

> 历史决策：下文记录 0006/0007 阶段的模型。当前一租户一空间、租户成员唯一授权、无独立空间成员和空间 owner，由 [20260923-one-space-per-tenant](../tenancy/20260923-one-space-per-tenant.md) 取代；本页保留旧迁移的决策背景。

Cloud 在 Tenant 之下新增协作 Space（产品 UI 称为 Workspace，表 `collab_workspaces`）作为多人
协作的共享与可见性边界。每个 Project 必须属于且仅属于一个 Space；Space 成员
（`collab_workspace_members`）使用 `owner/admin/member` 角色与 `active/disabled` 状态管理，
所有 Space 与 Project 访问都必须通过 Space membership 判定。Space 与运行时 `workspaces` 表
（执行环境）严格分离，互不共享表、外键与状态机；Space 删除执行归档而非物理删除，归档不影响
其下 Project 的运行时状态。每个 live tenant 恰有一个默认 Space（`slug=default`），tenant 成员
加入时自动获得其成员身份。

本决策已实现：migration 0006/0007、`internal/core/space.go`、`/spaces` 路由、SSE 事件流、
前端 Space 切换与成员页，并全量通过 PostgreSQL 集成测试与前端门禁。Tenant、用户身份与租户
成员模型不变；本决策不引入协作任务（Issue）、邀请流程、viewer 角色与 Space 间引用。

## 背景

Cloud 的 Project 原以 `owner_user_id` 作为可见性边界，只有创建者可读可改。引入多人协作需要
租户内的分组边界，但运行时 `workspaces` 表已有执行语义（`kind`、desired/observed 状态机、
admission 准入），被 tasks/operations/effects/sandboxes 等 7+ 张表外键引用，不能复用为协作
边界。因此新增独立的协作聚合，并把 Project 的可见性与授权从个人所有改为 Space 成员身份
（见 [Project Space 作用域](../project/0-project-space-scoping.md)）。

## D1：Space 是独立的协作聚合，与运行时工作区严格分离

`collab_workspaces` 只有协作语义：`tenant_id`、`name`、`slug`、`description`、`created_by`、
`version`、时间戳与 `archived_at`，不包含任何执行状态。运行时 `workspaces` 表及其全部外键、
状态机与 admission 语义不变，两套聚合不互相引用。

产品 UI 与前端路由沿用 Workspace 术语（`:workspaceSlug`），公开 API 路径段使用 `spaces`
（`/api/v1/tenants/:tid/spaces/...`），因为 `/workspaces/...` 已被运行时路由占用（见
[Space API 与事件流](../api-boundary/0-space-api-and-event-stream.md)）。

## D2：成员与角色

`collab_workspace_members` 以 `(workspace_id, user_id)` 为主键，字段为 `role`、`status`、
`version`、`created_by`、`joined_at`。成员必须是同 tenant 的 active user 且持有 active
tenant membership：SpaceMember 的前置条件是 TenantMember，但两者相互独立
（TenantMember ≠ SpaceMember），一个用户可属于任意多个 Space。

创建 Space 的事务同时插入创建者为 owner，两步原子完成。角色权限矩阵：

| 操作 | member | admin | owner |
| --- | :-: | :-: | :-: |
| 查看 Space 与成员 | ✓ | ✓ | ✓ |
| 修改 Space 基本信息（name/description） | - | ✓ | ✓ |
| 归档 Space | - | - | ✓ |
| 添加/移除成员、修改成员角色状态 | - | ✓ | ✓ |
| 授予/回收 owner | - | - | ✓ |
| Project 创建/读取/业务修改 | ✓ | ✓ | ✓ |
| Project 删除与敏感操作 | - | ✓ | ✓ |

## D3：最后 owner 不变量

每个未归档 Space 必须始终至少有一个 active owner。禁止删除、降级或禁用最后一个 owner，违反
时返回 `409 space_last_owner`。检查在事务内完成：先查询目标之外的其他 active owner，再执行
更新。所有公共事务经 `pg_advisory_xact_lock(67420911)` 全局串行化，因此并发降级两个 owner
时恰有一个成功、一个 409，Space 不会失去最后 owner。这与 tenant 的 last-admin 保护同构。

## D4：默认 Space 与成员自动加入

每个 live tenant（以及仍拥有 Project 的已删除 tenant）恰有一个未归档的默认 Space
（`slug=default`）。默认 Space 由三条路径保证存在：

1. migration 0007 为既有 tenant 确定性回填（见
   [Schema 与回填](../persistence/0-collab-space-schema-and-backfill.md)）；
2. `Bootstrap` 在创建 tenant 的同一事务内创建默认 Space，并让初始管理员成为 owner；
3. tenant 成员加入时（`PUT /tenants/:tid/members/:uid` 新建成员）自动加入默认 Space：
   admin 映射为 owner、member 映射为 member，`ON CONFLICT DO NOTHING` 保证幂等。

由此，新 tenant 成员立即通过默认 Space 获得共享可见性；legacy 的 tenant 级 Project 创建路径
（`POST /tenants/:tid/projects`）回退到默认 Space，保持旧客户端可用。

## D5：slug 归一化与不可变

slug 创建时先小写化并去除首尾空白再校验，格式 `^[a-z0-9][a-z0-9-]{0,63}$`，在 tenant 内
唯一（`UNIQUE(tenant_id, slug)` 加事务内预查；重复返回 `409 space_slug_conflict`，格式错误
返回 `400 invalid_slug`）。创建后不可修改：`PATCH` 只接受 name 与 description，严格字段
白名单拒绝携带 slug 的更新。slug 的稳定性保护前端路由、分享链接与 CLI 配置不发生漂移。

## D6：归档而非物理删除

`DELETE /spaces/:sid` 只写 `archived_at`（owner 专属，需匹配 version）。归档后 Space 对一切
读取与成员操作表现为不存在（404），列表不再出现；不支持恢复。归档不触碰其下任何 Project 行，
Project 的删除仍走既有 lifecycle 状态机，两者生命周期独立。

## 不变量

1. Project 属于且仅属于一个 Space；Space 与运行时工作区是两类互不共享状态的聚合。
2. 每个未归档 Space 至少有一个 active owner，任何路径都不能使其失去最后 owner。
3. SpaceMember 必须是同 tenant 的 active tenant member 与 active user；TenantMember 不等于 SpaceMember。
4. slug 创建后不可变且在 tenant 内唯一。
5. 归档 Space 对一切访问表现为不存在；归档不改变其下 Project 的状态。
6. 所有 Space 与 Project 访问都必须经过当前用户的 Space membership 判定。
7. 每个 live tenant 恰有一个未归档默认 Space，tenant 成员加入时自动获得其成员身份。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 复用运行时 `workspaces` 表表达协作 | 破坏 7+ 张执行表的外键与 isolated 1:1、admission 等运行时约束，协作与执行生命周期纠缠 |
| 不建 Space，Project 直接挂多人 ACL | 每个 Project 重复维护成员与角色，没有共享可见性与切换边界，无法表达产品 Workspace 概念 |
| Tenant 即协作边界 | 失去租户内分组能力；Multica 的实践表明 Workspace 层是协作的顶层边界 |
| 合并 TenantMember 与 SpaceMember | 组织角色与协作角色语义冲突，历史数据迁移风险高 |
| slug 允许修改 | 分享链接、路由与缓存漂移 |
| 物理级联删除 Space | 误操作不可恢复；Project 可能仍有运行与历史记录，级联删除未来更危险 |

## 落地与验收

实现位于 `internal/core/space.go`（`createSpace`/`listSpaces`/`patchSpace`/`archiveSpace`/
`putSpaceMember`/`spaceMember`/`defaultSpace`）与 migration 0006/0007。核心行为由
`integration/space_test.go` 的 `TestSpaceLifecycleMembershipAndOwnerInvariants`、
`TestSpaceMemberLastOwnerRace` 与 `TestIdentityConcurrencyMembershipAndIsolation` 直接验证，
前端 Space 切换与成员页由 `features/spaces`、`features/members` 的 Vitest 覆盖。核心用例见
[协作 Space 成员与归档](../../../test-cases/cloud/workspace/collab-space-membership.md)。

## 本决策未解决的问题

- 归档后的恢复（unarchive）暂不支持，是否开放待后续决策。
- 邀请流程（invitation token / pending 状态）与 viewer 角色不在本决策范围。
- 协作任务（Issue）属于下一阶段，其归属校验必须复用 Space 作用域不变量。
