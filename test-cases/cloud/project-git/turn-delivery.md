# 回合交付：每轮的交付记录、相对起点的快照、只新建的交付 ref、交付后才进待验收

依据 [回合交付](../../../decisions/cloud/project-git/20261112-single-machine-turn-delivery.md)（`approved`）。下文的 D 编号与
不变量编号都指这篇决策。

证据读取的是 cloud `0ca0a9a0` 的 `v2-server/`（路径相对该目录）。v2-server 还没有回合交付，变更视图读的是工作目录，
所以除了 Node 不能经本机服务推送这一条，其余都是 Missing，由 circus #1108 实现。

## Every Turn End Produces Exactly One Delivery Record

- 风险：某些回合没有交付，改动随工作目录一起丢失；同一轮被重复交付，出现两条互相矛盾的记录。
- 前置状态：一个对话、一个任务运行和一个带两个 Agent 节点的工作流运行，都在同一个项目里。
- 触发：三种来源各结束一轮，分别以成功、失败、被停止结束；工作流的两个节点依次结束；交付过程中重复收到同一轮的结束通知。
- 必须成立：每一轮结束恰好一条回合交付记录，结果是 `delivered`、`empty` 或 `failed`；工作流每个节点尝试一条，交付的是
  运行共用的工作目录；同一工作目录的两次交付按顺序进行。
- 禁止结果：失败或被停止的回合没有交付记录；同一轮有两条记录。
- 验证义务与证据：**Missing** — 由 #1108 实现。
- 决策依据：D1，不变量 1。

## A Delivery Is A Full Snapshot Against The Session Base And Leaves The Agent's Repository Untouched

- 风险：交付漏掉新文件、带进 Ora 物化的技能；父提交依赖上一次交付，中间一次失败后后面的交付无法单独使用；交付过程改动了
  Agent 的索引、分支或工作目录。
- 前置状态：会话起点提交为 B；Agent 改了已跟踪文件、新建了未跟踪文件、自己提交过一次又 `reset` 回去；工作目录里有 Ora
  物化的技能目录和被 `.gitignore` 忽略的文件。
- 触发：这一轮结束，做交付；随后再做一轮，第一轮的交付之后有一次交付失败。
- 必须成立：交付提交的父提交是 B，树包含所有未被忽略的改动和新文件，不含物化的技能和被忽略的文件；Agent 的索引、`HEAD`、
  分支和工作目录与交付前完全一致；树与 B 相同时记为 `empty` 且不写 ref；超过大小上限时记为 `failed` 并写明“改动过大”；
  任何一次交付都能单独在 B 上打出完整 patch。
- 禁止结果：交付提交以上一次交付或 Agent 的 `HEAD` 为父；物化的技能进入交付；交付改变 Agent 的 Git 状态。
- 验证义务与证据：**Missing** — 由 #1108 实现。
- 决策依据：D2，不变量 3、4。

## A Delivery Ref Is Written Before Its Record And Never Rewritten

- 风险：记录说已交付但 ref 不存在；交付 ref 被改写或删除，验收看到的不是当时的成果；同步时把交付 ref 清掉或覆盖。
- 前置状态：项目镜像里已有快照 ref 和若干交付 ref。
- 触发：写 ref 之后、写记录之前让 Cloud 崩溃；对同一个交付 ID 再写一次；对项目做一次手动同步，包括外部分支被强推的情况。
- 必须成立：结果为 `delivered` 的记录对应的 ref 一定存在并指向记录里的提交；崩溃只留下没有记录引用的 ref，它不出现在
  任何视图里；已存在的交付 ref 不能被改写；同步前后所有交付 ref 不变。
- 禁止结果：有记录无 ref；交付 ref 被改写、删除，或被同步动过。
- 验证义务与证据：**Missing** — 由 #1108 实现（同步由 #1057 实现，二者都要覆盖）。
- 决策依据：D3，不变量 2、3、5。

## A Task Reaches Review Only After Its Turn Is Delivered

- 风险：任务进了待验收，验收的人却看不到改动；交付失败的任务停在进行中或误进待验收；交付期间有人手动推进任务。
- 前置状态：一个指派给 Agent 的任务，处于进行中。
- 触发：这一轮成功结束、交付成功；成功结束但交付失败；没有改动（`empty`）；这一轮被停止；交付进行中有人手动推进任务。
- 必须成立：交付为 `delivered` 或 `empty` 后任务才进入待验收；交付失败时任务回到待处理并记下原因；被停止时照常交付、
  任务回到待处理；交付完成前人工推进返回 `409 run_active`。对话交付失败时会话不结束；工作流节点交付失败时节点和运行
  结果不变，运行页显示失败原因。
- 禁止结果：交付完成之前任务进入待验收；交付失败改变工作流节点的结果。
- 验证义务与证据：**Missing** — 现在任务在一轮结束时直接进入待验收（`internal/store/tasks.go` 的 `CanRunTransitionTask`），
  由 #1108 改为等交付。
- 决策依据：D5，不变量 6。

## The Change View And Patch Read The Delivery, Not The Working Directory

- 风险：工作目录被清理后改动就看不到了；patch 无法在起点上应用；同团队的人读到别人的对话交付。
- 前置状态：一个任务运行和一个对话各有两次交付，之后工作目录被 `-cleanup` 删除；一个回合正在进行；另有一次交付失败。
- 触发：读取变更视图、单文件 diff 和整份 patch；浏览文件；由对话主人以外的成员读取对话的交付；超过下载上限的 patch；
  删除对话后再读取它的交付。
- 必须成立：变更视图显示最近一次 `delivered` 或 `empty` 交付相对起点的改动，回合进行中显示上一次交付并标明本轮进行中；
  工作目录被清理后变更视图和 patch 仍可用，浏览文件返回 `410 workspace_removed`；patch 包含新文件和二进制文件，能在
  起点提交上 `git apply`；对话的交付只有主人能读，任务和工作流运行的交付团队成员都能读；超过下载上限时提示且交付保留；
  删除来源后它的交付不可读。
- 禁止结果：变更视图读工作目录；工作目录被清理后变更视图失效；越权读到对话的交付。
- 验证义务与证据：**Missing** — 现在变更视图读工作目录（`internal/workspacefs`），由 #1108 改为读交付。
- 决策依据：D4、D6，不变量 7。

## Cloud Restart Delivers Interrupted Turns Before Serving

- 风险：重启时正在跑的回合的改动随后被清理掉；补交付变成重做执行；补交付还没完成就开始服务，状态互相覆盖。
- 前置状态：Cloud 重启前有一个进行中的任务运行和一个进行中的对话回合，工作目录都还在；另一个被中断回合的工作目录已经不在。
- 触发：Cloud 重启。
- 必须成立：启动时先把未结束的执行标为中断（A1 D3），再为工作目录还在的被中断回合各做一次交付，然后才开始服务请求；
  工作目录不在的记为 `failed`；任务回到待处理；不重新执行任何回合。
- 禁止结果：重启后重新执行被中断的回合；开始服务之后才补交付；补交付把任务推进到待验收。
- 验证义务与证据：**Missing** — 由 #1108 实现。
- 决策依据：D5，不变量 1、6。

## Nodes And Agents Cannot Write Refs Through Any Cloud Entry Point

- 风险：Agent 经 Cloud 的 Git 入口推送，伪造或覆盖交付 ref 和快照 ref。
- 前置状态：一个项目的 Node 持有有效的本机服务凭据。
- 触发：Node 经本机服务发起 `git-receive-pack`；经其他 Cloud 接口尝试写入镜像的 ref。
- 必须成立：本机服务只放行只读的 upload-pack，推送被拒绝；Cloud 没有任何让 Node 或 Agent 写入镜像 ref 的接口。第一版不隔离
  （A1 D2），Agent 在磁盘上直接改动镜像不在这条承诺之内。
- 禁止结果：经 Cloud 的入口推送成功。
- 验证义务与证据：**Partial** — `internal/nodeservices/service_test.go` 的 `TestServicesRequireIdentityAdmissionAndLiveGrant`
  检查了 `git-receive-pack` 返回 403。仍缺：交付上线后，没有测试确认新增的交付相关接口不提供写入 ref 的路径。
- 决策依据：D3，不变量 8。
