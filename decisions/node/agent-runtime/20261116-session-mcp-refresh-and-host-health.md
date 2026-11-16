---
status: proposed
---

# Node 保留会话身份刷新 MCP，以有界准入和停止证据隔离故障

Controller 持久保存 Session 的显式 MCP 选择，向 Node 提交完整内存配置。Node 在安全点收敛：仅配置
变化时用原 provider ID 执行 ACP `session/load`；包版本变化时先停止本 execution 的 Agent，再启动
它自己的 Agent 并 load 原 provider ID。刷新失败阻塞新 prompt，下一轮自动重试，始终禁止回退到
`session/new`。配置完整、ACP 接受 revision、Host MCP Health 分别取证。

本文为 `proposed`，不是现有实现的描述。需要修改共享 `agent-runtime` 的宿主策略、Node 对话准入、
停止责任与持久账本、MCP 包租约及探测客户端。Node 生命周期模式禁止重建的策略不改变 Desktop 默认
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
| 共享 checkout D2：未清理恢复尝试独立阻止复用 | 扩展至 Agent 和探测的未确认停止；在同一准入事务中先检查停止责任，再检查恢复独占和数量（D5） |
| [停止确认](../revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)及[未启动证据](../revision/20261021-stop-confirmation-accepts-not-started.md)：超时不是停止证据 | 继承命名原执行责任、有限等待、跨重启不续期与迟到确认；不恢复旧 host/guardian 架构 |
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
  持久 Session 行可记录无 channel 的停止状态，但不得清除 provider 绑定或用它触发重建。
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

ACP 刷新使用共享 `session_setup_window`：有 MCP 为 **120 秒无活动窗口**，收到 session update
重置；空集合沿用 30 秒。等待 barrier 不计入此窗口，但计入已开始的 300 秒准入等待。持续更新可
让 ACP 窗口延长，却不能延长轮次截止；轮次失败不伪称刷新已结束。解析错误沿用 setup 稳定码，
其他刷新失败为 `mcp_refresh_failed`，无活动到期为 `mcp_refresh_timeout`，不带第三方响应。

资源等待另有执行级上限：无 active prompt 的未收敛区间连续达到 **600 秒**，Controller 派发
`EndSession{IdleTimeout}`；Node 同样有 600 秒本地上限，覆盖 Controller 失联。pending/refreshing/
blocked 相互切换或新输入不重置它，只有真正 current 才结束该区间。当前 prompt 执行期间不提前
计入，安全点后的等待开始计时。上限请求结束，停止证据仍按 D5，不能凭到期释放资源。

## D5：停止责任持久化，未确认时保留围栏并有限等待

Controller 丢失运行中 execution 的 S 时，立即停止派发新轮次并持久投递 EndSession；不能从
GetSessionMcpState 重建授权、发送空 update 或继续旧下限。仅丢失设置值且 S 仍可信时，先查询
再发送 S 的完整 unavailable 或新的当前配置。断线重连先对账 generation；Node 进程未重启则
继续原 execution，Node 已重启则不能通过重放凭据复活它。

“停止确认”指原 Agent/探测尝试的直接子进程已经退出，且原进程组的清理取得尽力完成证据；HTTP
指本地任务/连接/流已终止，不保证远端服务销毁会话。它按 `StopContainment::BestEffort` 报告，
不使用含混的“物理结束确认”或把 BestEffort 写成整棵树静止。首次停止请求、等待上限和原身份
沿用 `StopUnconfirmed { scope_id, run_id, first_stop_at_ms, bound_at_ms, guarantee }` 的词汇。

Node 增加**无值持久停止责任记录**，不是持久健康缓存。Agent 和 stdio 探测在外部 spawn 之前登记
原 Node 实例、execution、checkout、尝试身份、精确包版本/view 归属及可验证的进程组身份；HTTP
登记本地任务责任。stop 时先锚定首次时间，再发信号，后续重试/重启不续期。不记录 argv/env/URL/
headers/Settings、原始错误或秘密摘要。需要启动闸门：登记与 spawn 的崩溃窗口必须能区分“已证明
未启动”和“未知”，不能把没有 PID 当成未启动，也不能在重启后只凭可复用的 PID/PGID 杀进程。

Node 生命周期负责人在 driver 返回前接管未清理的 Agent/探测、catalog 租约、placement 和 view；
`stop_plugin` 失败不得仅记录 warning 然后 drop 这些资源，scheduler 结束也不是停止证据。更新包
而停止未确认时保持 blocked，不启动下一 Agent；执行结束时进入 stopping，不再接收轮次或探测。

终态和清理采用下列顺序，兼容现有 `ExecutionStatus` 不允许 Completed 带 stop_unconfirmed 的约束：

1. EndSession 或结束原因先停止准入，取消探测并请求停止本 execution 的 Agent；保留正常命令受理
   回复，不能因为清理慢阻塞控制连接、心跳或其他会话。
2. 只有该 execution 的 Agent 和所有探测责任都取得停止确认，才能释放相应租约/view、封口会话并
   写入唯一 `AgentSessionEnded`。此前状态保持非 Completed；不会先发终态后继续清理写入者。
3. 证据等待采用停止确认决策的**首次停止请求后默认 10 分钟**上限，可配置但必须有限且为正。
   上限内进行有界检查；超过上限或检查无法继续时登记 stop_unconfirmed，结束当前等待任务，
   将原责任交给残留对账。终态可以迟到，甚至因始终无证据而没有终态；这是明确可查询的停止
   未确认，不是无限挂住 driver future。Controller 业务结束窗口按其现有策略收尾并保留残留责任。
4. `GetExecutionStatus` 返回原 StopUnconfirmed，MCP 快照可显示 stopping/cleanup_pending。
   有界的显式残留对账只能凭原尝试证据补确认；不得因时间到、连接断、future drop 或目录存在性
   解除责任。取得迟到证据后补终态，原结束原因及序号规则不变。

同一 checkout 的任一 Agent/探测停止未知，或停止责任日志不可读时，**普通新会话和带 prior_revision
的恢复都拒绝为 checkout_busy**；检查与现有准入共用事务且先于数量限制。现存兄弟会话不被重启，
但不能借旧 execution 的业务收尾触发新恢复。未确认的执行继续占未结束名额；已存在历史终态的
残留也独立挡住复用和交付，不靠“未结束计数”单独保证安全。

Node 启动先恢复这些围栏及 catalog 的版本保护，再允许安装/移除、目录回收和 checkout 准入；
不能按“私有目录”整棵清空。原 Agent 不重启，只有取得原停止证据后才补 interrupted；仅 Node
进程更换而宿主仍存活不证明子进程退出。可靠的宿主重启/未启动证明可用于确认，缺证据继续保留。
HTTP 的本地责任可由原进程已退出的证据结算，不把它说成远端会话已销毁。

新增责任、轮次 Failed 结算及其事件需要 Node DB 前向迁移。旧 start 的 mcp_lifecycle 缺省 false；
旧库没有的责任不能凭空标为已清理，升级按既有停止/恢复围栏处理历史执行。旧二进制拒绝打开新
schema，回滚用相容备份，不承诺直接降级。后续 desired/active、探测结果及 Settings 仍只在内存。

## D6：版本变化重启自己的 Agent，精确租约保护使用中的字节

MCP 租约改为 canonical ID + 精确版本。新版本按原下载/摘要/静态校验路径发布到独立目录；被
使用版本不得改写或删除。发布、取租约、校验与删除共用 catalog 同步边界；并存前核对新旧 kind
均为 MCP，不允许同 ID 换 kind 绕过 Agent 保护。其他类别仍按 ID 保护。RemovePlugins 仍整 ID
移除，任一版本被用即 `plugin_in_use`；同版本修复被使用时同样拒绝。

版本变化走本 execution 独占的重启路径：

1. 先取得新完整集合的精确租约，解析并准备 view；准备失败 blocked，不触碰旧进程或部分切换。
2. 等当前 prompt/权限请求与 barrier 安全点，停止自己的 Agent 和该旧进程拥有的 MCP。execution、
   Session、provider ID、checkout、模型访问及历史归属保留，不影响兄弟 Agent。
3. 旧 Agent 按 D5 取得 BestEffort 停止确认后，释放它使用的旧版本租约/view；旧版本若仍被探测或
   兄弟会话使用，由那些使用者继续持有。停止未知时不释放、不启动新 Agent。
4. 启动新 Agent，仅 load 原 provider ID 与最新完整输入；成功才 current。load 失败保持无可用
   channel 的 blocked，下一轮只能按 D4 重试 load。provider 无法保留身份则刷新失败，不能 new。

因此不再设置“累计 64 个旧版本”上限，也不把每个发过 ACP 的版本保留到 execution 结束。单个
execution 同时只需旧 Agent 使用的集合与正在准备的新集合（各最多 16 个成员），加至多 2 个探测
使用的旧成员版本；未开始的中间目标不取租约。进行中的切换串行，晚到更新在同一安全点接续收敛。

只改配置时继续运行中 load，不主动重启 Agent。ACP 成功不证明 provider 注销了旧注册、旧凭据
已经撤销或 MCP 进程停止；升级重启也仅在 BestEffort 所覆盖的进程组内给出旧进程退出证据，不能
保证逃逸后代。UI 的“已送达”不能替代这两种边界。

`Separate` placement 的 MCP view 必须按 `(canonical ID, exact version)` 区分并由 execution 私有
目录拥有，不能继续仅用 ID 的路径覆盖；硬链接跨设备失败时可能完整拷贝，同样受租约保护。
`Shared` placement 不制造 view，使用对应精确版本的 installed 目录，也持有相同精确租约。
目录删除逐使用者结算，不能因 Agent 停止删除尚在被探测使用的整个 placement。

这些是 Node catalog/placement 的新行为，不改变 Desktop 安装器的版本激活/移除语义。本文批准后
在 plugin-runtime 根决策添加 `> 历史决策：` 指向本 ADR 的局部修改；proposed 阶段不把旧契约标成
已被替换。前序两份 ADR 的 implemented 状态及 README 由各自落地运行维护。

## D7：探测绑定当前解析产物，明确执行身份与 HTTP 边界

Node 对当前 desired 中 configured 且静态有效的成员使用共享 resolver 的同一 transport 绑定。
身份包含 NodeRuntimeIdentity、execution、generation、canonical ID、包版本、configuration revision、
transport 和实际探测身份。stdio 额外绑定 workload UID/GID 与 **Node 从 checkout 解析的真实绝对 cwd**；
探测进程的 current_dir 就设为该 cwd，workspace 占位参数也用它。缺少真实上下文时不 spawn，
显示 Unknown(context_missing)，不继承偶然的 Node 工作目录。ACP 不承诺 provider 的 MCP 子进程
cwd，因此该结果仅证明这一个 Node 上下文，不声称复现了 Agent 内部执行。

HTTP 握手在 **Node 进程的网络身份/namespace** 中执行，不伪称按 workload UID 发出；headers/URL
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
| 所有配置更新也重启 Agent | 可更快回收同版本旧注册，但增加每次凭据保存的中断成本；当前仅版本变化重启，配置刷新保留 provider 资源增长风险 |
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
5. 刷新失败保留原 provider ID，下一轮只 load；升级只重启本 execution 的 Agent。
6. 超时、断线或任务结束不构成停止证据；未知责任保留精确包、view 与 checkout 围栏。
7. Agent/探测停止责任先于 spawn 登记；重启先恢复围栏，不能刷新额度或复活 Agent。
8. 探测预算从许可开始，排队不是不健康；探测与当前身份不符则不能写回。
9. ACP 接受、Host 健康和 BestEffort 清理互不替代，不宣称旧凭据必然撤销或 Strong 静止。
10. 终态之后没有事件；停止未确认时不伪造终态、不交付、不复用。

## 验收场景与评审重点

本 ADR 尚未 approved，以下是可观察场景；此阶段不新增核心测试用例或宣称已有实现覆盖。

- v2 连接只有 session_mcp 的旧 Node，非空选择拒绝，空选择仍可启动；重载保留 S，运行中丢 S 后
  停止派发并 EndSession。重发原 command 保留原 generation，拒绝 mcp_update_required 后不能记已送达。
- idle/busy/权限等待/已排队轮次与两次 Update 交错；以 Actor 准入排序，不能提前 Executed 或误报
  agent_unavailable。解析失败也 blocked，ACP 失败隔离后无 channel，下一轮只 load 同 provider ID。
- 120 秒无活动窗口与 session update 重置；barrier 等待独立计时；300 秒轮次期限和 600 秒未收敛
  期限不因更新续期。EndSession 与 Failed 结算竞争时只有一种结果，无终态后事件。
- 版本升级停止自己的 Agent；兄弟会话不受影响。stop 失败保留精确租约/view，不启动替代 Agent，
  重启 Node 后仍禁止普通 checkout 复用和 prior_revision 恢复；迟到原证据才补终态、释放资源。
- spawn 登记前后、进程启动闸门、stop 锚定及确认提交前后崩溃；未知身份不能伪装成未启动；PID
  复用不能让新进程被误杀。Shared/Separate 和跨设备复制均覆盖版本 view 与清理边界。
- 16 成员、2 个 execution 许可、8 个 Node 许可：排队不 timeout；8 个 cleanup_pending 耗尽容量
  可见，重启不能恢复假额度。重测 attempt 丢回复/Controller 重启先查询，旧 flight 不覆盖新身份。
- HTTP 307 + X-API-Key（含同源/跨源/DELETE），验证没有第二次请求；超 1 MiB、SSE 无结束、stdio
  超长帧、401/5xx、tools/list 失败及挂死；不发 tools/call，验证直接 child 与同组后代清理。
- 特征 Settings 追踪覆盖控制帧、ACP、探测以及 ledger/WAL、JSONL、日志、停止责任和页面；值只出
  现在允许的内存/外发，停止记录不能保存 resolved URL/argv/env。
- 配置成功、帧写出、ACP 拒绝、ACP 成功但探测失败分别注入；Desktop 原默认 attach 策略不变，
  共享 HTTP 不跳转/有界响应的行为变化有独立回归。

## 风险、资源边界与落地顺序

以下是本提案要求明确接受的边界，不能用局部上限声称 Node 整体有界：

| 资源或保证 | 最坏情况与边界 |
| --- | --- |
| 会话总数 | 每 checkout 8 个，不设全 Node 上限；N 随 checkout 数增长，Agent 进程、会话输入和最多 16N 个等待探测意图都随 N 增长 |
| MCP 版本/view | 活跃切换至多约 32N 个版本引用，加 8 个探测成员引用；同版本共享 installed 字节，但 EXDEV 的私有 view 可以完整复制，不能当作硬链接的零成本 |
| 磁盘 | 包大小、插件根全局配额和无租约历史版本占用没有本 ADR 提供的总上限；原插件根也未解决磁盘配额。安装失败不得删除在用版本腾空间 |
| 配置刷新 | 不变包的多次 load 可能让 provider 累积注册/子进程，数量可随刷新次数增长；本 ADR 未定义全局或每 Agent 硬进程上限，600 秒未收敛期限不能约束一直刷新成功的长寿命会话 |
| blocked 与停止未知 | 无 prompt 的未收敛区间 600 秒内请求结束，但物理资源只有停止证据才释放；不可中断进程可使残留无限保留并耗尽 8 个探测许可，必须对账/运维处理，不能自动放宽围栏 |
| 进程保证 | 进程组 BestEffort 是对原“进程树清理”范围的有意收窄；逃逸后代及其凭据不在完整退出保证内，强 containment 留给后续决策 |
| Agent 重启 | 版本更新付出重启延迟并丢失 provider 仅驻内存的状态；若 provider 无法以原 ID load，则 blocked，禁止静默重建 |
| HTTP | Node 网络身份与 workload 可能不一致；关闭全部跳转会使依赖重定向的 Desktop/Node endpoint 探测失败，需配置最终 URL |

未设全局会话/磁盘/配置刷新进程上限是本提案保留的风险，尚不能因局部数值已经选定就宣称 owner
接受了这些无界项。确认后以部署监测与明确生命周期管理承担，强 containment、全局配额、provider
注册回收策略或历史健康审计另行决策。本文也不解决 Node 崩溃后恢复 Agent、MCP-over-ACP 或 Agent
实际工具可见性；Host 健康不替代它们。

落地分三步，代码义务由 Node 产品实现运行承担，不转嫁给仅实现 v2 UI 的运行：

1. 完成本文复审后标 approved；再在镜像目录登记核心用例为 Missing，并为被修改的 plugin-runtime、
   会话命令及停止/共享 checkout 契约补历史指针。不在本 PR 提前改其他 ADR 的 implemented/README。
2. Node 产品 PR 同批交付下表责任并通过边界测试，全部具备才广告新能力；Controller/v2 同版本接入
   轮次拒绝/失败、generation/attempt 对账、IdleTimeout 和能力门禁。v2 的选择恢复/状态展示可独立
   先行，但不能提前宣称刷新/健康已完成。

   | 责任面 | 必须新增的行为 |
   | --- | --- |
   | `crates/agent-runtime` 的 actor_mcp / cleanup / attach / live state | Node 宿主策略保留 Blocked 与 provider ID；resolver 失败纳入状态机；只 load、无 new fallback；有类型的非终结准入结果与通知；版本切换的独占 Agent 重启接缝 |
   | `apps/ora-node/src/session/driver.rs` | 在 settle Executed 前等门禁及 Actor 确认；300 秒 Failed 结算与 600 秒结束；停止失败移交资源 owner，不能直接 drop placement/leases |
   | `crates/node-protocol` / `crates/node-db` / Node service | 新消息和条件必填 generation；按原因区分投递拒绝；可靠 SessionTurnFailed；无值停止责任、启动闸门与前向迁移；恢复围栏先于准入/回收 |
   | Node catalog / `session/workload.rs` | MCP 精确版本租约、按 id/version 的 Separate view、Shared installed 路径、Agent/探测各自持有与迟到释放 |
   | Node probe owner / `ora-utils::mcp` | 8/2 许可、从许可起 10 秒、独立 workload launcher 与确定 cwd、停止责任和容量诊断；共享 HTTP 禁跳转与流式响应上限 |

3. 产品实现评审通过后另开落地运行，以实际实现修订 ADR 并回填直接测试证据；产品 PR 合并后再合并
   implemented 状态。保留旧模式冻结 setup、独立会话身份、Node 不自动恢复 Agent 和可靠事件承诺，
   不把 proposed 的资源取舍或尚未验证的停止行为称为已实现。
