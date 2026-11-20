# 取消一轮与一轮结束后保留会话

依据[已批准的取消一轮决策](../../../decisions/node/protocol/20261116-cancel-turn-and-keep-session-after-turn-ends.md)。
本文的 30 个核心用例与该决策"验证义务"一节的 30 条一一对应，用例标题前的编号就是验证义务的编号。决策尚未实现，
所有义务都没有直接证据，状态为 `Missing`。会话命令、事件序号与终态的既有规则继续受
[Agent 运行的协议消息用例](agent-run-messages.md)约束；结束会话时的停止责任与停止证据受
[会话 MCP 刷新与宿主健康用例](../agent-runtime/session-mcp-lifecycle.md)约束，本文不推导它们的证据。
[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md)（`implemented`）修改了 D3 复述的停止证据：用例 11、12、13 各增加一行带“（尝试标记）”的
义务，不另占验证义务编号，一一对应关系不变。这三行按 [desktop#802](https://github.com/ora-space/desktop/pull/802) 的 head
`198cc415`（尚未合入 main）登记：11、12 为 `Partial`，13 为 `Missing`。上文“所有义务都没有直接证据”指取消决策自身的
30 条义务，它们的状态不在这次登记范围内。

| 验证义务 | 核心用例 |
| --- | --- |
| 1 | [Cancelling a running turn keeps the session for the next turn](#cancelling-a-running-turn-keeps-the-session-for-the-next-turn) |
| 2 | [Cancelling a queued turn settles immediately regardless of the turn ahead](#cancelling-a-queued-turn-settles-immediately-regardless-of-the-turn-ahead) |
| 3 | [Cancelling an ended or unknown turn reports already ended or unknown](#cancelling-an-ended-or-unknown-turn-reports-already-ended-or-unknown) |
| 4 | [A cancel racing a natural turn end never cancels the next turn](#a-cancel-racing-a-natural-turn-end-never-cancels-the-next-turn) |
| 5 | [Duplicate cancels during a stop share one conclusion](#duplicate-cancels-during-a-stop-share-one-conclusion) |
| 6 | [A cancel during a failed turn's silent window follows the failure judgement](#a-cancel-during-a-failed-turns-silent-window-follows-the-failure-judgement) |
| 7 | [Turn traffic inside the silent window ends the session](#turn-traffic-inside-the-silent-window-ends-the-session) |
| 8 | [A late update with the previous turn's identity is detected as a leak](#a-late-update-with-the-previous-turns-identity-is-detected-as-a-leak) |
| 9 | [Unfinished tools or subagent projection prevent keeping the session](#unfinished-tools-or-subagent-projection-prevent-keeping-the-session) |
| 10 | [An unlisted agent version never keeps the session](#an-unlisted-agent-version-never-keeps-the-session) |
| 11 | [An unanswered cancel settles in time and ends only after stop evidence](#an-unanswered-cancel-settles-in-time-and-ends-only-after-stop-evidence) |
| 12 | [Missing stop evidence never produces an early terminal or delivery](#missing-stop-evidence-never-produces-an-early-terminal-or-delivery) |
| 13 | [Turn boundary conformance gates the allowlist](#turn-boundary-conformance-gates-the-allowlist) |
| 14 | [A full in-flight window delays delivery but not local settlement](#a-full-in-flight-window-delays-delivery-but-not-local-settlement) |
| 15 | [The Controller declares an unsettled cancel only after silence beyond T](#the-controller-declares-an-unsettled-cancel-only-after-silence-beyond-t) |
| 16 | [An old Controller cannot take over an opted-in session](#an-old-controller-cannot-take-over-an-opted-in-session) |
| 17 | [Old Controller handshakes still decode](#old-controller-handshakes-still-decode) |
| 18 | [A failed turn keeps the session when failure keeping is chosen](#a-failed-turn-keeps-the-session-when-failure-keeping-is-chosen) |
| 19 | [Every mandatory end still ends the session and settles pending cancels](#every-mandatory-end-still-ends-the-session-and-settles-pending-cancels) |
| 20 | [Sessions without either choice keep the previous behaviour](#sessions-without-either-choice-keep-the-previous-behaviour) |
| 21 | [EndSession takes precedence over a queued cancel](#endsession-takes-precedence-over-a-queued-cancel) |
| 22 | [Session-level notifications do not break quiescence](#session-level-notifications-do-not-break-quiescence) |
| 23 | [Cancel, admission and MCP wait timeout have one result](#cancel-admission-and-mcp-wait-timeout-have-one-result) |
| 24 | [A cancel after admission but before the prompt write keeps the session](#a-cancel-after-admission-but-before-the-prompt-write-keeps-the-session) |
| 25 | [Cancels merged into a watchdog stop share one deadline](#cancels-merged-into-a-watchdog-stop-share-one-deadline) |
| 26 | [Leak detection spans all later turns but not normally completed ones](#leak-detection-spans-all-later-turns-but-not-normally-completed-ones) |
| 27 | [A restart before the conclusion settles the cancel as session ending](#a-restart-before-the-conclusion-settles-the-cancel-as-session-ending) |
| 28 | [A restart after the conclusion keeps the original end decision](#a-restart-after-the-conclusion-keeps-the-original-end-decision) |
| 29 | [A persistently unwritable ledger never settles from memory](#a-persistently-unwritable-ledger-never-settles-from-memory) |
| 30 | [The cancel and failure choices act independently](#the-cancel-and-failure-choices-act-independently) |

## Cancelling a running turn keeps the session for the next turn

### 风险

取消一轮后会话被结束，用户只能新开对话；或者事件顺序错乱，Controller 先看到下一轮再看到取消结论。

### 前置状态与触发

会话选择了 `keep_session_on_cancel`，Agent 版本在轮次边界清单中，一轮正在执行；Controller 发 `CancelTurn`，Agent 在截止时刻前回应，静止条件成立。

### 必须成立

- 事件顺序为这一轮的 Thread 事件 → `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}` → 下一轮的 Thread 事件。
- 同一会话接着受理 `SubmitUserTurn`，下一轮能看到之前的上下文。

### 禁止结果

会话因取消而结束；`TurnCancelSettled` 早于 `TurnEnded`；下一轮丢失上下文。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 1：取消运行中的一轮后会话保留、上下文保留、事件顺序正确 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1、D2、D4（事件顺序），不变量 7。

## Cancelling a queued turn settles immediately regardless of the turn ahead

### 风险

取消排队轮次要等前一轮跑完才有结论，超过 Controller 的等待上限；或者被取消的轮次仍被发给 Agent。

### 前置状态与触发

前一轮正在执行且运行时间超过 T，目标轮次已受理、仍在排队；Controller 取消目标轮次。

### 必须成立

- 受理后立即写入 `TurnCancelSettled{not_started}`，目标命令结算为 Discarded。
- 目标轮次从未发给 Agent，没有它的用户消息和 `TurnEnded`；前一轮照常执行。

### 禁止结果

结论等前一轮结束才写入；目标轮次被发给 Agent；前一轮被取消或丢弃。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 2：排队轮次的取消立即结算，不受前一轮运行时间影响 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（排队），不变量 1、5。

## Cancelling an ended or unknown turn reports already ended or unknown

### 风险

对已有结果或不存在的轮次，Controller 收不到结论，只能等到自己的超时。

### 前置状态与触发

分别取消：正常完成的轮次；以取消或失败结束且已判定保留会话的轮次；这个会话没有受理过的 `turn_id`。

### 必须成立

前两者立即得到 `already_ended`，最后一个立即得到 `unknown`；会话状态不变。

### 禁止结果

没有结论；改写已有结果；结束会话。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 3：已结束与不存在的轮次立即得到对应结论 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（已有结果、没有这个 `turn_id`），不变量 2。

## A cancel racing a natural turn end never cancels the next turn

### 风险

用户点停止时这一轮恰好自然结束，取消落在用户随后提交的下一轮上。

### 前置状态与触发

一轮自然结束与 `CancelTurn` 的生效竞争，用户随后提交下一轮。

### 必须成立

`TurnEnded` 保留 Agent 自己的停止原因；结论为 `already_ended`；下一轮照常执行，不被取消。

### 禁止结果

下一轮被取消；`TurnEnded` 被改写为取消。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 4：取消与自然结束竞争时只得到 `already_ended`，不误伤下一轮 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（为什么带 `turn_id`）、D4（先到者为准），不变量 5。

## Duplicate cancels during a stop share one conclusion

### 风险

停止中受理的第二个取消提前得到 `already_ended`，而这次停止最终以会话结束收场，两个结论互相矛盾。

### 前置状态与触发

同一轮先后受理两个 `CancelTurn`：第二个分别落在等待回应的 G 内、静默窗口 Q 内；另覆盖这次停止以兜底结束。

### 必须成立

两者得到相同结论；兜底时两者都得到 `session_ending`；各自的 `TurnCancelSettled` 按受理顺序紧接 `TurnEnded`。

### 禁止结果

任何一个得到 `already_ended`；两者结论不同；第二个重复发 `session/cancel`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 5：停止中重复取消并入同一次停止 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（停止中），不变量 4。

## A cancel during a failed turn's silent window follows the failure judgement

### 风险

一轮失败后的静默窗口内受理的取消，结论与这一轮最终的判定不一致。

### 前置状态与触发

会话选择了 `keep_session_on_failure`，一轮失败，Agent 已回应，正处于静默窗口；此时受理 `CancelTurn`。

### 必须成立

判定会话保留时结论为 `already_ended`；判定走兜底时结论为 `session_ending`。

### 禁止结果

在判定之前写结论；得到 `stopped`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 6：失败后静默窗口内的取消随判定结算 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（停止中）、D2，不变量 4。

## Turn traffic inside the silent window ends the session

### 风险

Agent 回应后仍在发上一轮的工具更新，下一轮仍被发出，两轮的工作混在一起。

### 前置状态与触发

取消后 Agent 回应，静默窗口 Q 内又收到上一轮的工具更新。

### 必须成立

这些更新不记入任何一轮；取消得到 `session_ending`；会话以 `turn_not_quiescent` 结束；下一轮从未发给 Agent。

### 禁止结果

更新记入上一轮或下一轮；会话保留；下一轮被发出。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 7：静默窗口内的轮次流量触发兜底 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（静止条件 3）、D3，不变量 8。

## A late update with the previous turn's identity is detected as a leak

### 风险

静默窗口之后才到达的上一轮更新混进下一轮，被当作下一轮的内容。

### 前置状态与触发

静默窗口已过，下一轮已开始，之后收到引用上一轮工具调用身份的更新。

### 必须成立

该更新不记入下一轮；会话以 `turn_leak_detected` 结束。

### 禁止结果

更新记入下一轮；会话继续。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 8：跨进下一轮的有身份更新被检出 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（跨轮次检测）、D3，不变量 8。

## Unfinished tools or subagent projection prevent keeping the session

### 风险

回应时仍有工具在运行，或子代理在父轮次结束后继续运行，保留会话后它们继续修改 checkout。

### 前置状态与触发

取消或失败后 Agent 回应时，分别存在：未终态的工具调用；这一轮启动过子代理投影。

### 必须成立

两种情况下会话都以 `turn_not_quiescent` 结束。

### 禁止结果

会话保留；把子代理的终态从父 prompt 结束推断出来。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 9：未终态工具或子代理投影使静止条件不成立 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（静止条件 2），不变量 7。

## An unlisted agent version never keeps the session

### 风险

没有验证过轮次边界的 Agent 版本也保留会话，无身份的迟到更新无法排除。

### 前置状态与触发

Agent 插件版本不在轮次边界清单中，一轮取消或失败结束。

### 必须成立

会话以 `turn_boundary_unverified` 结束，即使会话选择了对应的保留字段。

### 禁止结果

会话保留。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 10：不在清单中的版本走兜底 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（可依赖的轮次边界）、D3，不变量 7。

## An unanswered cancel settles in time and ends only after stop evidence

### 风险

Agent 不回应取消时，Controller 等不到结论；或者 Node 在取得停止证据之前就写终态。

### 前置状态与触发

正在执行的一轮被取消，截止时刻前 Agent 没有回应。

### 必须成立

- 受理后 G + Q 内已写入 `TurnEnded{cancelled}` 与 `TurnCancelSettled{session_ending}`。
- 之后不再写出任何 `session/prompt`。
- 取得停止证据之后才写 `AgentSessionEnded{agent_failed}`，detail `turn_cancel_unanswered`。
- 这一轮工具起的、脱离进程组但保留环境的后台作业（`cmd &`、`setsid`）在终态之前与会话的其他带标记进程一起被杀；
  停止证明记 `lock_processes_and_marks`。

### 禁止结果

结论晚于 G + Q；结论之后仍有 `session/prompt`；终态早于停止证据；后台作业仍存活时写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 11：没有回应时按期结算，终态等停止证据 | 尚未实现，无直接证据 | Missing |
| 11（尝试标记）：以 `session_ending` 结束时本轮的脱组后台作业在终态前被杀 | [ending_a_running_turn_kills_its_detached_tool_job](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L211)：轮次运行中以 EndSession 结束会话，本轮工具起的 `setsid` 作业在 `AgentSessionEnded` 之前退出、证明为 `lock_processes_and_marks`、之后不写入。缺口：仓库里没有经 `CancelTurn` 以 `session_ending` 结束会话的用例。评审在 `198cc415` 上核对取消以 `turn_boundary_unverified` 结束后进入同一停止确认，并用临时用例与真实插件复测确认作业先被杀，这些是运行记录，补进仓库后再登记 | Partial |

### 决策依据

D1（正在执行）、D3、D4（本地结算期限），不变量 2、9；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D4。

## Missing stop evidence never produces an early terminal or delivery

### 风险

停止证据始终取不到时，Node 提前写终态或允许交付、恢复，残留进程继续修改 checkout。

### 前置状态与触发

G 内没有回应；停止请求返回（分别覆盖成功与失败）之后，原尝试锁仍被持有、直接进程仍在，或仍有被尝试标记扫描
认领的存活进程（例如杀不掉的脱组后台作业）。

### 必须成立

- `TurnCancelSettled{session_ending}` 按期写入。
- 停止确认期限内和期限到达后都没有 `AgentSessionEnded`；期限到达后 `GetExecutionStatus` 返回 `StopUnconfirmed`。
- 这个执行自己的交付被拒绝（`session_not_settled`），带 `prior_revision` 的恢复被拒绝。
- 之后对账取得原证据，才补发终态，detail 仍是 `turn_cancel_unanswered`；终态之后没有事件。

### 禁止结果

把停止请求的返回当作停止证据；期限到达就写终态；交付或恢复被放行；补发的终态改了原因；锁与线索干净而带标记
进程仍存活时写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 12：证据缺席时不提前发终态、不提前交付，迟到证据才补报 | 尚未实现，无直接证据 | Missing |
| 12（尝试标记）：只剩带标记进程存活时同样缺证据 | [a_surviving_job_or_damaged_scope_is_never_proof](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/src/session/stop/tests/marks.rs#L109)：插件进程已退出、锁与线索干净，只剩脱组作业时证据为 Running；[end_session_kills_a_detached_tool_job_and_spares_unmarked_processes](https://github.com/ora-space/desktop/blob/198cc41517a837ec436a897655403adc790b8c0d/apps/ora-node/tests/standalone/agent_sessions/background_jobs.rs#L150) 等真实 Node 用例断言作业存活期间账本从未 `confirmed`。缺口：没有作业杀不掉时终态延后、期限后 `StopUnconfirmed`、交付与恢复被拒绝的用例 | Partial |

### 决策依据

D3（沿用会话 MCP 刷新 D5）、D4（Controller 等待终态），不变量 9；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D3。

## Turn boundary conformance gates the allowlist

### 风险

插件版本未经测试就进入轮次边界清单，回应后仍有轮次流量的 Agent 被允许保留会话。

### 前置状态与触发

对清单登记的每个 Agent 版本，用真实 Agent 构造：取消运行中的 shell 工具；取消带子代理和后台任务的轮次；模型请求错误。记录 prompt 回应之后到达的全部流量。

### 必须成立

只有回应之后没有任何轮次流量的版本才进入清单。会话被保留的取消不发任何停止信号，这一轮起的后台任务继续运行，
测试照实记录它们在回应之后的流量，不能假定 Node 已杀掉它们。

### 禁止结果

未测或测试中有残余轮次流量的版本进入清单；保留会话的取消杀掉后台任务，掩盖了回应之后的流量。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 13：轮次边界一致性测试决定清单 | 尚未实现，无直接证据 | Missing |
| 13（尝试标记）：保留会话的取消不杀任何进程 | 一致性测试中会话被保留的取消之后，带标记的后台任务与常驻进程仍存活、其流量照实记录的测试。desktop#802 `198cc415` 没有这样的测试 | Missing |

### 决策依据

D2（可依赖的轮次边界），不变量 7；[尝试环境标记决策](../../../decisions/node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) D4，不变量 7。

## A full in-flight window delays delivery but not local settlement

### 风险

在途窗口满时，Node 已按时结算，Controller 却把投递延迟误判为停止失败。

### 前置状态与触发

执行已有 256 个已发送未确认的事件时受理 `CancelTurn`；Controller 之后释放确认。

### 必须成立

- 结论在本地期限内写入账本。
- Controller 释放确认后按序收到结论。
- Controller 持续收到并确认事件期间，不判为超时。

### 禁止结果

结论绕过窗口越序发送；Controller 在持续收到事件时判为超时。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 14：窗口背压只延迟投递 | 尚未实现，无直接证据 | Missing |

### 决策依据

D4（本地结算期限、投递、Controller 等待取消结论），不变量 2、3。

## The Controller declares an unsettled cancel only after silence beyond T

### 风险

Node 确实没有结算时，Controller 一直等下去；或者按固定时长过早下结论。

### 前置状态与触发

连接仍在，Controller 已确认它收到的全部事件，自受理与最后一次确认中较晚的时刻起超过 T 没有新事件。

### 必须成立

Controller 判定 Node 未在期限内结算，把这次停止视为未确认，按自己的故障规则处理。

### 禁止结果

无限等待；把这次停止当作已停下。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 15：超过 T 的静默才判为未结算 | 尚未实现，无直接证据 | Missing |

### 决策依据

D4（Controller 等待取消结论）。

## An old Controller cannot take over an opted-in session

### 风险

不理解新事件的旧 Controller 接管已选择新行为的会话，收到无法解码的事件，或为它发出永远等不到结果的命令。

### 前置状态与触发

新 Controller 启动选择了新行为的会话，在确认 `TurnCancelSettled` 前断开；旧 Controller 重连，之后具备能力的 Controller 连接。

### 必须成立

- 旧 Controller 在 `HelloAccepted` 之前被以 `4403` 拒绝，没有收到任何事件，命令不被受理。
- 具备能力的 Controller 从最小未确认序号起按序收到全部事件。

### 禁止结果

旧 Controller 收到或确认任何事件；事件丢失或越序。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 16：接管边界与降级重连 | 尚未实现，无直接证据 | Missing |

### 决策依据

D5（接管边界、拒绝的持续时间），不变量 10。

## Old Controller handshakes still decode

### 风险

Node 升级后，对能力使用封闭枚举的旧 Controller 握手失败，即使没有任何会话选择新行为。

### 前置状态与触发

用本决策之前版本的 desktop `ora-node-protocol`，以及 cloud v2-server 的握手解码，分别解码新 Node 回给旧 `Hello`（不含 `turn_cancel`）的 `HelloAccepted`；Node 上没有选择新行为的会话。

### 必须成立

- 两者都接受这次握手，旧 Controller 正常连接、收发命令与事件。
- 只有声明了 `turn_cancel` 的 `Hello` 才收到这项能力和 `turn_cancel: true`。

### 禁止结果

旧 Controller 因未知能力值握手失败；未声明的 Controller 收到 `turn_cancel`。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 17：旧 Controller 的实际握手解码 | 尚未实现，无直接证据 | Missing |

### 决策依据

D5（能力协商），不变量 10。

## A failed turn keeps the session when failure keeping is chosen

### 风险

选择了失败后保留的会话仍因一轮失败而结束；或者 provider 会话已失效时下一轮发到失效的会话上。

### 前置状态与触发

会话选择了 `keep_session_on_failure`；一轮因 ACP prompt 报错或看门狗放弃而结束，满足 D2；另覆盖报错后 provider 会话已失效，以及 MCP 生命周期模式的会话。

### 必须成立

- 会话保留。
- provider 会话已失效时，下一轮先恢复再继续。
- MCP 生命周期模式的会话只 load 原 provider ID，失败时 blocked，不新建。

### 禁止结果

会话结束；下一轮发到已失效的 provider 会话；生命周期模式的会话新建 provider 会话。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 18：失败后保留与 provider 会话恢复 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（续用方式）、D5，不变量 7。

## Every mandatory end still ends the session and settles pending cancels

### 风险

新行为让某些必须结束的情况被误当作可保留；或者结束时有取消没有结论。

### 前置状态与触发

逐一触发 D3 表中的每一种情况，其中一部分发生时有尚未结算的 `CancelTurn`。

### 必须成立

每种情况都结束会话，终态与 D3 表一致；结束决定作出时尚未结算的 `CancelTurn` 都得到 `session_ending`。

### 禁止结果

任何一种情况保留会话；有已受理的 `CancelTurn` 没有结论。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 19：D3 的每种情况结束会话并结算取消 | 尚未实现，无直接证据 | Missing |

### 决策依据

D3，不变量 2。

## Sessions without either choice keep the previous behaviour

### 风险

没有选择新行为的调用方行为被改变，依赖"失败即终态"的 Controller 看到会话停在空闲。

### 前置状态与触发

两个字段都没有选择的会话；另覆盖旧 Node 与旧 Controller 的消息、握手与输入编解码。

### 必须成立

- `CancelTurn` 被拒绝（`turn_cancel_unsupported`）。
- 一轮失败仍以 `agent_failed` 和既有 detail 结束。
- 旧 Node 与旧 Controller 的消息、握手与输入编解码不变。

### 禁止结果

会话因失败而保留；产生 `TurnCancelSettled` 或新的 detail；旧形状消息无法解码。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 20：未选择的会话与旧对端保持原行为 | 尚未实现，无直接证据 | Missing |

### 决策依据

D5，不变量 11。

## EndSession takes precedence over a queued cancel

### 风险

`EndSession` 与 `CancelTurn` 同时排队时，会话被保留，或者取消没有结论。

### 前置状态与触发

`EndSession` 与 `CancelTurn` 同时在队列中。

### 必须成立

会话按 `EndSession` 结束；`CancelTurn` 得到 `session_ending`，早于终态。

### 禁止结果

会话保留；`CancelTurn` 没有结论或结论晚于终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 21：`EndSession` 优先 | 尚未实现，无直接证据 | Missing |

### 决策依据

D4（`EndSession` 优先），不变量 6。

## Session-level notifications do not break quiescence

### 风险

Agent 在两轮之间正常发送的标题、用量等通知被当作违约，本可续用的会话被结束。

### 前置状态与触发

取消后 Agent 回应，静默窗口内只收到会话级通知（标题、用量、可用命令）。

### 必须成立

会话保留；这些通知不记入任何一轮。

### 禁止结果

会话以 `turn_not_quiescent` 结束；通知记入某一轮。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 22：会话级通知不影响静止判定 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（轮次流量与会话级通知），不变量 13。

## Cancel, admission and MCP wait timeout have one result

### 风险

在 MCP 门禁中等待的轮次同时被取消、准入和等候超时，产生两个结果，或者取消之后仍被发给 Agent。

### 前置状态与触发

目标轮次已到队首、正在等 MCP 门禁或会话接入，分别让取消、等候超时、准入先提交；另对初始轮次（没有命令）覆盖三种顺序。

### 必须成立

- 取消先提交：`TurnCancelSettled{not_started}`，目标命令 Discarded，之后到 300 秒也没有 `SessionTurnFailed`，没有 `TurnEnded`；MCP 收敛继续，下一轮在门禁放行后照常准入。
- 等候超时先提交：`SessionTurnFailed{mcp_refresh_wait_timeout}` 与 Failed 已落账，之后生效的 `CancelTurn` 得到 `already_ended`，不改写失败。
- 准入先提交：取消按"已准入"或"正在执行"处理。
- 初始轮次的三种结果互斥，各至多记录一次。

### 禁止结果

同一轮既有 `not_started` 又有 `SessionTurnFailed`；取消后这一轮被准入；取消中止了 MCP 收敛。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 23：准入前三方竞争只有一个结果 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（取消、准入与 MCP 等候超时的先后），不变量 1。

## A cancel after admission but before the prompt write keeps the session

### 风险

准入已提交、prompt 还没写出时受理取消，Agent 仍收到这次 prompt，或者会话被无谓地结束。

### 前置状态与触发

Actor 已准入目标轮次，`session/prompt` 还没有写给 Agent 时受理 `CancelTurn`。

### 必须成立

Agent 从未收到这次 prompt；事件为 `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}`；会话保留，与轮次边界清单无关。

### 禁止结果

prompt 被写出；会话以 `turn_boundary_unverified` 或其他兜底原因结束。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 24：准入后、写出前的取消直接保留会话 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（已准入，`session/prompt` 还没有写给 Agent），不变量 7、14。

## Cancels merged into a watchdog stop share one deadline

### 风险

后到的取消重新起算宽限期，把先到的取消拖过它自己的期限；或者看门狗在用户取消后仍重发这一轮。

### 前置状态与触发

看门狗取消期间受理第一个 `CancelTurn`（A），在 A 的 G 即将到期时受理第二个 `CancelTurn`（B）；Agent 分别恰在截止时刻之前、之后回应。

### 必须成立

- 截止时刻为 A 的受理时刻加 G，B 不延长它。
- 截止前回应时两者都得到 `stopped`；截止后回应时两者都得到 `session_ending`，且 A 的结论不晚于 A 受理后 G + Q。
- 这一轮不再重发。

### 禁止结果

B 延长截止时刻；A 的结论超过 G + Q；看门狗重发这一轮。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 25：看门狗并入后只有一个截止时刻 | 尚未实现，无直接证据 | Missing |

### 决策依据

D1（一次停止只有一个截止时刻）、D2（时间参数），不变量 4、14。

## Leak detection spans all later turns but not normally completed ones

### 风险

被取消的一轮隔了几轮才到达的迟到更新漏检；或者正常完成的一轮之后的空闲期更新误结束会话。

### 前置状态与触发

被取消的一轮之后隔了一轮以上，才到达引用它的工具调用身份的更新；另覆盖一轮正常完成后空闲期到达的更新。

### 必须成立

前者会话以 `turn_leak_detected` 结束；后者按运行时既有规则丢弃，不结束会话。

### 禁止结果

隔轮的泄漏漏检；正常完成后的空闲更新结束会话。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 26：跨轮次检测的范围 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（跨轮次检测），不变量 8。

## A restart before the conclusion settles the cancel as session ending

### 风险

Node 在取消结论写入前重启，已受理的取消永远没有结论。

### 前置状态与触发

`CancelTurn` 受理之后、结论写入之前 Node 重启；另覆盖有未确认停止责任的情况。

### 必须成立

重启收口中这个命令得到 `session_ending`，早于 `interrupted` 终态；有未确认停止责任时，终态在取得原证据后才写入。

### 禁止结果

取消没有结论；结论晚于终态；没有停止证据就写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 27：结论写入前重启的收口 | 尚未实现，无直接证据 | Missing |

### 决策依据

D3（重启后沿用结束决定）、D4（本地结算期限），不变量 2。

## A restart after the conclusion keeps the original end decision

### 风险

`session_ending` 已写入、停止信号还没发出时 Node 重启，重启收口按 `interrupted` 重新结算，丢掉原因和停止时钟，或者重复产生结论。

### 前置状态与触发

G 内没有回应，`TurnCancelSettled{session_ending}` 已写入（分别覆盖已投递、未投递），停止信号还没有发出时 Node 重启。

### 必须成立

- 重启后不重复产生取消结论。
- 终态原因不是 `interrupted`，仍是 `turn_cancel_unanswered`。
- 停止时钟沿用原来的 `first_stop_at_ms` 与 `bound_at_ms`，不续期。
- 取得原停止证据之后才写终态。

### 禁止结果

重复的 `TurnCancelSettled`；终态改记为 `interrupted`；停止时钟被重新锚定；没有证据就写终态。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 28：结论可见后重启沿用原结束决定 | 尚未实现，无直接证据 | Missing |

### 决策依据

D3（结束决定与结论同一事务落账、重启后沿用结束决定），不变量 15。

## A persistently unwritable ledger never settles from memory

### 风险

账本写失败时，Node 把内存状态当作结论发出，或者继续派发下一轮；恢复后重复补写或漏写结论。

### 前置状态与触发

`CancelTurn` 成功受理后，注入账本持续写失败；之后让账本恢复可写。

### 必须成立

- Node 不发送任何没有落账的结论，不再写出 `session/prompt`。
- Controller 超过 T 后按未确认收尾，不把超时或内存状态当成已结算。
- 账本恢复后，在终态之前为这个命令补写恰好一条 `session_ending`，结束原因为 `ledger_unavailable`。
- 恢复前已有结论的命令不重复写。

### 禁止结果

发出未落账的结论；账本不可写期间仍写出 `session/prompt`；Controller 把这次停止当作已停；一个命令有两条结论。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 29：账本持续不可写时的结算与恢复 | 尚未实现，无直接证据 | Missing |

### 决策依据

D4（账本持续不可写），不变量 2。

## The cancel and failure choices act independently

### 风险

只选择取消后保留的会话在失败时也被保留，失败对调用方不可见；或者两个字段互相影响。

### 前置状态与触发

分别构造：只选择 `keep_session_on_cancel`；只选择 `keep_session_on_failure`；两者都选择；以及看门狗取消期间并入了 `CancelTurn` 的一轮。

### 必须成立

- 只选择 `keep_session_on_cancel`：取消后满足 D2 时会话保留；一轮失败时以 `agent_failed` 和既有 detail 结束，并入的 `CancelTurn` 得到 `session_ending`。
- 只选择 `keep_session_on_failure`：`CancelTurn` 被拒绝（`turn_cancel_unsupported`），一轮失败后满足 D2 时会话保留。
- 两者都选择：两种结束分别按对应规则。
- 看门狗取消期间并入了 `CancelTurn` 的一轮，按取消处理。

### 禁止结果

只选择取消的会话因失败而保留；只选择失败的会话受理 `CancelTurn`；并入取消的看门狗轮次按失败处理。

### 验证义务与证据

| 义务 | 直接证据与缺口 | 状态 |
|---|---|---|
| 30：两个字段的组合 | 尚未实现，无直接证据 | Missing |

### 决策依据

D2（结束方式表）、D5（两个字段与启用顺序），不变量 7、11。
