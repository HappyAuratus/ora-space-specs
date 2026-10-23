---
status: implemented
---

# Node 只从 Git Server 快照 clone 并校验 `baseCommit`，checkout 不配置 origin；Agent 经沙盒内固定动词的 `ora-git` 请求 Node 代为 fetch 与推送 Workspace 分支

Node 的 clone 不接受外部仓库地址，也不使用部署凭据。clone 输入是
[Git Server 代码中心决策](../../cloud/git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约 ADR”）D5 第
1 项的快照输入 `{repository_url, snapshot_ref, base_commit, workspace_ref}`。Node 沿用
[Node 交付推送与 Git 恢复决策](../revision/20261017-push-delivery-ref-and-restore-from-git-server.md)（下称“交付推送决策”）D1 的私有传输仓库：
以 Node 身份、带 git-access 证书把 `snapshot_ref` 拉进传输仓库，核对它指向 `base_commit`，再以工作负载身份从传输仓库把
对象取入新的 checkout，检出名为 Workspace 分支短名的本地分支。checkout 不配置任何 remote，也不留下任何与证书有关的配置；
成功前校验 HEAD 等于 `base_commit`。会话期间，Agent 用沙盒里的 `ora-git` 命令发起 fetch 与推送：命令只有固定的几个动词，
经本地 Unix socket 请求 Node 执行，Node 按 socket 所属的会话判定身份，用该会话执行的 ID 访问 Git Server，私钥不离开 Node。
推送把 checkout 的提交当数据搬进传输仓库，带声明的旧值推到 `refs/heads/workspaces/<workspaceId>/<branch>`；fetch 把 upstream
与其他 Workspace 的分支取到 checkout 的 `refs/remotes/ora/...`。

本 ADR 处于 `implemented`：正文按最终实现表述，与批准版本不同的地方集中在“与批准版本的偏离”一节；核心测试用例见
[内部线用例](../../../test-cases/node/repository/snapshot-clone-and-ora-git.md)。它继承 [clone 根决策](0-clone-selected-repository-branch.md)、
[最小 clone 执行契约](20260918-minimal-clone-execution-contract.md)（下称“clone 契约”）、
[Git 副作用移出受理 worker](20260925-clone-git-effects-run-outside-the-admission-worker.md) 与
[中断失败判定](20260925-terminated-clone-attempt-fails-as-interrupted.md)，承担契约 ADR 中“Node 怎样 clone、fetch、推送 Workspace
分支”。Cloud 一侧的输入与授权见 [clone 步骤消费快照决策](../../cloud/operation/20261016-clone-step-consumes-binding-snapshot.md) 与
[快照 clone 输入与执行授权决策](../../cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md)。涉及
desktop 的 Node 协议（`crates/node-protocol`）、Node 的 clone 执行与核实（`apps/ora-node/src/repository/`）、Git Server 访问与传输仓库
（`apps/ora-node/src/git_server/`）、`ora-git` 命令（`apps/ora-git`）与 Node 的 socket 服务（`apps/ora-node/src/ora_git/`）、Node 数据库
（`crates/node-db`）；cluster 的 Node 配置模板、echo agent fixture 与验收脚本。交付推送（含 Workspace 分支的快进）与续作恢复的规则由
交付推送决策承担，本文只给出它们用到的 clone 记录与账本。

兼容性边界：**破坏性切换**（契约 ADR 范围边界）。Node 协议与 Cloud、Controller 同时发布，旧形状的命令被拒绝；Node 数据库升级时
不改写旧的 clone、会话与交付记录，它们的输入已无法解码，所属 Workspace 重建而不续作。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 根 D1、不变量 1：仓库与分支是显式输入，不静默用另一个仓库或分支替代 | 继承；输入改为快照输入，Node 从不以别的 ref 或提交替代（D1、D3） |
| 根 D2：完成是业务事实；根 D3：Worktree 不是 Node 的能力 | 继承，不变 |
| clone 契约 D1：输入为 HTTPS 或 `ssh://` 地址与分支名 | 修改：只接受 Git Server 的 `https` 仓库地址与快照输入四个字段；不再接受外部地址与 SSH（D1） |
| clone 契约 D2：凭据由 Node 部署提供（HTTPS 凭据来源、SSH 身份与主机校验） | 修改：clone 不使用任何部署凭据；只用 git-access 证书，且只在 Node 身份的传输仓库中使用（D2） |
| clone 契约 D3：Node 在仓库根下分配并独占目标目录 | 继承，不变 |
| clone 契约 D4：以单分支 clone 获取完整历史并设置 `origin`；禁 hooks、不递归 submodule、抑制 LFS | 修改：改为传输仓库拉取快照、工作负载身份建立 checkout；不配置 remote；完整历史、禁 hooks、submodule 与 LFS 规则不变（D2） |
| clone 契约 D5：结果是实际获取的提交，不承诺接受时的远端快照；需要锁定提交时另行引入以提交为输入的能力 | 修改：这就是那项能力。输入锁定 `base_commit`，结果必须等于它，不等即失败（D3） |
| clone 契约 D6：先持久关联再派发；未知不自动重跑或删除 | 继承；传输仓库的只读拉取例外，见 D4 |
| clone 契约 D7：clone 是独立能力与消息 | 继承；消息形状见 D10 |
| clone 契约 D8：网络 Git 使用 `network_timeout_ms` | 继承，并用于传输仓库的拉取、`ora-git` 的 fetch 与推送 |
| Git 副作用移出 worker D1–D5；中断失败判定 D1–D4 | 继承，不变；它们同样适用于 clone 中工作负载身份的命令（D4） |
| 交付推送决策 D1：带证书的 Git 访问以 Node 身份在私有传输仓库中执行，不读取 checkout 的配置、钩子或 ref | 继承，并扩展到 clone、`ora-git` 的 fetch 与推送（D2、D6、D7） |
| 交付推送决策 D2、D3：交付把最终提交推到交付 ref，再带账本中的旧值快进 `workspace_ref`，冻结计划后重放 | 继承；本文提供 `workspace_ref` 的来源（clone 记录）与共用的账本（D6、D8） |
| 交付推送决策 D5：恢复把 Workspace 本地分支移到前序最终提交，改写提示以本 checkout 的 `base_commit` 判定，不改 `refs/remotes/ora/**` 与配置 | 继承；本文提供分支短名与 `base_commit`（D9） |
| 交付推送决策 D7：Cloud 模式必须配置 `git_access` | 继承；clone 也需要它，Node 只在配置了 `git_access` 时声明 clone 能力（D10） |
| [Git Server 根决策](../../cloud/git-server/0-project-code-hub-git-server-over-https.md) D5：沙盒里由 Node 进程执行 Git，证书与上下文头只在当次命令中传入，不写入 checkout 的 `.git/config`、全局配置、日志或 journal | 继承，不变；本文另规定 checkout 不配置任何 remote（D2） |
| Git Server 根决策 D5：Agent 在会话中自己执行 `git push` 没有凭据，会失败；平台管理的写入只经持证组件进入 Git Server | 继承：Agent 仍然没有凭据，它的 Workspace 分支经 `ora-git` 请求 Node 这一持证组件推送（D5、D6） |
| Git Server 根决策 D7：Workspace 的代码输入只来自 Git Server，Node 不为平台管理的代码输入访问外部仓库 | 继承；本文是它在 Node 一侧的落实：clone 只接受 Git Server 的快照输入（D1、D2） |
| [Node 协议决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)：会话执行的输入与事件 | 继承；会话输入增加 Workspace 目录，其余 Workspace 信息从 checkout 的 clone 记录取得（D10） |
| Git Server 根决策 D2：沙盒持 Sandbox Server 按租户、Workspace、代次另签的 git-access 证书 | 继承，不变；Node 以部署提供的这份身份访问 Git Server |

## 决策

### D1：clone 输入与校验

`CloneExecutionSpec` 为 `{node_id, repository_url, snapshot_ref, base_commit, workspace_ref}`，四个业务字段由 Cloud 固定、
Controller 原样转交，消息拒绝未知字段。Node 受理时校验：

| 字段 | 规则 |
| --- | --- |
| `repository_url` | `https`；没有用户信息、查询与片段，主机与路径非空；拒绝空白、控制字符与反斜杠。不接受 `ssh://` |
| `snapshot_ref` | `refs/ora/upstreams/<bindingId>/snapshots/<oid>`，`<bindingId>` 是规范的小写 UUID，`<oid>` 等于 `base_commit` |
| `base_commit` | 完整提交 ID（40 或 64 位小写十六进制） |
| `workspace_ref` | `refs/heads/workspaces/<workspaceId>/<branch>`，`<workspaceId>` 是规范的小写 UUID，`<branch>` 满足 D5 的分支参数规则 |

- 同一执行身份必须匹配原输入与目标 Node，同身份不同输入冲突（clone 契约 D1，不变）。
- `<branch>` 是 Workspace 分支短名，也是 checkout 中本地分支的名字；`<workspaceId>` 与 `<bindingId>` 随 clone 记录保存，
  供 `ora-git` 使用（D5）。
- `workspace_ref` 的分支短名与 `ora-git` 的分支参数使用同一条规则，Workspace 自己的分支因此总能被 `ora-git push` 表达。

为什么只接受 `https`、不接受 SSH：Git Server 只提供 smart HTTP（Git Server 根决策 D1）；输入里的地址只可能指向它。
保留 SSH 等于保留一条 Node 访问外部仓库的路径，违背契约 ADR 需求 3。

### D2：经私有传输仓库 clone，checkout 不配置 origin

一次 clone 分两段，身份不同：

1. **拉取快照（Node 身份）。** Node 按交付推送决策 D1 为该执行建立私有的裸传输仓库（位于 clone 根目录下的 `.ora-transport/`，
   由执行 ID 派生名字，只有属主可访问），只把 `snapshot_ref` 拉进传输仓库的输入 ref，不取 tag。证书、私钥、CA 与
   `Ora-Execution-Id: <clone 执行 ID>` 只作为当次命令的参数传入；credential helper、askpass、终端提示、重定向、hooks、系统与
   全局配置都被禁用，只允许 `https`。然后确认输入 ref 是提交且等于 `base_commit`。
2. **建立 checkout（工作负载身份）。** 传输仓库中没有任何凭据，Node 把它的属主交给工作负载用户。在已预留的目标目录中，
   以工作负载身份、沿用 clone 契约的受管执行与加固配置（禁用 hooks、抑制 LFS、不递归 submodule、不读模板、禁用 SSH），依次
   执行三条固定命令：初始化仓库并让 HEAD 指向 `refs/heads/<branch>`；从传输仓库把输入 ref 取入本地分支 `<branch>`；按 HEAD
   填充工作区。只有取入这一条命令放开 `file` 协议（与恢复时的取入相同）。

完成后（无论成败）删除传输仓库；结果为 Unknown 时保留。Node 启动时按交付推送决策 D1 清理残留，只保留工作负载段尚未完成本地
取入的 clone 传输仓库（这一步之前不会再拉取快照，它的 Run 也可能仍在 guardian 下读取传输仓库）。

checkout 的结果：

- 没有任何 remote（没有 `origin`，也没有指向传输仓库的 remote），没有 `remote.*`、`http.*`、`credential.*` 配置；
- 有完整的、非浅的历史；没有 `objects/info/alternates`，对象全部在 checkout 自己的对象库里；
- 当前分支是 `<branch>`，指向 `base_commit`，已跟踪文件与 HEAD 一致；
- 没有 `refs/remotes/**`。之后的 `refs/remotes/ora/**` 只由 `ora-git fetch` 与推送写入（D6、D7）。

为什么不让工作负载身份直接 clone：证书私钥只有 Node 身份可读（Git Server 根决策 D5），以工作负载身份访问 Git Server
就要把私钥交给工作负载。为什么不让 Node 身份直接 clone 出 checkout：checkout 属于工作负载，之后 Agent 会改写它的配置；
Node 身份的 Git 进程从一开始就不进入 checkout，规则最简单，也和交付、恢复一致。

为什么不配置 origin：checkout 里没有任何凭据，origin 指向 Git Server 也无法访问；指向外部仓库则会诱导 Agent 或工具绕过
平台直接访问外部仓库。没有 remote 时，所有远端交互只能经 `ora-git`，失败的原因也明确。

### D3：成功的核实与失败分类

成功前，在收尾确认之后（中断失败判定不变量 1），Node 以只读命令核实：

- 目录身份与预留一致，是非 bare、非浅、使用自身 `.git` 的仓库，`objects/info/alternates` 与 `commondir` 不存在；
- `git remote` 输出为空，没有 `refs/remotes/**`；
- `refs/heads/<branch>` 存在，`HEAD` 是指向它的符号引用；
- `HEAD^{commit}` 等于 `base_commit`；已跟踪文件相对 HEAD 没有改动。

成功结果 `CloneReady` 的 `commit` 等于 `base_commit`，协议层同样拒绝不等的成功结果。

| 情况 | 失败码 |
| --- | --- |
| Git Server 拒绝 Node 的身份（`401`）、拒绝访问（`403` 及其他 `4xx`，无权与不存在给出相同应答），或报告没有 `snapshot_ref` | `SnapshotUnavailable`，不重试 |
| 拉取到的输入 ref 不是提交，或不等于 `base_commit` | `BaseCommitMismatch` |
| `5xx`、`429`、连接、DNS、TLS、低速中止、时限到期，以及无法识别的传输失败，重试耗尽 | `SourceUnavailable` |
| 传输仓库的本地读取或移交失败，工作负载段的命令以非 0 退出 | `OperationFailed` |
| 任一受管命令被终止且收尾已确认 | `Interrupted` |

- `BranchNotFound` 不再产生：快照 ref 缺失归入 `SnapshotUnavailable`；协议拒绝带 `BranchNotFound` 的新结果。
- 目标目录在创建时已经存在，或核实不通过（包括 HEAD 不等于 `base_commit`）：保留 Unknown 与目录，不报告成功。
  `DestinationConflict` 保留在协议中，Node 的 clone 不产生它。
- 为什么 `BaseCommitMismatch` 单独成码：它说明 Git Server 上的快照 ref 与 Cloud 登记的不一致，正常情况下不会发生
  （快照 ref 只能新建，契约 ADR D4）；与网络失败混在一起，会掩盖数据不一致。
- 为什么无法识别的失败按暂时性失败处理：Node 只能从 Git 的诊断输出分类；把未知原因当作暂时性失败，代价是最多多拉取两次，
  而把暂时性失败误判为快照缺失会让本可成功的 clone 直接失败。

### D4：持久阶段与恢复

clone 的持久阶段沿用 Git 副作用移出 worker 的步骤机，并在 `DirectoryCreated` 与 `Dispatched` 之间增加 Node 身份的拉取：
`Reserved → DirectoryCreated → Fetching{累计次数} → Dispatched{当前步骤} → 终态`，阶段只向前推进。

| 阶段 | 内容 | 恢复 |
| --- | --- | --- |
| `Fetching` | 拉取快照到传输仓库并核对（D2 第 1 段）；每次拉取尝试开始前先持久记录累计次数 | 重启后丢弃传输仓库，立即重新拉取；累计次数到 3 次后以 `SourceUnavailable` 结束 |
| `Dispatched` | 工作负载段的三条固定命令（D2 第 2 段），每条命令先持久关联再派发 | 按中断失败判定：已退出、已终止、未核实分别处理；不在目标目录中重跑 |

工作负载段持久记录当前步骤，每一步最多一个 Run；前一步成功且清理完成后才能记录下一步。重启按当前步骤的原 Run 核对，
不在目标目录中新建第二个 Run。

- 为什么拉取可以在重启后重做：它只写 Node 私有的传输仓库，不碰目标目录，也不改变 Git Server（只读）。clone 契约 D6
  禁止重跑，是为了不在已有残留中再次写入目标目录；这一段没有这样的残留。
- 拉取对暂时性失败（D3 表中 `SourceUnavailable` 一行）重试，间隔 5 秒、30 秒，与交付推送决策 D4 的分类相同；`SnapshotUnavailable`
  一行不重试。每次拉取受 `network_timeout_ms` 约束。
- **段内重试与重启后的重新拉取共用同一个累计次数，上限 3 次**：一次拉取无论是因为重试还是因为重启而发起，都先把累计次数
  加一并持久化，再开始拉取；最坏一共拉取 3 次。退避间隔不持久化，重启后的下一次拉取不再等待剩余的间隔。为什么合并成一个
  计数：两个计数相乘会让最坏情况变成 9 次拉取，Node 反复重启时 clone 迟迟不失败；一个持久计数也让“已经试了几次”只有一个
  事实来源。
- 拉取失败时目标目录为空，残留按现有规则保留并报告。

### D5：`ora-git`：Agent 发起 Git 远端操作的唯一入口

Node 镜像在工作负载的 `PATH` 上提供 `ora-git` 命令。它不执行 Git，也不持有凭据，只把请求经本地 Unix socket 交给 Node，
由 Node 执行并返回结果。

**动词**（固定集合，不接受任何选项、refspec、URL 或透传给 Git 的参数）：

| 命令 | 访问 Git Server | 作用 |
| --- | --- | --- |
| `ora-git info` | 否 | 打印本 Workspace 的 `workspaceId`、分支短名、`base_commit` 与远端跟踪 ref 的前缀 |
| `ora-git list` | 读 | 列出 upstream 跟踪分支与各 Workspace 的分支（ref 名与提交，来自 Git Server 的 ref 广告），并按会话输入中的 Workspace 目录附上 Workspace 类型、Issue 标题、Issue 与 run 的 ID 和 run 状态（[快照 clone 输入与执行授权决策](../../cloud/controller-integration/20261016-snapshot-clone-input-and-execution-ref-grants.md) D7）；目录里没有的 Workspace 只显示 ID，并注明目录是会话开始时的快照 |
| `ora-git fetch upstream` | 读 | 取 `refs/ora/upstreams/<bindingId>/heads/*` 到 `refs/remotes/ora/upstream/*`（D7） |
| `ora-git fetch workspace <workspaceId> [<branch>]` | 读 | 取 `refs/heads/workspaces/<workspaceId>/<branch>`（省略时为该 Workspace 的全部分支）到 `refs/remotes/ora/workspaces/<workspaceId>/<branch>`（D7） |
| `ora-git push [<branch>]` | 写 | 把本地 `refs/heads/<branch>`（省略时为 Workspace 分支短名）推到 `refs/heads/workspaces/<自己的 workspaceId>/<branch>`（D6） |

**参数限制**（命令行与 Node 各校验一次）：

- `<workspaceId>` 必须是规范的小写 UUID；可以是自己的 Workspace。
- `<branch>` 必须通过 `GitBranchName` 校验，不是 `HEAD`，不以 `-` 开头，不超过 200 字节，不含 `*`、`:`、`+`。
- 多余的参数、未知动词、未知选项一律是 `usage` 错误。`<bindingId>` 不由 Agent 给出：Node 取自本 checkout 的 clone 记录。
- 推送的目标永远在自己的 Workspace 前缀下：命令无法表达写别的 Workspace 的分支。

**启用条件**：Node 配置了 `agent.ora_git_directory`（绝对路径，父目录只由可信属主控制，不与 Node home、host 目录、clone 根目录
或 workload 目录重叠，否则拒绝启动），同时配置了 `clone` 与 `git_access`，且 `process.workload_uid` 已配置并不同于 Node 的有效
UID 时才提供 socket 服务；否则会话没有 socket，`ora-git` 得到 `no_session`（退出码 3）。部署把目录设为 `/run/ora-git`。
Node 强制检查工作负载身份：`workload_uid` 缺失或等于 Node 有效 UID 时不启用 `ora-git`，启动日志写明配置字段与原因，普通
Agent 会话继续运行。Node 通过 `ORA_GIT_DISABLED_REASON` 把原因交给 Agent，CLI 在尝试连接 socket 前以 `no_session` 显示该
原因；不回退到 Node UID。配置独立 UID 时，原有 socket、对端 UID 检查与 Git 请求路径不变。

**socket 与身份判定**：

1. Agent 会话执行进入运行时，Node 在 `ora_git_directory` 中为它创建一个监听 socket，文件名随机生成，并把路径放进 Agent
   插件进程的环境变量 `ORA_GIT_SOCKET`，Agent 启动的子进程随之继承。`ora-git` 只连接这个路径。socket 无法创建时，会话以
   `agent_failed{ora_git_unavailable}` 结束。
   - 目录属主 Node、权限 `0711`：任何本地 UID 都能按完整路径进入，但不能列出目录，拿不到别的会话的 socket 名。
   - socket 文件属主 Node、权限 `0666`：文件系统层不挡连接，是否接受只由第 3 条的对端 UID 检查决定。
2. **会话由 socket 决定，不由请求声明。** 请求中没有执行 ID、Workspace ID 或仓库地址，带未知字段的请求是 `usage`；Node 把
   从某个 socket 收到的请求归属于创建该 socket 的会话执行。
3. Node 在读取请求之前检查连接对端的凭据（`SO_PEERCRED`），对端 UID 必须是工作负载 UID，否则以 `forbidden` 拒绝。
4. 每个请求在访问 Git Server 前（以及每次重试前），Node 确认该会话在 Node 账本中仍在运行：已接受 `EndSession`、会话已结束或
   Node 正在停止时为 `no_session`。交付只在会话结束后开始，所以交付开始后不会再有 `ora-git` 请求访问 Git Server。
5. 会话离开运行时（结束、取消、Node 停止）时，socket 立即拒绝新请求，并等待已在执行的请求完成后再关闭、删除 socket 文件；
   因此会话记为结束之后，它发起的任何操作都不会再写 checkout 或 Git Server。Node 启动时删除目录中按自己命名规则留下的 socket。
6. Node 访问 Git Server 时用该会话执行的 ID 作为 `Ora-Execution-Id`，仓库地址取自 clone 记录。最终授权由 Git Server
   按 Cloud 当前状态判定（快照 clone 输入与执行授权决策 D5）；Node 的检查只是让结束的会话更早、更明确地失败。第一阶段只有
   IssueRun 的会话获得授权，WorkflowRun 的 Agent 会话中访问 Git Server 的动词得到 `denied`。
7. 每个会话同一时刻只处理一个请求，并发的请求立即得到 `busy`。请求是一行不超过 4 KiB 的 JSON，应答以一行结果结束；
   客户端把结果之前的进度行转到 stderr。连接在得到结果之前关闭时，客户端报告 `no_session`。

**错误码**（`ora-git` 的退出码与 stderr 中的稳定代码 `ora-git: <code>: <提示>`；结果写到 stdout）：

| 代码 | 退出码 | 含义 |
| --- | --- | --- |
| `usage` | 2 | 参数或请求格式不合法 |
| `no_session` | 3 | 没有 `ORA_GIT_SOCKET`、socket 不存在、会话不在运行，或 Node 因工作负载身份未独立而禁用服务 |
| `forbidden` | 4 | 对端不是工作负载用户 |
| `busy` | 5 | 本会话已有请求在进行 |
| `not_found` | 6 | 推送时本地分支不存在；fetch 时远端没有匹配的 ref |
| `conflict` | 7 | 推送时远端的当前值不等于声明的旧值（D6） |
| `not_fast_forward` | 8 | 推送的提交不包含账本中该分支的已知值（D6） |
| `denied` | 9 | Git Server 返回 `401`、`403`，或以其他原因拒绝 ref 更新 |
| `unavailable` | 10 | `5xx`、网络、TLS 或时限，重试耗尽 |
| `internal` | 11 | Node 本地失败，原因只记在 Node 日志中 |

`conflict` 与 `not_fast_forward` 的提示写明下一步：先 `ora-git fetch workspace <自己的 workspaceId>`，在本地合并或变基，再推送。
提示通常只含 ref 名与提交 ID，不含 Git 的原始输出。工作负载身份未独立而禁用服务时，`no_session` 的提示例外地包含配置
字段与原因，便于定位和修正部署；该实现偏离及其依据见“与批准版本的偏离”。

为什么用固定动词的命令加 socket，而不是给 Agent 一个能用的 remote：任何能让 `git push` 直接成功的方式，都要让工作负载拿到
证书私钥，或在沙盒里开一个不经 Node 的代理；前者违背 Git Server 根决策 D5，后者让 Node 无法按会话收口。固定动词让 Node 只
执行它能完整校验的操作，Agent 也不需要了解 ref namespace。

为什么身份绑在 socket 上：同一沙盒中工作负载的进程都以同一个 UID 运行，请求内容可以任意伪造；socket 由 Node 为某个会话
创建，连接从哪个 socket 进来是 Node 自己知道的事实。对端 UID 检查挡住 Node 身份以外、工作负载以外的进程（例如以其他 UID
运行的插件）。Node 强制要求独立工作负载 UID 才启用 `ora-git`，缺失或共用 UID 时不创建服务，避免对端检查把工作负载与
Node 当成同一身份。此拒绝仅保证不提供不成立的 `ora-git` 隔离：共用 UID 时 Agent 仍以 Node 身份运行，能读取该身份可读的
git-access 私钥并直接访问 Git Server，停用 `ora-git` 不使这类部署变得安全。私钥不可读仍是[既有安全缺口](#本决策未解决的问题)，
不由本次启用条件检查补齐。

为什么 socket 是 `0666`、只靠对端 UID 判定，而不是用 `0660` 加工作负载组在文件系统层挡住：用组控制要在镜像里多维护一个组，
并让拒绝分散在两层，非工作负载 UID 在文件系统层得到 `EACCES`，`forbidden` 永远不可达，验收也就无法证明 Node 的检查生效。
对端 UID 检查是唯一的判定点，`0711` 的目录与随机名只是让别的进程找不到 socket，不承担授权。

为什么关闭时等待在途请求：若关闭只拒绝新请求而不等待，已经开始的推送可能在会话记为结束、交付开始之后才写入 Workspace
分支或账本，与交付的快进（D8）竞争同一份账本。

### D6：push-workspace：带声明旧值推送自己的 Workspace 分支

`ora-git push [<branch>]`：

1. 以工作负载身份、加固配置在 checkout 中解析 `refs/heads/<branch>^{commit}`，得到提交 `C`；没有该分支则 `not_found`。
2. 声明的旧值 `O` 取 Node 账本中本 checkout 对 `refs/heads/workspaces/<self>/<branch>` 的上次已知值；没有记录时为“不存在”。
   账本只在本 checkout 的推送成功、`ora-git fetch workspace <self>` 与交付的快进成功（D8）之后更新。
3. Node 建立本请求的传输仓库，借用 checkout 的对象库（借用前确认 checkout 没有自己的 `objects/info/alternates` 或 `commondir`，
   否则以 `internal` 拒绝，不发起请求），不读取 checkout 的 ref；以 Node 身份把 `C` 推到 `<dst>`（即
   `refs/heads/workspaces/<self>/<branch>`），只推这一个 ref，不推 tag，并以 `O` 作为声明的旧值。声明旧值只用来比较；能否非快进
   由 Git Server 决定，它总是拒绝非快进（契约 ADR D4）。
4. 结果：

| 情况 | 处理 |
| --- | --- |
| `O` 存在且其对象在 checkout 中可读，`C` 不包含 `O` | `not_fast_forward`，不发起推送 |
| 更新成功，或远端已是 `C` | 成功；更新账本，并以工作负载身份把 checkout 的 `refs/remotes/ora/workspaces/<self>/<branch>` 设为 `C` |
| 旧值不符 | 读取远端当前值 `R`：`R = C` 时按成功处理（上一次推送已写入，但账本没来得及更新）；否则 `conflict`，提示中给出 `R` |
| 其他 ref 更新被拒、`401`、`403` 及其他 `4xx` | `denied`，不重试 |
| `5xx`、`429`、网络、TLS、时限 | 重试，共 3 次，间隔 5 秒、30 秒；耗尽为 `unavailable` |

- 非快进由 Node 在推送前判定：Git Server 的拒绝只给出统一的信息，Node 无法从应答区分非快进与其他拒绝；推送后仍被拒的，
  一律按 `denied` 处理。`O` 的对象在 checkout 中已不存在时（例如 Agent 删除了跟踪 ref 并回收了对象），Node 无法在本地判定，
  推送照常带旧值发出，由 Git Server 的快进规则与旧值比较决定结果（`denied` 或 `conflict`）。
- 推送不改变 checkout 的本地分支、HEAD、工作区与 index。
- 交付与此无关：交付仍把最终提交推到交付 ref（交付推送决策 D2），Workspace 分支是否推送过不影响 Revision。

为什么声明旧值，而不是只依赖服务端的快进规则：快进规则只保证不丢提交；声明旧值还保证“Agent 以为远端在哪”与远端一致，
例如同一 Workspace 的上一次会话推过新提交而本会话没有 fetch，推送会以 `conflict` 失败，提示 Agent 先看一眼再推，而不是
在不知情时把自己的提交叠上去。旧值取 Node 账本而不是 checkout 里的跟踪 ref，因为后者可以被 Agent 任意改写。

### D7：fetch：取入 `refs/remotes/ora/...`

`ora-git fetch upstream` 与 `ora-git fetch workspace <id> [<branch>]`：

1. Node 建立本请求的传输仓库，借用 checkout 的对象库（与 D6 相同的检查）；协商只用 Node 自己写入的 ref，指向本 checkout 的
   `base_commit` 与账本中本 checkout 的已知值（对象不在本地的跳过），使 Git Server 只发送 checkout 尚未拥有的对象。
2. 以 Node 身份、不取 tag，把下表的源 ref 取入传输仓库的暂存 ref：

| 动词 | 源 ref | checkout 中的目标 |
| --- | --- | --- |
| `fetch upstream` | `refs/ora/upstreams/<bindingId>/heads/*` | `refs/remotes/ora/upstream/*` |
| `fetch workspace <id>` | `refs/heads/workspaces/<id>/*` | `refs/remotes/ora/workspaces/<id>/*` |
| `fetch workspace <id> <branch>` | `refs/heads/workspaces/<id>/<branch>` | `refs/remotes/ora/workspaces/<id>/<branch>` |

   没有匹配的 ref 时 `not_found`。失败分类与重试同 D6。
3. 把传输仓库交给工作负载用户，以工作负载身份、加固配置在 checkout 中从传输仓库取入暂存 ref 到上表的目标（只有这条命令放开
   `file` 协议）；取的是自己 Workspace 的分支时更新账本。删除传输仓库。

- fetch 只写 `refs/remotes/ora/**` 与对象，不改本地分支、HEAD、工作区、index 与仓库配置；合并由 Agent 自己做。
- 远端的 ref 不会被删除（契约 ADR D4），所以不需要 prune。
- `refs/remotes/ora/**` 是 Agent 可写的普通 ref，Node 从不信任它们的值：推送的旧值取账本，协商只用账本与 `base_commit`，
  被 Agent 改写的跟踪 ref 在下一次 fetch 时被覆盖。

### D8：交付时把最终提交快进到 `workspaceRef`

交付输入携带 `workspace_ref`（快照 clone 输入与执行授权决策 D6），它与 `base_commit` 都必须等于本 checkout clone 记录中的值，
否则交付以 checkout 不可用失败且不访问 Git Server。交付怎样在推送交付 ref 之后快进它、失败为什么不影响交付，由交付推送决策 D2
承担。本文只规定它与 `ora-git push` 共用 D6 的账本：快进带账本中的已知值作为声明的旧值（没有记录时声明“不存在”），成功后
更新账本；与 `ora-git push` 不同，交付的快进不做本地祖先检查，非快进由 Git Server 拒绝。

### D9：续作恢复用到的 clone 记录

续作恢复由交付推送决策 D5 承担。它从本 checkout 的 clone 记录取得两项输入：协商起点与改写提示的判定基准 `base_commit`，以及
恢复时强制重置的本地分支名（`workspace_ref` 的分支短名，即 clone 时检出的分支）；前序 Revision 的仓库必须等于 clone 记录的
`repository_url`。恢复在 Agent 插件启动之前完成，所以恢复期间会话还没有 `ora-git` 的 socket；恢复不改 `refs/remotes/ora/**`
与仓库配置。

### D10：Node 协议与持久记录

按交付推送决策 D8 的写法列出：

- `CloneExecutionSpec`：只有 `node_id`、`repository_url`、`snapshot_ref`、`base_commit`、`workspace_ref`，校验见 D1；旧的
  `repository`/`branch` 字段与 SSH 来源不再存在。
- `CloneReady`：形状不变，`commit` 必须等于 `spec.base_commit`。
- `CloneFailureCode`：增加 `SnapshotUnavailable`、`BaseCommitMismatch`；`BranchNotFound` 只出现在历史记录中。
- `DeliverRevision`：按交付推送决策 D8 不含 bundle，携带仓库地址、交付 ref、`workspace_ref` 与 `base_commit`（D8）。
- 会话的 `AgentSession`：按交付推送决策 D8 改 `PriorRevision`；携带 `workspace_directory`，Node 只用于 `ora-git list`，不据此做
  任何判断。仓库地址、`workspaceId`、分支短名、`bindingId` 与 `base_commit` 从 `checkout_execution_id` 指向的 clone 记录取得。
- Controller–Node 协议不增加消息：`ora-git` 是工作负载与 Node 之间的本地接口；Workspace 分支的状态由 Cloud 需要时经 ref 查询
  读取（契约 ADR D6）。
- 能力：Node 只在同时配置了 `clone` 与 `git_access` 时声明 clone 能力；Cloud 模式缺少 `git_access` 时以 `git_access_required`
  拒绝启动（交付推送决策 D7，不变）。
- Node 数据库：schema 升到 v10，新增本地账本 `workspace_branch_heads`，按 checkout 与 ref 记录自己 Workspace 分支的已知值
  （D6、D8）；不迁移旧形状的 clone、会话与交付记录。
- Node 镜像包含 `ora-git`，置于工作负载的 `PATH`。

Node 协议的消息拒绝未知字段，新旧 Node 与 Controller 不能混用，与“同时发布”一致。

## 不变量

1. Node 的 clone 只访问输入中的 Git Server 仓库，只读取 `snapshot_ref`；不访问任何外部仓库，不使用任何部署凭据。
2. clone 成功时，checkout 的 HEAD 是本地分支 `<branch>`，指向 `base_commit`；结果的 `commit` 等于 `base_commit`。
3. checkout 中没有任何 remote，也没有证书、私钥路径、`Ora-Execution-Id` 或其他凭据相关的配置。
4. git-access 私钥只被 Node 身份的 Git 进程经命令行引用；工作负载身份的进程从不获得它，带证书的进程从不读取 checkout
   的配置、钩子或 ref。
5. `ora-git` 的请求只归属于创建该 socket 的会话执行，对端必须是工作负载 UID；会话不在运行时请求被拒绝。
6. `ora-git` 只能推送 `refs/heads/workspaces/<自己的 workspaceId>/` 下的分支，每次只推一个 ref，带声明的旧值。
7. `ora-git fetch` 只写 checkout 的 `refs/remotes/ora/**` 与对象。
8. 推送的旧值与 fetch 的协商起点只来自 Node 账本与 `base_commit`，不来自 checkout 中 Agent 可写的 ref。
9. 恢复与交付快进所用的分支短名、`base_commit` 与 `workspace_ref` 都来自本 checkout 的 clone 记录。
10. 旧形状的 clone 命令被拒绝。
11. 交付对 `workspace_ref` 的快进与 `ora-git push` 共用同一份账本。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 以工作负载身份直接 clone Git Server，私钥经继承的文件描述符传入 | 少一个传输仓库；但私钥进入工作负载进程，违背 Git Server 根决策 D5（交付推送决策已否决同一做法） |
| 以 Node 身份直接 clone 出 checkout，再把属主交给工作负载 | 一条命令完成；但之后的 fetch 与推送仍要回到传输仓库模式，clone 成为唯一以 Node 身份写 checkout 的路径，还要额外删除 Node 写下的 remote 配置 |
| checkout 配置 `origin` 指向 Git Server | Agent 熟悉 `git fetch origin`；但没有凭据它永远失败，且诱导 Agent 改 remote 去访问外部（D2） |
| checkout 保留指向外部仓库的 `origin` | 与开发者习惯一致；但让 Agent 与工具绕过平台访问外部仓库，与契约 ADR 需求 4 的方向相反 |
| 沙盒内运行一个 Git HTTP 代理，由它附加证书，Agent 用普通 `git push` | Agent 不需要学新命令；但代理要解析 Git 协议才能按会话与 ref 收口，等于在沙盒里复制一份 Git Server 的授权，且代理本身需要私钥 |
| `ora-git` 透传任意 Git 参数 | 更灵活；但 Node 以 Node 身份运行 Git，任意参数可以改写目标、配置或读取文件 |
| 请求中声明会话执行 ID | socket 可以共用一个；但同 UID 的进程能任意伪造请求内容，身份失去意义（D5） |
| socket `0660` 加工作负载组，在文件系统层拒绝其他 UID | 多一层防护；但要多维护一个组，拒绝分散在两层，Node 的对端 UID 检查无法被观察和验收（D5） |
| clone 拉取的段内重试与重启后的重新拉取各自计数 | 单次重启内的重试不受之前重启影响；但最坏 9 次拉取，反复重启时 clone 迟迟不失败（D4） |
| 推送只依赖服务端的快进规则，不声明旧值 | 少一份账本；但 Agent 在不知道远端已前进时叠加提交，冲突发现得更晚（D6） |
| 旧值取 checkout 里的 `refs/remotes/ora/...` | 不需要账本；但 Agent 可以改写这些 ref，旧值失去意义（D6） |
| fetch 直接写本地分支或自动合并 | Agent 少一步；但会改写 Agent 的工作分支，合并冲突也要由 Node 决定怎样处理 |
| clone 失败时按外部 clone 的方式回退 | 快照缺失时仍能工作；但外部凭据会回到 Node，违背契约 ADR 需求 3 |

## 与批准版本的偏离

对照 desktop `ff9de8f`、cloud `a9c5f20` 与 cluster `b0186a6`，并补入 [desktop#732](https://github.com/ora-space/desktop/pull/732)
（`ae02179`）的 Node 强制 UID 检查，最终实现与本文批准时相比有以下偏离，正文已按最终实现表述：

| 批准时的决策 | 最终实现 | 原因 |
| --- | --- | --- |
| socket 固定在 `/run/ora-git/`，每个 Agent 会话都有 socket | 目录由 `agent.ora_git_directory` 配置（部署设为 `/run/ora-git`），且只在同时配置 `clone` 与 `git_access` 时启用；未启用时会话没有 socket，`ora-git` 为 `no_session`；socket 无法创建时会话以 `agent_failed{ora_git_unavailable}` 结束 | 目录是部署事实，需与 Node 的其他目录分开校验；没有 Git Server 访问的 Node 没有可代理的远端 |
| Node 启动时清空 `/run/ora-git/` | 只删除按 Node 命名规则留下的 socket，目录其余内容不动；目录在启动时恢复为 `0711` | 目录可配置后，Node 只删除能确认是自己创建的文件 |
| socket 随机名含 128 位随机数 | 随机名是一个 v4 UUID（122 位随机数） | 随机名只让别的进程找不到 socket，不承担授权（D5）；授权只由对端 UID 检查决定 |
| 对端 UID 必须是工作负载 UID；独立 UID 曾仅由部署保证 | [desktop#732](https://github.com/ora-space/desktop/pull/732)（`ae02179`）实现 Node 强制检查：缺失 `workload_uid` 或等于 Node 有效 UID 时禁用 `ora-git`，日志说明原因，普通 Agent 会话继续；CLI 得到 `no_session` / 3 | 补齐 circus#410 的要求，不再静默回退到 Node UID；独立 UID 路径保持原样 |
| 会话环境仅约定 `ORA_GIT_SOCKET`，未约定禁用原因变量 | 新增 `ORA_GIT_DISABLED_REASON`，禁用会话导出原因而非 socket；CLI 优先读取原因并返回 `no_session` | 保留普通会话的可用性，同时让 Agent 直接获知服务未启用的原因 |
| 错误 hint 只含 ref 名与提交 ID | 身份配置导致禁用时，`no_session` hint 包含 `process.workload_uid` 配置字段与原因；其他提示约束不变 | circus#410 的 owner 要求禁用状态可诊断，不能只报不透明的 socket 失败；不包含 Git 原始输出或私钥 |
| 会话“没有进入结束、没有被取消、交付尚未开始”时请求可执行；会话离开运行时即关闭并删除 socket | 已接受 `EndSession` 也视为不在运行；每次重试前复查；关闭时先拒绝新请求、等待在途请求完成，再删除 socket | 保证会话记为结束、交付开始之后，`ora-git` 不再写 checkout、Git Server 或账本（D5 第 5 条） |
| 应答是若干行进度与一行结果 | Node 只返回一行结果；客户端仍把结果之前的行作为进度转到 stderr | 当前动词没有需要逐步报告的长过程；协议保留进度行的位置 |
| 非快进在推送前判定：`O` 存在且 `C` 不包含 `O` | 仅当 `O` 的对象在 checkout 中可读时才在本地判定；对象缺失时照常带旧值推送，由 Git Server 决定，结果为 `denied` 或 `conflict` | 祖先判定需要 `O` 的对象；Node 不为判定另行拉取对象，Git Server 的快进规则仍保证不丢提交 |
| checkout 自带 alternates 时拒绝借用（未规定错误码） | 以 `internal` 拒绝，不发起请求 | 这是 checkout 被改坏的本地故障，不是参数或授权问题 |
| clone：`401`、`403` 与 ref 不存在为 `SnapshotUnavailable`；`5xx`、`429` 与网络失败重试 | 其他 `4xx`（含 `404`）同样为 `SnapshotUnavailable`；无法从诊断输出识别的失败按暂时性失败重试，耗尽为 `SourceUnavailable`；`ora-git` 的 `denied`、`unavailable` 按同一分类 | Git Server 对无权与不存在给出相同的拒绝；未知原因按暂时性处理的代价有界（D3） |
| clone 失败表含“目录冲突 → `DestinationConflict`” | 目标目录在创建时已存在按 Unknown 保留，Node 的 clone 不产生 `DestinationConflict`，该码只保留在协议中 | 无法确认归属的目录不能判为已知失败（clone 契约 D3、D6） |
| 工作负载段为 `init`、取入并建立本地分支、让 HEAD 指向分支并检出 | `init` 时即让 HEAD 指向 `<branch>`，取入直接写本地分支，第三步按 HEAD 重置工作区 | 结果与核实条件不变（D2、D3），少一次分支切换 |

## 本决策未解决的问题

- **大仓库的首次 clone 时限**：与交付推送决策相同，本文沿用 `network_timeout_ms`。
- **Workspace 分支的非快进改写**：第一阶段只快进（契约 ADR 未解决项）；`ora-git` 不提供 force 推送。
- **非 Agent 的工作负载进程使用 `ora-git`**：目前只有 Agent 会话拥有 socket；插件进程等其他工作负载若需要，另行决定。
- **WorkflowRun 会话的 Workspace 分支**：第一阶段 Git Server 只授权 IssueRun 的会话（D5 第 6 条），WorkflowRun 怎样协作另行决定。
- **纵深安全加固**（契约 ADR 未解决项，第一阶段不处理）：
  - Node 以 Node 身份借用 checkout 的对象库，读取工作负载可控的对象文件，带来的解析与资源消耗攻击面；
  - 本文依赖“工作负载与插件进程读不到 git-access 私钥”这个前提（Git Server 根决策 D5 的既有缺口），端到端验收逐项检查它
    （场景 5g），但不在本文增加新的防护。

## 落地

desktop、cloud 与 cluster 一起合入，属于契约 ADR 落地顺序第 2 步中的内部线，不依赖外部线。
cluster 的 Sandbox Server 按 Git Server 根决策 D2 为每个沙盒、每个代次签发 git-access 客户端身份，存入管理 tmpfs；Node 配置
模板启用 `git_access`（交付推送决策 D7）与 `agent.ora_git_directory`。

保持不变：受理与 Git 执行的线程划分、中断判定、目标目录的独占与保留、会话 JSONL。
从此改变：Node 的代码来源只有 Git Server 快照；交付只经 Git，不再有 bundle；Agent 可以经 `ora-git` 与其他 Workspace 协作。
外部线合并之前，只有用预置工具准备了快照的 Project 能创建 Workspace（契约 ADR 落地顺序）。

## 验收

内部线三篇 ADR 共用一个端到端验收：cluster 的 `task workspace-git:fixture:up` 起栈，`task workspace-git:acceptance` 运行
`scripts/workspace-git-acceptance.py`。

- **起栈**：cluster 的 Compose 加上 Git Server 与 echo agent fixture，Node 配置启用 `git_access` 与 `agent.ora_git_directory`。沙盒
  网络建成 `--internal`，脚本先证明沙盒访问公网仓库失败；被验收 Project 的外部仓库地址故意填外部地址，clone 成功本身证明 Node
  没有使用它。
- **替身**：外部线不在栈中。经公开 API 创建 Project 后，由底座的预置工具把本地 fixture 仓库的提交写成快照 ref 与 upstream 跟踪
  ref，写当前快照记录并推进同步 Operation；需要时再写一份子提交快照与一份无关快照。预置工具不写 Workspace 记录。
- **echo 标记**：`[commit]` 提交（固定身份 `echo <echo@ora.invalid>`），`[push]` 运行 `ora-git push`，`[fetch <ws>]` 运行
  `ora-git fetch workspace <ws>` 并合并 `refs/remotes/ora/workspaces/<ws>/main`；每个标记的结果（`ok` 或错误码）追加在回复里。
  desktop 与 cluster 的 echo fixture 使用相同标记。
- **断言手段**：`psql` 查 Cloud 数据库；`docker exec` 进沙盒以指定 UID 运行 `git` 与 `ora-git`；进 `git-server` 容器查仓库 ref。
  证据只含资源 ID、ref 名、提交与错误码，不含证书内容。

| # | 场景 | 断言 |
| --- | --- | --- |
| 1 | 从快照 clone 并校验 | clone 结果与 Workspace 的 `base_commit_id` 等于快照提交；checkout 的 `git remote` 为空，配置不含 `remote.`、`http.`、`credential.`，`HEAD` 指向 `refs/heads/main` 且等于快照提交，没有 alternates |
| 2 | commit 后 push-workspace | `[push: ok]`；`workspaces/<A>/main` 指向新提交，父提交是快照提交 |
| 3 | 第二个 Workspace fetch、merge 再推送 | `workspaces/<B>/main` 的祖先包含 A 的提交；B 的 checkout 有 `refs/remotes/ora/workspaces/<A>/main`；A 的分支不变 |
| 4 | 交付、Revision 登记、续作恢复 | 交付 ref 指向登记的最终提交，并把 Workspace 分支快进到它（含从未推送的 Workspace）；续作的 checkout 续接前序最终提交，`[push: ok]`；无关快照时出现历史改写提示 |
| 5a | 没有快照 | 同步未结束时等待、同步失败后结束，都没有 clone 执行（见 clone 步骤消费快照决策） |
| 5b | 写别人的分支 | `ora-git push` 带 refspec 或 `--force` 为 `usage`；以 Node 身份直接推别人的分支被 Git Server 拒绝；ref 不变 |
| 5c | 非快进 | 改写历史后推送为 `not_fast_forward`（退出码 8），ref 不变 |
| 5d | 声明的旧值不符 | 远端被另一写入者前进后推送为 `conflict`（退出码 7）并给出远端值；fetch 自己的分支并合并后推送成功 |
| 5e | 旧代次 | 重启 main Workspace 后证书 SAN 的 generation 递增，旧证书访问得到 `403` |
| 5f | 执行结束后再推 | `no_session`（退出码 3），socket 已不存在；以 Node 身份用会话执行 ID 推送得到 `403` |
| 5g | 工作负载读不到证书 | 工作负载 UID 读不到 git-access 证书、私钥与 CA；Agent 环境只有 `ORA_GIT_SOCKET`；不能列出 socket 目录；另一个 UID 能连上 socket 但被 `forbidden`（退出码 4）拒绝 |

5b、5c、5d、5g 在对应会话结束之前执行，5f 在会话结束之后执行。

## 核心测试用例

本 ADR 的核心测试用例见[内部线用例](../../../test-cases/node/repository/snapshot-clone-and-ora-git.md)，逐项登记验证义务、代表性证据与
仍存在的缺口。
