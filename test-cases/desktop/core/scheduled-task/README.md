# Scheduled Task 核心测试用例

本目录是 Desktop Scheduled Task 的 reviewer 风险导航图，与
[决策目录](../../../../decisions/desktop/core/scheduled-task) 镜像。系统承诺来自
[定时任务执行归属根决策](../../../../decisions/desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md)。

## 阅读与维护规则

- 用例按调度身份、授权、执行隔离和历史事实组织，不按 crate 或页面组织。
- `Covered` 仅表示已有直接证据；定义层或 repository 层只覆盖部分义务时标为 `Partial`。
- Cloud Web 首期已有独立 [implemented ADR](../../../../decisions/cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md)
  与[核心用例](../../../cloud/scheduled-task/admission-and-session-bounds.md)；Desktop Cloud 创建与管理仍待后续决策，保持不可激活。
- 根决策保持 `approved`。已有实现和历史验收不自动补足 manual、资格/截止及跨边界验证缺口；源码存在不等同直接测试通过。

| 文件 | 覆盖的风险面 |
| --- | --- |
| [schedules-and-occurrences.md](schedules-and-occurrences.md) | 受限周期、occurrence 幂等、迟到/离线 missed、manual 身份与时区边界 |
| [admission-and-approval.md](admission-and-approval.md) | Git 项目资格、Cloud 不可激活、无人值守同意、重授权和撤销 |
| [execution-isolation.md](execution-isolation.md) | revision 冻结、新会话、独立 worktree、并发、超时和无自动重试 |
| [history-notifications-and-residency.md](history-notifications-and-residency.md) | 状态事实、通知、软删除、100 条保留与本地数据驻留 |
| [worktree-ownership-and-recovery.md](worktree-ownership-and-recovery.md) | worktree 原子归属提交、故障注入恢复、retention 分类边界与既有路径回归 |
| [run-result-annotation-and-artifacts.md](run-result-annotation-and-artifacts.md) | 独立业务 verdict、运行产物持久化与安全边界 |

