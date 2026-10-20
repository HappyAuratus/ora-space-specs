# 任务级串行 Agent 交接

依据[已批准的任务串行交接决策](../../../decisions/cloud/issue-run/20261018-serialize-task-agent-handoffs.md)。
此处区分真实 PostgreSQL 的调度证据、阶段 fixture 与真实服务/Agent 验收，不把任一层替代其他层。
不维护升级前存量并发运行或旧失败记录补确认的兼容用例；运行后的失败与暂停仍需验证。

## Every Entry Point Respects The Handoff Barrier

### 风险

旧追加接口持续激活即将交接的会话，或多窗口/旧入口绕过队列启动并行执行。

### 前置状态与触发

任务有活跃 Agent；存在发给另一 Agent 或另一发起者的 waiting 消息，或有持久暂停。
从手动建 Run、评论 targets、新任务对话及旧 `POST .../runs/:rid/thread/messages` 继续提交。

### 必须成立

- 旧 Thread 追加接口以 `409` 拒绝交接屏障和暂停，不写条目或命令，不延长旧会话活跃时间。
- 交接前已接受的同 Agent 消息先处理；队列工作者只检查该消息之前的屏障，后来的切换不能反过来拒绝它。
- 任务内执行占用在数据库确定，所有启动入口和不同 Cloud 实例均不能绕过。

### 禁止结果

靠旧接口连续追加无限推迟交接；后来的切换丢弃已接受消息；前端禁用按钮而服务端仍允许并行。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 旧追加拒绝且无副作用 | Cloud `internal/core/task_conversation_append_db_test.go::TestTaskConversationLegacyAppendRejectsHandoffAndPause`：跨 Agent、跨发起者与暂停下反复调用生产 Public 入口，返回 409 Fault，条目/命令不增加，读模型禁止追加；正常到达 idle 后交接进入 ending | Covered |
| 已接收前缀不被误拒绝 | 同文件 `TestTaskConversationQueuedPrefixDrainsBeforeHandoff`：同 Agent 消息先接受，之后接受切换，调度仍先把前缀提交到原运行 | Covered |
| 启动入口与竞争 | Cloud `agent_run_serial_dispatch_db_test.go` 的 `TestTaskAgentClaimWaitsForOlderRunAndConfirmedRelease`、`TestTaskAgentClaimPausesAfterFailure`，以及 `task_conversation_claim_db_test.go::TestTaskAgentClaimCannotBypassUnresolvedTaskMessage` 覆盖 failed/waiting 消息下生产 Public 创建拒绝、派发和扫描重查，取消后恢复；`::TestTaskAgentClaimPreservesMessageAndDirectRunOrder` 验证较晚消息不阻塞较早 Run，较晚直接创建不能占住更早消息所需的 pending 名额。尚缺手动、评论、旧追加、新对话全部入口在多实例中交错竞争的完整直接证据 | Partial |

### 决策依据

D1，不变量 1、2。

## Saving And Confirmed Release Precede The Next Agent

### 风险

Thread ended 或运行 done 就启动下一位，实际上最新成果尚未保存、旧沙盒尚未释放。

### 前置状态与触发

交接已排队，原运行依次经过活跃轮次、idle、ending、delivering、releasing 和 done。

### 必须成立

- 仅选择目标不启动或停止；发送后等待当前轮与交接前已接收消息处理，到安全点正常结束。
- 成果已登记或确认无改动、资源释放已确认后，新运行才有派发资格；不继承旧 Thread、Workspace 或凭据。
- 浏览器断线、超时和 done 字段本身都不能代替释放证据。

### 禁止结果

立即取消正在做的工作；在保存/释放中并行启动；恢复早于最后一次已保存 Revision；永久保留运行沙盒以续聊。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 状态屏障 | Cloud `TestTaskConversationWaitsForHandoffThenStartsNextAgent`、`TestTaskConversationIdleExpiryWaitsForReleaseBeforeFreshRun`、直接领取屏障测试，覆盖 ending/交付/释放及 done 但未删除 Workspace 的拒绝；删除与结算使用阶段 fixture | Partial |
| 实际保存与释放 | cluster 候选四运行验收检查不同 Workspace、Revision、实际文件与容器/卷释放；尚未运行真实 OpenCode 与第二 Agent | Missing |

### 决策依据

D2，不变量 3、4、6。

## Failure And Manual Stop Persist A Recoverable Pause

### 风险

停止或失败后队列悄悄继续，或者恢复时跳过未保存成果、错误借用前一发起者配置。

### 前置状态与触发

任务已有排队消息，当前执行停止、失败、交付失败或释放未确认；用户取消消息、恢复或显式放弃未保存成果。

### 必须成立与禁止结果

暂停与待执行消息持久化；恢复重新校验权限、成果与资源状态，不能重放已执行/取消输入，不能以确认后面的
排队运行跳过前面的失败。放弃未保存成果必须独立、明确；服务重启不丢队列或自动消除暂停。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 持久暂停与明确恢复 | Cloud `task_conversation_stop_db_test.go`、`TestTaskConversationFailedMessageHoldsFollowingMessagesUntilResolved`、`TestTaskConversationPausesAfterFailureUntilExplicitResume`、`TestTaskConversationUndeliveredRunRequiresExplicitDiscard` 验证数据库状态和恢复权限；旧入口也不能绕过暂停 | Partial |
| 重启与实际故障 | 仍需真实 Cloud/Controller/Node 重启、模型撤权和保存故障演练，证明运行、队列与成果恢复一致 | Missing |

### 决策依据

D3、D4，不变量 4、5。
