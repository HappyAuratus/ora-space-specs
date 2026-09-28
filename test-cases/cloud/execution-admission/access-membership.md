# 执行准入的成员身份边界

决策依据：[执行访问准入跟随 Space 成员身份](../../../decisions/cloud/execution-admission/0-access-follows-space-membership.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/cloud_test.go`、
`integration/endpoints_test.go`）验证。

## Access admission follows tenant membership instead of project ownership

- 风险：准入仍按 owner 过滤，租户成员无法运行共享 Project；或遗漏活动租户成员检查，
  让非成员获得执行能力。
- 前置状态：alice 在租户唯一 Space 创建 Project 并推进到 active；new-user 加入 tenant
  成为 member，但不是项目创建者。
- 触发：new-user 读取该 Project 与其运行时 workspace，并尝试建立执行访问。
- 必须成立：读取均 200（共享可见性）；非 tenant 成员被 403 拒绝；准入的运行时条件
  （admissionOpen、Node 心跳等）仍是必要条件。
- 禁止结果：非成员获得可见性或准入、成员被 owner 检查拒绝。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  断言非成员内部 access 403、成员对他人项目的内部 execute access 200。

## Disabled or removed users cannot obtain execution access

- 风险：成员资格被撤销后仍能通过缓存或旧凭证获得准入。
- 前置状态：member 创建 Project 并推进到 active，随后其 tenant membership 被禁用。
- 触发：member 读取运行时 workspace 并调用 `/internal/v1/admissions`。
- 必须成立：两者均被拒绝（403），不产生新 ticket。
- 禁止结果：禁用成员读到 workspace 或建立执行 ticket。
- 验证义务与证据：**Covered** — `integration
  TestRemainingPublicContractsAndMembershipRevocation` 断言禁用后 workspace 读取与
  admissions 均 403。

## Shared operations stay tenant-scoped and administrative results stay restricted

- 风险：共享 operation 仍被项目创建人检查拒绝，或行政停止的受限投影泄漏基础设施细节。
- 前置状态：new-user 加入 tenant 后可读 alice 的 Project。
- 触发：new-user `GET /operations/:oid`（alice 创建 Project 的 operation）。
- 必须成立：同租户有效成员可读取共享 Project 的 operation；行政停止仍需管理员且使用受限投影。
- 禁止结果：非成员读取 operation、普通成员读取行政停止结果、受限投影暴露请求/结果细节。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  断言共享 Project 与 operation 均可读，同时行政停止结果使用受限投影。
