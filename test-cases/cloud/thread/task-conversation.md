# 任务持续对话与跨运行上下文

依据[已批准的任务持续对话决策](../../../decisions/cloud/thread/20261018-task-conversation-across-runs.md)。
下列证据包括已合并代码的测试、固定 main 上的真实 OpenCode 分项，以及[正式发布包四运行](https://github.com/ora-space/circus/issues/312#issuecomment-6347169056)：Claude v0.4.0 + OpenCode 0.6.11，cluster `6eecd5c1f7ce52afd19376ca417e4198eb27c1ba`、cloud `c5bcd147bf3abbe87823f9a4e155cc14b6c30cf8`、desktop `d271fb3b94234ab0f5d7f5ae3dcf94c738f104af`，结果 `passed-api`。
该记录的 `task-conversation.json` SHA-256 为 `1cb2f9d250ff59c97c1d0896cf466a811c552f8a608bde95d1c2ef4db5c0771e`；Claude 来自 [v0.4.0 GitHub Release](https://github.com/ora-space/claude-code-agent/releases/tag/v0.4.0) 的 x86_64-linux 包（SHA-256 `2feb3eba6edfbd66c3b4eea64584b76bdb0bf9acaeed61325f4168f2e7d86682`），不是 marketplace 条目或本地候选。四个不同 Workspace 均完成并释放；API 通过不替代浏览器组合、真实截断和跨发起者验证。
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
| 多来源展示与刷新 | Cloud 对话、Thread、session-data 组件测试覆盖消息呈现；[#312 最新 main 正常历史](https://github.com/ora-space/circus/issues/312#issuecomment-6334326387)、[失败历史](https://github.com/ora-space/circus/issues/312#issuecomment-6334859661)与[取消历史](https://github.com/ora-space/circus/issues/312#issuecomment-6334750487)均经浏览器按 Run URL 重读、刷新并看到执行状态；长回复 UI 还覆盖表格/代码块、375px、键盘、危险链接及普通评论不触发 Run；[本地候选四运行](https://github.com/ora-space/circus/issues/312#issuecomment-6333843436)逐一重开四份历史。正式包下的引用、结论来源去重和跨服务记录一致性仍缺 | Partial |
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
| 空闲、释放与新执行 | Cloud `task_conversation_lifecycle_db_test.go` 两项真实 PostgreSQL 测试覆盖窗口内复用、空闲期满结束、未释放阻塞、新执行与既有正文；[#312 默认窗口真实模型分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332636097)经历默认 15 分钟窗口，以新的 Run/Workspace 续接，核对两次 Revision、Git 结果和容器/卷物理释放；[正式发布包四运行](https://github.com/ora-space/circus/issues/312#issuecomment-6347169056)验证存活 OpenCode 会话续聊、Claude 交接、切回和主动结束后新 Run，四个不同 Workspace 各自释放。默认窗口与正式包分属两批固定版本证据，不声称本轮正式包重复了 15 分钟等待 | Covered |
| 正文投影与预算 | Cloud `task_conversation_context_db_test.go` 验证 Node 正文投影、缺失标记、预算、稳定排序和收据重放；`task_conversation_budget_db_test.go::TestTaskConversationRetainsOversizeRecentTurn` 以真实 PG 验证单条 64 KiB、跨块聚合和 JSON 转义膨胀仍保留最近一轮、UTF-8 前缀及截断标记，并按实际渲染结果检查预算。Desktop `apps/ora-node/src/session/thread.rs` 验证正文提取与截断；[正式发布包四运行](https://github.com/ora-space/circus/issues/312#issuecomment-6347169056)核对 Claude 读取前一运行文件、复述仅在对话中出现的代码，切回 OpenCode 后恢复两份文件和代码。真实超预算截断与可见范围展示仍缺 | Partial |
| 真实连续协作 | cluster `scripts/task-conversation-acceptance.py` 定义四运行检查；[#312 默认 idle 单 Agent 验收](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)与[正式发布包四运行](https://github.com/ora-space/circus/issues/312#issuecomment-6347169056)分别覆盖默认窗口后续接和 OpenCode→Claude→OpenCode→新 Run，Revision 链与四次物理释放均核实。正式双 Agent 故障联合矩阵仍缺；独立永久失败释放样本不替代该组合 | Partial |

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
| 发起者隔离与就绪缺口 | Cloud `TestTaskConversationMemberCannotAppendOrEndAnotherInitiatorsRun`、`TestTaskConversationReadinessUsesTheCurrentUsersConfiguration` 及旧入口跨发起者拒绝测试；[#312 最新 main 凭据撤销分项](https://github.com/ora-space/circus/issues/312#issuecomment-6332993328)实测任务/旧 Thread 追加均 409 且无新消息或 Run；[#312 非成员隔离](https://github.com/ora-space/circus/issues/312#issuecomment-6334438380)四读两写均 403；[#312 同租户成员受控模型分项](https://github.com/ora-space/circus/issues/312#issuecomment-6334568694)证明可读原 Run 但旧追加/结束均 403，自身模型消息 waiting、取消后未建新 Run。[G3/G5/G9 直接验证](https://github.com/ora-space/circus/issues/553)在两个不同成员、两份独立真实供应商 Key 下完成各自 OpenCode 调用；B 可读 A，但旧追加/结束均 403；B 消息等待 A 合法结束并物理释放后才派发。两个 Run 的冻结 userId/connectionId 分别属于各自发起者；撤销 B 凭据后任务/旧追加均 409 且消息、Run、Thread、评论、interaction、activity 数不变，此时 A 凭据仍有效。两次 Run completed/done 且容器/卷释放 | Covered |

### 决策依据

D2，不变量 3。

## 证据边界

上表 Partial 的缺口由 [circus#531](https://github.com/ora-space/circus/issues/531) 承接：多来源显示、历史引用与组合一致性对应 G1；真实截断范围对应 G2；双 Agent 故障联合矩阵对应 G4。G5 已有上面的真实双发起者直接证据；数据库级/受控模型证据仍不替代其他未测组合。
[circus#532](https://github.com/ora-space/circus/issues/532) 已明确：永久失败的恢复是显式步骤；G3/G5/G9 是收尾门槛，G1/G2/G4/G6/G7/G8/G10 允许延期，由 #531 继续承接。上表仍有缺口的义务保持 Partial；本证据 PR 不修改 ADR 状态。

本批 [G3/G5/G9 直接验证](https://github.com/ora-space/circus/issues/553)固定 cluster `45130795863037b365176ec51af611d46eb7f3da`、cloud `54bedcc95a5595ed23d87e2b841b55abd3dee71b`、desktop `8e497d7382441babd19595bfc0c0f61a7511fc31`；隔离普通成员通过公开邀请流程建立，真实模型调用返回 HTTP 200/succeeded。记录、产物摘要、失败样本和清理结果在运行中独立保存，不将凭据或会话正文复制到规格仓库。
