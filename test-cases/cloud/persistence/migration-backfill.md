# 协作 Space Schema 回填与单空间迁移

0006/0007 的默认空间回填仍是不可变的历史步骤；0008 收紧为单空间租户和全局 slug。当前只有测试数据，多个租户共用 `default` slug 或一个租户含多个空间的旧测试库需重建，不能静默拆分。

决策依据：[协作 Space 的 Schema、确定性回填与迁移不可变性](../../../decisions/cloud/persistence/0-collab-space-schema-and-backfill.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/cloud_test.go`）验证。

## Upgrading a pre-space database binds every project to a default space

- 风险：0007 回填遗漏 Project，或回填顺序错误导致 NOT NULL 收紧失败、升级中断。
- 前置状态：0001–0003 建库并灌入 tenant、admin membership、active Project、workspace、
  sandbox、node、ticket、operation、effect 种子数据。
- 触发：`store.Migrate` 升级到最新（含 0006–0008）并 `CheckSchema`，随后校验兼容行保真。
- 必须成立：升级成功；外部 effect 意图与 ticket tenant 回填保持；既有业务行不重建。
- 禁止结果：迁移失败（NOT NULL 收紧即证明回填覆盖全部 Project）、checksum 不符、历史数据丢失。
- 验证义务与证据：**Covered** — `integration TestMigrateUpgradesPreviousSchemaAndData`
  以旧 schema 种子数据升级到最新（含 0008）并通过 checksum 与数据校验。

## Previous space schema upgrades only when tenant and space agree

- 风险：0008 静默把旧空间名称改为租户名称，或在失败时留下部分 schema 变更和错误的 migration 记录。
- 前置状态：0001–0007 已应用，租户有一个旧空间和独立空间成员关系。
- 触发：分别以同名和不同名的租户/空间运行 0008，再重复兼容库的迁移。
- 必须成立：兼容库保留身份、slug 和租户管理员，删除旧空间成员表，重复迁移安全；不兼容库明确失败，原空间与迁移历史不变。
- 禁止结果：静默覆盖名称、失败后记录 0008 已应用、旧成员表继续作为第二权限来源。
- 验证义务与证据：**Covered** — `integration TestMigrateFromPreviousSpaceSchemaPreservesCompatibleTenant`、`TestMigrateRejectsIncompatiblePreviousSpaceNames`。

## Default-space backfill is deterministic and needs no online IDaaS

- 风险：回填依赖外部身份服务或非确定性排序，导致不可重入或各 tenant 结果不一致。
- 前置状态：0007 升级前的历史数据；0008 之后该数据形状只允许全局唯一 slug。
- 触发：重复执行回填路径；观察 created_by 与成员映射。
- 必须成立：0007 历史回填为每 tenant 创建默认 Space，并按当时规则映射成员；0008 不重新解释这些已应用迁移，只对兼容数据收紧唯一约束。
- 禁止结果：重复执行产生第二个 Space、在 0008 静默删除或改写不兼容历史行。
- 验证义务与证据：**Partial** — 单 tenant 升级路径由
  `TestMigrateUpgradesPreviousSchemaAndData` 覆盖；多 tenant 与 created_by 回退分支的自动化
  断言缺失（手动 psql 检查见 test.md 2.4 节）。

## Bootstrap creates tenant, membership, and default space atomically

- 风险：Bootstrap 只建 tenant 与管理员而不建唯一 Space，新租户违反单空间不变量；
  或分步提交留下半初始化租户。
- 前置状态：空库。
- 触发：`store.Bootstrap` 创建租户；随后新用户加入 tenant 并读取默认 Space 的共享 Project。
- 必须成立：同一事务产出 tenantId/userId/spaceId；加入 tenant 后立即经唯一 Space 获得共享
  可见性（200）。
- 禁止结果：唯一 Space 缺失、或成员加入后共享读取失败。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  的 fixture 全部经 Bootstrap 建立，且断言成员加入后共享 Project/workspace 读取 200，
  直接依赖 Bootstrap 的默认 Space 与运行时同步路径。

## Migrations are immutable, checksummed, and upgrades are reentrant

- 风险：迁移文件被事后改写，历史部署 checksum 破裂；或迁移不可重入导致部署中断。
- 前置状态：已应用 0001–0007 的库。
- 触发：重新执行 `Migrate` 与 `CheckSchema`；篡改任一已应用迁移内容。
- 必须成立：重复迁移成功（无重复 DDL）；checksum 逐条核对；篡改被检出。
- 禁止结果：重复迁移失败、已应用文件静默改变。
- 验证义务与证据：**Covered** — `TestMigrateUpgradesPreviousSchemaAndData` 每次运行
  `CheckSchema` 校验全部迁移 checksum；可重入性由 `task migrate` 连续两次成功的手册步骤
  覆盖（test.md 1.3）。
