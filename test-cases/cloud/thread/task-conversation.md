# 任务持续对话与跨运行上下文

依据[已批准的任务持续对话决策](../../../decisions/cloud/thread/20261018-task-conversation-across-runs.md)。
下列证据引用 Cloud、Desktop 与 cluster 的候选实现和测试；它们不等于产品已合并或真实 Agent 联合验收完成。
跨运行成果恢复继续受[Revision 续接用例](../issue-run/resume-from-latest-revision.md)约束，不能以对话存在替代成果证据。

## Task Messages Have Stable Order And One Execution Intent

### 风险

多窗口重试创建重复执行，或首条提示、用户回声与结论在统一对话中重复显示。

### 前置状态与触发

任务关联项目和已安装 Agent；多个窗口以相同或不同幂等键发送消息、取消等待消息并刷新读取历史。

### 必须成立

- 同键同内容返回同一消息，任务内序号唯一且有序；消息与执行意图持久化后才允许派发。
- 每条已接收消息有可查询的等待、提交、失败或取消状态；仅评论不触发 Agent。
- 同一来源的首条提示、回声、逐轮回复与运行结论不重复显示；历史引用不暗中回退 Revision。

### 禁止结果

同键重试产生多个执行；刷新丢消息；评论正文里的普通 `@名称` 隐式改变目标；相同来源重复成为多条用户消息。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 并发顺序与幂等 | Cloud `internal/core/task_conversation_db_test.go::TestTaskConversationConcurrentWindowsPreserveOrderAndIdempotency`、`::TestTaskConversationMessageOrderIdempotencyAndCancellation`，真实 PostgreSQL 与生产 Public 入口 | Covered |
| 多来源展示与刷新 | Cloud 对话、Thread、session-data 组件测试覆盖消息呈现；仍需真实浏览器验证历史重开、引用、结论来源去重与跨服务记录的一致性 | Partial |
| 工具轮次关联 | Cloud `integration/agent_run_thread_read_test.go::TestThreadReadReturnsNodeTurnIdentityOverHTTP` 经真实 HTTP/PG 验证排队用户消息之间 Node 条目的真实 turnId；`frontend/src/features/issues/thread/thread-entries.test.ts` 覆盖同调用 ID 跨轮隔离、稀疏更新及旧公开响应缺少 Node turnId 时的用户边界。此项仅证明公开读取与显示归并，不替代真实 Agent 输出验证 | Covered |

### 决策依据

D1、D4，不变量 2、4。

## Ended Threads Continue Through Fresh Runs And Separately Restored Results

### 风险

为了续聊永久保留沙盒，或将聊天记录误认为代码成果已经保存；新运行缺少对话上下文却声称完整恢复。

### 前置状态与触发

同一任务的会话在空闲窗口内、窗口期满或已结束；用户继续发给同一 Agent 或切换另一 Agent。

### 必须成立

- 存活、同 Agent、同发起者且没有屏障的会话可在原空闲窗口内追加；结束后 Thread 保持 ended。
- 新运行使用新的 Run/Thread/Workspace，恢复最新合法 Revision，并单独冻结有来源的用户输入和 Agent 对外正文。
- 最近 20 轮、64 KiB 的预算与缺失范围可见；思考、工具原始数据和凭据不默认进入交接上下文。
- 历史读取不启动沙盒；存活会话的空闲结束与交付后释放机制保持有效。

### 禁止结果

重开 ended Thread；跨 Run 复用一次性 Workspace；静默退回旧成果；把缺失或截断历史描述成完整记忆。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 空闲、释放与新执行 | Cloud `task_conversation_lifecycle_db_test.go` 两项真实 PostgreSQL 测试覆盖窗口内复用、空闲期满结束、未释放阻塞、新执行与既有正文；物理删除在该测试中使用阶段证据 fixture，不是 Docker 联合证据 | Partial |
| 正文投影与预算 | Cloud `task_conversation_context_db_test.go` 验证 Node 正文投影、缺失标记、预算、稳定排序和收据重放；`task_conversation_budget_db_test.go::TestTaskConversationRetainsOversizeRecentTurn` 以真实 PG 验证单条 64 KiB、跨块聚合和 JSON 转义膨胀仍保留最近一轮、UTF-8 前缀及截断标记，并按实际渲染结果检查预算。Desktop `apps/ora-node/src/session/thread.rs` 的测试验证正文提取和 UTF-8 截断。CloudStore 转发与真实跨 Agent 上下文恢复尚待贯通 | Partial |
| 真实连续协作 | cluster `scripts/task-conversation-acceptance.py` 定义 OpenCode、第二 Agent、切回与主动结束后新运行续聊的四次运行检查；尚无实际模型运行结果 | Missing |

### 决策依据

D2、D3，不变量 1、5、6。

## Read Access Does Not Grant Another Initiator's Model Authority

### 风险

读取任务历史的人能向别人的模型会话追加，或通过切换目标强行结束其运行并继承凭据。

### 前置状态与触发

同任务中存在另一成员发起的会话；成员查看、发送、切换或在授权撤销后重试。

### 必须成立与禁止结果

可读权限不扩展模型调用权限；新运行使用自身发起者配置，接收与派发均重新校验。无结束权限时等待合法结束，
不借用旧模型绑定，不因目标或配置失效而自动改发其他 Agent。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 发起者隔离与就绪缺口 | Cloud `TestTaskConversationMemberCannotAppendOrEndAnotherInitiatorsRun`、`TestTaskConversationReadinessUsesTheCurrentUsersConfiguration`，及旧入口跨发起者交接拒绝测试；实际授权撤销与浏览器/模型进程联合演练尚缺 | Partial |

### 决策依据

D2，不变量 3。
