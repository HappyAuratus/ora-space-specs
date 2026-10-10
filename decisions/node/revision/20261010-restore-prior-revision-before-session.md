---
status: proposed
---

# Node 在 Agent 启动前恢复前序 Revision：下载校验 bundle、补齐基础提交、默认分支指向前序最终提交；交付把“等于前序”视为无改动

会话输入带前序 Revision 时，Node 在定位会话 checkout 之后、把 checkout 交还工作负载用户并启动 Agent 插件
之前，依次：经 Controller 申请只读下载授权，下载 bundle 并核对大小与 SHA-256，确认 bundle 依赖的提交都在
checkout 中（缺失时按 SHA 从 `origin` 补取一次），`git bundle verify` 后把 bundle 里唯一的 Revision ref 取入
checkout，再把本地默认分支指向前序最终提交。任何一步失败都不启动 Agent，会话以 `agent_failed` 结束。交付时，
最终提交等于前序最终提交也算无改动，只上传会话记录。

本 ADR 处于 `proposed`，继承 [Node 交付 Revision 根决策](0-internal-snapshot-and-incremental-bundle-upload.md)，
业务规则见 [IssueRun 续接决策](../../cloud/issue-run/20261010-resume-run-from-latest-issue-revision.md)，消息形状见
[契约决策](../../cloud/controller-integration/20261010-revision-restore-contract.md)。涉及 Node 的会话启动、交付判定、
Node 协议与 HTTP 下载。兼容性边界：不带前序的会话与交付行为不变；Node 本地目录布局只增加一个私有的临时
下载目录，启动时清空。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D2：快照不改动工作区与分支 | 继承；恢复发生在会话开始前，是会话输入要求的初始状态，不属于交付 |
| 根 D3：最终提交等于基础提交时 `unchanged` | 修改：或者等于交付输入中前序的最终提交（D5） |
| 根 D3：bundle 相对 clone 基础提交生成 | 继承，不变；续接后的 bundle 因此包含前序的提交 |
| 根“本决策不涉及从 Revision 恢复 Workspace” | 修改：本文承担恢复到新运行 Workspace |
| clone 契约：Git 以受管执行与加固配置运行 | 继承；恢复中的每条 Git 命令使用与交付相同的执行方式和配置 |

## 决策

### D1：恢复在会话 checkout 定位后、Agent 启动前进行

顺序固定为：解析 checkout → 确认已安装的插件版本（并持有插件租约）→ 恢复 → 把 checkout 交还工作负载用户 →
准备会话目录 → 启动插件 → 发送首条提问。插件缺失的会话因此不会下载任何东西。
恢复期间会话已是运行中，但 Agent 尚不存在，所以失败时只需结束会话：

- 结束结果为 `agent_failed`，`detail` 为 `prior_revision_unavailable` 或 `prior_revision_base_unavailable`
  （契约 D3）；Thread 中只有首条提问与结束。
- Node 重启时进行中的恢复与会话一样以 `interrupted` 结束，不重做；此时 checkout 可能已指向前序最终提交，
  交付判定（D5）对两种状态都给出确定的结果。

### D2：下载授权与下载

- Node 发送下载授权请求（不占事件序号），在 Controller 连接上等待授权；等待期间每 2 秒、以及每个新连接上
  重发。等待与下载合计有上限（5 分钟），超时按 `prior_revision_unavailable` 失败；Controller 转告 Cloud 拒绝
  签发时立即失败。
- 下载写入 Node 私有目录下按执行区分的临时文件（目录 `0700`、文件 `0600`）；完成后核对大小与 SHA-256，不符
  （包括比声明大）即失败。网络错误、429 与 5xx 重试 3 次；403 视为授权失效并重新申请一次，之后的 403 与其他
  4xx 立即失败。
- 下载器不跟随重定向；授权 URL 与签名头只在内存中，不写日志、不进入错误信息或会话记录。临时文件在恢复结束
  （无论成败）后删除，Node 启动时清空该目录。

### D3：Git 恢复步骤

所有命令沿用交付的受管执行与加固配置（禁用 hooks、fsmonitor 与签名），以工作负载身份运行。

1. 读取 bundle 头：只接受 v2/v3 格式；前置提交列表（`-<oid>` 行）与 head 列表。head 必须恰好一个，名称在
   `refs/ora/revisions/` 下，指向会话输入中的前序最终提交；否则 `prior_revision_unavailable`。
2. 逐个确认前置提交在 checkout 中存在。有缺失时执行一次 `git fetch --no-tags origin <缺失的 oid…>`（使用 clone
   的环境与 CA 配置），再确认。只有远端明确回答没有该对象（`not our ref`、未公布的对象、没有该远端 ref），或
   补取成功后仍缺失，才是 `prior_revision_base_unavailable`；远端不可达或拒绝访问是 `prior_revision_unavailable`，
   以免一次网络故障让 Cloud 永久拒绝该 Revision。
3. 把校验过的 bundle 从 Node 私有目录复制到 checkout 的 Git 目录（属主为工作负载用户，结束后删除），因为以
   工作负载身份运行的 Git 读不到 Node home；然后 `git bundle verify`，失败则 `prior_revision_unavailable`。
4. `git fetch <bundle> <head>:<head>`，把 Revision ref 取入 checkout；clone 的协议策略禁止 `file`，只有这一条
   命令放开它。所有补取与取入都关闭自动 gc 与 maintenance。再确认前序最终提交可解析为提交。
5. `git checkout --force -B <默认分支> <前序最终提交>`，默认分支是 clone 时检出的分支；`origin/<默认分支>`
   不动。之后工作区与 index 等于前序最终提交。
6. 若前序 bundle 的任一前置提交不是 `origin/<默认分支>` 的祖先（远端历史被改写，D4），记下以便提示。

为什么读 bundle 头而不是先 `verify`：`verify` 在前置提交缺失时只报错，无法区分“基础提交取不到”与“bundle
损坏”，而这两种失败在 Cloud 的处理不同（IssueRun 续接决策 D3）。为什么只补取一次、只从 `origin`：基础提交
只可能来自项目仓库；反复尝试不会改变已被回收的结果。

### D4：远端历史被改写时提示 Agent

D3 第 6 步发现前置提交不在 `origin/<默认分支>` 的历史中时，Node 在发给 Agent 的首条提问末尾追加一段固定
格式的说明：已恢复的前序最终提交、它依赖的基础提交，以及“远端默认分支的历史已不包含该基础提交”。这段文字
随首条提问进入会话记录与 Thread，用户与 Agent 都能看到。其他情况不追加任何内容。

### D5：交付判定

交付输入带 `prior_revision` 时：

| 最终提交 | 结果 |
|---|---|
| 等于基础提交 | `unchanged`（与根 D3 相同；例如恢复在中途被中断） |
| 等于前序最终提交 | `unchanged`，`final_commit` 为前序最终提交，不生成 bundle |
| 其他 | `delivered`，bundle 仍是 `<revision_ref> ^<base_commit>` |

第二行只在没有任何新提交、工作区也没有未提交改动时成立（有改动时快照会产生新的最终提交）。

### D6：能力声明

Node 在会话与交付能力都可用时声明 `revision_restore`。未声明时 Controller 不会派来带前序的会话（契约 D5）。

## 不变量

1. 恢复失败时 Agent 插件从未启动。
2. 下载的 bundle 在交给 Git 之前通过大小与 SHA-256 校验。
3. 下载授权与 URL 不持久化、不出现在日志与错误信息中。
4. 恢复只改变本地默认分支、工作区、index 与 `refs/ora/revisions/` 下的 ref，不改 `origin` 的远端跟踪分支与仓库配置。
5. 恢复成功后 HEAD 是默认分支，指向会话输入中的前序最终提交，工作区干净。
6. 交付在最终提交等于前序最终提交时不生成 bundle。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 恢复放在 clone 步骤里 | 恢复与会话天然一体（同一个前序选择）；放进 clone 会让 Workspace operation 依赖 IssueRun 的业务状态 |
| `git pull <bundle>` 合并到默认分支 | 一条命令；会产生合并提交或冲突，而 checkout 刚 clone，直接指向即可 |
| 检出为分离 HEAD 或新分支 | 不动默认分支；Agent 容易困惑或自行建分支，与续接决策 D2 不符 |
| 恢复失败时降级继续 | 见续接决策 D3 的否决理由 |

## 本决策未解决的问题

- 前序 bundle 很大时的下载进度与磁盘配额。
- Agent 留在其他分支上的提交（根决策未保存，续接也拿不回来）。

## 落地

Node：会话启动前的恢复、下载授权等待、受限下载器、交付判定、能力声明；真实 Git 测试覆盖正常恢复、前置提交
缺失后从 `origin` 补取成功与失败、bundle 摘要不符、head 不符、远端改写提示，以及三种交付判定。Controller：
中继与能力检查的集成测试。端到端由 cluster 验收覆盖。
