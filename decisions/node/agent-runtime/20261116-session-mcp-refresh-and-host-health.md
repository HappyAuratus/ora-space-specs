---
status: proposed
---

# Node 保留会话身份刷新 MCP，以有界准入和停止证据隔离故障

Controller 持久保存 Session 的显式 MCP 选择，向 Node 提交完整内存配置。Node 在安全点收敛：仅配置
变化时用原 provider ID 执行 ACP `session/load`；包版本变化时先停止本 execution 的 Agent，再启动
它自己的 Agent 并 load 原 provider ID；同一 Agent 连续 8 次配置刷新后的下一次也走重启路径。
刷新失败阻塞新 prompt，下一轮自动重试，始终禁止回退到
`session/new`。配置完整、ACP 接受 revision、Host MCP Health 分别取证。

本文为 `proposed`，不是现有实现的描述。需要修改共享 `agent-runtime` 的宿主策略、Node 对话准入、
停止责任与持久账本、MCP 包租约及探测客户端。D5 的停止移交、持久证据、终态前提及门禁适用于
升级后的**所有 Node Agent 会话**，包括未启用 mcp_lifecycle 和空 MCP 的 v2 会话；不能通过能力
协商退出这项停止保证，正常终态也要等待原停止证据。Node 生命周期模式禁止重建的策略不改变 Desktop 默认
策略；共享 `ora-utils` HTTP 探测改为不跳转、限制响应体，会同时改变 Desktop 的探测行为。Node 重启
仍不恢复 Agent；Controller 的业务存储与页面实现遵守下述消费边界。

## 继承与修改

承接本目录前序[共享 checkout 会话](20261115-shared-checkout-agent-sessions.md)，经它继承
[显式 Session MCP](20261114-explicit-session-mcp-in-agent-session-spec.md)及
[共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)。MCP 生命周期由会话宿主拥有，
本文集中定义状态机及跨协议约束。

| 前序约定 | 处理 |
| --- | --- |
| 显式 Session MCP D1：有序完整选择，最多 16 个 Server，Settings 紧凑 JSON 合计 64 KiB | 继承；每份输入独立校验，空选择不自动发现 |
| 显式 Session MCP D2/D3：一次冻结 setup，configuration revision 为 0；重发 start 不更新值 | 修改：逐会话启用生命周期能力后，独立的内存更新可改变绑定；start 重发仍不更新值 |
| [会话命令](../protocol/20260928-streamed-thread-events-and-session-commands.md) D4：持久受理，拒绝即登记已送达 | 修改：新增可重试的未受理拒绝；准入前不得结算 Executed；已受理轮次可用可靠的无值失败事件结算（D4） |
| [实时预览](../protocol/20261018-live-thread-previews.md)：非持久观察丢失后重新查询 | 继承观察边界；MCP 更新、快照和健康不进入 Thread 或事件窗口；轮次失败和停止责任不是健康观察 |
| Node 重启不重启原 Agent，结算 interrupted | 继承不恢复 Agent；修改停止前提：有未确认 Agent/探测责任时，先恢复围栏，取得原停止证据后才补报 interrupted（D5） |
| [工作流显式选择](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) | 继承 Session 自身保存白名单；运行中丢失白名单时停止派发并 EndSession，不从 Node 观察重建授权 |
| [ACP setup/refresh](../../desktop/core/agent/session/mcp/0-acp-session-setup-delivery.md)：完整快照、barrier、失败后下一轮先重试 | 修改 Node 宿主策略：隔离后保留 Blocked 和原 provider ID，仅允许 load；不沿用现有 attach 失败后 new 的路径；Desktop 策略保持原状 |
| ACP refresh 的预算和进程边界 | 继承有 MCP 时 120 秒无活动窗口；修改 Node 包升级：允许同一 execution 内重启其独占 Agent（D4、D6） |
| [Host 健康探测](../../desktop/core/agent/session/mcp/20260916-mcp-health-probing.md) | 继承完整握手、三事实分离、无值内存结果；Node 只探测已授权 Session；共享 HTTP 客户端增加不跳转与有界读取 |
| [插件安装](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md) D2/不变量 4：按 ID 使用期间拒绝替换/移除 | 修改 MCP 为精确版本租约，可并存不可改写；其他类别保持原规则（D6） |
| 共享 checkout D2：恢复尝试独立阻止复用 | Git 恢复围栏不变；新增 Agent/探测的 stop_unconfirmed 围栏只挡 prior_revision 恢复和交付，普通新会话仍按共享规则及数量准入（D5） |
| [停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)及[未启动证据](../revision/20261021-stop-confirmation-accepts-not-started.md)：超时不是停止证据 | 继承命名原执行责任、有限等待、跨重启不续期与迟到确认；不恢复旧 host/guardian 架构 |
| [Git 尝试锁](../git-execution/0-node-runs-git-directly-with-attempt-locks.md)：attempt_lock 给出跨重启停止证据 | 扩展同一机制至 Agent/stdio 探测，独立目录与记录类型；不复活 host/guardian，不把未继承 fd 的后代算入证据（D5） |
| 范围决策中的“进程树清理” | 明确收窄：首版只负责直接子进程及仍处于原进程组的后代，以 `StopContainment::BestEffort` 报告；不声称 Strong 或整棵树静止（D5、D8） |

## 问题与约束

当前 Node 启动快照不能更新。共享运行时的刷新失败会隔离 channel、标记 Stopped 并清除 live MCP；
下一次 attach 的 load 失败又可以 new 新会话。Node 对话循环则在 prompt 受理前结算命令，受理失败
按 `agent_unavailable` 结束 execution。现有停止路径也会在 stop 失败后释放租约并删除 view。
因此下面的等待、保留身份和保留资源都需要新增行为，不能仅把内存配置源接入现有 Actor 就宣称完成。

设置值不能进入命令队列或账本。ACP 成功只证明输入被接受，不证明旧 MCP 进程已退出；健康握手也
不能证明 Agent 采用了相同连接。停止和资源释放必须另有证据，否则共享 checkout 会被残留写入者破坏。

## D1：能力逐会话选择，白名单独立于可变绑定

Node 只有同时支持 `agent_session`、`session_mcp` 及本文全部行为，才声明 `session_mcp_lifecycle`。
Controller 观察到能力后才可设置 `AgentSessionSpec.mcp_lifecycle: true`。该无值字段持久化并参与
重复 start 比较；省略/false 的其他调用方保持一次冻结 setup，已有执行不被隐式升级。

**Cloud v2 对非空 MCP 选择必须要求新能力，缺少时拒绝启动带 MCP 的工作流会话**，即使 Node 已有
`session_mcp` 也不降级为冻结 setup。空选择仍可按原能力启动。旧调用方不能接管生命周期会话提交
缺少 D4 generation 的轮次，但可以沿用查询执行及 EndSession 结束它。

新模式从 `spec.mcp`、generation 0 启动。canonical ID 集合 S 在 execution 全程不变，空 S 只允许
空集合。Controller 创建业务 Session 时一同持久化 S；页面重载、挂接与 provider 恢复都读 Session，
不反查工作流草稿或已安装插件。尚未启动且选择元数据缺失时，采用空显式集合或拒绝恢复；**运行中**
丢失 S 的处理见 D5，不能用空 update 改写已有授权。

更新必须按 canonical ID 排序，恰好为 S 的每个成员提供一项：`configured` 含精确 `plugin_version`、
无值 `configuration_revision` 和 Settings map；`unavailable` 含 ID 与 `mcp_plugin_missing` 或
`mcp_setting_missing`，不带值。unavailable、Node 缺包或解析失败使完整 desired blocked，不删除
成员、不发送部分列表。仅 S 内绑定变化才产生更新；业务上增减 ID 需要新的业务 Session。

## D2：更新是有序内存输入，不是持久命令

Controller 对每个业务 Session 串行分配 generation，在配置保存的权威事务中持久化无值目标身份
后发送完整更新。generation 与 configuration revision 为 `0..=2^53-1` 的 JSON 整数，新更新的
前者必须大于 0；上限处拒绝，不回绕，不以值或值摘要生成代号。

以下是待评审的消息语义，均沿用 `protocol_version`、`operation_id`、`execution_id`，请求另有
`node_id` 和有界 `request_id`。Controller 归属、执行身份及受控 incarnation/control scope 校验
先于访问会话和解析配置；调用方不能提供 Node 路径。

| 请求 | 行为与业务结果 |
| --- | --- |
| `UpdateSessionMcp` | generation + S 的完整列表；结果为 accepted、duplicate、stale、conflict、not_found、session_ended、not_enabled、invalid_selection 或 capacity_exceeded |
| `GetSessionMcpState` | 立即返回无值快照；not_found、session_ended、not_enabled、capacity_exceeded 均为明确结果；不触发解析、刷新、探测或 Agent |
| `ProbeSessionMcp` | 当前 generation、单个成员 ID、probe_attempt；结果为 accepted、duplicate、stale、not_found、session_ended、not_enabled、invalid_selection、not_configured 或 capacity_exceeded；立即回复，完成后查询 |

`SessionMcpReply` 用 request_id 关联，以结果和可用快照回复；not_found 不返回任何其他执行信息。
结构错误仍按协议校验失败处理，业务拒绝保持连接。更新仍限 16 项和 64 KiB Settings。单连接最多
64 个待答请求，满时立即回复新增 `capacity_exceeded`，不再保留请求/快照；若连这个控制回复也
无法写出，走原有慢连接关闭策略，不无限积压。被容量拒绝的请求没有任何更新/探测副作用。

Node 在 execution 内串行比较 generation，先原子替换 desired 与 prompt admission 的目标，再回复
accepted；不等待包解析、ACP 或探测，不表示 applied 或崩溃后可重放。

| 输入 | 处理 |
| --- | --- |
| g > 当前 G，选择/结构合法 | 推进目标并唤醒 Actor；包或设置不可用仍推进并 blocked，不继续旧配置 |
| g = G，完整输入相同 | duplicate；不重做 ACP 或主动探测 |
| g = G，输入不同 | conflict；不改变目标，不输出值差异或摘要 |
| g < G | stale；不解析旧值、不回滚 |
| 同一仍持有的成员配置身份，却给出不同 Settings | conflict + `configuration_revision_conflict`；不推进 generation；Controller 必须修正 revision 后重发，不派发依赖被拒目标的轮次 |
| 已结束/未启用/不存在 | 对应拒绝，不留值、不复活 |

配置身份包含成员 ID、精确包版本、configuration revision；比较仅针对仍在内存中持有的输入，
不保留无限历史或秘密摘要。Controller 对任何值变化都必须推进 configuration revision，不能
把“历史值已经释放”当成可复用旧 revision 的许可。

Controller 重发同 generation 必须用相同输入；凭据/包变化或显式重新提交用新 generation。若
丢失旧请求值，先查询，再从权威配置分配新的 generation，不能用新值重建旧请求。**下一轮对同
一 blocked generation 的自动重试不分配新 generation**，也不把重复 Update 当作重试触发器。

运行时最多保留 active、正在发送和最新 desired 三份输入，丢弃未发送中间值；这只是会话输入的
边界，另有至多 8 个 Node 探测 flight（每 execution 2 个）的成员绑定。flight 从开始到清理责任
转交都计数，不遗漏 cleanup_pending。原始 Settings 有值数据量上界为 `3 × N × 64 KiB + 8 × 64 KiB`，
N 为存活生命周期会话数；这不包括序列化缓冲、绑定展开及外部 Agent/服务内存，不能当作总 RSS 上界。
绑定帧仍受协议帧上限约束，解析与发送临时缓冲按实际使用者释放。排队意图只存无值身份，不能复制
Settings 到通用后台队列。秘密不写 WAL、历史、事件、日志、Workspace 或临时文件；错误只含稳定码
及适用的 plugin/setting/transport 标识。

generation 纳入 Desired/Active 的无值 revision。新 generation 即使内容相同也要完成新收敛，
不能把旧 ACP 响应改标为新目标；异步解析和成功响应写回前复查身份，过时结果不解除新目标的阻塞。

## D3：当前状态只通过查询取得

快照含 `NodeRuntimeIdentity { node_id, incarnation_id }`（本文的“Node 实例”）、execution、单调
`state_version`、desired/active 的无值身份、convergence、稳定错误及逐成员健康。convergence 为
`pending | refreshing | current | blocked | stopping | ended`。blocked 可保留“上次送达”的 active
身份，但 channel 可以为空；current 才表示最新目标已收敛。stopping 不可提交新 prompt。

逐成员快照另外给出当前 generation 的最高 `probe_attempt`（初始为 0）、`probe_in_flight`、
`queued`、`cleanup_pending`。Controller 重启或丢计数后先查询，再以最大 attempt + 1 重测；不能
从 1 静默重试。identity 不匹配的回复丢弃，查询后竞争造成 stale 时重新查询，不盲目增号。

查询回复不进入 Thread、execution sequence 或 EventAck，也不主动广播给不认识新消息的连接。
等待状态的 Controller 有界轮询，页面订阅消失即停止；按 Node 实例 + execution + state_version
合并。断线、旧实例或缓存丢失时展示未知并重新查询，不从帧写出或旧缓存恢复 current/Healthy。
结束的执行只返回 ended，不还原历史健康。

Node 保留跨会话的探测容量汇总：总数 8、正在执行、cleanup_pending、等待意图数及可用名额。
运维诊断能看到该汇总与命名残留，名额全部被 cleanup_pending 占用时明确标为“探测容量不可用”；
会话快照只暴露这一容量状态及本执行意图，不泄露其他执行身份。它不是某个 MCP Server 的 Unhealthy。

新消息和轮次失败事件要求 Controller/Node 同版本发布；不能声称对旧 Rust enum 解码器透明。能力
协商限制新模式的派发，不替代部署兼容检查。健康观察仍是内存事实；D4 的轮次失败及 D5 的停止
证据分别走其可靠路径，不以健康快照承担结算责任。

## D4：对话循环和 Actor 共同把住准入，失败仅 load 原身份

Node 对话循环拥有等待与命令结算，Actor 拥有最终准入、provider ID 和 Agent Session Barrier。
二者以状态变更通知协作，不能各自独立检查后直接发送。空闲时立即刷新；已进入 Agent 的 prompt
继续到安全点。权限请求未结算、prompt 未终止或 barrier 被占用时不提前 load，不为了温和刷新
额外 cancel。只有 active 等于最新 desired 且 current 时才能准入新 prompt。

共享 `agent-runtime` 增加 Node 生命周期宿主策略，具体义务如下；Desktop 默认策略不变：

- resolver/能力失败也进入 Blocked，而非停留 RefreshPending；ACP 失败隔离后保留 Blocked、最新
  desired、上次 active 身份和原 provider ID，`channel = None`，不能通过通用 mark_stopped 清成 Inactive。
  同一策略覆盖 `AgentProcessReplaced`、replaced 通知、load 期间进程替换及本文主动重启；这些路径
  都不能调用会清空 live MCP 的通用 mark_stopped。替换后状态是 pending/blocked、channel 为空，
  保留 desired 与 provider ID；上次 active 只能作为历史送达身份。持久 Session 行可记录无 channel
  的停止状态，但不得清除 provider 绑定或用它触发重建。
- attach 与 reconnect 使用同一策略：已有 provider ID 时只 `session/load`；不支持 load、原 ID
  不存在或 load 失败均 blocked，不经过 `create_provider_session`/transcript 迁移。只有最初尚未
  获得 provider ID 的创建允许 `session/new`，一次刷新绝不成为再次创建的理由。
- 下一轮触发对当前 blocked generation 的一次自动收敛；失败后保留该轮等待修复/新目标，不忙循环。
  每个轮次/目标最多触发一次这种重试；新 generation 可触发一次新尝试。没有新轮次/目标时不周期重试。
  后续另一轮可触发重试，合并同一 in-flight 尝试；重复命令和重复 Update 均不能制造重试风暴。
- MCP 导致的待刷新、busy 或刷新失败作为有类型的非终结 `McpAdmissionPending` / `McpAdmissionBlocked`
  返回，和真正的 agent unavailable 分开。通用 `session_busy` 不能被 Node 映射成整 execution 失败。

远程配置和轮次以 `SubmitUserTurn.mcp_generation` 关联下限。wire 上它是可选整数（兼容旧的
`deny_unknown_fields` schema 需同步升级）；生命周期模式必填，旧模式缺省且不得借它启用刷新。

| 阶段 | 必须成立 |
| --- | --- |
| Controller 接纳轮次 | 与配置保存串行读取当前 generation，把该下限与 turn/command 的无值派发记录一同持久化 |
| Node 接纳命令 | 已受理的重复命令先按原内容返回 accepted；新命令若 desired 低于下限，以新增 `mcp_update_required` 拒绝，不写队列、不开始 prompt |
| Controller 收到拒绝 | `mcp_update_required` 不登记已送达，先补 Update 再重试原 command；`session_ended` 仍登记已送达并按终态结算 |
| 重复命令比较 | mcp_generation 是内容的一部分；同 command 改下限构成内容冲突。重放原下限，Node 用最新 desired 满足它，不用旧配置回退 |
| Node 出队 | 在 `settle(Executed)` 前等待 pending/refreshing/blocked 收敛；初始轮次也经过此准入，不存在绕行 |
| Actor 最终受理 | 再原子检查最新目标并占有本轮准入；若 Update 赢得排序，返回 MCP 非终结结果，Node 保留队列并等待通知；受理成功后才能结算 Executed |
| 已受理后的账本失败 | 按内部错误停止，不重发 prompt；崩溃重启仍不重放未结算的 Agent 轮次，按 interrupted 收尾 |

新 blocked 会话的轮次可持久受理并触发上述自动重试，不新增 `mcp_refresh_blocked` 的投递拒绝。
准入成功只表示本轮归 Actor 所有，不表示轮次完成。按 Node 最终准入与 Update 的原子顺序判定：
准入先发生的轮次可以用旧目标；Update 先发生则尚未准入的全部轮次都等最新目标。Controller 保存
配置与 Node 准入之间不存在全局原子时钟，已开始轮次的凭据撤销不属于温和刷新保证。

**排队等待上限为 300 秒。** 每个已受理轮次从首次因 MCP 门禁受阻起计时，包括等 barrier、刷新和
blocked；普通前序轮次的业务运行时间不计入，只有 MCP 门禁阻止调度后开始计时。所有受阻轮次各自
记录首次时间，不因队首更换、generation 更新、活动通知、重连或重试重置。EndSession 始终优先。

到期仅该轮以稳定码 `mcp_refresh_wait_timeout` 失败，execution 仍可修复后接受下一轮。新增可靠
`SessionTurnFailed { turn_id, command_id?, code }` 事件与命令 `Failed` 结算同事务提交；初始轮次没有
command_id，但也有唯一 turn_id 和至多一次失败记录。它共享 execution 的连续序号和 Ack/重放规则，
不是伪造的 Agent Thread 记录；不能标 Executed 或普通 Discarded。Controller 区分“命令已送达”和
“轮次失败”。EndSession 先在同一串行结算边界生效则按原规则丢弃，终态之后不再发轮次失败事件。
Node 重启时未结算轮次按 interrupted 路径丢弃，不能重启一个新的 300 秒等待。

300 秒也包括因 MCP 重启导致的 barrier 等待、停止、spawn、initialize / agent_ready_timeout 和
load；不等进入 load 才开始。**接受正常但较慢的重启使已排队轮次（包括初始轮次）到期失败**，
否则反复启动失败会让 O4 的等待上限失效；会话仍可随后恢复，用户以新 turn 重试，旧轮次不复活。

ACP 刷新使用共享 `session_setup_window`：有 MCP 为 **120 秒无活动窗口**，收到 session update
重置；空集合沿用 30 秒。等待 barrier 不计入此窗口，但计入已开始的 300 秒准入等待。持续更新可
让 ACP 窗口延长，却不能延长轮次截止；轮次失败不伪称刷新已结束。解析错误沿用 setup 稳定码，
其他刷新失败为 `mcp_refresh_failed`，无活动到期为 `mcp_refresh_timeout`，不带第三方响应。

blocked 但没有排队轮次时，不增加 Node 自有的会话结束时钟，也不修改 Controller 的空闲窗口。
只有已有 Controller 空闲策略决定投递 `EndSession{IdleTimeout}`；配置更新或交互节点等待本身不
构成一个新的强制结束原因。300 秒仅限制轮次等待，不结束 Session，不取消仍在进行的收敛。

## D5：所有 Node 会话使用尝试锁交接停止责任，明确门禁和运维出口

本节覆盖所有 Node Agent 会话，不以 mcp_lifecycle 为开关。Controller 丢失运行中 execution 的 S
时，立即停止派发并持久投递 EndSession；不能从 GetSessionMcpState 重建授权或用空 update 改写它。
仅丢失值且 S 可信时，先查询再发送完整 unavailable 或新的当前配置。重连不替换 execution，Node
重启不重放凭据来复活 Agent。

### 停止证据与记录

复用 Node 的 `managed/attempt_lock` 机制，Agent/stdio 探测使用 Node home 下独立的私有
`session-attempts` 目录，避免和 Git 尝试的扫描/退休职责混淆。锁文件及目录先 fsync，再持久登记
尝试，然后在串行派发临界区加独占 flock 并 spawn；回收不能与这一临界区竞争，把尚未派发的记录
提前判为已停止。目标进程继承锁 fd，其他并发子进程不能继承；Node 在 spawn 后关闭自身副本。

每个 Agent/stdio 尝试新增一行 `session_stop_attempts`，含原 Node 实例、boot_id、execution、
checkout、attempt ID、kind、精确包版本/view 归属、状态及首次停止/截止时间。**新增专用表，不把
MCP argv 塞入 Git 的 `process_attempts` / `managed_executions`**：后者及现有 `AttemptCommand`
仍归 Git；复用的是尝试身份/锁机制。HTTP 本地任务也登记同表的独立 kind，由 Node 持有的锁 fd
覆盖本地任务存续，不能把远端服务状态记录成进程退出证据。记录不含 argv/env/URL/headers/Settings
或其摘要。schema 及序列化变更通过 Node DB 前向迁移。

spawn 后旁路写入 `pid/start_ticks/boot_id` 线索，仅用于安全终止原进程组，不能代替锁证据。PID
复用、boot 不符或线索不可读时不盲杀。Agent CLI 及适配器必须保有被授予的 fd 到其退出；若 CLI
启动 MCP 子进程时未继续传递 fd，锁只覆盖真正持有 fd 的进程。直接 Agent 退出与原进程组清理也要
观察；从不把一个提前关闭 fd 的活 Agent 判为停止。进程组加锁仍是 `StopContainment::BestEffort`，
不能宣称逃逸/关 fd 后代已退出或 Strong 静止。

Node 重启后先持有数据库独占权并恢复责任，再作有界证据检查：

| 原尝试证据 | 处理 |
| --- | --- |
| 原锁可取得，且没有仍存活的原直接进程线索 | 按 BestEffort 确认已停止或从未派发，正常立即结算 interrupted；不人为等满停止期限 |
| 原锁被占用，或原直接 Agent 仍存活 | 按原 pid/start_ticks/boot_id 尝试终止原组，再查锁/直接进程；没有取得停止证据前保留原责任 |
| 锁缺失、损坏、不可读，且没有可用的宿主重启证明 | Unknown，不能把无 PID 或无文件当成未启动；转交对账 |
| 持久 boot_id 与当前宿主 boot_id 不同 | 原内核的进程不可能存续，可记录宿主重启证据确认；这和仅 Node 进程重启不同 |

因此“记录已写、尚未 spawn 就崩溃”留下已刷盘且空闲的锁，能证明未启动/已停止，不造成假门禁；
已派发但未写 PID 线索时仍由继承 fd 证明存续，缺线索只限制终止手段。停止请求先持久锚定
first_stop_at_ms / bound_at_ms，再发信号，跨重试/重启不续期。原状态还没有停止时钟的重启收口只
锚定一次，不把每次重启都当作新的等待周期。

### 终态、转交与门禁

Node 生命周期 owner 在 driver 返回前接管未清理 Agent/探测、catalog 租约和 placement/view；
stop_plugin 失败不能仅记 warning 后 drop。更新中的旧 Agent 未确认停止时保持 blocked，不启动
替代 Agent；结束 execution 时 stopping，不再接受轮次/探测。HTTP 停止只确认本地任务/流/连接
已结束，远端 DELETE 仍是 BestEffort。

1. EndSession 先停止准入、取消探测并请求停止 Agent，正常回复持久受理；不阻塞心跳或兄弟会话。
2. 只有本 execution 全部 Agent/探测责任已取得停止确认，才封口并发唯一 AgentSessionEnded；
   对应使用者的租约/view 在其停止确认后释放，不必等其他使用者。Completed 不带 stop_unconfirmed。
3. 首次停止请求后按既有停止确认策略有界等待（默认 10 分钟，可配置为正有限值）。超过上限，
   或 Node 重启收口无法取得原证据，明确登记 stop_unconfirmed，结束等待任务并转交残留对账。
   只有这些状态触发新增残留门禁；“正在运行”或“期限内正常停止”都不是 stop_unconfirmed。
4. GetExecutionStatus 返回原 `StopUnconfirmed { scope_id, run_id, first_stop_at_ms, bound_at_ms,
   guarantee }`；scope_id 绑定原 execution，run_id 绑定原 attempt。有限的后续对账取得原证据后
   才补终态；首次原因、身份、序号不变。可能始终没有终态，Controller 依现有业务窗口收尾并保留
   残留责任，不能把业务结束或等待到期当停止证明。

同一 checkout 上新增 Agent/探测门禁与旧规则的关系如下；事务内判定，日志不可读时不能作出
“无残留”的肯定判断：

| 责任状态 | 普通新会话（无 prior_revision） | prior_revision 恢复 / 交付 |
| --- | --- | --- |
| 活跃 Agent/探测，或期限内正常停止 | 不增加 checkout_busy；未终态 execution 仍占原 8 个名额 | 恢复仍受原“有未结束会话不可恢复”规则；交付仍需自身会话已结算 |
| stop_unconfirmed（期限已过或重启收口缺证据） | 不增加残留拒绝；仍占未结束名额并按共享规则准入 | 恢复拒绝 checkout_busy；交付拒绝 session_not_settled，直到原责任被证据解除 |
| 已确认停止并完成责任交接 | 按原共享准入 | 解除这一残留围栏，仍检查其他旧恢复/交付约束 |

选择不挡普通新会话，是因为共享 checkout 本就允许其他活跃写入者；残留不增加恢复分支或快照
一致性之外的新承诺。A 在正常停止的数秒内不妨碍普通后继 B；带 prior_revision 的 B 仍按原规则
等 A 终态。**Git 的 legacy_unknown / restore_process_attempts 既有门禁不变**，不能用此新增规则
放宽旧 Git 围栏。残留独立于历史业务终态检查，不能靠只数未结束 execution 放行恢复/交付。

### 留审计记录的运维对账

新增 Node 本地管理操作 `reconcile-session-stop`，只允许持有 Node 管理权限的操作者通过受保护
本地入口调用，不放进普通 Controller/MCP 消息。每次明确提供 execution、原 attempt、幂等
request_id 和 reason_code（维护收口或探测容量恢复）；不能批量按模糊 PID 或目录释放。支持两种证据模式：

- `recheck`：在至多 10 秒内核对原锁及直接进程线索，必要时按已验证线索终止原组，再观察；得到
  Stopped 才确认，否则返回仍为 Running/Unknown，不刷新首次停止时间，不释放许可。
- `after-host-reboot`：核对记录里的原 boot_id 与当前不同，才以可验证的整机重启证据解除；同一
  boot 上缺锁且无法证明退出时拒绝。永久不可中断的进程需运维在 Node 排空策略下重启宿主后
  对账，不能用“强制清标记”制造停止证据。

操作先写 `session_stop_audit` 的请求记录，再执行，完成后写结果：操作者本地 UID、request_id、
原 execution/attempt/boot 身份、时间、原因码、证据类型、前后状态与释放的资源身份。只允许无秘密
枚举和标识，不接收自由文本配置/日志或采集进程命令行。审计写入失败则不提交解除。
重复 request_id 返回同一审计结果，尚在执行则返回进行中，输入不同则冲突；中途崩溃恢复后仍须
重新查证，不能仅凭请求已记录就释放资源。确认、回收责任标记和额度释放事务衔接；
随后后台幂等移除对应 view/锁，不能先删证据再提交确认。由此能查到谁以什么证据释放 checkout
围栏和探测名额，失败也留痕。已确认责任的计数从 Node 恢复时重建，重启不能凭空返还许可。

### 回收、部署与兼容

每小时对确认满 7 天的记录分批回收（每批最多 1,000 条）：须已持久记录确认、资源释放完成、没有
活跃尝试/终态结算/outbox 对该行的引用，才删除停止明细和已退休锁/线索。长期未结束会话的已完成
重启尝试也可回收，不必等整个 execution 结束；结算所需的无值汇总先事务压缩到 execution 归属。
未确认记录永不按年龄删除。审计记录至少保留 90 天后按运维日志保留策略归档/压缩；账本事件和
Ack 历史按原可靠性规则，不能借停止记录 GC 裁掉。回收积压量与最老记录年龄进入运维监控。

所有 Node 停止记录/锁使用本地文件系统。Separate workload 目录必须与 Node home 及实际 installed
包目录在**同一文件系统**；启动先检查设备及硬链接能力，每次 view 创建仍核对并拒绝 EXDEV，不
退化为完整复制。既有跨设备部署须先排空并迁移目录，失败报部署配置错误，不偷偷复制或删除旧目录。
Shared 无私有 view；不改其他调用方的通用 linked_tree 回退策略。

本次**不支持原地回滚旧二进制/schema，或把当前账本恢复到旧备份**。旧快照会丢失 Controller 已
确认的序号和新停止责任，排空以外的回滚尤其不安全；故障采用前向修复，任何未来回滚流程必须先
证明无未结束 execution、无未清除停止责任且事件接管已对齐。旧 start 的 mcp_lifecycle 缺省 false
仅兼容输入，不豁免全会话停止保证。升级前要求旧进程排空，因为旧会话没有继承尝试锁，不能为
仍活着的旧 Agent 补造证据；未满足则拒绝升级启动。desired/active、健康及 Settings 不进这些表。

## D6：版本变化及第九次配置刷新重启自己的 Agent

MCP 租约改为 canonical ID + 精确版本；发布、取租约、校验与删除共用 catalog 同步边界。在用
版本不得改写/删除，新旧均为 MCP 才可并存，不能换 kind 绕过 Agent 保护。其他类别仍按 ID 保护。
整 ID RemovePlugins 或在用同版本修复继续返回 plugin_in_use。

版本改变必须重启本 execution 独占的 Agent。另设每个 Agent 进程的配置刷新计数：初始 load/new
不计；随后每次仅配置变化且完整 load 请求已经写出就加一（失败响应也计，避免靠失败无限积累
注册）；重复 Update、未发送的 superseded 目标和解析前失败不计。**已连续发送 8 次后，下一次
配置刷新必须先重启**，不允许第 9 次运行中 load。只在旧进程停止确认且新进程首次 load 成功后
重置计数；失败重启不能恢复额度。尚未完成的重启可在新进程上重试首次 load，这不是一次额外的
运行中配置刷新；成功后才归零。普通 prompt 不清零。它限制由 Host 发出的连续刷新尝试，不
声称约束 provider 每次 load 自行启动的任意数量后代，也不声称远端凭据因此被吊销。

重启和失败按下表处理，每次独立重试至多一次 spawn，不在一次重试内部循环拉起 Agent：

| 阶段 / 结果 | 行为 |
| --- | --- |
| 准备完整目标 | 先取得精确租约并准备 view/解析；失败 blocked，保留旧进程，不部分切换 |
| 等安全点和停止旧 Agent | 当前 prompt、权限请求及 barrier 先结束；只停自己的 Agent；未确认停止不释放其租约、不启动替代 Agent |
| 旧 Agent 已停 | 释放它独占的旧版本引用；探测/兄弟仍使用的版本由各自持有；保留 execution、Session、provider ID、checkout、HOME/XDG 与历史目录 |
| 新 Agent spawn 失败 | 记录未启动/停止证据，blocked；下一轮可重新走一次 spawn + initialize + load，不要求对不存在的进程 load |
| initialize / agent_ready_timeout，或 load 前崩溃 | 保留原 provider ID，先停止并结算这次新尝试；有证据后下一轮至多再 spawn 一次；停止未知不叠加第二个 Agent |
| load 返回失败但新 Agent 仍可用 | channel 为空、blocked；下一轮复用该进程 load 原 ID；若该进程不可用，则按上一行在确认停止后 spawn + initialize + load |
| load 成功 | 仅目标仍为最新时 current；旧目标成功不能解除新目标阻塞，继续欠账；仍禁止 new/fork/transcript 迁移 |

每次 spawn 都用 D5 的“锁已刷盘 → 登记 → 加锁派发”，因此记录后、spawn 前崩溃能由空闲锁结算。
`AgentProcessReplaced` 和任何重启失败都遵守 D4 的保留身份策略，不能清成 Inactive 后借通用 attach
重建。原 ID 不存在、不支持 load 或状态目录丢失时 blocked，由用户修复/结束，不静默新建。

目标合并的边界是**新 Agent spawn 前最后一次串行读取 desired**：V2 已准备、旧 Agent 正在停止
时 V3 到达，丢弃未派发 V2 的租约/view，重新准备 V3；V3 无法解析则 blocked，不启动 V2。若 V2
Agent 已派发才收到 V3，不能直接删除 V2 正在用的 view：先按 D5 停止该尝试、确认后释放，再按
最新目标准备。一次重试不因目标变化多次 spawn，剩余目标排到下一次收敛尝试，等待截止不续期。
只存旧使用集合、一个在切换的集合和最新无值唤醒；未开始的中间目标不取额外租约。

模型授权与 ora-git socket 是 execution 生命周期资源，跨 Agent 重启保留，由同一个 owner 继续
监控撤销/结束。新 Agent 使用原冻结路由和**同一有效授权环境**、同一 HOME/XDG/history 与 socket
地址，不额外申请模型 grant、不用 MCP generation 轮换模型令牌。spawn 前再次确认授权仍有效；
撤销/到期按原模型访问结束策略处理，不能从缓存环境复活授权。只有整个 execution 结束才关闭
ora-git/model access，不能套用当前 driver 在退出 manager 分支时立即关闭的路径。

Node 同一 execution 最多持有旧 Agent 的集合与一个正在准备的集合（各 16 成员），另有至多 2 个
探测的旧版本引用。Separate 的 view 按 `(canonical ID, exact version)` 区分，硬链接要求见 D5；
Shared 使用精确 installed 目录、无私有 view。删除逐使用者结算，不能因 Agent 停止删掉仍在探测
使用的 placement。包升级/定期重启仅给出 BestEffort 覆盖进程的退出证据，不能等同于旧远端凭据
失效；只有 ACP 成功更不能证明旧注册消失。

### 选择依据：固定真实 Agent 的跨进程恢复

以保留相同私有 HOME/XDG、checkout 和 provider 历史为前提，真实 CLI 的“new → 首轮完成 → 旧
进程退出 → 不同 PID 初始化 → load 同一 ID → 第二轮完成”已得到以下必要前提证据：

| 固定 Agent 版本 | 观察 |
| --- | --- |
| OpenCode 1.19.3（Ora Agent 包 0.6.11） | 声明 loadSession；新进程 load 成功，两轮均 end_turn；第二轮模型输入含第一轮用户标记和 Agent 回复 |
| claude-agent-acp 0.99.1 + Claude Code 2.1.312（Ora Agent 包 0.4.5） | 新进程 load 返回原 session ID，两轮均 end_turn；第二轮模型输入保留同样的历史 |

这是 Linux 上真实 Agent 的 ACP/持久历史行为，模型使用本地确定性 OpenAI/Anthropic fixture；
mcpServers 为空，未验证 Node 重启接缝、实际包升级或工具重连。它支持选用保留身份重启，不能
推广为所有 Agent/版本都能恢复。没有持久历史或不能跨进程 load 的 provider 在需要重启时会 blocked；
UI 对已知此类 Agent 的包升级及达到刷新阈值的重启提前提示内存状态可能丢失，不能承诺自动新会话。

这些修改不改变 Desktop 安装器激活语义。本文批准后再为 plugin-runtime 根决策补历史指针；本 PR
只在 README 新增本 ADR 的 proposed 入口，不改前序两份 ADR 的状态。

## D7：探测绑定当前解析产物，明确执行身份与 HTTP 边界

Node 对当前 desired 中 configured 且静态有效的成员使用共享 resolver 的同一 transport 绑定。
身份包含 NodeRuntimeIdentity、execution、generation、canonical ID、包版本、configuration revision、
transport 和实际探测身份。stdio 额外绑定 workload UID/GID 与 **Node 从 checkout 解析的真实绝对 cwd**；
探测进程的 current_dir 就设为该 cwd，workspace 占位参数也用它。缺少真实上下文时不 spawn，
显示 Unknown(context_missing)，不继承偶然的 Node 工作目录。ACP 不承诺 provider 的 MCP 子进程
cwd，因此该结果仅证明这一个 Node 上下文，不声称复现了 Agent 内部执行。

HTTP 握手在 **Node 进程的网络身份/namespace** 中执行，显式禁用 reqwest 系统代理自动发现，
忽略 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 及其小写形式（无代理时 NO_PROXY 也不参与）。需要
企业代理的 endpoint 在本版不受支持，不隐式把秘密交给 Node 环境中的代理；此选项同样作用于
共享客户端的 Desktop 探测，列为兼容风险。不伪称按 workload UID 发出；headers/URL
仍来自同一解析输入。不继承 Node 管理认证、cookie jar 或额外管理请求头。workload 的 UID 防火墙、
代理或网络配置可能不同，Host Healthy 不能外推为 Agent 可达。stdio 使用对应 Agent 的 workload
身份及受控基础环境，叠加本次 env，不继承 Node 管理密钥或全量环境。

新 desired 解析成功及 session/new/load 完成时，为 Unknown 成员异步补探测；同身份 single-flight。
手动 `probe_attempt` 范围 `1..=2^53-1`，按当前 generation/成员递增，快照暴露最高受理值：相同
attempt 返回 duplicate，较小为 stale；较大在已有 flight/排队意图时加入，不重复外连，否则新建
意图。自动触发用 0，不覆盖手动结果。当前缓存至多 16 成员，另有有界旧 flight 清理责任；新 generation
不复用旧健康。无 TTL、周期重测或自动失败重试；未选择/未配置成员不能探测，安装本身不取得会话凭据。

握手完成 `initialize`、`notifications/initialized`、`tools/list`，不调用工具，不用 GET 或只有
initialize 的成功替代发现。一次 tools/list 响应就是本次证据，不无界追随分页。

共享 `ora-utils` HTTP 客户端显式关闭**全部跳转**，包括同源、跨源和 DELETE；每次 3xx 按所处
握手阶段失败，绝不向 Location 发出后续请求，任意自定义秘密头也不会随跳转外发。响应流在读取
过程中累计限 **1 MiB**，超限立即终止，不在 `.text()` 分配完整 body 后才检查；JSON/SSE、错误体
及不需正文的回复都不得无界读取。stdio 单帧与一次探测累计读取也限 1 MiB。超限/3xx 使用既有
封闭的 handshake/tools 阶段失败码，不回传 body 或 Location。此安全行为同时影响 Desktop 的
HTTP 探测，须覆盖回归；Desktop 的选择、刷新策略和健康三事实语义不变。

结果只有 `Healthy`、`Unhealthy{code}`、`Unknown{not_probed|context_missing}`，另用 queued /
probe_in_flight / cleanup_pending 描述资源状态。身份失效立即使旧结果不可见；晚到回调不能覆盖
当前成员。探测失败不改 desired/active，不阻塞 prompt，停止责任异常则独立按 D5 处理。

## D8：探测从取得许可起计时，预算耗尽不遗弃清理

Node 最多 8 个探测 flight，同 execution 最多 2 个；cleanup_pending 继续占许可，恢复出的未确认
探测也计入额度，不能以重启刷新额度。每个当前成员至多一个无值等待意图，以轮转方式跨 execution
调度；不留重复请求等待者或旧 generation 队列。排队期间保持 Unknown/queued，**不启动 10 秒预算，
不判 Unhealthy**。16 个成员在 2 个许可下依次探测，末尾成员不会因排队已过 10 秒而直接失败。

取得许可后总预算 **10 秒**，覆盖 spawn、完整握手和正常清理：最多前 8 秒握手，至少最后 2 秒
保留关闭 stdin、终止原进程组并 wait；HTTP 同样关闭本地流/连接并有界尝试会话 DELETE。stdio
探测使用独立进程组，不加入 Agent 组；探测结束不能杀 Agent 或兄弟会话。launcher 和清理任务由
Node 生命周期持有，断线、取消查询、丢弃 future 均不转移这项责任。

只有握手成功且本地清理完成才报告 Healthy。超时、失效、EndSession、Node 正常停止均触发清理；
预算内停止未确认则按时给出 `mcp_probe_timeout`、cleanup_pending，保留名额/租约/view 并登记 D5
的责任，不再为同身份开第二个 flight。2 秒是本次探测的清理预算，D5 的停止证据等待上限是独立
的有限对账周期，二者不重置。等待上限后停止后台等待，不自动释放许可；运维可按 D3 看见容量
耗尽及命名原因，显式对账取得原证据后才能恢复。

实现要验证直接子进程和同组后代退出，不能只断言函数返回 timeout。首版明确只提供进程组
BestEffort；恶意 setsid/脱组、无法确认的内核状态及宿主崩溃后的未知证据不被说成 Strong。
需要 cgroup/subreaper 或其他强纳管的保证另立后续决策，本次不以“整棵树清理”宣传这项能力。

## D9：三层状态按证据展示

| 事实 | 权威证据 | 不代表 |
| --- | --- | --- |
| 已配置 | Controller 保存的选择/配置状态，Node 仍可能发现缺包或解析失败 | 已送达、可连接 |
| 已送达 ACP | 对原 provider ID 的完整 setup/load 成功及 active generation | 当前 desired 已收敛、旧注册/凭据/进程已退出、工具已调用 |
| 宿主健康 | 当前匹配身份的完整 Node 探测及 BestEffort 本地清理 | Agent 已连接或调用成功、整棵进程树静止 |

只有 active 匹配当前 desired 且 current 才显示“当前配置已送达”；其他状态可显示“上次送达”
但不能冒充当前可用。页面重载保留 S，重新取得 Active/健康。缺新能力按 D1 拒绝 v2 带 MCP 派发，
旧执行只显示未支持/未知。刷新失败引导修复或下一轮重试，轮次等候超时显示稳定失败；探测失败
可手动重测，排队/容量不可用与服务失败分开。停止未确认显示 stopping 与残留责任，不能显示已停止。

展示仅使用成员 ID、setting ID 和稳定码，不显示值或第三方文本。读会话的授权不等于修改凭据
或发起带凭据探测的授权；所有请求仍检查原 execution/control scope。

## 为什么不是这些替代方案

| 方案 | 优势与取舍 |
| --- | --- |
| Controller 重新解析并探测 | 接线较少，但无法证明 Node package view、workload 身份和实际绑定一致 |
| 重发 start 或结束 execution 再新建 | 前者不更新值，后者丢失执行和会话身份，不采用 |
| 同一 execution 内重启自己的 Agent | Node 没有 Desktop 的共享 Agent 代价；包版本变更采用，保留 provider ID，但有重启延迟和内存状态损失 |
| 每次配置更新都重启 Agent | 更快回收旧注册，但每次保存都付出重启成本；选择最多 8 次运行中刷新，第 9 次转重启，保留 BestEffort 边界 |
| load 成功即删旧包 | 节省空间，但 ACP 不证明旧使用者已退出；改用 Agent 停止证据和每个探测的独立租约 |
| 更新进入 durable queue | 易于重放，却把 Settings 写入 WAL；本方案只持久化无值身份、轮次结算与停止责任 |
| 用健康事件证明已送达 | 可历史查询，但健康与 ACP 独立，且跨重启会过期；保持内存观察 |
| 清理超时就发终态并释放 checkout | UI 容易收敛，但残留进程仍可写树；采用有限等待后命名转交，终态留待证据 |
| 默认 stdio spawn / 默认 HTTP 重定向 | 改动小，却可能以 Node 管理身份执行或将自定义秘密头交给跳转目标，不采用 |

## 不变量

1. S 不随挂接、刷新或 provider 恢复扩大；运行中缺授权元数据时停止派发并结束。
2. Update 与 Actor 最终准入有确定顺序；非 current 不准入新 prompt。
3. Settings 只在受权内存与必要外发中，不随停止责任、事件或诊断持久化。
4. MCP 非终结拒绝不结束 execution；轮次只在真实准入后结算 Executed，等候超时可靠报告 Failed。
5. 刷新/进程替换失败保留原 provider ID；重试最多一次 spawn 后 load；升级和配置刷新阈值只重启本 execution。
6. 超时、断线或任务结束不构成停止证据；未知责任保留精确包、view 与 checkout 围栏。
7. 停止尝试锁先刷盘再登记/派发；重启先核对原锁，不能刷新额度、伪造停止或复活 Agent。
8. 探测预算从许可开始，排队不是不健康；探测与当前身份不符则不能写回。
9. ACP 接受、Host 健康和 BestEffort 清理互不替代，不宣称旧凭据必然撤销或 Strong 静止。
10. 终态之后没有事件；stop_unconfirmed 不伪造终态、不恢复分支、不交付，普通共享仍按原数量准入。

## 验收场景与评审重点

本 ADR 尚未 approved，以下是可观察场景；此阶段不新增核心测试用例或宣称已有实现覆盖。

- v2 连接只有 session_mcp 的旧 Node，非空选择拒绝，空选择仍可启动；重载保留 S，运行中丢 S 后
  停止派发并 EndSession。重发原 command 保留原 generation，拒绝 mcp_update_required 后不能记已送达。
- idle/busy/权限等待/已排队轮次与两次 Update 交错；以 Actor 准入排序，不能提前 Executed 或误报
  agent_unavailable。解析失败也 blocked，ACP 失败隔离后无 channel，下一轮只 load 同 provider ID。
- 120 秒无活动窗口与 session update 重置；barrier 等待独立计时；300 秒轮次
  截止不因更新续期，慢重启可耗尽它；blocked 无轮次不触发 Node 自有结束时钟。EndSession 与 Failed
  结算竞争时只有一种结果，无终态后事件。
- 版本升级停止自己的 Agent；兄弟会话不受影响。stop 失败保留精确租约/view，不启动替代 Agent，
  stop_unconfirmed 只阻止 prior_revision 恢复/交付；正常停止的 A 不挡普通后继 B，仍检查 8 会话
  限制。Agent/探测带继承锁重启收口，空闲锁迅速确认，迟到证据才补终态、释放资源。
- spawn 登记前后、进程启动闸门、stop 锚定及确认提交前后崩溃；未知身份不能伪装成未启动；PID
  复用不能让新进程被误杀；子进程未继承 fd 不外推 Strong。Shared/Separate 均覆盖版本 view，
  跨设备部署及 EXDEV 必须拒绝。第 9 次配置刷新走重启，失败调用也计数，普通 prompt 不重置。
- 新 Agent spawn/initialize/load 前崩溃，旧 Agent stop 未确认，V2 停止中到 V3、V2 已启动后到 V3，
  分别验证至多一次 spawn、未派发租约释放与已派发责任保留；模型授权和 ora-git 只随 execution 结束。
- 本地运维 recheck/after-host-reboot 的权限、幂等和审计；同 boot 缺锁不能清标记，确认记录 GC 不
  丢未确认责任或未完成结算，旧 schema/备份回滚被拒绝。
- 16 成员、2 个 execution 许可、8 个 Node 许可：排队不 timeout；8 个 cleanup_pending 耗尽容量
  可见，重启不能恢复假额度。重测 attempt 丢回复/Controller 重启先查询，旧 flight 不覆盖新身份。
- HTTP 307 + X-API-Key（含同源/跨源/DELETE），验证没有第二次请求；超 1 MiB、SSE 无结束、stdio
  超长帧、401/5xx、tools/list 失败及挂死；不发 tools/call，验证直接 child 与同组后代清理。
- 特征 Settings 追踪覆盖控制帧、ACP、探测以及 ledger/WAL、JSONL、日志、停止责任和页面；值只出
  现在允许的内存/外发，停止记录不能保存 resolved URL/argv/env。
- 配置成功、帧写出、ACP 拒绝、ACP 成功但探测失败分别注入；Desktop 原默认 attach 策略不变，
  共享 HTTP 不跳转/有界响应/忽略系统代理的行为变化有独立回归。

## 风险、资源边界与落地顺序

以下风险已按 owner 的复审结论纳入方案；细化后的停止/对账、计数和兼容行为仍待本轮批准：

| 项目 | 边界与承担方式 |
| --- | --- |
| R1 会话总数 | 接受无全 Node 上限；每 checkout 8 个，输入/Agent 数与最多 16N 个探测意图随 N 增长 |
| R2 版本/view | 接受旧/新集合约 32N 个引用加 Node 8 个探测引用；部署强制 workload、Node home、installed 包同文件系统并检验硬链接，EXDEV 拒绝，不完整复制 |
| R3 磁盘 | 接受插件根无全局配额；部署监测可用空间、在用/历史包字节和回收积压；空间不足拒绝安装，不删在用版本 |
| R4 同版本刷新 | 修改为每进程至多连续 8 次配置 load，第 9 次重启；已写出的失败也计数。仍不保证 provider 每次只起一个进程、关 fd/逃逸后代退出或远端凭据被撤销 |
| R5 残留责任 | 修正门禁和 attempt_lock 证据；只挡恢复/交付，不挡正常普通共享；本地可审计 reconcile-session-stop 提供 recheck 与宿主重启后确认，禁止无证据强制释放 |
| R6 纳管保证 | 接受进程组 + 继承锁的 BestEffort；不传 fd 的后代可能不在证据内，强 containment 后续决策 |
| R7 Agent 恢复 | 接受重启丢失仅驻内存状态；固定 OpenCode/Claude 版本已证明保留 HOME/cwd 后跨进程 load。其他不支持者在重启时 blocked，升级/阈值重启前 UI 提示，禁止 new fallback |
| R8 HTTP | 接受 Node 网络身份及不跳转；明确共享客户端忽略系统代理，需要代理/重定向的 endpoint 将失败，Desktop 同受影响 |
| R9 无终态 | 接受极端缺证据时 execution 永远没有终态；Node 有限等待转交，Controller 业务收尾保留残留，不能凭时间补造停止 |
| R10 记录增长 | 已确认停止明细 7 天后每小时分批删除/压缩，审计至少 90 天；不删未确认/仍被引用的记录，监控积压，不借 GC 裁账本事件 |
| R11 重启次数 | 接受版本变更导致的重启次数不设累计上限，Controller 串行 generation、Node 合并未派发目标；每次独立重试至多一次 spawn |

另外接受 300 秒轮次截止涵盖完整重启，可能让正常慢 Agent 的这一轮失败；它不延长为无限恢复等待，
也不强制结束 idle blocked Session。初始轮次同样受此约束。停止证据改善例行重启的正常收口，
却不提升为强纳管。Node 全局配额、强 containment、历史健康审计、跨 Node 崩溃恢复 Agent 及实际
工具可见性均不在本文；所述真实 Agent 前提验证不代替产品实现证据。

落地分三步，代码义务由 Node 产品实现运行承担，不转嫁给仅实现 v2 UI 的运行：

1. 完成本文复审后标 approved；再在镜像目录登记核心用例为 Missing，并为被修改的 plugin-runtime、
   会话命令及停止/共享 checkout 契约补历史指针。不在本 PR 提前改其他 ADR 的 implemented/README。
2. Node 产品 PR 同批交付下表责任并通过边界测试，全部具备才广告新能力；Controller/v2 同版本接入
   轮次拒绝/失败、generation/attempt 对账、原 IdleTimeout 策略、重启风险提示和能力门禁。v2 的选择恢复/状态展示可独立
   先行，但不能提前宣称刷新/健康已完成。

   | 责任面 | 必须新增的行为 |
   | --- | --- |
   | `crates/agent-runtime` 的 actor_mcp / cleanup / attach / live state | Node 宿主策略保留 Blocked 与 provider ID；resolver 失败纳入状态机；只 load、无 new fallback；有类型的非终结准入结果与通知；版本切换的独占 Agent 重启接缝 |
   | `apps/ora-node/src/session/driver.rs` | 在 settle Executed 前等门禁及 Actor 确认；300 秒 Failed 结算；失败路径每尝试至多一次 spawn、8 次配置刷新计数、保留模型授权/socket；停止失败移交资源 owner，不能直接 drop placement/leases |
   | `crates/node-protocol` / `crates/node-db` / Node service | 新消息和条件必填 generation；按原因区分投递拒绝；可靠 SessionTurnFailed；新增 session_stop_attempts / audit、attempt_lock 派发与前向迁移；stop_unconfirmed 围栏只限恢复/交付，可审计对账和确认记录 GC |
   | Node catalog / `session/workload.rs` | MCP 精确版本租约、按 id/version 的 Separate view、Shared installed 路径、同文件系统硬链接门禁、Agent/探测各自持有与迟到释放 |
   | Node probe owner / `ora-utils::mcp` | 8/2 许可、从许可起 10 秒、独立 workload launcher 与确定 cwd、停止责任和容量诊断；共享 HTTP 禁跳转、禁系统代理与流式响应上限 |

3. 产品实现评审通过后另开落地运行，以实际实现修订 ADR 并回填直接测试证据；产品 PR 合并后再合并
   implemented 状态。保留旧模式冻结 setup、独立会话身份、Node 不自动恢复 Agent 和可靠事件承诺，
   不把 proposed 的资源取舍或尚未验证的停止行为称为已实现。
