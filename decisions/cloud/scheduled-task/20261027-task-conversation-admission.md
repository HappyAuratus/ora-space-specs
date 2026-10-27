---
status: proposed
---

# Cloud Scheduled Task 接纳受任务对话的准入与暂停约束，被拒时记 `task_paused` 或 `task_handoff_pending`

本文修改 [Cloud Scheduled Task 根决策](0-cloud-owned-scheduled-prompt-runs.md) D5 的资格校验表：每次接纳
在创建 agent IssueRun 之前，按 [同一任务 Agent 执行串行决策](../issue-run/20261018-serialize-task-agent-handoffs.md)
对绑定 issue 的任务对话做与直接输入相同的准入判断。对话已暂停、或有尚未解决的失败消息时，occurrence
记 `failed / task_paused`；有先前接收、仍在等待的消息时，记 `failed / task_handoff_pending`。两种情况都不
创建 IssueRun，不重试，也不排队等待对话恢复。

本 ADR 处于 `proposed`：失败原因码与 waiting 消息的处理方式待评审拍板，核心测试用例在 `approved` 后补充。
涉及 Cloud 的定时任务接纳事务与 `scheduled_task_occurrences.failure_reason` 的取值集合；任务对话、直接
输入入口、manual run 与 Workflow 路径不变。

## 问题

串行决策要求所有 Agent 输入入口「受同一准入、交接与暂停约束，多个窗口或 Cloud 实例不能绕过」。定时任务
接纳在根决策 D3 的事务里直接创建 agent IssueRun，没有经过这项准入，导致：

- 对话已暂停或有未解决的失败消息时，occurrence 记为接纳，新 IssueRun 却因任务级领取屏障一直停在
  `queued`。根决策 D4 的截止取消只收敛已启动的会话，排队中的运行收不到 `cancelled`；此后每个到期点
  又因该运行未达终态而记 `skipped_busy`，创建者看不到真正原因。
- 有先前接收的 waiting 消息时，定时运行先被领取执行，越过了任务内的接收顺序。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [Cloud Scheduled Task 根决策](0-cloud-owned-scheduled-prompt-runs.md) D3：资格校验、occurrence 写入与 agent IssueRun 创建在同一事务完成，复用既有编排，不另建运行入口 | 继承，不变：任务对话准入是同一事务里的一项资格校验 |
| 同上 D5：保存与每次接纳都校验资格；接纳失败映射到失败原因集，issue / Project / Space / Agent 不可用记 `target_unavailable`，模型配置不可用记 `model_unavailable`；失败不重试、不回退 | **修改**：资格表增加「任务对话接受新的 Agent 输入」一行，只在每次接纳时校验；失败原因集增加 `task_paused` 与 `task_handoff_pending`（D1、D2） |
| 同上 D6：同一任务上一次接纳的运行未达终态时记 `skipped_busy` | 继承，不变：任务对话准入先于 `skipped_busy` 判断，被对话拒绝的到期点不产生运行，也不会让后续到期点误判为忙 |
| [desktop Scheduled Task 根决策](../../desktop/core/scheduled-task/0-local-and-cloud-execution-ownership.md) D7：失败**至少**区分 `target_unavailable`、`model_unavailable`、`approval_policy_invalid` 等原因 | 继承：新增的两个原因是 Cloud 侧对「至少」集合的扩展，不改变 Desktop 的原因集 |
| [同一任务 Agent 执行串行决策](../issue-run/20261018-serialize-task-agent-handoffs.md) D1/D3：输入按任务内接收顺序处理，任务暂停或存在先前的 waiting 消息时直接输入以 `409` 拒绝；失败形成持久暂停点，恢复需要显式授权 | 继承并落到定时入口：定时接纳采用同一判断与同一错误码语义，不新增恢复或排队机制 |

## 决策

### D1：任务对话准入是每次接纳的资格校验项

| 校验 | 保存时 | 每次接纳 |
|---|---|---|
| 绑定 issue 的任务对话接受新的 Agent 输入（未暂停、无未解决的失败消息、无先前的 waiting 消息） | —（保存不读取对话状态，对话状态随时可变） | ✓ |

- 判断与直接输入入口共用同一规则，不另写一套条件：直接输入以 `409` 拒绝的状态，定时接纳同样拒绝。
- 校验位于根决策 D5 其余资格项之后、D6 的 `skipped_busy` 判断之前；被拒的 occurrence 一次写成失败，
  不创建 IssueRun，不重试，之后也不会改记为 `missed`。
- 拒绝不改变任务对话本身：不写对话消息，不解除暂停，不推进或改写队列；用户恢复对话后，下一次到期
  按正常流程接纳。被拒的到期点不补跑。

### D2：失败原因沿用直接输入的错误码

| 对话状态 | occurrence 的失败原因 |
|---|---|
| 已暂停，或有尚未解决的失败消息 | `task_paused` |
| 有先前接收、仍在等待的消息 | `task_handoff_pending` |

选择依据：两种状态要求创建者到任务对话里处理（恢复暂停、处理失败消息或等待先前消息执行），而
`target_unavailable` 指向 issue、Project、Space 或 Agent 配置，会把创建者引向错误的修复位置。沿用直接
输入的错误码，使同一对话状态在手动发送和定时触发两条路径上表达一致。

waiting 消息选择直接拒绝而不是让定时运行排到消息之后：排队会让一次到期跨越不定时长，与根决策 D4 的
60 分钟截止和 D2「迟到不补跑」相冲突，也需要为定时输入引入新的排队状态。

## 不变量

1. 任何到期点都不能在任务对话拒绝 Agent 输入时创建 agent IssueRun。
2. 定时接纳与直接输入对同一对话状态给出相同的准入结论。
3. 被对话准入拒绝的到期点恰有一条 `failed` occurrence，原因是 `task_paused` 或 `task_handoff_pending`。
4. 拒绝不改写任务对话的消息、暂停点或队列顺序。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 记 `target_unavailable`，不扩展原因集 | 改动最小，前端无需新文案；但原因无法区分「目标不可用」与「对话被阻塞」，创建者会去检查 Agent 或 issue 配置，违背 D5「失败明确到原因」 |
| 记 `skipped_busy` | 不新增失败原因；但 `skipped_busy` 的语义是同一任务上一次运行尚未结束（根决策 D6），与对话暂停无关，历史记录会误导 |
| waiting 消息时让定时运行排到消息之后执行 | 不丢这次触发；但等待时长不定，与 60 分钟截止和不补跑规则冲突，并需要新的排队状态 |
| 保持现状，依赖领取屏障拦截 | 屏障只阻止执行，不结束运行：occurrence 显示为接纳，运行永久排队，后续到期点全部 `skipped_busy`，waiting 消息的顺序仍会被越过 |

## 本决策未解决的问题

- 失败原因码（D2）与 waiting 消息的处理方式（D2 末段）待评审拍板；拍板后按结论修订本文并标记
  `approved`，再补充核心测试用例。
