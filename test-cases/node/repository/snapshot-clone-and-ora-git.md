# Node 从 Git Server 快照 clone 与 `ora-git`

依据 [Node 从快照 clone 决策](../../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)（`implemented`）。
下文的 D 编号与不变量编号都指这篇 ADR；“交付推送决策”指
[Node 交付推送与 Git 恢复决策](../../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)。

证据路径以产品仓库名开头：`desktop/`、`cloud/`、`cluster/` 分别指 ora-space 对应仓库的根目录；`文件::函数` 是测试函数，
cluster 验收脚本 `cluster/scripts/workspace-git-acceptance.py` 的证据写检查项名（脚本中检查项说明的开头）。

[仓库获取与结果边界](clone-and-result-boundaries.md)、[最小 clone 执行](minimal-clone-execution.md) 与
[准入与 Git 执行](admission-and-git-execution.md) 以外部 clone 的前序决策为依据；其中关于外部地址、SSH、部署凭据与 `origin`
的义务已被本文取代，目录独占、受管执行、中断判定与恢复的义务仍按那几份用例维护。

## Clone Accepts Only A Git Server Snapshot Input

- 风险：旧形状或外部地址的 clone 命令仍被执行，Node 以部署凭据访问外部仓库；`snapshot_ref` 与 `base_commit` 不对应；没有
  Git Server 身份的 Node 仍声明 clone 能力。
- 前置状态：新 Node 协议与执行器。
- 触发：带 `repository`/`branch` 的旧命令；`ssh://` 或带用户信息、查询、片段的地址；`snapshot_ref` 末段不等于 `base_commit`；
  `workspace_ref` 短名为 `HEAD`；同一执行身份换了输入；Node 缺少 `git_access`。
- 必须成立：只接受 D1 形状的输入；同身份不同输入冲突；clone 只访问输入中的 Git Server 仓库并只读取 `snapshot_ref`；只有同时
  配置 `clone` 与 `git_access` 才声明 clone 能力，Cloud 模式缺少 `git_access` 拒绝启动。
- 禁止结果：旧形状被执行；任何外部仓库或部署凭据被使用。
- 决策依据：D1、D2、D10，不变量 1、10。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 地址只接受 Git Server 的 `https`，拒绝 `ssh://`、用户信息、查询、片段、空白与控制字符 | `desktop/crates/node-protocol/tests/protocol/repository.rs::clone_accepts_only_git_server_https_addresses` | Covered |
| `snapshot_ref`、`base_commit`、`workspace_ref` 的形状规则，`snapshot_ref` 末段必须等于 `base_commit` | `desktop/crates/node-protocol/tests/protocol/repository.rs::clone_rejects_malformed_snapshot_inputs`、`::clone_rejects_a_base_commit_the_snapshot_ref_does_not_name` | Covered |
| 旧的 `repository`/`branch` 形状与未知字段被拒 | `desktop/crates/node-protocol/tests/protocol/repository.rs::clone_rejects_the_retired_repository_and_branch_shape`、`::clone_rejects_execution_controls_in_payload_and_spec` | Covered |
| 同一执行身份换了输入时冲突 | `desktop/crates/node-db/src/tests/repository.rs::clone_acceptance_deduplicates_and_reserves_across_capabilities` 只在数据库层、只改 `workspace_ref`；Node 受理层没有直接测试 | Partial |
| clone 不使用部署凭据，也不访问外部仓库 | `desktop/apps/ora-node/tests/standalone/repository_policy.rs::snapshot_clone_ignores_deployment_credentials_and_checkout_extensions` 证明部署的 credential helper 不被调用；cluster 验收“沙盒网络是 --internal”与“场景 1”证明外部地址不可达时 clone 仍成功；没有测试断言拉取只请求 `snapshot_ref` | Partial |
| 能力只在同时配置 `clone` 与 `git_access` 时声明；Cloud 模式缺 `git_access` 拒绝启动 | `desktop/apps/ora-node/src/service/tests.rs::a_cloud_node_without_git_access_refuses_to_start`；`desktop/apps/ora-node/tests/standalone/repository_ipc.rs::git_server_capabilities_follow_the_git_access_identity` 覆盖有无 `git_access`，未覆盖只有 `git_access` 没有 `clone` | Partial |

## A Snapshot Checkout Has No Remote And Points At The Base Commit

- 风险：checkout 留下 origin、凭据相关配置或 alternates，Agent 绕过平台访问外部，或删除传输仓库后对象缺失；成功时 HEAD 不是
  快照提交；传输仓库残留或在仍被读取时被清理。
- 前置状态：Git Server 上有快照 ref 指向 `C`。
- 触发：clone 成功；核实时 HEAD 被篡改为另一个提交；工作负载段中途重启。
- 必须成立：HEAD 是指向 `refs/heads/<branch>` 的符号引用，指向 `C`，已跟踪文件干净；`git remote` 为空，没有 `remote.*`、`http.*`、
  `credential.*` 配置，没有 `objects/info/alternates` 与 `refs/remotes/**`，历史完整非浅；`CloneReady.commit = C`；传输仓库在结束后
  删除，启动清理只保留尚未完成本地取入的 clone 传输仓库。
- 禁止结果：HEAD 不等于 `C` 时报告成功（必须保留 Unknown 与目录）；checkout 出现证书路径或 `Ora-Execution-Id`。
- 决策依据：D2、D3，不变量 2、3。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 成功后的 checkout 形状：无 remote、无 `remote.`/`http.`/`credential.` 配置、无跟踪 ref、无 alternates、非浅，HEAD 指向分支且等于 `C` | `desktop/apps/ora-node/tests/standalone/repository.rs::managed_snapshot_clone_checks_out_the_base_commit_without_a_remote`；cluster 验收“场景 1” | Covered |
| 成功结果的 `commit` 必须等于 `base_commit` | `desktop/crates/node-protocol/tests/protocol/repository_results.rs::clone_results_require_complete_destination_and_commit_facts`；`desktop/apps/ora-node/tests/standalone/repository.rs::managed_snapshot_clone_checks_out_the_base_commit_without_a_remote` | Covered |
| 核实不通过（HEAD 不等于 `base_commit`）时保留 Unknown 与目录 | 无直接测试 | Missing |
| 传输仓库在结束后删除（成功与失败） | `desktop/apps/ora-node/tests/standalone/repository.rs::managed_snapshot_clone_checks_out_the_base_commit_without_a_remote` 只覆盖成功；失败后的删除没有直接测试 | Partial |
| 启动清理保留尚未完成本地取入的传输仓库，清除其余残留 | `desktop/apps/ora-node/tests/standalone/repository_commit.rs::restart_inside_the_workload_sequence_keeps_its_transport_repository` 只覆盖保留一侧；清除拉取段与已完成取入的残留没有直接测试 | Partial |

## Clone Failures Are Classified By The Git Server Answer

- 风险：快照缺失被当作网络失败反复重试；快照 ref 与 Cloud 登记不一致被掩盖；被终止或失败的尝试被报告为成功。
- 前置状态：可注入应答的 Git Server 替身。
- 触发：`401`、`403`、快照 ref 不存在；取到的提交不等于 `base_commit`；`5xx`、`429`、连接与 TLS 失败、低速中止；工作负载命令
  以非 0 退出或被终止；目标目录已存在。
- 必须成立：拒绝与缺失为 `SnapshotUnavailable` 且不重试；不等为 `BaseCommitMismatch`；暂时性失败重试后为 `SourceUnavailable`；
  非 0 退出为 `OperationFailed`；被终止为 `Interrupted`；`BranchNotFound` 不再产生；已存在的目标目录保留为 Unknown。
- 禁止结果：`401`/`403` 被重试；不等被报告为 `SourceUnavailable`。
- 决策依据：D3。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 快照 ref 缺失、`403`、`401` 为 `SnapshotUnavailable`，失败保留目录，`401` 不重试 | `desktop/apps/ora-node/tests/standalone/repository.rs::snapshot_failures_retain_targets_and_new_retry_uses_new_directory`；分类规则见 `desktop/apps/ora-node/src/git_server/outcome.rs::failures_follow_the_status_and_ref_answers`；clone 层没有断言 `403` 只请求一次 | Partial |
| 快照 ref 指向别的提交为 `BaseCommitMismatch` | `desktop/apps/ora-node/tests/standalone/repository.rs::snapshot_failures_retain_targets_and_new_retry_uses_new_directory` | Covered |
| `5xx` 重试，耗尽为 `SourceUnavailable` | `desktop/apps/ora-node/tests/standalone/repository.rs::unavailable_git_server_fails_the_clone_once_the_attempts_run_out`、`::unavailable_git_server_is_retried_within_the_fetch_pass` | Covered |
| `429`、DNS、TLS、低速中止与无法识别的失败按暂时性失败处理 | `desktop/apps/ora-node/src/git_server/outcome.rs::failures_follow_the_status_and_ref_answers` 只验证分类函数；clone 层没有直接测试 | Partial |
| `BranchNotFound` 不再出现在新结果中 | `desktop/crates/node-protocol/tests/protocol/repository_results.rs::clone_failure_codes_cover_snapshot_outcomes` | Covered |
| 工作负载命令被终止且收尾确认后为 `Interrupted` | `desktop/apps/ora-node/tests/standalone/repository_recovery.rs::command_deadline_settles_clone_as_interrupted`、`::killed_clone_recovers_original_run_without_redispatch` | Covered |
| 工作负载命令以非 0 退出为 `OperationFailed` | 无直接测试 | Missing |
| 目标目录在创建时已存在则保留 Unknown，不报告 `DestinationConflict` 或成功 | 无直接测试 | Missing |

## Snapshot Fetch Attempts Share One Persistent Counter Across Restarts

- 风险：段内重试与重启后的重新拉取各自计数，最坏 9 次拉取，反复重启时 clone 迟迟不失败；拉取段在重启后写入目标目录；工作
  负载段的同一步被派发两次。
- 前置状态：Git Server 持续返回 `5xx`；可以在拉取中杀掉 Node。
- 触发：第一次拉取失败后杀掉 Node 并重启，直到失败；工作负载段的某一步中途重启。
- 必须成立：每次拉取开始前先持久化累计次数，段内重试与重启后的拉取共用这个计数，满 3 次以 `SourceUnavailable` 结束；重启后
  不等待剩余退避；阶段只向前推进；工作负载段每一步一个 Run，重启按原 Run 核对，不重跑。
- 禁止结果：总拉取次数超过 3；同一步出现第二个 Run；拉取段写入目标目录。
- 决策依据：D4。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 段内重试与重启后的拉取共用持久计数，满 3 次失败；重启后立即拉取 | `desktop/apps/ora-node/tests/standalone/repository_commit.rs::restart_during_the_fetch_pass_fetches_again_up_to_three_passes`、`::fetch_attempts_before_a_restart_count_toward_the_shared_limit`、`::restart_during_the_first_fetch_pass_completes_on_the_second` | Covered |
| 阶段只向前推进，下一步只在前一步的 Run 清理后记录 | `desktop/crates/node-db/src/tests/repository.rs::clone_phases_move_forward_through_the_snapshot_and_workload_steps`、`::clone_records_next_step_only_after_the_previous_run_is_cleaned` | Covered |
| 重启按工作负载步骤的原 Run 核对，不新建 Run | `desktop/apps/ora-node/tests/standalone/repository_recovery.rs::killed_clone_recovers_original_run_without_redispatch` | Covered |
| 拉取段失败时目标目录为空并按残留规则保留 | `desktop/apps/ora-node/tests/standalone/repository.rs::snapshot_failures_retain_targets_and_new_retry_uses_new_directory` 断言目录保留，未断言目录为空 | Partial |
| 重试间隔 5 秒、30 秒，单次拉取受 `network_timeout_ms` 约束 | 测试使用零间隔；没有直接测试 | Missing |

## Credentialed Git Never Reads The Checkout Configuration

- 风险：Node 身份的 Git 读取 Agent 写过的 `credential.helper`、`core.askPass`、`http.<url>.*` 或 `url.<base>.insteadOf`，以 Node
  身份执行任意命令、关闭证书校验或改写目标；私钥进入工作负载进程。
- 前置状态：checkout 中写入上述恶意配置与钩子，以及一个 `objects/info/alternates`。
- 触发：clone 之后的 `ora-git fetch`、`ora-git push`；Node 借用对象库。
- 必须成立：恶意配置与钩子都不生效，请求仍带会话自己的 `Ora-Execution-Id`；checkout 自带 alternates 时拒绝借用且不发起请求；
  私钥只以命令行配置引用，不出现在任何仓库配置、日志或结果中。
- 禁止结果：工作负载身份的进程打开私钥；传输仓库保存任何凭据。
- 决策依据：D2、D6、D7，不变量 4；交付推送决策 D1。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| checkout 中的 `credential.helper`、`core.askPass`、`http.<url>.sslVerify`、`http.<url>.extraHeader`、`url.<base>.insteadOf` 对 push 与 fetch 不生效 | `desktop/apps/ora-node/src/ora_git/tests.rs::hostile_checkout_configuration_has_no_effect_through_the_socket`；`desktop/apps/ora-node/src/git_server/tests.rs::checkout_configuration_has_no_effect_on_node_identity_git` | Covered |
| checkout 自带 alternates 时拒绝借用，不访问 Git Server | `desktop/apps/ora-node/src/ora_git/tests.rs::hostile_checkout_configuration_has_no_effect_through_the_socket` | Covered |
| checkout 中的钩子不生效 | clone 段见 `desktop/apps/ora-node/tests/standalone/repository_policy.rs::snapshot_clone_ignores_deployment_credentials_and_checkout_extensions`；`ora-git` 的 push 与 fetch 没有恶意钩子的测试 | Partial |
| 工作负载读不到 git-access 证书、私钥与 CA，Agent 环境中没有证书路径 | cluster 验收“5g” | Covered |
| 私钥不出现在仓库配置、日志与结果中 | checkout 配置见 cluster 验收“场景 1”与 `desktop/apps/ora-node/tests/standalone/repository.rs::managed_snapshot_clone_checks_out_the_base_commit_without_a_remote`；日志与结果没有直接测试 | Partial |

## ora-git Requests Belong To The Session That Owns The Socket

- 风险：任意进程伪造请求冒充别的会话；结束的会话仍能推送；非工作负载 UID 借 socket 访问 Git Server；文件系统权限挡住连接，
  让 Node 的检查无法验收；会话结束后在途请求仍写入 Git Server。
- 前置状态：两个会话各有自己的 socket；socket 目录为 `0711`，socket 为 `0666`。
- 触发：工作负载 UID 与另一个 UID（如 `nobody`）分别连接 socket；列出 socket 目录；`EndSession` 被接受后或会话结束后连接；
  同一会话并发两个请求；请求中夹带执行 ID 或地址；Node 重启。
- 必须成立：请求只归属于创建该 socket 的会话；另一个 UID 能连上但被 `SO_PEERCRED` 检查以 `forbidden` 拒绝；目录不能列出；
  `EndSession` 被接受后请求为 `no_session`；关闭时等待在途请求完成后删除 socket；并发请求为 `busy`；未知动词、选项、多余参数与
  未知字段为 `usage`；启动时只删除 Node 命名的 socket。
- 禁止结果：请求内容影响会话归属；Node 以外的检查成为唯一拒绝点。
- 决策依据：D5，不变量 5。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 请求不携带身份，夹带的执行 ID 等字段为 `usage` | `desktop/apps/ora-node/src/ora_git/tests.rs::socket_validates_requests_and_closes_with_the_session`；`desktop/apps/ora-git/src/lib.rs::wire_and_exit_codes_are_stable` | Covered |
| 对端 UID 不是工作负载 UID 时为 `forbidden`，且不访问 Git Server；文件系统层不挡连接 | `desktop/apps/ora-node/src/ora_git/tests.rs::socket_refuses_a_peer_other_than_the_workload_user`；cluster 验收“5g” | Covered |
| socket 目录为 `0711`、不能列出；启动只删除 Node 命名的 socket | `desktop/apps/ora-node/src/ora_git/tests.rs::the_socket_directory_loses_only_the_sockets_an_earlier_node_left`；cluster 验收“5g” | Covered |
| socket 路径经 `ORA_GIT_SOCKET` 交给 Agent 插件及其子进程 | `desktop/apps/ora-node/tests/standalone/agent_sessions.rs::echo_collaboration_markers_push_through_the_session_socket`；cluster 验收“5g” | Covered |
| `EndSession` 被接受后或会话结束后为 `no_session`，socket 被删除 | `desktop/apps/ora-node/src/ora_git/tests.rs::requests_fail_once_an_end_is_requested`、`::socket_validates_requests_and_closes_with_the_session`；cluster 验收“5f / 执行结束后再推” | Covered |
| 关闭时拒绝新请求、等待在途请求完成；并发请求为 `busy` | `desktop/apps/ora-node/src/ora_git/tests.rs::closing_waits_for_the_request_in_flight` | Covered |
| 未知动词、选项、多余参数为 `usage`（退出码 2） | `desktop/apps/ora-git/src/lib.rs::fixed_verbs_parse`、`::anything_else_is_a_usage_error`、`::wire_and_exit_codes_are_stable`；cluster 验收“5b / 写别人的分支” | Covered |
| socket 无法创建时会话以 `agent_failed{ora_git_unavailable}` 结束 | 无直接测试 | Missing |
| 请求超过 4 KiB 被拒 | 无直接测试 | Missing |
| WorkflowRun 的 Agent 会话访问 Git Server 的动词得到 `denied` | 授权由 Git Server 按 Cloud 状态判定；desktop 与 cloud 都没有直接测试 | Missing |

## ora-git Push Writes Only The Own Workspace Branch With A Declared Old Value

- 风险：Agent 推到别的 Workspace 或改写历史；旧值取自 Agent 可写的 ref；重放时把已写入的推送报告为冲突。
- 前置状态：Workspace B 的会话；账本记录 `workspaces/<B>/main` 的已知值。
- 触发：推送新提交；推送非快进的提交；远端被另一次推送前进后推送；上一次推送写入但账本未更新时重放；尝试 `--force` 或 refspec；
  本地分支不存在。
- 必须成立：只推 `refs/heads/workspaces/<B>/<branch>` 一个 ref，带账本中的旧值；成功后更新账本与 `refs/remotes/ora/workspaces/<B>/<branch>`，
  不改本地分支、HEAD、工作区与 index；非快进在本地判定为 `not_fast_forward`、不发起推送；旧值不符为 `conflict` 并给出远端值；远端
  已等于本次提交时按成功处理；`--force` 与 refspec 为 `usage`；Git Server 拒绝写别的 Workspace 的分支。
- 禁止结果：旧值取自 `refs/remotes/ora/**`；推送其他 ref 或 tag。
- 决策依据：D6，不变量 6、8。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 推送以账本中的已知值为旧值，旧值不符为 `conflict` 并给出远端值，fetch 自己的分支后可再推 | `desktop/apps/ora-node/src/ora_git/tests.rs::push_uses_the_ledger_and_fetch_repairs_a_stale_lease`；cluster 验收“5d / 旧值不符” | Covered |
| 非快进在本地判定为 `not_fast_forward`，不发起推送 | `desktop/apps/ora-node/src/ora_git/tests.rs::push_uses_the_ledger_and_fetch_repairs_a_stale_lease`；cluster 验收“5c / 非快进” | Covered |
| 账本已知值的对象在 checkout 中缺失时，推送带旧值发出并由 Git Server 决定 | 无直接测试 | Missing |
| 成功后更新账本与自己的跟踪 ref，不改本地分支、HEAD、工作区与 index | `desktop/apps/ora-node/src/ora_git/tests.rs::a_stale_lease_already_at_the_commit_is_success_and_pushes_leave_the_checkout`、`::push_uses_the_ledger_and_fetch_repairs_a_stale_lease` | Covered |
| 重放：远端已等于本次提交时按成功处理 | `desktop/apps/ora-node/src/ora_git/tests.rs::a_stale_lease_already_at_the_commit_is_success_and_pushes_leave_the_checkout` 走“远端已是 `C`”的路径；“旧值不符但远端等于 `C`”的分支没有直接测试 | Partial |
| 每次只推一个 ref，各种应答映射到更新、已是最新、旧值不符、被拒 | `desktop/apps/ora-node/src/git_server/tests.rs::push_reports_each_fate_of_the_ref`；`desktop/apps/ora-node/src/git_server/outcome.rs::porcelain_lines_name_the_fate_of_the_destination`；没有断言不推 tag | Partial |
| 被拒与 `401`/`403` 为 `denied` 不重试；暂时性失败共 3 次后为 `unavailable` | `desktop/apps/ora-node/src/git_server/tests.rs::denied_requests_fail_at_once_and_unavailable_ones_are_retried`；重试间隔没有直接测试 | Partial |
| 本地分支不存在为 `not_found` | `desktop/apps/ora-node/src/ora_git/tests.rs::missing_branches_are_not_found` | Covered |
| `--force` 与 refspec 为 `usage`；以 Node 身份直接写别的 Workspace 的分支被 Git Server 拒绝，ref 不变 | cluster 验收“5b / 写别人的分支”；`cloud/internal/gitserver/rules_test.go::TestFastForwardRuleOnWorkspaceBranches`；`cloud/internal/core/git_access_db_test.go::TestGitAccessNeverWidensBeyondTheCertificateWorkspace` | Covered |
| 会话结束后以会话执行 ID 推送得到 `403` | cluster 验收“5f / 执行结束后再推”；`cloud/internal/core/git_access_db_test.go::TestGitAccessSessionAccessEndsWithTheSession` | Covered |

## ora-git Fetch Writes Only Remote Tracking Refs

- 风险：fetch 改写 Agent 的本地分支、HEAD 或工作区；协商使用 Agent 可写的 ref；`list` 依据目录做判断。
- 前置状态：Git Server 上有 upstream 跟踪分支与其他 Workspace 的分支；checkout 有未提交改动。
- 触发：`ora-git fetch upstream`、`fetch workspace <id>`、`fetch workspace <id> <branch>`、取不存在的分支；`ora-git list`、`ora-git info`。
- 必须成立：只写 `refs/remotes/ora/upstream/*` 与 `refs/remotes/ora/workspaces/<id>/*` 和对象；协商只用账本与 `base_commit`；取自己的
  分支时更新账本；不存在为 `not_found`；`list` 按会话输入的目录附上 Workspace 信息，不在目录中的只显示 ID；`info` 不访问网络。
- 禁止结果：本地分支、HEAD、工作区、index 或仓库配置改变；目录参与任何判断。
- 决策依据：D5、D7，不变量 7、8。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| upstream 取自本 Binding，写入 `refs/remotes/ora/upstream/*` | `desktop/apps/ora-node/src/ora_git/tests.rs::fetch_upstream_reads_the_binding_heads` | Covered |
| 其他 Workspace 的分支写入 `refs/remotes/ora/workspaces/<id>/*`，被改写的跟踪 ref 被覆盖 | `desktop/apps/ora-node/src/ora_git/tests.rs::fetch_preserves_local_work_and_ignores_untrusted_tracking_refs`；cluster 验收“场景 3” | Covered |
| 本地分支、HEAD、工作区、index 与仓库配置不变 | `desktop/apps/ora-node/src/ora_git/tests.rs::fetch_preserves_local_work_and_ignores_untrusted_tracking_refs` 断言 HEAD、工作区状态与 remote；本地分支与完整配置没有比较 | Partial |
| 协商只用账本与 `base_commit` | 无直接测试 | Missing |
| 取自己的分支时更新账本 | `desktop/apps/ora-node/src/ora_git/tests.rs::push_uses_the_ledger_and_fetch_repairs_a_stale_lease`；cluster 验收“5d / 旧值不符” | Covered |
| 远端没有匹配的 ref 为 `not_found` | `desktop/apps/ora-node/src/ora_git/tests.rs::missing_branches_are_not_found`；`desktop/apps/ora-node/src/git_server/tests.rs::fetch_reads_into_the_transport_and_names_missing_refs` | Covered |
| `list` 按目录附上 Workspace 信息，目录外只显示 ID 并注明目录是会话开始时的快照 | `desktop/apps/ora-node/src/ora_git/tests.rs::push_uses_the_ledger_and_fetch_repairs_a_stale_lease` 只断言标题与 run；cluster 验收“Workspace 目录”；目录外的 Workspace 与快照说明没有直接测试 | Partial |
| `info` 不访问网络 | `desktop/apps/ora-node/src/ora_git/tests.rs::socket_validates_requests_and_closes_with_the_session` 断言没有请求与 Workspace ID；分支、`base_commit` 与前缀字段没有断言 | Partial |

## Delivery Fast Forward And ora-git Push Share One Ledger

- 风险：交付快进与 `ora-git push` 各自记旧值，交付后 Agent 下一次推送以过时的旧值失败；交付或恢复用的分支名与 `workspace_ref`
  不来自 clone 记录；恢复期间 Agent 已能使用 `ora-git`。
- 前置状态：Agent 推送过 Workspace 分支后进入交付；续作的会话。
- 触发：交付快进 `workspace_ref`；交付输入的 `workspace_ref` 与 clone 记录不同；续作恢复。
- 必须成立：快进以账本中的值为旧值，成功后更新同一份账本；不同的 `workspace_ref` 被拒且不访问 Git Server；恢复的分支短名与
  `base_commit` 取自 clone 记录；恢复在 Agent 插件启动前完成，不改 `refs/remotes/ora/**` 与配置。
- 禁止结果：存在第二份记录 Workspace 分支已知值的账本。
- 决策依据：D8、D9，不变量 9、11。

| 验证义务 | 证据 | 状态 |
| --- | --- | --- |
| 交付输入的 `workspace_ref` 与 clone 记录不同则拒绝，不访问 Git Server | `desktop/apps/ora-node/tests/standalone/agent_sessions/delivery.rs::delivery_for_another_workspace_branch_is_refused` | Covered |
| 快进以账本中的已知值为声明的旧值，重放与被拒不影响交付 | `desktop/apps/ora-node/src/revision/tests/push.rs::every_outcome_is_pushed_and_a_replay_succeeds`、`::a_refused_workspace_fast_forward_never_fails_the_delivery` 直接传入旧值，未验证从账本读取 | Partial |
| 快进成功后更新同一份账本，交付后 `ora-git push` 不以过时旧值失败 | 交付后的 Workspace 分支值见 `desktop/apps/ora-node/tests/standalone/agent_sessions/delivery.rs::ended_session_is_delivered_with_requested_grants` 与 cluster 验收“场景 4 / 交付更新 Workspace 分支”；账本写入与交付后的再次推送没有直接测试 | Missing |
| 账本按 checkout 与 ref 只记一份已知值 | `desktop/crates/node-db/src/tests/repository.rs::branch_heads_record_the_last_known_value_per_checkout` | Covered |
| 恢复的分支短名与 `base_commit` 取自 clone 记录，前序仓库必须等于 clone 的仓库 | `desktop/apps/ora-node/tests/agent_session/resume.rs::a_failed_restore_ends_the_session_before_any_plugin_starts` 只用默认分支名 `main`，`::a_prior_revision_from_another_repository_is_refused_before_any_restore`；cluster 验收“场景 4 / 续作的组合” | Partial |
| 恢复在 Agent 插件启动前完成，不改 `refs/remotes/ora/**` 与配置 | `desktop/apps/ora-node/src/revision/tests/restore.rs::restore_moves_the_branch_and_judges_divergence_by_the_snapshot` 断言跟踪 ref 与 remote 不变，未比较完整配置；`desktop/apps/ora-node/tests/agent_session/resume.rs::an_end_during_the_restore_cancels_it_before_any_plugin_starts`；恢复期间没有 socket 未直接断言 | Partial |
