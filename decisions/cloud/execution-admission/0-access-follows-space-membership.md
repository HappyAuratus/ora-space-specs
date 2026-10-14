---
status: implemented
---

# 执行访问准入跟随 Space 成员身份

> 后续已实现决策：[独占操作准入](20260927-exclusive-runtime-control.md)增加操作会话、冲突写活动和安全交接，后端状态为 `implemented`。成员资格仍是前提，不能替代新增准入；UI 与完整故障验收由后续 ADR 承接，本文保留原状态。

> 历史决策：已准入任务按原生命周期收尾的规则保留；下文的独立空间成员表已由 [20260923-one-space-per-tenant](../tenancy/20260923-one-space-per-tenant.md) 取代。当前新访问与准入核验活动租户成员身份。

`/internal/v1/access` 与 `/internal/v1/admissions` 的用户侧授权从「project owner 专属」改为
「project 所属 Space 的 active member」。准入判定本身（`admissionOpen`、desired/observed
state、lifecycle active、Node 30 秒在线）与 Execution Ticket 的 actor 绑定、controller
lease/epoch fencing、idle 证据全部不变；本决策只改变「谁可以被准入」的边界。

动机：多人协作要求 Space 成员能启动/停止共享 Project 的运行时，owner 专属会阻断核心协作
场景。共享只发生在可见性与准入层：被准入者身份仍写入 `actor_user_id`，operations 仍按 actor
作用域隔离，成员不能读取他人的 operation。

本决策已实现并随 Project Space 化一并验证，`status: implemented` 表示规则与实现同步。
controller-integration 域（lease、epoch、接管边界）不受影响。

## D1：授权链

准入授权链为：workspace → 其 project → `project.space_id` → `spaceMember` 判定（active
tenant membership + active user + active tenant + 未归档 Space）。归档 Space 的 Project 不再
产生新的 access；租户成员资格被禁用或移除的用户同样被拒绝（tenant membership 前置条件）。

## D2：不变部分

- controller lease 获取/续期与 epoch 校验不变；ticket 幂等（`idempotency_conflict`）不变。
- 准入的运行时条件（admissionOpen、状态机、Node 心跳）不变，成员身份只是必要条件。
- 用户凭证要求不变：`kind=user` JWT 且 `caller` 绑定 service subject。

## 不变量

1. 只有 project 所属 Space 的 active member 能通过 access/admissions 获得执行准入。
2. Ticket 的 actor 绑定与 operations 作用域不因共享可见性而共享。
3. 归档 Space 后其 Project 不再产生新准入。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 放宽到任意 tenant member | 跨 Space 泄漏执行能力，可见性边界与执行边界分裂 |
| 保持 owner 专属 | Space 成员只能看不能跑，多人协作核心场景不可用 |
| 准入时复制成员快照到 ticket | 引入第二份权威，成员变化后准入状态漂移 |

## 落地与验收

实现位于 `internal/core/node.go` 的 `access`（经 `workspace`/`project` 授权改造）。共享可见性
由 `TestIdentityConcurrencyMembershipAndIsolation` 验证（tenant 成员加入默认 Space 后可读
Project 与运行时 workspace）；成员被禁用后的拒绝路径由
`TestRemainingPublicContractsAndMembershipRevocation` 验证。核心用例见
[执行准入成员身份](../../../test-cases/cloud/execution-admission/access-membership.md)。
