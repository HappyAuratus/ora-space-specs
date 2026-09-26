# 受理与 clone 的 Git 执行分离

依据[已批准的受理与 Git 执行分离决策](../../../decisions/node/repository/20260925-clone-git-effects-run-outside-the-admission-worker.md)。
本文件登记“受理不等待 Git”与“在途执行不被重复驱动”两项承诺。决策尚未实现，证据均为 Missing；
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
- 第二个 clone 被持久受理，在第一个的 host 步骤结束前不派发（首版串行）。
- Node 数据库的全部写入发生在 worker 上；派发前尝试意图已经落盘，结束事实先于收尾完成落盘。

### 禁止结果

受理回复等待 Git 结束；会话因 Git 占用而关闭或重建；执行器持有数据库连接或写入 Node home；
为及时回复而跳过“先落盘再派发”。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| Git 阻塞期间受理及时回复且会话不断开 | 真实 host／guardian 与暂停的 HTTPS 远端，生产会话在阻塞超过 `frame_timeout_ms` 期间查询、确认并受理第二个 clone，断言回复时限与会话未关闭 | Missing |
| 数据库只由 worker 读写 | 执行器接口不接收数据库或 journal 句柄的类型层证据，配合上述测试断言单一已清理尝试与完整终态 | Missing |

### 决策依据

受理与 Git 执行分离 D1、D2、D4，不变量 1–3；执行契约 D6；中断失败判定 D4。

## In-Flight Clone Is Never Driven Twice

### 风险

Git 移出 worker 后，恢复节拍或重复受理在原 Run 仍运行时再次驱动同一执行：重复收尾会在旧 Git 仍可能
写入时读取目录，重复派发会产生第二个 Run，违反“每个执行最多一个 Run”。

### 前置状态

一个 clone 的 host 步骤（派发或收尾）正在执行器上运行，持续时间跨越多个 `recovery_interval_ms`。

### 触发

- worker 的恢复节拍多次到期；
- Controller 以同一执行身份重传 `CloneRepository`；
- Node 在步骤进行中正常停止，或被强杀后重启。

### 必须成立

- 在途期间恢复节拍不驱动该执行，重传只返回持久进度；该执行始终只有一个已记录尝试。
- 步骤结束后 worker 只落盘与持久尝试相符的观察；不相符的过期结果被丢弃。
- 正常停止时 worker 等待在途步骤结束并落盘（上限 `shutdown_grace_ms + cleanup_timeout_ms`）；
  强杀后重启只收尾原 Run，不以新身份派发。

### 禁止结果

同一执行出现第二个尝试；在途执行被再次收尾或派发；过期结果改写持久进度；停止时丢弃已取得的观察
而不收尾原 Scope。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 在途执行不被恢复节拍或重传重复驱动 | 小 `recovery_interval_ms` 下阻塞的真实 clone，断言多次节拍与重传后只有一个尝试、完整终态正确 | Missing |
| 过期结果不落盘 | worker 步骤机单元测试：执行已不在在途集合或 Run 不符时，结果被丢弃且持久进度不变 | Missing |
| 停止与强杀不产生新派发身份 | 现有 `stopped_node_settles_its_interrupted_clone_before_exit`、`killed_clone_recovers_original_run_without_redispatch` 在新结构下原样通过，另需断言停止时 worker 等待在途步骤后落盘 | Missing |

### 决策依据

受理与 Git 执行分离 D2、D3、D5，不变量 2、4、7；执行契约 D6；process integration D3。
