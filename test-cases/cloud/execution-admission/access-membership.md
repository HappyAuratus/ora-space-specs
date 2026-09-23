# 执行准入的成员身份边界

决策依据：[执行访问准入跟随 Space 成员身份](../../../decisions/cloud/execution-admission/0-access-follows-space-membership.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/cloud_test.go`、
`integration/endpoints_test.go`）验证。

## Access admission follows space membership instead of project ownership

- 风险：准入仍按 owner 过滤，Space 成员无法运行共享 Project；或反向放宽到 tenant 级，
  跨 Space 泄漏执行能力。
- 前置状态：alice 在默认 Space 创建 Project 并推进到 active；new-user 加入 tenant
  （自动成为默认 Space member）但非 owner。
- 触发：new-user 读取该 Project 与其运行时 workspace，并尝试建立执行访问。
- 必须成立：读取均 200（共享可见性）；非 tenant 成员被 403 拒绝；准入的运行时条件
  （admissionOpen、Node 心跳等）仍是必要条件。
- 禁止结果：非成员获得可见性或准入、成员被 owner 检查拒绝。
- 验证义务与证据：**Partial** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  直接断言共享读取 200（准入授权链的公开侧）；非 owner Space member 直调
  `/internal/v1/access` 尚无独立测试，公开侧通过依赖同一条 `workspace`/`project` 授权链。

## Disabled or removed users cannot obtain execution access

- 风险：成员资格被撤销后仍能通过缓存或旧凭证获得准入。
- 前置状态：member 创建 Project 并推进到 active，随后其 tenant membership 被禁用。
- 触发：member 读取运行时 workspace 并调用 `/internal/v1/admissions`。
- 必须成立：两者均被拒绝（403），不产生新 ticket。
- 禁止结果：禁用成员读到 workspace 或建立执行 ticket。
- 验证义务与证据：**Covered** — `integration
  TestRemainingPublicContractsAndMembershipRevocation` 断言禁用后 workspace 读取与
  admissions 均 403。

## Shared visibility never shares operation scope

- 风险：Space 化把 operation 作用域也共享化，成员可读取他人 operation 的敏感载荷。
- 前置状态：new-user 加入 tenant 后可读 alice 的 Project。
- 触发：new-user `GET /operations/:oid`（alice 创建 Project 的 operation）。
- 必须成立：404，operation 仍按 actor 作用域隔离。
- 禁止结果：非 actor 读到 operation。
- 验证义务与证据：**Covered** — `integration TestIdentityConcurrencyMembershipAndIsolation`
  断言共享 Project 可见的同时 operation 读取 404。
