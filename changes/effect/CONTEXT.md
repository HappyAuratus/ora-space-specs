# Effect v2 设计交接背景

## 1. 文档用途

本文用于将 Effect v2 数据库设计讨论交接给后续设计者。已确认表的完整 SQL 和设计
结论记录在同目录的 `v2.md`；本文先解释重构背景和跨表约束，再为尚未逐表确认的表
提供初步设计。后半部分均为讨论起点，不是最终结论。

原始第一版方案见 `plan.md`，当前 Skill 专属迁移见
`crates/db/src/migration/schema_v0006.rs`。

## 2. 为什么设计 v2

v1 的 Effect 基础设施目标是通用的，但数据库核心表直接使用了 `skill_name`、
`skill_md_digest`、Skill package path 和 `desired_skills`/`managed_skills` 等字段与表名。
这会迫使 MCP 及后续 Effect kinds 复制一套 Desired、Managed、传播和状态表。

v2 将稳定生命周期放入通用 Effect Core：Source、Revision、Desired、Surface、Managed、
Status、Condition、Request 和 Operation。Skill、MCP 等 adapter 只负责类型专属校验、
payload、target mapping、扫描、fingerprint、ownership marker 和物化。

## 3. 已确认的统一约定

### 3.1 数据库约定

- Effect 自己拥有的实体使用 UUID `TEXT` 主键；纯一对一扩展或关联关系使用父键或复合
  主键，不额外创建等价 UUID。
- UUID 是实体身份，业务键使用 `UNIQUE` 单独约束。例如 Source UUID 与 SourceKey 是
  两个概念。
- 本次设计不为 `TEXT` 字段添加数据库 `CHECK`。允许值、格式和非空字符串要求写在 SQL
  注释中，由领域类型与 repository 写入边界验证。
- generation、revision counter 和时间关系等非文本约束仍可使用 `CHECK`。
- `Observed` 和 `Preserved` 是实时扫描结果，不持久化为数据库事实。
- 当前状态与历史分开：Status/Condition 保存当前事实，Audit 保存历史。

### 3.2 粒度约定

Effect Source 是最小可选择单元，Managed target 是最小 ownership 与 mutation 单元。
一个 Plugin package 可以提供多个 Effect Sources，一个 Surface 可以包含多个 Managed
targets；Plugin package 和 Surface 都不是选择或 ownership 的最小粒度。

例如一个 `official-tools` Plugin 同时包含 `formatter` 和 `reviewer` 两个 Skill，应创建
两个 Source。Workspace 可以只为 `formatter` 创建 Desired item，只物化对应 Managed
target。对于 Plugin Source，`namespace` 必须是全局稳定的 Plugin ID/namespace，不能只
是可能发生冲突的宽泛 publisher 名称。

Filesystem Skill 的层级为：

```text
.agents/skills                    Surface
├── formatter/                    Managed target
│   ├── SKILL.md
│   └── .ora-managed.json
├── reviewer/                     Managed target
└── user-created-skill/           Preserved target
```

Surface 是可扫描、可协调的物化容器；target 是容器内最小 ownership unit。Skill 目录内
的文件不分别成为 Managed items。

### 3.3 身份与状态流

```text
Plugin package / Local catalog
        │
        ├── Effect Source                 stable selectable identity
        │       ├── Source Revision R1    exact immutable content state
        │       ├── Source Revision R2
        │       └── Source Head ────────→ current Revision
        │
WorkspaceEffect generation
        └── Desired item ───────────────→ exact Source Revision
                    │
                    ├── Surface A ──────→ Managed target
                    └── Surface B ──────→ Managed target
```

- SourceKey 是 `(effect_kind, source_kind, namespace, identifier)`；Source row 另有 UUID。
- Revision 另有 UUID，来源提供的 `revision` 是不透明字符串，只做完全相等比较。
- 同一 `(source_id, revision)` 再次登记属于幂等重试；digest/payload 不一致是制品完整性
  错误。
- Head 显式指向当前 Revision，不能按 revision 字符串排序推导。
- Desired item 持久化确切 `revision_id`，不能在读取时动态跟随 Head，否则 Desired 会在
  Workspace generation 不变时发生变化。
- Source Head 传播更新 Desired `revision_id` 时保留 Desired item UUID，并通过
  generation CAS 推进 Workspace generation。

### 3.4 generation 与收敛

- `workspace_effects.generation` 属于完整 Desired Effect 集合，Skill、MCP 等共享同一
  generation。
- 规范化 Desired 集合变化才推进 generation；相同提交是 no-op。
- Managed、Observed、Surface/Consumer status 变化不推进 Workspace generation。
- Desired replace、generation CAS、Desired rows 更新和 reconcile request upsert 必须
  在同一个数据库事务完成。
- 不同 Surface 独立收敛；同一 Surface 严格串行。

### 3.5 Surface 与 Consumer

- Surface 使用 UUID 身份；`locator_key` 用于规范化目标相等和 collision，
  `locator_json` 用于 adapter 实际访问与恢复。
- `locator_json` 和 `format_kind` 创建后不可修改。目标位置或 format 改变时，旧 Surface
  Retiring，新 Surface 使用新 UUID。
- 同一物理目标不能存在两个 writer。同 locator、兼容 format 的 Consumer 合并；format
  不兼容时报告 `IncompatibleSurfaceDeclarations`。
- Consumer 是读取 Surface 结果并可能需要在 mutation 前后协调的运行时组件。Surface
  采用所有 Consumer 中最严格的协调策略。
- 最后一个 Consumer 消失后 Surface 进入 Retiring，不能立即删除。

### 3.6 ownership 安全边界

- Managed item UUID 就是 ManagedIdentity，并写入目标侧 ownership marker。
- 只有数据库 ledger、目标 marker 和 `applied_fingerprint` 同时匹配，Effect 才能覆盖或
  删除目标。
- Managed item 不引用 Desired item，因为 Desired 删除后 ledger 必须保留到安全清理
  完成；Reconciler 使用 `source_id` 与当前 Desired 匹配。
- Managed item 的 Surface、Source、Revision 外键不 cascade。必须先验证并清理目标，
  再删除 ledger，最后删除父记录。
- Desired item 可以 cascade 删除，因为 ledger 不依赖它；Surface Consumer 可以
  cascade 删除，因为它只是声明关系，不是 ownership 证据。
- 无法在目标侧保存并恢复 ManagedIdentity 的 format 不能成为安全 Managed adapter；其
  已有内容只能视为 Preserved。

## 4. 已确认的表

以下表已在 `v2.md` 中逐表确认：

1. `effect_sources`
2. `effect_source_revisions`
3. `effect_source_heads`
4. `workspace_effects`
5. `effect_surfaces`
6. `effect_surface_consumers`
7. `workspace_effect_desired_items`
8. `effect_managed_items`

原先候选的 `effect_kinds` 已删除；Effect kinds 由领域枚举显式支持。原先候选的
`effect_desired_item_sources` 也已删除；Desired row 直接保存一个 `source_id` 和属于该
Source 的确切 `revision_id`。

# 剩余表的初步设计

以下设计尚未逐表确认。接手者应继续按当前顺序讨论，并在确认后将最终结论写回
`v2.md`。

## 5. `effect_surface_status` — Surface 收敛状态

初步职责：保存一个 Surface 对当前 Workspace Desired generation 的观察和应用进度。
每个 Surface 一条状态记录，创建 Surface 时同步创建。

```sql
CREATE TABLE effect_surface_status (
    surface_id          TEXT PRIMARY KEY
                        REFERENCES effect_surfaces(id) ON DELETE CASCADE,
    desired_generation  INTEGER NOT NULL DEFAULT 0,
    observed_generation INTEGER NOT NULL DEFAULT 0,
    applied_generation  INTEGER NOT NULL DEFAULT 0,
    phase               TEXT NOT NULL, -- pending | waiting_for_idle | quiescing |
                                       -- applying | resuming | current | degraded |
                                       -- retiring | recovery_required
    revision            INTEGER NOT NULL DEFAULT 1,
    created_at          INTEGER NOT NULL,
    updated_at          INTEGER NOT NULL,
    CHECK (desired_generation >= 0),
    CHECK (observed_generation >= 0),
    CHECK (applied_generation >= 0),
    CHECK (applied_generation <= observed_generation),
    CHECK (observed_generation <= desired_generation),
    CHECK (revision > 0),
    CHECK (updated_at >= created_at)
);
```

初步语义：

- `desired_generation` 是该 Surface 当前应收敛到的 Workspace generation。
- `observed_generation` 表示 Reconciler 已针对哪个 generation 完成事实读取和扫描，不
  表示把 Observed snapshot 落库。
- `applied_generation` 只有在整个 Surface 对该 generation 的安全 mutations 完成后推进。
- `revision` 是状态 watch 的单独版本；任何 phase、generation 或 Condition 变化都在同一
  事务中推进它。
- Conditions 不再保存为 JSON，由 `effect_conditions` 规范化保存。
- Surface 只有在 Managed/Operation 已安全结束后才能删除，因此 status cascade 不会
  绕过 ownership。

待确认：Retiring Surface 是否仍沿用 Workspace generation，还是需要一个独立 retire
revision；初步倾向继续使用最后一次确认的 Workspace generation，并通过 phase 表达
Retiring。

## 6. `effect_consumer_status` — Consumer 就绪状态

初步职责：分别记录共享 Surface 的每个 Consumer 已恢复到哪个 generation。文件已经
应用但某个 Consumer 恢复失败时，Surface applied 可以推进，该 Consumer ready 不推进。

```sql
CREATE TABLE effect_consumer_status (
    surface_id      TEXT NOT NULL,
    consumer_id     TEXT NOT NULL,
    ready_generation INTEGER NOT NULL DEFAULT 0,
    phase           TEXT NOT NULL, -- pending | quiesced | resuming | ready | degraded
    revision        INTEGER NOT NULL DEFAULT 1,
    created_at      INTEGER NOT NULL,
    updated_at      INTEGER NOT NULL,
    PRIMARY KEY (surface_id, consumer_id),
    FOREIGN KEY (surface_id, consumer_id)
        REFERENCES effect_surface_consumers(surface_id, consumer_id)
        ON DELETE CASCADE,
    CHECK (ready_generation >= 0),
    CHECK (revision > 0),
    CHECK (updated_at >= created_at)
);
```

初步语义：

- 声明 Consumer 时同步创建状态 row；永久撤销声明时可以 cascade 删除。
- 暂时启动、quiesce 或 resume 失败不能删除 Consumer 声明，只更新 phase、revision 和
  Conditions。
- `ready_generation` 不能超过 Surface applied generation；该跨表约束由 repository
  transaction 验证，SQLite 不直接表达。
- 不保存 Consumer Session 列表；协调器在执行时读取实时 runtime 状态。

## 7. `effect_conditions` — 当前条件与阻塞原因

初步职责：保存 Surface/Consumer status 当前仍成立的结构化条件。条件解除后删除当前
row；历史写入 Audit。

```sql
CREATE TABLE effect_conditions (
    id                TEXT PRIMARY KEY, -- UUID
    surface_id        TEXT NOT NULL
                      REFERENCES effect_surfaces(id) ON DELETE CASCADE,
    scope_kind        TEXT NOT NULL, -- surface | consumer
    scope_id          TEXT NOT NULL, -- surface UUID or consumer identity
    subject_kind      TEXT NOT NULL, -- desired_item | managed_item | surface | consumer |
                                     -- source_revision | operation
    subject_id        TEXT NOT NULL,
    reason            TEXT NOT NULL,
    retry_policy      TEXT NOT NULL, -- on_change | backoff | manual
    failed_generation INTEGER,
    message           TEXT NOT NULL,
    first_observed_at INTEGER NOT NULL,
    last_observed_at  INTEGER NOT NULL,
    UNIQUE (
        surface_id,
        scope_kind,
        scope_id,
        subject_kind,
        subject_id,
        reason
    ),
    CHECK (failed_generation IS NULL OR failed_generation >= 0),
    CHECK (last_observed_at >= first_observed_at)
);
```

初步语义：

- `scope_*` 表示条件属于 Surface status 还是某个 Consumer status；`subject_*` 表示具体
  被阻塞对象。
- `message` 只用于安全可读说明，调用方只依赖稳定的 `reason` 和 `retry_policy`。
- message 不得保存敏感路径、secret 或制品内容；详细 error chain 仅写日志。
- 条件 upsert/remove 与对应 status `revision` 更新必须在同一事务完成。

待确认：这张表使用通用 subject key，数据库无法为所有 subject 建立强外键。可选替代是
拆成 Surface Conditions 与 Consumer Conditions，或为不同 subject 建扩展表。初步方案
选择通用表，由领域类型和 repository 校验 `scope_kind/scope_id`、subject 与 Surface 的
一致性。

## 8. `effect_reconcile_requests` — Durable Surface 唤醒

初步职责：持久化 Surface 需要重新扫描/reconcile 的事实，并把连续唤醒合并成一条 row。

```sql
CREATE TABLE effect_reconcile_requests (
    surface_id           TEXT PRIMARY KEY
                         REFERENCES effect_surfaces(id) ON DELETE CASCADE,
    requested_generation INTEGER NOT NULL,
    request_revision     INTEGER NOT NULL DEFAULT 1,
    attempt_count        INTEGER NOT NULL DEFAULT 0,
    requested_at         INTEGER NOT NULL,
    not_before_at        INTEGER NOT NULL,
    updated_at           INTEGER NOT NULL,
    CHECK (requested_generation >= 0),
    CHECK (request_revision > 0),
    CHECK (attempt_count >= 0),
    CHECK (updated_at >= requested_at)
);
```

初步语义：

- Desired、Surface descriptor、filesystem watcher、Consumer 或显式 retry 都可以 upsert
  请求。
- `requested_generation` 合并为最新 generation；同 generation 的 watcher 唤醒仍推进
  `request_revision`，避免被正在运行的 worker 错误吞掉。
- Worker 读取 `(surface_id, request_revision)` 快照，成功后只执行带 revision CAS 的
  delete。处理期间出现新唤醒时，旧 worker 无法删除新请求。
- `attempt_count/not_before_at` 支持 durable backoff；显式 retry 可以重置 backoff，但不能
  修改 Desired 或绕过 ownership/recovery 冲突。
- 第一版单进程，不在此表提前加入 lease/owner 字段。

## 9. `effect_propagation_requests` — Durable Source 传播

初步职责：Source Head 更新时，可靠唤醒 worker，将仍引用旧 Revision 的 Workspace
Desired items 推进到最新 Head。

```sql
CREATE TABLE effect_propagation_requests (
    source_id        TEXT PRIMARY KEY REFERENCES effect_sources(id) ON DELETE CASCADE,
    head_revision_id TEXT NOT NULL,
    request_revision INTEGER NOT NULL DEFAULT 1,
    attempt_count    INTEGER NOT NULL DEFAULT 0,
    requested_at     INTEGER NOT NULL,
    not_before_at    INTEGER NOT NULL,
    updated_at       INTEGER NOT NULL,
    FOREIGN KEY (source_id, head_revision_id)
        REFERENCES effect_source_revisions(source_id, id),
    CHECK (request_revision > 0),
    CHECK (attempt_count >= 0),
    CHECK (updated_at >= requested_at)
);
```

初步语义：

- 新 Revision、Head 更新和 request upsert 必须在同一个 Source write transaction 提交。
- 连续 R1 → R2 → R3 更新合并为最新 Head；worker 每次仍重新读取 Source Head，请求 payload
  只是 durable wakeup 和 CAS token。
- Worker 按 Workspace 独立执行 generation CAS，只更新仍引用该 Source 旧 Revision 的
  Desired item；已删除或替换的用户选择不恢复。
- 部分 Workspace 成功后崩溃可以安全重跑；已到最新 Revision 的 Desired item 是 no-op。
- 与 reconcile request 相同，`request_revision` 防止处理期间的新请求被旧 worker 删除。

待确认：Source 是否允许物理删除。当前 Source 有 `retired` 生命周期并被 Desired/Managed
外键保护，因此预计长期保留；若禁止物理删除，应去掉这里的 `ON DELETE CASCADE` 以保持
一致。

## 10. `effect_operations` — Durable mutation journal

初步职责：记录一个 Managed target 的跨数据库/外部目标 mutation，使进程崩溃后能够从
Prepared、Applied 或 Finalized 边界恢复。

```sql
CREATE TABLE effect_operations (
    id            TEXT PRIMARY KEY, -- UUID
    surface_id    TEXT NOT NULL REFERENCES effect_surfaces(id),
    generation    INTEGER NOT NULL,
    target_key    TEXT NOT NULL,
    operation_kind TEXT NOT NULL, -- create | update | replace | delete
    phase         TEXT NOT NULL,   -- prepared | applied | finalized
    payload_json  TEXT NOT NULL,   -- adapter plan plus ownership identity transition
    prepared_at   INTEGER NOT NULL,
    applied_at    INTEGER,
    finalized_at  INTEGER,
    updated_at    INTEGER NOT NULL,
    CHECK (generation >= 0),
    CHECK (updated_at >= prepared_at)
);

CREATE UNIQUE INDEX effect_operations_active_target_unique
    ON effect_operations(surface_id, target_key)
    WHERE phase <> 'finalized';
```

初步语义：

- 每个 target 同一时刻最多有一个未 Finalized operation；同一 Surface 的 worker 仍在应用
  层严格串行。
- `payload_json` 由 adapter 定义，至少保存 previous/planned fingerprint、旧/新
  ManagedIdentity、Revision、可验证 mutation plan 及恢复判断所需信息。
- create/replace 的 planned ManagedIdentity 可能尚未存在于 ledger，delete 的 ledger 在
  finalize 后消失，因此 operation 不强行使用一个含义模糊的 Managed item 外键。
- 当前目标匹配 previous fingerprint 时安全重试；匹配 planned fingerprint 时完成
  Finalize；两者均不匹配时进入 `RecoveryRequired`。
- Operation 与 Surface/受控 artifacts 之间不 cascade。Surface 只能在 operations 和
  Managed items 都结束后删除。
- Finalized row 的保留/归档周期待确认；业务历史最终由 Audit 保存。

## 11. `effect_operation_paths` — 受控恢复资源（名称待确认）

当前标题来自 filesystem v1，但 v2 Surface adapter 不一定操作路径。初步建议在正式确认
时将表名改为 `effect_operation_artifacts`，避免 Effect Core 再次被限制为 filesystem。

```sql
CREATE TABLE effect_operation_artifacts (
    id                   TEXT PRIMARY KEY, -- UUID
    operation_id         TEXT NOT NULL REFERENCES effect_operations(id),
    artifact_role        TEXT NOT NULL,    -- staging | backup
    locator_key          TEXT NOT NULL,
    locator_json         TEXT NOT NULL,    -- exact adapter-owned recovery locator
    expected_fingerprint TEXT,
    cleanup_state        TEXT NOT NULL,    -- retained | pending_cleanup | cleanup_failed
    created_at           INTEGER NOT NULL,
    updated_at           INTEGER NOT NULL,
    UNIQUE (operation_id, artifact_role, locator_key),
    CHECK (updated_at >= created_at)
);
```

初步语义：

- 只允许恢复、删除或清理 durable row 精确记录的 locator；不得扫描并批量删除未知同名
  staging/backup 资源。
- Filesystem staging/backup 必须与目标位于支持原子 swap 的同一文件系统；具体 containment
  和 reparse/symlink 安全由 adapter 与共享 path 工具验证。
- `expected_fingerprint` 用于确认 artifact 没有被外部替换。
- Finalized 后 backup cleanup 可以重试；cleanup 失败不回滚已成功的业务状态。
- Operation 不应在 artifacts 清理或显式移交前删除，因此外键不 cascade。

待确认：非 filesystem adapter 是否也需要 durable artifacts。如果所有非文件操作都能将
恢复数据完整放入 operation payload，可以保留原表名并仅供 filesystem adapter 使用；
否则推荐采用 `effect_operation_artifacts`。

## 12. `effect_audit_events` — 追加式历史事件

初步职责：保存已发生的业务历史，不参与当前 reconcile 事实判断。

```sql
CREATE TABLE effect_audit_events (
    id           TEXT PRIMARY KEY, -- UUID
    workspace_id TEXT,             -- snapshot UUID; absent for global Source events
    surface_id   TEXT,             -- snapshot UUID
    subject_kind TEXT NOT NULL,
    subject_id   TEXT NOT NULL,
    event_kind   TEXT NOT NULL,
    generation   INTEGER,
    payload_json TEXT NOT NULL,
    occurred_at  INTEGER NOT NULL,
    CHECK (generation IS NULL OR generation >= 0)
);

CREATE INDEX effect_audit_events_workspace_time
    ON effect_audit_events(workspace_id, occurred_at, id);

CREATE INDEX effect_audit_events_subject_time
    ON effect_audit_events(subject_kind, subject_id, occurred_at, id);
```

初步语义：

- Audit append-only，不 update 已写事件。
- `workspace_id`、`surface_id` 和 subject 使用快照 identity，不建立父表外键，使 Audit 可以
  在业务实体安全删除后继续保留。
- `payload_json` 保存结构化、安全、可演进的事件细节，但不得包含 secret、敏感绝对路径
  或制品原文。
- Current phase、Condition 和 retry 状态不得从 Audit 回放推导；它们仍由 Status、
  Condition、Request 和 Operation 表提供当前事实。

待确认：Audit 的保留期、容量上限、清理策略，以及 Source 全局事件是否需要单独的
`source_id` 可查询列。

## 13. 建议的后续讨论顺序

1. 先确认 Surface/Consumer status 的 generation 与 revision 语义。
2. 决定 Conditions 保持通用 subject，还是拆表换取更强外键。
3. 确认两个 durable request 的同 generation 唤醒和 CAS delete 协议。
4. 用 create/update/replace/delete 各走一遍 crash recovery，收敛 Operation payload。
5. 决定 `effect_operation_paths` 是否改为 `effect_operation_artifacts`。
6. 最后确定 Audit 保留与查询需求，避免它反向承担当前状态职责。
