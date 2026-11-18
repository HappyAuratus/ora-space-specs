---
status: proposed
---

# Controller 经 AgentSessionSpec 交付宿主 MCP，Node 在每次 ACP setup 附加

Controller 在 `start_agent_session` 的 `AgentSessionSpec.host_mcp` 中交付自己提供的 HTTP MCP Server
（名称、URL 与请求头），Node 把它们原样附加到该会话的每一次 ACP setup：`session/new`、`session/load`、
生命周期刷新的 load 以及 Agent 重启。它们是 [Host Session Capability](../../../domain/glossary.md#session-mcp)
在 Node 上的承载方式，不是插件 MCP：不进入 `mcp` 显式选择，不受插件白名单、上限和刷新协议约束，也不
改变“空选择不注入插件 MCP”。URL 与请求头只在内存中存在；Node 不执行宿主命令、不探测宿主 URL。首个
使用者是工作流交互节点的完成工具：经 Node 执行的工作流也能获得与 Desktop 相同的
[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md)。

本决策为 `proposed`，等待评审。影响 `node-protocol` 的会话 spec 与能力枚举、Node 会话宿主对共享运行时
`SessionSetup::host_mcp_servers` 的实现，以及 Node 对执行输入的持久投影；不改变 Desktop Backend 自己注入
宿主能力的路径，不改变插件 MCP 的解析、生命周期、刷新、健康探测或失败语义。旧 Node 的
`AgentSessionSpec` 使用 `deny_unknown_fields`，因此 Controller 只向声明新能力的 Node 交付非空列表；
省略或空列表的会话与现状完全相同。不迁移历史会话，不恢复 Node 进程重启前的 Agent。完成工具本身的
语义（attempt token、信号绑定、确认与审计）由提供它的 Controller 按完成信号决策实现，不在本 ADR。

## 继承与修改

承接本目录前序[会话 MCP 刷新与宿主健康](20261116-session-mcp-refresh-and-host-health.md)，经它继承
[显式 Session MCP](20261114-explicit-session-mcp-in-agent-session-spec.md)与
[共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)；宿主能力的边界继承
[Agent 完成信号](../../desktop/core/workflow/20261110-agent-completion-signal.md) D1。

| 前序约定 | 处理 |
| --- | --- |
| 宿主拥有进程、会话和授权生命周期 | 继承，不变；宿主能力的授权仍由提供它的宿主（这里是 Controller）负责，Node 只负责送达 |
| 显式 Session MCP D1：`mcp` 是有序完整的插件选择，最多 16 个 Server，空列表不注入插件 MCP | 继承，不变；`host_mcp` 是独立字段，不计入 16 项，不参与插件 ID 排序与唯一性 |
| 显式 Session MCP D3：Setting 值只在运行时边界存在，持久输入只留身份 | 继承并扩展：`host_mcp` 的 URL 与请求头同样只在内存中，持久输入只留 Server 名称 |
| 显式 Session MCP D4：Node 与 Agent 能力门禁分离，能力错误在 `initialize` 后、`session/new` 前 | 继承并扩展：非空 `host_mcp` 要求 Agent 声明 ACP HTTP MCP，缺少时以新的稳定 code 失败 |
| 显式 Session MCP D5：stdio 命令必须属于已安装包的 `assets/`，不信任 Controller 提供的主机命令 | 继承，不变；宿主 MCP 只接受 HTTP，没有 stdio 形态 |
| 刷新 D1/D2：`mcp_lifecycle` 会话的绑定通过 `UpdateSessionMcp` 按 generation 更新 | 继承，不变；宿主 MCP 不在更新输入里，会话内不可更新 |
| 刷新 D4/D6：刷新只 `session/load` 原 provider ID；版本变化或第九次 load 时重启本 execution 的 Agent | 修改：每一次这类 ACP setup 都附加同一份宿主 MCP 列表 |
| 刷新 D7–D9：Host MCP Health 只针对已安装插件 MCP | 继承，不变；宿主 MCP 不被 Node 探测，也不产生健康事实 |
| 完成信号 D1：完成工具是独立宿主能力，不进入插件白名单；授权来自节点策略与单 attempt token；进程级注册表可能让工具在别处可见 | 继承；本文把同一边界搬到 Node 承载的会话，见 D5 |
| 共享 checkout 会话：共享的是 checkout，不是会话身份、进程或历史 | 继承，不变；宿主 MCP 不改变每会话独立插件进程的承诺 |

[协议根决策](../protocol/0-controller-node-protocol.md)的身份、帧上限及能力协商继续有效；
[会话执行与命令](../protocol/20260928-streamed-thread-events-and-session-commands.md)的先持久受理、
终态与重放继续有效，“不持久化输入”的例外在显式 Session MCP 的 Setting 之外再增加本文的宿主 MCP。

## 问题与约束

Desktop Backend 直接与 Agent 进行 ACP 通信，它在 `session/new`、`session/load` 时通过共享运行时的
`SessionSetup::host_mcp_servers()` 附加自己在本机 loopback 上提供的完成工具。经 Node 执行的会话由 Node
与 Agent 通信，Node 的会话宿主使用这个方法的默认空实现；`AgentSessionSpec.mcp` 只能选择 Node 已安装的
插件包，其 HTTP transport 的 URL 写死在包内且必须是 https。Controller 因此没有办法把自己提供的工具交给
Agent。

把宿主能力做成插件包会把它挂到租户插件的安装、市场和 MCP 生命周期上，违背完成信号 D1“它不是插件
MCP”的边界。要求 Node 解析工作流语义也不合适：Node 只负责送达，授权和工具语义留在宿主。宿主 Server
的 URL 可能把凭据放在路径里，请求头可能带 bearer token；Node 不持久化凭据的要求与执行输入整体持久化
仍然冲突，必须照 Setting 值的规则在首次数据库写入前分离。

## D1：wire 形状与能力协商

`host_mcp` 是可省略的列表；省略和空列表都表示没有宿主 MCP，行为与现状完全相同：

```json
"host_mcp": [
  {
    "name": "ora-workflow-completion",
    "url": "http://127.0.0.1:41234/mcp/3f1c…",
    "headers": { "authorization": "Bearer …" }
  }
]
```

- 每项只有 `name`、`url`、`headers`；`headers` 必填，没有请求头时是空对象，未知字段拒绝。
- `name`：1–64 字节，只含小写 ASCII 字母、数字和 `-`，以字母开头，列表内唯一，按 UTF-8 字节序严格递增。
  canonical Plugin ID 总是 `<namespace>/<name>`，而 `name` 不含 `/`，因此宿主 Server 名永远不会和插件
  MCP 的 Server 名冲突，ACP 中两类 Server 可以并列而不需要额外的冲突检查。
- `url`：绝对 URL，不含 userinfo 与 fragment，长度最多 2 KiB。scheme 为 `https`，或为 `http` 且 host 是
  字面 loopback 地址（`127.0.0.0/8` 或 `::1`）；其他 `http` URL 拒绝。
- `headers`：最多 8 项；名称是小写的 HTTP token，值只含可见 ASCII 与空格，不含 CR、LF；所有 URL 与请求头值
  的 UTF-8 字节数合计最多 8 KiB。
- 一次会话最多 **4** 个宿主 Server。

Node 在 `HelloAccepted.capabilities` 广告 `session_host_mcp`；Controller 只向同时声明它和 `agent_session`
的 Node 发送非空 `host_mcp`。Controller 必须在发送前镜像上面的全部规则。结构、上限或格式不合法是 spec
校验错误，按显式 Session MCP D4 的帧校验失败处理：Node 以协议错误关闭控制连接，不受理该执行，不启动
Agent。

**为什么放宽到 loopback 明文 http**：插件 MCP 的 URL 由第三方包声明，要求 https 是为了不让包把流量
引到明文端点；宿主 Server 由 Controller 自己提供，loopback 明文只在 Controller 与 Node 同机时有意义，
此时 https 只增加一套证书分发而不提高机密性。非 loopback 一律要求 https，远程部署不会退化为明文。

## D2：独立于插件 MCP，每次 ACP setup 原样附加

- Node 的会话宿主实现 `SessionSetup::host_mcp_servers()`，返回由 `host_mcp` 映射出的 ACP HTTP Server
  （名称、URL、请求头），顺序与输入相同，放在插件 MCP 之后。
- 该会话的每一次 ACP setup 都附加同一份列表：首次 `session/new`、`session/load`、`mcp_lifecycle` 刷新时
  的 load、刷新 D6 的 Agent 重启。插件 MCP 刷新只改变插件部分，不重建、不增删宿主部分。
- 宿主 MCP 不进入 `mcp` 选择、Effective MCP Set、Desired/Active revision 或 `GetSessionMcpState` 快照，
  不计入 16 项上限，不受插件白名单约束；“空 `mcp` 不注入任何插件 MCP”的语义不变。
- 会话内不可更新：`UpdateSessionMcp` 不携带宿主 MCP，也没有单独的更新命令。宿主要更换 URL 或凭据，
  就结束当前 execution 再启动新的。

**为什么不跟插件 MCP 共用刷新协议**：宿主能力的授权来自宿主自身（例如节点冻结策略与单 attempt
token），不是可变的插件绑定；为它引入 generation、blocked 状态和查询会把宿主语义带进 Node 的插件生命周期，
而目前的使用者在会话内不需要变化。一旦附加，每次 setup 都必须带上，否则一次刷新就会让 Agent 失去工具。

## D3：URL 与请求头只在运行时边界存在

- `host_mcp` 的 URL 与请求头只经 `start_agent_session` 控制帧进入 Node 内存，并在 ACP setup 请求中交给
  Agent。不得进入 ledger（包括 SQLite WAL）、会话 JSONL、Thread、日志、错误链或输入对象的 `Debug`。
- 持久输入只保留 Server 名称列表。重复 start 比较名称列表，不比较 URL 与请求头；重发相同 execution 不替换
  正在使用的值。运行值与持久投影必须在首次数据库写入前分开。
- 受理后 Node 进程崩溃时不从持久投影启动 Agent，按原规则结算 `interrupted`；不要求 Controller 重放宿主
  凭据。宿主在新的 execution 中重新提供。

**为什么 URL 也算运行时值**：Desktop 的完成 Server 把进程 token 放在 URL 路径里，HTTP MCP 常见的鉴权
也可能出现在查询或路径中。只持久化名称，Node 就不必区分 URL 的哪部分是秘密。

## D4：只接受 HTTP，能力不足明确失败

- 宿主 MCP 只有 HTTP 形态，没有 stdio：Node 不执行 Controller 指定的任何本地命令（继承显式 Session MCP D5）。
- 非空 `host_mcp` 要求 Agent 在 ACP `initialize` 响应中声明 HTTP MCP 能力。缺少时在 `initialize` 之后、
  `session/new` 之前以 `agent_failed` 结束会话，detail 为 `host_mcp_http_capability_missing`，不附带
  URL、请求头或名称以外的信息；失败路径不向 Agent 发送任何宿主 MCP 值。
- 只有宿主 MCP、没有插件 MCP 的会话不要求 Agent 的 `session/load`；同时带非空插件 MCP 时按显式 Session
  MCP D4 的原规则要求。
- Node 不探测宿主 URL、不做健康检查，也不把它计入 Host MCP Health。URL 从 Node 主机是否可达由 Controller
  负责：同机部署用 loopback，远程部署用 Node 能访问的 https 地址。

**为什么失败而不是静默附加**：宿主能力缺失会改变工作流的完成方式（例如节点永远等不到完成信号）。Agent
不支持 HTTP MCP 时静默附加，工具就不存在，问题只能在节点卡住后才被发现；稳定 code 让 Controller 在启动时
得到可解释的失败。Desktop Backend 当前不做这项检查，本决策不改变 Desktop 的行为。

## D5：工具可见性不等于授权

OpenCode 把 ACP 送来的 MCP 添加到进程级注册表，后续 setup 的列表缩小也不会删除早先的注册项
（[工作流显式选择](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md)）。在 Node 上，
[共享 checkout 会话](20261115-shared-checkout-agent-sessions.md)只共享 checkout，每个会话执行仍有自己的插件
进程，所以宿主 Server 不会出现在兄弟会话的 Agent 里；但同一 execution 的 Agent 进程会在它的全部轮次和刷新
中一直看得见它，Desktop 共享进程中的会话也会看见别的会话注册的 Server。以后若 Node 让多个会话共用一个
Agent 进程，兄弟会话同样会看见。

因此宿主能力的授权不能依赖可见性。以完成工具为例：授权来自宿主为单个 attempt 签发、只写在该会话当轮
prompt 中的 attempt token，其他会话拿不到它，同一会话里早先轮次的 token 在那次 attempt 结束时已经作废，越权或过期的调用被宿主 Server 拒绝且
不改变状态。由此要求：

- 经 `host_mcp` 提供的 Server 必须自行对每次调用鉴权，不能把“工具出现在某个 Agent 进程里”当作“调用来自
  该会话的当前 attempt”。
- Node 不承诺宿主 Server 只对附加它的会话或当前 attempt 可见。

## 不变量

1. 未声明 `session_host_mcp` 的 Node 不会收到 `host_mcp`；省略或空列表时会话行为与本决策之前相同。
2. 宿主 MCP 不进入插件 MCP 的选择、Effective MCP Set、revision、上限、刷新或健康探测，也不改变它们的失败语义。
3. 同一 execution 的每一次 ACP setup 附加的宿主 Server 列表相同。
4. 宿主 Server 名不含 `/`，不会与任何插件 MCP 的 Server 名相同。
5. URL 与请求头只驻留内存和成功的 ACP setup 请求，不属于任何持久或诊断数据；持久输入只有 Server 名称。
6. Node 不执行宿主提供的命令，不连接、不探测宿主 URL。
7. 非空宿主 MCP 遇到不支持 HTTP MCP 的 Agent 时，会话以 `host_mcp_http_capability_missing` 失败，失败路径不向 Agent 发送宿主值。

## 为什么不是这些替代方案

| 方案 | 优势与未采用原因 |
| --- | --- |
| 把完成工具做成内置 `.orax` MCP 包，经 `mcp` 交付 | 不改 Node 协议，复用已落地的安装与刷新。但宿主能力会依赖租户插件的安装、市场和 MCP 生命周期，违背完成信号 D1；包内 HTTP URL 写死且必须 https，只能改用包内 stdio 中转程序，离线部署还要额外分发它 |
| Agent 在回答里输出约定的标记 | 不需要任何协议改动。但要解析和剥离文本，模型可能漏写或误写，与完成信号决策否决的方案相同，且不能在 turn 中途表达 |
| Node 自己在本地提供 MCP，把工具调用经控制协议转发给 Controller | 不要求 Node 主机能访问 Controller 的地址，凭据也不必跨控制协议到达 Agent。代价是新增双向的调用和结果消息、超时与重放语义，Node 还要理解宿主工具的请求和响应；对目前同机部署的使用者没有收益。若以后出现 Node 无法访问 Controller 的部署，可作为新的决策另行评审 |
| 允许宿主提供 stdio 命令 | 不需要 HTTP 能力。但等于让 Controller 在 Node 主机上执行任意命令，违反显式 Session MCP D5 |
| 宿主 MCP 并入 `mcp`，由 Node 区分来源 | 少一个字段。但会让插件白名单、上限、revision 和刷新协议都要区分两类成员，也让“空选择不注入插件 MCP”失去直观含义 |

## 风险、本决策未解决的问题与落地顺序

- **凭据到达 Agent**：宿主请求头会出现在 Agent 进程内，与插件 MCP 的 Setting 值相同；本决策不提供对恶意
  Agent 回显凭据的内容过滤。宿主应签发短期、只对自身 Server 有效的凭据。
- **可见性**：见 D5；宿主 Server 必须自行鉴权。
- **可达性**：Node 不验证 URL 可达，连接失败表现为 Agent 侧工具调用失败；宿主应从自身的调用结果判断，
  不期待 Node 上报。
- **未解决**：远程 Node 无法访问 Controller 地址时的转发方案（见替代方案第三行）、宿主能力的运行中更新，
  都留给后续决策。

落地顺序：

1. 评审本决策的字段、上限、loopback 明文规则、能力门禁与 D5 的授权前提；评审通过后改为 `approved` 并补核心测试用例。
2. Node 实现：`node-protocol` 增加 `host_mcp` 字段与 `session_host_mcp` 能力，Node 会话宿主实现
   `host_mcp_servers()`，在首次数据库写入前分离运行值与持久名称，接入 HTTP 能力门禁。以真实 ACP 帧证明
   `session/new`、`session/load`、刷新与 Agent 重启都附加同一份列表，并证明值不出现在持久、日志与诊断表面。
3. 能力广告与实现和测试同批交付。Controller 的使用者（工作流完成工具）在声明能力的 Node 上另行接入。

从声明 `session_host_mcp` 的 Node 开始，Controller 才能为会话附加宿主 MCP；Desktop Backend 的宿主能力
路径、插件 MCP 的全部行为、旧 spec 与重启中断规则保持不变。

## 核心测试用例

本决策处于 `proposed`，按规则暂不维护核心测试用例；评审通过、改为 `approved` 时，在
`test-cases/node/agent-runtime/` 补充宿主 MCP 用例，覆盖字段校验与能力协商、每次 ACP setup 附加同一列表、
与插件 MCP 的隔离、运行值的持久与诊断隔离，以及 HTTP 能力门禁。
