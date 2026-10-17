---
status: implemented
---

# 首个 Node 业务闭环改为 clone 指定仓库的指定分支

最小闭环以调用方指定的仓库和分支为输入，由 Node 在执行环境完成 clone，Controller 协调并接管结果，
调用方可观察状态。不再以已有 Main Workspace 上的 task worktree 创建／删除为首个目标。

本 ADR 处于 `implemented`：clone 已作为独立能力实现于 desktop 的 `apps/ora-node/src/repository.rs` 与
`repository/` 子模块、`crates/node-protocol`（`CloneRepositoryMessage` 及结果）和 `crates/node-db`（clone 执行记录），
Controller 在 Cloud 模式下把它作为 Workspace 操作的 clone 步骤派发并接管结果。输入、目标目录与失败恢复由
[最小执行契约](20260918-minimal-clone-execution-contract.md)规定。仓库获取是独立能力边界，不塞进 Worktree 创建协议。

## 继承与修改

这是 repository 叶子的根决策。

| 既有约定 | 处理 |
|---|---|
| Node 执行 Git，Controller 协调并接管结果 | 继承，不变 |
| 第一条闭环要求已有 Main Workspace，创建／删除任务 worktree | 修改：首条闭环改为仓库 clone，不把已有主工作树作为 clone 前置 |
| 执行身份、持久去重、结果查询、重放和确认 | 继承可靠性原则；clone 的消息与结果由执行契约规定 |
| 显式状态路径、文件保护、Linux rootless BestEffort | 继承，不变；clone 目标目录不是 Node／process 状态目录 |
| 当前信任本机 Node，安全体系与 Strong 延期 | 继承；不据此认定私有 Git 仓库无需凭据 |

依据：[Controller–Node Protocol](../protocol/0-controller-node-protocol.md)、[process](../process/README.md)。

## 决策

### D1：明确仓库与分支输入

目标是 clone 指定仓库的指定分支，不自行改为默认分支或依赖已有 checkout。地址类型、分支约束、移动分支与
实际 commit 结果由执行契约规定。

### D2：完成是业务事实，不是一次启动回复

Node 负责实际 Git 执行，结果经 Controller 回到调用方；进程启动或退出不能直接代表仓库获取已经完成。原请求、
原尝试、实际仓库事实与调用方可见状态分别记录，Controller 不自己 clone。

### D3：Worktree 不再是 Node 的业务能力

clone 闭环落地后，Worktree 管理没有继续实现：Node 服务拒绝 `EnsureWorktree` 与 `RemoveWorktree`，Controller
不派发 Worktree 执行，[Worktree 根决策](../worktree/0-worktree-management.md)已标为放弃。协议类型与崩溃恢复路径
只为兼容与恢复已有记录而保留；已有记录和用户文件不因目标变化被清空或重新解释。

实际决策与偏离：原决策保留 Worktree 能力，只暂停其排期。实现中 clone 成为唯一的仓库获取能力，Worktree
被放弃而不是暂停，原因是没有调用方需要它，保留执行路径只会增加需要维护的恢复分支。

## 不变量

1. 指定仓库与分支是显式输入，不静默用另一个仓库或默认分支替代。
2. clone 结果只由 clone 自身的执行证据判定，不借用 Worktree／process 的测试或记录。
3. 调整目标不授权覆盖现有目录、删除部分未知文件或重置历史执行身份。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 继续先做 Worktree 端到端 | 可复用更多业务代码，但不符合调整后的首个交付目标 |
| 将 clone 当作 EnsureWorktree 的隐式准备动作 | 少一个显式能力，却混淆已有仓库与仓库获取的前置和失败责任 |
| 直接调用 CLI 就宣布闭环完成 | 能证明 Git 可运行，但缺少 Controller／Node／调用方的结果与恢复闭环 |

## 核心测试用例

[仓库获取与结果边界](../../../test-cases/node/repository/clone-and-result-boundaries.md)登记本决策的验证义务；
执行层义务见[最小 clone 执行契约用例](../../../test-cases/node/repository/minimal-clone-execution.md)。

## 落地

Node clone 执行、clone 专用协议与持久记录、Controller 派发与接管按执行契约落地。保持不变：已有 Node／process
状态文件的含义。开始改变：Node 的仓库获取只经 clone，Worktree 请求被拒绝。
