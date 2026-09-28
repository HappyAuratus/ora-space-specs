# Agent 运行编排

依据[已批准的 IssueRun 根决策](../../../decisions/cloud/issue-run/0-agent-run-in-disposable-isolated-workspace.md)，
事务边界见 [controller-integration D6](../../../decisions/cloud/controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)。
本文件登记 Agent IssueRun 从评论触发到释放运行 Workspace 的阶段承诺。证据预期来自 `ora-space/cloud` 的
`integration` 包（PostgreSQL 集成测试）；实现尚未开始，全部为 `Missing`。

## Agent Target Requires A Project

### 风险

没有 Project 的 Issue 创建出永远无法开始的 `queued` 运行，用户误以为 Agent 在排队。

### 前置状态

Space 有 `active` 的 Space Agent；Issue 未设置 `project_ref`。

### 触发

Task Mode 评论的 `targets` 含该 Agent。

### 必须成立

- 请求以 `409 issue_project_required` 拒绝，评论、IssueRun、Timeline 活动都不写入。
- 设置 `project_ref` 后同样的评论创建 `queued` 运行，运行输入固定了 Agent 的插件 ID 与版本。

### 禁止结果

写入评论但不创建运行；创建运行后再失败。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 无 Project 时整条评论拒绝 | 集成测试：无 `project_ref` 时评论返回 `issue_project_required`，评论与运行计数不变 | Missing |
| 运行输入固定插件版本 | 集成测试：创建运行后升级 Space 插件版本，运行输入与后续会话工作项中的版本不变 | Missing |

### 决策依据

IssueRun D1、D2，不变量 6、7。

## Run Phases Advance Only In Evidence Transactions

### 风险

阶段在证据落库前推进或倒退，导致会话在 Workspace 就绪前下发，或同一阶段的工作被放出两次。

### 前置状态

`queued` 的 Agent 运行，Controller 持有有效租约。

### 触发

依次接管：运行 Workspace `create_workspace` 成功、首条 Thread 事件、会话终态、交付终态、`delete_workspace` 成功。

### 必须成立

- 每次转换与对应证据的接管在同一事务中提交：`provisioning → starting → running → delivering → releasing → done`。
- `starting` 放出恰好一个会话工作项；`delivering` 放出恰好一个未登记的交付工作项。
- Project 忙时运行保持 `queued`/`provisioning`，派发循环稍后重试，不报错。
- `create_workspace` 失败（或 `blocked` 超过 30 分钟）时进入 `releasing`，`status = failed`，`failure_reason = workspace_unavailable`。

### 禁止结果

阶段倒退；同一运行同时存在两个未登记的工作项；会话在 Workspace 就绪前放出；Agent 会话占用 operation。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 完整阶段链 | 集成测试：以 simulator 扮演 Controller/Node 驱动一次运行到 `done`，逐步断言阶段、`status` 与放出的工作 | Missing |
| Project 忙时排队 | 集成测试：同一 Project 已有进行中的 operation，运行保持 `queued`，operation 结束后派发循环创建运行 Workspace | Missing |
| 建 Workspace 失败 | 集成测试：`create_workspace` 失败后运行进入 `releasing`/`failed` 并创建 `delete_workspace` | Missing |
| 钩子失败回滚 | 集成测试：注入业务钩子错误，控制面收据与业务阶段都未写入，重放后成功 | Missing |

### 决策依据

IssueRun D3、D7，不变量 2、5；controller-integration D6。

## Run Workspace Is Released Only After Delivery Settles

### 风险

Revision 未登记就删除运行 Workspace，Agent 的修改丢失；或交付永远失败导致沙盒无限期占用。

### 前置状态

运行处于 `delivering`。

### 触发

交付执行成功、失败、连续失败超过 2 小时、运行 Workspace 的 Node 未知超过 30 分钟；或运行在 `provisioning`/`starting` 被取消。

### 必须成立

- 只有 Revision 已登记、交付按上限放弃、或无会话取消时才创建 `delete_workspace`。
- 交付失败以 30 秒起、10 分钟封顶的退避放出新的交付工作项；放弃时 `deliveryState = failed`。
- `status` 与 `deliveryState` 分别记录：`failed`/`cancelled` 的运行也可以有 Revision。
- `completed` 时 Agent 最后一条消息写为 Agent 署名的回复评论。

### 禁止结果

交付未结算就释放；Node 报告“已上传”即视为交付完成；取消立即删除正在运行的会话的 Workspace。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未结算不释放 | 集成测试：交付失败时运行保持 `delivering`、无 `delete_workspace`，新交付工作项的 `available_at` 符合退避 | Missing |
| 放弃上限 | 集成测试：以注入时钟推进 2 小时，运行进入 `releasing` 且 `deliveryState = failed` | Missing |
| 取消是请求 | 集成测试：`running` 时取消只放出 `EndSession{cancelled}`，会话结束并交付后以 `cancelled` 结束；`starting` 时取消直接进入 `releasing`、`deliveryState = skipped` | Missing |

### 决策依据

IssueRun D4、D5、D6，不变量 3、4。
