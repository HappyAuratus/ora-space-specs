---
status: proposed
---

# clone 的 Git 副作用移出受理 worker：worker 独占 SQLite 与受理，执行器只做 host 交互

Node 服务中唯一的阻塞 worker 线程目前同时负责 SQLite、会话受理和 clone 的全部 Git 执行：
`recover_clones` 在 worker 上同步驱动 clone，等待 host 上的 Git 结束（可达 `command_timeout_ms`），
期间状态查询、确认和事件重放都排在队列里等待。本文决定把 clone 中所有 host 交互（派发 Run、
等待并收尾、读取结束观察、只读核实仓库）移到独立的执行器；worker 只保留 SQLite 读写、目录
预留与证据、状态推进和会话受理，从而使受理回复的延迟不再取决于 Git。持久顺序、收尾屏障、
判定规则和对外协议均不改变。

当前 `status: proposed`，待 Eric 评审。涉及 desktop `apps/ora-node` 的 `service`（worker 与
会话之间的队列）、`repository.rs` 的 clone 状态机、`managed` 的受管 runner 拆分，以及 Node 文档
`docs/node/local-ipc.md` 中“worker 繁忙可导致会话关闭”的承诺。不改变 Node 协议消息、Node 数据库
schema、Controller 行为，也不改变 Git 与 Node 进程同生共死的生命周期策略。

## 继承与修改

| 前序决策约定 | 本文处理 |
|---|---|
| [clone 契约](20260918-minimal-clone-execution-contract.md) D1–D5、D7：输入、认证、独占目标、获取策略、结果形状 | 继承，不变 |
| clone 契约 D6：有副作用的尝试先持久关联再派发；恢复先核实旧 Scope 收尾，再检查仓库；未知不自动重跑 | 继承，不变；持久写入仍在派发之前完成，只是由 worker 写入、由执行器派发 |
| [中断失败判定](20260925-terminated-clone-attempt-fails-as-interrupted.md) D1：先关闭 Scope 再读取 Run 观察，运行与恢复共用 `AttemptSettlement` 判定 | 继承；判定函数不变，改为“执行器取得观察，worker 落盘并判定” |
| 中断失败判定 D3：Git 与 Node 进程同生共死，恢复前收尾屏障不变 | 继承，不变 |
| 中断失败判定 D4：结束事实先于“收尾完成”落盘 | 继承，不变 |
| [process integration](../process/integration/0-shared-process-entry-and-resource-handoff.md) D2、D3：变更关联先于派发；恢复先观测原 Scope Closed | 继承，不变 |
| [传输 ADR](../protocol/20260924-frame-channel-with-ipc-and-websocket.md) D5：应用层心跳与 I/O 期限是存活的唯一依据 | 继承；本文只消除“受理回复被 Git 拖过期限”这一原因，不改期限规则 |
| Node 服务现状：单一 worker 串行处理受理与 Git，受理回复可能等待整个 clone | 修改：受理回复只等待 SQLite 与本地目录操作（D1、I3） |

## 背景

2026-09-25 经 cluster Sandbox Server 联调时发现（问题 2）：clone 进行中，Controller 的
`GetExecutionStatus` 进入 worker 队列后要等 Git 结束才得到回复；会话读循环 `await` 这个回复
而停止读取，超过 `frame_timeout_ms`（10 s）即断开会话，约每 11 s 重建一次。经中继时后果更重：
Node 不读取对端 close，中继关闭 Node 侧要等关闭握手超时；控制槽迟迟不释放，代理恢复后的首次
重连被 `4409` 拒绝。

问题分两层：

- **2a 会话读写耦合**：读循环等待受理回复时不读帧。由 T2（会话读写解耦）处理，不在本文范围。
- **2b 受理排在 Git 之后**：即使读循环不再阻塞，状态查询的回复本身仍要等 Git 结束，Controller
  得到的仍是超时。只要 Git 在 worker 上同步执行，任何会话侧改动都无法让受理及时回复。本文处理这一层。

现状代码（desktop `9e51ccb`）中的阻塞点：

| 位置 | 在 worker 上等待的 host 交互 |
|---|---|
| `service/worker.rs` 恢复节拍 | `recover()`（Worktree）与 `recover_clones()` 串行驱动全部可恢复执行 |
| `repository.rs` `drive_clone` | `execute_clone`：派发 Run 并轮询至结束或期限到期（最长 `command_timeout_ms`，停止时 `shutdown_grace_ms`） |
| `managed/repository.rs` `settle` | 关闭 Scope（最长 `cleanup_timeout_ms`）并 `QueryRun` |
| `repository/inspection.rs` `verify` | 成功退出后以只读 Git 核实 checkout 与 commit |

`reserve_clone`（受理）本身只写 SQLite，已经与 Git 分离；Git 实际在下一次恢复节拍中执行，
但仍在同一线程上，因此受理被间接阻塞。

## 决策

### D1：worker 是 Node 数据库唯一的读写者，执行器不持有数据库连接

worker 继续拥有 `NodeDatabase` 以及现有的第二条连接 `ProcessJournal`，负责：

- 会话请求：受理 clone、状态查询、确认、事件重放；
- clone 状态机的全部持久推进：`Reserved → DirectoryCreated → Dispatched`、记录尝试意图、
  记录结束事实与收尾完成、写入终态或 Unknown；
- 目标目录的创建、`fsync` 与目录身份证据读取（本地文件系统操作，不经过 host）。

执行器只执行 host 交互，不打开 SQLite，也不写 Node home 下的任何文件。它从 worker 接收一个已经
持久化的步骤，返回一个观察，由 worker 决定如何落盘和推进。

为什么由 worker 独占数据库，而不是让执行器带一条自己的连接：

- Node 数据库使用 SQLite 默认的回滚日志模式，没有设置 `busy_timeout`。现有两条连接之所以安全，
  是因为它们在同一线程上交替使用，事务从不重叠。执行器在另一线程写库，会引入 `SQLITE_BUSY`、
  WAL 迁移和锁等待，把“事务短、只在数据库内”的约束变成需要额外证明的并发问题。
- “先持久关联再派发”（clone 契约 D6、process integration D2）和“结束事实先于收尾完成落盘”
  （中断失败判定 D4）都是写入顺序约束。由单一线程完成所有写入，顺序由代码顺序直接保证。
- 目录创建与证据读取留在 worker：它们与随后的 `DirectoryCreated` 写入属于同一步骤，时长受本地
  磁盘约束，不随网络或 Git 变长。

代价是 `drive_clone` 需要从“一次调用跑到底”改写为“按持久阶段逐步推进”的步骤机，`ManagedGitRunner`
需要拆成“journal 写入”（留在 worker）与“host 交互”（移到执行器）两部分。

### D2：clone 按持久阶段拆成步骤，每个 host 步骤的输入都已落盘

worker 根据执行的持久进度决定下一步；需要 host 的步骤交给执行器，结果回到 worker 后再决定
下一步。步骤划分如下：

| 持久进度 | worker 在交出前完成的持久写入 | 执行器步骤 | 返回给 worker 的观察 | worker 随后 |
|---|---|---|---|---|
| `Reserved` | —（本地目录创建与证据，无 host 步骤） | — | — | 写 `DirectoryCreated` |
| `DirectoryCreated` | `manage_clone`、推进到 `Dispatched`、记录尝试意图（Scope／Run） | 派发：CreateScope、Start，等待结束或期限，然后收尾 | 收尾后的 Run 观察 | 见下 |
| `Dispatched`，无已记录尝试 | 记录尝试意图 | 同上 | 同上 | 见下 |
| `Dispatched`，恰有一个尝试 | — | 收尾：关闭原 Scope 并 `QueryRun` | 收尾后的 Run 观察 | 见下 |
| `Dispatched`，多于一个尝试 | — | — | — | 写 Unknown（沿用现状） |

执行器对“派发”和“收尾”返回同一种观察：原 Scope 已观测 Closed 后的 Run 视图，或“收尾未确认／
查询失败”。worker 用现有的 `AttemptSettlement` 规则归类（中断失败判定 D1），先写结束事实，再写
收尾完成，然后核对目录身份：

- `Exited(0)`：交给执行器做只读核实（inspect），返回 commit、分支不存在或核实失败；worker 再次核对
  目录身份后写成功、`BranchNotFound` 或 Unknown。
- `Exited(非 0)` → `OperationFailed`；`Terminated` → `Interrupted`；`Unverified` → Unknown。

为什么要求每个 host 步骤的输入已经落盘：执行器可能在任何时刻随 Node 退出而消失。只要 host 步骤
的全部输入都能从数据库重新推出，重启后的恢复就能从同一持久阶段重新开始，不需要执行器留下任何
状态。派发步骤的 Scope／Run 身份在交出前写入，因此重启后只会收尾原 Run，不会以新身份重跑。

为什么执行器返回观察而不是自己写结束事实：收尾完成后、worker 落盘前若 Node 退出，重启后恢复会
再次关闭原 Scope（`transport::close` 先重放 CreateScope，再 Close，可重复执行）并再次 `QueryRun`，
得到相同观察。代价是这个窗口内 host 若已不再保留该 Run 视图，结果降为 Unknown；该窗口只有一次
进程内消息往返，与现状中“`record_outcome` 之前退出”的窗口性质相同。

### D3：同一执行任一时刻最多一个在途步骤，恢复节拍跳过在途执行

worker 在内存中维护在途集合（按 `ExecutionId`）。一个执行的步骤交给执行器后即加入集合，结果回到
worker 并落盘后移除。恢复节拍与新受理只驱动不在集合中的执行；过期结果（执行已不在集合中、
或返回的 Run 与持久尝试不符）直接丢弃并记录日志，不落盘。

现状下这一约束由“单线程串行”隐式保证；移出 Git 后必须显式维护，否则恢复节拍会在 Git 仍在运行时
对同一执行再次收尾或派发。集合只在进程内有效即可：Node 重启时执行器也随之消失，没有跨进程的在途
步骤需要防护；重启后的恢复按持久阶段重新开始（D2）。

### D4：执行器首版串行，一次只运行一个 host 步骤

执行器同一时刻只运行一个 clone 的 host 步骤，其他已受理的执行停留在各自持久阶段，按受理顺序
等待。worker 只在执行器空闲时把 `DirectoryCreated` 推进到 `Dispatched` 并交出，避免大量执行在
没有运行的情况下停在 `Dispatched`。

为什么首版不并行：本文要解决的是受理延迟，不是吞吐。串行保持现有“一次一个 clone”的外部可观察
行为，不引入新的配置项，也不需要先回答 host 的 Run 数量上限与磁盘、网络争用问题。D3 的在途集合
已经按执行而不是按全局加锁，将来并行只需要把执行器容量从 1 改为有界 N，不改变本文的不变量。

执行器的形态（独立线程加 current-thread runtime，或服务 runtime 上由 `JoinSet` 持有的异步任务）
属于实现选择；无论哪种，都必须由服务显式持有、在停止时等待其结束（D5），不允许无主任务。

### D5：停止与失败

- **正常停止**：`Shutdown` 请求后，worker 停止受理和新的派发，但继续运行，直到执行器上的在途步骤
  结束并把观察落盘，然后执行现有的 `shutdown` 收尾（关闭剩余 Scope）。在途派发会在
  `shutdown_grace_ms` 到期后被收尾，按中断失败判定落为 `Interrupted`，与现状一致。worker 等待
  执行器的时间以 `shutdown_grace_ms + cleanup_timeout_ms` 为上限；超时则不再等待，直接进入收尾，
  未落盘的观察由重启后的恢复补齐。
- **执行器返回错误**（host 不可达、协议不符）：按现状处理为 Unknown，保留预留。
- **执行器任务异常终止**（panic）：视为非预期错误，服务以错误退出，不静默转为 Unknown；重启后按
  持久阶段恢复。这与 Go 侧“非预期 panic 必须可见”的原则一致。
- **worker 异常退出**：服务退出，执行器随之取消；受管 Git 按同生共死策略由 guardian 终止。

### D6：Worktree 恢复不在本文范围，受理回复的承诺只覆盖 clone

Node 服务的会话拒绝 `EnsureWorktree`／`RemoveWorktree`，服务中不会产生新的 Worktree 执行；
`recover()` 只可能处理由其他嵌入方式写入同一数据目录的遗留执行，在正常部署中没有可恢复项，
不会执行 Git。本文不改写 Worktree 状态机：`recover()` 保留在 worker 上。等 Worktree 能力接入会话时，
再按本文的 D1–D3 拆分其步骤，另立 ADR。

## 不变量

1. Node 数据库的全部读写都在 worker 上发生；执行器不持有数据库连接。
2. 每个交给执行器的 host 步骤，其输入（Scope／Run 身份、目标、命令）在交出前已经持久化。
3. worker 处理一条会话请求时，不等待任何 host 交互；受理回复的延迟只取决于 SQLite 与本地目录操作。
4. 同一执行任一时刻最多一个在途步骤；恢复节拍不驱动在途执行。
5. 结束事实先于收尾完成落盘，终态只由已落盘的结束事实支撑（继承中断失败判定 D4）。
6. 同一 host 观察，无论来自运行路径还是恢复路径，得出相同的 `AttemptSettlement` 与业务结果。
7. 执行器在任意时刻消失，重启后的恢复都能仅凭持久状态继续，不产生新的派发身份。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 执行器持有自己的 SQLite 连接，`drive_clone` 整体搬到执行器 | 代码改动最小，状态机保持“一次跑到底”。但需要把数据库切到 WAL 并处理 `SQLITE_BUSY`，写入顺序的保证从“单线程代码顺序”变为“跨连接事务顺序”，违背 I1 带来的可审查性；worker 与执行器还会同时修改 `NodeState`。若 Eric 更看重改动量，这是可接受的备选，但需要补充数据库并发的证据 |
| 只做 T2（读写解耦），Git 留在 worker | 读循环能收 close／ping，但状态查询的回复仍要等整个 clone，Controller 仍按 `frame_timeout_ms` 判定超时；问题 2b 不消失 |
| 状态查询绕过 worker，由会话侧用只读连接直接读库 | 能让查询及时回复，但确认（`EventAck`）和重放仍排在 Git 之后；并且引入第二个线程读库，同样需要 WAL 才能在写事务期间读取。只解决一部分，并破坏 I1 |
| 执行器首版就有界并行 | 提高吞吐，但需要先决定容量、host Run 上限和资源争用，超出“受理不被 Git 阻塞”的目标；D3 已为之预留 |
| Node 重启后重新接管仍在运行的 Run（live-restore） | 见“本决策未解决的问题”；它以本文为前提，不能替代本文 |

## 兼容与风险

- **对外行为**：Node 协议、数据库 schema、判定规则和终态不变。可观察变化只有：clone 进行中的
  状态查询、确认和重放能在 `frame_timeout_ms` 内得到回复；`docs/node/local-ipc.md`（中英）中
  “worker 繁忙可导致会话关闭”的说明需要删除或改写。
- **单独落地不能让“存活”场景通过**：空闲会话仍需 T1（Controller 上行心跳），读循环仍需 T2。
- **观察落盘窗口**：见 D2。与现状相比，Run 收尾到结束事实落盘之间多一次进程内消息往返。
- **状态机改写的回归风险**：`drive_clone` 拆成步骤后，现有真实 host／guardian 测试（正常停止、命令
  超时、强杀恢复、guardian 丢失、`Interrupted` 经 WS 接管）必须全部原样通过，作为行为未变的证据。

## 本决策未解决的问题

- **Node 重启后重新接管仍在运行的 Run**（类似 containerd shim、Docker live-restore）。对云端沙盒有
  价值，但需要改变 Git 的生命周期策略（不随 Node 退出）、持久化 `command_timeout_ms` 的起算时间，
  并决定恢复时等待 Run 自然结束的边界。本文让“等待 Git”不再阻塞受理，是它的前提之一；方向本身
  另立 ADR，本文不做任何承诺。
- **执行器并行度**：见 D4。
- **Worktree 步骤拆分**：见 D6。

## 落地顺序

1. **前置：T2 会话读写解耦先落地。** T2 只改会话层，与本文互不依赖代码，但先完成 T2 可以让本文的
   验收直接观察到“clone 进行中会话不断开”，不被 2a 的症状干扰。
2. **拆分 `ManagedGitRunner`**：journal 写入与 host 交互分离，行为不变；现有测试全部通过即完成。
   单独提交，降低第 3 步的评审面。
3. **clone 步骤机与执行器**：实现 D1–D5，删除 worker 上的同步 Git 调用；补充一个真实 host 测试：
   clone 的 Git 被阻塞期间，状态查询、确认在 `frame_timeout_ms` 内得到回复，且会话不断开。
4. **文档与核心用例**：ADR 批准后，在 `test-cases/node/repository/` 增加“受理不等待 Git”与“在途执行
   不被重复驱动”的核心用例；同步修改 `docs/node/local-ipc.md` 中英文。

从第 3 步起，受理回复不再等待 Git；持久顺序、判定、终态和协议保持不变。
