# Node 恢复前序 Revision

依据[已实现的恢复前序 Revision 后续决策](../../../decisions/node/revision/20261010-restore-prior-revision-before-session.md)。
证据来自 `ora-space/desktop`：`apps/ora-node/src/revision/restore/tests` 用真实 Git 与本地 HTTP 替身，
`apps/ora-node/tests/agent_session` 以替身插件驱动会话，`apps/ora-node/tests/standalone` 以真实 Node 进程运行，
`crates/utils` 测试受限下载器。Controller 的中继与能力检查见
[恢复契约](../../cloud/controller-integration/revision-restore-contract.md)。

本文件描述已实现的 bundle 恢复决策与 desktop main 的现状。[Node 恢复截止与停止确认决策](../../../decisions/node/revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)修改了该决策 D2 的时限范围（从授权等待与下载扩展到恢复全程、命令预算裁剪到剩余截止）与 D5 的交付判定（恢复未完成的会话不探查工作区）；这些新义务由[恢复截止与停止确认用例](restore-deadline-and-stop-confirmation.md)登记，本文件的证据只按已实现行为计，不推导新决策已覆盖。

## Restore Runs Before The Agent Starts And Failure Ends The Session

### 风险

恢复失败后 Agent 仍在缺少前序成果的仓库上运行，下一次交付覆盖掉用户的工作。

### 前置状态

会话输入带 `prior_revision`，插件已安装。

### 触发

恢复失败（bundle 取不到或基础提交缺失）。

### 必须成立

- 会话以 `agent_failed` 结束，`detail` 为 `prior_revision_unavailable` 或 `prior_revision_base_unavailable`。
- 插件从未启动，Node 不产生 Thread 事件，插件租约被释放。

### 禁止结果

插件启动；降级为全新 checkout 继续。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 失败时插件不启动 | `apps/ora-node/tests/agent_session/resume.rs::a_failed_restore_ends_the_session_before_any_plugin_starts`：两种 `detail` 都结束会话，没有插件进程与 Thread 事件，租约已释放 | Covered |
| 插件缺失时不下载、恢复中停机 | 尚无测试 | Missing |

### 决策依据

恢复决策 D1，不变量 1。

## Restore Distinguishes A Missing Base From An Unavailable Bundle

### 风险

一次网络故障被当作“基础提交永久缺失”，Cloud 永久拒绝续接该 Revision；或损坏的 bundle 被交给 Git。

### 前置状态

会话 checkout 已 clone；前序 bundle 的前置提交可能不在 checkout 中。

### 触发

下载 bundle 并按 D3 恢复。

### 必须成立

- 下载结果的大小与 SHA-256 不符时失败，Git 不被调用。
- bundle 头只接受 v2/v3、恰好一个 `refs/ora/revisions/` 下且指向前序最终提交的 head。
- 前置提交缺失时从 `origin` 补取一次；远端明确没有该对象或补取后仍缺失为 `prior_revision_base_unavailable`，
  远端不可达为 `prior_revision_unavailable`；`bundle verify` 失败为 `prior_revision_unavailable`。

### 禁止结果

把不可达归为 `base_unavailable`；未校验的 bundle 进入 Git；接受多个 head 或其他命名空间的 head。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 下载校验 | `restore/tests/restorer.rs::refusal_mismatch_and_deadline_are_unavailable`：篡改字节、拒绝签发与超时都是 `prior_revision_unavailable`；`crates/utils/src/http/fetch_tests.rs::refuses_oversized_bodies_and_existing_destinations_without_the_url` | Covered |
| bundle 头形状 | `restore/bundle.rs::parses_v2_and_v3_revision_bundles`、`::refuses_other_shapes`；`restore/tests/git.rs::a_wrong_head_or_damaged_bundle_is_unavailable` | Covered |
| 两种失败的划分 | `restore/tests/git.rs::fetches_a_missing_base_and_reports_the_rewritten_remote_history`、`::an_unfetchable_base_is_base_unavailable_and_leaves_the_checkout`、`::an_unreachable_origin_is_unavailable_not_base_unavailable` | Covered |

### 决策依据

恢复决策 D2、D3，不变量 2。

## Restored Checkout Points The Default Branch At The Prior Final Commit

### 风险

Agent 在分离 HEAD 或新分支上工作；恢复改写了远端跟踪分支或仓库配置；远端历史被改写时 Agent 不知情。

### 前置状态

bundle 已校验，前置提交都在 checkout 中。

### 触发

取入 Revision ref 并检出。

### 必须成立

- HEAD 为 clone 的默认分支，指向前序最终提交，工作区干净；`origin/<默认分支>` 不变；复制到 Git 目录的 bundle 已删除。
- 前置提交不是 `origin/<默认分支>` 的祖先时，首条提问末尾追加固定格式的说明；否则不追加。

### 禁止结果

检出为分离 HEAD；改动远端跟踪分支；无改写时追加说明。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 检出位置与不动远端 | `restore/tests/git.rs::restores_the_prior_final_commit_onto_the_clone_branch`。缺口：没有断言仓库配置不变 | Partial |
| 改写提示 | Git 判定见 `restore/tests/git.rs::fetches_a_missing_base_and_reports_the_rewritten_remote_history`；提示全文与未改写时不追加见 `apps/ora-node/tests/agent_session/resume.rs::a_rewritten_remote_history_is_noted_in_the_first_turn` | Covered |

### 决策依据

恢复决策 D3、D4，不变量 4、5。

## Download Grants Stay In Memory And Downloads Are Bounded

### 风险

签名 URL 被写入账本或日志；下载无限等待或无限重试；临时文件残留在 Node 目录。

### 前置状态

会话带前序，Node 等待 Controller 转交的下载授权。

### 触发

授权到达、过期、被拒绝或返回错误状态；Controller 重连。

### 必须成立

- 只接受本次恢复登记的对象的授权；等待期间与新连接上重发请求。
- 5xx 有界重试，第一次 403 换授权一次，之后的 403 立即失败；不跟随重定向；只写属主可读的文件。
- 等待与下载合计有上限；恢复结束后删除私有目录。
- 签名 URL 不进入账本。

### 禁止结果

接受其他对象的授权；跟随重定向；授权被持久化；私有目录残留。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 授权匹配与重发 | `restore/tests/grants.rs::accepts_only_the_registered_restores_object`、`::waiting_requests_repeat_until_answered_and_end_with_the_registration` | Covered |
| 重试与换授权 | `restore/tests/restorer.rs::retries_server_errors_and_renews_a_forbidden_grant_once` | Covered |
| 下载器限制 | `crates/utils/src/http/fetch_tests.rs::stores_a_success_body_owner_only_with_its_digest`、`::reports_other_statuses_without_following_redirects` | Covered |
| 目录清理与不落盘 | `restore/tests/restorer.rs::restores_through_a_granted_download_and_removes_its_directory`；`apps/ora-node/tests/standalone/agent_sessions/resume.rs::resumed_session_restores_the_prior_revision_with_a_relayed_grant`：真实 Node 经中继授权完成恢复，账本中没有签名。缺口：目录 `0700`、启动时清空与 Node 日志不含授权没有直接断言 | Partial |

### 决策依据

恢复决策 D2，不变量 3。

## Delivery After A Restore Is Unchanged Only At The Prior Final Commit

### 风险

续接后的空运行重复上传与前序相同的 bundle；或有新提交的运行被记为 `unchanged`。

### 前置状态

交付输入带 `prior_revision`。

### 触发

会话结束后交付。

### 必须成立

- 最终提交等于前序最终提交时为 `unchanged`，不生成 bundle。
- 有新提交时为 `delivered`，bundle 仍相对本次基础提交，包含前序的提交。
- 最终提交等于基础提交时为 `unchanged`。

### 禁止结果

停在前序最终提交时生成 bundle；新 bundle 缺少前序的提交。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 停在前序为 `unchanged` | `apps/ora-node/src/revision/tests/snapshot.rs::resumed_delivery_stopping_at_the_prior_final_commit_is_unchanged`；`restore/tests/git.rs::deliveries_after_a_restore_are_unchanged_or_bundle_the_prior_work`；`crates/node-db/src/tests/revision.rs::unchanged_plans_stop_at_the_base_or_the_prior_final_commit` | Covered |
| 新提交的 bundle 含前序 | `restore/tests/git.rs::deliveries_after_a_restore_are_unchanged_or_bundle_the_prior_work` | Covered |
| 带前序但停在基础提交 | Node 快照层尚无测试；账本层由 `unchanged_plans_stop_at_the_base_or_the_prior_final_commit` 覆盖 | Partial |

### 决策依据

恢复决策 D5，不变量 6。
