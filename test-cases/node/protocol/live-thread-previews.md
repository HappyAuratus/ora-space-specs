# 实时 Thread 预览旁路

依据[已批准的实时预览决策](../../../decisions/node/protocol/20261018-live-thread-previews.md)。
可靠记录、命令与终态继续受[既有 Agent 协议用例](agent-run-messages.md)约束。固定 main 上的真实 OpenCode
已产生 Node → Controller → Cloud → 浏览器预览；含 Claude 候选的本地四运行只验证了至少一处实时正文与工具过程，正式包、慢消费者与故障组合仍需直接证据。

## Stable Identity And Final Records Defeat Reordered Previews

### 风险

重放或乱序片段造成重复正文、错误工具配对，或者最终记录被迟到预览复活。

### 前置状态与触发

双方协商预览能力，执行中有正文、思考及工具块；重复、低版本、跨执行更新与定型记录交错到达。

### 必须成立

- 块在执行、轮次和历史位置上有稳定身份，版本递增；工具按调用身份合并，不能仅靠名称配对。
- 已定型记录覆盖相同位置的预览；未知/缺失身份不能伪造正文或成功状态。
- 未协商的对端只走原持久路径，界面明确能力差异。

### 禁止结果

重复追加相同正文；跨轮次或跨执行合并；迟到预览覆盖定型记录；旧对端被迫接收未协商消息。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 线上协商、身份和边界 | Desktop `crates/node-protocol/tests/protocol/thread_preview.rs` 覆盖可选协商、版本及位置约束；[#312 最新 main 真实 OpenCode 分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)经 SSE 观察到最终正文的严格前缀和工具终态配对，证明本组合的转发已接通。旧 Node 降级与正式第二 Agent 的独立预览仍缺 | Partial |
| 定型优先和乱序 | Cloud `internal/core/thread_preview_test.go::TestThreadPreviewAuthorityAndFinalRecordDominance`、替换/隔离测试与 `frontend/src/features/thread-preview/preview.test.tsx` 覆盖采用/覆盖规则；[#312 生产者 RPC/SSE 注入](https://github.com/ora-space/circus/issues/312#issuecomment-6320732885)验证低版本、重复、冲突、定型覆盖和重放，但不代表 Node WebSocket 逐跳乱序或最新 main 的全部故障组合 | Partial |

### 决策依据

D1、D3，不变量 1、3。

## Preview Backpressure Never Enters Durable Acknowledgements

### 风险

逐片段写数据库，或者慢浏览器导致 Agent 阻塞、持久记录丢失或无界内存增长。

### 前置状态与触发

Agent 持续产生预览，连接中继/消费者变慢或断线，可靠事件仍在写入和等待确认。

### 必须成立

每层预览缓冲有界、可替换并从完整快照恢复；旁路不占持久序号或 Ack。慢消费者与预览连接失败不终止
Agent，也不改变 Run/Thread/Revision 的权威状态；原持久写失败仍按原有语义收尾。

### 禁止结果

预览产生收据或 Thread 行；慢发布阻止可靠记录与心跳；连接退出留下工作者或订阅；内存随输出无限增长。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| Node 真实发送与有界缓冲 | Desktop Node 单元与 `apps/ora-node/tests/standalone/agent_sessions/previews.rs::negotiated_preview_streams_before_the_agent_turn_finishes` 使用真实 IPC 和 Echo Agent；[#312 最新 main 真实 OpenCode 分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)又在轮次定型前观测到正文严格前缀和 5 个工具配对。有界缓冲承诺仍以定向测试为证据，真实慢消费者尚缺 | Partial |
| 慢发布与取消 | Desktop `apps/ora-controller/src/session/previews.rs::slow_publisher_coalesces_and_connection_drop_cancels_workers` 验证替换和连接归属；`::completed_executions_do_not_exhaust_preview_capacity` 验证同一中继连续 128 次完成后仍可发布；`::terminal_takeover_cancels_only_its_in_flight_publisher` 验证终态取消阻塞发布、重复回收及其他活跃执行隔离。Cloud 的并发 hub、订阅容量和取消测试验证有界通知。[#312 真实模型分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)在 277 次新 SSE 连接后仍完成交付；实际慢消费者与迟到预览组合仍缺 | Partial |

### 决策依据

D2，不变量 1、2、4。

## Authorized Reconnect Restores Durable History Before Live Blocks

### 风险

刷新丢失已保存回复，旧执行片段覆盖新执行，权限撤销后继续泄露预览，或关闭页面误停 Agent。

### 前置状态与触发

浏览器重连、生产者断线或缓存丢失；执行代次变化，成员权限被撤销，或用户离开页面。

### 必须成立

先恢复持久 Thread，再恢复未定型块；无快照时显式降级，不能称为完整输出。订阅与接收验证当前权限、
租约及执行绑定；撤权后清空该订阅预览并停止后续传输。浏览器断开只释放订阅，不发送终止命令。

### 禁止结果

重连重复正文；未定型内容误称完整；旧执行/外租户片段进入当前视图；撤权后继续传输；离开页面取消执行。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 服务器权限与执行围栏 | Cloud `TestThreadPreviewAuthorityAndFinalRecordDominance` 覆盖旧租约/外执行拒绝；`integration/thread_preview_test.go::TestThreadPreviewSSEStopsAndClearsAfterMembershipRevocation`、`::TestThreadPreviewRPCDoesNotCreateReceipt` 覆盖撤权清空与无回执；`frontend/src/features/thread-preview/preview.test.tsx` 覆盖恢复与取消订阅；[#312 最新 main 浏览器分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)实测执行中离线重连后消息/回复不丢。真实成员撤权与多服务组合仍缺 | Partial |
| 实际供应商流式与工具过程 | [#312 最新 main 真实 OpenCode 分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)在 277 次新 SSE 连接中取得 103 次可用快照，严格正文前缀、5 个进行中工具与终态配对均通过；[#497 本地候选四运行](https://github.com/ora-space/circus/issues/497#issuecomment-6333754438)进一步验证至少一处正文前缀和工具状态配对，但未把这项全局断言单独归因于 Claude。正式第二 Agent 包的预览及慢消费者故障仍缺 | Partial |

### 决策依据

D3，不变量 3、5。
