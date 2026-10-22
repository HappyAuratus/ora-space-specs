# 仓库凭据分类、成员离开与受控重绑

依据[凭据连续性决策](../../../decisions/cloud/project/20260927-repository-credential-continuity.md)（`implemented`）。下文的 D 编号与不变量编号都指这篇 ADR。
Cloud 保存引用和业务状态，真实凭据由 Credential Store 保管；Ora 权限不等同外部 Git 平台授权。
测试路径指 cloud 仓库的 [`fc12dbe`](https://github.com/ora-space/cloud/tree/fc12dbe4722b291a806ea07a1217655cdd7789b5)，Node 一侧指 desktop 的
[`8b79b4f`](https://github.com/ora-space/desktop/tree/8b79b4f2284c67a7d47c373611b2094572f0c81f)；没有直接测试的义务保持 **Partial** 或 **Missing**，并写明缺口。
同步、发布、Credential Store 与 Repo Executor 的细节义务分别登记在
[仓库同步](repository-binding-and-snapshot-sync.md)、[显式快进发布](explicit-fast-forward-publish.md)、
[Git 凭据存储](../identity-access/git-credential-store.md)和[Repo Executor 边界](../repo-executor/repo-executor-boundary.md)用例，本文只登记凭据连续性自己的承诺。

## Member Departure Freezes Personal Or Unknown References Without Deleting Local Work

- 风险：创建者离开后继续使用其个人凭据，或为收紧远程权限误删团队代码与任务。
- 前置状态：项目分别绑定团队、个人和未知引用；有本地克隆、已开始任务、未接受远程意图和未知派发结果。
- 触发：关联成员离开/停用，恢复成员关系，再请求同步、发布、本地编辑、启动或清理。
- 必须成立：个人/未知引用不供新远程操作，团队引用不因旧 owner 离开误停；恢复成员不解冻。已 `running` 的同步按原范围收尾，未知派发需核对；本地内容和快照 clone 按运行时权限继续，清理审计引用保留。
- 禁止结果：自动转移个人凭据、冻结全部本地工作、删除项目、把外部凭据吊销导致的任务失败承诺为成功。
- 验证义务与证据：**Partial** — `integration/repository_credentials_test.go` 的
  `TestCredentialDepartureFreezesPersonalAndUnknownButRetainsTeamHistory`（停用后个人与未知冻结、团队保持可用、owner 外键不变，重新加入后仍冻结）、
  `TestCredentialedProjectClonesItsSnapshotWithoutTheCredential`（带凭据的 Project 照常从快照 clone，凭据不进入 clone 输入）；
  `internal/repoworker/operation_test.go` 的 `TestRefusedStartNeverReachesTheExecutor`（拒绝的开始不会到达执行器）；
  `integration/external_git_test.go` 的 `TestRepositorySyncSystemSuccessorRecordsSkipReason`（冻结凭据阻止系统触发的后继，原终态事务照常提交）。
- 仍缺：没有用户侧 `409 repository_credential_frozen` 的测试，没有已 `running` 的同步在冻结后收尾的测试，也没有已有任务与本地数据的联合停用矩阵。
- 决策依据：D1、D2，不变量 1、2。

## A Team Reference Becomes Active Only After Scoped Capability And Version Validation

- 风险：普通成员或管理员取得他人的个人凭据，或迟到的验证结果恢复过期的绑定。
- 前置状态：原绑定可用或冻结；管理员、个人凭据所有者、普通成员各提交候选；另有并发的绑定更新、成员停用或引用撤销。
- 触发：提交候选凭据、候选同步成功/失败/结果不明/迟到、Binding 版本在其间改变。
- 必须成立：团队引用只有管理员能创建和绑定，个人引用只有所有者能绑定；候选先经同步验证，同步成功且 Binding 版本未变时在记录快照的同一事务里切换；失败保留原绑定及冻结状态；读取成功不记为可推送；Cloud 不接收秘密，也不自己访问远端验证。
- 禁止结果：失败或迟到的结果覆盖原绑定、并发更新后候选仍生效、Cloud 接收 token、改写 owner/创建者/任务 actor。
- 验证义务与证据：**Partial** — `integration/repository_publish_external_test.go` 的
  `TestRepositoryPublishPersonalApprovalMatrixAndIdempotency`（管理员不能绑定或批准他人的个人凭据）；`integration/external_git_test.go` 的
  `TestGitCredentialSubmissionRotationConfirmationAndRevocation`（秘密不经 Cloud，提交与确认分离）；能力证据见
  [Git 凭据存储](../identity-access/git-credential-store.md)的 `Capabilities Hold For One Store Version And Are Proven, Not Declared`。
- 仍缺：没有候选同步成功后原子切换、`version` 已变时不切换、候选失败保留原绑定、迟到结果不生效的测试，
  也没有 `409 sync_in_progress` 的测试。
- 决策依据：D1、D3，不变量 3。

## Git Execution Uses Its Planned Reference Without Static Credential Fallback

- 风险：Cloud 标冻结，但执行端仍用全局配置里的另一套凭据，或用户代码读取长期管理密钥后绕过引用治理。
- 前置状态：授权固定引用、版本、仓库和能力；部署环境另有可用凭据；引用分别失效、越界、版本过期。
- 触发：新的同步或发布、旧任务收尾、提供方拒绝、Git 平台吊销；工作负载尝试读取管理材料或转交资格。
- 必须成立：只兑换授权中的引用与版本，缺失/失效/能力不符明确拒绝，无静态回退；公开仓库不使用任何凭据；明文只在当次任务的材料目录，结束后销毁，不进入工作目录、Cloud、前端或日志；Node 不持有任何仓库凭据、不允许 SSH。
- 禁止结果：复用另一项目的秘密、复制长期 token、提供方失败时读取个人 SSH、宣称 Ora 撤销了外部个人账号权限。
- 验证义务与证据：**Partial** — `internal/repoworker/operation_test.go` 的 `TestRefusedStartNeverReachesTheExecutor`；
  `internal/repoexec/sync_test.go` 的 `TestSyncExternalFailures`（匿名访问私有仓库为 `repository_requires_credential`，认证失败为 `credential_rejected`）；
  材料目录与脱敏见 [Repo Executor 边界](../repo-executor/repo-executor-boundary.md)的 `Credential Plaintext Never Leaves The Task Material Directory`；
  cluster 的 `scripts/cross-line-acceptance.py` 第 11 项与 `external-git-acceptance.sh` 在日志、数据库、Git Server、执行器任务目录和凭据卷中扫描 token 残留（带阳性对照）。
  Node 一侧由 desktop `apps/ora-node/src/repository/config.rs` 固定禁用 SSH、清空 askpass 并忽略系统 Git 配置。
- 仍缺：Node 的这条配置没有直接断言的测试；没有提供可用回退凭据后仍被拒绝的联合测试；没有扫描进程参数、环境及编码变体的秘密残留测试。
- 决策依据：D4，不变量 4。

## Credential Upgrade Preserves Unknown Ownership And Historical References

- 风险：迁移按 owner 猜测团队归属，或切换引用删除原执行和恢复关联。
- 前置状态：旧库有历史引用、owner 外键、运行和失败验证记录；另准备空库。
- 触发：追加迁移并重入、切换团队引用、外部引用过期。
- 必须成立：历史无法证明的类型为未知，保留关联和既有有效成员下的使用边界；离开时收紧。新库和升级库都保留 owner 及原执行引用版本；可用性及验证状态可审计且无秘密。
- 禁止结果：修改已发布 SQL、静态 SSH 默认升级为团队、切换删除历史引用。
- 验证义务与证据：**Partial** — `integration/runtime_upgrade_test.go` 的
  `TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences`（未知引用与旧 owner/credential 外键保留）；
  `integration/migration_upgrade_path_test.go` 的 `TestMigration0043ExternalGitLineAppliesFreshAndUpgrades`（新库与升级库的 Binding 与凭据列）。
- 仍缺：没有完整执行历史快照比较，没有外部凭据过期后历史引用与恢复关联保留的测试。
- 决策依据：D1、D5，不变量 5。
