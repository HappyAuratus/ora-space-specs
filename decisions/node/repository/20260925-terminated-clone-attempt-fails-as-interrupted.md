---
status: implemented
---

# clone 尝试被终止且收尾已确认时判为中断失败，不再停留在 Unknown

Node 受管 clone 的 Git 进程被信号终止，且 host 确认原 Scope 已收尾时，本次执行以新增的失败码
`Interrupted` 结束，保留目标目录；只有收尾或退出证据确实缺失时才保留 Unknown。Node 正常停止、
命令超时、Node 强杀后恢复以及外部终止都走同一判定。调用方看到明确终态，可用新执行重试，
而不是面对一个永远 `pending` 的操作。

本文继承[最小 clone 执行契约](20260918-minimal-clone-execution-contract.md)，修改其 D6 对“被终止”
这一类事实的处置，2026-09-25 经 Eric 批准并实现，状态为 `implemented`。涉及 Node 受管 clone 的收尾判定、Node 数据库 schema（v5）、Node 协议
`CloneFailureCode`、Controller 的 minicloud DTO 与 Cloud 映射，以及 Cloud 仓库的
Controller–Cloud 契约和公开 API 枚举。不改变 Worktree 的恢复语义，不改变 Git 进程的生命周期策略，
不引入重试、清理或原目录复用。

## 继承与修改

| 前序决策约定 | 本文处理 |
|---|---|
| D1–D5：输入、非交互认证、独占目标、固定获取策略、实际 commit 为结果 | 继承，不变 |
| D6：有副作用的尝试先持久关联再派发；恢复先核实旧 Scope 收尾，再检查仓库 | 继承，不变 |
| D6：“已知命令失败且原尝试已收尾 → 保存失败和残留说明，保留目标记录” | 继承，并明确“被信号终止”属于这一行，以 `Interrupted` 报告 |
| D6：“清理证据不可用，或目录／成功事实无法核实 → 保留 Unknown” | 修改：收窄为收尾未确认、退出事实缺失或查询失败；被终止且收尾已确认不再归入此行 |
| D6：不自动重跑、不在残留中重新 clone、用户重试使用新执行和新目录 | 继承，不变；`Interrupted` 同样只允许新执行重试 |
| D7：clone 是独立能力，结果形状属于 clone 协议 | 继承；在 clone 失败码中新增 `Interrupted` |
| [process integration D3](../process/integration/0-shared-process-entry-and-resource-handoff.md)：恢复先关闭原 Scope 并观测到 Closed；正常停止给有限宽限，超时请求收尾 | 继承，不变 |

## 背景

2026-09-25 经 cluster Sandbox Server 手动联调时，clone 进行中正常停止 Node：Node 在宽限期后
终止了 git，host 记录了收尾完成，但没有进程结果；重启后执行停留在 `Unknown(Dispatched)`，
Node 不再推进，Controller 一直显示 `pending` 并持续查询。

这不是契约要求的结果，而是实现把两类事实混在了一起：

- 运行路径：等待期限（`shutdown_grace_ms` 或 `command_timeout_ms`）到期时直接返回错误，随后关闭
  Scope，但不再读取 host 对该 Run 的终态观察，于是执行被记为 Unknown。
- 恢复路径：关闭原 Scope 后读取 host 观察，只接受“以退出码结束且收尾完成”，把“被信号终止且收尾
  完成”也当成证据缺失。

结果是凡 Node 主动终止的尝试都无法收敛：正常停止、超过命令期限（例如超过 5 分钟的 clone）、
Node 强杀后的恢复都会留下永久 Unknown。Unknown 会阻止原目标被复用，也不给调用方任何可行动的
结论，对只想重试的用户来说与卡死无异。

## 决策

### D1：按 host 的收尾观察把尝试分为三类，运行路径与恢复路径共用一个判定

每个受管 clone 尝试结束时，无论是 Git 自行退出、期限到期后由 Node 请求收尾，还是 Node 重启后
恢复，Node 都先关闭原 Scope 并观测到 Closed，再读取 host 对原 Run 的观察，按下表归类：

| host 观察 | 归类 | 后续 |
|---|---|---|
| 以退出码结束，收尾完成 | 已退出 | 退出码为 0 时按 D5 核实仓库并完成成功；非 0 为 `OperationFailed`（或已有的更具体失败码） |
| 被信号终止，收尾完成 | 已终止 | 以 `Interrupted` 失败结束，保留目录（D2） |
| 退出结果未知、尚未启动、收尾未完成、查询失败、关闭未确认 | 未核实 | 保留 Unknown 与预留，恢复继续按原规则核实 |

为什么“被信号终止且收尾完成”足以判定失败：

- 这是 host 可核实的事实，不是推测。收尾完成保证旧尝试已不可能继续写入；被信号终止说明 Git 没有
  给出自己的结论，因此不可能报告成功。前序 D6 禁止的是“把推测封为成功或失败”，而这里依据的是
  进程层的明确观察。
- 成功与终止的竞态已经被正确区分：若 Git 恰好在终止前正常退出，host 观察到的是退出码，走“已退出”
  并照常核实仓库，不会被误判为中断。
- 运行路径不再在期限到期时丢弃结果。关闭 Scope 之后重新读取观察，使 Node 在本次进程内就能落定
  终态；正常停止时终态随 outbox 保存，重启后交付，不依赖恢复路径补救。

“未核实”保持原来的保守处置：guardian 丢失、host 无法回答或收尾未确认时，Node 无法排除旧写入，
仍不能做出任何结论。`尚未启动` 也暂不改变：它意味着派发与启动之间的交接中断，其责任边界不在
本次讨论范围内。

### D2：新增失败码 `Interrupted`，保留目录，只允许新执行重试

`CloneFailureCode` 新增 `Interrupted`，表示“尝试在 Git 给出结论之前被终止”。它与其他失败一样是
不可变终态：

- 残留为 `Retained`，保留目标记录与目录，不删除、不在其中续传或重新 clone；
- 原执行身份重传只重放该失败结果；
- 用户重试使用新的 operation／execution，获得新目录。

为什么新增失败码而不复用 `OperationFailed`：两者对调用方意味着不同的下一步。`OperationFailed`
表示 Git 自己判定失败（网络、认证、仓库内容等），重试前通常需要排查；`Interrupted` 表示尝试被外部
条件打断（Node 停止或升级、超出期限、被外部终止），通常可以直接重试。把两者合并会让 Client 和
运维无法区分“该修配置”与“再试一次即可”。

`Interrupted` 不细分终止原因。Node 停止、命令超时与外部终止在 host 层是同一类观察；细分需要额外的
持久证据，目前没有调用方需要据此做出不同决定，原因可写入诊断日志。

失败码沿所有已有的结果传递路径同步增加，不新增其他字段：

| 位置 | 变化 |
|---|---|
| Node 协议 `CloneFailureCode` | 新增 `Interrupted` |
| Controller minicloud DTO（`ora-contracts::controller_api`）与 minicloud Client | 新增 `interrupted` 及文案 |
| Controller–Cloud 契约 `CloneFailureReason`（Cloud 仓库 proto） | 新增 `CLONE_FAILURE_REASON_INTERRUPTED` |
| Cloud 公开 API 的 clone `reason` 枚举与 Cloud 前端 | 新增 `interrupted` 及文案 |

### D3：保持 Git 与 Node 进程同生共死，恢复前的收尾屏障不变

受管 Git 的生命周期继续绑定 Node 进程：Node 退出时由 guardian 终止其 Run。恢复继续先关闭原 Scope
并观测到 Closed，再观察和修复资源。本文只改变“收尾之后如何解读观察”，不改变“何时收尾”。

收尾屏障是必需的：在确认旧执行者已不可能继续写入之前读取目录，得到的仓库事实不可信。这与租约
系统的 fencing、Kubernetes 在 Pod 终止后才卸载卷是同一原则，前序 D6 与 process integration D3 已经
确立，本文不放宽。

同生共死是对本机部署合理的选择：Node 退出通常意味着用户关闭或升级，留下占用仓库锁的孤儿 Git
更糟；配合 D1–D2，Node 退出的代价被限定为一次可重试的 `Interrupted`，不再是永久 Unknown。

### D4：终止事实作为 Node 数据库中的持久证据，schema 升至 v5

Node 数据库在写入终态的事务里核查已派发尝试的结束证据，这是前序 D6“没有进程证据不下结论”在
存储层的最后防线，不能在运行时绕过。因此“被信号终止且收尾完成”是数据库可核查的持久证据，
而不仅是内存中的判定：

- `ora-node.sqlite3` 新增 `process_terminations`（按 Run 记录终止信号），schema version 从 4 升到 5；
  v4 → v5 只建表，不改写任何已有尝试、结果或事件。
- 一个 Run 只能有一种结束事实：退出码与终止互斥，由数据库触发器与写入接口共同拒绝。
- 写入终态时，已派发尝试须恰好有一种结束事实：退出码支撑成功或 Git 失败；终止只支撑
  `Interrupted`；未派发的执行不能判为中断。
- 终止事实与退出码一样，在标记收尾完成之前写入，确保写入终态时证据已经落盘。

为什么新增表而不是改造 `process_outcomes`：新增表是纯增量迁移，已有行保持原样；改造原表需要重建
并搬迁已有退出码，风险更高而收益只是模型集中。旧 v4 程序按现有版本校验拒绝打开 v5 文件，不会把
没有退出码的 Run 误读为证据缺失；降级不重建或清空文件。

## 不变量

1. 只有 host 观测到原 Scope 已 Closed 之后，Node 才解读原 Run 的结束事实或读取目标目录。
2. 被信号终止且收尾完成的尝试以 `Interrupted` 失败结束，不以成功结束，也不停留在 Unknown。
3. 收尾未确认、退出事实缺失或 host 无法回答时，执行保持 Unknown 与预留。
4. `Interrupted` 是不可变终态：原执行重传只重放它，不重新派发、不在原目录续传或删除目录。
5. 运行路径与恢复路径对同一 host 观察得出相同归类。
6. 每个 Run 最多记录一种结束事实；`Interrupted` 终态只能由已记录的终止事实支撑，退出码不能支撑它。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 正常停止时等待进行中的 clone 完成（优雅排空） | 能减少正常停止造成的中断，但宽限总是有限：超过宽限或命令期限的尝试仍会落入同一个 Unknown，不能解决判定问题；可作为独立的停止策略另议 |
| 被终止后自动重跑（原目录或新目录） | 用户无需手动重试，但违反前序 D6 的“不在残留内重新 clone、不自动创建新尝试”；自动新目录还会在反复停止时累积残留 |
| 复用 `OperationFailed` | 零协议改动，但丢失“可直接重试”与“需先排查”的区别（见 D2） |
| 保持 Unknown，由调用方手动放弃 | 不需要改判定，但“强行放弃未知责任”本身需要单独决策；而这里的事实并不未知，没有理由让调用方承担 |
| Node 重启后重新接管仍在运行的 Run，而不是终止它（类似 containerd shim、Docker live-restore） | 云端 Node 升级时不打断长 clone，更优；但要求 Run 不随 Node 退出、恢复时等待 Run 自然结束，而当前 Node 在单一 worker 线程上同步执行 Git，等待会阻塞全部会话受理，且命令期限的起算时间未持久化。留作后续 ADR（见“本决策未解决的问题”） |

## 兼容与风险

- **存量 Unknown 可能自动收敛。** 升级后，恢复流程会再次处理 `Unknown(Dispatched)` 的执行；若 host
  仍保留原 Run 的观察，且结果为“被信号终止、收尾完成”，该执行会变为 `Interrupted`。host 已不再
  持有该 Run 时仍为 Unknown。这是按新规则对既有事实的重新解读，不改写任何已完成的终态。
- **Node 强杀后恢复的结果改变。** 此前恢复后停留在 Unknown，之后以 `Interrupted` 结束。依赖“强杀后
  必为 Unknown”的测试或运维判断需要更新；“不重跑、单一尝试、保留目录”的承诺不变。
- **跨仓库发布顺序。** Cloud 必须先接受新的 `CloneFailureReason` 值，desktop 才能固定到包含它的契约
  版本。旧 Cloud 收到未知枚举值时无法映射到公开 API 的 `reason`，因此不能先发布 desktop。
- **Node 数据库升级到 v5。** 升级后旧 Node 程序拒绝打开该文件；回退版本需要保留 v5 文件并使用新程序，
  不能靠删除或重建数据库回退。
- **诊断精度有限。** Git 若捕获信号后以非 0 退出码结束，host 观察到的是退出码，归为
  `OperationFailed` 而不是 `Interrupted`。结论仍然是失败且保留目录，只是分类不够精确；首版接受。

## 本决策未解决的问题

- **Node 重启后重新接管仍在运行的 Run。** 对云端沙盒（Node 升级、崩溃频繁，clone 耗时长）有价值。
  前提是 Git 执行移出 Node 的受理 worker，且命令期限的起算时间持久化，使接管后期限仍从原始启动
  时刻计算。届时另立 ADR，决定 Run 的生命周期策略与恢复时的等待边界，本文的收尾屏障保持不变。
- **`尚未启动` 且收尾完成的尝试** 是否可以判为可重试失败，需要结合派发交接的持久顺序另行确认。
- **优雅排空** 是否在正常停止时等待进行中的 clone，属于停止策略，另议。

## 落地与验收

按发布顺序落地：Cloud 契约先接受 `CLONE_FAILURE_REASON_INTERRUPTED`，desktop 再固定到该契约版本，
随后实现 D1、D2 与 D4。Node 主动或被动终止的 clone 尝试从此以 `Interrupted` 结束；成功判定、恢复
屏障、Worktree 恢复语义和目录保留规则不变。

验收由真实 host／guardian 测试覆盖正常停止、命令超时、强杀恢复与 guardian 丢失，由 Node 数据库测试
覆盖结束证据的互斥与 v4 升级，并由生产 Controller 会话经 WebSocket 接管正常停止后的中断终态、
不再保留待查询派发来覆盖端到端交付。原计划经 cluster Sandbox Server 手动复核，改由该自动化测试
承担：中继只透传帧，其转发行为已在此前联调中单独验证。验证义务与证据见
[最小 clone 执行契约核心用例](../../../test-cases/node/repository/minimal-clone-execution.md#terminated-clone-attempt-settles-as-interrupted-failure)。
