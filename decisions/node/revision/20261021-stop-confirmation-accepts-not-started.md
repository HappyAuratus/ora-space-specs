---
status: approved
---

# 恢复停止确认接受 host 权威的 NotStarted：原 Scope 已 Closed 且该 Run 从未启动即视为已停止，UnknownIdentity 仍按未知并等 host 能证明记录完整后重估

恢复的停止确认在[完整恢复截止与停止确认决策](20261018-complete-restore-deadline-and-cancel-confirmation.md)（下称“截止决策”）D6 中定义为「受管执行观测到原 Scope 已 `Closed` 且该 Run 的直接进程已退出」。本文扩展这条证据：原 Scope 已 `Closed`，且 host 权威报告该 Run 为 `NotStarted`（平台在 spawn 时明确报告未启动，或 guardian 在接受到启动请求之前已取消；cleanup 为 `CleanupState::Complete`，证据按实际取得的结果分级报告）同样构成停止确认，与「直接进程已退出」并列。host 返回 `UnknownIdentity`（从未记录该 Run）仍不构成停止确认，继续走 `stop_unconfirmed`，交给残留回收或运维显式确认；以后 host 协议能证明记录完整时再另开决策重估。预算到期、关闭请求、清理预算耗尽与等待上限仍然都不是停止确认。

本文为 `approved`，只改截止决策 D6 的停止确认证据，D1–D5 以及正常路径、未知路径、上限与回收规则全部继承。涉及 desktop 恢复编排与受管执行查询（`apps/ora-node/src/revision/`）；不涉及 Cloud 的 2/6/5 窗口、`stop_evidence` 写入与残留账本状态，也不新增会话状态或协议字段。兼容性边界：不改变正常路径的清理顺序与终态形状，不改变 `stop_unconfirmed` 的登记与转交；`UnknownIdentity` 一侧保持现状。实现尚未合入 desktop main，核心用例已按最终同意的方案补在[恢复截止与停止确认用例](../../../test-cases/node/revision/restore-deadline-and-stop-confirmation.md)，义务如实标为 `Missing`。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 截止决策 D1：完整恢复截止覆盖全程，命令预算裁剪到剩余时间 | 继承，不变 |
| 截止决策 D2：与 Cloud 2/6/5 窗口衔接、停止证据上限与转交 | 继承，不变；未知路径仍然存在（`UnknownIdentity` 一侧），只是收窄了落在 `NotStarted` 的那部分，窗口、上限与转交语义不变 |
| 截止决策 D3：取消点在 Git 命令边界，终态以停止确认为前提 | 继承，不变；取消等待的那条命令同样适用本文的 `NotStarted` 证据 |
| 截止决策 D4：半恢复交付隔离，停止确认未知时不交付 | 继承，不变 |
| 截止决策 D5：与传输介质及内部线决策的关系 | 继承，不变；本文的证据与传输介质无关 |
| 截止决策 D6 停止确认：「受管执行观测到原 Scope 已 `Closed` 且该 Run 的直接进程已退出」 | 修改：加上「或 host 权威报告该 Run 为 `NotStarted`」，两者并列；`UnknownIdentity` 明确不在其中 |
| 截止决策 D6 正常路径、未知路径、上限到期与回收路径 | 继承，不变；`NotStarted` 走正常路径，`UnknownIdentity` 仍走未知路径 |
| [进程恢复根决策](../process/recovery/0-single-authority-and-failure-recovery.md) D1：没有独立清理证据时保持未知，不把 Scope 锁空闲解释为 Closed | 继承：`NotStarted` 是 host 记录里的正面事实，不是证据缺失；`UnknownIdentity` 仍然只表示查不到记录，保持未知 |
| [纳管保证根决策](../process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D2/D3：错误或超时不制造完成证据，尽力清理不等于已确认静止 | 继承：`NotStarted` 必须来自 host 对已登记 Run 的观察，不能由预算到期、关闭请求、清理预算耗尽或上限到期推导；cleanup 证据按实际取得的结果分级——guardian 取消路径固定 `CleanupEvidence::BestEffortComplete`，平台 spawn 路径按 `cleanup_evidence(guarantee)`，BestEffort 的尽力清理完成不等于已确认静止 |
| [资源交接决策](../process/integration/0-shared-process-entry-and-resource-handoff.md) D3：先确认原 Scope 已 Closed，再观察和修复资源 | 继承，不变 |

## 背景

恢复命令的 attempt 已经持久化，但 `Start` 没有送达 host 时，交接只能拿到两种应答：

- `QueryRun` 返回 `HostRejection::UnknownIdentity`：host 从未记录这个 Run。attempt 落盘后 Node 立刻崩溃、`Start` 没有发出时会出现。
- Run 已登记但从未派发：平台 spawn 时明确报告未启动，或 worker 在 `Stop` 收到 `GuardianRunRejection::UnknownRun` 后合成 `NotStarted`；host 重启时恰好有恢复命令被准入、`Start` 还没来得及执行时会出现。

两种应答都不是「直接进程已退出」。按截止决策 D6，该会话永远停在 `stop_unconfirmed`，不会结束，整个 Node 的受控准入一直被拦住，只能等残留回收（specs#126）或运维显式确认释放；而窗口在 host 重启时并不罕见。

这两种应答的可信度并不相同：

- `NotStarted` 是 host 记录的正面事实。`crates/process-runtime/src/scope.rs` 只在平台 spawn 明确返回未启动时写 `DirectProcessState::NotStarted`；`crates/process-runtime/src/host/worker.rs` 在 `Stop` 收到 guardian 的 `UnknownRun` 时用 `cancelled()` 合成同一种事实，可信前提是 fencing（D1）。对已确认启动的 Run 报 `NotStarted` 会作为矛盾报错，不会静默出现。
- `UnknownIdentity` 只是「查不到记录」。它既可能来自「从未记录」，也可能来自 host 重启后记录丢失；当前协议没有信号能证明 host 记录完整，也没有 fencing 保证能排除后续 `Start` 到达，因此不能把「查不到」当作「没跑过」。

## 决策

### D1：原 Scope 已 Closed 且 host 权威报告 NotStarted 也算停止确认

停止确认的证据定义改为：受管执行观测到原 Scope 已 `Closed`，且满足其一：

1. 该 Run 的直接进程已退出（截止决策 D6 的既有证据）；
2. host 权威报告该 Run 为 `NotStarted`：`QueryRun` 返回的 Run 快照里 `direct` 为 `DirectProcessState::NotStarted`，`launch` 为 `LaunchFact::NotStarted`，cleanup 为 `CleanupState::Complete`。

`NotStarted` 只接受 host 对已登记 Run 的观察。来源限两类，共同事实都是没有任何进程被启动：

- **平台在 spawn 时明确报告未启动**：`crates/process-runtime/src/scope.rs` 只在平台 spawn 明确返回未启动时写 `DirectProcessState::NotStarted`，cleanup 证据取 `cleanup_evidence(guarantee)`——Strong 为 `CleanupEvidence::ConfirmedQuiescence`，BestEffort 为 `CleanupEvidence::BestEffortComplete`。
- **guardian 在接受到启动请求之前已取消**：`crates/process-runtime/src/host/worker.rs` 里的 `NotStarted` 是 worker 推出来的——它先向 guardian 发 `Stop`，guardian 回 `GuardianRunRejection::UnknownRun`，worker 再用 `cancelled()` 合成 `LaunchFact::NotStarted("cancelled before guardian acceptance")` 与 `DirectProcessState::NotStarted`，cleanup 固定为 `CleanupState::Complete(CleanupEvidence::BestEffortComplete)`。

guardian 这条来源为什么可以当证据：worker 只在“已经决定停止”的路径上合成这个事实，而且此时有两项 fencing 保证——当前 host binding 会挡住旧请求，worker 发出 `Stop` 之后不会再排 `Start`（代码注释：`Only a never-attempted guardian or its authoritative UnknownRun after fencing permits this fact`）。前者保证没有更早的 `Start` 还能穿过来，后者保证之后不会再有新的 `Start` 到达，因此 `UnknownRun` 只能表示这个 Run 从未被 guardian 接受启动请求。这正是它与 `UnknownIdentity` 的区别：`UnknownIdentity` 只说明按当前记录查不到这个 Run，既没有 fencing 证明旧请求已被挡住、后续不会再有 `Start`，也无法证明记录完整（journal 缺失、`process_attempts` 被清理），所以不能把“查不到记录”当作“没有进程在跑”。`NotStarted` 的可信度来自“不可能再有 `Start` 到达”这一结构保证，不是“此刻没看到进程”的观测。

cleanup 证据按实际取得的结果报告：平台 spawn 路径取 `cleanup_evidence(guarantee)`——Strong 为 `CleanupEvidence::ConfirmedQuiescence`，BestEffort 为 `CleanupEvidence::BestEffortComplete`；guardian 取消路径固定为 `CleanupEvidence::BestEffortComplete`。`CleanupState::Complete` 不承诺超出该证据的静止，`BestEffortComplete` 只表示完成承诺的跟踪与清理、仍不能排除未跟踪残留，不能读成“已确认静止”，也不能在报告里改写成 `ConfirmedQuiescence`（[纳管保证根决策](../process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D2/D3）。当前交付的 Linux BestEffort 下，两条来源都只能给出 `BestEffortComplete`。

| host 应答 | 事实 | 本文结论 |
|---|---|---|
| 直接进程已退出 | 进程已结束 | 停止确认成立（既有证据） |
| `NotStarted`（Run 已登记、未派发或接受前取消、cleanup 完成） | 没有进程被启动，且有 fencing 保证不会再有 `Start` 到达 | 停止确认成立，走正常路径 |
| `UnknownIdentity`（查不到该 Run） | 记录里没有，且无法证明记录完整 | 不成立，走未知路径 |

取得后走与直接进程退出完全相同的正常路径：在有界清理预算内清理临时目录与传输仓库（失败只记录），报告 `prior_revision_unavailable`（取消先生效时按取消原因），会话以 `agent_failed` 结束；不进入 `stop_unconfirmed`，原 Scope/Run 的冲突责任随正常路径解除，交付仍按 D4 的隔离规则处理。

矛盾检测：host 对已确认启动的 Run 报 `NotStarted` 必须作为错误拒绝，不能当作停止确认。这条规则保证第二种证据只在“host 明确知道它没有启动”时成立，不会因为状态不完整被扩展成“任何异常都算没跑过”。

其余前提不变：不能用预算到期、关闭请求、清理预算耗尽或等待上限推导 `NotStarted`，必须以 host 的观察为准；“取得停止确认后才发终态”的顺序、清理预算、上限与交付隔离也都不变。

### D2：UnknownIdentity 仍不是停止确认，写明有意保守与重估条件

`UnknownIdentity` 继续按未知处理：

- 不发终态、不交付、不读 HEAD、不重开或复制执行、不解除 checkout 冲突责任；
- 从首次停止请求起算的停止证据上限到期后，在原执行上登记 `stop_unconfirmed`，交残留回收（specs#126）或运维显式确认；之后的迟到证据仍按原执行身份补报一次。

这是有意保守：接受 `UnknownIdentity` 等价于把「查不到记录」当作「没有进程在跑」，而 host 记录不完整（重启后丢失 journal、`process_attempts` 被清理等）时两者无法区分。把证据缺失当作停止确认，会在可能仍有 Git 写入时发终态并开始交付，违反[进程恢复根决策](../process/recovery/0-single-authority-and-failure-recovery.md) D1 与[纳管保证根决策](../process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D2。

重估条件：当 host 协议能证明自己的记录完整——例如恢复 attempt 的 host epoch 等于当前 epoch 且 journal 连续——再另开决策评估是否把 `UnknownIdentity` 纳入停止确认。届时需要先给出可判定的完整性信号，而不是靠重试或时间推断。

## 不变量

1. 停止确认 = 原 Scope 已 `Closed` 且（该 Run 的直接进程已退出 **或** host 权威报告该 Run 为 `NotStarted`）。
2. `NotStarted` 只来自 host 对已登记 Run 的观察；预算到期、关闭请求、清理预算耗尽与上限到期都不能推导它。
3. 对已确认启动的 Run 报 `NotStarted` 是矛盾，必须拒绝，不能作为停止确认。
4. `NotStarted` 与 `UnknownIdentity` 的区分靠 fencing：只有“已决定停止、旧请求已被挡住、收到 `UnknownRun`、且之后不会再有 `Start` 到达”的路径才能合成为 `NotStarted`；`UnknownIdentity` 没有这项保证，不能当作停止确认。
5. `NotStarted` 的 cleanup 证据按实际取得的结果分级报告：guardian 取消路径固定 `BestEffortComplete`，平台 spawn 路径按 `cleanup_evidence(guarantee)`，`BestEffortComplete` 不等于已确认静止。
6. `UnknownIdentity` 不是停止确认；未知路径的禁令（不发终态、不交付、不复用、保留原责任）不因本文改变。
7. 本文不新增会话状态、不改 Cloud 的 2/6/5 窗口与残留账本状态：`NotStarted` 走正常路径，`UnknownIdentity` 仍走 `stop_unconfirmed`。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 同时接受 `UnknownIdentity`（评审中提过） | 窗口能全部收敛，运维负担最小；但 host 记录不完整时无法区分「没跑过」与「记录丢了」，等于用证据缺失制造停止确认，违反进程恢复根决策 D1 与纳管保证 D2。等完整性证明再评估（D2） |
| 保持现状，两种应答都不接受 | 不改任何证据定义；但窗口在 host 重启时并不罕见，整个 Node 的受控准入会被长期拦住，只能等残留回收或运维确认释放（D1） |
| 把 `NotStarted` 与「直接进程已退出」合并成「没有进程在跑」的单一事实 | 语义更简单；但两者证据来源不同——一次性平台应答 vs 持续观测的进程状态，合并会掩盖 `NotStarted` 的矛盾检测，也无法分别验证（D1、不变量 3） |
| 用等待上限到期推断「没有进程在跑」 | 不需要 host 应答；但到期不是证据，会把「不再等待」变成「已停止」，违反截止决策 D6 与纳管保证 D2 |
| 把接受与否放到 Cloud 侧判断 | 少改 Node；但 Cloud 看不到 host 应答，只能按自己的窗口推断，等于把业务收尾当作物理停止证明（D1、D2） |

## 兼容与风险

- **未知路径收窄。** 窗口里落在「Run 已登记但未派发」的会话将从 `stop_unconfirmed` 转为正常失败终态；`UnknownIdentity` 一侧行为不变。依赖「窗口内会话必然停在 `stop_unconfirmed`」的测试或运维判断需要更新。
- **`NotStarted` 的可靠性取决于 fencing 与 host 记录的正确性。** 只要旧 `Start` 仍可能到达、`Stop` 之后仍可能排入 `Start`，或 host 错误地对已启动 Run 报 `NotStarted`，本文的判定就会跟着错；D1 的 fencing 前提与矛盾检测是防线，实现必须同时落实。
- **不新增协议字段与状态。** desktop 只在既有 `QueryRun` 结果上多接受一种事实，Controller、Cloud 与残留账本无需改动；已批准实现的 `stop_unconfirmed` 规则保持有效。

## 本决策未解决的问题

- **host 记录完整性的协议形状**：`UnknownIdentity` 的重估依赖 host 能证明记录完整（epoch 与 journal 连续）。本文只写重估条件，不定义字段，届时另开决策。
- **窗口内的实际分布**：`NotStarted` 与 `UnknownIdentity` 在 host 重启窗口里各占多少，目前没有数据；由承接实现的运行实测并记录在 circus 的运行 issue 里，本文不预设比例。
- **能否更早识别该窗口**：attempt 已落盘、`Start` 未送达时，恢复编排是否可以更早发现并直接进入查询，属于实现范围；本文不改变派发顺序与持久化顺序。

## 落地顺序

1. **规格**（本文）：扩展 D6 的停止确认证据并同步核心用例。完成条件：本文 `approved`、截止决策标题下注明 D6 已被本文扩展、核心用例含 `NotStarted` 与 `UnknownIdentity` 两侧义务。
2. **实现**（desktop）：停止确认接受 `NotStarted`，合成 `NotStarted` 的路径必须满足 D1 的 fencing 前提，含矛盾检测与覆盖测试；`UnknownIdentity` 维持未知路径。完成条件：原 Scope `Closed` + `NotStarted` 时走正常路径发终态并清理；`Closed` + `UnknownIdentity` 时不发终态、到期登记 `stop_unconfirmed`；已确认启动的 Run 报 `NotStarted` 被拒绝；不满足 fencing 前提时不合成 `NotStarted`。实现还要记录窗口内两种应答的实测占比，作为重估依据。

保持不变：D1–D5、`stop_unconfirmed` 的登记与转交、交付隔离、Cloud 2/6/5 窗口与残留账本状态；截止决策继续为 `approved`，整份恢复 ADR 转 `implemented` 仍由原落地运行负责。

## 核心测试用例

见[恢复截止与停止确认用例](../../../test-cases/node/revision/restore-deadline-and-stop-confirmation.md)：在既有 `Stop Confirmation Requires Scope Closure And Direct Process Exit` 上写明基础证据与扩展的关系，并新增 `Stop Confirmation Also Accepts A Host-Authoritative NotStarted`，覆盖 `NotStarted` 走正常路径、两类来源、fencing 前提、矛盾检测、`UnknownIdentity` 保守与 cleanup 证据分级。实现尚未合入 desktop main，证据标为 `Missing`。
