# 同一 Issue 的运行续接最近一次 Revision

迟到原 Revision 不得越过同 Issue 后创建 Run 的新增登记限制见
[迟到交付事实与续作顺序](late-delivery-fact-takeover.md)。其顺序守卫及 residual 路径义务当前为 `Missing`，
下列已有最近候选选取证据不自动覆盖这一限制。

依据[已实现的 IssueRun 续接决策](../../../decisions/cloud/issue-run/20261010-resume-run-from-latest-issue-revision.md)。
续接决策的 bundle 候选、沿用 bundle 与一跳规则（D1、D5）已被 [Cloud Revision Git 交付决策](../../../decisions/cloud/revision/20261017-git-native-delivery-refs-and-resume.md) D7 修改：
候选是有交付 ref 且有新提交的最近 Revision，选取与恢复失败的义务见 [Revision 经 Git Server 交付与续作](../revision/git-native-delivery-refs-and-resume.md)；
本文件继续登记运行入口处的选取固定与对外可见性。
证据来自 `ora-space/cloud` 的 `integration/agent_run_resume_test.go`（真实 gRPC 与 PostgreSQL，以 simulator 扮演
Controller/Node）、迁移测试与前端测试。Node 的恢复步骤见
[Node 经 Git Server 交付与恢复前序 Revision](../../node/revision/git-delivery-push-and-restore.md)。

跨仓库真实链路证据来自 cluster 的 `task revision:resume:acceptance`（见
[Revision 交付与续接验收](https://github.com/ora-space/cluster/blob/main/docs/testing/revisions.md)）：同一 Issue
连续运行三次，第二次看到第一次的提交，第三次的交付 ref 包含前两次的提交，空运行也有自己的交付 ref。验收失败不能单独
定位到下列某条义务，因此只作为补充引用，不据此标记 `Covered`。

## New Runs Resume The Issue's Latest Revision Fixed At Session Start

### 风险

续接了错误的 Revision（其他 Issue、其他仓库、较旧的成果），或前序在会话启动后被改变，Node 恢复的不是 Cloud
登记的那一个。

### 前置状态

同一 Issue 上有已登记的 Revision，其中一些停在基础提交，一些有新提交。

### 触发

新运行进入 `starting`，放出会话工作项。

### 必须成立

- 选取同一 Issue 其他运行中按登记时间最新、符合候选条件的 Revision（候选条件见 [Revision 经 Git Server 交付与续作](../revision/git-native-delivery-refs-and-resume.md)），写入会话输入的 `priorRevision` 与
  `issue_runs.resume_revision_id`；首条提问的 Context 带 `resume`。
- 没有候选时会话输入不带 `priorRevision`，Context 不带 `resume`。
- 候选的 Project 或仓库地址与本次不同时从头开始，Context 记 `{skippedRevisionId, reason: project_changed}`。
- 交付输入的前序从会话输入复制，只带 `{revisionId, finalCommit}`。

### 禁止结果

续接其他 Issue 或其他仓库的 Revision；会话启动后前序变化。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 续接最近一个并固定 | `TestFollowUpRunResumesTheIssueLatestRevision`；`TestResumedRunWithoutNewCommitsHasItsOwnDeliveryRef`：第三次运行续接第二次，第二次有自己的交付 ref，不需要一跳 | Covered |
| 没有历史时不变 | `TestRunWithoutHistoryStartsFresh` | Covered |
| 换了项目时从头开始 | `TestRunOfAMovedIssueStartsFresh`。缺口：没有“同一项目只改了仓库地址”的单独用例 | Partial |

### 决策依据

续接决策 D1、D4，不变量 1、2、5；Git 交付决策 D7。

## Resume Is Visible To The Agent And The Public API

### 风险

用户与 Agent 不知道本次运行接着哪一次成果工作，或公开 API 泄漏内部标记。

### 前置状态

运行续接了前序 Revision。

### 触发

读取运行；查看 Thread 首条记录。

### 必须成立

- 运行带 `resumeRevisionId`，`revision` 投影带 `priorRevisionId`；不可续接标记不公开。
- Issue 页显示续接来源，续接后没有新提交时显示沿用了续接的成果。

### 禁止结果

公开 `resume_refused_*`；续接运行看不出来源。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 公开投影 | `TestPublicRunProjectsResume` | Covered |
| 前端展示 | 前端 `run-resume.test.tsx`；`run-delivery.test.tsx` 中续接后没有新提交的用例 | Covered |

### 决策依据

续接决策 D4。
