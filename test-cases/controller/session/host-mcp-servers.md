# Controller 交付宿主 MCP

依据[已批准的 Controller 交付宿主 MCP 决策](../../../decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md)
D1 与 D6。这些义务只能在 Controller 及其宿主 Server 上验证，Node 侧义务见
[Node 承载的宿主 MCP 用例](../../node/agent-runtime/host-mcp-servers.md)。Controller 的使用者（工作流完成
工具）尚未接入，下表证据一律为 `Missing`，随该使用者的交付登记直接测试。

## Controller Sends Only Valid Host Servers To Capable Nodes

### 风险

Controller 与 Node 对同一 URL 判断不同，把 Node 判为非法的输入发出去，关闭整条控制连接；向未声明能力的
旧 Node 发送字段让它拒绝整帧；需要宿主能力的节点被派到不具备能力的 Node 后静默降级；给 Agent 无法到达的
网络环境配了 loopback 地址。

### 前置状态

可派发的 Node 分别声明与不声明 `session_host_mcp_servers`；部署配置为 Node 指定宿主端点地址；Controller
使用 `node-protocol` 的共用接受与拒绝样例。

### 触发

为 `agent` 完成策略的交互节点派发会话；用样例中的每一项构造宿主列表；目标 Node 不具备能力。

### 必须成立

- 发送前的校验对共用样例逐项给出与标注相同的判断，非法输入不发送。
- 只向同时声明 `session_host_mcp_servers` 与 `agent_session` 的 Node 发送非空列表。
- 需要宿主能力而没有具备能力的 Node 时，在派发前拒绝或另选 Node，不降级为不需要宿主能力的行为。
- 每个 Node 使用的 URL 来自部署配置；loopback 明文只用于与 Agent 同网络命名空间的端点。

### 禁止结果

- 任何样例在 Controller 上的判断与标注不同；旧 Node 收到 `host_mcp_servers`。
- 需要宿主能力的节点在不具备能力的 Node 上运行。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 共用样例判断一致 | Controller 校验测试遍历共用样例文件，逐项断言接受或拒绝与标注相同 | Missing |
| 按能力派发 | Controller 派发测试：不声明能力的 Node 收不到字段；需要宿主能力而无可用 Node 时派发前拒绝 | Missing |
| URL 来自部署配置 | Controller 配置测试：每个 Node 的宿主端点取自部署配置，远程沙箱没有 Agent 可达的 https 端点时不提供宿主能力 | Missing |

### 决策依据

D1、D6“按能力派发”与“可达性由部署配置决定”，不变量 1、5。

## Host Endpoints Stay Valid For The Whole Execution

### 风险

Controller 照 Desktop 的写法每个进程随机绑定端口、随机生成 token，Controller 重启后 Node 上仍在运行的会话
拿着失效的 URL，节点永远等不到完成信号；Controller 靠重发 start 轮换凭据，但 Node 忽略重发的值。

### 前置状态

一个 Running 的 execution 带宿主 Server；Controller 持久保存或可推导该 execution 的凭据。

### 触发

重启 Controller 进程；之后 Agent 调用宿主工具；再模拟凭据无法恢复（例如推导密钥丢失）。

### 必须成立

- 重启后同一 URL 与请求头仍然被宿主 Server 接受，调用在当前 attempt 内照常生效。
- 凭据无法恢复时，Controller 结束该 execution。
- Controller 不通过重发 start 更换在用 execution 的端点或凭据。

### 禁止结果

- 重启后在用 execution 的宿主调用被拒绝而 execution 继续运行，只能靠 `force` 结束节点。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 跨 Controller 重启有效 | Controller 重启测试：同一 execution 在重启前后都能成功调用宿主工具 | Missing |
| 无法恢复时结束 execution | Controller 测试：凭据不可恢复时发出结束该 execution 的命令 | Missing |
| 不轮换 | Controller 测试：重发 start 时宿主列表与首次发送逐字节相同 | Missing |

### 决策依据

D3“重发 start 时值被忽略”、D6“端点与凭据在整个 execution 内有效”，不变量 3、9。

## Host Servers Authenticate Every Call Per Execution And Attempt

### 风险

宿主 Server 把“请求到达了端点”或“工具出现在某个 Agent 里”当作授权，同一 workload UID 下的兄弟会话或本机
其他进程伪造完成信号；会话结束或停止未确认后凭据仍然有效；过期或别的 attempt 的 token 被接受；开启
`autoAcceptAgentCompletion` 的节点因伪造信号直接完成。

### 前置状态

同一 Node 上两个 execution 各带自己的宿主凭据；其中一个节点开启 `autoAcceptAgentCompletion`。

### 触发

用另一个 execution 的凭据或 token 调用；用早先 attempt 的 token 调用；收到 `AgentSessionEnded` 或
`stop_unconfirmed` 后再调用。

### 必须成立

- 凭据只对签发它的 execution 有效；token 只在绑定的会话与 attempt 内短期有效。
- 收到该 execution 的 `AgentSessionEnded` 或 `stop_unconfirmed` 后凭据被吊销。
- 越权、过期或已吊销的调用被拒绝且不改变任何状态，包括开启自动接受的节点。

### 禁止结果

- 跨 execution 或跨 attempt 的调用产生完成信号；吊销后的调用被接受。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 按 execution 隔离凭据 | 宿主 Server 测试：另一个 execution 的凭据被拒绝且状态不变 | Missing |
| attempt 绑定与有效期 | 宿主 Server 测试：早先 attempt 或过期的 token 被拒绝且状态不变 | Missing |
| 终态与停止未确认时吊销 | Controller 测试：收到 `AgentSessionEnded` 或 `stop_unconfirmed` 后同一凭据被拒绝 | Missing |
| 自动接受不放大伪造 | 宿主 Server 测试：开启自动接受的节点收到越权调用时不完成 | Missing |

### 决策依据

D5、D6“按 execution 签发、及时吊销”与“attempt 级授权”，不变量 9；完成信号 D1、D2。
