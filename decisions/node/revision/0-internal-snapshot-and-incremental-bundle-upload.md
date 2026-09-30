---
status: proposed
---

# Node 交付 Revision：保留 Agent 的提交，剩余改动以内部快照提交，相对基础版本生成增量 bundle，连同会话记录直传对象存储

Agent 会话结束后，Node 以一次交付执行保存成果：保留 Agent 在 checkout 上产生的普通提交；工作区中仍有
应保存的改动时，用独立临时 index、`write-tree`、`commit-tree` 在 HEAD 之后追加一个**内部快照提交**，不
经过提交钩子、不改动工作区、暂存区与分支；Revision ref 指向最终提交。Node 生成相对 clone 基础提交的
累计增量 bundle 并校验，然后用 Cloud 签发的上传授权把 bundle 与封口后的会话 JSONL 直接上传到对象存储，
报告对象摘要与最终提交。没有任何改动时不生成 bundle。

本 ADR 处于 `proposed`，是新建领域 `node/revision` 的根决策。它把 2026-09-15 确认的“云端沙盒与 Revision
方案”中 Node 负责的部分固定为执行契约；Cloud 的登记与校验见
[Cloud Revision 根决策](../../cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)，消息见
[Node 协议后续决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)。继承
[clone 最小执行契约](../repository/20260918-minimal-clone-execution-contract.md) 的 Git 执行约束（禁用 hooks、
不改用户仓库配置、受 host/guardian 纳管）。本文不涉及从 Revision 恢复 Workspace，也不推送到用户远端。

## 决策

### D1：交付只在会话结束且其进程已清理后开始

`DeliverRevision` 引用的会话执行必须已有终态、且其进程 scope 已确认关闭；否则交付以
`failed{session_not_settled}` 结束。这保证快照期间没有 Agent 或其子进程继续修改文件。
交付对 checkout 的引用是 clone 执行 ID，Node 从账本解析路径（协议 D1）。

### D2：内部快照不经过提交钩子，也不改动工作区与分支

1. 以 `git status --porcelain=v2 --untracked-files=all` 判断是否有应保存的改动：已跟踪文件的修改与删除、
   未被忽略的新文件。被 `.gitignore` 忽略的文件不收录。
2. 有改动时：设置 `GIT_INDEX_FILE` 为临时 index，`git read-tree HEAD`，`git add -A`，`git write-tree`，
   `git commit-tree <tree> -p HEAD`。author 为会话输入中的用户 git 身份，committer 固定为
   `Ora <revision@ora.invalid>`，提交信息固定格式注明这是运行的未提交改动快照。
3. 无改动时最终提交就是 HEAD。
4. `git update-ref <revision_ref> <最终提交>`；`revision_ref` 由交付输入给出（Cloud 生成，形如
   `refs/ora/revisions/<runId>`），Node 只接受 `refs/ora/revisions/` 下的名字。

- 真实 index、工作区文件与当前分支都不变；`commit-tree` 不运行任何钩子，所以项目的 pre-commit、
  commit-msg 失败不会让成果无法保存。快照不代表项目校验通过。
- 第一版只保存 HEAD 所在的提交链；Agent 另建的分支、stash 与 tag 不在 Revision 中。
- 子模块只记录 gitlink，不收录子模块工作区内容；LFS 指针按普通文件收录，不上传 LFS 实体。

### D3：累计增量 bundle

- 最终提交等于基础提交（clone 结果的 commit）时结果为 `unchanged`，不生成 bundle。
- 否则 `git bundle create <tmp> <revision_ref> ^<base_commit>`，再 `git bundle verify`（以该
  checkout 为对照）。bundle 只含基础提交之后的对象，恢复时需要从用户远端取得基础提交。
- 每次交付都相对 clone 基础提交生成，不依赖任何更早的 bundle；Agent 若改写了基础提交之前的历史，bundle
  仍以基础提交为前提，恢复要求不变。

### D4：上传与摘要

- 会话 JSONL 在会话结束时已封口；Node 读取完整文件。bundle 与 JSONL 分别先在本地计算大小与 SHA-256，
  在首个 PUT 前按执行持久固定对象字节、最终提交、对象键、大小及 SHA-256，再以授权 `PUT` 上传。
  请求原样携带签名头，包括 `x-amz-checksum-sha256` 与 `If-None-Match: *`；对象只能首次创建。
- 首次 PUT 的回复丢失后，重复 PUT 的 412 只说明对象已存在。Node 重放原声明，由 Cloud 核对对象大小与摘要，
  不能只凭状态码推断成功。上传授权不持久化，恢复时重新请求。
- 授权缺失或过期时发送 `UploadGrantNeeded` 等待；单个对象上传失败重试 3 次，仍失败则
  `failed{upload_failed}`。
- 成功结果：`delivered{final_commit, base_commit, revision_ref, bundle{key, size, sha256}, history{key, size, sha256}}`
  或 `unchanged{final_commit, base_commit, revision_ref, history{...}}`。

### D5：失败码与重试

| 失败码 | 含义 |
|---|---|
| `session_not_settled` | 会话尚未结束或进程未确认清理 |
| `checkout_unavailable` | 找不到 clone 执行或 checkout 已损坏 |
| `snapshot_failed` | 快照命令失败 |
| `bundle_failed` | bundle 生成或校验失败 |
| `history_unavailable` | 会话 JSONL 不存在或无法读取 |
| `upload_failed` | 上传重试耗尽 |

首次上传前未持久固定内容的执行可以重新准备快照；首次上传后，Controller/Node 重启必须继续同一执行的
持久对象字节和元数据，不能重新生成带不同时间戳的快照或重新打包 bundle。原声明和对象内容保留至终态证据
持久化，此后保留可重放的终态证据直至 Cloud 接管。Controller 生命周期取消不生成 Node 交付失败终态。
需要改变交付内容时由 Cloud 新建交付工作项与执行，使用新对象键，不能覆盖原执行的首次对象。

## 不变量

1. 快照与交付不修改工作区文件、真实 index 或任何分支。
2. 交付开始时会话的进程 scope 已关闭。
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
- 仓库尚无任何提交时（没有基础提交）的处理。
- 交付后工作区是否需要保持 clean（交互式场景）。

## 落地

Node 实现交付执行（Git 命令沿用 clone 的受管执行与加固配置）；真实 Git 测试覆盖有/无改动、删除与新增、
忽略文件、提交钩子失败不影响快照、工作区与分支不变、bundle 可在只有基础提交的新仓库中 fetch 出最终提交；
上传以本地 HTTP 替身测试校验头与失败重试。
