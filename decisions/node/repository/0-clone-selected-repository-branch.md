---
status: approved
---

# 首个 Node 业务闭环改为 clone 指定仓库的指定分支

最小闭环以调用方指定的仓库和分支为输入，由 Node 在执行环境完成 clone，Controller 协调并
接管结果，Client 可观察状态。不再以已有 Main Workspace 上的 task worktree 创建／删除为首个目标。

2026-09-18 Eric 已批准本根决策（approved），确认目标、职责与保留边界。输入、目标目录、
失败恢复由[后续最小执行契约](20260918-minimal-clone-execution-contract.md)进一步批准，具体 wire 与存储仍需细化；
不将已有 Worktree 测试计为 clone 验收。
仓库获取是独立能力边界，不将 clone 塞入 Worktree 创建协议；不改变现有文件或数据库布局。

## 继承与修改

这是 repository 叶子的根决策，与已有 Worktree／protocol 叶子协作，不取代它们的全部业务规则。

| 既有约定 | 处理 |
|---|---|
| Node 执行 Git，Controller 协调并接管结果 | 继承，不变 |
| 第一条闭环要求已有 Main Workspace，创建／删除任务 worktree | 修改：首条闭环改为仓库 clone，不把已有主工作树作为 clone 前置 |
| 执行身份、持久去重、结果查询、重放和确认 | 继承可靠性原则；clone 的具体消息／结果另行设计 |
| 显式状态路径、文件保护、Linux rootless BestEffort | 继承，不变；clone 目标目录不是 Node／process 状态目录 |
| 当前信任本机 Node，安全体系与 Strong 延期 | 继承；不据此认定私有 Git 仓库无需凭据 |
| 现有 Worktree 能力及 Backend 写入路径 | 保留，不删除、不自动迁移或切换 |

依据：[Worktree 根决策](../worktree/0-worktree-management.md)、
[Controller–Node Protocol](../protocol/0-controller-node-protocol.md)、[process](../process/README.md)。

## 背景与约束

首条垂直闭环现在关注仓库获取，而不是预先准备主工作树后的任务分支生命周期。
已有 process 与 Node 持久执行基础可复用，但业务结果和资源前置不同，不能直接宣称 clone 已实现。
本次改变优先交付目标，不扩大到 Cloud、SSH 编排、通用任务引擎或全部 Git 能力。

## 决策

### D1：明确仓库与分支输入

目标是 clone 指定仓库的指定分支，不自行改为默认分支或依赖已有 checkout。
地址类型、分支约束、移动分支与实际 commit 结果由后续最小执行契约明确，本根决策不重复定义。

### D2：完成是业务事实，不是一次启动回复

Node 负责实际 Git 执行，结果经 Controller 回到调用方；进程启动或退出不能直接代表
仓库获取契约已经满足。后续方案必须区分原请求、原尝试、实际仓库事实与调用方可见状态。
本次不定消息名称、schema 或新的结果枚举，不要求 Controller 自己 clone。

### D3：保留已有能力，暂停旧闭环排期

旧 Worktree 实现和评审证据仍有效，但不再是首条闭环的完成条件。未来 Worktree 功能按需求继续。
进入 clone 实现前按实际需求适配现有 Worktree 专用握手／执行结果／存储，不提前建立通用抽象。
已有记录和用户文件不能因目标变化被清空或重新解释。

## 不变量

1. 指定仓库与分支是显式输入，不静默用另一个仓库或默认分支替代。
2. 当前 Worktree／process 的 implemented 或通过测试不等于 clone 完成。
3. 调整目标不授权覆盖现有目录、删除部分未知文件或重置历史执行身份。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 继续先做 Worktree 端到端 | 可复用更多业务代码，但不符合已调整的首个交付目标 |
| 将 clone 当作 EnsureWorktree 的隐式准备动作 | 少一个显式能力，却混淆已有仓库与仓库获取的前置和失败责任 |
| 直接调用 CLI 就宣布闭环完成 | 能证明 Git 可运行，但缺少 Controller／Node／Client 的结果与恢复闭环 |

## 本决策未解决的问题

- 本机首版支持的仓库地址／传输及凭据来源；交互认证是否允许。
- clone 目标的分配、授权根、已有目录和失败残留的归属／恢复／清理规则。
- checkout、单分支、浅克隆、submodule、LFS 与 hooks 的首版边界。
- 分支移动、重传、网络中断及强杀后的结果判定、重试身份和恢复顺序。
- clone 协议、持久状态、Controller 接管和最小 Client 入口的具体契约。

上述前四项的首版政策已由[最小执行契约](20260918-minimal-clone-execution-contract.md)批准；
具体协议与存储、自动清理及扩展能力仍按其后续边界细化，不是默认授权。

## 核心测试用例

[仓库获取与结果边界](../../../test-cases/node/repository/clone-and-result-boundaries.md)登记
已批准范围的验证义务；clone 尚未实现，直接证据均为 Missing。获批的恢复和目录政策已补入
[执行核心用例](../../../test-cases/node/repository/minimal-clone-execution.md)，不由测试文档自行决定未决项。

## 落地顺序

先评审最小输入、结果与恢复边界，再按 Node clone 执行、IPC／Controller 协调、Client 可观察
结果拆分可检视提交；每片包含其职责范围内的失败验证。具体 commit 计划在契约确定后重排，
不机械复用旧 Worktree 七步计划。此轮只维护文档，代码与运行数据保持不变。
