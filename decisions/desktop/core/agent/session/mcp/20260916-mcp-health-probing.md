---
status: proposed
---

# Host 持有 MCP 运行健康：一次性握手探测，失败不改变投递语义

Ora 在安装完成、配置保存成功、会话建立三个时机，以 MCP 客户端身份对配置完整的已安装 MCP 执行一次
有界握手（`initialize` 与 `tools/list`），把「Server 运行健康」建立为 Host 自己持有的第三类事实，
并通过插件卡片独立状态行、会话内非阻塞横幅、secret-free 查询与 AppEvent 呈现给用户。会话横幅只
呈现该 Session 的 Effective MCP Set——普通聊天是自动发现的合格集合，工作流是冻结的显式白名单。
探测失败不改变任何投递语义：不失败 setup/refresh、不改变集合成员、不产生部分 `mcpServers` 列表。
健康状态只存在内存，呈现面消费健康抽象而非探测来源；传输实现是可复用的正式 MCP 客户端，而不是
一次性脚本。

`status: proposed` 表示本文尚在评审，不构成实现契约，也不维护核心测试用例。涉及模块为
`crates/backend` Session MCP（状态、触发、single-flight、身份）、`ora-contracts`（secret-free DTO、
查询与 AppEvent）、`ora-utils`（不含 Ora 领域词汇的 MCP 传输客户端，必要时以 Cargo feature 门控）、
`packages/app-shell`（插件卡片第三状态行与会话横幅）。兼容性边界：ACP 1.6.0 原生 `mcpServers`
wire contract、Effective MCP Set 资格、全量成败、Desired/Active 收敛、Agent Session Barrier、
Workspace 文件边界与 Setting 实值不得外泄全部不变。本文不承担文件系统布局或已发行版本的迁移义务；
进程重启后没有可信的健康历史，必须重新探测。

根决策把「Server 连接健康不属于 setup 成功语义」明确留在未解决问题中；本决策接手该边界，但
**不把健康重新解释为 setup 成功**。健康仍是与投递并行的独立通道。

## 继承与修改

本决策继承
[0-acp-session-setup-delivery](0-acp-session-setup-delivery.md)、
[20260912-workflow-explicit-selection](20260912-workflow-explicit-selection.md)
与
[0-static-mcp-configuration](../../../../plugin/category/mcp/0-static-mcp-configuration.md)
的资格、选择、投递与安装验证边界，只补充 Host 侧运行健康事实：

| 前序决策约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 根决策 D1：Effective MCP Set 由 installed snapshot 与配置完整性决定 | 继承，经工作流决策过滤 | 健康不是资格条件；探测结果不改变集合成员。卡片探测配置完整的已安装 MCP；会话横幅只看该 Session 过滤后的 Effective MCP Set |
| 根决策 D2：所有 new/load 共用同一无副作用 resolver | 继承，扩展消费方 | 探测只消费同一 resolver 的绑定产物，不存在第二条解析路径 |
| 根决策 D3–D4：Live 收敛与 Agent Session Barrier | 继承，不变 | 探测不进入 Barrier，不构成 refresh，不阻塞 prompt admission |
| 根决策 D5：只用 ACP 1.6.0 原生 `mcpServers` | 继承，不变 | 本文不启用 MCP-over-ACP，也不新增 Host/Agent 私有 MCP 方法 |
| 根决策 D6：canonical Plugin ID 与 `SessionMcpMemberRevision` | 继承，落实健康身份 | 健康身份与成员 revision 对齐；workspace context 额外绑定 Session `cwd` |
| 根决策 D7：入选成员按单个 setup/refresh 全量成败 | 继承，不变 | 探测失败绝不触发 setup 失败或部分列表；分辨率失败仍按 D7 全量失败 |
| 根决策 D8–D9 与不变量 11：事实分离、脱敏、不物化、收敛状态只在内存 | 继承，落实第三事实 | 健康是第三个独立事实，同样只在内存，不得命名为 `MCP Ready` |
| 根决策「未解决：Server 连接健康不属于 setup 成功语义」 | 接手 | 健康仍不属于 setup 成功；本文定义 Host 侧事实来源、时机与呈现 |
| 工作流决策 D1：Automatic 与 Explicit 选择 | 继承，探测与选择正交 | 安装/保存探测服务插件卡片，不改变会话选择。会话补探测与横幅只覆盖该 Session 的 Effective MCP Set；Explicit 空集合不注入 MCP，也不出现 MCP 横幅 |
| 工作流决策 D2：选择随 Session 持久化 | 继承，不变 | 会话补探测读取 Session 自身的选择，不从工作流草稿或节点运行关系反查 |
| 工作流决策 D3：发送边界记录 secret-free 选择身份 | 继承，并列探测结果 | D3 仍只证明 Host 发送了什么；本决策追加 Host 探测结果，两者都不证明 Agent 已落实 |
| 工作流决策 D4：安装不等于节点授权 | 继承，不变 | 安装触发的探测不构成工作流授权；未选择的 MCP 即使卡片 Unhealthy 也不进入该工作流会话横幅 |
| 静态决策 D5：安装期验证不执行 command、不发起 HTTP | 澄清，不改验证语义 | 验证路径仍不探测；installed snapshot 提交且成员已合格之后允许异步探测，结果不回写 descriptor 或资格 |

## 术语与边界

- **Host MCP Health** 是 Host 对配置完整的已安装 MCP 执行有界握手得到的运行健康事实。它不是
  Configured MCP Eligibility，不是 Active Session MCP Revision，也不是 Agent 在会话内已经连上
  Server 或工具已经对模型可见。会话横幅只展示该 Session Effective MCP Set 内的健康结果。
- **MCP Health Probe** 是一次完整的 MCP 客户端会话：按该成员的 transport 完成 `initialize` 与
  `tools/list`，然后拆除连接并回收进程。它证明 Host 视角下该绑定产物此刻可以握手，不证明后续
  Agent 进程一定能复现同一结果。
- 健康身份的成员维与 `SessionMcpMemberRevision` 对齐：canonical Plugin ID、精确 package version、
  Configuration revision、transport。依赖 `{ "context": "workspace" }` 的成员额外绑定本次 Session
  的绝对 `cwd`；没有真实 `cwd` 的呈现面不得伪造探测。
- Ora 负责探测、缓存与呈现；Agent 仍按 ACP 自行连接并运行 Server。探测不是 MCP 流量代理，也不把
  Agent 侧成功或失败回写为 Host 健康。

## 背景：发送之后的运行期盲区

根决策把 Ora 的保证停在 ACP 帧写出之前。安装验证只证明静态 containment；setup/refresh 的 resolver
对绑定、能力与文件 containment fail-closed。帧离开 Host 之后，运行健康没有一等事实来源：

1. **协议无回执。** ACP `NewSessionResponse` / `LoadSessionResponse` 没有 MCP 状态字段；
   `SessionUpdate` 也没有 MCP 相关变体。规范只说 Agents SHOULD connect to all MCP servers，连不上时
   报错、拒绝还是跳过未定义。「静默跳过、会话照常建立」完全合规。
2. **Host 管道对未知通知不透明。** `agent/acp` relay 不解析 payload。ACP peer 对未知
   notification 直接丢弃，对未知 request 返回 JSON-RPC `-32601`。即便某个 Agent 自发上报 MCP
   状态，当前也没有消费通道。
3. **MCP 进程归 Agent 所有。** spawn、握手、崩溃、重连都发生在 Agent 侧，Host 没有进程句柄与退出码。
4. **失败证据只进 operator 日志。** Agent CLI 的报错经插件 stderr 进入 runtime log，非结构化，无法
   作为关联插件身份与会话的用户通知。

这不是 Ora 独有的缺口。Claude Code headless 模式对 MCP 连接失败完全静默
（[claude-code #43968](https://github.com/anthropics/claude-code/issues/43968)）；
claude-agent-acp 对 session 级注入的 MCP 从未调用 `tools/list`
（[claude-agent-acp #883](https://github.com/agentclientprotocol/claude-agent-acp/issues/883)）。
MCP 健康感知稳定的产品，共同点是客户端自己持有连接。本决策采取与该共同点相容、且不破坏根决策
投递语义的最小步骤：Host 先自己连一次，把结果当作只在当前进程有效的内存事实呈现出来。

## 决策

### D1：健康是第三个独立事实，由 Host 一次性探测建立，异步于触发路径

系统对每个 MCP 插件维护三个互不推导的事实：安装与配置完整（既有资格事实）、Session 已接受
Snapshot（既有收敛事实）、Server 运行健康（本决策新增）。健康由 Host 以 MCP 客户端身份执行一次
完整连接尝试建立，不询问 Agent，不依赖 Agent 上报。

配置不完整的插件保持未探测，插件卡片继续只展示既有的「需要配置」，不展示健康行。无 Settings 的
MCP 在安装完成后即合格；带 Settings 的 MCP 必须在 Configuration revision 状态为 `Complete` 之后才进入
探测。安装与保存探测的是「配置完整的已安装 MCP」，服务于插件卡片，与某次 Session 是否选择它无关。

会话建立时的补探测和横幅只覆盖该 Session 的 Effective MCP Set：普通聊天（`Automatic`）是全部合格
成员；工作流（`Explicit`）是冻结白名单中仍然合格的成员。Explicit 空集合不注入任何 MCP，因此不发起
会话补探测，也不展示 MCP 横幅。显式选择的插件缺失、声明无效、配置不完整或 Agent 能力不足时，仍按
工作流决策 D1 使该节点失败；那是选择/资格错误，不是本决策的 Unhealthy。

触发点如下。安装、保存、会话建立均不得因等待探测而阻塞：

| 触发 | 探测对象 | 是否阻塞触发路径 | 说明 |
| --- | --- | --- | --- |
| 安装完成 | 新合格成员 | 否 | 拦截「包本身不可运行」（架构不符、缺动态库、非 MCP 程序） |
| 配置保存成功且变为 `Complete` | 该成员的新身份 | 否 | 直接回答「刚填写的值能不能用」 |
| 会话建立（`session/new` 或 `session/load`） | 该 Session Effective MCP Set 中仍为 `Unknown` 的成员 | 否 | 为横幅补齐 Host 视角事实；已有匹配身份的结果直接复用 |
| 用户「重新检测」 | 指定合格成员 | 可以等待同一次 single-flight | 仍受硬超时约束；这是用户发起的同步查询，不是 setup 的一部分 |

成员卸载、更新到新 package version、Configuration revision 变化或失去资格后，旧身份上的健康结果
立即失效，不得继续呈现。新身份从 `Unknown(not_probed)` 开始，并由对应触发点重新探测。

会话建立读取的是内存中与当前身份匹配的结果。身份不匹配视为 `Unknown(not_probed)`，而不是过期；
本文不引入 TTL 或后台周期复检。

选择异步触发，是因为探测包含 spawn 或外发 HTTP，把等待放进安装、保存或 setup 会把运行期抖动放大
为会话不可用，从而违反根决策 D7 的投递粒度。用户显式重测是唯一允许等待的路径，因为它的完成条件
就是这次探测本身。

### D2：探测复用 Snapshot 绑定产物，执行有界的完整 MCP 握手

探测输入是 resolver 的既有产物：`resolve_mcp_transport` 输出的 command/args/env 或 url/headers，
加上 D1 的健康身份。探测把同一份产物解释为 MCP 客户端语义，而不是重新解析配置，以保证「探测的
对象」与「发给 Agent 的对象」是同一事实。

stdio 与 HTTP 使用同一健康定义：按该 transport 的 MCP 协议完成 `initialize`，并成功调用
`tools/list`。探测只执行这两个 MCP 方法，不调用其他方法。握手完成后拆除连接；stdio 必须
关闭 stdin 并在超时边界内确认进程回收，避免探测留下僵尸进程。

- **stdio**：以解析产物 spawn 进程，使用换行分隔 JSON-RPC 完成上述两个方法。
- **HTTP**：向解析出的端点按 Streamable HTTP 完成同一对方法，足以区分 DNS/连接/TLS 失败、
  401/403、其他 4xx/5xx、协议错误、超时与工具列表失败。
- **超时**：硬超时。超时返回 `mcp_probe_timeout`，触发路径照常继续。超时上限必须明显短于
  session setup 的 30 秒预算，使探测不能在时间上冒充 setup。
- **single-flight**：同一健康身份的并发触发共享一次探测，不重复 spawn 或重复外发。
- **workspace context**：args 依赖 `{ "context": "workspace" }` 的成员在没有真实 Session `cwd`
  时保持 `Unknown(context_missing)`。插件卡片没有 Session `cwd`，因此这类成员在卡片上不得显示
  Healthy/Unhealthy，也不得对「重新检测」伪造占位 cwd。会话横幅使用该 Session 的绝对 `cwd` 探测。

不把 HTTP 探测缩成只做 `initialize`：健康码包含 `mcp_tools_unavailable`，若 HTTP 不调用
`tools/list`，该码对一种 transport 无定义，两种 transport 就不再共享「可握手且可列出工具」这一
事实。

```text
身份匹配且探测成功     → Healthy
身份匹配且探测失败     → Unhealthy{code}
合格但尚未探测         → Unknown(not_probed)
合格但缺少真实 cwd     → Unknown(context_missing)
不合格或身份已消失     → 不展示健康
```

### D3：探测失败不改变投递语义

探测失败绝不失败 `session/new` / `session/load`、不把成员移出 Effective MCP Set、不产生部分
`mcpServers` 列表。投递语义完全维持根决策：分辨率失败按 D7 全量失败，运行期失败由 Agent 自行
处理，Host 照发完整列表。

健康呈现是与投递并行的独立通道。这是感知增强，不是投递收紧。运行期失败的正确粒度是非阻塞提示，
而不是把还能使用的会话整体废掉。

反例：HTTP MCP 的 API Key 填错。保存后卡片展示 `mcp_http_unauthorized`，随后建立的会话仍把该
成员放进完整 `mcpServers`；横幅提示 Host 视角不可用，但 setup 成功，用户仍可在该会话里使用其他
工具。若把探测失败升级为 setup 失败，就会让一个远端鉴权问题阻断整个会话，把根决策已经 fail-closed
的分辨率错误与运行期错误混成同一粒度。

### D4：状态模型只在内存，绑定成员身份，使用封闭错误码族

健康状态为封闭枚举 `Healthy / Unhealthy{code} / Unknown{reason}`，绑定 D1 的健康身份。状态只存在
内存：Ora 重启后全部回到 `Unknown(not_probed)`，不以持久化数据冒充当前健康。这与根决策 D9 对
收敛状态的处理一致。

`Unhealthy` 的稳定错误码一次设计为探测失败与未来活连接死亡两用（词表来源中立）。活连接本身不在
本文落地；本文只保证码表不绑定「一次性探测」这一数据源。错误码不得与既有 setup 稳定码碰撞
（`mcp_load_capability_missing`、`mcp_http_capability_missing`、`mcp_setting_missing`、
`mcp_illegal_runtime_text`、`mcp_command_not_in_package`、`mcp_workspace_cwd_unresolved`、
`mcp_revision_changed`、`mcp_configuration_unavailable`、`mcp_catalog_unavailable`）。

| 码 | 含义 |
| --- | --- |
| `mcp_spawn_failed` | stdio 进程无法启动 |
| `mcp_exited_prematurely` | 启动后在握手完成前退出 |
| `mcp_handshake_failed` | `initialize` 无响应或协议错误 |
| `mcp_probe_timeout` | 硬超时 |
| `mcp_tools_unavailable` | 握手成功但 `tools/list` 失败 |
| `mcp_http_unreachable` | DNS / 连接 / TLS 失败 |
| `mcp_http_unauthorized` | HTTP 401 / 403 |
| `mcp_http_server_error` | 其他 HTTP 4xx / 5xx |

`Unknown` 的 reason 只使用 `not_probed` 与 `context_missing`。新增细分码是对外兼容性变更，必须
另作决策，不能在实现中私下加码。

超时与握手失败分开，是因为「对端慢」和「对端不是 MCP」对用户与 operator 的下一步不同。spawn
失败与 `mcp_command_not_in_package` 也分开：后者是 setup 的包内文件资格错误，前者是文件存在但
当前平台无法启动。

### D5：感知面由插件卡片、会话横幅、查询、AppEvent 与配对日志构成

呈现面只消费健康抽象，不绑定探测是否来自安装、保存、会话或手动重测。

- **插件卡片独立第三状态行**：在安装状态与配置完整性之外呈现健康（含错误码与「重新检测」）。
  不得并入或改写前两个事实的展示，不得命名为 `MCP Ready`。配置不完整或已失去资格的插件不展示
  该行。依赖 workspace context 的成员在卡片上保持 `Unknown(context_missing)`，「重新检测」不得
  用占位 cwd 把它变成 Healthy/Unhealthy。
- **会话内非阻塞横幅**：setup 完成后呈现该 Session Effective MCP Set 中的 Unhealthy 成员与错误码，
  并提供跳转配置入口。交互式聊天与工作流会话都使用同一呈现，但成员范围遵循该 Session 的选择：
  普通聊天看自动发现集合，工作流只看冻结白名单。未选择的 MCP 即使在插件卡片上 Unhealthy，也不
  进入该工作流会话横幅。横幅不阻塞 prompt，也不把探测未完成渲染成失败。
- **secret-free 查询**：查询按可选 `cwd` 返回当前合格成员的健康。无 `cwd` 对应卡片视角；有
  `cwd` 对应该 Session 横幅。返回值只含身份、状态、稳定码或 Unknown reason，不含 Setting 实值、
  凭据、argv、env、headers 或第三方响应体。
- **AppEvent**：新增 `McpHealthChanged { plugin_id }`，对齐 `PluginStatusChanged`：只通知前端
  按身份重新查询，不携带状态本体。
- **配对日志**：setup/refresh 发送路径增加一条可与该次 Session 关联的结构化结果事件，字段限于
  成员身份、稳定码、耗时。它补充「Host 发出了完整列表」这一既有事实，而不是改写现有 debug 日志
  的语义。
- **原始错误文本**：OS 与第三方响应文本只进 operator 日志；UI 与公开 DTO 只携带稳定错误码与
  身份。措辞必须是 Host 视角的「当前连不上」或「工具列表失败」，不承诺「会话内已生效」。

卡片与横幅拆开，是因为同一成员在无 cwd 与有 cwd 下不是同一个健康身份。把 Session 探测结果写回
全局卡片，会让一个 Workspace 的失败污染另一个 Workspace 的展示。

### D6：探测可见实值，但实值不出探测执行体；副作用有界

探测进程与 HTTP 请求使用解析后的真实 env 与 headers，与发给 Agent 的完全相同，否则探测无效。
Setting 实值与凭据不得进入探测结果、日志、事件、UI DTO、digest 或查询响应；错误只含稳定码与
身份。

HTTP 探测是对用户配置端点的一次带凭据外发请求，与 Agent 实际使用时的流量同一目的地。配置保存
成功触发一次探测，不自动重试。手动「重新检测」与会话建立时对 `Unknown` 的补探测是仅有的再次
探测来源。卸载、失去资格或身份变化会丢弃旧结果，但不构成对远端的额外请求。

接受握手级副作用（stdio 多跑一份短进程，HTTP 多一次 initialize/tools 往返），拒绝更深的探测
方法，并把自动重试排除在外，是为了把付费 API 与有状态 Server 的额外成本限制在用户可预期的
触发点上。

### D7：探测传输层是正式客户端组件，状态与呈现面与数据源解耦

一次探测等于一次完整的 MCP 客户端会话，因此：

1. stdio/HTTP 传输实现（spawn + 换行 JSON-RPC、Streamable HTTP 的 `initialize` / `tools/list`）
   写成可长期复用的正式组件，不写一次性脚本。组件不含 Ora 领域词汇，按仓库规则进入 `ora-utils`
   （必要时以 Cargo feature 门控）。它不得放进 `plugin-manager` 的安装验证路径，以免把静态发现
   重新变成有外部副作用的安装。
2. 健康状态模型与 D5 全部感知面消费「健康状态」抽象，不绑定探测来源。未来若以活连接事件替换
   数据源，状态模型、错误码、AppEvent、查询与 UI 保持同一契约。
3. 健康身份与 `SessionMcpMemberRevision` 对齐。这是迁移钉子，不是现在实现连接池。

传输与领域状态分开，是为了让「如何说话」和「这句话对哪个插件身份为真」可以独立演进。把探测
写成 Session MCP 内部闭包，会在将来替换数据源时连卡片、横幅和错误码一起拆掉。

### D8：现在不采用 MCP-over-ACP；采用时必须另立 ADR

MCP-over-ACP（ACP RFD `unstable_mcp_over_acp`）一旦被主流 Agent 支持，可以把连接权收回 Host，
使失败成为 Host 的一等错误，并使密钥不再随 `mcpServers` 帧交给 Agent。那是根决策 D5 的真实修改，
不是本决策的实现范围。

现在不做，因为生态尚无 Ora 依赖的对手方、协议仍标记为 unstable，而且连接生命周期是与 Agent
Runtime 同量级的新受监管状态类。用当前工程量换一个上线即闲置的双模式分发，会把未稳定协议写成
当前承诺。

本决策为此只留下约束：

- 不得在本文落地 MCP-over-ACP、`mcp/connect` 路由或长活连接池。
- 未来若采用，必须另立 ADR；该 ADR 必须复用本文的健康抽象、错误码、感知面与传输组件。
- Desired/Active 收敛与 prompt admission 不变量保持根决策原样；按 Agent 能力双模式并存的细节
  留给那篇 ADR。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 维持现状（失败只进 Agent 日志） | 零成本，不增加握手副作用 | 用户无法区分「Agent 没用工具」与「MCP 已死」；违反本决策要把运行健康做成可观察事实的目的 |
| 依赖 Agent 上报（自定义通知或推动上游扩展 `SessionUpdate`） | 语义最准，报告的是真实会话内状态 | ACP v1 无 MCP 状态字段；未知 notification 被丢弃；需要逐家 Agent 配合，覆盖不了不配合者，无法成为 Host 的一手事实 |
| 直接实现 MCP-over-ACP | 失败成一等错误，密钥可不出 Host | 无对手方、unstable 协议、全新状态类；用最大工程量换一个上线即闲置的功能，且会修改根决策 D5。见 D8 |
| 探测失败硬失败会话 | 严格，绝不带病运行 | 把可用会话整体废掉；运行期事实劫持投递语义，违反根决策 D7 与不变量 1 |
| 解析 Agent stderr 反推 | 零协议改动 | 逐家格式脆弱、无结构化保证、无法关联插件身份；不能作为通知通道，只配做 operator 诊断辅助 |
| 同步探测阻塞 session setup | 状态在 prompt 前一定「最新」 | 多成员串行 spawn + 握手放大 setup 延迟；探测抖动被放大为会话不可用，违反 D1 与不变量 8 |
| 安装验证路径内探测 | 安装结果即可宣称可运行 | 把无副作用的包发现改写成网络/代码执行；短暂不可用会错误改变 installed snapshot，违反静态决策 D5 |
| 为 HTTP 省略 `tools/list` | 少一次远端往返 | 两种 transport 不再共享同一健康定义，`mcp_tools_unavailable` 失去封闭性 |

## 风险与代价

- **探测通过不等于会话生效。** Agent 端仍可能失败；探测失败也不等于 Agent 一定连不上（瞬时抖动）。
  呈现必须保持 Host 视角，不能把第三事实宣传成 Active revision 或工具已对模型可见。该缺口只能由
  未来的 MCP-over-ACP 或 Agent 上报缩小，不在本文消灭。
- **有状态 MCP 的探测副作用。** 探测等于多跑一份短进程或多一次 HTTP 握手。以「只做 initialize +
  tools/list、硬超时、single-flight、无自动重试」收敛；接受握手级副作用。
- **HTTP 凭据外发。** 探测与实际使用同一目的地；保存一次、无自动重试；结果不含响应体。仍须在
  落地时做脱敏审查，证明任何表面都没有实值。
- **错误分类的平台差异。** spawn 失败在 Windows/Unix 上的 OS 语义不同，稳定码映射必须跨平台一致
  并有测试；不得把本地错误字符串送进 UI。
- **维护负担。** 错误码族是对外稳定承诺，新增细分码是兼容性变更。

本文不改写已安装包、配置存储或 Workspace 文件，因此没有历史状态迁移。唯一需要「重建」的是内存
健康：进程重启后全部回到 `Unknown(not_probed)`，由 D1 的触发点重新建立。这是有意为之，避免把
上一次进程的探测结果当成当前健康。

## 不变量

1. 探测结果不改变 Effective MCP Set 成员资格、不使任何 setup/refresh 失败、不产生部分 `mcpServers`
   列表。
2. 运行健康是独立事实，不与安装、配置完整或 Session 接受 Snapshot 合并呈现，不得命名为
   `MCP Ready`。
3. 健康身份绑定 canonical Plugin ID、精确 package version、Configuration revision 与 transport；
   依赖 workspace context 的成员额外绑定 Session 绝对 `cwd`。身份不匹配的结果不得当作当前状态
   呈现。
4. 探测使用与 ACP 投递相同的绑定解析产物；系统不存在第二条 MCP 配置解析路径。
5. 探测只执行 MCP `initialize` 与 `tools/list`，带硬超时；同一身份的并发触发共享一次探测；stdio
   探测进程必须确认回收。
6. Setting 实值与凭据可以到达探测执行体，但不得进入探测结果、日志、事件、查询响应、UI DTO 或
   digest；公开呈现只含稳定错误码与身份。
7. 健康状态只存在于内存；Ora 重启后回到 `Unknown(not_probed)`，不得以持久化数据冒充当前健康。
8. 安装、配置保存与会话建立均不得因等待探测而阻塞。
9. 依赖 workspace 上下文的成员在无真实 Session `cwd` 的触发点保持 `Unknown(context_missing)`，
   不得以占位 cwd 伪造探测。
10. 成员卸载、更新、configuration revision 变化或失去资格后，旧身份的健康结果立即失效，不得继续
    展示。
11. 配置不完整的插件不被探测，也不在插件卡片上展示健康行。
12. 探测与 Session MCP Selection 正交：安装/保存探测不改变会话选择；会话横幅只呈现该 Session
    Effective MCP Set 中的 Unhealthy 成员；Explicit 空集合不出现 MCP 横幅。
13. MCP-over-ACP 落地必须另立 ADR；届时 Desired/Active 收敛与 prompt admission 不变量不变。

## 落地顺序

- **1a 地基。** 错误码族、健康状态模型、MCP 客户端传输组件。完成条件：stdio/HTTP 的成功与失败
  矩阵（spawn 失败、提前退出、握手失败、超时、`tools/list` 失败、401、5xx、非 MCP 响应）有组件
  级测试证据；传输组件不含 Ora 领域词汇。此阶段用户可见行为不变。
- **1b 触发与呈现。** D1 触发点、查询、`McpHealthChanged`、插件卡片第三状态行、会话横幅、配对
  日志。完成条件：下方验收场景全部有行为证据；脱敏审查通过（任何表面无实值）。此阶段开始出现
  卡片健康行与会话横幅；投递、收敛、Barrier 与 Workspace 文件行为保持不变。

拆成 1a/1b，是为了先在没有 UI 的情况下固定握手语义和错误码，避免把传输分类争论带进呈现评审。

## 验收场景

- 安装架构不符的 stdio MCP：安装完成即呈现 `mcp_spawn_failed`；会话仍可建立；横幅提示该成员
  Host 视角不可用；该会话的 `mcpServers` 仍包含该成员。
- HTTP MCP 填错 API Key：保存后插件卡片呈现 `mcp_http_unauthorized`；查询、事件、日志与 UI 均
  不含 Key 原值与响应体。
- required Settings 仍不完整：不发起探测；卡片不展示健康行，只保留既有的配置不完整展示。
- 健康成员：无任何新增阻塞或提示；探测进程握手后确认回收，无僵尸进程。
- 探测挂死：硬超时内返回 `mcp_probe_timeout`，安装、保存与会话建立均未被阻塞。
- workspace-context 成员：安装或保存后卡片为 `Unknown(context_missing)`；会话建立后以真实 `cwd`
  探测；不得出现占位 cwd。
- 同一成员并发触发（保存与建会话同时）：只执行一次探测，两处共享结果。
- 用户「重新检测」：等待同一次 single-flight 结束（受硬超时约束）后查询到新结果。
- Ora 重启后：所有健康状态回到 `Unknown(not_probed)`，界面不显示旧状态。
- 卸载或失去资格后：旧健康立即从卡片和横幅消失，不得残留 Unhealthy。
- 探测失败成员的 session setup：仍发送完整 `mcpServers`；该次 Session 可关联到 secret-free 的
  Host 探测结果日志。
- 工作流显式选择会话：横幅只列出该 Session 白名单中的 Unhealthy 成员；未选择的 MCP 即使卡片
  Unhealthy 也不出现。选择语义与工作流决策的失败语义不变。
- 工作流 Explicit 空集合：不发起会话补探测，不展示 MCP 横幅，发送空 `mcpServers`。
- 探测失败场景的全量日志、事件、查询与 UI 断言：不含 Setting 实值、凭据或第三方响应文本。
- 措辞：任何新增文案都不把 Host 探测成功说成「会话内已生效」或 `MCP Ready`。

## 本决策未解决的问题

- **后台周期复检与缓存 TTL。** 本文明确不引入过期；是否需要周期复检、付费 API 频控，要等用户
  反馈后另立决策。提前写 TTL 会与「重启即 Unknown」和「无自动重试」打架。
- **工作流严格模式 / require-mcp。** 节点能否声明「该 MCP 必须健康才允许启动」，以及它与工作流
  显式选择失败语义（缺失/不完整已经 fail closed，运行期不健康仍放行）的关系，需要新的用户意图
  模型，超出本文。
- **MCP-over-ACP 的采用决策。** 双模式分流、连接管理器生命周期、连接持有期死亡如何通知 Agent，
  均在采用时另立 ADR。RFD 本身尚未定义 Host 如何主动告知 Agent 底层连接已死。
- **探测结果的持久化诊断导出**（如工单附件）是否需要。当前 operator 日志已保留原始错误文本；
  导出格式会重新打开脱敏与留存边界。
- **Agent 会话内真实连接状态。** 本文只建立 Host 视角事实；缩小「探测通过 ≠ 会话生效」的缺口
  依赖 Agent 上报或 MCP-over-ACP，不能由更多 Host 侧探测消除。

## 核心测试依据

`status: proposed` 阶段不编写核心测试用例。评审通过并将状态改为 `approved` 后，依据最终决策
补充至 `specs/test-cases/desktop/core/agent/session/mcp/mcp-health-probing.md`。该路径与本 ADR
所在领域目录镜像，当前尚未创建文档。
