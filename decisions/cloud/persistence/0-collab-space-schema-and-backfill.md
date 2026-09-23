---
status: implemented
---

# 协作 Space 的 Schema、确定性回填与迁移不可变性

新增 migration 0006（`collab_workspaces` + `collab_workspace_members`）与 0007
（`projects.space_id` 回填与约束升级）。编号在合入上游 main 时顺延：上游已占用 0005
（gateway 认证表）。0007 的默认 Space 回填完全确定、不依赖在线 IDaaS：
为每个 live 或仍拥有 Project 的 tenant 创建 `slug=default` 的 Space，把全部 active tenant 成员
按 admin→owner、member→member 映射加入，再把既有 Project 绑定到默认 Space，最后将
`space_id` 收紧为 NOT NULL 并加复合外键。已应用迁移文件不可变（checksum 校验），后续变更一律
走新的 forward migration；`Bootstrap` 与 tenant 成员加入路径同步维护默认 Space，保证该
Schema 在运行时永远满足。

本决策已实现并通过全新库与旧库升级两条测试路径，`status: implemented` 表示规则与实现同步。
运行时 `workspaces` 表、既有 0001–0004 迁移及其约束不变。

## D1：0006 协作 Space 表

`collab_workspaces`：`(id, tenant_id)` 复合唯一（复合外键骨架）、`UNIQUE(tenant_id, slug)`、
name 长度 CHECK、slug 格式 CHECK（`^[a-z0-9][a-z0-9-]{0,63}$`）、`version > 0`、
`archived_at` 可空、`created_by REFERENCES users(id)`；索引 `collab_workspace_list
(tenant_id, id)`。

`collab_workspace_members`：主键 `(workspace_id, user_id)`，`role` CHECK
（owner/admin/member）、`status` CHECK（active/disabled）、`version > 0`、`created_by` 可空、
`joined_at`；索引 `collab_workspace_member_user(user_id)`。

## D2：0007 回填与约束升级

按固定顺序执行，全部在一条迁移内完成：

1. 为 `deleted_at IS NULL` 或仍拥有 Project 的每个 tenant 创建默认 Space（name `Default`、
   slug `default`）；`created_by` 取 `COALESCE(最早 tenant membership 的用户, 最早 users 行)`
   ——确定性，无在线 IDaaS 调用，重复执行结果一致（slug 唯一约束保证每 tenant 恰一个）。
2. 全部 active tenant member 加入默认 Space：admin→owner、member→member。实现比实施计划
   「只回填管理员」更宽：任何 active 成员都获得默认 Space 成员身份，与运行时「成员加入自动
   同步默认 Space」的规则一致。
3. `projects.space_id` 回填为所属 tenant 的默认 Space；随后
   `SET CONSTRAINTS ALL IMMEDIATE` 强制触发回填 UPDATE 排队的 deferred 约束触发器——PostgreSQL
   在 pending trigger events 期间拒绝 `ALTER TABLE`，必须先清空。
4. `space_id SET NOT NULL`、复合外键 `(space_id, tenant_id) → collab_workspaces(id, tenant_id)`、
   索引 `project_space_list(space_id, id)`。

回填之后 NOT NULL 才能收紧，因此迁移成功本身即证明不存在 orphan Project；复合外键在 SQL 级
杜绝跨 tenant 归属。

## D3：迁移纪律与运行时同步

- 已应用迁移不可变：checksum 逐条校验，`CheckSchema` 在启动与测试中核对；任何后续变更必须
  新写 forward migration，不得改写 0001–0007。
- 测试覆盖两条路径：全新库按序执行全部迁移；升级路径由
  `TestMigrateUpgradesPreviousSchemaAndData` 以 0001–0003 种子数据升级到最新并校验 checksum
  与数据保真。
- `Bootstrap` 在创建 tenant 的同一事务内创建默认 Space 并让初始管理员成为 owner（返回
  `spaceId`）；`putMember` 新建 tenant 成员时以 `ON CONFLICT DO NOTHING` 幂等加入默认 Space。
  两条运行时路径与 0007 的回填映射一致（admin→owner、member→member）。

## 不变量

1. 已应用迁移文件不可变，任何后续变更走新的 forward migration。
2. 每个 live tenant 恰有一个未归档默认 Space（slug 唯一约束 + 三条创建路径一致）。
3. `projects.space_id` 非空且复合外键保证与 tenant 一致。
4. 迁移与回填不依赖在线 IDaaS 调用，可确定重放。
5. 升级路径（旧数据 → 最新 schema）与全新路径都通过测试与 checksum 校验。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 回填调用在线 IDaaS 取管理员 | 迁移不可重入、依赖外部可用性，破坏确定性 |
| 默认 Space 应用层 lazy 创建 | NOT NULL 约束时机失控，出现未归属 Project 窗口 |
| 改写旧迁移（0001–0004） | checksum 破坏，历史部署无法升级 |
| 用触发器表达跨表 tenant 一致性 | CHECK/触发器无法跨表；复合外键是 SQL 级可靠表达 |
| 回填只加管理员、其他成员不加默认 Space | 与运行时「成员自动加入默认 Space」不一致，共享可见性残缺 |

## 落地与验收

实现位于 `internal/core/migrations/0006_collab_spaces.sql`、`0007_project_space_scope.sql` 与
`internal/core/store.go`（`Bootstrap`、`putMember` 同步）。升级路径由
`TestMigrateUpgradesPreviousSchemaAndData`（`integration/cloud_test.go`）验证，运行时同步由
`TestIdentityConcurrencyMembershipAndIsolation` 验证。核心用例见
[Schema 回填与升级](../../../test-cases/cloud/persistence/migration-backfill.md)。
