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
| 状态屏障 | Cloud `TestTaskConversationWaitsForHandoffThenStartsNextAgent`、`TestTaskConversationIdleExpiryWaitsForReleaseBeforeFreshRun`、直接领取屏障测试覆盖 ending/交付/释放及 done 但未删除 Workspace 的拒绝；删除与结算使用阶段 fixture。[#312 同租户成员受控模型分项](https://github.com/ora-space/circus/issues/312#issuecomment-6334568694)实测另一成员消息等待、不结束原 idle 会话，取消后未派发；真实跨发起者交接仍缺 | Partial |
| 实际保存与释放 | [#312 最新 main 真实 OpenCode 分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332636097)核对默认 idle 后的 Revision、交付 ref、实际文件、历史对象和 Workspace 物理释放；[#312 A 标准四运行](https://github.com/ora-space/circus/issues/312#issuecomment-6337641836)在 owner 认定的环境（固定 main 源码加 `--agent-configuration` 构建包，记录三个仓库完整 SHA、镜像 revision 与声明摘要）进一步验证不同 Workspace 的串行交接、两份文件和 Revision 链、四次释放；[#312 暂时交付失败分项](https://github.com/ora-space/circus/issues/312#issuecomment-6338204804)停掉 Git Server 后结束 Thread，内部阶段停在 `delivering`、未提前完成，发给另一 Agent 的消息保持 waiting（8 秒 Run 数不变），恢复后同一 Run 保存 Revision 并释放，Claude 才被派发且完成 Revision 的前驱正是该 Saved Revision。正式 release 产物及与 #407 永久失败相交的保存/释放仍缺 | Partial |

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
| 持久暂停与明确恢复 | Cloud `task_conversation_stop_db_test.go`、`TestTaskConversationFailedMessageHoldsFollowingMessagesUntilResolved`、`TestTaskConversationPausesAfterFailureUntilExplicitResume`、`TestTaskConversationUndeliveredRunRequiresExplicitDiscard` 验证数据库状态和恢复权限；[#312 最新 main Node SIGKILL 分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)实测失败后排队消息保持 waiting、同键显式 resume 只建一个新 Run。[#312 跨 Agent 失败屏障分项](https://github.com/ora-space/circus/issues/312#issuecomment-6338030083)在真实工具执行中 SIGKILL Node 后，发给另一 Agent 的消息保持 waiting（`blockedRunId` 指向失败运行，5 秒与 12 秒两次观测 Run 数不变），幂等显式恢复只创建一个由该 Agent 执行的新 Run；其持久线程里的 `Read` 工具调用记录了前一运行的 `ora-312b-result.txt` 与内容（口令也出现在前文，单凭复述不能证明，见[澄清](https://github.com/ora-space/circus/issues/312#issuecomment-6338095415)）。放弃未保存成果、交付失败（#407）与多实例交错竞争仍缺 | Partial |
| 重启与实际故障 | [#312 最新 main 分项验收](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)分别覆盖工具执行中 Controller/Cloud 重启、Node SIGKILL 与显式恢复、个人模型凭据撤销、暂时 Git Server 交付失败；各自核对同 Run 历史、队列或成果和物理释放。[最新 main 独立 90s 永久交付失败演练](https://github.com/ora-space/circus/issues/312#issuecomment-6333268333)却卡在 `releasing`、Workspace 未释放，证明该故障下的资源释放义务尚未成立；#407 承接原结果接收与续接保护。[候选组合实测](https://github.com/ora-space/circus/issues/407#issuecomment-6338723864)（cloud#142 `273f7f5a` + desktop#726 `13de5a35`）在放弃后 240 秒内仍不自发释放，但 Controller 重启/真实重连后原交付终态重新提交并被接受，Run 到 `done`、Workspace 删除；因此验收必须把重启/重连写成显式步骤。合并后仍须从新 main 复验，不能以候选通过替代。正式双 Agent 故障联合矩阵仍缺 | Partial |

### 决策依据

D3、D4，不变量 4、5。
