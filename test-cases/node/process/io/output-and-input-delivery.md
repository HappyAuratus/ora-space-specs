# 有界结果输出与控制隔离

当前只有丢弃／有界内存捕获和偏移读取。保留原英文 anchor；标题中的 lease／input 不表示已交付。
延期能力的 Missing 不影响仅当前范围的 implemented，也不能被标成 Covered。

## Output Limits Preserve Each Channel Contract And Expose Gaps

### 风险与前置状态

Git 分别向 stdout／stderr 输出，消费者可离线；有限捕获不能被误解为完整或持久结果。

### 触发与必须成立

达到限额、直接退出但后代持管道、EOF 先于退出、host 重启后读取。
超限停止相应 Run，截断／失败可见；EOF 与直接退出分离；guardian 存活时仍能读取原前缀。

### 禁止结果

截断前缀按完整 Git 结果解析；输出无限缓存；guardian 死亡后声称内存输出可靠保存。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 精确限额与偏移、无消费者读取 | crates/process-runtime/tests/linux_output.rs::exact_limits_replay_and_partial_reads、readers_progress_without_consumer | Covered |
| 超限只停止所属 Run | crates/process-runtime/tests/linux_output.rs::overflow_stops_only_its_run | Covered |
| EOF 与退出分离 | crates/process-runtime/tests/linux_output.rs::descendant_pipe_outlives_direct_exit、eof_can_precede_process_exit | Covered |
| 真实 host 重启仍从原 guardian 读取 | apps/ora-process-host/tests/service.rs::host_kill_reconnect_stop_and_close_use_original_attempt | Covered |
| 插件背压、日志轮转、持久尾部 | 未来 I/O ADR；不属于当前结果捕获 | Missing |

决策依据：[I/O](../../../../decisions/node/process/io/0-bounded-output-and-fenced-input.md)。

## Input Delivery Uncertainty Cannot Cause Automatic Command Replay

范围：未来 stdin／插件会话 ADR 的验收输入，当前 stdin 关闭。

- 风险／前置：有副作用的插件命令已写管道但记录或回复可能丢失。
- 触发：写入、落盘、回复之间故障并重连。
- 必须成立：区分接受、写入与业务确认，交付未知不自动重发；未交付队列在实际写入点检查资格。
- 禁止结果：把回复丢失改写成未交付、撤权后写旧队列。
- 验证义务与证据：真实管道、插件确认及故障恢复均 Missing，恢复该能力时另行设计。
- 决策依据：I/O 根决策的未来 ADR 边界；不是当前 API 的承诺。

## Output Congestion Cannot Block Lease Revocation Or Stop

### 风险与前置状态

真实输出连接停滞，host／guardian 控制入口仍需可用。当前没有租约，保留旧 anchor 仅为引用稳定。

### 触发与必须成立

阻塞输出客户端，同时查询和停止；输出等待不占用控制锁或控制连接名额。

### 禁止结果

等待消费者才能停止运行、无界积压、以 I/O 存活伪装租约有效。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| host 输出停滞不阻止控制 | apps/ora-process-host/tests/service.rs::lost_reply_executes_once_and_stalled_output_does_not_block_control | Covered |
| guardian I/O 握手停滞不阻止控制 | apps/ora-process-guardian/tests/bootstrap.rs::stalled_io_probes_do_not_block_control | Covered |
| 租约／stdin 撤权与订阅背压 | 未来 control／I/O ADR | Missing |

决策依据：[I/O](../../../../decisions/node/process/io/0-bounded-output-and-fenced-input.md)。
