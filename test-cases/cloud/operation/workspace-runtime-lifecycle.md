# 运行时 Workspace 生命周期

依据[已批准的运行时 Workspace 生命周期决策](../../../decisions/cloud/operation/0-workspace-runtime-follows-desktop-node.md)。
本文件登记 Cloud 运行时 Workspace（表 `workspaces`）按 sandbox → node → clone 推进、丢弃 storage/worktree
步骤、由 Controller 代报 Node 的承诺；gRPC 形状见
[Workspace 操作与 Node 上报用例](../controller-integration/workspace-operations-and-node-reports.md)。
实现尚未开始，证据均为 `Missing`。

## Workspace Opens Admission Only After Node And Clone Succeed

### 风险

Node 已连接但仓库尚未 clone 时开放准入，任务在没有代码的沙盒里执行；或 clone 失败、未知时仍把
Workspace 标为 ready。

### 前置状态

新建 Project 或 isolated Workspace，Controller 持有有效租约。

### 触发

`create_project`/`create_workspace` 依次推进 sandbox、node、clone；clone 分别成功、失败（含 `Interrupted`）、
结果未知。

### 必须成立

- sandbox 阶段只计划 `sandbox_ensure`（请求不含仓库字段），成功证据 `sandboxInstanceId` 等于 effect ID。
- node 阶段要求当前 generation 的 Node 已由 Controller 登记、initialized、30 秒内心跳。
- clone 成功后写入基线提交、`admission_open = true`、Workspace `ready`；失败进入 `retry_wait`/`blocked`
  且保留执行记录；未知保持 `blocked`，不自动重试。
- 一个 operation 的 clone 阶段至多一个进行中的执行；重试是新执行。
- `start` 走 sandbox → node → done，不再 clone。

### 禁止结果

clone 未成功时开放准入；计划 `storage_ensure`、`worktree_ensure`、`worktree_delete`、`storage_delete`；
未知 clone 结果被自动重试。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 新步骤表与只计划保留的 effect | PostgreSQL 集成测试逐步推进 create_project、create_workspace、start，断言每步计划的 effect 种类 | Missing |
| clone 成功才开放准入 | 在 node 阶段完成、clone 未完成时调用执行准入，断言 `execution_closed` | Missing |
| clone 失败与未知的处理 | 以失败与未知结果推进，断言 operation 状态与执行记录 | Missing |

### 决策依据

运行时 Workspace 生命周期 D1、D2，不变量 1、2、5。

## Workspace Data Is Deleted Only After Every Sandbox Is Terminated

### 风险

Workspace 或 Project 删除时仍有沙盒在写数据卷，清理与写入竞争。

### 前置状态

Workspace 有未终止或已终止的 sandbox。

### 触发

`delete_workspace`/`delete_project` 推进到 cleanup。

### 必须成立

- cleanup 为每个 Workspace 计划 `workspace_data_delete`，计划前要求该 Workspace 没有未终止 sandbox。
- `delete_project` 在全部 Workspace 的数据删除成功后才推进到 done，不再有 `storage_delete`。

### 禁止结果

存在未终止 sandbox 时计划 `workspace_data_delete`；Project 删除跳过任何 Workspace。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 终止后才计划删除 | 集成测试在 sandbox 未终止时计划 cleanup，断言 `termination_unconfirmed` | Missing |
| Project 删除覆盖全部 Workspace | 两个 Workspace 的 Project 删除，断言两个 effect 均成功后才 done | Missing |

### 决策依据

运行时 Workspace 生命周期 D3，不变量 6。

## Retired Storage And Worktree Steps Leave History Intact

### 风险

迁移删除或改写已有 storage/worktree 记录，或在途旧步骤 operation 永远卡住。

### 前置状态

升级前数据库中存在 `project_storage`、`workspace_worktrees` 行，以及停在 `storage`/`worktree` 步骤的 operation。

### 触发

应用新迁移。

### 必须成立

- 旧行保持原值；在途旧步骤 operation 以 `failed`、`lifecycle_flow_retired` 结束；已计划的旧 effect 记录保留。
- 迁移可重试；新 Project/Workspace 不再写入旧表。

### 禁止结果

删除或改写旧行；在途旧步骤 operation 继续推进或保持非终态。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 从上一版 schema 升级 | 迁移测试在旧 schema 上构造数据后升级，断言旧行不变与 operation 终态 | Missing |
| 迁移可重试 | 重复执行迁移，断言结果不变 | Missing |

### 决策依据

运行时 Workspace 生命周期 D6，不变量 7。
