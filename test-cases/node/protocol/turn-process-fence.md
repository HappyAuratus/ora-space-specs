# 轮次进程栅栏

依据[保留会话前先停掉这一轮新起的进程决策](../../../decisions/node/protocol/20261123-turn-process-fence-before-keeping-session.md)
（`approved`）。本文的 8 个核心用例与该决策“验证义务”一节的第 1–8 条一一对应，用例标题前的编号
就是验证义务的编号。决策由 [desktop#807](https://github.com/ora-space/desktop/pull/807) 实现；各义务的状态为 `Missing`，
表中写明所需的直接证据与候选测试，由落地运行按合并提交核对后登记证据、改状态。该决策对取消决策验证义务 10、13 的修改登记在
[取消一轮用例](cancel-turn-and-keep-session.md)的对应用例中；结束会话时停止整个尝试的义务继续由
[会话 MCP 刷新与宿主健康用例](../agent-runtime/session-mcp-lifecycle.md)约束，本文不推导它们的证据。

| 验证义务 | 核心用例 |
| --- | --- |
| 1 | [A kept cancel kills the turn's detached jobs before settling stopped](#a-kept-cancel-kills-the-turns-detached-jobs-before-settling-stopped) |
| 2 | [The fence spares lazily started processes in the Agent's own session and everything older than the turn](#the-fence-spares-lazily-started-processes-in-the-agents-own-session-and-everything-older-than-the-turn) |
| 3 | [The turn scope follows descendants that clear their environment](#the-turn-scope-follows-descendants-that-clear-their-environment) |
| 4 | [An unclean fence ends the session as turn processes unstopped](#an-unclean-fence-ends-the-session-as-turn-processes-unstopped) |
| 5 | [The fence runs between the static checks and the quiet window within its own bound](#the-fence-runs-between-the-static-checks-and-the-quiet-window-within-its-own-bound) |
| 6 | [Normal completion and sessions that cannot keep never run the fence](#normal-completion-and-sessions-that-cannot-keep-never-run-the-fence) |
| 7 | [A restart around the fence ends the session and stops the whole attempt](#a-restart-around-the-fence-ends-the-session-and-stops-the-whole-attempt) |
| 8 | [Forged marks only kill the forger's own turn processes or end the session](#forged-marks-only-kill-the-forgers-own-turn-processes-or-end-the-session) |

## A kept cancel kills the turn's detached jobs before settling stopped

### 风险

会话在取消后被保留，而这一轮工具起的后台作业仍在运行，与下一轮并发修改 checkout（circus #1339 的反例）；或者下一轮
在栅栏完成之前就被派发。

### 前置状态

Linux；会话选择了 `keep_session_on_cancel`；Agent 的（插件，版本）为 Linux 登记；一轮正在执行，它的工具在新会话中
起了后台作业（工具命令以 `setsid` 启动，作业放到后台），作业会在 Agent 回应之后继续写 checkout；Controller 已提交下一轮。

### 触发

Controller 发 `CancelTurn`，Agent 在截止时刻前回应，工具调用都已终态，没有子代理投影。

### 必须成立

- `TurnCancelSettled{stopped}` 写入时，这些作业都已不在；之后 checkout 不再被它们写入。
- 事件顺序为 `TurnEnded{cancelled}` → `TurnCancelSettled{stopped}` → 下一轮的 Thread 事件；下一轮的 `session/prompt` 在
  `stopped` 之后才写出。
- 作业在回应与被杀之间写下的内容留在 checkout 中。
- 栅栏干净之后，Agent 这类豁免进程不发轮次流量地新起一个 `setsid` 进程，它在结算前的复扫时仍存活：会话以
  `turn_processes_unstopped` 结束，不写 `stopped`；复扫不向它发信号，它由结束会话的停止杀掉。

### 禁止结果

`stopped` 写入时作业或栅栏之后新起的范围内进程仍存活；`stopped` 之后出现作业的写入；下一轮在栅栏与复扫完成之前写出
prompt；栅栏向 Agent 进程或其进程组发信号；复扫发出任何信号。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 1：保留会话的取消在 `stopped` 之前杀掉这一轮新会话中的后台作业，下一轮在其后派发 | 需要：真实进程下，这一轮的 `setsid` 作业在 `stopped` 时已不在、延迟写入没有出现、会话接着执行下一轮的测试。候选：`apps/ora-node/tests/agent_session/turn_processes.rs` 的 `a_kept_cancel_kills_the_turns_setsid_job_before_it_settles`。真实插件的一致性运行（取消一轮用例 13）是运行记录，不算此项证据 | Missing |
| 1（复扫）：栅栏之后豁免进程静默新起的 `setsid` 进程让会话以 `turn_processes_unstopped` 结束 | 需要：Agent 在栅栏干净之后、Q 之内不发轮次流量地新起一个 `setsid` 进程（例如延迟数百毫秒），断言没有 `stopped`、detail 为 `turn_processes_unstopped`、复扫期间该进程未收到信号、终态前它被杀的测试 | Missing |

### 决策依据

D1、D2（结算前的复扫）、D4，不变量 3、5。

## The fence spares lazily started processes in the Agent's own session and everything older than the turn

### 风险

栅栏杀掉 Agent 懒启动、要跨轮次使用的 LSP 或 MCP，保留下来的会话功能退化（实测“本轮新起一律杀”会杀掉
`deno lsp`）；或者栅栏碰到轮次开始前就存在的进程。

### 前置状态

同上一条；此外，之前正常完成的一轮工具在自己的新会话里起了一个作业或开发服务器，仍在运行；被取消的这一轮里，
Agent 在自己的会话中懒启动了一个常驻进程（例如 LSP 或 MCP），它又起了子进程。

### 触发

取消这一轮，Agent 回应，栅栏运行。

### 必须成立

- 懒启动的常驻进程及其同一会话中的子进程在栅栏之后存活，下一轮找到的是同一个进程。
- 之前一轮留下的作业、插件运行时、Agent CLI 以及轮次开始前的任何进程，都没有收到冻结或杀信号；之前一轮留下的作业
  只在会话结束时被杀。

### 禁止结果

豁免会话中的进程被冻结或杀掉；轮次开始前就存在的进程收到信号；Agent 的进程组被终止。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 2：豁免会话中的懒启动进程与轮次开始前的进程都不收到信号 | 需要：Agent 在自己会话中懒启动的 helper 在保留会话的取消后存活、下一轮找到同一进程；正常完成一轮留下的 `setsid` 作业在下一轮被取消后仍存活、会话结束时才被杀的测试。候选：`turn_processes.rs` 的 `the_agents_own_helper_survives_a_kept_cancel` 与 `jobs_of_a_finished_turn_survive_the_next_turns_cancel`。一致性测试不专门构造懒启动（决策 D5 末段） | Missing |

### 决策依据

D1（轮次范围、为什么用 sid）、D7，不变量 3。

## The turn scope follows descendants that clear their environment

### 风险

轮次范围内的进程 fork 出清空环境的子进程，栅栏先杀父进程，子进程成为孤儿后再也认领不到，在会话保留后继续写入；
反过来，豁免会话中常驻进程的子进程被当作这一轮的进程杀掉；或者一个已经没有更早被认领成员的会话、一个已经消失的
会话的编号，让这一轮新起的进程被误豁免。

### 前置状态

这一轮的后台作业（新会话中）fork 出用 `env -i` 清空环境的子进程；更早启动、在另一个会话中的 helper 在这一轮里起了
一个子进程；某个更早的会话里只有不带标记的进程，这一轮有带标记的进程留在其中。

### 触发

取消这一轮，Agent 回应，栅栏运行。

### 必须成立

- 清空环境的子进程经被冻结的父进程认领，与父进程一起被杀；栅栏结论只在它们都不在之后才是干净。
- helper 的子进程留在 helper 的会话中，不被杀。
- 只含不带标记的更早进程的会话不豁免任何进程，留在其中的这一轮进程被杀；豁免会话只取自同一遍扫描中存活的更早被认领
  进程。
- 待定进程只在范围内计数：更早启动、读不到环境的进程不使栅栏为 Unknown；这一轮新起、不在豁免会话中、读不到环境的进程
  使栅栏为 Unknown。

### 禁止结果

清空环境的子进程在 `stopped` 之后存活；豁免会话中常驻进程的子进程被杀；不被认领的更早进程或已消失的会话让这一轮的
进程被豁免。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 3：轮次范围对后代封闭，豁免会话的后代不在范围内，豁免只来自同一遍扫描中存活的更早被认领进程，待定进程只在范围内计数 | 需要：收窄规则的表驱动单测，覆盖较早的待定进程不计数（去掉启动时刻过滤时必须失败）；真实进程下较晚的 `setsid` 作业被杀、较早进程及其会话中的较晚进程与 helper 的子进程存活；范围内进程 fork 出清空环境子进程时两者一起被杀的测试。候选：`crates/utils/src/process/environment_mark/scan.rs` 的 `a_new_session_reach_spares_the_sessions_of_earlier_claimed_processes`、`crates/utils/tests/environment_mark.rs` 的 `a_new_session_scope_kills_later_jobs_and_spares_earlier_sessions` | Missing |

### 决策依据

D1（范围对后代封闭）、D2，不变量 3、4。

## An unclean fence ends the session as turn processes unstopped

### 风险

栅栏判断不了这一轮的进程是否停下，或杀不掉它们，会话仍被保留；或者会话结束时没有可解释的 detail、并入的取消没有结论。

### 前置状态

会话选择了保留，版本已登记，一轮正在执行。

### 触发

分别注入：Agent 回应后不断起新的 `setsid` 作业，直到 F 用完；尝试的范围记录被删除；栅栏的扫描失败；范围内有待定进程
（Node 与 workload 不同用户、没有 `CAP_SYS_PTRACE`）；观察排在其他会话的停止观察之后超出预算；这一轮没有开始时刻（时钟
读取失败）；接收这一轮 prompt 的尝试在这一轮里被 MCP 生命周期替换；结算前的复扫超出 R 或扫描失败。之后取消这一轮或让它
失败，Agent 回应。

### 必须成立

- 会话以 `agent_failed`、detail `turn_processes_unstopped` 结束；结束决定与各个 `TurnCancelSettled{session_ending}` 同一事务
  落账，之后不再写出 `session/prompt`。
- 提前返回的不干净观察只在 F 剩余时重来，最后一次观察不干净才结束；没有开始时刻、范围记录不可用或尝试已被替换时
  不发信号，直接以同一 detail 结束。
- 结束后停止整个尝试（替换时覆盖两个尝试），栅栏没杀掉的进程在终态之前被杀；终态在取得停止证据之后才写入。

### 禁止结果

任何一种情况下会话被保留；以其他 detail 结束；并入的取消没有结论或得到 `stopped`；终态早于停止证据。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 4：栅栏或复扫不干净、没有开始时刻、范围记录不可用或尝试被替换时以 `turn_processes_unstopped` 结束，并按取消决策 D3 停止整个尝试 | 需要：每种注入各一个测试，断言 detail、并入取消得到 `session_ending`、之后没有 prompt、终态在停止证据之后；时钟读取失败要求能向轮次开始注入时钟。候选：`turn_processes.rs` 的 `jobs_that_keep_coming_back_end_the_session`（不断重起的作业使 F 用完）与 `an_unknown_fence_ends_the_session`（范围记录被删除） | Missing |

### 决策依据

D1（轮次开始）、D2、D3，不变量 4；取消决策 D3。

## The fence runs between the static checks and the quiet window within its own bound

### 风险

栅栏放错位置：在已决定结束会话时仍运行，推迟结论；在静默窗口之后运行，杀进程引起的 Agent 反应混进下一轮；栅栏
期间到达的轮次流量被忽略；或者栅栏的耗时让结论超出期限、并入的取消与 `EndSession` 得到不一致的结果。

### 前置状态

会话选择了保留，版本已登记。

### 触发

分别构造：回应时仍有未终态的工具调用；版本未为本平台登记；栅栏杀掉作业后 Agent 发来一条轮次流量（例如自发起一轮）；
栅栏期间受理第二个 `CancelTurn`；栅栏期间受理 `EndSession`（栅栏分别干净与不干净）；一轮因 ACP 错误失败（选择了
`keep_session_on_failure`），栅栏运行到接近 F。

### 必须成立

- 前两种情况不运行栅栏，分别以 `turn_not_quiescent`、`turn_boundary_unverified` 结束，结论期限与取消决策相同。
- 栅栏期间的轮次流量使会话以 `turn_not_quiescent` 结束；静默窗口 Q 从栅栏干净时起算。
- 栅栏不因命令中断；第二个 `CancelTurn` 在栅栏返回后并入，与这次停止得到同一结论；`EndSession` 在栅栏返回后优先，
  无论栅栏是否干净，会话都以它的原因结束，未结算的取消得到 `session_ending`。
- 结算前的复扫在 Q 与空闲期流量检查之后、`stopped` 之前运行，只扫描不杀；复扫期间到达的命令在它返回后处理。
- 结论在发起的 `CancelTurn` 受理后 G + F + Q + R（23 秒）内落账；失败发起时在回应后 F + Q + R 内落账。

### 禁止结果

已决定结束会话后仍运行栅栏；Q 在栅栏之前或与之并行结束；栅栏期间的轮次流量不被计入；复扫早于 Q 或晚于 `stopped`；
两个取消得到不同结论；`EndSession` 之后会话仍被保留；结论超出期限。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 5：栅栏在静态检查之后、Q 之前，复扫在 Q 之后、`stopped` 之前，期限为 G + F + Q + R，并入与 `EndSession` 在栅栏返回后按取消决策处理 | 需要：未登记或静止条件 2 不成立时栅栏没有运行的直接断言（例如没有发出任何信号、本轮作业仍存活到会话停止）；Agent 在自己的作业被杀后发轮次流量、结果为 `turn_not_quiescent` 的测试（把栅栏挪到 Q 之后时必须失败）；栅栏期间受理的 `CancelTurn` 与 `EndSession` 在栅栏干净与不干净时的测试；失败发起的栅栏与期限的测试。候选 `a_version_verified_only_on_another_platform_is_unverified` 只断言 detail，区分不出栅栏是否运行，不能单独作为此项证据；Claude 0.4.5 `background` 以 `turn_not_quiescent` 结束是一致性运行记录 | Missing |

### 决策依据

D4，不变量 2、6；取消决策 D1、D4。

## Normal completion and sessions that cannot keep never run the fence

### 风险

栅栏扩大到它不该作用的路径：一轮正常完成后杀掉这一轮起的开发服务器；没有选择保留或未登记的会话也运行栅栏、
扫描 procfs；或者记下轮次开始给每一轮增加扫描。

### 前置状态

一个会话选择了保留、版本已登记；另一个会话没有选择保留；第三个会话的版本未登记。三者的某一轮都起了后台作业。

### 触发

三个会话的这一轮都正常完成，随后各提交下一轮。

### 必须成立

- 三个会话都不运行栅栏，不发任何冻结或杀信号；后台作业进入下一轮。
- 记下轮次开始只是一次时钟读取，任何会话都不因轮次开始扫描 procfs。

### 禁止结果

正常完成的一轮之后有进程被冻结或杀掉；不能保留会话的会话运行栅栏；轮次开始时扫描 procfs。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 6：正常完成不运行栅栏；不能保留的会话不运行栅栏；轮次开始不扫描 | 需要：正常完成后作业存活的测试；没有选择保留的会话一轮结束后作业存活到会话停止的测试；轮次开始不扫描 procfs 的直接断言。候选：`turn_processes.rs` 的 `jobs_of_a_finished_turn_survive_the_next_turns_cancel`（覆盖正常完成） | Missing |

### 决策依据

D1（轮次开始）、D4，不变量 1、7；标记决策 D4。

## A restart around the fence ends the session and stops the whole attempt

### 风险

轮次范围只在内存中，Node 崩溃后没人杀这一轮的进程；被栅栏冻结的进程永远停在 T 状态；或者为此改了账本 schema。

### 前置状态

会话选择了保留，版本已登记，这一轮起了后台作业。

### 触发

分别在以下时刻 SIGKILL Node 并重启：轮次开始记下之后、prompt 写出之前；栅栏冻结了作业、还没杀掉时；
`TurnCancelSettled{stopped}` 写入之后、下一轮执行中。

### 必须成立

- 会话以 `interrupted` 结束；尚未结算的 `CancelTurn` 在重启收口中得到 `session_ending`，早于终态。
- 重启收口按尝试的范围记录冻结并杀掉原尝试的全部被认领进程，包括被冻结的作业；取得停止证据之后才写终态。
- 账本 schema 版本与本决策之前相同。

### 禁止结果

会话在重启后被保留或恢复；本轮作业在终态之后存活；被冻结的进程留在 T 状态；为轮次开始新增账本表或列。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 7：重启在任何时刻都结束会话、停止整个尝试，账本 schema 不变 | 需要：分别在一轮开始后、栅栏冻结作业时、保留会话后 SIGKILL Node 并重启，断言 `interrupted`、未结算取消得到 `session_ending`、本轮作业在终态前被杀、schema 版本不变的测试。重启收口杀掉原尝试带标记进程的既有证据见[会话 MCP 刷新与宿主健康用例](../agent-runtime/session-mcp-lifecycle.md#a-session-stop-kills-every-attempt-marked-process-and-finished-turns-kill-none)，不推导本义务 | Missing |

### 决策依据

D6，不变量 8；协议 D6；取消决策 D3、D4；标记决策 D2、D3。

## Forged marks only kill the forger's own turn processes or end the session

### 风险

伪造本尝试标记的进程借栅栏杀掉不归这一轮的进程，或让会话在仍有本轮进程存活时被保留。

### 前置状态

同 uid 的进程读到了标记；它在这一轮开始之前启动了一个带伪造标记的进程，又在这一轮开始之后、在新会话中启动了另一个；
另一种情形下，它在这一轮里持续 fork 带标记的进程。

### 触发

取消这一轮，Agent 回应，栅栏运行。

### 必须成立

- 轮次开始前启动的伪造进程不收到任何信号：它让自己所在的会话豁免。
- 轮次开始后、新会话中的伪造进程被杀。
- 持续 fork 时栅栏在 F 内不干净，会话以 `turn_processes_unstopped` 结束。

### 禁止结果

轮次开始前的进程收到信号；持续 fork 期间会话被保留。

### 验证义务与证据

| 义务 | 所需直接证据与候选测试 | 状态 |
|---|---|---|
| 8：伪造只影响伪造者自己在这一轮新会话中的进程，或让会话结束 | 需要：轮次开始前启动、带伪造标记的进程不收到信号，轮次开始后新会话中的伪造进程被杀，持续 fork 的伪造者让会话以 `turn_processes_unstopped` 结束的进程级测试。候选：持续 fork 一项为 `jobs_that_keep_coming_back_end_the_session` | Missing |

### 决策依据

D8，不变量 3、4；标记决策 D8。
