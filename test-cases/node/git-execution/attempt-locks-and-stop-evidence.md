# Node 直接执行 Git 的核心用例

依据[已批准的 Node 直接执行 Git 决策](../../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)。
下文的 D 编号、不变量编号和"验收义务"编号都指这篇 ADR。

本文覆盖尝试锁给出的停止证据、锁的归属、命令收尾、终止线索、`ora-reaper`、重启收口、升级和常驻进程。clone、
续作恢复和交付各自的业务义务登记在 [repository](../repository/minimal-clone-execution.md) 和
[revision](../revision/restore-deadline-and-stop-confirmation.md) 的用例里，本文的证据不替代它们。

证据按 desktop 实现分支的提交 `3e0b5607` 核对，该提交的 CI 全部通过，尚未合入 desktop 的主干。测试路径相对
desktop 仓库根目录。需要 root 才能运行的测试默认被忽略，不在 CI 里运行，不算直接证据，对应义务如实标为 Partial。

| 验收义务 | 用例 | 状态 |
|---|---|---|
| 1、2、10 | [A Held Attempt Lock Is The Only Proof That A Command May Still Run](#a-held-attempt-lock-is-the-only-proof-that-a-command-may-still-run) | Covered |
| 4、5 | [An Attempt Lock Reaches Only Its Own Command](#an-attempt-lock-reaches-only-its-own-command) | Partial |
| 3、6 | [A Command Returns Only After Its Stop Is Proven](#a-command-returns-only-after-its-stop-is-proven) | Partial |
| 7 | [A Kill Clue Ends Only The Processes It Still Identifies](#a-kill-clue-ends-only-the-processes-it-still-identifies) | Covered |
| 8 | [The Reaper Ends Git When The Node Dies](#the-reaper-ends-git-when-the-node-dies) | Partial |
| 9、11 | [A Restarted Node Settles Old Attempts Before It Recovers](#a-restarted-node-settles-old-attempts-before-it-recovers) | Partial |
| 12 | [An Upgrade Settles What The Retired Host Left](#an-upgrade-settles-what-the-retired-host-left) | Covered |
| 13 | [A Finished Clone Leaves No Resident Process](#a-finished-clone-leaves-no-resident-process) | Partial |

## A Held Attempt Lock Is The Only Proof That A Command May Still Run

### 风险

Node 用超时、已发出终止信号、进程组已空或"查不到记录"推断旧命令已停止，在旧 Git 仍可能写入时读取目录、释放输入
或开始新命令。

### 前置状态

一条有副作用的 Git 命令已建好锁文件并记下尝试；可制造 Node 在启动 Git 之后、写下终止线索之前被强杀，后代换到别的
进程组或会话，以及锁文件缺失。

### 触发

Node 被强杀后重启并探测；直接子进程退出而后代仍在运行；探测一个没有锁文件的尝试。

### 必须成立

- 只要还有进程持有继承来的 fd，探测结果就是未停止；这些进程全部结束后，探测结果变为已停止。
- 结论不依赖终止线索：线索没写下来时，结论一样正确。
- 后代换了进程组或会话，仍被覆盖。
- 锁文件不存在或探测出错时，结论是未知。

### 禁止结果

- 把超时、终止信号已发出、进程组已空、reaper 已退出或拿到数据库锁当成停止证据。
- 把锁文件不存在解释为"没有进程在跑"。
- 旧命令仍持有锁时，clone 得出失败或成功的结论。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 没有终止线索时，进程在则未停止，进程结束后已停止（义务 1） | `apps/ora-node/src/managed/attempt_lock_tests.rs` 的 `a_descendant_in_another_session_keeps_the_lock_until_it_ends`：不写线索，后代存活时为 Running，结束后为 Stopped；`apps/ora-node/tests/standalone/repository_recovery.rs` 的 `clone_stays_unknown_while_its_git_outlives_the_node`：杀掉 reaper、删掉线索后强杀真实 Node，旧 Git 存活期间 clone 保持 Unknown，结束后同一尝试收口 | Covered |
| 后代换了进程组或会话仍被覆盖（义务 2） | 同上 `a_descendant_in_another_session_keeps_the_lock_until_it_ends`（`setsid` 的后代）；`apps/ora-node/tests/standalone.rs` 的 `escaped_descendant_keeps_conflicting_recovery_unknown_but_allows_other_work`：真实 Node 上冲突的执行保持 Unknown，不冲突的任务照常进行，后代结束后恢复 | Covered |
| 锁文件不存在时保持未知（义务 10） | `attempt_lock_tests.rs` 的 `lock_files_are_private_and_a_missing_one_is_unknown`；`apps/ora-node/src/managed/restore_tests.rs` 的 `unknown_stop_expires_without_terminal_then_original_evidence_settles_once`；`apps/ora-node/tests/standalone/agent_sessions/unknown_delivery.rs` 的 `missing_lock_blocks_complete_delivery_before_and_after_bound_and_restart` | Covered |

### 决策依据

D2、D4；不变量 2、3、4。

## An Attempt Lock Reaches Only Its Own Command

### 风险

尝试锁的 fd 泄漏给 Node 同时启动的 Agent 会话或别的 Git，无关进程一直占着锁，尝试停在未知；或者工作负载用户自己
打开锁文件占住锁。

### 前置状态

Node 进程里有一把已加锁的尝试锁；Node 同时在启动别的子进程；可配置独立的工作负载用户。

### 触发

持锁期间启动另一个子进程；降权到工作负载用户启动 Git；工作负载用户尝试自己打开锁文件。

### 必须成立

- 只有 spec 里指明要继承这个 fd 的子进程拿到它，同时启动的其他子进程拿不到。
- 降权后的 Git 仍然继承尝试锁。
- 锁目录权限 0700，锁文件权限 0600，属主是 Node 的运行用户；配置了独立工作负载用户时，该用户打不开锁文件。

### 禁止结果

- 无关子进程持有尝试锁。
- 降权步骤把尝试锁的 fd 一并关掉，Git 没有持锁就开始运行。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| fd 不泄漏给同时启动的其他子进程（义务 4） | `crates/process/tests/descriptor.rs` 的 `only_the_named_child_inherits_the_descriptor`：父进程持有 fd 时启动的旁路子进程没有拿到锁。缺口：只在 `ora-process` 一层证明，没有"真实 Node 同时启动 Agent 会话"的用例 | Partial |
| 锁目录和锁文件只有 Node 的运行用户能打开（义务 5） | `attempt_lock_tests.rs` 的 `lock_files_are_private_and_a_missing_one_is_unknown` 断言 0700 和 0600 | Covered |
| 降权的 Git 继承锁，工作负载用户打不开锁文件（义务 5） | `apps/ora-node/src/managed/exec_tests.rs` 的 `workload_git_inherits_a_lock_it_cannot_open` 和 `descriptor.rs` 的 `a_dropped_identity_keeps_only_the_named_descriptor` 需要 root，默认忽略，不在 CI 里运行 | Partial |

### 决策依据

D1、D2、D8 第 2 条；不变量 5。

## A Command Returns Only After Its Stop Is Proven

### 风险

Git 以 0 退出或期限到期后，Node 在后代仍可能写入时把结果交给业务层；或者 Git 的后台维护脱离出去占着锁，把停止确认
拖到维护结束。

### 前置状态

Node 正在执行一条有副作用的 Git 命令；可制造期限到期、Git 退出后进程组里留下后代、后代脱离进程组。

### 触发

命令期限到期；Git 以 0 退出但进程组里还有后代；后代脱离进程组后 Git 退出；执行会触发后台维护的命令。

### 必须成立

- 期限到期后整组终止，停止证据成立后才返回结果。
- Git 退出后，进程组里留下的进程被终止，结果等到停止证据成立才返回。
- 清理期限内停止证据不成立时，结果是未核实：退出码 0 不作为成功上报，锁文件保留。
- 每条 Git 命令都关闭了后台 gc 和 maintenance，它的子进程看到同样的设置。
- 输出超过上限时命令按失败处理。

### 禁止结果

- 后代仍持有锁时把退出码 0 当作成功交给业务层。
- 停止证据成立之前删除锁文件。
- fetch 之后有脱离的维护进程占着锁。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 期限到期后整组终止，停止证据成立后才返回（义务 3） | `exec_tests.rs` 的 `a_deadline_ends_the_group_before_the_result_returns`；`repository_recovery.rs` 的 `command_deadline_settles_clone_as_interrupted` | Covered |
| Git 以 0 退出后清掉组内残留再返回 | `exec_tests.rs` 的 `a_clean_exit_ends_what_git_left_in_its_group` | Covered |
| 清理期限内不成立时结果为未核实（义务 3） | `exec_tests.rs` 的 `a_descendant_outside_the_group_blocks_the_next_command`：退出码 0 不上报，锁保留 | Covered |
| 每条命令关闭后台 gc 和 maintenance（义务 6） | `exec_tests.rs` 的 `every_command_disables_background_maintenance`：命令和它的子进程读到 `gc.auto=0`、`maintenance.auto=false`。缺口：没有"真实 fetch 达到 gc 阈值后没有脱离进程占着锁"的用例 | Partial |
| 输出超过上限按失败处理 | `exec_tests.rs` 的 `output_beyond_the_limit_fails_the_command` | Covered |
| 尝试按 Git 的实际结束方式收尾 | `exec_tests.rs` 的 `a_dispatched_attempt_settles_by_how_git_ended`：退出码、终止信号、没有退出状态三种 | Covered |

### 决策依据

D2、D3；不变量 2、3、9。

## A Kill Clue Ends Only The Processes It Still Identifies

### 风险

Node 重启后按过期的终止线索杀进程，杀到复用了同一个 pid 的无关进程；或者组长已退出就认为进程组没了，留下组员继续
运行。

### 前置状态

一条命令正在运行，终止线索已写下；可改写线索里的启动时间和本机启动标识，可让组长先退出。

### 触发

按启动时间对不上的线索终止；按另一次开机留下的线索终止；组长退出后按线索终止。

### 必须成立

- 启动时间对不上或本机启动标识不一致时，不发任何信号。
- 线索仍然指向原进程时，整组终止。
- 组长已退出、组员还在时，按线索整组终止。

### 禁止结果

- 向 pid 已被复用的进程发信号。
- 因为线索缺失或对不上而改变停止证据的结论。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| pid 复用或跨开机时不误杀（义务 7） | `attempt_lock_tests.rs` 的 `a_clue_that_no_longer_identifies_its_process_kills_nothing` | Covered |
| 组长已退出时仍能整组终止（义务 7） | `attempt_lock_tests.rs` 的 `a_clue_ends_the_members_its_exited_leader_left` | Covered |
| 重启的 Node 按线索终止原命令 | `restore_tests.rs` 的 `a_restarted_node_ends_and_confirms_the_original_restore_command` | Covered |

### 决策依据

D2 的终止线索、D4；不变量 6。

## The Reaper Ends Git When The Node Dies

### 风险

Node 被强杀后没有东西终止它启动的 Git，Git 一直运行到自己结束；或者 Node 带着失效的 reaper 继续启动新的 Git。

### 前置状态

Node 已拉起 `ora-reaper`，正在执行的 Git 已向它登记。

### 触发

强杀 Node 且不再启动新的 Node；配置一个拉不起来的 reaper 启动 Node；reaper 在运行中自己退出。

### 必须成立

- Node 死亡后，reaper 终止仍登记着的进程组。
- reaper 拉不起来时 Node 启动失败。
- reaper 在运行中死亡后，Node 不再启动新的 Git，以明确错误退出。
- Agent 会话进程不向 reaper 登记。

### 禁止结果

- 把 reaper 已退出当成停止证据。
- Node 在没有 reaper 的情况下启动或继续启动 Git。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| Node 被强杀后 reaper 终止它的 Git（义务 8） | `standalone.rs` 的 `reaper_stops_git_without_node_restart` | Covered |
| reaper 拉不起来时 Node 启动失败（义务 8） | `standalone.rs` 的 `node_does_not_start_without_its_reaper` | Covered |
| 启动方可以让自己的子进程不登记 | `apps/ora-reaper/tests/registration.rs` 的 `a_spawner_can_keep_its_children_out_of_the_reaper` | Covered |
| reaper 运行中死亡后 Node 停止启动 Git 并退出 | `apps/ora-reaper/tests/exit.rs` 的 `a_reaper_that_exits_on_its_own_is_reported` 只证明能看出 reaper 自己退出了。缺口：没有"真实 Node 在 reaper 死亡后拒绝新命令并退出"的用例 | Partial |

### 决策依据

D1；不变量 2。

## A Restarted Node Settles Old Attempts Before It Recovers

### 风险

Node 重启后直接进入业务恢复，在旧 Git 仍在运行时重新派发、读取目标目录或开始新的交付，两条命令同时写同一个目录。

### 前置状态

上一次运行留下了未收尾的尝试和锁文件：clone、续作恢复各有数据库记录，交付和 `ora-git` 没有。

### 触发

强杀 Node 后立即重启；重启时旧命令已停止、仍在运行、或终止手段全部失效。

### 必须成立

- 收口先于任何业务恢复：有终止线索且身份一致时整组终止再探测，否则直接探测。
- 停止证据成立时，clone 以 `Interrupted` 结束并保留目录，原尝试不重新派发；续作恢复视为已确认停止并释放输入。
- 停止证据不成立时保持未知，保留目录和输入，不影响不冲突的其他工作。
- 交付、`ora-git` 和传输仓库的旧命令仍占着锁时，不开始新的同类命令。

### 禁止结果

- 以 Node 已重启、reaper 已退出或等待超时推断旧命令已停止。
- 把"已停止，没有退出状态"上报为成功。
- 旧的交付命令仍在运行时开始新的交付。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 收口先于业务恢复（义务 9） | `standalone.rs` 的 `node_kill_and_immediate_restart_wait_for_old_git_cleanup`；`repository_recovery.rs` 的 `killed_clone_recovers_original_attempt_without_redispatch` | Covered |
| clone 以 `Interrupted` 结束并保留目录（义务 9） | `repository_recovery.rs` 的 `clone_stays_unknown_while_its_git_outlives_the_node`、`stopped_node_settles_its_interrupted_clone_before_exit` | Covered |
| 续作恢复在停止证据成立后释放输入，未知时保留（义务 9） | `restore_tests.rs` 的 `a_restarted_node_ends_and_confirms_the_original_restore_command`；`apps/ora-node/src/service/agents/tests.rs` 的 `startup_cleanup_keeps_restore_inputs_until_original_stop_is_confirmed` | Covered |
| 交付的旧命令仍占着锁时不开始新的交付（义务 11） | `exec_tests.rs` 的 `a_descendant_outside_the_group_blocks_the_next_command` 和 `attempt_lock_tests.rs` 的 `a_leftover_blocks_until_its_command_is_gone` 在执行层证明：遗留的锁挡住下一条没有数据库记录的命令，旧命令结束后放行。缺口：没有"真实 Node 重启后旧 push 仍在运行、新的交付不开始"的集成用例 | Partial |

### 决策依据

D4、D5；不变量 1、3、7、8。

## An Upgrade Settles What The Retired Host Left

### 风险

从经 host 执行 Git 的版本升级后，旧方式记录的未收尾尝试永远停在未知，或者新 Node 拒绝启动、要求人工清理。

### 前置状态

Node 数据库里有按旧方式记录、尚未收尾的 clone 和续作恢复尝试；旧 host 目录仍在；旧的 Node、host、guardian 进程树
已经结束。

### 触发

新版本的 Node 第一次启动。

### 必须成立

- 新 Node 能启动。
- 旧尝试按"已停止，没有退出状态"收口：clone 记为 `Interrupted`，续作恢复视为已确认停止。
- host 目录被删除；已有的执行历史和结果保留。
- Node 的数据目录不能放在旧 host 目录里。

### 禁止结果

- 迁移改写已有的结果行，或者把旧尝试上报为成功。
- 因为存在旧 host 目录或旧尝试而拒绝启动。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 新 Node 能启动，旧尝试被收口，host 目录被删除（义务 12） | `repository_recovery.rs` 的 `upgrade_settles_the_retired_hosts_attempt_and_removes_its_directory` | Covered |
| 数据库迁移保留历史并收口旧尝试 | `crates/node-db/src/tests/repository_migration.rs` 的 `version_two_upgrade_settles_the_host_attempt_and_preserves_legacy_results`；`crates/node-db/src/tests/session/restore.rs` 的 `upgrade_confirms_the_stop_of_a_restore_attempt_the_host_left` | Covered |
| 数据目录不在待删除的 host 目录里 | `standalone.rs` 的 `node_data_directory_cannot_live_in_the_retired_host_directory` | Covered |

### 决策依据

D7；不变量 8。

## A Finished Clone Leaves No Resident Process

### 风险

clone 完成后仍留下常驻进程或锁文件，开销随使用量增长，退回被取代的体系当初的问题。

### 前置状态

Node 已启动并拉起 reaper。

### 触发

一次 clone 执行到完成。

### 必须成立

- clone 完成后，Node 的子进程只有 reaper。
- 没有留下锁文件和终止线索。
- 一次 clone 不再有经 host 轮询带来的固定等待。

### 禁止结果

- 每次 clone 留下随次数累积的进程或文件。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 完成后只剩 reaper，没有锁文件（义务 13） | `repository_recovery.rs` 的 `completed_clone_leaves_no_process_but_the_reaper` | Covered |
| 固定等待的前后对照（义务 13） | 没有自动化用例；前后对照是一次性的实测，不由测试守护 | Missing |

### 决策依据

D1、D3；背景。
