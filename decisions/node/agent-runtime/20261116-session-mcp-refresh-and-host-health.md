---
status: proposed
---

# Node 以完整内存输入刷新会话 MCP，并独立报告宿主健康

Controller 保留工作流 Session 的显式 MCP 选择，向 Node 提交同一选择的新版本和配置。Node 在共享
Agent Session Barrier 内通过 ACP `session/load` 收敛，不中断已经开始的 prompt，也不替换 execution
或 provider session identity。健康探测由 Node 使用同一解析产物独立执行；配置完整、ACP 已接受的
revision、Host MCP Health 是三个不能互相推导的事实。

本文为 `proposed`。已批准的是扩展范围，以下 wire 形状、资源上限、确认和重放规则仍需评审。影响
`node-protocol`、Node 会话宿主、Node 的 MCP 包租约、共享运行时与 MCP 客户端的宿主接缝；不改变
Desktop Backend 既有行为，不启用 Node 重启后的 Agent 自动恢复。Controller 的业务存储和 v2 页面
实现分别遵守其所属契约，本文规定它们必须满足的消费边界，不规定数据库表或 UI 组件。

## 继承与修改

承接本目录前序[共享 checkout 会话](20261115-shared-checkout-agent-sessions.md)，经它继承
[显式 Session MCP](20261114-explicit-session-mcp-in-agent-session-spec.md)及
[共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)。MCP 生命周期由会话宿主拥有，
因此在本目录定义协议扩展，不再在 protocol 目录复制同一状态机。

| 前序约定 | 处理 |
| --- | --- |
| 显式 Session MCP D1：有序完整选择、最多 16 个 Server、Settings 紧凑 JSON 合计 64 KiB | 继承；每次新输入独立校验，空选择不自动发现 |
| 显式 Session MCP D2/D3：一次冻结 setup，configuration revision 为 0；重发 start 不替换值 | 仅对逐会话选择新能力的执行扩展；start 重发仍不更新值，后续更新走独立的内存路径 |
| [会话命令](../protocol/20260928-streamed-thread-events-and-session-commands.md) D4：用户轮次及结束命令持久受理 | 继承；新增 MCP 更新、查询与重测不进入 durable command queue，不冒充 `SessionCommandAccepted` |
| [实时预览](../protocol/20261018-live-thread-previews.md)：非持久观察可失效、丢失后重新查询 | 继承这项边界；MCP 状态是独立查询回复，不进入 Thread 记录、事件序号或 Ack 窗口 |
| Node 重启结算 interrupted，不重启原 Agent | 继承；MCP 值、Active 和健康不从 ledger 重建 |
| [工作流显式选择](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md) | 继承 Session 自身保存白名单和 fail closed；Node 始终没有 Automatic |
| [ACP setup/refresh](../../desktop/core/agent/session/mcp/0-acp-session-setup-delivery.md) | 继承同一 resolver、完整快照、barrier、成功响应推进 Active、失败阻塞；远程配置与 prompt 的排序见 D4 |
| [Host 健康探测](../../desktop/core/agent/session/mcp/20260916-mcp-health-probing.md) | 继承完整握手、事实分离、无值结果与内存缓存；Node 只探测已授权会话的绑定，不在插件安装时取得会话凭据 |
| [插件安装](../plugin-runtime/0-node-installs-plugins-into-workspace-home.md) D2：按 ID 使用期间拒绝替换/移除 | 对 MCP 增加精确版本租约与不同版本并存；被使用的字节仍不可替换/移除，Agent 等其他类别保持原规则（D6） |
| 共享 checkout 的独立会话、恢复独占、数量上限和受控单例 | 继承；一次 MCP 更新/探测只属于一个 execution，不重启兄弟 Agent，也不改变 checkout 归属 |

## 问题与约束

Node 的启动快照没有 Controller 配置源，不能通过再次发送相同 `StartAgentSession` 更新 Settings。
把值装进已有会话命令会写入 ledger；把 Controller 控制帧写出当成 ACP 送达，则无法区分未受理、
等待安全点和 `session/load` 失败。与此同时，按 ID 租约阻止安装新版本，直接解除租约又可能删除
旧 Agent 仍使用的文件。这些边界必须一起定义。

现有 MCP 握手组件可复用协议与错误分类，但 Node 还必须注入工作负载身份、版本视图和进程组清理。
现有直接 spawn 并等待单个 child 的路径不能自动证明探测后代已清理，也不能默认以 Node 管理身份
执行用户选择的 MCP。

## D1：能力逐会话选择，显式白名单独立于可变绑定

Node 增加能力 `session_mcp_lifecycle`，只有同时支持 `agent_session`、`session_mcp` 和本文全部行为
时才声明。Controller 只有观察到该能力才在新 `AgentSessionSpec` 中设置 `mcp_lifecycle: true`；省略
或 false 保持一次冻结 setup，已有执行不被隐式升级。该布尔值是无秘密持久输入的一部分，参与
重复 start 的比较。没有选择此模式的会话不接受本文更新或主动探测。

新模式仍以 `spec.mcp` 启动，初始 generation 为 0。启动列表中的 canonical ID 集合 S 是该 execution
永久不变的白名单；空 S 永远只能得到空集合。Controller 在创建业务 Session 时把 S 与 Session 一同
持久化，恢复页面、重新挂接以及 provider 重新附着都读它，不在运行时反查可变工作流草稿。历史
选择元数据缺失或不可信时采用空显式集合或拒绝恢复；不得从“已安装插件”重建权限。

后续更新必须为 S 中每个 ID 提供且只提供一项，仍按 canonical ID 排序。每项是下列之一：

- `configured`：精确 canonical `plugin_version`、无秘密 `configuration_revision` 和必填 Settings map；
- `unavailable`：ID 以及 `mcp_plugin_missing` 或 `mcp_setting_missing`，不带值，也不猜测版本。

`unavailable` 表达选中插件被卸载、配置被清除等事实；它不移除 S 的成员，也不允许把其余成员组成
部分 ACP 列表。Node 把这一完整 desired 输入记为 blocked。包在 Node 缺失或解析失败也得到 blocked，
即使 Controller 报告 configured。未选择插件的变化不提交新 generation。

这保留白名单，又允许同一 ID 的版本和配置变化。业务上需要增减 ID 时应建立新的业务 Session，
不能借 refresh 扩大已有会话授权；这与为了刷新而替换 execution 是两件事。

## D2：更新是带顺序的内存输入，不是持久命令

Controller 为每个业务 Session 的配置变化串行分配严格递增的 `generation`，持久化无值目标身份后
发送完整更新。generation 与 configuration revision 都是非负且不超过 `2^53-1` 的 JSON 整数；新更新
generation 大于 0。选用这个范围使 Go、Rust 与 JavaScript 能无损比较，不以 Setting 值或其摘要
计算代号。到达上限明确拒绝新更新，不回绕。

建议的消息形状如下，均携带既有 `protocol_version`、`operation_id`、`execution_id`，请求另带
`node_id` 与有界 `request_id`。身份、当前 Controller 归属及受控模式的 incarnation/control scope
验证先于读取/改变会话，也先于解密或解析配置；不允许传递 Node 路径。

| 请求 | 输入与回复语义 |
| --- | --- |
| `UpdateSessionMcp` | `generation` 与 S 的完整成员列表；回复 `SessionMcpReply`，区分 accepted、duplicate、stale、conflict、session_ended、not_enabled、invalid_selection，并附当前无值状态 |
| `GetSessionMcpState` | 无可变配置；立即返回当前无值快照，不触发解析、刷新、探测或新 Agent |
| `ProbeSessionMcp` | 目标 generation、单个 plugin ID 与递增 `probe_attempt`；只允许当前 configured 成员，加入/发起一次探测并立即回复状态，完成后由查询观察 |

`request_id` 只用于关联回复，更新去重以 execution + generation 为准。所有消息经严格 schema 和
既有帧上限校验；更新仍限 16 项和 64 KiB Settings。结构错误走现有协议校验失败路径；上述业务拒绝
保留控制连接。查询只返回所属执行的结果，未知执行返回 not_found，不暴露其他执行的数据。

Node 对单个 execution 串行比较 generation，并在回复 accepted **之前**原子替换内存 desired 输入
和 prompt admission 所见的目标。此受理不等待包解析、ACP 或探测，不代表 applied，也不表示崩溃后
可重放。语义是：

| 与已接收的最高 generation G 比较 | 处理 |
| --- | --- |
| g > G 且白名单/结构合法 | 取代 desired；唤醒 Actor。即使包/设置不可用也必须推进目标并阻塞，不能继续旧配置 |
| g = G，完整输入相同 | 返回 duplicate 和当前状态，不重做 ACP 请求或主动探测 |
| g = G，输入不同 | 返回 conflict，保持原输入；值只在进程内做比较，不输出差异、不保存摘要 |
| g < G | 返回 stale，不解析旧值，不改变状态 |
| execution 已结束/模式未启用 | 返回对应拒绝，不留存值，不复活会话 |

Controller 重发同一 generation 必须发送相同输入；凭据轮换、包恢复或对 blocked 状态的显式重试
均分配新 generation。不能用当前凭据重建旧 generation 的请求。若 Controller 丢失旧请求的内存值，
先查询 Node，再从当前权威配置分配新 generation；不能从 Node ledger 补回秘密。

Node 最多保留 active、正在发送的快照和最新 desired 三份有值输入，丢弃未发送的中间版本；每份
仍受 64 KiB 限制。后台任务只收无值唤醒和身份，不复制有值输入到通用消息队列。值无 Debug、无日志，
不进 WAL、历史、事件、Workspace 或临时文件；错误只含稳定码和适用的 plugin/setting/transport 标识。

Node 把 generation 纳入其 Desired/Active 的无秘密 revision 身份；新 generation 即使绑定内容相同，
也必须完成本次收敛，不能把旧 ACP 响应改标为新 generation 的成功。这样显式重试与 duplicate 有
可观察的区别；Controller 不应对未变配置空发 generation。异步解析完成时再次比较当前 generation，
过时解析结果不覆盖最新 desired 或解除其阻塞。

## D3：状态是可查询的内存事实，断线不创造应用证据

`SessionMcpReply` 携带请求 ID 和一份完整快照：Node 实例身份、execution、单调递增 `state_version`、
desired generation/成员无值身份、active generation/成员无值身份（可空）、convergence 状态、错误和
逐成员健康。convergence 为 `pending | refreshing | current | blocked | ended`。pending/refreshing
不能被渲染为已送达；blocked 中若仍保有此前 Active，只表示“上次成功送达”，不表示当前配置可用。

快照只在内存中，通过上述请求回复，不主动向未请求新消息的连接广播，也不进入执行事件序号、
Thread 或 `EventAck` 窗口。一个慢查询不阻塞 Actor、心跳或 EndSession；发送缓存每个查询只保留
一份最新快照，单连接待答请求上限 64，超限按既有受理容量错误拒绝，不建立无界队列。

Controller 可以对 pending/refreshing/探测中的会话有界轮询，界面订阅结束即停止其轮询；查询不构成
健康复检。状态按 Node 实例 + execution + state_version 合并，低版本、旧实例及已终态执行的迟到
回复不能覆盖新状态。断线或状态缓存丢失后展示未知并重新查询，不能从最后一次写帧或磁盘缓存恢复
current/Healthy。Node 重连且进程未重启时仍有当前内存状态；Node 重启按原协议结算 interrupted，
不恢复 active、健康、值或原 Agent。已终态查询只返回 ended，历史的“曾送达”若需归档属于后续决策。

Rust Controller 与 Node 继续采用同版本发布边界，不能声称新增 enum 对旧解码器透明。v2 的字符串
能力列表可接受未知能力，但必须显式消费本文能力才发送新字段/请求。旧启动输入仍有效；不支持
生命周期的调用方不能接管一个已启用模式的会话继续提交 prompt，D4 的必填 generation 检查使其
在进入 Agent 前失败。它仍可使用原 `GetExecutionStatus` / `EndSession` 对账和结束，不必解码 MCP
回复；任何连接变化都不自动结束执行。

## D4：Node 在安全点收敛，Controller 显式携带 prompt 的配置下限

Node 复用共享 Actor 的 desired/active 比较和 Agent Session Barrier，不建立第二条 restart 或 load
路径。空闲会话立即刷新；busy 会话允许当前已进入 Agent 的 prompt 完成，只保留最新 desired 欠账。
下一轮进入 Agent 前必须 active 等于当前 desired 且状态 current。刷新重读完整输入并持有对应租约，
以原 provider ID 调用 `session/load`，只在成功响应后推进 Active；完成时若有更新目标，继续阻止
后续 prompt，不能让过期成功清除新欠账。

远程 Node 无法直接观察 Controller 数据库的一次提交。为防止更新与下一轮投递乱序，对启用模式的
`SubmitUserTurn` 增加必填 `mcp_generation`，它是该轮允许使用的配置下限：

1. Controller 在接纳该轮的权威操作中读取当前目标 generation，与 turn 的无值派发记录一起保存；
   同一配置保存操作为受影响的 Session 推进目标，尚未送达 Node 的更新不能让后续轮次沿用旧下限。
2. Node 受理用户轮次时，若本地 desired 低于该下限，返回新增 `mcp_update_required` 拒绝，不写入
   用户命令队列、不开始 prompt；Controller 先补更新再重试同一 command。拒绝不是持久受理。
3. 若本地已 blocked，新增轮次以 `mcp_refresh_blocked` 拒绝。pending/refreshing 时可以沿用原队列
   受理，但 Actor 只能在最新 desired 成功后执行。已经受理的排队轮次在刷新失败时保留为等待，
   不假报完成、失败或丢弃；用户修复配置并提交新 generation 后继续，或 EndSession 按原规则丢弃。
4. 出队时再次检查本地最新 desired，使用更新版本满足较低下限；不会为了旧轮次回退配置。已经
   受理命令的重复请求仍按原规则返回 accepted，即使此时 blocked，不能把已有队列记录改说成拒绝。

排序点是 **Controller 接纳轮次与配置保存的顺序**，以及 **Node 接纳更新与 prompt admission 的顺序**。
先于配置保存已被接纳的轮次可能在网络延迟后进入 Agent，使用尚未收到更新的旧版本；本文不承诺
跨进程以绝对时间实现“配置 SQL 提交后一切 ACP prompt 立刻使用新值”。若 Node 已接纳新目标，
任何尚未进入 Agent 的轮次（包括旧排队轮次）都必须等它收敛。这一明确边界避免用控制帧顺序代替
跨系统原子性；需要撤销正在执行工具的凭据属于凭据系统，不由温和 refresh 承诺。

权限请求未结算、prompt 尚未终止或已有 Agent barrier 时，不提前执行 load。刷新预算为进入安全点
后的 30 秒，不用健康探测延长它；busy 等待沿用既有轮次取消/看门狗，不发额外 cancel。EndSession
始终优先且不受 MCP blocked 或查询阻塞。

解析/能力失败发生在完整 ACP 请求前；ACP 拒绝或超时则将 channel 按共享运行时规则隔离，状态 blocked，
不得继续旧集合或用 `session/new` 替代。明确的 resolver 错误沿用 setup 的稳定码；其余分别使用
`mcp_refresh_failed`、`mcp_refresh_timeout`。这些码不含第三方响应或设置值。后续显式新 generation
重试仍以原 provider ID load；无法安全恢复时保留 blocked，或者由用户结束会话。原有终态规则和
正在评审的“取消一轮”协议分别演进，本文不假设取消一定保留会话。

## D5：Controller 失联与持久化边界

业务 Session 的 S、模式和 Controller generation 是持久事实；Node start 的持久投影仍只保留无值
MCP 身份与版本。本文不持久化后续 desired、active、探测结果或含值更新命令，不新增 Node 数据库
表，也不重写历史 start 输入。原数据库中没有 `mcp_lifecycle` 的记录按 false 读取，不改历史语义。
旧二进制可能无法解码带新模式的持久执行输入，因此本文不承诺升级后的原库可直接降级运行；部署
继续遵守 Node/Controller 同版本及原有数据库回滚边界。

同一 Node 进程短暂失联时继续已经开始的 turn，保留最新 desired 和阻塞状态；恢复连接先查询状态
并对账最高 generation，再派发后续轮次。Controller 不得把重新连接当成重新 start 的理由。控制
scope 被撤销时，更新、重测、查询与用户轮次和既有权限规则一样被拒；不得通过 MCP 通道重新授权。

Controller 重启若只剩配置引用而没有旧值，按 D2 发送一个新 generation 的完整当前输入。若不能
取得配置，不得自动扩大 S、删掉成员或重发空集合；发送覆盖全部 S 的 unavailable 目标以阻止后续
prompt。Node 已重启的执行则只按 interrupted 对账，不向它重放凭据使 Agent 复活。

## D6：MCP 新旧版本并存，租约覆盖真实使用者

只对 MCP 包把 Node 使用租约细化为 canonical ID + 精确版本。安装新 MCP 版本可以在旧版本被使用
时按原下载/摘要/静态校验路径原子发布到其版本目录，但不能重写同一被租用版本，不能在 publish
时删除仍被任一会话、刷新尝试或探测使用的其他版本。非 MCP 插件仍保持按 ID 的原保护语义。
允许并存前须验证新包与当前被使用版本均为 MCP；不能通过同 ID 改 kind 绕过 Agent 等类别的保护。

租约取得、包身份核对、发布和删除共用原 catalog 同步边界。不同版本可共存，同一版本无效且被使用
时返回 `plugin_in_use`，不能以修复为由就地替换。`RemovePlugins` 仍是整 ID 移除：只要任一版本被
使用即返回 `plugin_in_use`；Controller 的业务卸载可先提交 unavailable 使会话 blocked，磁盘移除
要等相关会话结束，不用释放租约假造卸载成功。

每次更新先取得新精确版本租约与会话拥有的只读 view，再解析和发送。旧 view/租约至少保留到成功
切换；本提案选择更保守的释放点：**任何曾写入 ACP 帧的包版本都保留到该 Agent 进程确认停止**。
原因是 ACP 的 load 成功只证明接受输入，不保证 Agent 已停止旧 MCP 子进程；提前删除旧可执行文件
或依赖文件仍可能破坏已运行服务。未发送的 superseded 快照可立即释放；正在探测的版本至少保留到
探测清理确认。Settings 可在不再被内存解析/发送使用后释放，不以保留字节租约为由延长其持有。

为防止持续升级形成无界保留，一个 execution 至多保留 64 个不同的已发送 MCP 包版本身份（初始
集合计入，最多 16 个 ID）。超过预算的更新仍推进 desired，并以 `mcp_package_retention_limit`
blocked，绝不删除旧版本腾位置；这是需要结束该会话再发起新的业务会话的明确资源限制，不是每次
refresh 的实现方式。Node 不在后台扫描并删除历史版本；结束后释放租约，后续安装可以按原规则
清理无租约旧版本，整 ID 移除清理所有无租约版本。磁盘全局配额仍遵守插件存储的既有边界。

这些规则修改 Node catalog，不改变 Desktop 安装器的激活/移除语义、不改变包布局或工作区文件。
版本 view 使用本次 execution 私有目录，新旧 view 不覆盖彼此，也不得删除兄弟会话的视图。

## D7：探测绑定同一解析产物和执行身份

Node 对当前 desired 中配置完整、静态校验通过的成员使用共享 resolver 的同一 transport 绑定产物，
不另写配置解释器，不向 Controller 返回 argv/env/headers/url。探测身份包含 Node 实例、execution、generation、
canonical ID、精确包版本、configuration revision 和 workload 身份；依赖 workspace context 时再
包含 Node 从 checkout 解析的真实绝对 cwd。相同 configuration revision 的不同值在同一 execution
中不被允许：Controller 变更值必须推进其 revision；Node 对仍持有的同身份值进行内存一致性校验。
初始 generation 0 的身份不与后续正 revision 混用。

探测在新完整 desired 被成功解析以及 `session/new`/`session/load` 后异步补齐 Unknown；重复触发
共享同一个身份的 flight 或已完成结果。手动 `ProbeSessionMcp` 的 `probe_attempt` 在当前 generation
和成员内严格递增，范围为 1 到 `2^53-1`；Node 只保留该身份的最高 attempt。相同 attempt 重发只返回
当前结果，较小 attempt 返回 stale，避免丢回复后的重发再次外连；较大 attempt 在有 flight 时加入
同一个 flight，没有 flight 时才重测。自动补探测使用 attempt 0，不覆盖手动结果。无周期复检、
无自动重试、无 TTL。跨 execution 不共享缓存，即使 ID/cwd 相同，避免将不同凭据/授权的结果混用。

缓存只保留当前 desired 的至多 16 个成员，以及仍须收尾的有界 flight；新 generation 不复用旧
generation 的健康结果。这比跨配置代复用多一次握手，但不需要持有历史凭据来证明结果仍相同，
也不会让旧 configuration revision 被误复用时显示过期 Healthy。

Node 没有不带会话的配置来源：插件安装本身不触发带凭据探测，不借安装执行伪造 cwd 或会话身份。
没有真实会话上下文的插件卡片只能显示未探测；依赖 workspace context 时显示
`Unknown(context_missing)`。会话视图可显示该 Node/Session 的结果，不能把它提升为全租户全局健康。
没有配置资格的成员不呈现 Healthy/Unhealthy；空 S 不发起探测。

握手复用现有 MCP 客户端：`initialize`、`notifications/initialized`、`tools/list`，不执行
`tools/call` 或其他业务操作。HTTP 使用同一 URL/headers 的 Streamable HTTP，凭据不随跨源跳转
转发，不用简单 GET 或只有 initialize 的成功冒充工具发现成功。stdio 使用同一 command/args/env，
workspace context 只替换声明中的占位参数，不擅自改变 ACP 未承诺的进程 cwd。

配置 revision、包版本、执行或 cwd 身份失配后旧结果立即失效。旧 flight 可收尾，但结果不得写回
当前身份；释放其租约前仍要完成清理。状态只在内存：`Healthy`、`Unhealthy{code}` 或
`Unknown{not_probed|context_missing}`，另有 `probe_in_flight` 指示等待，不把等待当作失败。错误
沿用 Host 探测的封闭码族，不输出第三方文本。普通健康失败不改变 desired/active，不阻止 prompt。

## D8：探测必须有界，并由 Node 拥有收尾责任

每个 Node 最多同时执行 8 次探测，同一 execution 最多 2 次；至多为每个当前成员保留一个待探测
意图，不保存重复的调用方等待者或旧 generation 队列。容量不足时手动请求回复 busy，自动补探测
保持 Unknown；许可释放后可消费当前意图，旧身份不再补跑。排队从请求时计入 10 秒总预算，避免
排队 + 握手各自重置超时。

10 秒预算覆盖等待许可、spawn、initialize、tools/list 和正常清理；最多前 8 秒用于等待/握手，最后
至少 2 秒保留给关闭 stdin、终止探测进程组和 wait。HTTP 同样关闭响应流和客户端会话。读入单帧、
累计响应均限制为 1 MiB；超限按 handshake/tools 阶段返回对应稳定失败，不无界读 stdout 或响应体。
一次 tools/list 响应即为本探测的发现证据，不自动追随分页执行无限请求。

stdio 启动使用 Node 注入的 launcher：与对应 Agent 相同的 workload UID/GID、权限降级及只读包
view，在独立探测进程组内运行；不继承 Node 管理密钥或全量宿主环境，只继承 Agent 所用的基础
运行环境并叠加本次解析 env。不得直接复用默认以调用者身份 spawn 的便捷入口。探测从不加入 Agent
进程组，结束探测不能杀 Agent 或兄弟会话。

探测任务由 Node 生命周期持有，客户端断线、取消 HTTP 查询或丢弃 future 不放弃收尾责任。超时、
成员失效、EndSession 和 Node 正常停止都触发取消并等待清理。正常成功只有握手成功且进程/连接
已清理才能成为 Healthy；必须测试直接子进程和同组后代的退出，而不只断言返回了 timeout。

如果操作系统在预算内无法确认退出，对外按时返回 `mcp_probe_timeout` 并标记
`cleanup_pending: true`；这一标记是独立资源责任，不是健康成功。Node 保留该 flight 的并发名额、
包租约及 view，由受控的收尾任务继续等待；不因等待超时假报已清理、释放额度或开始相同身份的
新探测。该 execution 的物理结束确认必须等它拥有的探测停止，不能先删除 view。普通握手失败仍
只影响健康；停止未确认属于宿主责任异常，不能用“健康不阻塞 prompt”掩盖资源未释放。

本方案保证正常子进程和同组后代的超时清理，不新增强 containment 或旧 host/guardian 体系；恶意
自行脱离进程组及宿主崩溃后的完整后代纳管继承 Node 进程体系的已知边界，不能标为已解决。实现
必须单独给出所声明进程组范围的真实证据，不以杀直接 child 的结果替代。

## D9：三层状态按证据展示

| 事实 | 权威证据 | 不代表 |
| --- | --- | --- |
| 已配置 | Controller 保存的选择/配置状态；Node 还可能发现包缺失或解析失败 | ACP 已接收、服务可连接 |
| 已送达 ACP | Node 观察到对应 generation 的完整 `session/new`/`session/load` 成功响应，成为 Active | desired 已是最新、工具实际被调用、宿主健康 |
| 宿主健康 | 当前匹配身份的完整探测及清理结果 | Agent 自己已连接或调用成功、会话应被阻塞 |

Controller 只把当前匹配的 active/desired 显示为“当前配置已送达”；pending/blocked 时可显示“上次
送达版本”，并明确当前刷新状态。缺少新 Node 能力或当前观察时显示未支持/未知，不伪造失败或成功。
页面重载后 S 不变，但 Active 和健康必须重新取得当前观察；持久化选择不能变成持久化健康。

刷新失败提示阻塞原因和配置入口；探测失败是非阻塞提示，可显式重新检测。成员 ID、setting ID 和
稳定码用于定位，不显示设置值或第三方错误。权限检查沿用会话读取/操作授权；看得到运行不自动
等于有权改凭据或发起带凭据的重新检测。

## 为什么不是这些替代方案

| 方案 | 优势与未采用原因 |
| --- | --- |
| Controller 用 Go 重新解析并探测 | 较易接现有后端，但不能证明与 Node package view、workload 身份及解析结果一致 |
| 重发 start 或停止旧 execution 再新建 | 可复用旧入口，但前者按契约不更新值，后者改变会话上下文及执行生命周期 |
| 更新直接进入 durable command queue | 天然有 ACK/重放，但会把 Settings 写进 WAL；只持久化摘要也泄露值关系且不能恢复凭据 |
| 只做无值 invalidation，等下一次 prompt 取值 | 秘密路径少，但 idle 会话不会立即收敛，且 Node 没有 Controller 的配置存储 |
| 以可靠 Thread 事件保存健康 | 便于历史查询，但跨重启会制造过期健康；事件窗口背压还会延迟当前诊断 |
| load 成功即删旧包 | 磁盘占用更小，但 ACP 不证明旧 MCP 进程已退出；本提案保留到 Agent 停止并限制版本数量 |
| 所有 Session 共享一个探测缓存 | 少一次握手，但 Node 的独立会话可能有不同配置/授权；首期按 execution 隔离 |
| 直接复用现有 stdio child spawn | 改动少，但以 Node 身份执行以及只回收直接 child 都不满足工作负载与清理边界 |

## 不变量

1. S 在创建、挂接、provider 恢复和刷新中不变；缺元数据不触发 Automatic。
2. 最新合法 generation 先更新 Node admission，再回复受理；旧 generation 不回滚目标。
3. Settings 只存在于受权运行内存和必要的 ACP/探测外发，不进入任何持久或诊断表面。
4. 非 current 会话不能让新的 prompt 进入 Agent；已开始 prompt 不被 MCP 更新强行取消。
5. Active 只有完整 ACP 请求成功才能推进；晚到的成功不清除更新目标。
6. 当前健康来自相同执行/成员/版本/配置/cwd 身份的探测，且不决定 MCP 选择或 prompt 准入。
7. 进程、连接、租约和 view 有明确 owner；停止未确认不被超时或客户端断线抹掉。
8. 当前与被使用的旧包字节不被安装/移除替换；资源上限拒绝而不破坏旧使用者。
9. 断线重新查询，Node 重启仍 interrupted；不会从持久投影恢复 Active、健康或旧值。
10. 更新、观察和探测以当前 execution/control scope 为边界，不影响兄弟会话。

## 验收场景与评审重点

本 ADR 尚未 approved，以下是方案的可观察验收场景，尚不创建核心测试用例或宣称实现证据。

- reload/关联元数据丢失/空选择：S 不扩大；旧 start 保持冻结行为；新模式缺 capability 时不发送新输入。
- idle 更新、busy 更新、权限等待、两次更新穿插 ACP 成功：只在安全点 load；后续 prompt 看到最新目标。
- 断线时丢更新回复、重复/乱序/冲突更新、Controller 丢旧值、Node 重启：内存受理不被说成持久重放。
- prompt 的 generation 领先 Node、已有 accepted 排队后配置失效、EndSession 到达：拒绝/等待/丢弃的
  归属清晰；MCP blocked 不阻塞结束控制。评审特别核对 D4 的跨进程排序点，而非假定全局提交原子。
- 升级时旧包正在被 Agent/探测使用、两个会话使用不同版本、移除和同版本修复：不覆盖使用中的字节，
  不提前删 view；保留版本达到预算时明确 blocked。
- stdio/HTTP 的成功、401/5xx、协议错误、tools/list 失败、超大响应和挂死；同身份 single-flight，
  不同执行不串缓存；等待/取消/关闭/超时后验证同组后代和连接，停止未知时责任仍在。
- 在真实控制帧、ACP fixture、ledger/WAL、JSONL、日志、错误、查询及页面中追踪特征 Settings 值，
  只在允许的内存/外发边界出现。包版本保留不能顺便保留秘密。
- 三层状态分别故障注入：仅配置成功、控制帧写出、ACP 拒绝、ACP 成功但探测失败，不能互相代替。

## 风险、未解决的问题与落地顺序

Controller 必须维护每 Session 的 generation 与配置保存/轮次接纳顺序；这是新增调用方义务，不能
仅升级 Node binary 就宣称支持。内存回复需要查询对账，牺牲历史健康审计；历史证据归档另行设计。
包版本保留可能增加磁盘占用，并在 64 个身份后要求结束旧会话；评审需要确认这一有界取舍。探测
增加一次带凭据握手，接受其握手级副作用，但不允许工具调用或自动重试。

本文不解决 Node 崩溃后恢复原 Agent、强进程 containment、MCP-over-ACP、Agent 内部是否真正连接/
调用工具、全租户插件卡片的独立探测或后台周期复检。交互等待恢复仍由对应工作流/会话决策承接，
本方案只保证重新建立的会话继续使用原显式选择。

落地分为三步：

1. 完成本文评审，纠正模式/排序/租约/探测边界，标为 approved 后再在镜像目录维护核心用例，明确
   每项义务为 Missing，不以现有启动 setup 测试推导新能力已覆盖。
2. Node 实现与 PR：协议严格校验和兼容、内存配置源、Actor/barrier、版本租约与 view、工作负载探测
   launcher、无值状态查询及直接边界测试同批交付；所有行为具备后才广告能力。Desktop 原行为测试
   与 Node 门禁通过。Controller/v2 再按能力接入并独立验收。
3. 实现 review 通过后另开落地运行，按最终实现及真实测试修订 ADR 和核心用例；产品 PR 合并后再
   合并 implemented 状态。原冻结 setup、Thread 事件可靠性、Node 重启中断、checkout 共享围栏和
   Desktop MCP 行为保持各自既有承诺。
