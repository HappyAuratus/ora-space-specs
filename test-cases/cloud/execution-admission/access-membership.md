# 执行准入的成员身份边界

决策依据：[执行访问准入跟随 Space 成员身份](../../../decisions/cloud/execution-admission/0-access-follows-space-membership.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/cloud_test.go`、
`integration/endpoints_test.go`）验证。

后续已批准规则的核心用例为[运行时内容权限](../workspace/runtime-workspace-use-permissions.md)与[独占准入](exclusive-runtime-control.md)。成员身份继续是必要条件，但不再足以获得他人运行时内容或写准入；以下证据只按明确义务保留，不作为新独占保证。

## Access admission follows tenant membership instead of project ownership

- 风险：准入仍按 owner 过滤，租户成员无法运行共享 Project；或遗漏活动租户成员检查，
  让非成员获得执行能力。
- 前置状态：alice 在租户唯一 Space 创建 Project 并推进到 active；new-user 加入 tenant
  成为 member，但不是项目创建者。
- 触发：new-user 读取共享 Project、运行时安全概况，并尝试读取内容或建立执行访问。
- 必须成立：共享 Project 与安全概况按成员身份可见；非 tenant 成员被拒。内容需要创建者或当前管理员身份，写准入另需独占与活动校验；既有 admissionOpen、Node 条件仍必要，owner 不代替这些判断。
- 禁止结果：非成员获得可见性或准入、成员因为 owner 不同无法读取共享项目、成员身份直接授予他人运行时操作权。
- 验证义务与证据：**Partial** — `integration TestIdentityConcurrencyMembershipAndIsolation` 直接证明前序租户成员门槛；其中成员对他人运行时 execute access 200 的旧断言须按新权限更新，不是当前目标的通过证据。新增权限和独占义务为 `Missing`，见上述新核心用例。

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
- 必须成立：共享项目 operation 的安全概况按成员可见，关联运行时的受限内容按创建者/管理员校验；行政停止仍需管理员且使用受限投影。
- 禁止结果：非成员读取 operation、无内容权限的成员读取执行详情、普通成员读取行政停止结果、受限投影暴露请求/结果细节。
- 验证义务与证据：**Partial** — `integration TestIdentityConcurrencyMembershipAndIsolation` 证明原租户与行政停止投影边界；新增运行时详情过滤尚无直接证据，见运行时内容权限核心用例。
