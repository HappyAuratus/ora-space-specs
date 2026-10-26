# 无删除操作的 D8 残留：显式接管、真实来源与两阶段处置

依据[已实现的 D8 补充决策](../../../decisions/cloud/issue-run/20261025-d8-residual-without-delete-operation.md)，下文 D 编号指该 ADR。既有[残留回收用例](residual-resource-reclamation.md)覆盖原失败删除操作存在的 D6；其证据不自动覆盖本变体。

Cloud 实现见 [cloud#173](https://github.com/ora-space/cloud/pull/173)，以下直接测试固定在 `1b772b442d2346aa19e85e80b40438faf0cc0bed`。数据库测试执行真实 D8 worker 与 Cloud 事务，终止及移除成功结果由测试提交，不运行真实 Controller/Substrate，也不证明真实卷或原任务已恢复。10 项完整义务中 8 项 `Partial`、2 项 `Missing`；状态按每项完整要求判断，不把较窄直接断言提升为完整覆盖。

未来产生路径的自动登记/正常回收由 [circus#643](https://github.com/ora-space/circus/issues/643) 维护，不在本历史入口用例中冒充已实现。

## Missing Delete D8 Takeover Requires Explicit Exact Original Scope

- 风险：默认命令扩大到尚未终结、错绑或已有删除责任的资源；查询和租户入口成为隐式接管旁路。
- 前置状态：真实 D8 worker 在没有任何 `delete_workspace` 的情况下形成 `done/workspace_unavailable`；保留原 tenant/Project/Run/隔离 Workspace/代次与运行时身份，admission 关闭，无残留账本，Project active 且无在途操作。
- 触发：默认终止、显式选择 `--allow-missing-delete-operation`；缺 Run/Workspace、混入 operation-id、错误 tenant/Project/Run/代次、非终态或非 D8、缺交付结论、admission open、数据已释放、已有删除操作/残留/强停/回收责任；其他命令携带该选项；租户访问隐藏 Workspace。
- 必须成立：默认拒绝新变体；仅部署数据库运维身份带精确原资源、reason、幂等键及真实影响确认时受理；事务内重读全部资格；矛盾身份或责任拒绝且不遗留半份意图。已有删除历史继续要求原失败删除身份，查询不授予权限，HTTP 边界不变。
- 禁止结果：只凭 Project/tenant 猜测目标；选项绕过既有删除/账本责任；放宽公开 API；拒绝后留下副作用计划。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 合法状态的产生与默认拒绝 | [直接回归][missing-db] `TestControlledD8MissingDeleteRequiresExplicitScopeAndKeepsHistory` 使用真实 D8 worker，默认拒绝、显式成功且原删除历史为空；尚未在默认拒绝前后比较完整 Run/资源/意图快照 | Partial |
| 显式精确准入与拒绝矩阵 | 同文件 `TestControlledD8MissingDeleteRejectsOtherResources` 覆盖缺 Run/Workspace、异租户/错误 Run、非 D8/终态、缺交付结论、admission open 和已有原删除，拒绝后意图数为零；[CLI 测试][cli] `TestCloudctlResidualManagement` / `TestCloudctlResidualRejectsUnsafeOrAmbiguousArguments` 覆盖选择与 scope。尚缺本变体错误 Project/代次、已释放、账本/强停/其他责任及拒绝后目标计划的完整矩阵 | Partial |
| 部署身份与入口约束 | [CLI 测试][cli] 直接验证新选择解析、精确 scope 与原 operation 冲突，并验证普通终止缺影响确认拒绝；尚缺新变体非 terminate 选项、影响确认缺失以及隐藏 Workspace HTTP 的专项直接测试 | Partial |

- 决策依据：D1，不变量 1、3、5。

## Missing Delete Provenance Survives Fresh Upgrade And Replay Without Inventing History

- 风险：缺失被补写为失败删除、来源组合矛盾，或升级抹掉旧意图身份与回放语义。
- 前置状态：迁移前的有原删除操作 legacy_d8、residual_ledger 根/从属意图和旧命令回执；新变体没有任何原删除操作。
- 触发：fresh、前一 schema 升级、重复迁移；创建新意图及从属删除；写入矛盾来源组合；重放旧记录。
- 必须成立：有原操作 legacy_d8 为非空引用/false；缺失变体为 `source=legacy_d8`、空引用/true；residual_ledger 为空引用/false。其余组合由数据库拒绝；旧行默认 false，历史与身份保留，重复迁移不重置进度。从属删除继承来源，新清理 operation 不被冒充原历史。
- 禁止结果：补造原删除 operation 或 D4/D5 账本；重写已合并迁移；升级按推测回填来源或成功证据；从属意图丢失缺失标记。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| fresh/upgrade/replay 保真 | [直接回归][missing-db] `TestControlledD8MissingDeleteMigrationUpgradesAndRepeats` 在隔离数据库建立新 schema，再重建前一 schema，比较既有 legacy 根意图除新字段外的完整内容，验证默认 false、重复 Migrate 与 CheckSchema；尚缺旧从属/ledger 意图、回执及非初始进度的完整升级矩阵 | Partial |
| 来源约束与继承 | 同文件首个准入测试与 `TestControlledD8MissingDeleteRetainsThenReleasesWithoutRewritingRun` 直接核对新根/从属的空原引用和 true 标记，且没有伪造原删除；升级测试拒绝有原引用/true 及空引用/false 两种矛盾组合。尚缺 residual_ledger 各组合与全部根/从属来源的约束矩阵 | Partial |

- 决策依据：D2，不变量 2、3。

## Missing Delete Takeover Keeps One Responsibility And Compatible Command Receipts

- 风险：新默认字段导致旧回执冲突；重放或并发建立第二份清理责任，跨来源重解释同一资源。
- 前置状态：旧版默认命令回执、新变体根意图、同资源不同幂等键，以及并发到达的删除/强停/回收责任。
- 触发：同键同输入重放、同键改变新选择、不同键同资源、并发申请与事务失败；进程重启后查询和继续处置。
- 必须成立：未选择新变体时旧规范输入不变；选择参与幂等输入，同键不同输入拒绝并保留冲突审计；同资源只取得一个持久责任，重复命令返回同一意图，来源矛盾拒绝；事务失败全部回滚，重启沿原责任继续。
- 禁止结果：增加默认字段便让旧回执失效；同资源重复根意图或副作用；重放切换原删除来源；半份责任存活。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 旧回执兼容与新选择冲突 | [直接回归][missing-db] `TestControlledD8MissingDeleteRequiresExplicitScopeAndKeepsHistory` 比较同键相同输入重放的完整意图；尚缺迁移前真实旧规范回执，以及同键改变新选择后的冲突审计专项断言 | Partial |
| 单一责任、回滚和重启 | 同文件准入重放验证返回原意图，拒绝矩阵检查无半份意图；尚缺本变体双事务两种提交顺序、不同键竞争、冲突责任、事务内故障回滚和进程重启读回 | Partial |

- 决策依据：D1–D3，不变量 3。

## Missing Delete Termination Retains Data Until Separately Confirmed Cleanup

- 风险：终止被当作删卷授权；以 Unknown、超时或备份代替双确认；新清理改写原 Run 历史。
- 前置状态：新变体根意图、原 sandbox/ensure 计划目标及保留数据；一部分目标缺终止或围栏证明，另有 stale lease 或删除移除证明缺失。
- 触发：逐目标终止、错误/缺失证明、终止完成、缺独立确认的数据删除、合法独立数据删除与真实移除确认。
- 必须成立：沿原 Controller lease 和目标身份取得逐目标 `terminated` 与 `lateEnsureFenced`；缺证明不成功。终止后只到 `terminated_data_retained`，原卷仍在；独立确认后才创建新的清理 operation，真实移除后才到 `released` 并软删除 Workspace。全过程原 Run 全部字段与原历史不变，无伪造账本。
- 禁止结果：终止自动删卷；以 Node Unknown、404、超时或备份代替物理证明；新处置重写 deliveryState/Revision/status/phase/result；开放后续 ensure/admission。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 两阶段、来源继承和完整历史保护 | [直接回归][missing-db] `TestControlledD8MissingDeleteRetainsThenReleasesWithoutRewritingRun` 经 Cloud 控制入口提交双确认、终止 retained，再独立声明清理并提交 removed，核对根/从属来源、仅一条新清理 operation、无残留账本及完整 Run 行不变；尚缺本变体部分证明、stale lease、缺独立确认和完整 operation/账本前后快照矩阵 | Partial |
| 真实组件上的物理保留与移除 | 原目标双确认、有效围栏、终止后卷仍在、单独删卷后真实移除；不以注入成功结果代替 | Missing |

- 决策依据：D3，不变量 1–5。

## Original Task Recovery Preserves Undelivered Work And Saved Revisions

- 风险：副本恢复被当作原任务恢复；解除暂停前丢失未交付成果，或后续执行没有从真实保存结果继续。
- 前置状态：原任务被 D8 无删除操作残留阻挡；原卷、Git 快照、会话及未交付成果有校验通过的完整备份。
- 触发：已批准实现部署后经显式两阶段处置解除原阻挡，在原任务恢复成果并执行、保存，再从该 Revision 续作。
- 必须成立：部署并开始获批恢复处置前原卷与备份保留；之后原卷只按 D3 独立确认清理，完整备份贯穿恢复验收，原历史结果不改写。原任务能继续，真实执行产生成果并保存 Revision，下一轮从已保存结果继续，Git Server 最终文件字节/摘要与恢复目标一致。
- 禁止结果：只验证副本或只看到 saved 标记便宣告原任务恢复；先丢弃原卷和未交付文件；直接改数据库伪造完成。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 原任务恢复与两轮持久化 | [circus#621](https://github.com/ora-space/circus/issues/621) 在部署后提供原任务内真实执行、Revision、续作和最终文件逐项校验；已有副本证据不替代本义务 | Missing |

- 决策依据：D3、验收边界与落地顺序；未来产生路径由 D4/#643 独立承接。

[missing-db]: https://github.com/ora-space/cloud/blob/1b772b442d2346aa19e85e80b40438faf0cc0bed/internal/core/agent_run_controlled_missing_delete_db_test.go
[cli]: https://github.com/ora-space/cloud/blob/1b772b442d2346aa19e85e80b40438faf0cc0bed/cmd/cloudctl/residual_test.go
