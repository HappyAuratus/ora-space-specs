# Scheduled Task 接纳与授权

## Local Activation Must Require A Git Project And Explicit Consent

- 风险：无人值守 Agent 在用户当前目录直接运行，或在未告知权限、凭据和额度风险时自动执行工具。
- 前置状态：用户保存 enabled local task，目标可能不存在、不是 Git 项目，或没有
  `full_auto_in_worktree` 同意。
- 触发：创建、编辑后启用，或 scheduler 每次接纳。
- 必须成立：active local task 的 Project 必须存在且为 Git；有效同意随任务持久化；缺失任一资格即拒绝保存或明确失败，
  不回退原目录。
- 禁止结果：默认勾选同意、只在 UI 校验，或项目无 worktree 能力时直接在原目录执行。
- 决策依据：根决策 D4、D8 与不变量 9、10、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| enabled local task 没有显式 policy 时拒绝创建 | `ora_application::scheduled_task::tests::enabled_local_task_requires_explicit_unattended_approval` | `Covered` |
| 合法 Project 和 consent 创建 revision 1 的 active local task | `ora_application::scheduled_task::tests::approved_local_task_is_created_at_revision_one` | `Partial` |
| 数据库约束使 policy 与 consented-at 同时存在或同时为空 | `ora_db::migration::schema::schema_v0015::tests::creates_constrained_scheduled_task_storage_and_rolls_back` | `Covered` |
| 非 Git、缺失目录和 worktree 创建失败均不能降级原目录 | `ora_application::scheduled_task::target_validator::tests::rejects_a_missing_or_non_git_project_directory`、`ora_application::scheduled_task::tests::enabled_local_task_rejects_a_target_without_git_worktree_capability`；尚缺 executor 创建 worktree 失败且不启动原目录的直接断言 | `Partial` |
| UI 同意默认不勾选 | `packages/app-shell/src/features/scheduled-task/scheduled-task-manager.test.tsx` 的 `opens the new-task form with unattended consent unchecked` | `Covered` |
| UI 在同意前展示 worktree 非沙箱、命令/凭据及额度风险 | 尚缺完整风险文字展示的直接断言；默认不勾选的测试不证明风险说明完整 | `Missing` |

## Reconsent Must Follow Project Or Policy Changes Only

- 风险：任意编辑都反复索取授权，或真正改变执行边界时沿用旧同意。
- 前置状态：任务已有有效 `full_auto_in_worktree` 同意。
- 触发：分别修改 Prompt/schedule、agent/model、Project 或 approval policy。
- 必须成立：Prompt 和 schedule 修改增加 revision 但保留同意；Project 或 policy 变化清除旧同意，保持 enabled 时必须重新同意。
- 禁止结果：以 revision 变化本身强制重授权，或换 Project 后继续使用旧同意。
- 决策依据：根决策 D2、D8 与不变量 6、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| Prompt/schedule 修改 revision+1 且 consent 时间不变 | `ora_application::scheduled_task::tests::prompt_and_schedule_changes_increment_revision_without_reconsent` | `Covered` |
| Project 变化且没有 fresh consent 时不能保持 enabled | `ora_application::scheduled_task::tests::changing_project_requires_fresh_consent_before_remaining_enabled` | `Covered` |
| policy 变化必须重新展示风险并写入新 consent | 尚无直接证据 | `Missing` |

## Revoking Consent Must Disable Future Scheduling

- 风险：用户撤销无人值守权限后，任务仍在后台接纳新运行，或通过删除历史掩盖旧事实。
- 前置状态：任务已启用并有既有运行或历史。
- 触发：用户撤销 consent。
- 必须成立：任务原子清除 approval 并停用；未来 occurrence 不再接纳；既有 revision、run 和 missed 历史不改写。
- 禁止结果：只清除 UI 开关、取消并删除已完成历史，或允许 scheduler 使用缓存授权继续运行。
- 决策依据：根决策 D8 与不变量 8、9。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| revoke 清除 approval、停用任务且不改变 definition revision | `ora_application::scheduled_task::tests::revoking_approval_disables_the_task_without_changing_revision` | `Covered` |
| scheduler 观察撤销后取消未来 job 且不改写历史 | 后端 `revoke_approval` 已取消 runtime handle，trigger 重读任务；尚缺撤销与触发竞态、历史不变的直接集成测试 | `Missing` |

## Local Eligibility Changes Must Fail Without Substitution

- 风险：保存后目标或模型失效，运行使用其他模型、直接回退用户目录，或以通用错误掩盖资格失效。
- 前置状态：local 任务已保存且启用，随后 Project、模型或策略变得不可用。
- 触发：下一次计划 occurrence 接纳并尝试启动。
- 必须成立：每次运行复查资格，明确记录 `target_unavailable`、`model_unavailable`、
  `approval_policy_invalid` 或 `worktree_failed`；不换模型、不回退原目录或 Cloud，不自动重试。
- 禁止结果：只在保存时验证模型、启动失败统一伪装为成功，或撤销同意后仍启动新工作。
- 决策依据：根决策 D8 与不变量 9、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 目标或模型在保存后失效时，每次运行复查并记录对应原因，不替换模型 | runtime 尚缺对应直接测试；保存时的模型校验测试不能证明运行时复查，session start 错误目前统一映射 `target_unavailable` | `Missing` |
| 策略失效或 worktree 创建失败时不启动目标，不回退用户目录 | 同意检查与 worktree 失败分支已有实现，尚缺运行链路故障注入测试 | `Missing` |

## Cloud Must Stay Visible But Inactive Until Its Real Adapter Exists

- 风险：Desktop 用本地 timer、mock success 或本机凭据冒充 Cloud 执行。
- 前置状态：Cloud Web 首期已有独立 implemented ADR 与真实执行/取消链，但 Desktop Cloud 创建、展示与历史入口仍待后续决策。
- 触发：用户查看 Cloud 说明或尝试保存 Cloud task。
- 必须成立：当前 Desktop UI 可解释 Cloud，但不能激活或保存为 execution owner；backend 同样拒绝请求；不会产生模拟 run。
  Cloud Web 的独立入口不受此 Desktop 限制；满足 Cloud 真实链条件不等同 Desktop 自动解锁。
- 禁止结果：仅禁用按钮却允许直接 IPC 保存，或失败后回退 local。
- 决策依据：根决策 D3、D5 与不变量 1、5、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| application 拒绝 Cloud definition | `ora_application::scheduled_task::tests::cloud_definitions_are_rejected_while_cloud_is_unavailable` | `Covered` |
| Desktop 显示 Cloud 说明但所有保存/激活入口不可用 | 尚无直接证据 | `Missing` |
| Cloud 启用前具备真实 issue-bound IssueRun/Thread、资格复查和 60 分钟取消写入者 | [Cloud 核心用例](../../../cloud/scheduled-task/admission-and-session-bounds.md) 的直接 integration 与 cluster Compose 真实栈证据；只证明 Cloud Web 首期，不能证明 Desktop 入口已交付 | `Covered` |

