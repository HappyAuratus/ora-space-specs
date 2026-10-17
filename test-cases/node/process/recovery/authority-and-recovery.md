# 独占恢复、原实例重连与故障隔离

范围：当前 Linux 可信本机；recovery 根决策与 bootstrap 切片均为 implemented，完整故障矩阵的缺口按下表登记。
保留原英文 anchor，不因标题包含 Closes 就宣称 guardian 丢失时已有 Closed 证据。

## Guardian Bootstrap Transfers Exclusive Ownership Without Reinitializing State

### 风险与前置状态

全新或已有 Scope 路径，host 持原稳定锁；锁交接空窗、旧文件重建和 Ready 误判可能产生双实例。

### 触发与必须成立

错误路径／版本、并发创建、初始化与 Ready 边界失败、启动方强杀或丢失回复。
显式路径、原锁交接、提交后 Ready、原实例发现成立；管理描述符不传给工作负载。

### 禁止结果

隐式 HOME 定位、重建旧 journal、释放后重抢锁、重跑未知启动、把 Ready 当业务结果。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 接受前路径校验、文件保留及锁匹配 | crates/process-runtime/tests/host_state.rs::creation_and_recovery_never_reinitialize_existing_paths、recovery_rejects_links_permissions_and_unknown_files；apps/ora-process-guardian/tests/bootstrap.rs::invalid_bootstrap_preserves_scope_and_never_publishes_endpoints | Covered |
| 初始化提交后 Ready，丢回复及启动方强杀仍发现原实例 | apps/ora-process-guardian/tests/bootstrap.rs::guardian_commits_before_ready_and_rejects_unauthorized_probes、launcher_kill_preserves_original_guardian_and_lost_ready_is_queryable、failed_exec_is_not_retried_after_recovery | Covered |
| 工作负载不持管理锁、guardian 死亡仍不重启原尝试 | apps/ora-process-guardian/tests/bootstrap/runs.rs::guardian_kill_preserves_attempt_and_workload_does_not_hold_scope_lock；crates/utils/tests/linux_file_lock.rs::detached_exec_closes_unintended_inheritable_lock | Covered |
| 真实 host／Node 使用显式目录 | apps/ora-process-host/tests/service.rs::endpoint_ownership_and_unknown_file_preservation；apps/ora-node/tests/standalone.rs::node_data_directory_cannot_alias_host_state | Covered |

覆盖上述具体失败边界，不声称穷尽任意指令窗口或物理掉电。
决策依据：[bootstrap](../../../../decisions/node/process/recovery/20260917-rootless-guardian-bootstrap-and-reconnect.md)。

## Exclusive Recovery Precedes Opening Management Operations

### 风险与前置状态

原目录与实例仍可能有效，两个启动者竞争；连接失败不能当作恢复资格。

### 触发与必须成立

并发恢复、缺锁／journal、回复丢失；唯一持锁者推进宿主绑定，原锁 inode 保留，不凭锁空闲放行业务。

### 禁止结果

竞争 guardian、迁移换锁、恢复前接受副作用、删 endpoint 绕过原实例。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 创建竞争、原锁与缺文件拒绝 | crates/process-runtime/tests/host_state.rs::concurrent_creation_has_one_owner、recovery_keeps_original_creation_intent_and_stable_lock、missing_lock_or_journal_is_not_recreated | Covered |
| 实际宿主竞争及旧文件保留 | apps/ora-process-host/tests/service.rs::endpoint_ownership_and_unknown_file_preservation | Covered |
| 所有暂停／记录缺损／恢复阶段竞争组合 | 尚无完整直接矩阵 | Partial |

决策依据：[恢复](../../../../decisions/node/process/recovery/0-single-authority-and-failure-recovery.md)。

## Host Failure Preserves Each Scope Survival Policy

### 风险与前置状态

guardian 持有运行和输出，host 可被强杀；旧会话可能继续提交。

### 触发与必须成立

host 强杀重启并重连原 guardian，持久绑定提交后使旧通道／执行队列失效；
选定的断线策略不能由死亡 host 执行。Controller 租约不在本次范围。

### 禁止结果

重复 Run、新绑定未提交即确认、把宿主代次当 Node 授权、历史 Running 当实时事实。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 原 Run／输出跨 host 强杀恢复 | apps/ora-process-host/tests/service.rs::host_kill_reconnect_stop_and_close_use_original_attempt | Covered |
| 持久新绑定和三旧通道失效 | apps/ora-process-guardian/tests/bootstrap/management.rs::durable_takeover_fences_old_channels_and_replays_original_session | Covered |
| 提交失败与排队请求边界 | crates/process-runtime/src/guardian/management_tests.rs::persistence_failure_and_reopen_never_reset_authority、queued_request_checks_binding_after_takeover | Covered |
| 所有断线政策和存储故障组合 | 正向与完整故障矩阵未齐 | Partial |
| 服务管理器、机器重启、租约 | 未来部署／control ADR | Missing |

决策依据：恢复根决策及 bootstrap D3。

## Guardian Failure Closes Its Scope Without Inventing Results

### 风险与前置状态

guardian 被强杀、工作负载仍活着且身份句柄已丢失；另有不冲突任务。

### 触发与必须成立

尝试原 Scope 恢复；不启动替代 guardian，不按旧数字 PID 补杀，不将失联当 Closed；
Node 保留相关 Unknown 和预留，其他不冲突工作继续。

### 禁止结果

伪造退出码／清理完成、重开旧 Scope、从锁释放推定后代退出。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| guardian 强杀保留尝试且工作负载不持 Scope 锁 | apps/ora-process-guardian/tests/bootstrap/runs.rs::guardian_kill_preserves_attempt_and_workload_does_not_hold_scope_lock | Covered |
| 未知阻塞局部资源，其他工作可继续 | apps/ora-node/tests/standalone.rs::guardian_loss_keeps_conflicting_recovery_unknown_but_allows_other_work | Covered |
| 暂停恢复、数字身份强制复用的完整竞态 | 尚缺直接矩阵；不会以现有测试宣称已覆盖 | Missing |
| guardian 死亡后独立清理及机器重启恢复 | 未来 recovery／平台 ADR | Missing |

决策依据：恢复根决策及 bootstrap D4；不把无法自动清理当作已完成清理。
