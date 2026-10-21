# Node 经 Git Server 交付与恢复前序 Revision

依据 `implemented` 的 [Node 交付推送与 Git 恢复决策](../../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)。
下文的 D 编号与不变量编号都指这篇 ADR；“clone 决策”指
[Node 从快照 clone 决策](../../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)。

传输仓库与带证书 Git 的共同义务登记在
[clone 与 `ora-git` 用例](../repository/snapshot-clone-and-ora-git.md)（`Credentialed Git Never Reads The Checkout Configuration`）。
恢复的完整截止、停止确认与取消矩阵由[恢复截止与停止确认用例](restore-deadline-and-stop-confirmation.md)登记；本文件的恢复
用例只覆盖内部线自身的拉取、检出与失败分类语义。

证据路径相对各产品仓库根目录。desktop 的 Node 测试使用真实 Git 与带客户端证书的本地 Git Server 替身；cluster 的检查项是
验收脚本写入证据的检查名开头。端到端验收中的 Agent 是 echo 替身或使用栈内受控 model-fixture 的 OpenCode，真实模型供应商
驱动的 Agent 写出的提交、改写的历史与交付不在这些证据之内。

## Every Delivery Pushes Its Final Commit To The Delivery Ref

- 风险：`unchanged` 不推送，续作要一跳引用；交付 ref 未写入就报告成功；推送了 `refs/ora/revisions/*` 或其他 ref；交付在别的 Workspace 分支上进行。
- 前置状态：会话结束、插件进程树已停止；三种最终提交：等于基础提交、等于前序最终提交、新提交。
- 触发：交付；交付输入的 `workspaceRef` 与 clone 记录不符。
- 必须成立：三种情况都把最终提交推到输入的 `refs/ora/deliveries/<id>`，前两种为 `unchanged`、第三种为 `delivered`；成功结果不带 bundle；推送成功后才上传 JSONL 并发送成功结果；`workspaceRef` 与 clone 记录不符时以 `checkout_unavailable` 失败，不推送、不上传。
- 禁止结果：生成 bundle；推送交付 ref 与 `workspaceRef` 以外的 ref 或 tag；`revision_ref` 离开 checkout。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 三种最终提交都推到各自的交付 ref，Git Server 上只有交付 ref 与 Workspace 分支 | desktop `apps/ora-node/src/revision/tests/push.rs` `every_outcome_is_pushed_and_a_replay_succeeds`；cluster `scripts/cross-line-acceptance.py` 检查项“内部线交付” | Covered |
| `delivered`/`unchanged` 按最终提交判定，快照不经钩子、无改动不产生新提交 | desktop `apps/ora-node/src/revision/tests/snapshot.rs` `unchanged_checkout_freezes_only_the_history`、`resumed_delivery_stopping_at_the_prior_final_commit_is_unchanged`、`changes_become_a_hookless_snapshot_on_top_of_the_agent_commits`；desktop `crates/node-db/src/tests/revision.rs` `unchanged_plans_stop_at_the_base_or_the_prior_final_commit` | Covered |
| 端到端交付只存会话 JSONL，结果不带 bundle | desktop `apps/ora-node/tests/standalone/agent_sessions/delivery.rs` `ended_session_is_delivered_with_requested_grants`；`crates/node-protocol/tests/protocol/revision.rs` `revision_failures_name_push_outcomes`；cluster `scripts/revision-acceptance.py` 检查项“有改动的运行”“无改动的运行”，`scripts/revision-resume-acceptance.py` 检查项“第二次运行（只续接）” | Covered |
| `workspaceRef` 与 clone 记录不符时以 `checkout_unavailable` 失败，不推送、不上传 | desktop `apps/ora-node/tests/standalone/agent_sessions/delivery.rs` `delivery_for_another_workspace_branch_is_refused` | Covered |
| 交付 ref 推送失败时不上传 JSONL、不发送成功结果 | 推送层失败路径有测试（见 `Push Failures Are Classified And Retried Within Bounds`），交付编排层“推送失败则不上传”的顺序没有直接测试 | Partial |
| 真实模型供应商驱动的 Agent 写出的提交经交付推到交付 ref | 无：端到端证据只用 echo 替身与 model-fixture | Missing |

### 决策依据

D2，不变量 3、5、7。

## A Frozen Delivery Replays The Same Push After Restart

- 风险：重启后重新快照，推送另一个提交，被只能新建的交付 ref 拒绝；重放把 up to date 判为失败。
- 前置状态：冻结计划已写入后杀掉 Node；另一次在推送成功、JSONL 上传前杀掉。
- 触发：Node 重启后继续同一执行。
- 必须成立：沿用冻结的最终提交、声明与 JSONL；远端已指向该提交时按成功处理；冻结文件在终态持久化后删除；冻结之前的重启重新准备。
- 禁止结果：重新快照；重放以 `push_rejected` 失败。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 远端交付 ref 已指向同一提交时重放按成功处理 | desktop `apps/ora-node/src/revision/tests/push.rs` `every_outcome_is_pushed_and_a_replay_succeeds` | Covered |
| 丢失应答后重启沿用冻结的 JSONL 字节 | desktop `apps/ora-node/src/revision/tests/upload.rs` `restart_after_a_lost_reply_reuses_frozen_bytes` | Covered |
| 冻结之前的重启丢弃未完成的准备、重新准备 | desktop `apps/ora-node/src/revision/tests/snapshot.rs` `preparing_again_discards_an_abandoned_attempt` | Covered |
| 冻结文件在终态后删除 | desktop `apps/ora-node/tests/standalone/agent_sessions/delivery.rs` `ended_session_is_delivered_with_requested_grants` | Covered |
| 冻结后或推送成功后杀掉 Node，重启后沿用同一计划、不重新快照 | 无直接测试：没有在推送阶段杀掉 Node 再继续同一执行的用例 | Missing |

### 决策依据

D3，不变量 4。

## Push Failures Are Classified And Retried Within Bounds

- 风险：`403` 或 ref 被拒被反复重试，延长沙盒占用；不可达被判永久失败；单次推送无时限，越过 Cloud 的放弃窗口。
- 前置状态：可注入应答的 Git Server 替身。
- 触发：`401`、`403`、其他 `4xx`；ref 已存在或 `pre-receive` 拒绝；`5xx`、`429`、连接失败、低速中止、单次时限到期、无法识别的错误。
- 必须成立：前两类不重试，以 `push_rejected` 失败；其余共 3 次尝试、间隔 5 秒与 30 秒，耗尽以 `push_failed` 失败；每次尝试 30 分钟时限，并在一分钟低于 1 KiB/s 时中止。
- 禁止结果：`403` 被重试；尝试次数超过 3。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| `401`、`403`、ref 已被占用只尝试一次并以 `push_rejected` 失败；`503` 共 3 次后 `push_failed` | desktop `apps/ora-node/src/revision/tests/push.rs` `refused_and_unreachable_pushes_fail_with_their_codes`；`apps/ora-node/src/git_server/tests.rs` `denied_requests_fail_at_once_and_unavailable_ones_are_retried` | Covered |
| 其他 `4xx`、`429`、DNS、无法识别的输出与 `pre-receive` 拒绝的分类 | desktop `apps/ora-node/src/git_server/outcome.rs` `failures_follow_the_status_and_ref_answers`、`porcelain_lines_name_the_fate_of_the_destination` | Covered |
| 间隔 5 秒与 30 秒、单次 30 分钟时限、低速中止 | 只有常量与命令行参数，测试以零间隔运行，没有断言实际间隔、单次时限到期或低速中止的用例 | Partial |

### 决策依据

D4，不变量 7。

## A Failed Workspace Branch Fast Forward Never Fails The Delivery

- 风险：Agent 改写了已推送的历史，快进被拒拖垮整个交付；账本未更新导致下一次推送误报冲突。
- 前置状态：交付 ref 已推送成功；Agent 推送过的分支不是最终提交的祖先；另一次远端已等于最终提交而账本未更新。
- 触发：交付快进 `workspaceRef`；快进时 Git Server 返回 `403` 或重试耗尽。
- 必须成立：被拒、旧值不符、`403`、重试耗尽都只记一条分类日志，交付结果不变、JSONL 照常上传；旧值不符而远端等于最终提交时按成功处理并更新账本；Agent 没推送时新建该分支。
- 禁止结果：交付因快进失败而失败；快进前做本地祖先检查；交付结果增加字段。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 非快进与旧值不符时交付仍成功、分支不变 | desktop `apps/ora-node/src/revision/tests/push.rs` `a_refused_workspace_fast_forward_never_fails_the_delivery`；`apps/ora-node/src/git_server/tests.rs` `push_reports_each_fate_of_the_ref` | Covered |
| 没有记录时新建分支；旧值不符而远端等于最终提交时按成功处理 | desktop `apps/ora-node/src/revision/tests/push.rs` `every_outcome_is_pushed_and_a_replay_succeeds`；cluster `scripts/workspace-git-acceptance.py` 检查项“场景 4 / 交付更新 Workspace 分支” | Covered |
| 账本记录本 checkout 对 Workspace 分支的已知值 | desktop `crates/node-db/src/tests/repository.rs` `branch_heads_record_the_last_known_value_per_checkout`；交付成功后更新账本的编排没有直接测试 | Partial |
| 快进单独遇到 `403` 或重试耗尽时交付仍成功，只记一条分类日志 | 无直接测试 | Missing |

### 决策依据

D2，不变量 6。

## Restore Moves The Workspace Branch To The Prior Final Commit From The Git Server

- 风险：恢复访问外部仓库或另一个仓库；协商使用 Agent 可写的 ref；恢复后本地分支名或 HEAD 不对；外部历史被改写时没有提示，或不该提示时提示。
- 前置状态：续作的会话，`priorRevision` 指向 Git Server 上的交付 ref；前序 `baseCommit` 分别是、不是本 checkout 快照提交的祖先。
- 触发：会话开始、插件启动之前；前序的仓库地址与 clone 记录不同；恢复期间会话被结束或取消。
- 必须成立：只从输入的 Git Server 地址以会话执行 ID 拉取交付 ref，取到的提交等于 `finalCommit`；地址与 clone 记录不同时不访问 Git Server；协商只以 clone 记录的 `base_commit` 为起点；以工作负载身份取入后 `checkout --force -B <Workspace 分支短名>`，工作区干净；`baseCommit` 不是快照提交的祖先时才在首条提问末尾追加改写提示；会话结束或取消时在下一条 Git 命令前停止，插件不启动。
- 禁止结果：访问外部仓库；`refs/remotes/ora/**`、仓库配置或 clone 基线改变；恢复失败后插件启动。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 恢复后分支名、HEAD、工作区干净、没有 remote、`refs/remotes` 不变，只用会话执行 ID；改写提示按快照历史判断 | desktop `apps/ora-node/src/revision/tests/restore.rs` `restore_moves_the_branch_and_judges_divergence_by_the_snapshot`；`apps/ora-node/tests/agent_session/resume.rs` `a_rewritten_remote_history_is_noted_in_the_first_turn` | Covered |
| 端到端从交付 ref 恢复并在 Agent 启动前指向前序最终提交，改写提示出现与不出现 | desktop `apps/ora-node/tests/standalone/agent_sessions/resume.rs` `resumed_session_restores_the_prior_revision_from_its_delivery_ref`；cluster `scripts/workspace-git-acceptance.py` 检查项“场景 4 / 续作的组合”“场景 4：替身写入无关快照后再续作”，`scripts/revision-resume-acceptance.py` 检查项“第二次运行（只续接）”“第三次运行（续接后再 [write]）” | Covered |
| 前序仓库地址与 clone 记录不同时不恢复、插件不启动 | desktop `apps/ora-node/tests/agent_session/resume.rs` `a_prior_revision_from_another_repository_is_refused_before_any_restore` | Covered |
| 恢复失败或恢复前取消时插件不启动 | desktop `apps/ora-node/tests/agent_session/resume.rs` `a_failed_restore_ends_the_session_before_any_plugin_starts`、`an_end_during_the_restore_cancels_it_before_any_plugin_starts`；`apps/ora-node/src/revision/tests/restore.rs` `cancelled_or_expired_restores_reach_nothing` | Covered |
| 协商只以 clone 记录的 `base_commit` 为起点，不使用 Agent 可写的 ref | 无直接断言 | Missing |
| 恢复后仓库配置不变 | 只断言没有 remote，没有比较恢复前后的仓库配置 | Partial |
| Git 命令运行中途收到取消时等它结束、不再开始下一条 | 只测试了恢复开始前已取消的情况 | Partial |

### 决策依据

D5，不变量 8、9、10。

## Restore Failures Separate Permanent Absence From Retryable Errors

- 风险：一次部署配置错误让 Cloud 把前序永久标为不可续接；缺陷导致的不一致让 Revision 永久放弃；Git Server 短暂不可用被判永久。
- 前置状态：可注入应答的 Git Server 替身；恢复时限。
- 触发：交付 ref 不存在；`403`；`401`；`401`、`429` 以外的其他 `4xx`；取到的提交不等于 `finalCommit`；`5xx`、`429`、网络、时限到期；本地取入失败。
- 必须成立：交付 ref 不存在、`403` 与 `401`、`429` 以外的其他 `4xx` 为 `prior_revision_base_unavailable`；`401` 为 `prior_revision_unavailable` 且不重试；其余为 `prior_revision_unavailable`；可重试的失败在时限内最多 3 次，下一次退避越过时限时不再等待；都以 `AGENT_FAILED` 加 `detail` 结束会话。
- 禁止结果：`401` 被归为 `base_unavailable`；提交不等被归为 `base_unavailable`；下一次退避越过时限仍等待。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| ref 不存在、`403`、`404` 为 `prior_revision_base_unavailable`；`401` 为 `prior_revision_unavailable` 且只请求一次；`503` 为 `prior_revision_unavailable` 且共 3 次；提交不等为 `prior_revision_unavailable` | desktop `apps/ora-node/src/revision/tests/restore.rs` `restore_failures_follow_the_git_server_answer` | Covered |
| 时限到期为 `prior_revision_unavailable` | desktop `apps/ora-node/src/revision/tests/restore.rs` `cancelled_or_expired_restores_reach_nothing`（时限为零） | Covered |
| 以 `AGENT_FAILED` 加 `detail` 结束会话 | desktop `apps/ora-node/tests/agent_session/resume.rs` `a_failed_restore_ends_the_session_before_any_plugin_starts` | Covered |
| `429`、下一次退避越过时限时不再等待、本地取入失败的分类 | 无直接测试 | Missing |

### 决策依据

D5、D6。

## A Cloud Mode Node Without Git Access Refuses To Start

- 风险：缺少 git-access 证书的 Node 照常启动，直到一整次会话之后才以交付失败暴露配置错误。
- 前置状态：配置了 `control.target` 的 Node。
- 触发：缺少 `git_access`；证书与私钥不匹配；CA 含私钥块；同时配置 `clone` 与 `git_access`。
- 必须成立：前三种拒绝启动，错误只给有限代码；齐全时声明 clone、`revision_delivery` 与 `revision_restore`。
- 禁止结果：缺少 `git_access` 的 Cloud 模式 Node 声明交付或恢复能力。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| Cloud 模式缺少 `git_access` 时以 `git_access_required` 拒绝启动 | desktop `apps/ora-node/src/service/tests.rs` `a_cloud_node_without_git_access_refuses_to_start` | Covered |
| 不可用的材料以有限代码拒绝 | desktop `apps/ora-node/src/git_access/tests.rs` `unusable_material_fails_closed_with_bounded_codes`；其中的“不匹配”样本是把 CA 当私钥，没有真正不匹配的证书与私钥对 | Partial |
| 能力随 git-access 身份声明 | desktop `apps/ora-node/tests/standalone/repository_ipc.rs` `git_server_capabilities_follow_the_git_access_identity`；`revision_delivery` 与 `revision_restore` 的声明由 `apps/ora-node/tests/standalone/agent_sessions/` 交付与恢复测试的握手断言 | Covered |

### 决策依据

D7，不变量 11。

## Node Protocol Carries No Bundle Or Download Grant

- 风险：bundle 字段或下载授权消息残留，新旧 Node 与 Controller 混用时静默走旧路径。
- 前置状态：新 `node-protocol`。
- 触发：解码带 `bundle_key` 的 `DeliverRevision`、带 `bundle` 的 `PriorRevision`；交付输入的前序带 Git 字段；`UploadGrantNeeded` 含 JSONL 以外的键；交付终态与派发输入不符。
- 必须成立：未知字段被拒；交付输入带仓库地址、交付 ref 与 `workspace_ref`；失败码含 `push_rejected`、`push_failed`；`DownloadGrantNeeded` 与 `DownloadGrant` 不属于协议；Controller 按派发输入核对交付终态，上传授权只请求 JSONL 键，不访问 Git Server。
- 禁止结果：新结果出现 `bundle_failed`。

### 验证义务与证据

| 义务 | 证据 | 状态 |
|---|---|---|
| 交付输入的形状与拒绝规则（含 `bundle_key`），失败码含推送结果、不含 `bundle_failed` | desktop `crates/node-protocol/tests/protocol/revision.rs` `deliver_revision_round_trips`、`deliver_revision_rejects_refs_keys_and_commits_outside_the_contract`、`revision_failures_name_push_outcomes` | Covered |
| 会话前序带 bundle 被拒、交付前序带 Git 字段被拒；下载授权消息不属于协议 | desktop `crates/node-protocol/tests/protocol/revision_restore.rs` `prior_revision_round_trips_in_session_and_delivery_inputs`、`download_grant_messages_are_not_part_of_the_protocol` | Covered |
| Controller 拒绝带拉取位置的交付前序，按派发输入核对终态 | desktop `apps/ora-controller/src/cloud/deliveries/mapping.rs` `terminal_results_must_match_the_dispatched_delivery`、`delivery_inputs_naming_a_prior_fetch_location_are_refused`、`records_rebuild_the_delivery_for_the_target_node` | Covered |
| Controller 只为 JSONL 键申请上传授权 | desktop `apps/ora-controller/tests/workspaces/deliveries.rs` `upload_grants_are_requested_only_for_the_history_key` | Covered |

### 决策依据

D8。
