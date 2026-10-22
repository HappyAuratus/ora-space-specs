# 运行责任与业务身份隔离

范围：Linux 可信本机独立 Node／host／guardian；不包含 Controller 授权或插件迁移。

## Runtime Responsibility Survives Node Instance Replacement

### 风险与前置状态

已接受 Git 变更并持久关联 Scope／Run，独立 guardian 持有仍在运行的 Git hook。
若把 Node 内存当权威，重启可能重复副作用或抢先修复资源。

### 触发与必须成立

强杀 Node 并立即重启，或强杀 host 再重连。恢复原身份，先核实旧尝试收尾再做 Git 对账。
guardian 丢失时保留局部未知，不以旧结果放行冲突任务，不阻塞无冲突任务。

### 禁止结果

重连生成第二次 Run；把历史 Running 当实时证据；把退出码直接当业务成功；全 Node 停摆。

### 验证义务与证据

| 义务 | 直接证据（实现仓库相对路径） | 状态 |
|---|---|---|
| Node 更替保留原执行并等待旧 Git 收尾 | apps/ora-node/tests/standalone.rs::node_kill_and_immediate_restart_wait_for_old_git_cleanup | Covered |
| host 更替恢复原 Run，而非重新执行 | apps/ora-process-host/tests/service.rs::host_kill_reconnect_stop_and_close_use_original_attempt | Covered |
| guardian 丢失保留未知而无冲突工作继续 | apps/ora-node/tests/standalone.rs::guardian_loss_keeps_conflicting_recovery_unknown_but_allows_other_work | Covered |
| 已确认关闭的 Scope 不持续占满交换容量，独立 Scope 仍能被派发 | [host 回归][settled-host] `closed_scope_guardians_cannot_block_a_fresh_scope`：32 个已有原身份、已确认关闭但不应答的 guardian 下，新 Scope 在受控期限内到达真实 guardian 启动登记边界；不将派发等同于业务完成 | Covered（desktop#720，合并提交 `df5da60`） |

[settled-host]: https://github.com/ora-space/desktop/blob/df5da60297f8e3af6a271d17504dd34c4596e1e7/crates/process-runtime/src/host.rs

平台为本机 Linux BestEffort；不证明未跟踪后代退出、非 Linux、服务管理器或掉电保证。
决策依据：[架构](../../../../decisions/node/process/architecture/0-node-host-and-scope-guardians.md)。
