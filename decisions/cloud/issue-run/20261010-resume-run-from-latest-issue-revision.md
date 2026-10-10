---
status: proposed
---

# 同一 Issue 的新运行先续接最近一次 Revision，再启动 Agent；恢复不了就失败，远端改写导致的永久失败只跳过一次

Issue 上再次 @Agent 时，新运行仍然得到一个全新的 isolated Workspace（根决策 D2），但在 Agent 启动前，
Node 先把该 Issue 最近一个带 bundle 的 Revision 拉进 checkout，并把本地默认分支指向它的最终提交；Agent
因此接着上一次的成果工作。会话结束后的交付不变，仍相对本次 clone 的基础提交打 bundle，所以每个 Revision
自然包含此前尚未进入远端的全部提交，续接只需要最近一个。恢复失败时运行失败、不启动 Agent；只有“远端历史被
改写、基础提交再也取不到”这种永久失败会把该 Revision 标为不可续接，下一次运行从头开始。

本 ADR 处于 `proposed`，继承 [IssueRun 根决策](0-agent-run-in-disposable-isolated-workspace.md)，填补其“在已有
Revision 上续接运行”的未解决项。契约形状见 [controller-integration 后续决策](../controller-integration/20261010-revision-restore-contract.md)，
Node 的恢复步骤见 [node/revision 后续决策](../../node/revision/20261010-restore-prior-revision-before-session.md)。
涉及 Cloud 的会话启动、会话结束结算、Revision 登记、迁移与公开 API。兼容性边界：已有 `revisions` 行与运行
的含义不变，本文只增加列；没有前序 Revision 的运行行为与现在完全相同。本文不承担同一 Issue 并发运行的协调、
bundle 过期清理、用户选择“从头开始”的交互，以及把成果推送到用户远端。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D2：每次运行一个 isolated Workspace，`requestedRef` 为 Project 默认分支 | 继承；Workspace 仍从默认分支 clone，基础提交仍是 clone 结果 |
| 根 D3：`starting` 阶段放出会话工作项 | 继承；会话启动参数在同一事务里多带前序 Revision（D1） |
| 根 D4：`status` 由会话结束原因推导，`agent_failed` → `failed` | 继承；恢复失败用 `agent_failed` 与新的 `detail`，不新增结束原因（D3） |
| 根 D5：Revision 登记前不释放 | 继承，不变 |
| Revision D4：`base_commit = final_commit` 时无 bundle，记为 `unchanged` | 修改：续接运行的最终提交等于前序最终提交时也记为 `unchanged`，并沿用前序的 bundle（D5） |
| Revision D5：公开 API 只暴露元数据，`changed` 表示 `final ≠ base` | 修改：`changed` 表示本次是否产生了新 bundle；投影增加 `priorRevisionId`（D4、D5） |

## 决策

### D1：选取同一 Issue 最近一个可续接的 Revision，在会话启动时固定

会话启动（`starting` → 放出 `agent_session`）的事务里，Cloud 按以下规则选取前序 Revision：

1. 候选：同一 Issue 的其他运行登记的 Revision 中，带 bundle 的那些（自己存了 bundle，或沿用了前序的 bundle，
   见 D5），按登记时间取最新的一个。没有候选时从头开始。
2. 候选的 `project_id` 或 `repository_url` 与本次运行 Workspace 的不同（Issue 改了项目，或项目改了仓库）时，
   不续接，从头开始。前序成果对另一个仓库没有意义，这不算恢复失败。
3. 候选被标为不可续接（D3）时，从头开始。不回退到更早的候选：更早的 Revision 通常基于同一段已被改写的历史，
   回退只会再失败一次，或者悄悄丢掉最近的成果。
4. 否则把候选写入会话启动参数的 `priorRevision{revisionId, finalCommit, bundle{key, size, sha256}}`，
   并写入 `issue_runs.resume_revision_id`。

选取结果在启动参数里固定，之后不再改变；Node 只按参数恢复，不自己查询。

**前提：同一 Issue 上的运行是串行的。** 本文不做协调。两个运行同时处于启动到交付之间时，它们可能选中同一个
前序 Revision，各自产生分叉的成果，下一次运行只会续接其中登记较晚的那个。

为什么只要最近一个：交付的 bundle 相对本次 clone 的基础提交生成（node/revision D3），续接后 Agent 的提交建立在
前序最终提交之上，所以新 bundle 自动包含前序中所有尚未进入远端的提交。逐个应用整条链既不需要，也会让任何一个
旧 bundle 丢失都破坏后续恢复。

### D2：恢复后的 checkout 停在前序成果上，不合并远端

Node 恢复成功后，本地默认分支（clone 的分支）指向前序最终提交，工作区与之一致；`origin/<默认分支>` 保持
本次 clone 时远端的样子。两次运行之间远端前进了也不自动合并，需要时由 Agent 自己 merge 或 rebase。

为什么不自动合并：合并会产生 Ora 自己的提交并引入冲突路径，而续接的语义是“接着上次做”，不是“和远端同步”。
本地分支名不变，是因为 Agent 与用户都按默认分支理解仓库；`origin` 仍可用于对比。

### D3：恢复失败则运行失败；远端改写导致的永久失败把该 Revision 标为不可续接

Node 无法恢复时不启动 Agent，会话以 `agent_failed` 结束，`detail` 是以下之一：

| `detail` | 含义 | Cloud 的处理 |
|---|---|---|
| `prior_revision_unavailable` | bundle 取不到、大小或摘要不符、`bundle verify` 失败、不含预期的最终提交 | 运行 `failed`；不做标记，下一次运行仍会尝试同一个 Revision |
| `prior_revision_base_unavailable` | bundle 依赖的基础提交既不在本次 clone 中，也无法按 SHA 从 `origin` 取到 | 运行 `failed`；在同一结算事务里把该前序 Revision 标为不可续接 |

标记写在 `revisions.resume_refused_at` 与 `resume_refused_reason`，只由会话结束结算写入，只用于 D1 的选取。

为什么默认失败：在缺少前序成果的仓库上运行 Agent，再交付出一个“丢了上次成果”的 Revision，会让下一次续接
悄悄覆盖掉用户的工作。为什么基础提交缺失要例外：它只会因远端历史被改写而永久发生，不标记的话该 Issue 上
之后的每一次运行都会以同样原因失败，用户没有任何出路。对象丢失等其他失败可能是暂时的，保留重试。

### D4：续接对 Agent、用户与 API 可见

- 首条提问的 Context 增加 `resume`：续接时为 `{revisionId, runId, finalCommit, baseCommit}`；本应续接但
  从头开始时为 `{skippedRevisionId, reason}`，`reason` 是 `project_changed` 或 `resume_refused`。它与任务
  一起作为 Thread 的第一条记录出现。
- 公开运行读取增加 `resumeRevisionId`（会话启动后即可见）；运行的 `revision` 投影增加 `priorRevisionId`。
- 恢复成功但前序成果的基础提交已不在 `origin/<默认分支>` 的历史中（远端历史被改写）时，Node 在交给 Agent 的
  首条提问末尾追加一段说明，见 node/revision 后续决策。

### D5：续接后没有新提交时记为 `unchanged`，沿用前序 bundle

交付工作项的输入增加 `priorRevision{revisionId, finalCommit}`（与会话启动参数中的同一个）。Node 的判定变为：

| 最终提交 | 交付结果 | Cloud 登记 |
|---|---|---|
| 等于本次基础提交 | `unchanged`，无 bundle | 与现在相同；`prior_revision_id` 为空 |
| 等于前序最终提交（且不等于基础提交） | `unchanged`，无 bundle，只上传会话记录 | 新行：`prior_revision_id` 为前序，`bundle_revision_id` 指向实际持有 bundle 的 Revision |
| 其他 | `delivered`，上传新 bundle | 新行：自己持有 bundle，`prior_revision_id` 为前序 |

`bundle_revision_id` 总是指向持有 bundle 的那一行（前序自己没有 bundle 时继续沿用它的 `bundle_revision_id`），
所以选取与恢复只需要跳一次。运行结果的 `deliveryState` 按交付结果为 `saved` 或 `unchanged`；公开投影的
`changed` 表示本次是否存了新 bundle。

为什么沿用而不是再存一份：内容完全相同，再存只会让对象存储随空运行线性增长；对象一经写入不可覆盖
（Revision D3），多个 Revision 引用同一个对象是安全的。

## 不变量

1. 没有可续接前序 Revision 的运行，其 Workspace、会话与交付行为与本文之前完全相同。
2. 前序 Revision 在会话启动时固定，之后不变；Node 不自行选择前序。
3. 恢复失败的运行不启动 Agent，以 `agent_failed` 结束，`detail` 说明原因。
4. 只有 `prior_revision_base_unavailable` 会把 Revision 标为不可续接；标记只影响之后的选取。
5. 被选中的前序 Revision 与本次运行属于同一 Issue、同一 Project、同一仓库地址。
6. 每个可续接的 Revision 都能通过它自己或 `bundle_revision_id` 一跳找到 bundle，且该 bundle 的最终提交等于它的最终提交。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 按时间顺序应用该 Issue 的全部 bundle | 不需要（D1 的累积性质），且任何一个旧 bundle 丢失都会破坏之后所有续接 |
| 运行结束后保留 Workspace 等下一次 | 恢复最快；违背根决策“一次性 Workspace、成果保存后释放”，沙盒长期占用 |
| 恢复失败时降级为全新 clone 继续 | 不会卡住用户；但下一次交付会丢掉此前成果，且用户可能察觉不到 |
| 基础提交缺失时回退到更早的 Revision | 也许能恢复一部分；更早的 Revision 通常依赖同一段被改写的历史，且会悄悄丢掉最近的成果 |
| 用户在 @Agent 时选择“续接 / 从头开始” | 最灵活；需要公开 API 与前端交互，留给后续决策，本文只提供自动规则 |
| 续接后空运行也存一份新 bundle | 实现最简单；对象存储随空运行增长，`changed` 也会误导用户 |

## 本决策未解决的问题

- 同一 Issue 并发运行的协调（排队或拒绝）。
- bundle 过期与清理：过期对象会表现为 `prior_revision_unavailable`；清理任务需要决定是否同时标记不可续接。
- 用户显式选择“从头开始”或指定续接哪个 Revision。
- 把成果推送到用户远端。

## 落地顺序

1. specs：本文与契约、Node 两份后续决策。
2. Cloud：迁移（`revisions.prior_revision_id`、`bundle_revision_id`、`resume_refused_at/reason`，
   `issue_runs.resume_revision_id`）、选取、下载授权、结算与登记规则、公开投影，PostgreSQL 集成测试覆盖
   D1 各分支、D3 标记、D5 三种登记。完成后旧 Node 仍可运行没有前序的会话。
3. desktop：契约锁定、Controller 中继与派发能力检查、Node 恢复与交付判定。
4. cluster：在真实栈上连续运行三次，核对第二次看到第一次的提交、第三次的 bundle 包含前两次的提交、空运行
   沿用前序 bundle。

从第 2 步起，有前序 Revision 的新运行会带上 `priorRevision`。旧 Controller 不认识这个字段，会把会话当作普通
会话派给不会恢复的 Node，Agent 将在全新 clone 上运行；因此第 2、3 步必须一起部署，本地栈以同一次 `task up`
构建两者。第 3 步之后，Controller 只把带前序的会话派给声明了恢复能力的 Node。
