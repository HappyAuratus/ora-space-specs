# 本机分流、绑定与查询恢复

当前范围为 Linux 有界请求／回复和输出读取，不含事件订阅、游标或结果接管协议。
旧英文 anchor 保留；Authenticated 在当前表示 OS 对端及宿主绑定校验，不是秘密凭据认证。

## Every Local Stream Must Bind To The Same Authenticated Session

### 风险与前置状态

真实 host／guardian 通道已有宿主绑定；错误版本、错 Scope、旧会话或迟到请求可能错误执行。

### 触发与必须成立

发送不完整／超限消息、错误版本、错对端、接管后旧通道字节；丢失 Start 回复后按原身份恢复。
拒绝错误边界，三 guardian 通道核对原绑定；host 输出与控制分离，同 Run 不重复执行。

### 禁止结果

可解码即兼容、旧队列接管后执行、输出阻塞停止；把 Events 端点当已实现订阅。

### 验证义务与证据

| 义务 | 直接证据 | 状态 |
|---|---|---|
| 有界完整 framing、未知字段／身份拒绝 | crates/process-protocol/tests/guardian_wire.rs::framing_requires_one_complete_bounded_message、unknown_fields_and_noncanonical_identities_are_rejected | Covered |
| 真实 host 错误版本／对端拒绝 | apps/ora-process-host/tests/service.rs::incompatible_requests_and_wrong_peer_are_rejected | Covered |
| guardian 旧三通道及迟到字节失效 | apps/ora-process-guardian/tests/bootstrap/management.rs::durable_takeover_fences_old_channels_and_replays_original_session | Covered |
| 已排队请求在执行点核对绑定 | crates/process-runtime/src/guardian/management_tests.rs::queued_request_checks_binding_after_takeover | Covered |
| 回复丢失去重与控制／输出隔离 | apps/ora-process-host/tests/service.rs::lost_reply_executes_once_and_stalled_output_does_not_block_control | Covered |

决策依据：[协议](../../../../decisions/node/process/protocol/0-independent-local-streams-and-state-recovery.md)。

## Snapshot Subscription Handoff Cannot Silently Lose Committed Changes

范围：未来订阅 ADR，当前仅轮询。
风险／前置：快照与事件之间并发更新；触发：订阅竞争、游标过期、提交后故障。
必须成立：不静默漏变化，历史缺失要求重同步，发布责任可恢复。
禁止结果：把旧快照当全部最新事实或从漏通知推导未执行。
验证义务与证据：生产订阅、游标回收与提交／发布窗口均 Missing，不以当前查询测试替代。
决策依据：协议根决策的未来 ADR 边界。

## Completion Records Remain Until Durable Responsibility Takeover

范围：未来 process 结果接管／退休 ADR，不等于 Node 业务 EventAck。
风险／前置：指定接管方尚未持久接受结果；触发：确认丢失、轮转和回收。
必须成立：持久接管后确认，确认幂等，普通轮转不丢弃未接管结果。
禁止结果：查询即确认，诊断观察者隐式承担责任，删除记录导致旧尝试复活。
验证义务与证据：process 接管／回收接口均 Missing；当前不回收记录，不声称协议已实现。
决策依据：协议根决策及持久化根决策的未来 ADR 边界。
