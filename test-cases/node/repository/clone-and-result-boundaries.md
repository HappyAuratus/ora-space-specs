# 仓库获取与结果边界

依据[clone 根决策](../../../decisions/node/repository/0-clone-selected-repository-branch.md)，
本文件登记已批准的目标与职责边界。目标目录、凭据、分支移动、checkout 形态及失败恢复策略
已由[最小执行契约](../../../decisions/node/repository/20260918-minimal-clone-execution-contract.md)批准，
具体义务见[执行核心用例](minimal-clone-execution.md)。Node clone 已有直接证据；Controller／Client 闭环仍 Missing。

## Clone Uses The Requested Repository And Branch

### 风险

执行端隐式使用默认分支、另一个仓库或已有 checkout，会让实际结果偏离调用方意图。

### 前置状态

可识别的源仓库具有不同内容的多个分支，指定分支不是默认分支；执行环境没有已有 Main Workspace。
使用已批准的 HTTPS／显式 SSH 传输及 Node 分配的专用目标目录。

### 触发

调用方提交指定仓库和分支的 clone 请求；另提交无法满足的指定分支。

### 必须成立

- Node 根据显式仓库与分支执行，不要求预先准备 main worktree。
- 成功必须有匹配请求的实际仓库事实；不能满足指定分支时不能静默换成默认分支成功。
- 分支移动时返回实际获取并检出的 commit；重放不更新历史结果，具体义务见执行核心用例。

### 禁止结果

clone 另一个仓库、静默换分支、把 EnsureWorktree 或已有目录当作仓库获取完成。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 实际仓库与请求的非默认分支匹配，且不依赖已有 Main Workspace | `managed_https_clone_selects_branch_and_replays_without_network` 以空 repositories 注册执行真实 HTTPS，核实非默认分支 commit／内容 | Covered |
| 无法满足指定分支时不以其他分支宣称成功 | `managed_https_failures_retain_targets_and_new_retry_uses_new_directory` 覆盖缺失分支及 tag-only 源 | Covered |

### 决策依据

根决策 D1、继承与修改及不变量 1。

## Clone Completion Requires Node Facts And Controller Takeover

### 风险

一次 spawn 回复或 CLI 演示被当成业务闭环，或 Controller 错误地在自己的环境执行 clone。

### 前置状态

调用方、Controller 和 Node 的职责可区分；源仓库可访问，执行端使用显式注入路径。
请求／结果／持久接管的具体协议待所属决策冻结。

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
| Node 执行事实经过 Controller 回到调用方 | 独立进程与实际 Git、持久结果、Client 可观察状态；尚无 | Missing |
| 接受和完成分离，结果交接恢复保留身份 | 具体协议冻结后的交接故障与持久状态检查；尚无 | Missing |

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
