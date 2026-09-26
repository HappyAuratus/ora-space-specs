---
status: implemented
---

# 运行时 Workspace 生命周期改为适配 desktop Node：沙盒内由 Node clone，丢弃 Project 共享卷与 worktree 维护 Job

Cloud 运行时 Workspace（表 `workspaces`，不是协作 Space）的操作步骤改为与 desktop 的执行方式一致：
每个 Workspace 的沙盒里运行一个 desktop `ora-node`，仓库由 Controller 经 Controller–Node 协议派发 clone，
由 Node 在自己的 home 中完成。Cloud 不再计划 `storage_ensure`、`worktree_ensure`、`worktree_delete`、
`storage_delete` 四种 effect，不再有 Project 共享卷、bare repository、linked worktree 与维护 Job；
新增 `clone` 步骤与 `workspace_data_delete` effect。Node 在 Cloud 的登记与心跳改由 Controller 代报，
Node 本身不与 Cloud 通信。

本 ADR 处于 `implemented`，是 `cloud/operation` 的根决策；Cloud 侧由 ora-space/cloud#30（迁移
`0016_workspace_runtime_follows_node.sql`）落地，验证证据见
[运行时 Workspace 生命周期核心用例](../../../test-cases/cloud/operation/workspace-runtime-lifecycle.md)。Cloud 的
`docs/core-contract.md`、`docs/execution-contract.md` 已按本文改写。Rust Controller 与 cluster Sandbox Server
的联调不属于本 ADR 的完成条件，由各自的决策承担。兼容性边界是 PostgreSQL 中已有的数据与迁移历史：
本文不删除表或列，已有行按 D6 处理。gRPC 契约的具体形状见
[Controller 接管 Workspace 操作与 Node 上报的契约](../controller-integration/20260926-workspace-operations-and-controller-reported-nodes.md)，
Controller 侧行为见
[Controller 驱动 Workspace 沙盒](../../controller/node-management/0-controller-drives-workspace-sandboxes.md)，
本地 Docker 实现见
[Sandbox Server 的 Docker 沙盒生命周期](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md)。

## 背景：Cloud 与 desktop 的分歧

| 方面 | Cloud 阶段一契约 | desktop 当前实现 |
|---|---|---|
| 仓库在哪里 | 每个 Project 一个共享卷：`repository.git`（bare）+ 每个 Workspace 一个 linked worktree | 每个 Node 在自己的 home 中 clone（`CloneRepository` 执行），无共享对象库 |
| 谁做 Git | Substrate 以 Node 镜像的维护入口跑维护 Job（clone/init、worktree add/remove） | Node 在执行中运行 git，受 host/guardian 纳管，结果经协议事件交付 |
| Node 如何被 Cloud 知道 | Node 用带 `workspaceId`/`sandboxId`/`generation` 的服务凭据直连 `/internal/v1/nodes/*` | Node 只接受 Controller 的会话，不与 Cloud 通信 |
| Node 身份 | 每次进程启动新建一个 Node UUID | 稳定 `NodeId` + 每次进程的 `NodeIncarnationId` |
| 驱动者 | Go 模拟 Controller（JSON `/internal/v1`） | Rust Controller 只跑租户级 clone 闭环（gRPC），固定一个 Node |

阶段一契约写于 desktop Node 的执行方式确定之前，只有模拟器实现它。按“以 desktop 实现为准”，由
Cloud 适配：保留 Cloud 已有的持久操作、effect plan、epoch 对账与 generation fencing，只替换与 desktop
冲突的步骤和 Node 登记方式。

## 继承与修改

| Cloud 现有约定 | 本文处理 |
|---|---|
| 每个外部动作前先持久化 `external_effects` plan，ID 即外部幂等键；旧 epoch effect 先查询再登记 | 继承，不变 |
| `UNIQUE(workspace_id,generation)`、唯一未终止 sandbox；新 generation 前确认旧实例终止 | 继承，不变 |
| node 阶段须有当前实例已初始化、connected、30 秒内心跳的 Node | 继承；上报者改为 Controller（D4） |
| 执行准入 `execution_tickets`、stop 前 quiesce 与 idle 证据 | 继承；idle 证据由 Controller 代报（D5） |
| create_project：storage → worktree → sandbox → node | 修改为 sandbox → node → clone（D1） |
| create_workspace：worktree → sandbox → node | 修改为 sandbox → node → clone（D1） |
| delete_workspace / delete_project 的 cleanup 与 storage_delete | 修改为 `workspace_data_delete`（D3） |
| Project 共享卷 layoutVersion=1、bare repo、linked worktree、branch `ora/{workspace}`、维护 Job | 丢弃（D2） |
| Node 服务凭据直连注册与心跳 | 丢弃，改为 Controller 代报（D4） |
| 租户级 clone 请求（`clone_requests`，前端 clones 页） | 继承，不变；与 Workspace 的 clone 步骤并存（D2） |

## 决策

### D1：操作步骤

| Operation | 新步骤 |
|---|---|
| create_project | （同事务创建 main Workspace）sandbox → node → clone → done |
| create_workspace | sandbox → node → clone → done |
| start | sandbox → node → done |
| stop / administrative_stop | quiesce → terminate → done（不变） |
| delete_workspace | quiesce → terminate → cleanup → done |
| delete_project | quiesce → terminate → cleanup → done |

- `sandbox` 阶段计划 `sandbox_ensure`，请求为 `{kind, projectId, workspaceId}`，不再带仓库字段；
  成功证据仍是 `sandboxInstanceId` 等于 effect ID，并新增 `nodeId`（D4）。
- `node` 阶段的推进条件不变：当前 generation 的 Node 已初始化、connected、30 秒内有心跳。
- `start` 不再 clone：Workspace 的数据卷跨沙盒保留（cluster D4），重启后的 Node 仍有原来的 clone。
- 新 Workspace 在 `clone` 成功后才开放准入（`admission_open`）并进入 `ready`；在此之前 Node 已连接，
  但 Workspace 还没有代码，不能接受任务。

### D2：`clone` 步骤复用 clone 执行登记，结果成为 Workspace 的基线提交

`clone` 阶段由 Controller 向该 Workspace 当前 Node 派发一次 clone，仓库为 Project 的 `repositoryUrl`，
分支为 Workspace 的 `requestedRef`。执行登记沿用 `clone_executions` 与收据、提交身份规则
（controller-integration 根决策 D3、D4），执行额外绑定 `operation_id` 与 `workspace_id`；一个 operation 的
clone 阶段至多一个执行。

| clone 结果 | 阶段推进 |
|---|---|
| `succeeded`（含 commit） | 写入 Workspace 的基线提交，推进到 done，开放准入 |
| `failed`（任一失败码，含 `Interrupted`） | `defer` 为 `retry_wait`/`blocked`，保留执行记录；重试是同一 operation 的新执行 |
| 未知（Node 报告 `Unknown`） | 保持 `blocked`，不自动重试（clone 契约 D6） |

- 复用而不另建表：clone 的派发前登记、原子接管与幂等规则已经在 gRPC 契约中实现并有证据，Workspace
  的 clone 与租户级 clone 的差别只在于“谁发起、结果写到哪里”。
- 租户级 clone 请求保持现状，供没有 Workspace 的部署（固定 Node）与现有前端 clones 页使用；两者都经
  Controller 派发，互不影响。它是否退役留给后续决策。
- 仓库凭据仍按现有规则：Cloud 只提供 credential reference，由基础设施注入，Cloud 不保存密钥值。

### D3：Workspace 数据由 `workspace_data_delete` 清理，`cleanup` 只作用于本 Workspace

`cleanup` 阶段计划 `workspace_data_delete`，请求 `{kind, projectId, workspaceId}`，成功证据
`{ "removed": true }`。计划前 Cloud 要求该 Workspace 没有未终止的 sandbox（与原 `storage_delete` 的前置
条件同理）。`delete_project` 为 Project 下每个 Workspace 各计划一个，全部成功后才推进。

- Workspace 的全部持久数据（Node home、clone 结果、执行账本）都在这一份数据中；没有 Project 级存储后，
  不再需要 `storage_delete`。
- “stop 保留、delete 清理”的原规则不变，只是对象从 `workspaces/{id}/checkout|runtime` 变为 Workspace 的
  数据卷。

### D4：Node 由 Controller 代为登记与上报，Node 身份采用 desktop 的两段式

Controller 在与某个沙盒的 Node 完成握手后，以自己的服务身份（受租约 epoch fencing）向 Cloud 登记该
Node，并在会话存续期间按不超过 10 秒的间隔上报心跳；会话断开时上报 `disconnected`。上报内容：

| 字段 | 来源 |
|---|---|
| `sandbox_instance_id`、`generation` | Controller 调用 `sandbox_ensure` 时的 effect 与 claim 快照 |
| `node_id` | Node 在 `hello_accepted` 中报告，必须等于 `sandbox_ensure` 结果中的 `nodeId` |
| `node_incarnation_id` | Node 在 `hello_accepted` 中报告 |
| `initialized` | 握手完成即为 true（desktop Node 在接受会话前完成恢复） |

`node_instances` 的一行对应一个 incarnation：主键仍是 UUID，由 Cloud 按 `(sandbox_instance_id,
node_incarnation_id)` 幂等分配，新增列 `node_id` 与 `node_incarnation_id`。同一 sandbox 至多一个未结束的
Node（`one_live_node` 不变）；同一 sandbox 出现新 incarnation 时，旧行以 `ended` 结束后才登记新行。

- 为什么由 Controller 代报：desktop Node 只接受 Controller 的会话、不持有任何凭据（传输 D6），给它加一个
  Cloud 客户端和服务凭据，等于在沙盒里分发凭据并多一条出站通道。Controller 本来就持有与 Node 的唯一
  会话，握手时已校验 `NodeId` 与能力，它的观察就是 node 阶段需要的证据。
- 代价：“Node 已连接”的证据从 Node 自证变成 Controller 转述。Cloud 仍校验 Controller 的租约 epoch、
  sandbox 与 generation 的绑定，以及 `node_id` 与 ensure 结果一致；伪造需要攻破 Controller 的服务身份，
  这与 Cloud 已经信任 Controller 对 Substrate 的观察是同一信任边界。
- 原 Node 服务凭据（`sub` 为 Node UUID、带 `workspaceId`/`sandboxId`/`generation`）与
  `/internal/v1/nodes/register|status` 不再用于 desktop Node；模拟器保留这条路径，直到它被 Rust Controller
  取代。经这条路径登记的 Node，`node_id` 与 `node_incarnation_id` 都取服务凭据的 subject，仍须等于
  `sandbox_ensure` 结果中的 `nodeId`。模拟 Substrate 按 sandbox 分配 `nodeId`，只是测试替身的简化，不改变
  本决策对真实 Node 的要求。

### D5：quiesce 的 idle 证据由 Controller 依据派发账本给出

quiesce 阶段 Controller 为 Workspace 当前 Node 上报 idle，证据仍绑定
`operationId + Workspace + Node + admissionEpoch + Node version`。Controller 只有在该 Node 上没有尚无
终态的派发（包括状态未知的执行）时才上报 `idle=true`；否则上报 `idle=false`，按原规则失败该 operation
并恢复准入。

desktop Node 只执行 Controller 派发的工作，没有 Cloud 执行票据以外的旁路入口；Controller 是唯一派发者，
它的派发账本就是 Node 上全部可能的活动。Cloud 另外确认零活动 `execution_tickets`，两者共同构成原
“Node 本地原子 idle”的替代。Node 本地的准入关闭（idle 与新派发之间的竞争）由 Controller 在上报 idle
之前停止向该 Node 派发来保证。

### D6：已有数据与在途操作

- 表 `project_storage`、`workspace_worktrees` 与相关列保留，不再为新 Project/Workspace 写入；迁移不删除、
  不改写已有行，它们只作为历史与审计记录。
- 迁移时处于 `storage` 或 `worktree` 步骤、且未终态的 operation 以 `failed` 结束，错误码
  `lifecycle_flow_retired`；它们没有 sandbox，也没有外部可观察的 Node 活动。用户可重新发起同一操作，
  按新步骤执行。已计划但未对账的 `storage_ensure`/`worktree_ensure` effect 保持原记录。
- 已经 `ready` 的旧 Workspace（只可能由模拟器创建）在下一次 `start` 时按新步骤运行；它们没有数据卷
  中的 clone，需要重新创建 Workspace 才能得到代码。这是可接受的：阶段一没有生产数据。
- 迁移时停在 `storage_delete` 步骤、且未终态的 `delete_project` 退回 `cleanup`，按 D3 为每个 Workspace
  删除数据后完成，而不是以 `failed` 结束：它已越过 sandbox 终止与准入关闭，失败会留下一个已关闭却未删除、
  也无法重新删除到底的 Project。这是实现时对本节的补充。
- 数据库触发器拒绝新的退役步骤（`storage`、`worktree`、`storage_delete`）、退役 effect 与旧表新行，旧记录只读保留。
- 新的迁移是前向、可重试的 SQL 文件，不修改已应用的迁移。

## 不变量

1. 新计划的 effect 只有 `sandbox_ensure`、`sandbox_terminate`、`workspace_data_delete`（及插件 effect）；
   不再计划 storage/worktree 类 effect。
2. Workspace 只有在当前 generation 的 Node 已登记且 clone 成功后才开放准入。
3. node 阶段的证据只来自持有有效租约的 Controller，且其中的 `node_id` 等于 `sandbox_ensure` 结果。
4. 同一 sandbox 同时至多一个未结束的 Node incarnation。
5. 一个 operation 的 clone 阶段至多一个 clone 执行；失败后的重试是新执行，未知结果不自动重试。
6. `workspace_data_delete` 只在该 Workspace 没有未终止 sandbox 时计划。
7. 迁移不删除、不改写已有的存储与 worktree 记录。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 保留 Cloud 的共享卷 + worktree，改 desktop Node 适配 | desktop 已实现并验证 clone 执行、中断判定与恢复；共享 bare repo 需要跨沙盒 RWX 卷与 Git 锁语义，阶段一文档自己也列为未验证（本文背景） |
| 保留 `worktree` 步骤，由 Node 在其中 clone | 名称与证据（`jobTerminated`、linked worktree 路径）都不再成立，保留只会让契约名实不符 |
| Node 直连 Cloud 登记 | 需要在沙盒内分发 Cloud 服务凭据并开出站通道，违背传输 D6 的“Node 不持凭据”（D4） |
| 每个 Node 进程一个新 NodeId，沿用 Cloud 原模型 | 同一数据卷被新身份打开，需要迁移 Node 账本；desktop 协议已有 incarnation 区分进程（D4） |
| 新建 Workspace clone 专用表 | 与现有 clone 登记重复实现派发前登记与原子接管（D2） |
| 迁移时删除 storage/worktree 表 | 违反“PostgreSQL 数据是硬兼容边界”，且丢失审计记录（D6） |

## 本决策未解决的问题

- 租户级 clone 请求是否退役、前端 clones 页是否改为 Workspace 视图。
- isolated Workspace 的工作分支：原契约由 Cloud 生成 `ora/{workspace}`，desktop 的 clone 只检出
  `requestedRef`；分支创建属于 Node 的 worktree 决策。
- 生产 Substrate 适配器的服务认证与 `workspace_data_delete` 对应的平台操作。
- suspend/resume。
- Workspace 的 `requestedRef` 取自 Project 默认分支；默认分支为 `HEAD` 时 Cloud 原样下发，desktop 的 clone
  策略是否接受需与 Node 的 clone 决策对齐。

## 落地

Cloud 侧已在一次变更中完成：proto 与生成物（只增不改）、`docs/` 改写、迁移与 `internal/core` 的步骤表、
`workspace_data_delete` 与 Controller 代报的 Node 事务，以及 Go 模拟 Controller/Substrate 按新步骤工作
（模拟 Node 真实 clone）。剩余的联调，即前端创建 Workspace 经 Rust Controller 与 cluster Sandbox Server 完成
sandbox → node → clone，由 Controller 与 Sandbox Server 的决策承担。

保持不变的行为：租约、epoch、effect plan 与对账、generation fencing、执行准入与 stop 的 quiesce 规则、
租户级 clone。已经改变的行为：新 Project/Workspace 按新步骤执行，Controller 可通过 gRPC 登记 Node。
