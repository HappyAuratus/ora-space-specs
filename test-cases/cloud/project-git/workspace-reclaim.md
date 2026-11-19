# 工作目录回收：只回收能恢复或用户明确丢弃的已删对话目录、先登记后删除、精确归属、可重试、不动交付

依据 [删除对话时回收工作目录](../../../decisions/cloud/project-git/20261119-reclaim-deleted-conversation-workspace.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇决策；交付本身的承诺见 [回合交付](turn-delivery.md)。

实现由 circus #1196 的运行 #1319（cloud PR #419）承接，尚未合并，所以全部义务暂标 `Missing`；证据在落地运行按合并的
commit 登记。8 项汇总为 0 Covered / 0 Partial / 8 Missing。

## Only A Recoverable Workspace Is Reclaimed By Default

- 风险：默认的删除把运维也恢复不了的改动一起删掉：最后一轮交付之后又写入的文件、stash、没检出分支上的提交、子模块里
  没提交的改动；或者把还在交付、旧会话的目录当成能恢复。
- 前置状态：同一项目里若干对话，分别处于：最后一轮 `delivered` 且之后没有变化；最后一轮 `empty`；最后一轮交付失败；
  最近一条记录落在更早的回合上；交付 ADR 落地前的旧会话；第一轮还在跑；最后一轮交付之后又写了文件、只改了被忽略文件、
  只改了物化技能；有 stash、有不在起点与 `HEAD` 祖先里的本地 ref、有指向非提交对象的 ref、有改动过的 gitlink、有已
  初始化且有未提交改动的子模块、有 gitfile 形式的嵌套 `.git`；顶层 `.git` 是解析不出来的 gitfile。
- 触发：用户以默认模式（`reclaim`，或不带参数）删除这些对话。
- 必须成立：只有五条（D1）全部成立的目录被回收；其余目录保留，回收记录以 `kept` 结束并写明 D1 的原因（`delivering`
  除外，它保持 `pending`）；判定顺序先 `delivering` 再 `legacy`；只改了被忽略文件或物化技能的不判 `changed_after_delivery`；
  `.git` 解析不出来判 `unverifiable`；判定时磁盘、镜像、Git 临时失败记 `check_failed` 保持 `pending`，不判 `kept`。
- 禁止结果：上面任何一种不能恢复的目录被删除；临时失败被记成终态。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能恢复的两种（`delivered` 无变化、`empty`）被回收 | 回收后目录不存在，记录 `reclaimed` 并记下依据的交付 | Missing |
| 交付相关的不能恢复原因各一例都保留 | `delivering` 保持 `pending`；`legacy`、`last_turn_undelivered`、`changed_after_delivery` 以 `kept` 结束，目录不变 | Missing |
| Git 状态的各种情况都保留 | stash、越界 ref、非提交 ref、改动的 gitlink、有改动的子模块、gitfile 嵌套 `.git` 各判 `git_state`；解析不出来的 `.git` 判 `unverifiable` | Missing |
| 交付之后只改被忽略文件或物化技能不算变化 | 这两种情况判为能恢复并回收 | Missing |
| 临时失败可重试 | 第一次镜像不可读记 `pending`/`check_failed`，恢复后启动时回收 | Missing |

- 决策依据：D1、D3，不变量 2、3。

## Deleting Tells The User What Happens And Discard Requires The Previewed State

- 风险：用户在确认框里看到的和实际删除的不一致：预览之后改动又变了，`discard` 却仍然执行；`git_state` 时显示“没有
  未交付的改动”，用户以为丢弃什么都不会丢；预览读不到时界面承诺保留，实际却回收了。
- 前置状态：能恢复、未交付（含 `git_state`）、没有工作目录、目录已不存在的对话各一个；Cloud 有或没有 Agent 运行时。
- 触发：读预览；用预览给出的 `revision` 或过期的 `revision` 以 `discard` 删除；预览之后把一个文件同路径、同增删行数地
  改写；预览请求失败后在界面上删除。
- 必须成立：预览给出 `none`（`no_workspace` 或 `workspace_missing`）、`recoverable`（有被忽略文件时标出）、`undelivered`
  及原因、未交付改动列表、`git_state` 时的 stash 数量、ref 与提交数、嵌套仓库路径、最近一次 `delivered` 交付；
  `discard` 带回的 `revision` 与删除时（结束会话之前）重算的不一致、或会话有回合未结束时返回 `409 deletion_preview_stale`，
  对话、目录、回收记录都不动；内容改写也使 `revision` 改变；没有运行时时，有工作目录的对话的预览和 `discard` 在任何
  改动之前返回 `503 workspace_reclaim_unavailable`，默认删除照常并登记 `pending`/`reclaim_unavailable`；删除返回
  `reclaimed`（字节与条目数）、`kept`（原因）、`pending`（原因）或 `none`；界面在预览读不到时禁用删除、提供重试，
  能恢复时提供 patch 下载，`delivering` 时不提供丢弃。
- 禁止结果：过期的 `discard` 删掉了用户没看到的内容；没有运行时时登记了未经验证的 `discard`；确认框承诺了与结果不符的事。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 预览各种结果与字段 | 每种状态的预览内容，`git_state` 列出 stash、ref、嵌套仓库 | Missing |
| 过期或内容改写后的 `discard` 返回 409 且什么都不动 | 409 后对话仍在、目录不变、没有回收记录；用新 `revision` 成功 | Missing |
| 没有运行时时不登记未经验证的丢弃 | 预览与 `discard` 返回 503 且什么都不动；默认删除登记 `pending` | Missing |
| 界面不承诺与结果不符的事 | 预览失败禁用删除并可重试；下载入口；被忽略文件提示；`delivering` 不给丢弃 | Missing |

- 决策依据：D2，不变量 3。

## Every Reclaim Is Registered In The Deleting Transaction And Nothing Unregistered Is Removed

- 风险：先删对话记录、事后再扫目录找没有主人的；对话首轮失败的回滚被当成用户删除去回收；模式在登记后被改。
- 前置状态：一个有工作目录的对话；一个首轮被拒、正在回滚的新对话；一个没有工作目录的对话。
- 触发：用户删除第一个和第三个对话；第二个对话的创建失败回滚；登记后尝试修改回收的模式。
- 必须成立：删除对话记录的同一个事务里写入回收登记（工作目录身份、项目、会话、来源对话、路径、`clone_execution`、模式）；
  没有工作目录的不登记；回滚不登记；模式不可修改；Cloud 只删登记过的目录。
- 禁止结果：对话已删除却没有登记；没有登记的目录被删除；回滚产生登记。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 登记与删除对话在同一事务 | 事务中途失败时对话与登记都不存在或都存在；登记字段完整 | Missing |
| 回滚与无工作目录不登记 | 首轮失败回滚后没有登记；无工作目录的对话删除后没有登记 | Missing |
| 模式不可变 | 修改已登记回收的模式被拒绝 | Missing |

- 决策依据：D3，不变量 4。

## Only The Exact Workspace Of The Deleted Conversation Is Removed

- 风险：删到活跃会话、别的用户、工作流运行、任务运行的目录；登记的路径或它的某一级父目录被换成链接，删到项目之外；
  登记与 `v2_workspaces` 不一致时仍按登记删。
- 前置状态：项目里有活跃会话的目录、别的用户对话的目录、工作流运行共享的目录、任务运行的目录；一个待回收的目录，其
  路径越界、本身或某一级父目录是符号链接，或在检查之后、打开之前被移走并换成指向原处的链接；目录里有指向项目之外的
  链接；登记的路径或 `clone_execution` 与 `v2_workspaces` 不一致，或有别的行用同一路径。
- 触发：执行这些回收。
- 必须成立：上述身份或路径检查不通过的都以 `kept` 结束并记原因，目录和项目外文件都不变；目录里指向外部的链接只删除
  链接本身；只删除登记的那个 checkout。
- 禁止结果：任何不属于登记对话的目录或项目之外的文件被删除。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 活跃、他人、共享、任务目录不删 | 这些目录在回收后不变，回收记录 `kept` 或不存在 | Missing |
| 路径越界与符号链接不跟随 | 越界、checkout 或父目录是链接、检查后被换成链接都 `kept`，项目外文件不变；内部外链只删链接 | Missing |
| 登记与 `v2_workspaces` 核对 | 路径或 `clone_execution` 不一致、同路径另有行时 `kept` | Missing |

- 决策依据：D4，不变量 1、5。

## Reclaim Does Not Block The Project And Runs Once Per Workspace

- 风险：删除大目录期间整个项目的会话都启动不了；同一个目录被两次回收同时删除；回收与交付、技能投影互相破坏。
- 前置状态：一个大目录待回收，删除被拖慢；同项目有新会话要启动、技能要投影；同一条回收被删除请求和启动同时触发。
- 触发：执行回收的同时启动同项目会话、投影技能、对同一条回收再触发一次。
- 必须成立：回收期间同项目的会话准入、`attachSkills`、`attachUserSkills`、`ApplySkill` 不被阻塞；投影结果不含被回收的
  checkout，也不因它消失而报错（`ENOENT` 跳过）；项目 runtime 不在时，回收持项目锁期间新 runtime 的创建等待；同一条回收
  只删除一次；回收与同目录的交付串行，交付记录不变。
- 禁止结果：会话准入在回收期间卡住；投影失败；同一目录被删除两次或统计重复。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 回收不阻塞同项目会话与投影 | 删除被阻塞期间会话准入与投影照常完成，投影不含该 checkout | Missing |
| 项目 runtime 不在时与创建互斥 | 持项目锁期间新 runtime 的创建等待，释放后继续 | Missing |
| 同一条回收只执行一次 | 并发触发只有一次删除，记录与统计一致 | Missing |

- 决策依据：D4、D5，不变量 5、6。

## Interrupted Or Failed Reclaims Resume At Boot Without Re-judging

- 风险：删到一半崩溃后目录成了没有主人的残留；重启后重新判定，删了一半的目录因为“变了”落到 `kept`；删除请求超时后
  要等几周后的重启才继续；事务提交后的错误被报成“对话没删除”。
- 前置状态：登记后、删除前崩溃的回收；删到一半崩溃（`deleting`）的回收；IO 失败的回收；一个大到短时限内删不完的目录。
- 触发：重启 Cloud；删除请求在时限内删不完；事务提交后回收出错。
- 必须成立：启动补做发生在 `RecoverTurnDeliveries` 之后、恢复工作流和开始服务之前；`pending` 按 D1 判定，`deleting`
  只核对身份与路径就删完；IO 失败保持原状态、尝试次数加一、记最后一次错误；删除请求在时限内删不完返回
  `pending`/`in_progress` 并在后台删完，关机时留 `deleting`；事务提交后的错误返回 200 和回收状态，对话确已删除。
- 禁止结果：`deleting` 落到 `kept`；未完成的回收在启动时被遗漏；重试删到别的目录；对话已删除却返回失败。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 启动位置与补做 | 启动顺序在交付恢复之后、服务之前；`pending` 与 `deleting` 都被处理 | Missing |
| `deleting` 不重新判定 | 删到一半后目录内容变化，重启后仍删完 | Missing |
| 失败可重试、计数与日志 | IO 失败保持原状态、计数加一，恢复后启动时回收 | Missing |
| 删除请求的执行不随请求取消 | 大目录返回 `pending`/`in_progress` 后后台删完；关机后启动补完；提交后出错仍返回 200 | Missing |

- 决策依据：D3，不变量 6。

## Reclaim Leaves Deliveries And Their Retention Unchanged

- 风险：回收顺带删了交付记录、交付义务、交付 ref、快照 ref 或 `v2_workspaces`；回收之后运维无法恢复成果；任务运行和
  工作流运行的交付读取受影响。
- 前置状态：一个能恢复的对话，项目镜像里有它的交付 ref；同项目有任务运行和工作流运行的交付。
- 触发：回收这个对话的目录；运维按 D6 从镜像恢复。
- 必须成立：回收前后交付记录、交付义务、交付 ref、快照 ref、`v2_workspaces` 不变；回收记录里的交付提交与起点提交能在
  镜像上打出完整 patch，并能在起点上应用；任务运行与工作流运行的变更视图和 patch 读取不变。本用例与
  [回合交付](turn-delivery.md#the-change-view-and-patch-read-the-delivery-not-the-working-directory) 的读取承诺一起成立。
- 禁止结果：回收改动了任何交付数据；回收后成果在镜像里找不到。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 交付数据不变 | 回收前后交付记录、义务、ref、`v2_workspaces` 逐项一致 | Missing |
| 运维能从镜像恢复 | 用记录里的提交打出的 patch 能在起点上应用 | Missing |
| 其他来源的交付读取不受影响 | 回收后任务运行、工作流运行的变更视图与 patch 照常 | Missing |

- 决策依据：D6，不变量 7；[回合交付](../../../decisions/cloud/project-git/20261112-single-machine-turn-delivery.md) D4、D6。

## Operators Can List Reclaims And Discard Kept Workspaces Offline

- 风险：保留下来的目录和反复失败的回收没人看得到；离线丢弃在服务运行时执行，与服务争用目录；丢弃绕过 D4 的检查。
- 前置状态：有 `kept`、`pending`、`deleting`、`reclaimed` 的回收记录；服务在运行或已停止。
- 触发：运维运行列出子命令；在服务运行时和停止后分别运行离线丢弃。
- 必须成立：服务运行时能列出全部回收记录与原因、尝试次数、最后一次错误、释放量和依据的交付；数据目录锁被占用时离线
  丢弃拒绝执行；停服后离线丢弃只作用于 `kept` 的记录，按 D4 检查后删除，记为 `reclaimed` 并写明是运维丢弃。
- 禁止结果：服务运行时离线丢弃执行了删除；丢弃作用于非 `kept` 的记录或跳过路径检查。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 运行时可列出 | 服务运行时列出子命令输出全部记录与字段 | Missing |
| 离线丢弃与服务互斥 | 数据目录锁被占用时拒绝 | Missing |
| 离线丢弃按 D4 检查并留痕 | 只处理 `kept`，删除后记 `reclaimed` 与运维原因 | Missing |

- 决策依据：D6，不变量 4、5、6。
