# Agent 运行的协议消息

依据[已批准的 Node 协议后续决策](../../../decisions/node/protocol/20260928-streamed-thread-events-and-session-commands.md)。
消息与校验已在 desktop `crates/node-protocol` 合并（desktop#629），证据来自
`crates/node-protocol/tests/protocol/{plugin,agent_session,revision}.rs`。协议往返测试只证明线上形状，
持久化、重放与恢复由 Node/Controller 测试承担，因此相应义务为 `Partial` 或 `Missing`。

## Thread Events Share One Gap Free Sequence With The Terminal Event

### 风险

Thread 事件与终态事件序号冲突或出现空洞，Controller 无法按序接管；Node 在持久化前发送事件，崩溃后丢段。

### 前置状态

会话执行正在运行。

### 触发

会话产生定型记录、超大记录、会话结束；Node 在事件确认前断线重连。

### 必须成立

- Thread 事件是带序号的普通事件，与终态事件共用执行内序号，终态之后没有事件。
- 记录是 JSON 对象，编码后不超过 256 KiB；超大记录以截断版本发送并标注 `truncated`。
- 事件在账本持久化后才发送，未确认前保留，重连后按原身份重放；同一执行在途不超过 256 个。

### 禁止结果

记录不是对象或超限仍被编码；终态后仍有事件；未持久化即发送。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 线上形状与记录上限 | `agent_session.rs` `thread_events_round_trip_and_bound_the_record`：非对象记录解码失败，超过 `MAX_THREAD_RECORD_BYTES` 被双向拒绝 | Partial |
| 终态在结果族中可区分 | `agent_session.rs` `session_end_round_trips_as_event_and_completed_status`：终态事件与 Completed 查询都解码为会话结果族，自由文本 `detail` 被拒绝 | Partial |
| 持久化、窗口与重放 | `ora-node-db` 的 `exact_ack_and_sequence_survive_reopen`、`failed_event_rolls_back_all_session_effects` 证明落盘、精确 ACK 和重开序号；生产发送、256 窗口和重连接线尚未实现 | Partial |

### 决策依据

协议后续决策 D2、D3，不变量 2、3。

## Session Commands Execute At Most Once In Acceptance Order

### 风险

命令重复投递导致用户轮次执行两次；会话结束后的命令被执行。

### 前置状态

会话执行正在运行或已结束。

### 触发

`SubmitUserTurn`、`EndSession` 首次与重复发送；同一 `command_id` 不同内容；会话已终态时发送。

### 必须成立

- 命令持久化后回复受理；回复无序号、无需确认。
- 同一 `command_id` 重发只回复受理；不同内容为协议错误；已终态时回复 `session_ended` 拒绝。
- 用户轮次按受理顺序执行；`EndSession` 丢弃排队轮次。

### 禁止结果

同一命令执行两次；终态后执行命令。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 命令与回复形状 | `agent_session.rs` `session_commands_and_replies_round_trip`：四种消息往返，`command_id` 必填且非空 | Partial |
| 去重、顺序与终态拒绝 | `ora-node-db` 的 `commands_are_ordered_deduplicated_and_discarded_at_end` 证明持久队列规则；实际协议受理回复、唤醒与拒绝回复尚未接线 | Partial |

### 决策依据

协议后续决策 D4，不变量 4。

## Upload Grants Stay In Memory And Deliveries Stay In Their Namespace

### 风险

预签名 URL 进入账本或日志；交付写到分支或其他对象键。

### 前置状态

交付执行正在运行。

### 触发

收到 `UploadGrant`；授权过期或 Node 重启后需要上传；交付输入使用 `refs/ora/revisions/` 之外的 ref 或非规范对象键。

### 必须成立

- `UploadGrant`/`UploadGrantNeeded` 无序号、无确认，不写账本与日志；`Debug` 不输出 URL。
- 没有有效授权时发送 `UploadGrantNeeded` 并等待，执行保持 `Running`。
- 交付 ref 必须在 `refs/ora/revisions/` 下，两个对象键规范且不同；`unchanged` 与 `delivered` 与 commit 是否变化一致。

### 禁止结果

授权持久化或出现在日志；交付移动分支。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 授权脱敏与形状 | `revision.rs` `upload_grants_round_trip_without_exposing_the_url` | Partial |
| ref 与对象键校验 | `revision.rs` `deliver_revision_rejects_refs_keys_and_commits_outside_the_contract`、`revision_results_round_trip_and_keep_commits_consistent` | Covered |
| 授权不落盘、过期重请求 | Node 测试：账本与日志中无授权；过期后发送 `UploadGrantNeeded` | Missing |

### 决策依据

协议后续决策 D1、D5，不变量 5、6。
