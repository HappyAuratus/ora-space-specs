---
status: implemented
---

# Node 交付 Revision：保留 Agent 的提交，剩余改动以内部快照提交，相对基础版本生成增量 bundle，连同会话记录直传对象存储

Agent 会话结束后，Node 以一次交付执行保存成果：保留 Agent 在 checkout 上产生的普通提交；工作区中仍有
应保存的改动时，用独立临时 index、`write-tree`、`commit-tree` 在 HEAD 之后追加一个**内部快照提交**，不
经过提交钩子、不改动工作区、暂存区与分支；Revision ref 指向最终提交。Node 生成相对 clone 基础提交的
累计增量 bundle 并校验，然后用 Cloud 签发的上传授权把 bundle 与封口后的会话 JSONL 直接上传到对象存储，
报告对象摘要与最终提交。没有任何改动时不生成 bundle。

本 ADR 处于 `implemented`，是领域 `node/revision` 的根决策，描述 desktop 中 Node 当前的交付实现。Cloud 的
登记与校验见 [Cloud Revision 根决策](../../cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)，消息见
[Node 协议后续决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)。继承
[clone 最小执行契约](../repository/20260918-minimal-clone-execution-contract.md) 的 Git 执行约束（禁用 hooks、
不改用户仓库配置、受 host/guardian 纳管）。从 Revision 恢复 Workspace 由
[恢复前序 Revision 后续决策](20261010-restore-prior-revision-before-session.md) 承担，它同时修改了本文 D3 的
`unchanged` 条件；本文不推送到用户远端。

## 决策

### D1：交付只在会话结束且其进程已停止后开始

`DeliverRevision` 引用的会话执行必须已有终态，交付对 checkout 的引用是 clone 执行 ID，Node 从账本解析路径
（协议 D1）。

- 会话在账本中不存在或尚无终态时，交付以 `failed{session_not_settled}` 结束。
- 会话已有终态、但会话 actor 仍持有会话记录（尚未退出）时，交付等待下一轮再判断，不算失败。
- 会话 actor 在释放插件租约前停止插件进程树；交付开始时 Agent 插件及其进程树已停止，快照期间没有 Agent
  继续修改文件。

实际决策与偏离：原决策要求“会话的进程 scope 已确认关闭”。Agent 插件进程目前不在进程宿主的 scope 中，
Node 无法证明插件留下的后代进程都已退出，因此交付前提落实为“会话已有终态、会话 actor 已退出、插件进程树
已停止”。脱离 stdio 的后代进程要等进程纳管覆盖插件后才能回收。交付自身的每条 Git 命令都在独立的受管
Scope 中运行，并在下一条命令开始前关闭。

### D2：内部快照不经过提交钩子，也不改动工作区与分支

1. 解析 checkout 的 Git 目录、`HEAD`、`HEAD^{tree}` 与基础提交；任一不可解析时 `checkout_unavailable`。
2. 设置 `GIT_INDEX_FILE` 为临时 index，`git read-tree HEAD`，`git add -A`，`git write-tree`。生成的树与
   `HEAD^{tree}` 相同即无改动；不相同说明有应保存的改动：已跟踪文件的修改与删除、未被忽略的新文件。被
   `.gitignore` 忽略的文件不收录。
3. 有改动时 `git commit-tree --no-gpg-sign <tree> -p HEAD`。author 为会话输入中的用户 git 身份，committer
   固定为 `Ora <revision@ora.invalid>`，提交信息固定为运行的未提交改动快照，并说明它没有运行提交钩子、不代表
   项目校验通过。
4. 无改动时最终提交就是 HEAD。
5. 无论有无改动，`git update-ref <revision_ref> <最终提交>`。`revision_ref` 由交付输入给出（Cloud 生成，形如
   `refs/ora/revisions/<runId>`），协议只接受 `refs/ora/revisions/` 前缀加合法 ref 名。

- 真实 index、工作区文件与当前分支都不变；`commit-tree` 不运行任何钩子，所以项目的 pre-commit、
  commit-msg 失败不会让成果无法保存。快照不代表项目校验通过。
- 用树比较而不是 `git status` 判断改动：结果相同，但不受改动文件数量影响（受管 Git 的输出捕获有上限）。
- 只保存 HEAD 所在的提交链；Agent 另建的分支、stash 与 tag 不在 Revision 中。
- 交付 Git 沿用 clone 的受管执行与加固配置，以工作负载用户运行，并额外禁用提交钩子、fsmonitor 与提交签名。
  临时 index 与 bundle 写在 checkout 自己的 Git 目录（工作负载用户无权写 Node home），文件名按执行区分，
  开始前清除同名残留。
- 快照前 Node 以不跟随链接的 `lchown` 遍历把 checkout 交还工作负载用户，权限位不变，链接不能把这次特权
  改动引到 checkout 之外。Agent 插件已以工作负载用户运行，这一步用于兼容旧 Node 留下的 checkout。
- 子模块只记录 gitlink，不收录子模块工作区内容。clone 禁用了 LFS smudge，未改动的 LFS 指针按普通文件收录，
  不上传 LFS 实体。

已知边界：仓库本地配置中的 clean 过滤器（包括 `filter.lfs.clean`）没有被覆盖，仍可能在 `git add` 时以工作
负载用户运行；没有配置 clean 过滤器时，Agent 新写入 LFS 路径的文件按原样收入 blob。

### D3：累计增量 bundle

- 最终提交等于基础提交（clone 结果的 commit）时结果为 `unchanged`，不生成 bundle。续接运行另见恢复后续
  决策 D5：最终提交等于前序最终提交时同样为 `unchanged`。
- 否则 `git bundle create <tmp> <revision_ref> ^<base_commit>`，再在该 checkout 中 `git bundle verify`；
  失败时删除 bundle，结果为 `bundle_failed`。bundle 只含基础提交之后的对象，恢复时需要从用户远端取得基础提交。
- 每次交付都相对 clone 基础提交生成，不依赖任何更早的 bundle；Agent 若改写了基础提交之前的历史，bundle
  仍以基础提交为前提，恢复要求不变。
- 交付输入的 checkout 必须是该会话自己的 clone 执行，且 clone 记录的提交等于输入的基础提交，否则
  `checkout_unavailable`。

### D4：冻结、上传与摘要

- 会话 JSONL 在会话结束时已封口。Node 先把它复制到 Node home 下按执行固定的私有目录（`0700`，文件 `0600`），
  再做快照；bundle 生成后也复制到同一目录。两者分别计算大小与 SHA-256，连同最终提交、对象键写入账本冻结，
  之后才可能发出首个 PUT。
- 上传前重新测量冻结文件，与冻结的大小和摘要不一致即 `failed{upload_failed}`。先上传 bundle，再上传 JSONL；
  `unchanged` 只上传 JSONL。
- 请求原样携带授权中的签名头，包括 `x-amz-checksum-sha256` 与 `If-None-Match: *`；对象只能首次创建。2xx 与
  `412` 都视为该对象已提交。`412` 只说明对象已存在，Node 照常报告冻结的声明，由 Cloud 核对对象大小与摘要，
  不能凭状态码推断内容相同。
- 授权缺失或距过期不足 5 秒时发送 `UploadGrantNeeded`（带每个仍需上传对象的冻结 SHA-256，协议 D5 补充）并
  等待；等待期间每 2 秒、以及每个新的 Controller 连接上重发，等待不计入失败次数，也没有超时。`403` 视为授权
  失效，丢弃并重新请求。
- 每个对象最多 3 次 PUT（`403` 也计入），失败间隔 500 毫秒，仍失败则 `failed{upload_failed}`。
- 上传授权只在内存中持有，不进入账本、日志或错误信息。
- 成功结果：`delivered{final_commit, base_commit, revision_ref, bundle{key, size, sha256}, history{key, size, sha256}, node}`
  或 `unchanged{final_commit, base_commit, revision_ref, history{...}, node}`；`node` 是准备并冻结这些字节的 Node
  运行身份。

### D5：失败码与重试

| 失败码 | 含义 |
|---|---|
| `session_not_settled` | 会话不存在或尚无终态 |
| `checkout_unavailable` | 找不到 clone 执行、checkout 不属于该会话、基础提交不符，或 checkout 已损坏（包括仓库没有提交） |
| `snapshot_failed` | 快照命令失败 |
| `bundle_failed` | bundle 生成或校验失败 |
| `history_unavailable` | 会话 JSONL 不存在或无法读取 |
| `upload_failed` | 冻结字节被改动，或上传次数耗尽 |

账本冻结之前的执行在重启后从头重新准备，先清除上一次的残留；冻结之后，Controller/Node 重启必须继续同一
执行的持久对象字节和元数据，不能重新生成带不同时间戳的快照或重新打包 bundle。Node 启动时只保留已冻结
执行的目录。冻结目录在终态写入账本后删除，此后保留可重放的终态证据直至 Cloud 接管。Node 关停、Controller
断开与生命周期取消都不生成交付失败终态。需要改变交付内容时由 Cloud 新建交付工作项与执行，使用新对象键，
不能覆盖原执行的首次对象。

## 不变量

1. 快照与交付不修改工作区文件、真实 index 或任何分支，只写 `refs/ora/revisions/` 下的 Revision ref。
2. 交付开始时会话已有终态，会话 actor 已退出，Agent 插件进程树已停止。
3. 内部快照不运行提交钩子。
4. bundle 只相对 clone 基础提交生成，且在上传前通过 `git bundle verify`。
5. 上传的每个对象都带本地计算的 SHA-256 校验。
6. 没有改动时不生成 bundle，结果明确为 `unchanged`。
7. 同一执行在部分上传后重启仍使用相同对象字节与声明，且不持久保存上传授权。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 普通 `git commit` 保存剩余改动 | 最直观；提交钩子失败会让成果无法保存（Revision 方案已否决） |
| `git stash` | 保留 staged/unstaged 区别；恢复后重新出现未提交修改，不是可直接恢复的文件版本 |
| 相对上一次 Revision 的增量 bundle | 更小；上一次 bundle 过期会破坏后续恢复 |
| 上传整个工作区归档 | 不依赖 Git；无法表达提交历史，体积大，也包含忽略的依赖与缓存 |

## 本决策未解决的问题

- Agent 其他分支、子模块内容与 LFS 实体的保存。
- 仓库尚无任何提交时（没有基础提交）的处理；目前表现为 `checkout_unavailable`。
- 交付后工作区是否需要保持 clean（交互式场景）。
- Agent 插件纳入进程宿主 scope 后，交付前提收紧为“会话进程 scope 已关闭”。

## 验证

核心用例见 [Node 的快照与 bundle 交付](../../../test-cases/node/revision/snapshot-bundle-delivery.md)。
desktop `apps/ora-node/src/revision/tests/snapshot.rs` 以真实 Git 覆盖有/无改动、修改删除与新增、忽略文件、
提交钩子失败不影响快照、工作区与分支不变、bundle 可在只有基础提交的新仓库中 fetch 出最终提交、续接后停在
前序最终提交时为 `unchanged`；`upload.rs` 以本地 HTTP 替身覆盖签名头原样携带、`412`、`403` 换授权、次数耗尽、
部分上传后重启复用冻结字节、冻结字节被改动；`apps/ora-node/tests/standalone/agent_sessions/delivery.rs`
以真实 Node 进程完成交付且账本中没有签名 URL。子模块、LFS、clean 过滤器、无提交仓库以及 Node 服务层的
`session_not_settled` 分支尚无直接测试。
