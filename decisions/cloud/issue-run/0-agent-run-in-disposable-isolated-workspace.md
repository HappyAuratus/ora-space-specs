---
status: implemented
---

# Issue 上 @Agent 的 IssueRun 由 Cloud 编排：每次运行一个一次性 isolated Workspace，成果保存为 Revision 后释放

> 历史决策：D5/D8 放弃后的原交付事实入口与 ACK 含义由
> 已落地的[迟到交付事实接管决策](20261022-late-delivery-fact-takeover.md)（`implemented`）D1、D3 修改；无账本、无运维意图也接收，
> 接管不重写已结算 Run，ACK 不表示业务再次成功。迟到 Revision 另受其 D2 的后创建运行守卫约束。
> 本文保留 `implemented`，未改变本文的放弃窗口和正常释放条件。

用户在 Issue 评论中 @ 一个 Agent 并给出任务时，Cloud 已有的 Task Mode 路径创建 `queued` 的 IssueRun。
本决策把它接到真实执行：Cloud 为这次运行创建一个专属的 isolated Workspace（沙盒 → Node → clone →
插件），Workspace 就绪后下发 Agent 会话；会话称为 **Thread**，其记录实时回到 Issue 页，用户可以在
Thread 中继续对话；会话结束后 Node 保存 git 变更与会话记录为 **Revision**，Cloud 校验并登记后删除该
Workspace，释放沙盒与数据。整条编排的权威状态在 Cloud，Controller 与 Node 只执行 Cloud 放出的工作。

本 ADR 处于 `implemented`，是领域 `cloud/issue-run` 的根决策，承担“一次 Agent 运行从哪里开始、经过哪些
阶段、何时结束和释放”的业务不变量。其他环节分别由下列决策承担，本文只引用：

| 环节 | 决策 |
|---|---|
| Thread 记录、用户追加消息与结束 | [Cloud Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md) |
| 对象存储配置、上传授权与 Revision 登记 | [Cloud Revision 根决策](../revision/0-cloud-owned-object-store-and-verified-revisions.md) |
| 用户 git 提交身份 | [identity-access 后续决策](../identity-access/20260928-user-git-commit-identity.md) |
| create_workspace 的插件步骤与运行 Workspace 的删除 | [operation 后续决策](../operation/20260928-plugin-step-and-run-workspace-release.md) |
| 插件由 Node 执行安装 | [plugin-marketplace 后续决策](../plugin-marketplace/20260928-node-executes-plugin-installs.md) |
| gRPC 契约形状 | [controller-integration 后续决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md) |
| Controller 侧行为 | [Controller session 根决策](../../controller/session/0-controller-relays-agent-sessions.md) |
| Node 侧行为 | [协议](../../node/protocol/20260928-streamed-thread-events-and-session-commands.md)、[agent-runtime](../../node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md)、[plugin-runtime](../../node/plugin-runtime/0-node-installs-plugins-into-workspace-home.md)、[revision](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) |

涉及 Cloud 的 `internal/core`（IssueRun、Workspace、operation）、迁移与公开 API。兼容性边界是 PostgreSQL
已有数据：`issue_runs` 已有的列与状态值保持原义，本文只增加列与表；manual run（`POST .../runs`）与
Workflow 路径不变。本文不承担“续接上次 Revision 再次运行”“Team 执行”“多 Agent 协作”的设计。

## 背景

- Cloud 的 Issue 协作层：评论 `targets` 中的 `agent` 目标以 Task Mode 创建 IssueRun，事务提交后调用
  `ExecutionDispatcher.Dispatch`，外部执行经 `ExecutionObserver` 回写 started/progress/message/completed/failed。
  本决策之前只有 `collab` 包中的 fixture 目录和同步 mock Dispatcher。
- 运行时 Workspace 的 sandbox → node → clone 已由 Rust Controller 与 cluster Sandbox Server 跑通
  （[operation 根决策](../operation/0-workspace-runtime-follows-desktop-node.md)）。
- 云端沙盒与 Revision 方案（2026-09-15 讨论结论）已确定：看板自动化运行在成果保存成功后释放沙盒及存储；
  并行任务用独立 Workspace；Revision 以内部快照 + 相对基础版本的累计增量 bundle 保存。
- `one_project_operation` 约束同一 Project 同时至多一个未完成的 operation。

## 决策

### D1：Agent 目标是 Space 内的 Agent，第一版每个已安装的 agent 类插件对应一个

`CollaborationDirectory` 的 Cloud 实现从 fixture 改为读取 Space 的 Agent 表 `space_agents`
（`id uuid`、`space_id`、`tenant_id`、`plugin_id`（canonical `namespace/identifier`）、`display_name`、
`status`）。第一版规则：

- Space 安装一个 `kind = agent` 的插件并收敛为 `installed` 时，Cloud 在同一事务为它创建一行 Agent
  （已存在则恢复为 `active`）；插件被移除时该行变为 `retired`，不删除，历史 IssueRun 仍可解析执行者。
- `issue_runs.executor_id` 引用 `space_agents.id`；IssueRun 创建时把 `plugin_id` 与插件的 desired
  version 快照进运行输入，运行期间插件升级不改变本次运行。
- `retired` 的 Agent 不出现在 @ 列表，`ResolveTarget` 对它返回不存在。

为什么引入一张表而不直接用插件 ID：`issue_runs.executor_id` 是 UUID 列且已被 Task Mode 与 dedup 使用；
Agent 未来会有名称、默认模型、提示词等 Space 级配置，这些属于 Agent 而不属于插件包。第一版不开放 Agent
CRUD，前端 agents 页继续用 mock，直到另立决策。

### D2：Agent 运行必须绑定 Project，运行 Workspace 由运行专属

- Task Mode 选择 `agent` 目标时，Issue 必须已设置 `project_ref`，否则评论以 `409 issue_project_required`
  拒绝，整条评论不写入。拒绝而不是排队，是因为没有 Project 就没有仓库，运行永远无法开始，
  `queued` 会误导用户。
- 每个 Agent IssueRun 恰好对应一个运行 Workspace：`kind = isolated`，运行创建者与操作 actor 为触发用户，`requestedRef`
  为 Project 默认分支（[默认分支决策](../operation/20260926-require-concrete-default-branch.md) D2）。
  `issue_runs` 新增 `workspace_id`，`workspaces` 新增 `issue_run_id`，两者互相唯一引用。
  此处触发用户对应已有运行时权限模型的不可变 `creator_user_id` 与 operation actor；历史
  `owner_user_id` 继续遵循 Project durable owner 复合外键，不因运行触发而重写。
- 运行 Workspace 不出现在用户的 Workspace 列表中，只能经 IssueRun 访问；它不接受其他 IssueRun 或交互式会话。

一次运行一个 Workspace 延续 Revision 方案的“同一 Workspace 内执行串行、并行任务独立 Workspace”：
两个 Agent 同时处理同一 Issue 或同一 Project 的不同 Issue 时，修改归属和沙盒释放互不影响。

### D3：运行阶段是 Cloud 的持久状态机，每一步都由 Cloud 放出、Controller 领取

`issue_runs` 新增 `phase`，只对 `executor_type = agent` 使用；原 `status` 保持原义并由 phase 推导：

| phase | 进入条件 | Cloud 放出的工作 | `status` |
|---|---|---|---|
| `provisioning` | 评论事务提交后的派发 | 运行 Workspace 的 `create_workspace` operation（sandbox → node → clone → plugin） | `dispatched` |
| `starting` | 该 operation `succeeded`，Workspace `ready` | Agent 会话执行（`AgentSession` 工作项） | `dispatched` |
| `running` | Node 报告会话已开始（首条 Thread 事件或会话开始事件被接管） | Thread 命令（用户消息、结束请求） | `running` |
| `delivering` | 会话执行有终态结果（任何结束原因） | Revision 交付执行（`DeliverRevision` 工作项） | `running` |
| `releasing` | Revision 已登记（或按 D5 放弃交付） | 运行 Workspace 的 `delete_workspace` operation | 见 D4 |
| `done` | `delete_workspace` `succeeded`，或删除侧超过 D8 的上限（此时 `done` 不代表删除成功） | 无 | 终态不变 |

- 每个阶段转换都在接管上一阶段证据的同一事务中写入，并在提交后发 `Watch` 信号；Controller 丢信号时
  靠周期领取兜底。没有任何阶段依赖 Controller 内存中的“下一步”。
- `provisioning` 的 operation 受 `one_project_operation` 约束：Project 忙时 Cloud 不报错，运行保持
  `provisioning` 并由 Cloud 的派发循环（周期 ≤ 10 秒）重试创建 operation。operation 都很短（分钟级），
  Agent 会话本身不占用 operation，因此同一 Project 的多个运行只在建删 Workspace 时排队。
- create_workspace `failed` 时运行进入 `releasing`，`status = failed`，`failure_reason = workspace_unavailable`；
  已创建的部分由 `delete_workspace` 清理。Workspace 已就绪但运行固定的 agent 插件没有以固定版本装上时，
  运行同样进入 `releasing`，`failure_reason = agent_plugin_unavailable`。
- 偏离：approved 方案还规定 create_workspace `blocked` 超过 30 分钟视同失败，这一条没有实现。`blocked` 的
  create_workspace 让运行停在 `provisioning`，见“本决策未解决的问题”。

为什么不新增一个覆盖全程的 `run_agent` operation：会话可能持续数小时，期间 `one_project_operation`
会让同一 Project 的其他运行、插件安装和 Workspace 操作全部 `409`；而把会话、交付拆成执行项后，
operation 只覆盖本来就短的建删步骤，已有的 effect plan、epoch 对账和 generation fencing 原样复用。

### D4：执行结果与成果保存分别记录

IssueRun 的最终 `status` 只表达执行结果；成果保存状态由 Revision 表达：

| 会话结束原因 | `status` | Revision |
|---|---|---|
| Agent 完成且 Thread 按 [Thread D4](../thread/0-durable-agent-thread-with-user-turns.md) 结束（用户结束或空闲期满） | `completed` | 尽量保存 |
| 用户取消运行 | `cancelled` | 尽量保存（未完成成果） |
| Agent 失败、Node 中断、会话结果未知 | `failed` | 尽量保存 |

- `status` 由会话的结束原因推导，在进入 `releasing` 时与阶段一起写入，并追加对应
  `run.completed/failed/cancelled` Timeline 活动。
- 偏离：approved 方案还规定 `completed` 时把 Thread 中 Agent 的最后一条消息写为 Agent 署名的回复评论。这一条
  没有实现，因为消息文本在 `ora-history` 的业务记录里，而 [Thread D2](../thread/0-durable-agent-thread-with-user-turns.md)
  规定 Cloud 不解析这些业务字段、不改写 Node 记录；没有约定的字段路径，写评论就要猜测外部 schema。
- 现有 `ObserveCancelled` 只允许 `queued/dispatched → cancelled`；Agent 运行的取消改为请求（D6），
  实际的 `cancelled` 由会话结束后写入，不改变 manual run 的既有转换。
- `issue_runs.result` 保存 `{revisionId | null, deliveryState}`，`deliveryState` 为
  `saved | unchanged | failed | skipped`。

### D5：Revision 登记前不释放；交付失败时保留沙盒重试，有上限

- 进入 `releasing` 的唯一正常路径是 Revision 已登记（[Revision D4](../revision/0-cloud-owned-object-store-and-verified-revisions.md)）。
  Node 只报告“已上传”不构成交付完成。
- 交付执行失败（上传失败、校验失败、Node 报告失败）时，Cloud 以指数退避（30 秒起、上限 10 分钟）
  放出新的交付执行，运行保持 `delivering`，沙盒保持运行。
- 交付连续失败超过 2 小时（从该运行第一个交付工作项的创建时间起算，按数据库时钟比较）、或运行 Workspace
  已不可达（Cloud 观察到该 Workspace 的 Node 状态未知超过 30 分钟）时，Cloud 以 `deliveryState = failed` 放弃
  交付并进入 `releasing`。这两个上限是第一版默认值（Cloud 配置项），用来防止沙盒无限期占用；
  放弃意味着本轮未保存的修改丢失，这是 Revision 方案“第一版故障边界”已接受的代价。

### D6：取消与结束是请求，由会话收尾后生效

用户在 Issue 页取消运行，或在 Thread 中结束会话，Cloud 只记录请求（`issue_runs.cancel_requested_at` 或
Thread 结束请求）并放出 Thread 命令；会话收尾与交付照常进行。取消发生在 `provisioning`/`starting`
（尚无会话）时，Cloud 直接进入 `releasing` 并以 `cancelled` 结束，`deliveryState = skipped`。

不立即删除 Workspace 的原因：Revision 方案要求“正在停止并保存成果”，立即销毁会丢失 Agent 已做的修改。

### D7：真实 Dispatcher 只是把 IssueRun 交给 Cloud 自己的派发循环

生产装配 `AgentRunDispatcher`：`Dispatch` 在一个短事务中为 `queued` 的 Agent 运行创建运行 Workspace 与
`create_workspace` operation，并把运行置为 `provisioning`/`dispatched`；Project 忙时返回
`Accepted = false`，运行留在 `queued`，由 D3 的派发循环重试。它不实现 `SyncExecutor`，后续进展全部由
gRPC 接管路径直接驱动状态机，而不是回调 `ExecutionObserver` 的公开语义。`collab` fixture 继续用于开发
演示，二者由配置选择，默认生产装配。

### D8：运行 Workspace 的删除也有上限，且只以不可达为条件

- 重试来源不变：沿用 operation 后续决策 D4 的 quiesce 失败重试与“同一 operation 的新执行”规则，Cloud 以重新
  声明保证任一时刻至多一个未完成的 `delete_workspace`。
- 上限只取 D5 的不可达窗口：运行 Workspace 仍有存活的沙盒，且 Cloud 观察到的该沙盒 Node 状态未知超过同一个
  配置窗口（默认 30 分钟）。D5 的 2 小时连续失败窗口不适用于删除侧；Node 明确拒绝 quiesce 时 Node 仍然可达，
  不触发放弃。沙盒已终止、删除正在 `cleanup` 步骤推进时也不触发：那时已没有可判“未知”的 Node。
- 超过上限后，`delete_workspace` 以既有失败码 `node_unavailable` 终结，运行进入 `done`；`status` 保持进入
  `releasing` 时按 D4 写入的值，另记 `failure_reason = workspace_unavailable`。
- 残留如实保留：运行 Workspace 行不软删除，`issue_run_id` 仍指向该运行，并有一个终结失败的 `delete_workspace`。
  不新增列、表或后台清理机制；公开 API 对运行 Workspace 的删除恒为 `404`，残留只能由运维处置。

为什么只以不可达为条件：反复失败但 Node 可达的删除仍可能成功，按失败次数放弃会把仍在推进的释放判死；
不可达的 Workspace 则永远等不到 quiesce 证据，没有上限运行会一直停在 `releasing`。代价是放弃后的沙盒与数据不会
自动回收，`failure_reason = workspace_unavailable` 是唯一可见信号。

## 不变量

1. 每个 Agent IssueRun 至多一个运行 Workspace，运行 Workspace 只属于一个 IssueRun。
2. Agent 运行的阶段只由 Cloud 在接管证据的事务中推进；阶段不倒退。
3. 运行 Workspace 的 `delete_workspace` 只在 Revision 已登记、交付被 D5 明确放弃、或 D6 的无会话取消之后创建。
4. IssueRun 的 `status` 与 Revision 的保存状态分别记录；`failed`/`cancelled` 的运行也可以有 Revision。
5. 同一 Project 同时至多一个 operation 的约束不被 Agent 运行绕过；Agent 会话不占用 operation。
6. 运行输入中的插件 ID 与版本在 IssueRun 创建时固定。
7. 没有 Project 的 Issue 不能创建 Agent 运行。
8. `done` 不代表运行 Workspace 已删除；只有 `delete_workspace` `succeeded` 时才是。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 一个 `run_agent` operation 覆盖 sandbox 到释放 | 步骤连续、Controller 只需一个领取入口；但会话持续期间占用 `one_project_operation`（D3） |
| 复用 Project 的 main Workspace 运行 Agent | 省去每次建沙盒和 clone 的时间；但同一 Workspace 内执行必须串行，多个运行互相阻塞，且释放会影响交互式使用 |
| 运行结束后保留 Workspace 等待续接 | 续接更快；但与“自动化运行保存成果后释放存储”冲突，续接改由 Revision 恢复（未决） |
| Dispatcher 直接调 Controller | Controller 只拨出、不开端口（controller api-boundary 决策），且派发必须先持久化 |
| 运行输入只存 Agent ID，执行时再解析插件版本 | 运行期间的插件升级会改变正在执行的运行，无法复现 |

## 本决策未解决的问题

- 在已有 Revision 上续接运行：由 [续接后续决策](20261010-resume-run-from-latest-issue-revision.md) 承担，同一 Issue
  的新运行在 Agent 启动前续接最近一个 Revision。
- create_workspace 长期 `blocked` 时运行停在 `provisioning`：approved 方案的 30 分钟上限没有实现，需要后续决策
  给出上限与终态。
- `completed` 时在 Issue 时间线写出 Agent 结论：需要先约定 Cloud 可读的结论字段（D4 偏离）。
- Team 目标与多 Agent 协作。
- Agent 的 Space 级配置（名称、模型、提示词）与 agents 页的真实 API。
- D5 中各上限的最终取值、运行总时长上限与配额。
- manual run（`POST .../runs`）是否也走本编排。

## 落地

迁移 `space_agents`、`issue_runs.phase/workspace_id/cancel_requested_at`、`workspaces.issue_run_id`；生产装配
`AgentRunDispatcher` 与 10 秒周期的派发循环，`development_fixtures` 配置选择开发用 fixture；Thread、Revision
交付、运行 Workspace 释放与 D8 的放弃路径接入同一状态机，由 PostgreSQL 集成测试覆盖；cluster 的 Agent 验收在真实栈
中走通 `queued → provisioning → starting → running → delivering → releasing → done`。

保持不变：manual run、Workflow 与 Form Mode、fixture 开发装配、已有 Workspace 操作。开始改变：
`agent` 目标的 Task Mode 评论要求 Project，并在生产装配下真正执行。
