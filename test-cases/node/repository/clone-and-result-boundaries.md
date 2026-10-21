# 仓库获取与结果边界

依据[clone 根决策](../../../decisions/node/repository/0-clone-selected-repository-branch.md)，
本文件登记已实现的目标与职责边界。目标目录、checkout 形态及失败恢复策略
由[最小执行契约](../../../decisions/node/repository/20260918-minimal-clone-execution-contract.md)规定，
具体义务见[执行核心用例](minimal-clone-execution.md)。输入由[Node 从快照 clone 决策](../../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)
修改：clone 的输入是 Git Server 的快照，不再是外部仓库地址与分支，也不再有 SSH、部署凭据与 `origin`；对应义务见
[快照 clone 与 `ora-git`](snapshot-clone-and-ora-git.md)。证据核对基线为 desktop `ff9de8f9`。

## Clone Uses Only The Requested Snapshot

### 风险

执行端隐式使用另一个分支、另一个提交或已有 checkout，会让实际结果偏离调用方意图。

### 前置状态

Git Server 上的快照对应不同内容的多个分支，被请求的 Workspace 分支不是默认分支；执行环境没有已有 Main Workspace。
使用 Node 分配的专用目标目录与 `git_access` 身份。

### 触发

调用方提交指定快照（`snapshot_ref`、`base_commit`、`workspace_ref`）的 clone 请求；另提交无法满足的快照输入。

### 必须成立

- Node 根据显式快照输入执行，不要求预先准备 main worktree。
- 成功必须有匹配请求的实际仓库事实：Workspace 分支指向 `base_commit`；不能满足时不能静默换成其他分支或提交成功。
- 返回实际检出的 commit，必须等于 `base_commit`；重放不更新历史结果，具体义务见执行核心用例。

### 禁止结果

clone 另一个仓库、静默换分支或提交、把 EnsureWorktree 或已有目录当作仓库获取完成。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 实际提交与请求的快照匹配，且不依赖已有 Main Workspace | `apps/ora-node/tests/standalone/repository.rs` 的 `managed_snapshot_clone_checks_out_the_base_commit_without_a_remote` 以空 repositories 注册执行快照 clone，核实 Workspace 分支、提交与内容 | Covered |
| 无法满足请求的快照时不以其他提交宣称成功 | `snapshot_failures_retain_targets_and_new_retry_uses_new_directory` 覆盖快照缺失、授权被拒绝与快照所指提交不符 | Covered |

### 决策依据

根决策 D1、继承与修改及不变量 1；快照 clone 决策 D1、D3。

## Clone Completion Requires Node Facts And Controller Takeover

### 风险

一次 spawn 回复或 CLI 演示被当成业务闭环，或 Controller 错误地在自己的环境执行 clone。

### 前置状态

调用方、Controller 和 Node 的职责可区分；Git Server 的快照可访问，执行端使用显式注入路径。
请求／结果／持久接管的协议见 clone 的执行契约与快照 clone 决策 D10。

### 触发

提交 clone，观察接受、实际执行和结果到达调用方的过程；在回复／结果交接处中断并恢复连接。

### 必须成立

- Node 执行 Git，Controller 协调并持久接管结果，Client 能观察结果。
- 原请求、进程尝试与仓库事实分别解释，启动接受或单一退出码不直接当业务完成。
- 重连沿用稳定执行身份，查询／重放及确认遵循持久接管原则；恢复按最小执行契约核实旧尝试和仓库事实。

### 禁止结果

Controller 本地 clone 冒充 Node 执行、尚未完成就展示成功、用 Worktree 测试替代 clone 验收。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Node 执行事实经过 Controller 回到调用方 | `apps/ora-controller/tests/workspaces.rs` 的 `create_and_stop_drive_the_workspace_sandbox_and_its_node` 断言 Controller 在 Workspace 创建中派发 clone、接管结果后才开放准入，Cloud 侧由模拟器代替；`apps/ora-node/tests/standalone/repository_controller.rs` 的 `independent_controller_replays_durable_takeover_after_lost_ack_and_kill` 以独立进程与实际 Git 验证 Controller 接管；真实 Cloud 与浏览器链路只有 cluster 的手工验收 | Partial |
| 接受和完成分离，结果交接恢复保留身份 | `apps/ora-node/tests/standalone/repository_controller.rs` 的 `independent_controller_replays_durable_takeover_after_lost_ack_and_kill`：Controller 持久接管后、确认前被杀，独立的新 Controller 重放原结果，不再 clone | Covered |

### 决策依据

根决策 D2、继承的可靠性原则及不变量 2。

## Clone Scope Changes Preserve Existing Files And Execution History

### 风险

复用 Worktree 模型或改变最小闭环目标时，覆盖用户目录、抹除历史结果或隐式切换旧写入入口。

### 前置状态

存在 Node／process 状态目录、历史 Worktree 执行和用户文件；clone 目标与既有文件可能冲突。

### 触发

部署后续 clone 能力，提交目标冲突请求并重开持久记录。

### 必须成立

- 调整目标不删除或重新解释已有 Worktree 执行、数据和用户文件。
- clone 目标不充当 Node／process 状态目录；遇到未知已有文件不得无条件覆盖或递归删除。
- 现有 Backend 写入入口没有被范围批准隐式授权切换。具体目录复用、迁移和残留清理另行评审。

### 禁止结果

清空旧目录以使测试成功、历史执行身份被重置、把旧 Worktree 结果解释为 clone 成功。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 已有文件、状态目录及历史记录保持安全 | `repository_recovery.rs`／`repository_policy.rs` 覆盖现有目录及状态路径保护，node-db v1／v2 迁移测试保留历史结果／Run；尚缺完整混合升级流程 | Partial |
| 不隐式复用业务结果或切换写入入口 | 独立 clone 协议／数据库测试与 standalone 执行；Backend 入口未修改；尚缺 Controller 组合 | Partial |

### 决策依据

根决策 D3、继承与修改及不变量 3。
