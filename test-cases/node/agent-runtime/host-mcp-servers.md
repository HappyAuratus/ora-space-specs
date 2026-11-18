# Node 承载的宿主 MCP

依据[已批准的 Controller 交付宿主 MCP 决策](../../../decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)。
实现尚未开始，下表义务的证据状态一律如实标为 `Missing`；“所需直接证据”描述需要的直接测试（路径相对
desktop 仓库根目录），实现合入后按最终测试逐项回填路径与状态，不以目标变更中存在测试替代合入后的核对。
只能由 Controller 验证的义务（镜像校验、端点与凭据有效期、鉴权与吊销）在
[Controller 侧用例](../../controller/session/host-mcp-servers.md)维护，本文不登记。
[显式 Session MCP 用例](session-mcp.md)与[会话 MCP 刷新与宿主健康用例](session-mcp-lifecycle.md)的证据保持
各自范围，不推导本文义务。

## Host MCP Servers Must Be Canonical Bounded And Shared With The Controller

### 风险

两端 URL 解析器对 `127.1`、整数或十六进制 IPv4、IPv4 映射的 IPv6 给出不同判断，Controller 认为合法的
输入让 Node 以协议错误关闭整条控制连接；`localhost` 或其他主机上的明文 http 被接受；上限按开区间或按
会话/按 Server 混算；保留请求头被 Controller 覆盖；宿主名与插件 Server 名相同；新能力打破旧 Controller
或旧 Node。

### 前置状态

Node 以 `deny_unknown_fields` 解码 `AgentSessionSpec`；`node-protocol` 提供 D1 的接受与拒绝样例文件。

### 触发

用样例中的每一项分别发送 `host_mcp_servers`：省略、空列表、恰好与超过每项上限的输入；乱序、重复、
不以 `ora-` 开头或只有 `ora-` 的名称；不符合 D1 文法的 URL：`127.1`、`2130706433`、`0x7f.0.0.1`、大写
scheme 或主机、空路径（`http://127.0.0.1:41234`）、默认端口（`http://127.0.0.1:80/mcp`、
`https://example.com:443/x`）、前导 0 与越界端口（`:041234`、`:0443`、`:65536`、`:0`）、点段（`/a/./b`、
`/a/../b`、`%2E`）、小写十六进制的百分号转义、`?` 查询与 `#` fragment、IPv6 的其他写法与十六进制大小写
（`[::ffff:127.0.0.1]`、`[::FFFF:7f00:1]`、`[0:0:0:0:0:0:0:1]`）、非 ASCII 与 `xn--` 的 IDNA 主机、以数字
开头的末段、反斜杠、制表符与换行、空格与 `€` 等未编码的字符、userinfo；`http://localhost`、
`http://127.0.0.2`；保留请求头与逐跳头、空值或首尾带空格的值、含 CR/LF 的值、重复的 JSON 键、未知字段、
缺失 `headers`。

### 必须成立

- 省略与空列表等价且空列表不被序列化；旧格式 spec 照常解析。
- 只接受完整匹配 D1 文法的 URL；每个接受样例经 WHATWG 解析器解析再序列化后与原样逐字节相等；明文
  `http` 只接受逐字的 `127.0.0.1` 与 `[::1]`。
- 名称、请求头和全部上限按 D1 计数：每个 Server 8 个请求头，请求头名 1–64 字节，4 个 Server，URL 2 KiB，
  URL 与请求头名、值合计 8 KiB，全部是闭区间。
- 非法输入是帧校验失败：Node 以协议错误关闭控制连接，不受理、不启动 Agent。
- Node 在 `HelloAccepted` 广告 `session_host_mcp_servers`；样例文件中每一项在 Node 上的判断与样例标注一致。

### 禁止结果

- Node 端补排序、去重、规范化 URL 或补默认值后受理；任何样例在 Node 上的判断与标注不同。
- 非法输入被折叠为 `agent_failed` 会话终态而不是关闭控制连接。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 省略与空列表等价、旧 spec 兼容 | `crates/node-protocol/tests/protocol/` 下宿主 MCP 往返测试：旧输入等于空列表、空列表不序列化、分帧往返保留请求头 | Missing |
| 共用样例在 Node 上逐项判断一致 | 同目录测试遍历共用样例文件，在编码与接收边界逐项断言接受或拒绝与标注相同；样例覆盖上面“触发”的每一类与每个上限的恰好/超过一项 | Missing |
| 接受样例是 WHATWG 不动点 | 同目录测试对每个接受样例用 WHATWG 解析器解析再序列化，断言与原样逐字节相等 | Missing |
| 非法输入关闭控制连接 | `apps/ora-node/tests/agent_session/` 下的测试：发送非法宿主列表，断言控制连接以协议错误关闭、没有受理记录、没有 Agent 进程 | Missing |
| 能力广告 | Node 握手测试断言 `HelloAccepted.capabilities` 含 `session_host_mcp_servers` | Missing |

### 决策依据

D1、不变量 1、4、5；显式 Session MCP D4 的帧校验失败。

## Every ACP Setup Of An Execution Carries The Same Host Servers

### 风险

宿主 Server 只在首次 `session/new` 附加，`session/load`、生命周期刷新或 Agent 重启后 Agent 失去完成工具；
插件刷新重建或改写宿主部分；宿主 Server 混进插件选择、revision、上限、`GetSessionMcpState` 或 setup 窗口的
选择；重发 start 改写在用的值。

### 前置状态

会话带非空 `host_mcp_servers`，分别为无插件 MCP、带插件 MCP、`mcp_lifecycle` 三种；Agent 支持 HTTP MCP 与
`session/load`。

### 触发

启动会话；Node 以 `session/load` 重新连接；`UpdateSessionMcp` 触发刷新 load；插件版本变化或第九次 load
触发刷新 D6 的 Agent 重启；以不同的 URL 与请求头重发同一 execution 的 start。

### 必须成立

- 每一次 ACP setup 请求的 `mcpServers` 都包含同一份宿主 Server（名称、URL、请求头和顺序不变），位于插件
  MCP 之后。
- 刷新只改变插件部分；`GetSessionMcpState`、revision 与 16 项上限不包含宿主 Server；空 `mcp` 仍不注入插件 MCP。
- 只有宿主 MCP 的会话使用普通 setup 窗口，宿主 Server 不选择 MCP 窗口。
- 重发 start 时携带的值被忽略，后续 setup 继续使用首次受理的值。

### 禁止结果

- 任一 setup 漏带、增删或改写宿主 Server；宿主 Server 出现在插件 MCP 状态或计入上限。
- 重发 start 替换在用的 URL 或请求头。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 四类 setup 附加同一份列表 | `apps/ora-node/tests/agent_session/` 下以真实 ACP 帧记录 `session/new`、`session/load`、刷新 load 与重启后 `session/new`/`session/load` 的 `mcpServers`，逐次断言宿主部分完全相同且在插件之后 | Missing |
| 与插件 MCP 隔离 | 同目录测试：带宿主 Server 的会话查询 `GetSessionMcpState`、刷新与 16 项上限均不包含宿主 Server；空 `mcp` 只发宿主 Server | Missing |
| 不放宽 setup 窗口 | 运行时层的测试（`crates/agent-runtime` 的会话 setup 测试或 Node 集成测试）：只有宿主 Server 的会话，假 Agent 在 setup 上一直不回答，在普通窗口到期时以 `agent_timed_out` 失败，而不是等到 MCP 窗口；只看插件快照的窗口选择函数的单测不能作为证据 | Missing |
| 重发 start 不替换值 | 同目录测试：以不同 URL/请求头重发同一 execution，后续 load 仍发送首次受理的值 | Missing |

### 决策依据

D2、D3，不变量 2、3。

## Host Server Values Never Persist Or Reach Any Diagnostic

### 风险

URL 或请求头进入 ledger、SQLite WAL、会话 JSONL、Thread、日志、错误链、关闭原因或 `Debug`；帧解码失败时
JSON 解码器的错误文本回显字符串值；持久行复用线上类型，没有 URL 的行解码失败；Node 崩溃后从持久投影启动
Agent 或要求 Controller 重放凭据。

### 前置状态

会话带含唯一标记字符串的 URL 与请求头；另有一个在 `host_mcp_servers` 中带标记值但格式非法的帧。

### 触发

受理并运行会话、刷新、正常结束与以各类失败结束；发送非法帧；受理后杀死 Node 进程再启动。

### 必须成立

- 标记字符串不出现在数据库文件（含 WAL）、会话 JSONL、Thread、日志、终态、关闭原因与输入的 `Debug` 输出中。
- 解码与校验失败只记录错误类别和字段路径，不包含解码器原始错误文本。
- 持久投影只有 `host_mcp_server_names`；空列表不序列化，旧行照常解码。
- 重启后该 execution 按原规则结算 `interrupted`，不启动 Agent、不要求重放。

### 禁止结果

- 任何持久或诊断表面出现标记字符串；持久行因缺少 URL 或请求头而无法解码。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 运行值不落盘、不进诊断 | `apps/ora-node/tests/agent_session/` 下的测试扫描数据库文件、WAL、JSONL、Thread、日志与终态，断言不含标记 | Missing |
| 解码失败不回显值 | 同目录测试发送带标记的非法帧，断言日志与关闭原因不含标记 | Missing |
| 持久形状独立且兼容 | Node 数据库测试：持久投影只含名称、空列表不序列化、旧行可解码 | Missing |
| 崩溃后不重放 | 同目录重启测试：受理后崩溃，重启结算 `interrupted`，没有 Agent 进程 | Missing |

### 决策依据

D3，不变量 6。

## Host Servers Require Agent HTTP MCP And Fail Before Session New

### 风险

Agent 不支持 HTTP MCP 时仍发 `session/new` 让工具静默缺失；失败终态或日志带出宿主值；宿主门禁抢在插件
门禁前改写插件 MCP 的失败；Agent 重启到不支持 HTTP 的版本后在没有宿主工具的情况下继续；只有宿主 MCP 的
会话被错误地要求或豁免 `session/load`；Node 探测宿主 URL。

### 前置状态

假 Agent 可配置是否声明 HTTP MCP 与 `session/load`；会话分别带只有宿主 Server、宿主加需要 HTTP 的插件
MCP、`mcp_lifecycle` 三种。

### 触发

在不声明 HTTP MCP 的 Agent 上启动；刷新 D6 重启到不声明 HTTP MCP 的 Agent 版本；只有宿主 Server 的会话
在不支持 `session/load` 的 Agent 上启动或刷新。

### 必须成立

- 只有宿主列表需要 HTTP 时，`initialize` 之后、`session/new` 之前以 `agent_failed` 结束，`detail` 为
  `host_mcp_http_capability_missing`，终态不含宿主 Server 名、URL 或请求头，Agent 没有收到宿主值。
- 同时带需要 HTTP 的插件 MCP 时以原有的 `mcp_http_capability_missing` 与插件身份字段失败。
- 重启后的新进程同样经过检查，缺少能力时该 execution 以同一 code 结束。
- 宿主门禁本身不要求 `session/load`；`mcp_lifecycle` 会话照现有规则要求。
- Node 从不连接宿主 URL。

### 禁止结果

- 能力缺失仍发送 `session/new` 或 `session/load`；新 code 进入 `SessionMcpErrorCode` 或终态多出字段。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能力缺失在 `session/new` 前失败且不带值 | `apps/ora-node/tests/agent_session/` 下以假 Agent 断言没有 `session/new`、终态 `detail` 与字段、Agent 收到的请求不含宿主值 | Missing |
| 插件门禁在先 | 同目录测试：宿主加 HTTP 插件 MCP 时终态为 `mcp_http_capability_missing` 且带插件身份 | Missing |
| 重启重新检查 | 同目录刷新测试：重启到不声明 HTTP MCP 的版本时 execution 以 `host_mcp_http_capability_missing` 结束 | Missing |
| load 要求与豁免边界 | 同目录测试：只有宿主 Server 的非 lifecycle 会话在不支持 load 的 Agent 上启动成功；`mcp_lifecycle` 会话照现有规则失败 | Missing |
| 不探测宿主 URL | 同目录测试：宿主 URL 指向记录连接的监听器，Node 自身从不连接 | Missing |

### 决策依据

D4，不变量 7、8；显式 Session MCP D4。
