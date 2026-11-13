---
status: implemented
---

# Node 直接执行自己的 Git：进程组终止，尝试锁给出停止证据，重启先收口，取代 host／guardian

Node 自己执行的 Git（clone、续作恢复、Revision 交付和 `ora-git`）不再经过 host 和 guardian，改由 Node 用
`ora-process` 直接启动。每条有副作用的命令在派发前拿到一把**尝试锁**：一个只有 Node 能打开的锁文件，git 和它的
所有后代继承这把锁。之后只要还能重新加上锁，就说明原命令那棵进程树已经没有进程在运行，这就是业务层需要的
**停止证据**。终止残留进程是另一件事，由进程组、`ora-reaper` 和派发后记下的终止线索负责；终止手段全部失效时，
尝试停在未知，不会被当成已停止。Node 重启后先对未收尾的尝试逐条收口，再进入现有的业务恢复。

行为上的变化：

- 每次 clone 不再留下常驻进程，也没有经 host 轮询带来的固定等待；Node 只多一个常驻的 `ora-reaper`。实测一次
  clone 的 Git 准备时间中位数从 13.0 秒降到 1.0 秒（同一台 4 核 arm64 机器、同一项目，release 构建）。
- Node 重启后，原命令已停止但拿不到退出状态的 clone 以 `Interrupted` 失败结束；续作恢复在同样的证据下视为已
  确认停止。原先"host 说从未启动"和"host 查不到记录"的区分不再存在。
- 保证等级仍是 BestEffort，会逃过保证的情况见 D8。

本文为 `implemented`：Linux 部分已在 desktop 实现并验证，host、guardian、helper 和它们背后的 process 代码已经
删除；最终实现没有偏离评审通过的决策。核心用例和每条义务的证据状态登记在
[Node 直接执行 Git 的核心用例](../../../test-cases/node/git-execution/attempt-locks-and-stop-evidence.md)，
没有直接证据的义务在那里如实标为 Partial 或 Missing。

- **涉及**：desktop 的 `apps/ora-node`（`managed/` 一层和 Node 数据库）、`crates/process`（`ora-process`）、
  `apps/ora-reaper`；Node 的部署清单带上 `ora-reaper`，去掉 host 和 guardian。
- **不涉及**：gitlancer 的 `GitRunner` 接口，clone、续作恢复和交付的业务状态机，Node 协议，Agent 和插件进程的
  启动方式。
- **取代**：[`node/process`](../process/README.md) 下 architecture、integration、control、lifecycle、io、
  protocol、persistence、recovery、upgrade 九组决策整份被本文取代（D9）。Node 自己的 Git 是 host／guardian
  唯一的调用方，它不再经过 host 之后，这套体系没有调用方。
- **不承担的迁移**：不保留 host 客户端，不提供新旧执行方式并存的阶段；旧 host 目录里的遗留尝试按 D7 一次性收口。
- **平台**：只实现了 Linux。ora-node 目前只有 Linux 版；macOS 和 Windows 的对应机制和保证等级见 D6，等 Node
  支持这些平台时再落地。

## 背景

被取代的 host／guardian 体系为每条 Git 命令提供持久的运行身份、独立于 Node 的存续和收尾证据。它的代价在实际
使用中显现：

- 一次 clone 执行 16 条 Git 命令，每条各开一个 Scope，留下 16 个常驻 guardian，没有退休回收；
- 每条命令经 Node、host、guardian 三段轮询，一次 clone 的固定等待约 12 秒（debug 构建，4 核 arm64）；
- 一个空闲 Node 约占 7% 个核和 145 MB 内存，随使用量线性增长；
- process 相关代码约 1.1 万行，唯一的调用方是 Node 自己的 Git。

业务层对进程层的依赖只有一条：**原来那条命令已经停了**。clone 不需要 Git 跨 Node 重启继续运行
（[clone 中断判定](../repository/20260925-terminated-clone-attempt-fails-as-interrupted.md) D3 规定 Git 与 Node
同生共死），续作恢复要的是"停止确认"而不是"恰好执行一次"
（[恢复截止与停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md) D6），交付被打断后
从头准备。持久运行身份、输出保留和独立存续都没有调用方在用。

所以问题收窄为：去掉 host／guardian 之后，Node 怎么在自己被强杀、重启之后仍然拿到"原命令已停止"的证据。

约束：

1. 配置了独立工作负载用户的部署里，Node 以 root 运行，工作目录上的 git 降权到工作负载用户执行。容器里的 root
   默认没有 `CAP_SYS_PTRACE`，读不了另一个用户进程的 `/proc/<pid>/environ`，所以不能靠环境变量标记找进程。
2. Node 重启后，原来的 git 不再是它的子进程，拿不到退出状态。
3. Node 是多线程进程，同时还在启动 Agent 会话和别的 git；交给某条命令的东西不能泄漏给其他子进程。
4. clone 的 Git 命令在受理线程之外执行，那里不持有 Node 数据库连接
   （[clone 的 Git 副作用移出受理线程](../repository/20260925-clone-git-effects-run-outside-the-admission-worker.md)），
   所以停止证据不能依赖派发之后再写数据库。

## 决策

### D1：Node 用 `ora-process` 直接启动 Git，降权在进程内完成

- **执行入口**：`ManagedGitRunner` 直接启动 git，不再有 host、guardian、Scope 和 Run。gitlancer 的 `GitRunner`
  接口不变，clone、续作恢复和交付的状态机不变；改动集中在 Node 的 `managed/` 一层。
- **每条命令一个进程组**：git 是进程组长，Node 终止时对整个进程组发 `SIGKILL`。Node 启动 Agent 会话用的是
  `ora-process` 的同一种能力。
- **降权**：配置了工作负载用户时，在 fork 之后、exec 之前切换身份（`ProcessIdentity::Linux`：清空附加组、
  `setresgid`、`setresuid`、`no_new_privs`、清空能力集），不再套一层 `/usr/bin/setpriv`。少一层 exec，尝试锁的
  fd 只在一个地方处理，Node 镜像也不再依赖 `setpriv`。
- **`ora-reaper`**：Node 启动时拉起一个 `ora-reaper`。Node 启动的每条 git 都向它登记进程组，直接子进程退出时
  取消登记；Node 死亡时 reaper 从存活管道感知，杀掉仍登记着的进程组。
  - reaper 拉不起来，Node 启动失败。reaper 在运行中死亡后，Node 不再启动新的 git，以明确错误退出，由部署重启；
    不带着失效的 reaper 继续运行。
  - Agent 会话进程不登记，行为和现在一样。Agent 和插件的孤儿后代回收不在本文范围，以后要做时打开登记即可。
- **不用 `PR_SET_PDEATHSIG`**：它在父线程退出时触发而不是父进程，只作用于直接子进程，切换用户时会被内核清掉，
  而降权执行正好要切换用户。

### D2：尝试锁是唯一的停止证据，终止线索只用来杀进程

把"证明已停止"和"找到并杀掉"分成两件事。前者只有一个依据，后者可以有多种手段，任何一种失效都不影响前者的结论。

**尝试锁**

- 每条有副作用的 Git 命令有一个锁文件，放在 Node 数据目录下的私有目录里。目录权限 0700，文件权限 0600，属主是
  Node 的运行用户；配置了独立工作负载用户的部署里，这个用户是 root，工作负载用户（Agent 也是这个用户）打不开它。
- 锁文件必须在本地文件系统上。网络文件系统的 `flock` 语义不可靠，不能用来放锁。
- 锁文件里写着这条命令的工作目录。没有数据库记录的锁被占住时，Node 靠它知道旧命令可能还在写哪个目录（D4）。
- 有数据库记录的尝试，锁文件和锁目录在写记录之前刷盘：整机断电后记录还在，锁文件也必须还在，否则这条尝试会
  因为"锁文件不存在"永远停在未知。没有数据库记录的锁不刷盘，丢了只是从头准备。
- 顺序固定为：**建锁文件 → 在 Node 数据库记下尝试 → 加锁并派发**。
  1. 记录尝试之前，锁文件已经存在。
  2. 派发时 Node 打开锁文件，加独占 `flock`，然后启动 git。
  3. git 继承这个 fd，它的所有后代默认也继承。fd 继承不受文件权限影响，降权后的 git 照样拿得到。
- 这个 fd 在 Node 进程里始终带 `CLOEXEC`，只在目标子进程 exec 之前清掉。Node 并发启动的 Agent 和其他 git 拿不到
  它。`ora-process` 的降权步骤给 3 号以上的 fd 全部设 `CLOEXEC`，只保留启动方指明的那一个。

**停止证据**

Node 关闭自己手里的那份 fd，重新打开锁文件，尝试非阻塞地加独占锁：

| 结果 | 含义 | 结论 |
|---|---|---|
| 加上了 | 没有任何进程还持有原来的 fd | 原命令已停止（或从未启动） |
| 被占用 | 原进程树里还有进程活着 | 未停止 |
| 锁文件不存在，或打开、加锁出错 | 证据缺失 | 未知 |

为什么这是可靠的：`flock` 的锁属于打开的文件描述，不属于进程；只要还有一个进程持有继承来的 fd，锁就在，进程
全部结束时由内核释放。它不读 `/proc`，不受用户身份限制；后代换了进程组或会话也一样覆盖；Node 在任何时刻崩溃，
锁的状态都还是对的，所以不需要在派发之后补写任何记录，也不需要让子进程等 Node 写完记录再 exec。

"从未启动"和"已经停了"不区分：两种情况下都没有进程在写，业务层要的就是这个。锁文件不存在则不同：按上面的顺序，
有尝试记录就一定有锁文件，缺失说明记录不完整，只能按未知处理。

**终止线索**

- 启动成功后，Node 把直接子进程的 pid、进程启动时间和本机启动标识写在锁文件旁边，不刷盘。它只在 Node 重启后
  用来杀进程（D4），写丢了不影响停止证据。进程崩溃不会丢页缓存；整机断电时 git 也不在了。
- 按线索杀进程前核对身份：这个 pid 当前存在但启动时间对不上，说明 pid 已被别的进程复用，不杀；pid 当前不存在
  时，进程组里剩下的成员只可能是原来那批（Linux 不会把仍被用作进程组号的 pid 分配给新进程），可以整组终止。
  本机启动标识不一致说明机器重启过，不杀。

**关闭 git 的后台维护**

Node 执行的每条 git 命令都带 `-c gc.auto=0 -c maintenance.auto=false`，关闭后台 gc 和 maintenance。否则 fetch
之后 git 会脱离出去做 gc，它继承了锁，会把停止确认拖到 gc 结束，而这部分工作没有人认领。

**哪些命令要锁**

| 命令 | 尝试锁 | Node 数据库记录 |
|---|---|---|
| clone 的工作目录命令 | 要 | 记（现有） |
| clone 的传输仓库命令 | 要 | 不记：被打断后清掉传输仓库重新 fetch（现有） |
| 续作恢复的命令 | 要 | 记（现有） |
| Revision 交付、`ora-git` 的 fetch 和 push | 要 | 不记：被打断后从头准备（现有） |
| 只读查询（clone 验收、preflight） | 不要 | 不记 |

交付类命令和 clone 的传输仓库命令不记数据库，但仍然要锁：Node 被强杀后旧的 push 或 fetch 可能还在跑，重启后
从头准备之前必须先确认它停了（D4）。
只读命令的残留不会写入，不需要停止证据；它们同样在自己的进程组里运行并向 reaper 登记。

### D3：运行中的输出、期限和收尾

- **输入输出**：stdin 关闭；stdout 和 stderr 并发读入有界内存，超过上限按失败处理；调用方不需要输出时直接丢弃。
  上限沿用现有的值。输出不持久化，Node 重启后不可取回。
- **期限**：沿用现有的命令期限、网络期限、续作恢复的绝对截止和正常停止的宽限。期限到期或 Node 正常停止时，
  先整组终止，再走下面的收尾。
- **收尾**：直接子进程结束后，Node 再对进程组发一次终止，然后探测停止证据，在清理期限内重试。收尾的结果：

| 观察 | 结果 |
|---|---|
| 以退出码结束，停止证据成立 | 已退出（退出码） |
| 被信号终止，停止证据成立 | 已终止（信号） |
| 清理期限内停止证据不成立，或探测出错 | 未核实：按未知处理，不释放资源 |

- 取得停止证据并把结果交给业务层落盘之后，才删除锁文件和终止线索。
- 直接子进程以 0 退出但还有后代拿着锁时，结果不是成功：要等停止证据成立，才把退出码交给业务层。
- 没有数据库记录的命令收尾为未核实时，它的锁和上一次运行留下的锁一样处理：占用期间不开始新的同类命令（D4）。

### D4：Node 重启后先收口，再恢复

Node 启动时，在进入任何业务恢复（clone 恢复、续作恢复的交接、交付）之前，先处理上一次运行留下的尝试锁：

| 情况 | 处理 | 结果 |
|---|---|---|
| 有终止线索，身份核对一致 | 整组终止，再探测 | 证据成立：已停止，没有退出状态 |
| 没有终止线索，或身份核对不一致 | 不杀，直接探测 | 证据成立：已停止，没有退出状态 |
| 探测为被占用 | 在清理期限内重试 | 仍被占用：未知，保留锁文件，之后的查询和下次启动继续探测 |
| 数据库里有未收尾的尝试，但锁文件不存在 | 不处理 | 未知 |
| 锁文件没有对应的未收尾尝试，且已空闲 | 删除 | 无 |

- **有数据库记录的尝试**（clone、续作恢复）：启动时先按线索终止它的进程组；停止证据由这次执行自己的恢复来取，
  取到之前不读目标目录、不重新派发。结果交给 D5 的业务判定。未知的尝试继续占着它的目录和输入，不影响不冲突的
  其他工作。
- **没有数据库记录的锁**（交付、`ora-git`、clone 的传输仓库）：启动时就在清理期限内终止并探测，空闲的直接删除。
  锁被占用期间，Node 不开始新的交付、`ora-git`
  写入和传输仓库命令，继续按线索终止并重试探测。这种情况要求 Node 和 reaper 同时死亡且按线索终止失败，属于 D8
  列出的边界。
  - 被占用的锁属于某个 clone 自己的传输仓库目录时，这个 clone 保持 Unknown：不清理该目录，不重新 fetch。
  - 被占用的锁来自别处时，clone 等到下一轮再试，这次等待不计入它的 fetch 次数。
- 取得数据库锁、reaper 已退出、等待超时，都不能代替停止证据。

### D5：各业务怎么用停止证据

**clone**

[clone 中断判定](../repository/20260925-terminated-clone-attempt-fails-as-interrupted.md) D1 的三类归类保留，
证据来源从"host 对原 Run 的观察"改为 D3、D4 的结果，并增加一行：

| 结束事实 | 归类 | 后续 |
|---|---|---|
| 以退出码结束，停止证据成立 | 已退出 | 不变：0 时核实仓库并完成成功，非 0 为失败 |
| 被信号终止，停止证据成立 | 已终止 | 不变：`Interrupted` |
| **已停止，没有退出状态**（Node 重启后收口得到） | 已终止 | **新增**：`Interrupted`，保留目录，只允许新执行重试 |
| 停止证据不成立或未知 | 未核实 | 不变：保留 Unknown 与预留 |

- 为什么新增这一行：Node 重启后原 git 没有父进程回收，退出状态一定拿不到。照原表它属于"退出结果未知"，所有
  被 Node 重启打断的 clone 都会停在 Unknown，退回该决策当初修掉的问题。
- 为什么判 `Interrupted` 是安全的：停止证据成立保证旧尝试不可能再写入；没有退出状态说明 Node 没有拿到 git 自己
  的结论，不能报告成功。即使 git 实际上已经成功退出，判为 `Interrupted` 也只是多一次重试，目录保留，不丢数据。
- 原决策留下的"尚未启动且收尾完成"一并归入这一行：停止证据不区分从未启动和已经停止。
- Node 数据库：原决策要求 `Interrupted` 只能由已记录的终止信号支撑。新增第三种结束事实"已停止，没有退出状态"，
  与退出码、终止信号互斥，同样可以支撑 `Interrupted`，不能支撑成功或 Git 失败。迁移是纯增量，不改写已有行。

**续作恢复**

- [恢复截止与停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md) D6 的停止确认，
  证据改为：该命令的停止证据成立。执行预算、停止请求和停止确认仍是三件事；停止证据上限、`stop_unconfirmed`、
  迟到证据补报、证据未知时保留输入和冲突责任，全部不变。
- [`NotStarted` 停止确认](../revision/20261021-stop-confirmation-accepts-not-started.md) D1、D2 的区分被替换：
  - 原"host 权威报告 `NotStarted`" → 停止证据成立即可，不再区分是否启动过；
  - 原"`UnknownIdentity` 不算停止确认" → 锁文件不存在不算停止确认。两者是同一个意思：记录不完整时不能把
    "查不到"当成"没在跑"。
- 向上报告的实际纳管保证仍是 BestEffort。

**交付和 `ora-git`**

被打断后从头准备，push 继续用 `--force-with-lease`，都不变。新增的只有 D4 的屏障：旧命令的锁释放之前不开始新的。

### D6：平台和部署形态

只有 Linux 一列已经实现。其他平台写明对应机制，Node 支持该平台时按下表落地，不需要重新决策；做不到的写明降到
什么程度。

| | Linux | macOS | Windows |
|---|---|---|---|
| 整组终止 | `kill(-pgid)`，`ora-process` 已有 | 同 Linux | Job Object，`ora-process` 已有 |
| 停止证据 | 继承的 `flock` | 同 Linux | 继承的独占句柄：重新以独占方式打开成功即已停止 |
| 按线索核对身份 | pid 加启动时间 | pid 加启动时间 | 不需要：命名 Job Object 本身就是身份 |
| `ora-reaper` | 可用 | 可用（同一份 Unix 实现） | 可用（Job 实现），Node 场景落地时验证 |
| 保证等级 | BestEffort | BestEffort | BestEffort；Job 不允许脱离时接近 Strong，落地时另行确认 |
| 是否已实现 | 是 | 否 | 否 |

和部署形态的关系：

- [Cloud 第一版](../../cloud/lifecycle/0-single-process-cloud-on-enterprise-intranet.md) D2 里，Node 是 Cloud 的
  本地子进程，和 Cloud 同一个系统用户，没有独立的工作负载用户。这条路径上 Node 被杀后没有别的东西替它清理，
  reaper 是主要的兜底；该决策 D3 要求"确认旧 Node 已停止之前不重建"，本文 D7 的部署前提依赖它。
- Node 运行在容器或独立的 pid 命名空间里时，Node 退出后容器结束，里面的进程被一起清掉，比 reaper 更强。本文
  不依赖这一点，也不与它冲突：容器重启后所有锁都是空闲的，收口直接成立。

### D7：升级和部署

- **部署前提**：新 Node 启动之前，旧的 Node、host、guardian 进程树已经结束。Cloud 本地子进程路径在重建 Node 前
  先杀掉旧进程树；容器部署重启容器。
- **旧 host 的遗留尝试**：新 Node 首次启动时，把 Node 数据库里按旧方式记录、尚未收尾的尝试按"已停止，没有退出
  状态"收口：clone 记为 `Interrupted`，续作恢复视为已确认停止。然后删除 host 目录。不要求人工清理，不拒绝启动。
  依据是部署前提：旧进程树已经结束，就不存在仍在写入的旧命令。
  - 收口由 Node 数据库迁移完成，不读 host 目录里的内容。
  - 只删除认得出是 host 留下的目录，并且只在 host 自己的锁空闲时删除；host 仍持有锁或删除失败时保留目录并
    告警，Node 照常启动。这只影响磁盘占用。
  - Node 的数据目录不能放在 host 目录里，否则拒绝启动：那个目录会被删除。
- **Node 数据库迁移**：新增 D5 的结束事实；尝试记录不再保存 host 的运行意图。迁移失败时拒绝打开，不重建数据库。
- **配置**：新增可选的 reaper 程序路径 `reaper_program`，不写时用 Node 可执行文件同目录下的 `ora-reaper`。
  两个旧配置项保留为兼容字段，都不再必填，按旧格式写的配置不用改就能加载：
  - `host_directory`：只用来找到并删除旧的 host 目录；
  - `expected_uid`：接受但不使用。
- **部署清单**：Node 的运行资产去掉 host 和 guardian，加上 `ora-reaper`，desktop 的 Node 镜像已经这样打包。
  自己拉起 Node 的部署在升级到本实现之前，要先带上 `ora-reaper` 并停止启动 host：找不到 reaper 时 Node 启动
  失败（D1）。

### D8：保证边界

本文的保证等级是 BestEffort
（[纳管保证](../process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md)的定义沿用）。停止
证据成立，表示继承了尝试锁的进程已经全部结束。以下情况不在保证之内：

1. **后代主动关闭继承的 fd**（例如常驻程序启动时关闭所有 fd），再脱离进程组。它既逃过终止，也逃过停止证据。
   Node 执行的 git 目前没有这种后代：仓库由 Node 新建，没有钩子；访问 Git Server 走 HTTP；credential helper
   和后台维护都已关闭。
2. **锁文件被别的进程打开并占住**。配置了独立工作负载用户时，权限挡住了工作负载用户。**没有独立工作负载用户
   的部署（Cloud 第一版的本地子进程路径）里，Agent 和 Node 是同一个用户，挡不住**：Agent 可以打开锁文件占住
   锁，让尝试一直停在未知。后果是卡住，不是误报已停止。
3. **Node 和 reaper 同时死亡之后、Node 重启之前**，git 继续运行。重启后按 D4 收口。
4. **终止线索没写下来，reaper 也已死亡**。Node 找不到该杀谁，只能等进程自己结束，期间尝试停在未知。
5. **本地清理不代表外部副作用已停止**。已经发到 Git Server 的 push 是否生效，由业务层对账。
6. **整机断电或重启**。所有进程都不在了，锁全部空闲，收口直接成立。有数据库记录的尝试，锁文件在记录之前已经
   刷盘（D2）；终止线索和没有记录的锁不要求在断电后仍在。

Strong（纳管范围内完整的退出证据）需要 cgroup：它能列出并一次杀掉范围内的全部进程，后代逃不掉。它要求部署
给 Node 一个可写的 cgroup 子树，本文不引入；[Linux cgroup 纳管](../process/containment/linux/0-cgroup-v2-containment.md)
仍是这个方向的依据。

### D9：取代和修改的既有决策

**整份取代**（改为 `superseded`，标题下注明取代者）：

| 决策 | 原因 |
|---|---|
| [architecture](../process/architecture/0-node-host-and-scope-guardians.md) | Node、host、guardian 三方分离不再存在 |
| [integration](../process/integration/0-shared-process-entry-and-resource-handoff.md) | Node 的 Git 不再经 host；"先记录再派发""先确认停止再对账"由本文 D2、D4 承接 |
| [control](../process/control/0-authorized-generations-and-leases.md)、[protocol](../process/protocol/0-independent-local-streams-and-state-recovery.md) | 没有 host，就没有宿主会话和本机协议 |
| [lifecycle](../process/lifecycle/0-durable-run-attempts-and-scope-closure.md)、[persistence](../process/persistence/0-independent-journals-and-responsibility-retirement.md) | 没有 Scope、Run 和独立日志；尝试只记在 Node 数据库里 |
| [io](../process/io/0-bounded-output-and-fenced-input.md) | 输出规则由本文 D3 承接 |
| [recovery 根决策](../process/recovery/0-single-authority-and-failure-recovery.md)、[rootless bootstrap](../process/recovery/20260917-rootless-guardian-bootstrap-and-reconnect.md) | 恢复由本文 D4 承接 |
| [upgrade](../process/upgrade/0-version-coexistence-and-safe-migration.md) | 升级由本文 D7 承接 |

**保持有效**：`node/process/containment` 下的 guarantees、linux、macos、windows 保持 `approved`。它们定义保证等级
和各平台的强纳管方向，不依赖 host／guardian 是否存在；本文沿用其中 BestEffort 和 Strong 的定义（D8）。它们的
用例原先由 process 代码的测试作证据，这些测试随代码一起删除，对应义务的状态改为 Missing。

**部分修改**（原决策状态不变，标题下加历史决策提示）：

| 决策 | 修改 |
|---|---|
| [最小 clone 执行契约](../repository/20260918-minimal-clone-execution-contract.md) D6 | "先持久关联 Execution／目标／Scope／Run 再通过 host 派发"改为 D2 的顺序；"恢复先核实旧 Scope 收尾"改为先取得停止证据。表格里的处置不变 |
| [clone 的 Git 副作用移出受理线程](../repository/20260925-clone-git-effects-run-outside-the-admission-worker.md) | 受理线程之外做的事从"host 交互"改为直接执行 git；线程划分和数据库只在受理线程写的规则不变 |
| [clone 中断判定](../repository/20260925-terminated-clone-attempt-fails-as-interrupted.md) D1、D3、D4 | 见本文 D5；D3 的"由 guardian 终止"改为由 Node 和 reaper 终止；D4 增加一种结束事实 |
| [从 Git Server 快照 clone](../repository/20261016-clone-from-git-server-snapshot-and-ora-git.md) | 恢复部分提到的 Scope 收尾改为停止证据；其余不变 |
| [恢复截止与停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md) D6 | 停止确认的证据来源，见本文 D5 |
| [`NotStarted` 停止确认](../revision/20261021-stop-confirmation-accepts-not-started.md) D1、D2 | 见本文 D5 |

## 不变量

1. 有副作用的 Git 命令，先有锁文件，再有尝试记录，最后才派发；已记录的尝试不会被重新执行。
2. 停止证据只有一个来源：重新对尝试锁加锁成功。超时、已发出终止信号、reaper 已退出、拿到数据库锁，都不是停止证据。
3. 没有停止证据，就不读目标目录、不释放输入、不发终态、不在同一资源上开始新的命令。
4. 锁文件不存在或探测出错时按未知处理，不按已停止处理。
5. 尝试锁的 fd 只交给它对应的那条命令；Node 的其他子进程拿不到。
6. 终止线索只用于杀进程，它的缺失或错误不改变停止证据的结论；按线索杀进程前必须核对身份。
7. Node 重启后先收口，再进入业务恢复。
8. 每个尝试最多一种结束事实；"已停止，没有退出状态"只能支撑 `Interrupted` 和续作恢复的停止确认，不能支撑成功。
9. Node 执行的 git 不启动后台维护。
10. 保证等级如实报告为 BestEffort，不因收口成立而报告为已确认静止。

## 为什么不是这些替代方案

| 方案 | 优点 | 为什么没选 |
|---|---|---|
| 环境变量标记，扫描 `/proc/*/environ` | 能直接指出残留的是哪些进程 | 没有 `CAP_SYS_PTRACE` 的 root 读不了另一个用户进程的环境（约束 1），读不到时分不清"没有残留"和"看不见"；清空环境再 exec 即可逃过；macOS 和 Windows 需要特权接口 |
| 给 Node 加 `CAP_SYS_PTRACE` 再用环境标记 | 环境标记可用 | 等于允许 Node 调试容器里的所有进程，为一个探测扩大权限 |
| 降权到工作负载用户的扫描助手 | 不加能力 | 多一个要部署和验证的部件，只适用于 Linux |
| 会话号扫描（guardian 的做法） | `/proc/<pid>/stat` 任何人可读，代码现成 | 后代 `setsid` 之后找不到；必须在启动后记下会话号，记录之前 Node 崩溃就没有依据（约束 4） |
| 只靠进程组和 pid | 最简单 | 后代换进程组就漏掉；启动后、记下 pid 前崩溃没有依据 |
| cgroup | 能列出并一次杀掉全部后代，可以达到 Strong | 需要部署给 Node 可写的 cgroup 子树，现有部署都没有；只有 Linux。保留为 Strong 的方向（D8） |
| `PR_SET_CHILD_SUBREAPER` | Node 存活时能回收所有孤儿后代的退出 | Node 一死就失效，解决不了重启后的问题 |
| `PR_SET_PDEATHSIG` | 内核直接送信号 | 见 D1 |
| 保留 host／guardian，只做减负（共用 Scope、guardian 退休） | 已实现，已验证 | 保留全部 1.1 万行并继续增加复杂度，固定等待和常驻进程只能减少，不能消除 |
| 子进程等 Node 写完记录再 exec | 保证"有进程就有记录" | 尝试锁在派发前就建好，已经不需要派发后的记录；多一段 fork 之后的同步逻辑 |

## 本决策未解决的问题

- **Strong 纳管。** 需要时在 containment 下另写决策，至少回答 cgroup 子树从哪里来。
- **Agent 和插件进程的孤儿后代回收。** 本文只让 reaper 进入 Node，不改变 Agent 进程的行为。
- **没有独立工作负载用户时的锁文件保护**（D8 第 2 条）。要解决需要给工作负载单独的用户，这属于部署隔离的决策。
- **Node 重启后接管仍在运行的 Git。** 本文延续同生共死；需要长命令跨重启继续时另写决策。
- **macOS 和 Windows 的落地。** D6 只给出机制，等 Node 支持这些平台时实现并验证。

## 验收义务

下面 13 条义务登记在
[Node 直接执行 Git 的核心用例](../../../test-cases/node/git-execution/attempt-locks-and-stop-evidence.md)，
每条的证据状态以那里为准。本文为 `implemented` 不表示 13 条都是 Covered：

1. Node 在启动 git 之后、写下终止线索之前被强杀：进程还在时判为未停止，进程结束后判为已停止。
2. 后代换了进程组或会话，仍然活着时停止证据不成立。
3. 期限到期后整组终止，停止证据成立后才返回结果；清理期限内不成立时结果为未核实。
4. Node 并发启动 Agent 会话和其他 git 时，尝试锁的 fd 不泄漏给它们。
5. 降权到工作负载用户的 git 仍然继承尝试锁；工作负载用户不能自己打开锁文件。
6. Node 执行的每条 git 命令都关闭了后台 gc 和 maintenance，fetch 之后没有脱离的维护进程占着锁。
7. 按终止线索杀进程：pid 被复用时不误杀；组长已退出、组员还在时能整组终止。
8. Node 被强杀后，reaper 终止仍登记着的进程组；reaper 启动失败时 Node 启动失败。
9. 重启收口先于业务恢复；clone 以 `Interrupted` 结束并保留目录；续作恢复在停止证据成立后释放输入，未知时保留。
10. 有尝试记录而锁文件不存在时保持未知。
11. 交付的旧命令仍占着锁时，不开始新的交付。
12. 升级：旧 host 目录里有未收尾的尝试时，新 Node 能启动，尝试被收口，host 目录被删除。
13. 一次 clone 不留下常驻进程；固定等待的前后对照。

## 落地顺序

五步都已完成：

1. **本文评审**：`proposed` → `approved`，同时登记核心用例，把 D9 的九组决策改为 `superseded`，并删除只服务它们
   的用例。
2. **`ora-process` 扩展**：降权时保留指定 fd；启动方可以让自己的子进程不向 reaper 登记；能判断 reaper 是否自己
   退出了。不改变原有调用方的行为。
3. **Node 直接执行**：尝试锁、终止线索、收尾、重启收口、reaper 接入、Node 数据库迁移和旧尝试收口，替换
   `managed/` 里经 host 的实现。从这一步开始，Node 不再启动 host 和 guardian。
4. **删除**：process 的 runtime、protocol、client、host、guardian、helper 代码，以及部署清单里的对应资产。
5. **本文改为 `implemented`**：按最终实现同步正文和用例证据，包括 clone、续作恢复用例里原先按 host／guardian
   登记的证据，以及 containment 用例失去的证据。

第 2、3、4 步在 desktop 的同一个 PR 里，按上面的顺序分成独立的 commit：涉及权限的改动、行为变化和大量删除
各自可以单独检视。
clone、续作恢复和交付对外的结果形状自始至终不变，变的只有 Node 重启后拿不到退出状态时 clone 的结论（D5）。
