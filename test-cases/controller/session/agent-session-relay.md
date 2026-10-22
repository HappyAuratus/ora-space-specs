# Controller 中继 Agent 会话与 Revision 交付

已登记原交付的精确终态拒绝、单 execution unresolved 及成功 receipt 后解除由
[终态拒绝用例](refused-delivery-terminals.md)逐项登记。desktop 固定 head
`13de5a3567d0f2166a4ab27d0f5c7fa05626bc9d` 的单仓直接证据已覆盖局部隔离及 receipt/ACK；
[O4 真栈复测](https://github.com/ora-space/circus/issues/516#issuecomment-6345535566)已覆盖拒绝窗口及 Cloud 不可用导致 Controller↔Node 会话重建后的恢复，不要求 Controller 重启；Controller 重启后首次重放直接成功及真实 Node 新 incarnation 的缺口仍保留。
本文件的已有中继覆盖不自动证明这些新增义务，也不改变 Thread 批次冲突的既有规则。

依据[已实现的 Controller session 根决策](../../../decisions/controller/session/0-controller-relays-agent-sessions.md)。
本文件登记云端 Controller 派发会话与交付、中继 Thread 事件与命令、转交上传授权的承诺；Node 侧消息见
[Agent 运行的协议消息用例](../../node/protocol/agent-run-messages.md)，Cloud 侧契约见
[Agent 运行执行与 Thread 用例](../../cloud/controller-integration/agent-run-executions-and-thread.md)。证据来自 desktop
（核对基线 `a37ca7e0`）的 `apps/ora-controller/tests/workspaces/agents.rs` 与 `deliveries.rs`，使用假 Cloud gRPC 与
假 Node；跨仓库链路只有 cluster 的手工验收。

## Thread Events Reach Cloud In Order Before They Are Acknowledged

### 风险

Controller 先确认后提交，崩溃时丢失 Thread 记录；批次失败或回复丢失后重复或跳过记录；终态先于此前的 Thread
事件被结算；一个慢执行冻结其他执行的 Thread。

### 前置状态

Node 上有已登记的 Agent 会话执行，正在发出非终态 Thread 事件。

### 触发

Node 连续发送 Thread 事件和终态事件；Cloud 批次接管变慢、`UNAVAILABLE`、`CONFLICT` 或回复丢失；Controller 重启。

### 必须成立

- 每个执行至多一个在途批次（不超过 64 个事件、100 ms，或编码载荷约 1 MiB），序号按顺序到达 Cloud。
- 批次提交成功后才确认其中每个序号；失败时不确认，冲突时关闭连接触发 Node 重放。
- 终态在该执行此前的 Thread 事件全部接管后才经 `TakeOverNodeEvent` 接管，再确认；状态查询的 `Completed` 不结算会话。
- 不同执行的中继互不阻塞。
- Controller 重启后按原执行身份接收重放，不启动新会话。

### 禁止结果

确认先于 Cloud 提交；Cloud 中出现重复或缺失的 Thread 记录；终态先于 Thread 结算；单个批次超过 gRPC 消息上限。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 提交先于确认，终态等 Thread | `session_events_commit_before_ack_and_terminal_waits_for_thread`：延迟的 Thread 写入期间不确认、不结算终态，查询结果也不结算 | Covered |
| 失败与回复丢失不重复 | `batch_failures_and_lost_replies_preserve_exact_events`：中断、冲突与已提交但回复丢失三种情况都恢复为精确的记录 | Covered |
| 批次大小 | `large_records_are_split_into_grpc_sized_ordered_batches`：接近上限的记录被拆成按序、符合 gRPC 默认上限的批次 | Covered |
| 执行间互不阻塞 | `slow_execution_does_not_block_another_thread` | Covered |
| 重启重放 | `restart_replays_unacknowledged_events_without_starting_a_new_session` | Covered |

### 决策依据

Controller session D2，不变量 2；Node 协议 D2、D3。

## Sessions And Deliveries Start Only With Current Permission And Capability

### 风险

Controller 把会话或交付派发给不具备能力的 Node，或凭历史登记在运行时已关闭后启动执行；命令乱序或在会话
登记前到达；上传授权落盘或在没有校验和时申请；Controller 关闭被当作交付失败。

### 前置状态

Cloud 放出了会话或交付工作项，或为某运行持久化了用户命令。

### 触发

领取循环处理工作项与命令；Node 请求上传授权；Controller 重启或连接重建；Cloud 拒绝授权。

### 必须成立

- 只有握手声明 `agent_session`（模型绑定的会话还要 `model_proxy`）或 `revision_delivery` 的 Node 才会被登记派发。
- 发送前重新取得当前运行许可；已登记但许可已关闭的执行不启动。
- 每个运行只投递队首命令，Node 受理（或以 `session_ended` 拒绝）后才登记投递并推进；身份不符的回复不算投递。
- 授权只在 Node 带校验和请求时申请，原样转交，不写日志；Cloud 拒绝授权不使交付失败。
- 交付终态只从 Node 的 `RevisionResult` 事件接管，Cloud 提交后才确认；Controller 停止不写交付失败。
- 未结束的会话与交付使 quiesce 报告 busy。

### 禁止结果

向缺能力的 Node 派发；无当前许可时启动；后续命令先于未投递的命令；授权 URL 或签名出现在日志中；Controller
重启把交付判为失败；仍有会话或交付时报告 idle。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能力门禁 | `session_registration_waits_for_a_capable_handshake`、`model_bound_session_requires_model_proxy_capability`、`delivery_registration_waits_for_a_capable_handshake` | Covered |
| 当前许可 | `registered_sessions_wait_for_current_runtime_permission`；`delivery_waits_for_its_permit_and_relays_checksum_bound_grants_unlogged` | Covered |
| 命令顺序与投递证据 | `commands_retry_in_order_and_closed_session_rejection_is_delivered`、`restart_redelivers_the_unconfirmed_command_before_later_turns`、`foreign_command_reply_is_not_delivery_evidence` | Covered |
| 授权绑定校验和且不落日志 | `delivery_waits_for_its_permit_and_relays_checksum_bound_grants_unlogged`、`reconnect_refreshes_grants_for_an_unfinished_delivery`、`refused_grants_never_fail_the_delivery` | Covered |
| 交付结算与重启 | `revision_result_is_acknowledged_only_after_cloud_commits`、`restart_resumes_a_registered_delivery_without_failing_it`；精确拒绝后的恢复另见[终态拒绝用例](refused-delivery-terminals.md#real-reconnect-replays-the-original-fact-and-only-its-success-receipt-clears-refusal)，不能由这两个旧测试推断已覆盖 | Covered |
| idle 证据 | `quiesce_counts_unfinished_agent_sessions`、`quiesce_counts_unfinished_deliveries` | Covered |
| 真实 Cloud 与 Node 链路 | cluster 的 Agent 会话与 Revision 验收为手工运行，未纳入 CI；[原作者记录](https://github.com/ora-space/circus/issues/407#issuecomment-6341728665)及[O4 复测](https://github.com/ora-space/circus/issues/516#issuecomment-6345535566)直接证明迟到恢复、真实拒绝隔离、Cloud 重启导致会话重建后接管及物理释放；O4 子义务为 Covered，广义链路仍有逐事件 ACK/outbox、Controller 重启首个成功及真实新 incarnation 缺口，详见终态拒绝用例 | Partial |

### 决策依据

Controller session D1、D3、D4、D5，不变量 1、3、4、5。
