# 受理与 clone 的 Git 执行分离

> 执行方式：本文依据的决策已由[Node 直接执行 Git 决策](../../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md) D5、D9 修改，Node 直接执行自己的 Git，不经 host 和 guardian。下文的前置状态和证据已按最终实现同步；停止证据本身的义务见[Node 直接执行 Git 的核心用例](../git-execution/attempt-locks-and-stop-evidence.md)。

依据[已实现的受理与 Git 执行分离决策](../../../decisions/node/repository/20260925-clone-git-effects-run-outside-the-admission-worker.md)。
本文件登记“受理不等待 Git”与“在途执行不被重复驱动”两项承诺，desktop 实现见 `apps/ora-node/src/service/clones.rs`
与 `repository/drive.rs`；
clone 的判定、终态与恢复义务仍见[最小 clone 执行契约](minimal-clone-execution.md)，本文不重复登记。

## Admission Replies Never Wait For Clone Git

### 风险

clone 的 Git 在受理 worker 上运行时，状态查询、确认和重放排在 Git 之后，会话因受理期限到期被关闭，
经中继时还会拖延控制槽释放；或者为了及时回复，把数据库写入挪到别的线程，破坏持久写入顺序。

### 前置状态

Node 以控制会话运行，Controller 已连接；一个 clone 已受理并派发，其 Git 被暂停的远端阻塞，
持续时间超过 `frame_timeout_ms`。

### 触发

Git 被阻塞期间，Controller 发送该执行的 `GetExecutionStatus`、已交付事件的 `EventAck`，
并受理第二个 clone。

### 必须成立

- 每个请求都在 `frame_timeout_ms` 内得到回复，会话保持连接，不以受理期限到期关闭。
- 状态查询返回持久进度（`Dispatched`），不等待也不读取 Git。
- 第二个 clone 被持久受理，在第一个的 Git 步骤结束前不派发（首版串行）。
- Node 数据库的全部写入发生在 worker 上；派发前锁文件和尝试记录已经落盘，结束事实落盘之后才删除锁文件。

### 禁止结果

受理回复等待 Git 结束；会话因 Git 占用而关闭或重建；执行器持有数据库连接或写入 Node home；
为及时回复而跳过“先落盘再派发”。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Git 阻塞期间受理及时回复且会话不断开 | `apps/ora-node/tests/standalone/repository_admission.rs` 的 `paused_clone_git_leaves_admission_prompt_and_dispatches_each_clone_once`：真实 Node 直接执行 Git，远端是暂停的 HTTPS，Git 阻塞超过 2 s 受理期限的两倍期间，状态查询与第二个 clone 均在 500 ms 内得到回复，第二个 clone 保持 `Accepted` 直到第一个完成，会话不关闭；尚缺 Git 阻塞期间 `EventAck` 的直接断言 | Partial |
| 数据库只由 worker 读写 | 类型层：`DirectGit`（`managed/repository.rs`）不持有数据库连接，执行器线程只拿到 `DirectGit` 与 `CloneStep`；结束事实与收尾由 worker 的 `resume_clone` 写入。上述测试断言每个 clone 的三条工作目录命令各有一条已收尾的尝试、没有待定的尝试，且终态成功 | Covered |

### 决策依据

受理与 Git 执行分离 D1、D2、D4，不变量 1–3；执行契约 D6；中断失败判定 D4。

## In-Flight Clone Is Never Driven Twice

### 风险

Git 移出 worker 后，恢复节拍或重复受理在原命令仍运行时再次驱动同一执行：重复收尾会在旧 Git 仍可能
写入时读取目录，重复派发会把同一条已记录的命令执行两次，违反“已记录的尝试不会被重新执行”。

### 前置状态

一个 clone 的 Git 步骤（派发或收尾）正在执行器上运行，持续时间跨越多个 `recovery_interval_ms`。

### 触发

- worker 的恢复节拍多次到期；
- Controller 以同一执行身份重传 `CloneRepository`；
- Node 在步骤进行中正常停止，或被强杀后重启。

### 必须成立

- 在途期间恢复节拍不驱动该执行，重传只返回持久进度；每条命令始终只有一条已记录的尝试。
- 步骤结束后 worker 只落盘与持久尝试相符的观察；不相符的过期结果被丢弃。
- 正常停止时 worker 等待在途步骤结束并落盘（上限 `shutdown_grace_ms + cleanup_timeout_ms`）；
  强杀后重启只收口原尝试，不重新派发。

### 禁止结果

同一条命令出现第二条尝试；在途执行被再次收尾或派发；过期结果改写持久进度；停止时丢弃已取得的观察
而不收尾原尝试。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 在途执行不被恢复节拍或重传重复驱动 | `paused_clone_git_leaves_admission_prompt_and_dispatches_each_clone_once` 在 50 ms 恢复节拍下阻塞数秒（上百次节拍），断言两个 clone 的三条工作目录命令各只有一条已收尾的尝试且均成功；尚缺 Git 阻塞期间同一执行重传 `CloneRepository` 的断言 | Partial |
| 过期结果不落盘 | 串行执行器下只有一个在途步骤，`service/clones.rs` 丢弃与在途执行不符的结果；该分支无法由外部触发，没有直接测试 | Missing |
| 停止与强杀不重新派发 | `apps/ora-node/tests/standalone/repository_recovery.rs` 的 `stopped_node_settles_its_interrupted_clone_before_exit`：停止进程自身等待在途步骤并落定 `Interrupted`，已记录的尝试全部收尾；`killed_clone_recovers_original_attempt_without_redispatch` 覆盖强杀后重启的 Node 先收口原尝试，尝试数不增加 | Covered |

### 决策依据

受理与 Git 执行分离 D2、D3、D5，不变量 2、4、7；执行契约 D6；[Node 直接执行 Git 决策](../../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md) D2、D4。
