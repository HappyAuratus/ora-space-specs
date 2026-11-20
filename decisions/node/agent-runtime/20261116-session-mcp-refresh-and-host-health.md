---
status: implemented
---

# Node 保留会话身份刷新 MCP，以有界准入和停止证据隔离故障

> 历史决策：D5 的停止证据与重启收口表、D6 的 Agent 替换、D8 的纳管范围表述、不变量 7 与 R6，已由
> [会话尝试的进程带环境标记](20261120-attempt-mark-in-session-stop-evidence.md)（`approved`）修改：Linux 上尝试的
> 每个进程带 `ORA_SESSION_ATTEMPT`，停止前冻结并杀掉带标记的进程，两次扫描干净才确认并记
> `lock_processes_and_marks`。这些部分以该决策为准；它的实现合入前，已实现的 Node 行为仍是本文描述的版本。

Controller 持久保存 Session 的显式 MCP 选择，向 Node 提交完整内存配置。Node 在安全点收敛：仅配置
变化时用原 provider ID 执行 ACP `session/load`；包版本变化时先停止本 execution 的 Agent，再启动
它自己的 Agent 并 load 原 provider ID；同一 Agent 进程已写出 8 次运行时 load 后，下一次也走重启路径。
刷新失败阻塞新 prompt，下一轮自动重试，始终禁止回退到
`session/new`。配置完整、ACP 接受 revision、Host MCP Health 分别取证。

本决策的 Node 侧实现已随 [desktop#776](https://github.com/ora-space/desktop/pull/776) 合入
`28da0a4e77ec2257468b15bffd99e166a186b31c`，状态为 `implemented`。该状态表示 Node 的协议、
准入、停止责任、版本租约和探测机制已经落地，不表示 Controller/v2 消费端已经接线或所有验证义务均有
完整直接证据。核心用例逐项保留 Covered / Partial / Missing；Node 侧验收通过不外推跨仓库端到端通过。
Cloud v2 继续在启动前拒绝非空 MCP，直到消费端实现 generation、可靠轮次失败和状态查询协议；
仅升级 Node 二进制不解除此门禁。D5 的停止移交、持久证据、终态前提及门禁适用于
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
| Node 重启不重启原 Agent，结算 interrupted | 继承不恢复 Agent；修改停止前提：有未确认 Agent/探测责任时，先恢复围栏，取得原停止证据后才补报 interrupted；未排空升级的旧会话标 legacy_unknown，仅凭同机重启证据或部署的单点挂载保证结算（D5） |
| [工作流显式选择](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) | 继承 Session 自身保存白名单；运行中丢失白名单时停止派发并 EndSession，不从 Node 观察重建授权 |
| [ACP setup/refresh](../../desktop/core/agent/session/mcp/0-acp-session-setup-delivery.md)：完整快照、barrier、失败后下一轮先重试 | 修改 Node 宿主策略：隔离后保留 Blocked 和原 provider ID，仅允许 load；不沿用现有 attach 失败后 new 的路径；Desktop 策略保持原状 |
| ACP refresh 的预算和进程边界 | 继承有 MCP 时 120 秒无活动窗口；修改 Node 包升级：允许同一 execution 内重启其独占 Agent（D4、D6） |
| [Host 健康探测](../../desktop/core/agent/session/mcp/20260916-mcp-health-probing.md) | 继承完整握手、三事实分离、无值内存结果；Node 只探测已授权 Session；共享 HTTP 客户端增加不跳转与有界读取 |
| [插件安装](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md) D2/不变量 4：按 ID 使用期间拒绝替换/移除 | 修改 MCP 为精确版本租约，可并存不可改写；其他类别保持原规则（D6） |
| 共享 checkout D2：恢复尝试独立阻止复用 | 既有恢复围栏不变；新增 Agent/探测的 stop_unconfirmed 及升级遗留 legacy_unknown 围栏只挡 prior_revision 恢复和该 execution 自己的交付，普通新会话仍按共享规则及数量准入（D5） |
| [停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)及[未启动证据](../revision/20261021-stop-confirmation-accepts-not-started.md)：超时不是停止证据 | 继承命名原执行责任、有限等待、跨重启不续期与迟到确认；不恢复旧 host/guardian 架构 |
| [Git 尝试锁](../git-execution/0-node-runs-git-directly-with-attempt-locks.md)：attempt_lock 给出跨重启停止证据 | 扩展同一机制至 Agent/stdio 探测，独立目录与记录类型；Agent 尝试覆盖插件运行时及其全部宿主子进程，每个进程组一条线索；跨 boot 确认另需同一主机标识；不复活 host/guardian，不把未继承 fd 的后代算入证据（D5） |
| 范围决策中的“进程树清理” | 明确收窄：首版只负责直接子进程及仍处于原进程组的后代，以 `StopContainment::BestEffort` 报告；不声称 Strong 或整棵树静止（D5、D8） |

## 问题与约束

本决策之前的启动快照不可更新，刷新失败后的通用重建策略也不能满足 Node 保留 provider 身份的要求。
因此 Node 使用独立的宿主策略，并把 Actor 最终准入、轮次结算与停止责任分开拥有；内存配置源本身
不构成完整生命周期，停止失败也不能触发租约或 view 的提前释放。

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

消息语义如下，均沿用 `protocol_version`、`operation_id`、`execution_id`，请求另有
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
  这些 load（含重连和下一条的重试）都计入 D6 的每进程运行时 load 计数，计满后改走重启路径，
  仍只 load 原 provider ID。
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
load；不等进入 load 才开始。**owner 已决定接受**正常但较慢的重启使已排队轮次（包括初始轮次）
到期失败：它们以稳定码失败、可以重发，慢重启本就少见；否则反复启动失败会让 O4 的等待上限失效。
会话仍可随后恢复，用户以新 turn 重试，旧轮次不复活。

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

复用 Node 的 `managed/attempt_lock` 机制（锁归属打开的文件描述而非进程，最后一个持有者退出时
内核释放：`apps/ora-node/src/managed/attempt_lock.rs:1-12, 141-156`），Agent/stdio 探测使用 Node
home 下独立的私有 `session-attempts` 目录，避免和 Git 尝试的扫描/退休职责混淆。锁文件及目录先
fsync，再持久登记尝试，然后加独占 flock 并派发；回收不能与派发竞争，把尚未派发的记录提前判为
已停止。锁 OFD 在 Node 内保持 close-on-exec，只经 `ProcessSpec::inherit_descriptor` 交给本尝试
指定的子进程，其他线程并发 spawn 的进程不会继承（`crates/process/src/spec.rs:172-182`，
`crates/process/src/tokio_process.rs:152-155`）。

**一次 Agent 尝试 = 插件运行时 + 这一代插件请宿主 spawn 的全部子进程。** Agent 插件不在自己的
沙箱运行时里起 CLI，而是经 `ora/childprocess/*` 请宿主 spawn，进程归宿主所有
（`crates/plugin-lifecycle/src/childprocess.rs:1-10`；只有 Agent 插件获得此能力：
`crates/plugin-lifecycle/src/runtime.rs:149-161`）。宿主 spawn 的每个进程是 Node 的直接子进程且
各自成组（`crates/process/src/tree.rs:8, 76`；`crates/process/src/reaper.rs:233`），插件运行时与
它请求的子进程共用同一个会话 spawner（`apps/ora-node/src/session/process.rs:3, 33-45`），而且
不登记 reaper，Node 异常退出后它们照常存活（同处 35-36 行）。因此：

- 该 spawner 为本尝试派发的**每个**进程（插件运行时、Agent CLI 及其他宿主子进程）都传入同一个
  锁 OFD；Node 在整个尝试期间保留一份副本，以便后续宿主 spawn 仍能继承。请求停止时，Node 先
  拒绝该代插件之后的宿主 spawn，再关闭自身副本，然后才开始探查锁；Node 崩溃时内核随进程释放
  它的副本。
- 每个进程组在其 spawn 时各记一条 pid/start_ticks/boot_id/主机标识线索，同一尝试可有多条线索；
  终止和重启后的直接进程检查覆盖全部线索，不只插件运行时。
- `kill_all` / `stop_plugin` 的返回结果只用于诊断，不构成停止或未停止的证据。停止责任 owner
  始终核对原尝试锁与全部直接进程线索：请求返回成功不能提前确认，请求返回错误也不能跳过原期限内
  的证据等待。若随后取得真实停止证据，允许确认；否则保留责任、租约和 view。
  这修改了 approved 版本把 `kill_all` 返回纳入停止证据的约定：请求结果与进程实际存续并不等价。
- stdio 探测只有 Node 的一次直接 spawn，一个进程组、一条线索，派发后 Node 关闭自身副本，按原写法。

每个 Agent/stdio 尝试新增一行 `session_stop_attempts`，含原 Node 实例、boot_id、主机标识、
execution、checkout、attempt ID、kind、精确包版本/view 归属、状态及首次停止/截止时间。**新增专用表，不把
MCP argv 塞入 Git 的 `process_attempts` / `managed_executions`**：后者及现有 `AttemptCommand`
仍归 Git；复用的是尝试身份/锁机制。HTTP 本地任务也登记同表的独立 kind，由 Node 持有的锁 fd
覆盖本地任务存续，不能把远端服务状态记录成进程退出证据。记录不含 argv/env/URL/headers/Settings
或其摘要。schema 及序列化变更通过 Node DB 前向迁移。

线索在 spawn 后旁路写入，仅用于安全终止对应进程组，不能代替锁证据。PID 复用、boot 或主机标识
不符、线索不可读时不盲杀。Agent CLI 及适配器必须保有被授予的 fd 到其退出；若 CLI 启动 MCP 子进程
时未继续传递 fd，锁只覆盖真正持有 fd 的进程。各线索的直接进程退出与对应进程组清理也要观察；从不
把一个提前关闭 fd 的活进程判为停止。进程组加锁仍是 `StopContainment::BestEffort`，不能宣称逃逸/
关 fd 后代已退出或 Strong 静止。

这份证据**假定 workload 配合**：flock 属于打开的文件描述，任何持有继承 fd 的进程（包括以 workload
UID 运行、由模型驱动的 shell 命令）都能 `flock(fd, LOCK_UN)` 在进程仍存活时释放锁，或关闭 fd 后
脱组。Git 是可信的固定程序，Agent 不是；对恶意或失控 workload，锁和线索都只是 BestEffort，
不能作为对抗性隔离的证明。

Node 重启后先持有数据库独占权并恢复责任，再按下表顺序作有界证据检查。flock 和 PID 只在原内核上
有意义：boot_id 每次启动随机生成，记录的 boot_id 与当前相同即同一内核、同一台机器，锁与线索
可直接核对；boot_id 不同时，锁在当前内核上必然空闲，不能单独作为证据。

| 原尝试证据 | 处理 |
| --- | --- |
| boot_id 相同，原锁可取得，且该尝试全部线索的直接进程都已不在 | 按 BestEffort 确认已停止或从未派发，正常立即结算 interrupted；不人为等满停止期限（容器或 Node 进程重启的常见情形） |
| boot_id 相同，原锁被占用，或任一线索的直接进程仍存活 | 按各条已验证线索尝试终止对应进程组，再查锁/直接进程；没有取得停止证据前保留原责任 |
| boot_id 相同，锁缺失、损坏或不可读 | Unknown，不能把无 PID 或无文件当成未启动；转交对账 |
| boot_id 不同，且记录的主机标识与当前相同 | 同一台机器重启过，原内核的进程不可能存续，可记录宿主重启证据确认；这和仅 Node 进程重启不同 |
| boot_id 不同，主机标识不同、缺失或不可信 | 只说明当前内核不是原内核，不能证明原主机已停；按 Unknown 交给 `recheck`/对账，除非部署满足下文的单点挂载保证 |

主机标识是承载原内核的机器的稳定 ID（Linux 上为宿主的 `/etc/machine-id` 或部署提供的等价主机 ID）。
Node home 及本目录随 Workspace 数据卷跨沙盒保留
（[插件安装](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md) D1），换主机后 boot_id
必然不同；现有尝试锁也从不把 boot 不同当作停止证据，只据此拒绝发信号
（`apps/ora-node/src/managed/attempt_lock.rs:158-170`）。容器镜像自带、各实例相同或为空的
machine-id 不算主机标识；部署无法提供时，跨 boot 自动确认不可用。只有部署明确保证 Workspace 卷
**只能单点挂载，且重新挂载到另一主机前已在存储层隔离原主机**，才可把“主机不同”等同于原主机的
进程已无法触达该卷；这一保证属于部署契约，Node 不自行推断。

因此“记录已写、尚未 spawn 就崩溃”留下已刷盘且空闲的锁，能证明未启动/已停止，不造成假门禁；
Agent 尝试派发后 Node 仍持有副本，Node 崩溃即随进程释放，剩下的持有者只能是继承了 fd 的进程。
已派发但未写 PID 线索时仍由继承 fd 证明存续，缺线索只限制终止手段。停止请求先持久锚定
first_stop_at_ms / bound_at_ms，再发信号，跨重试/重启不续期。原状态还没有停止时钟的重启收口只
锚定一次，不把每次重启都当作新的等待周期。

### 终态、转交与门禁

Node 生命周期 owner 在 driver 返回前接管未清理 Agent/探测、catalog 租约和 placement/view；
`stop_plugin` / `kill_all` 错误只记诊断，不能据此 drop 资源或立即标记 `stop_unconfirmed`；
owner 仍按首次持久化期限等待原锁与线索。停止一旦请求，该代插件后续的宿主 spawn 一律拒绝，避免 Node 关闭
锁副本后再起不持锁的进程。更新中的旧 Agent 未确认停止时保持 blocked，不启动替代 Agent；结束
execution 时 stopping，不再接受轮次/探测。HTTP 停止只确认本地任务/流/连接已结束，远端 DELETE
仍是 BestEffort。

1. EndSession 先停止准入、取消探测并请求停止 Agent，正常回复持久受理；不阻塞心跳或兄弟会话。
2. 只有本 execution 全部 Agent/探测责任已取得停止确认，才封口并发唯一 AgentSessionEnded；
   对应使用者的租约/view 在其停止确认后释放，不必等其他使用者。Completed 不带 stop_unconfirmed。
3. 首次停止请求后按既有停止确认策略有界等待（默认 10 分钟，可配置为正有限值）。超过上限，
   或 Node 重启收口无法取得原证据，明确登记 stop_unconfirmed，结束等待任务并转交残留对账。
   只有这些状态及下文升级迁移标出的 `legacy_unknown` 停止责任触发新增残留门禁；“正在运行”
   或“期限内正常停止”都不是 stop_unconfirmed。
4. GetExecutionStatus 返回原 `StopUnconfirmed { scope_id, run_id, first_stop_at_ms, bound_at_ms,
   guarantee }`；scope_id 绑定原 execution，run_id 绑定原 attempt。有限的后续对账取得原证据后
   才补终态；首次原因、身份、序号不变。可能始终没有终态，Controller 依现有业务窗口收尾并保留
   残留责任，不能把业务结束或等待到期当停止证明。

同一 checkout 上新增 Agent/探测门禁与旧规则的关系如下；事务内判定，日志不可读时不能作出
“无残留”的肯定判断：

| 责任状态 | 普通新会话（无 prior_revision） | prior_revision 恢复 / 交付 |
| --- | --- | --- |
| 活跃 Agent/探测，或期限内正常停止 | 不增加 checkout_busy；未终态 execution 仍占原 8 个名额 | 恢复仍受原“有未结束会话不可恢复”规则；交付仍需自身会话已结算 |
| stop_unconfirmed（期限已过或重启收口缺证据），或升级迁移标出的 legacy_unknown | 不增加残留拒绝；仍占未结束名额并按共享规则准入 | 恢复拒绝 checkout_busy；**该 execution 自己的**交付拒绝 session_not_settled，直到原责任被证据解除 |
| 已确认停止并完成责任交接 | 按原共享准入 | 解除这一残留围栏，仍检查其他旧恢复/交付约束 |

交付列只指残留责任所属 execution 自己的交付：它没有终态，本就不是可快照的已结算会话，沿用
`session_not_settled` 语义准确。兄弟会话已经结算后的交付不因这份残留被拒绝，与
[共享 checkout](20261115-shared-checkout-agent-sessions.md) D5 接受交付混入兄弟写入的规则一致。

选择不挡普通新会话，是因为共享 checkout 本就允许其他活跃写入者；残留不增加恢复分支或快照
一致性之外的新承诺。A 在正常停止的数秒内不妨碍普通后继 B；带 prior_revision 的 B 仍按原规则
等 A 终态。**恢复既有的 `session_restores` legacy_unknown 阶段与 `restore_process_attempts` 门禁不变**，不能用
此新增规则放宽旧恢复围栏。残留独立于历史业务终态检查，不能靠只数未结束 execution 放行恢复/交付。

### 留审计记录的运维对账

新增 Node 本地管理操作 `reconcile-session-stop`，不放进普通 Controller/MCP 消息。入口是**运行中
Node 进程提供的本地 Unix socket**：位于 Node 私有目录，只有 root 能连接；Node 用 `SO_PEERCRED`
取得对端 UID/PID，只接受 UID 0，审计记录的操作者 UID 也取自它，不接受调用方自报。命令行工具只是
该 socket 的客户端，不另开数据库，与“Node 持有数据库独占权”一致；Node 未运行时没有这个入口，
重启后的自动收口先按上表执行。每次明确提供 execution、原 attempt、幂等 request_id 和
reason_code（维护收口、探测容量恢复或升级遗留）；不能批量按模糊 PID 或目录释放。支持两种证据模式：

- `recheck`：在至多 10 秒内核对原锁及该尝试全部直接进程线索，必要时按已验证线索终止对应进程组，
  再观察；得到 Stopped 才确认，否则返回仍为 Running/Unknown，不刷新首次停止时间，不释放许可。
  没有尝试锁的 legacy_unknown 责任无法由 recheck 取得锁证据；对它，recheck 只重新核对下文的同机
  重启条件，不能单凭“现在找不到进程”解除。
- `after-host-reboot`：只在自动收口没有确认时使用，典型是记录早于本次 Node 启动时的检查、检查当时
  证据不可读，或部署事后补齐主机标识/单点挂载声明。它按上表同一规则核对：记录的主机标识与当前
  相同且 boot_id 不同，或部署明确满足单点挂载与重新挂载前隔离；否则拒绝。主机不同、缺主机标识或
  同一 boot 上缺锁且无法证明退出都不能解除。永久不可中断的进程需运维在 Node 排空策略下重启宿主后
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
仅兼容输入，不豁免全会话停止保证。desired/active、健康及 Settings 不进这些表。

升级应先排空旧会话，因为旧会话没有继承尝试锁，不能为仍活着的旧 Agent 补造证据。**排空检查在
任何迁移步骤之前执行**，只读取旧 schema；但未排空**不拒绝启动**，避免一次意外的崩溃/OOM 重启
让 Workspace 再也起不来（旧二进制只接受 `user_version` 1..=13，见
`crates/node-db/src/schema.rs:44-45`，迁移后无法退回）。未排空时照常迁移，并在同一迁移事务中为每个
未结束的旧 Agent 会话 execution 写入 `legacy_unknown` 停止责任（`session_stop_attempts` 中的独立
状态，不是 `session_restores.phase`），沿用恢复迁移的先例：旧库缺少进程
关联时保留不确定性，不制造中断或安全交付（`crates/node-db/src/restore.sql:9-13, 24-28`；
[恢复截止与停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)）。
与恢复的 `legacy_unknown` 不同，这份责任有证据出口：

- 它不重启原 Agent，按上文门禁表挡住 prior_revision 恢复和该 execution 自己的交付，不挡普通新
  会话，并继续占原 8 个未结束名额；
- 迁移时把当前主机标识与 boot_id 记为这份责任的参照。只有之后的启动收口、`recheck` 或
  `after-host-reboot` 核实同机重启（参照主机标识相同、boot_id 已变），或部署的单点挂载保证，才把它
  结算为 interrupted；没有尝试锁，不能单凭“现在找不到进程”解除。同机重启只证明在本机运行过的
  旧进程已不存在，旧会话若可能在升级期间随卷迁移前的另一主机上运行，只有单点挂载保证能补足；
- 缺证据时它可能始终没有终态，按 R9 接受，并作为“升级遗留”进入运维监控。

已完成的旧 execution 只保留历史终态，不补登记责任；仍为 accepted、从未启动 actor 的旧 execution
不可能有 Agent 进程（同一先例，`restore.sql:14-15`），按原重启规则收口，不标 legacy_unknown。

## D6：版本变化及第九次运行时 load 重启自己的 Agent

MCP 租约改为 canonical ID + 精确版本；发布、取租约、校验与删除共用 catalog 同步边界。在用
版本不得改写/删除，新旧均为 MCP 才可并存，不能换 kind 绕过 Agent 保护。其他类别仍按 ID 保护。
整 ID RemovePlugins 或在用同版本修复继续返回 plugin_in_use。

版本改变必须重启本 execution 独占的 Agent。另设**每个 Agent 进程**的运行时 load 计数：该进程
写出的第一个 `session/new` 或 `session/load` 不计；此后写给同一进程的每个完整 `session/load` 都加一，
不论原因——仅配置变化的刷新、`ConnectionLost` 后重连或重新挂接时重发不变的 `mcpServers`、O1 对
失败刷新的下一轮重试，以及新进程首次 load 失败后在同一进程上的重试；失败响应也计，因为每次
写出的 load 都可能让 provider 再注册一次。重复 Update、未发送的 superseded 目标和解析前失败
没有写出 load，不计。**同一进程已计满 8 次后，下一次需要写出 load 时必须先走重启路径**，不允许
第 9 次运行中 load。计数随进程存在：重启后的新进程从 0 开始，旧进程停止确认前不启动新进程，
因此失败的重启不能凭空恢复额度。普通 prompt 不改变计数。它限制 Host 写给一个进程的 load 次数，
不声称约束 provider 每次 load 自行启动的任意数量后代，也不声称远端凭据因此被吊销。

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

每个新 Agent 尝试都用 D5 的“锁已刷盘 → 登记 → 加锁派发”，该尝试之后的宿主子进程继承同一锁并
各记线索，因此记录后、spawn 前崩溃能由空闲锁结算。
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
mcpServers 为空，未验证 Node 重启接缝、实际包升级或工具重连；旧进程在 `end_turn` 后以 SIGTERM
正常停止，强制 SIGKILL 时未落盘历史是否保留、新进程以变化的非空 `mcpServers` load、以及 Separate
placement 下的 workload UID 均未覆盖。测试记录与复现材料以
[circus #1204](https://github.com/ora-space/circus/issues/1204) 为准，不进入本仓库。它支持选用保留身份重启，不能
推广为所有 Agent/版本都能恢复。没有持久历史或不能跨进程 load 的 provider 在需要重启时会 blocked；
UI 对已知此类 Agent 的包升级及达到运行时 load 阈值的重启提前提示内存状态可能丢失，不能承诺自动新会话。

这些修改不改变 Desktop 安装器激活语义。批准时已为 plugin-runtime 根决策补 `> 历史决策：` 指针，
不改前序两份 ADR 的状态。

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
的有限对账周期，二者不重置；探测 owner 读取已持久化的 `bound_at_ms`，不以并发请求的本地计算值续期。
等待上限后停止轮询，保留 owner 和许可并等待确认通知；自动收口或 root 显式对账取得原证据后都必须
唤醒 owner，完成名额及精确版本租约释放。运维可按 D3 看见容量耗尽及命名原因。

探测 runner/owner task panic 或异常退出时，整个 Node fail closed，不把故障局限在单个 execution 后
继续服务。持久停止记录保留责任，Node 重启后先恢复围栏和占用的探测容量，再按原证据收口；不得因任务
消失而返还名额。选择这一策略是为了避免活着的 Node 留下无 owner 的清理责任；代价是其他会话也受
Node 退出影响。重启仍不自动复活 Agent，只恢复停止责任。

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
| 每次配置更新都重启 Agent | 更快回收旧注册，但每次保存都付出重启成本；选择每进程最多 8 次运行时 load（含重连与重试），第 9 次转重启，保留 BestEffort 边界 |
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
5. 刷新/进程替换失败保留原 provider ID；重试最多一次 spawn 后 load；升级和运行时 load 阈值只重启本 execution。
6. 超时、断线或任务结束不构成停止证据；未知责任保留精确包、view 与 checkout 围栏。
7. 停止尝试锁先刷盘再登记/派发；Agent 尝试的插件运行时及全部宿主子进程继承同一锁，Node 副本保留到请求停止；重启先核对原锁及全部线索，跨 boot 只认同一主机，不能刷新额度、伪造停止或复活 Agent。
8. 探测预算从许可开始，排队不是不健康；探测与当前身份不符则不能写回。
9. ACP 接受、Host 健康和 BestEffort 清理互不替代，不宣称旧凭据必然撤销或 Strong 静止。
10. 终态之后没有事件；stop_unconfirmed 与 legacy_unknown 不伪造终态、不恢复分支、不交付自身，普通共享仍按原数量准入。

## 验收场景与评审重点

以下是长期验证义务；当前直接证据与缺口以文末链接的核心用例为准，不因产品合入而全部视为通过。

- v2 连接只有 session_mcp 的旧 Node，非空选择拒绝，空选择仍可启动；重载保留 S，运行中丢 S 后
  停止派发并 EndSession。重发原 command 保留原 generation，拒绝 mcp_update_required 后不能记已送达。
- idle/busy/权限等待/已排队轮次与两次 Update 交错；以 Actor 准入排序，不能提前 Executed 或误报
  agent_unavailable。解析失败也 blocked，ACP 失败隔离后无 channel，下一轮只 load 同 provider ID。
- 120 秒无活动窗口与 session update 重置；barrier 等待独立计时；300 秒轮次
  截止不因更新续期，慢重启可耗尽它；blocked 无轮次不触发 Node 自有结束时钟。EndSession 与 Failed
  结算竞争时只有一种结果，无终态后事件。
- 版本升级停止自己的 Agent；兄弟会话不受影响。stop 失败保留精确租约/view，不启动替代 Agent，
  stop_unconfirmed 只阻止 prior_revision 恢复/自身交付；正常停止的 A 不挡普通后继 B，仍检查 8 会话
  限制。Agent/探测带继承锁重启收口，空闲锁迅速确认，迟到证据才补终态、释放资源。
- 插件运行时退出而宿主 spawn 的 Agent CLI（另一进程组）仍存活：锁仍被占用、CLI 线索被终止或保留
  责任，不能判 Stopped；`kill_all` 的成功或失败都不替代原锁与线索；请求停止后的宿主 spawn 被拒绝。
- spawn 登记前后、进程启动闸门、stop 锚定及确认提交前后崩溃；未知身份不能伪装成未启动；PID
  复用不能让新进程被误杀；子进程未继承 fd 不外推 Strong。Shared/Separate 均覆盖版本 view，
  跨设备部署及 EXDEV 必须拒绝。同一进程第 9 次运行时 load（含重连重发与 O1 重试）走重启，失败调用
  也计数，普通 prompt 不改变计数。
- 新 Agent spawn/initialize/load 前崩溃，旧 Agent stop 未确认，V2 停止中到 V3、V2 已启动后到 V3，
  分别验证至多一次 spawn、未派发租约释放与已派发责任保留；模型授权和 ora-git 只随 execution 结束。
- 本地运维 root-only socket 的 `SO_PEERCRED` 权限、幂等和审计；recheck/after-host-reboot 中同 boot
  缺锁、主机标识不同或缺失都不能清标记；Workspace 卷迁到另一主机后不自动确认。确认记录 GC 不
  丢未确认责任或未完成结算，旧 schema/备份回滚被拒绝。
- 未排空升级：排空检查早于迁移且只读旧 schema；仍迁移并为运行中的旧会话标 legacy_unknown，挡
  prior_revision 恢复及其自身交付、不挡普通新会话，只有同机重启证据或单点挂载保证才结算 interrupted。
- 16 成员、2 个 execution 许可、8 个 Node 许可：排队不 timeout；8 个 cleanup_pending 耗尽容量
  可见，重启不能恢复假额度。重测 attempt 丢回复/Controller 重启先查询，旧 flight 不覆盖新身份。
- HTTP 307 + X-API-Key（含同源/跨源/DELETE），验证没有第二次请求；超 1 MiB、SSE 无结束、stdio
  超长帧、401/5xx、tools/list 失败及挂死；不发 tools/call，验证直接 child 与同组后代清理。
- 特征 Settings 追踪覆盖控制帧、ACP、探测以及 ledger/WAL、JSONL、日志、停止责任和页面；值只出
  现在允许的内存/外发，停止记录不能保存 resolved URL/argv/env。
- 配置成功、帧写出、ACP 拒绝、ACP 成功但探测失败分别注入；Desktop 原默认 attach 策略不变，
  共享 HTTP 不跳转/有界响应/忽略系统代理的行为变化有独立回归。

## 风险与资源边界

以下风险已按 owner 的复审结论纳入方案，并随本文一同批准：

| 项目 | 边界与承担方式 |
| --- | --- |
| R1 会话总数 | 接受无全 Node 上限；每 checkout 8 个，输入/Agent 数与最多 16N 个探测意图随 N 增长 |
| R2 版本/view | 接受旧/新集合约 32N 个引用加 Node 8 个探测引用；部署强制 workload、Node home、installed 包同文件系统并检验硬链接，EXDEV 拒绝，不完整复制 |
| R3 磁盘 | 接受插件根无全局配额；部署监测可用空间、在用/历史包字节和回收积压；空间不足拒绝安装，不删在用版本 |
| R4 同版本刷新 | 修改为每进程至多 8 次运行时 load（配置刷新、重连重发、O1 重试均计），第 9 次重启；已写出的失败也计数。仍不保证 provider 每次只起一个进程、关 fd/逃逸后代退出或远端凭据被撤销 |
| R5 残留责任 | 修正门禁和 attempt_lock 证据（含宿主子进程）；只挡恢复/自身交付，不挡正常普通共享；本地 root-only socket 的可审计 reconcile-session-stop 提供 recheck 与同机重启后确认，禁止无证据强制释放。未排空升级的 legacy_unknown 只能凭同机重启或单点挂载保证解除，宿主不重启时可能长期占名额并挡恢复 |
| R6 纳管保证 | 接受进程组 + 继承锁的 BestEffort；证据假定 workload 配合，持 fd 的进程可主动解锁，不传 fd 的后代可能不在证据内，强 containment 后续决策 |
| R7 Agent 恢复 | 接受重启丢失仅驻内存状态；固定 OpenCode/Claude 版本已证明保留 HOME/cwd 后跨进程 load。其他不支持者在重启时 blocked，升级/阈值重启前 UI 提示，禁止 new fallback |
| R8 HTTP | 接受 Node 网络身份及不跳转；明确共享客户端忽略系统代理，需要代理/重定向的 endpoint 将失败，Desktop 同受影响 |
| R9 无终态 | 接受极端缺证据（含未排空升级的 legacy_unknown、换主机后缺主机证据）时 execution 永远没有终态；Node 有限等待转交，Controller 业务收尾保留残留，不能凭时间补造停止 |
| R10 记录增长 | 已确认停止明细 7 天后每小时分批删除/压缩，审计至少 90 天；不删未确认/仍被引用的记录，监控积压，不借 GC 裁账本事件 |
| R11 重启次数 | 接受版本变更及运行时 load 阈值导致的重启次数不设累计上限（后者每次需先写满 8 次 load），Controller 串行 generation、Node 合并未派发目标；每次独立重试至多一次 spawn |

owner 另已决定接受 300 秒轮次截止涵盖完整重启，可能让正常慢 Agent 的这一轮失败；它不延长为无限恢复等待，
也不强制结束 idle blocked Session。初始轮次同样受此约束。停止证据改善例行重启的正常收口，
却不提升为强纳管。Node 全局配额、强 containment、历史健康审计、跨 Node 崩溃恢复 Agent 及实际
工具可见性均不在本文；所述真实 Agent 前提验证不代替产品实现证据。

Node 与消费端分别承担协议责任：Node 提供完整内存控制输入、原子准入与可靠失败、停止证据、
精确版本租约和宿主探测；Controller/v2 必须完成轮次拒绝/失败、generation/attempt 对账、原 IdleTimeout
策略、重启风险提示和能力门禁后才能启用非空 MCP。Node 能力广播不替代消费端接入验收。

等待旧 supervisor 退出没有独立超时；不能把超时当作已退休并提前启动替代者。极端挂起的恢复依赖
Node 重启后的持久停止责任收口。该路径及若干竞争分支仍有直接测试缺口，详见核心用例；不据此
承诺每次 supervisor 退休都在固定时间内完成。

## 核心测试用例

见[会话 MCP 刷新与宿主健康用例](../../../test-cases/node/agent-runtime/session-mcp-lifecycle.md)：能力门禁与
白名单、generation 排序与无值内存输入、准入与轮次结算、只 load 原身份的失败恢复、每进程运行时 load
计数与版本重启、尝试锁停止证据与重启收口、残留门禁、未排空升级的 legacy_unknown、可审计运维对账、
精确版本租约与同文件系统 view、探测许可/预算/清理与 HTTP 边界，以及三层状态与 Setting 值隔离。
Node 侧验收 4、5、6、8、12 为 Passed，其余明确保留跨仓库或直接测试缺口；[显式 Session MCP 用例](../../../test-cases/node/agent-runtime/session-mcp.md)、
[共享 checkout 会话用例](../../../test-cases/node/agent-runtime/shared-checkout-sessions.md)与
[尝试锁用例](../../../test-cases/node/git-execution/attempt-locks-and-stop-evidence.md)的证据保持各自范围，
不推导本文新义务。
