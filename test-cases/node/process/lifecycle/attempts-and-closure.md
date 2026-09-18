# 启动尝试、收尾与关闭并发

当前 Linux 已有持久 host／guardian 与真实 Node Git 证据；下面完整竞争义务仍保留缺口。
Controller 授权／租约和插件关闭不属于当前范围。三类停止策略区分内核接口与 host 仅强停的远程接口。

## Replaying A Run Never Creates A Second Launch Attempt

### 风险

接受、spawn 与记录不是原子事务，回复丢失后重跑会重复外部副作用。

### 前置状态

Node 已持久关联 Execution／Scope／RunId 和参数；宿主与 guardian 可在接受、纳管、spawn、事实提交及回复处被外部强杀。

### 触发

逐个切断窗口并以原 RunId 重连查询／重传，同时提交一次同身份不同参数请求。

### 必须成立

- 同身份同参数恢复原尝试，不同参数冲突；有完成记录返回原结果，有原运行恢复观察。
- 当前远程已接受尝试即使 exec 失败也不重复执行；启动未知进入核实与收尾，不能重写为未执行。
- 即使尽力清理和对账允许新工作，原未知事实仍保留；新业务重试由 Node 决定并使用新 RunId。

### 禁止结果

回复丢失就第二次 spawn；启动未知时换平台实现重启；运行层自行生成业务重试身份。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 接受／spawn／落盘／回复窗口故障后原身份不会重复启动 | apps/ora-process-host/tests/service.rs::lost_reply_executes_once_and_stalled_output_does_not_block_control；apps/ora-process-guardian/tests/bootstrap/runs.rs::run_journal_failures_never_retry_an_accepted_exec、lost_reply_takeover_force_stop_and_close 已覆盖主要窗口，任意接受／执行强杀点未穷尽 | `Partial` |
| 未启动、已启动、已完成和未知分别驱动恢复，冲突参数拒绝 | `crates/process-runtime/tests/launch.rs::unknown_launch_replays_original_facts_and_rejects_changed_parameters` 验证未知重放及参数冲突；`crates/process-runtime/tests/facts.rs` 的 `discovering_original_process_confirms_launch_without_starting_again`、`observed_launch_cannot_be_rewritten_as_not_started` 验证受控平台事实确认与拒绝矛盾事实；仍需持久恢复和其他启动事实验证 | `Partial` |

### 决策依据

[根决策](../../../../decisions/node/process/lifecycle/0-durable-run-attempts-and-scope-closure.md) D1、D2 与不变量 1。

## Scope Closure Cannot Lose A Concurrently Accepted Run

### 风险

关闭扫描与并发启动之间的窗口可能产生未纳入收尾的运行。

### 前置状态

一个开放 scope 已有运行，新 Run 正等待接受；关闭流程可停在封闭入口和扫描责任之间。

### 触发

在显式同步点交错提交启动与关闭，再重传关闭及尝试重开。

### 必须成立

- 并发 Run 要么拒绝，要么被接受并加入本次收尾；关闭先封闭新运行入口。
- 关闭方向不可逆，关闭后不重开；开放 scope 暂无活进程不被当作已关闭。

### 禁止结果

关闭完成后存在已接受但未管理的 Run；空 scope 自动关闭或旧 scope 被重开。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 每个接受的并发启动均属于关闭责任，无遗漏窗口 | 尚无直接证据；需真实启动与受控状态转换竞争测试 | `Missing` |
| 关闭单向且持久，开放空 scope 与已关闭 scope 区分 | `crates/process-runtime/tests/lifecycle.rs::closure_seals_launches_and_waits_for_cleanup_evidence` 验证关闭封闭准入、原身份重放和证据完成；仍需持久重启及空 Scope 验证 | `Partial` |

### 决策依据

[根决策](../../../../decisions/node/process/lifecycle/0-durable-run-attempts-and-scope-closure.md) D3 与不变量 2。

## Direct Exit And Single Run Termination Preserve Scope Boundaries

### 风险

把直接进程退出当成全部完成会遗漏后台后代；把单 Run 终止实现为总组清空会误杀其他 Run。

### 前置状态

同一 scope 中 Run A 的启动器会退出而后代继续写入，Run B 正常运行；分别配置默认收尾与等待全部受管进程策略。

### 触发

让 A 的直接进程退出，随后单独终止 A；另构造无法核实 A 局部归属的情况。

### 必须成立

- 默认策略保存直接结果并限时收尾后代；等待全部策略继续管理后代与 I/O，直到结束或终止。
- 单 Run 终止不影响 B 或隐式关闭 scope；确需扩大 scope 收尾时明确报告，不能声称仅处理 A。

### 禁止结果

启动器退出码代替后代业务成功或收尾完成；单 Run 完成提前释放共同资源；静默扩大终止范围。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 两种退出策略分别遵守后代处理和完成阶段 | `crates/process-runtime/tests/lifecycle.rs::direct_exit_cleans_descendants_only_under_the_cleanup_policy` 验证受控退出事实下的通知、期限和等待；另有 crates/process-runtime/tests/linux_best_effort.rs::direct_exit_policy_cleans_background_descendants、direct_exit_keeps_background_descendants_pending_until_stopped 与 linux_output.rs::descendant_pipe_outlives_direct_exit；其他平台延期 | `Partial` |
| 单 Run 清理保持 B 存续，扩大范围可观察 | `crates/process-runtime/tests/lifecycle.rs::stopping_one_run_never_extends_its_deadline_or_stops_its_neighbor` 验证单 Run 信号边界；另有 crates/process-runtime/tests/linux_best_effort.rs::stopping_one_run_preserves_its_neighbor；完整故障升级仍缺失，其他平台延期 | `Partial` |

### 决策依据

[根决策](../../../../decisions/node/process/lifecycle/0-durable-run-attempts-and-scope-closure.md) D1、D3 与不变量 3。

## Stopping And Cancelling Waits Cannot Abandon Accepted Responsibility

### 风险

重复停止延长期限、取消等待取消操作或重试耗尽释放责任，会留下无人管理的副作用。

### 前置状态

已持久接受的运行可等待、接受进程级通知或强制收尾；存在可注入的瞬时和确定性清理错误。

### 触发

分别提交三种停止策略，再重复提交较长／较短期限、升级停止、取消等待，暂停重试并恢复条件。

### 必须成立

- 限时等待不提前打断，请求退出策略先通知，立即强制策略不等待；重复停止只缩短或升级。
- Node 正常停止先停派发并等待已有 Git，owner 身份退出触发清理；Controller 租约与插件业务关闭由未来 ADR 负责。
- 取消等待不取消已接受责任；瞬时错误有界退避，确定性错误显式阻塞，条件恢复后继续核实与清理。

### 禁止结果

重复请求延长期限或撤销关闭；guardian 伪造插件业务关闭；超时／重试耗尽自动释放资源。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 三种停止策略和升级规则对真实运行生效 | `crates/process-runtime/tests/lifecycle.rs::stopping_one_run_never_extends_its_deadline_or_stops_its_neighbor` 验证受控时间下通知和到期强制；`closure_seals_launches_and_waits_for_cleanup_evidence` 验证强制关闭；另有 apps/ora-node/tests/standalone.rs::shutdown_bounds_a_blocked_git_hook、shutdown_allows_git_to_finish_within_grace；更多升级组合未齐 | `Partial` |
| 取消等待、暂停或重试耗尽不解除责任，条件恢复后能推进 | `crates/process-runtime/tests/lifecycle.rs::cleanup_failure_recovers_without_erasing_exit_evidence` 验证信号错误后责任保留与恢复；`unavailable_observation_does_not_prevent_force_stop` 验证观测失败时仍发送强制动作；持久接受、取消等待、退避与重启未验证 | `Partial` |

### 决策依据

[根决策](../../../../decisions/node/process/lifecycle/0-durable-run-attempts-and-scope-closure.md) D4 与不变量 3。
