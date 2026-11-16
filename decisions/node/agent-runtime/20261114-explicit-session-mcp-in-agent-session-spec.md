---
status: approved
---

# Node 从 AgentSessionSpec 的显式选择解析 Session MCP

> 历史决策：D2/D3 的“一次冻结 setup”对声明 `mcp_lifecycle: true` 的会话已由
> [会话 MCP 刷新与宿主健康](./20261116-session-mcp-refresh-and-host-health.md) D1–D4 修改：独立的内存更新可在安全点改变绑定；
> 未启用生命周期模式的会话及 start 重发不更新值的规则不变。

Controller 在 `start_agent_session` 的 `AgentSessionSpec.mcp` 中交付完整的 MCP 插件选择、精确版本和
Setting 值；Node 使用已安装包解析一次完整的 Session MCP Snapshot，通过 ACP `session/new` 交付。
省略或空列表均表示不交付插件 MCP；Node 不自动发现。Desktop 与 Node 共享纯解析规则和 ACP 映射。

本决策为 `approved`：owner 于 2026-11-15 经两轮评审后批准 D1–D5，核心测试用例见文末“核心测试
用例”。实现尚未合入 desktop 的 main，用例义务如实标为 `Missing`；实现合入并完成验证后，由落地运行
按最终实现同步正文并改为 `implemented`。影响 `node-protocol`、共享 agent-runtime、Node
会话宿主和 Desktop 的纯解析器归属；不改变 Desktop 的选择、配置存储、刷新或健康探测行为。不迁移
历史会话、不恢复 Node 进程重启前的 Agent；旧格式 spec 继续表示空选择。旧 Node 使用
`deny_unknown_fields`，因此 Controller 只向声明新能力的 Node 交付非空选择。

## 继承与修改

继续[子代理投影、历史与诊断](20261020-child-projection-history-and-diagnostics.md)，沿用其汇总的
共享运行时、history 和宿主边界，并修改 Node 原来没有 MCP 输入的约定。

| 前序约定 | 处理 |
| --- | --- |
| 宿主拥有进程、会话和授权生命周期 | 继承，不变；Node 持有包使用租约直到 Agent 停止 |
| JSONL 与 Thread 使用共享 history，保留 provider 来源和已观察事实 | 继承，不变；MCP setup 输入不成为 history 或 Thread 记录 |
| 父子投影隔离、Timeout / Wait、取消及清理 | 继承，不变 |
| 配置与连接快照不属于持久 transcript | 继承，不变；Setting 值也不得进入 Node ledger |
| Node SessionSetup 恒为空，没有 MCP 输入 | 修改：只接受 spec 的显式完整选择，由 Node 解析 |
| 执行输入完整持久化及重复输入比较 | 修改：MCP Settings 是运行时例外；持久输入只保留 MCP 身份和版本，不比较或恢复 Setting 值 |
| `AgentSessionEnded` 只携带终态原因和有界 detail | 修改：MCP setup 失败还可携带无值的 `plugin_id`、`setting_id` 与 `transport`，其余终态省略这些字段 |

本 ADR 在会话所有权边界描述协议字段变化，无需另建 protocol ADR。
[协议根决策](../protocol/0-controller-node-protocol.md)的身份、帧上限及能力协商继续有效；
[会话执行与命令](../protocol/20260928-streamed-thread-events-and-session-commands.md)的先持久受理、
终态、事件确认和重放继续有效，但“上传授权是唯一不持久化输入”新增本 ADR 的 MCP Setting 例外。

## 问题与约束

Cloud v2 工作流的 Agent 节点在 Ora Node 上执行，目前 `AgentSessionSpec` 没有 MCP 字段，Node
组合 `NoSessionMcp`，选择无法到达 Agent。Desktop 已经以 `ora-plugin-config` 编译包内声明，再映射到
ACP，但纯解析规则仍位于 Backend；让 Node 再写一份会使参数、HTTP header、路径和能力检查发生漂移。

包已通过 `InstallPlugins` 交付到 Node，Controller 不知道也不应传递 Node 的本地路径。Node 不持久化
凭据的要求，与原执行输入整体持久化存在明确冲突，必须在 SQLite 写入前分离运行时值和持久身份，
不能先写入再删除，也不能把 Setting 值的摘要当作 secret-free revision。

## D1：显式、完整、确定的选择

`mcp` 是列表，每项仅有 `plugin_id`、`plugin_version`、`settings`；`settings` 必填，但没有覆盖值时是空
对象，不能因 `omitempty` 省略。Settings 复用 Plugin Configuration 的 string、number、boolean 标量语义，
不接受对象、数组或 null。`plugin_id` 必须通过严格的 `PluginId` 解析，并按原始 ID 的 UTF-8 字节序
严格递增，拒绝重复、乱序或非规范 ID；`plugin_version` 必须满足
`semver::Version::parse(version).to_string() == version`，因此 `v1.2.0`、非完整版本、版本范围或
自动选最高版本都被拒绝。

Controller 必须在发送前镜像这些客户端规则：一次会话最多 **16** 个 Server；所有 `settings` map 的
紧凑 JSON UTF-8 编码长度之和最多 **64 KiB**，包括键、标点与转义；原始 ID 的字节序、严格 ID 解析、
精确 canonical 版本字符串以及 `settings` 必填都按上一段执行。两项上限独立于控制帧上限：限制进程
扇出和凭据驻留量；超限是 spec 校验错误，不启动 Agent。空列表不读取无关插件，也不退回 `Automatic`。

Controller 冻结完整选择；Node 对每项校验包和 Setting，任何一项失败都拒绝整个快照，绝不跳过插件。
Settings 使用声明默认值，显式覆盖必须已声明且类型正确；缺少必填值或被 transport 引用而不存在的值
失败，不猜测值。工作区 context 只由 Node 的权威绝对 cwd 替换。

## D2：Node 解析，Desktop 与 Node 共用映射

Node 从自身 plugin catalog 取得每项的精确版本并持有使用租约，编译包内 `assets/config.json`，使用
`ora-plugin-config` 解析 transport，再由共享纯解析器映射到 ACP。`node-protocol` 为此直接依赖
`ora-domain`、`ora-plugin-config` 与 `semver`，不引入 `ora-utils` 的 default/`validation`；
`SettingValue` 的 untagged string、number、boolean wire 格式仍由 `ora-plugin-config` 拥有，协议 crate
复用而不另立格式。Server 名称为 canonical Plugin ID，
输出按该 ID 排序。revision 仅含身份、版本、配置 revision 与 transport；冻结会话输入使用配置 revision
`0`，不散列 Setting 值，也不把它用于不同会话之间的值相等判断。

共享解析器继承 Desktop 的
[ACP Session setup 交付](../../desktop/core/agent/session/mcp/0-acp-session-setup-delivery.md)与
[工作流显式选择](../../desktop/core/agent/session/mcp/20260912-workflow-explicit-selection.md)，
但 Node 的配置来源为 spec，没有 Desktop 的配置 store。Desktop 的既有自动选择和刷新规则不变；
[Host 健康探测](../../desktop/core/agent/session/mcp/20260916-mcp-health-probing.md)仍为 Desktop 职责。

## D3：值只在运行时边界存在

Setting 值只通过 `start_agent_session` 控制帧进入 Node 内存，在成功解析后进入 ACP `session/new`。
不得进入 ledger（包括 SQLite WAL）、会话 JSONL、Thread、日志、错误链或输入对象的 `Debug`。
Node 的结构化 setup 失败携带稳定 code，并在适用时携带 Plugin ID、Setting ID 与 transport；preflight
返回和 capability 失败的日志同样只记录这些标识与 code，绝不记录值。`AgentSessionEnded.detail`
继续只含稳定 code；MCP setup 失败时，Controller 还收到可选的 `plugin_id`、`setting_id` 与
`transport`（`stdio` 或 `http`），无法确定的字段省略，其他终态也省略。它们是标识而非值，包编译或
I/O 错误不透传原始文本。
不把启动帧当作诊断记录，也不把 ACP 请求体写日志。

持久输入保留 MCP 身份、版本和空 Settings。身份、版本及其他原字段参与重复执行比较，Setting 值不参与：
重发相同 execution 不替换正在使用的内存值；需要改变值必须创建新 execution。受理后进程崩溃不从
持久投影启动 Agent，按原规则结算 `interrupted`，不要求 Controller 重放凭据来恢复。运行和持久输入
必须在首次数据库写入前分开，避免凭据残留在空闲页或备份中。历史空选择无需数据迁移。

## D4：分离 Node 与 Agent 能力门禁

Node 在 `HelloAccepted.capabilities` 广告 `session_mcp`；Controller 只向声明该能力且支持会话执行的
Node 发送非空列表。它表示 Node 能解析，不保证所选 Agent 支持 MCP。

非空选择必须要求 Agent 的 `session/load`；HTTP 另要求 ACP HTTP MCP 能力。能力来自 ACP
`initialize` 响应，不能在任何 ACP 帧之前断言。包、Setting 与 spec 错误在 Agent 启动前检查；能力错误
在 `initialize` 后、`session/new` 前检查。任何解析失败都不发送 `session/new`，因此失败路径不向
Agent 发送任何 MCP Setting 值。不启动健康探测来代替 Agent 能力协商。

| 失败 | 对外 code / 行为 |
| --- | --- |
| spec 重复、乱序、非 canonical ID / version、超限 | 帧校验失败，Node 以协议错误关闭控制连接（继承 [close-code ADR](../protocol/20260925-controller-heartbeat-and-websocket-close-codes.md)），不受理该执行，不启动 Agent |
| 未安装精确版本、包无效、不是 MCP、stdio 命令越界或丢失 | `mcp_plugin_missing` |
| 必填或 transport 引用值缺失，覆盖不合法，值含非法运行时控制字符 | `mcp_setting_missing` |
| Agent 未声明 HTTP MCP | `mcp_http_capability_missing` |
| 非空选择且 Agent 未声明 `session/load` | `mcp_load_session_missing` |

运行时失败均以 `agent_failed` 和上述稳定 detail 结束会话，不暴露值；能力失败遵循原进程清理和终态顺序。

## D5：沿用安装与文件系统隔离

复用 `InstallPlugins` 的精确版本安装路径，不新增 MCP 下载器或凭据文件。现有 plan 安装入口没有
仅限 Agent 的 kind 过滤，`kind = "mcp"` 使用现有 MCP 包验证。stdio command 必须是该版本包
`assets/` 内的普通文件，发送前重新验证 containment，不能信任 Controller 提供的主机命令。

独立 workload 身份不能读取 Node 私有 home，因此与 Agent 包相同，将不可写的 MCP 包文件链接到
本次会话拥有的 package view 后交付该 view 内的 command。view 保留同一版本字节及 assets containment；
不修改 Node home 权限、不把 Setting 写入 view。Agent 停止后释放 view 与包租约。会话子目录使用独立
名称，复用既有会话目录的清理，不能碰其他会话、工作区或用户文件；没有全局目录迁移义务。

## 不变量

1. 省略和显式空选择均只得到空插件 MCP 集合，Node 从不自动发现。
2. 输入身份有序唯一且规范，输出 Server 顺序确定；所有成员成功才交付完整快照。
3. Setting 值只驻留内存和成功的 ACP setup 请求，不属于任何持久或诊断数据。
4. revision 与失败 code 不含 Setting 值或它们的摘要。
5. 能力失败发生在 initialize 之后、session/new 之前；失败路径没有 MCP 值到达 Agent。
6. stdio 命令属于被选版本的 assets，租约和只读 view 活到进程停止之后。
7. 重发不改变正在运行的值，重启不从持久投影恢复 Agent。

## 为什么不是这些替代方案

| 方案 | 优势与未采用原因 |
| --- | --- |
| Controller 在 Go 中解析后发送 ACP Server | 更早检查声明，但需要复制 Rust 编译器、路径和 ACP 能力规则；Controller 无法证明 Node 的已安装字节和本地 command containment |
| 类似 model proxy 的 MCP proxy，凭据不跨控制协议 | 减少控制面凭据暴露，但 stdio 不是统一远程 HTTP；需要新的代理授权、网络与生命周期契约，也改变 Agent 直连语义。首期接受受保护控制帧中的短期内存交付 |
| Node Automatic 发现 | 配置更少，但本地安装变化会扩大工作流工具权限，不能表达冻结选择或明确的空集合 |
| 把完整 spec 持久化后擦除 Settings | 保留原去重方式，但 WAL、空闲页和备份仍可能保存凭据，违反不变量 3 |

## 风险、本决策未解决的问题与落地顺序

凭据将到达 Agent 和所选 MCP，平台不能以“控制面无凭据”描述此路径。需要保证帧诊断、错误及输入
Debug 不输出值；本 ADR 不定义对恶意 Agent 在普通输出中主动回显任意凭据的通用内容过滤。
本决策不增加 live refresh、会话 restore、Node Host health probes、取消回合（#1110）或 Cloud 实现；
前三项由 [circus #1171](https://github.com/ora-space/circus/issues/1171) 承接。这些涉及新的凭据续期和
生命周期，不由一次冻结 setup 隐式承诺。

1. 协议字段、边界、上限与持久投影已评审通过（2026-11-15），核心用例已依据最终决策补充，不以当前代码替代决策审批。
2. 提取纯解析器，以 Desktop 原行为测试证明没有选择、映射和错误漂移；宿主 store 与 health 保持原归属。
3. Node 接入冻结输入、包租约、workload view 和无值持久投影，以真实 session/new 帧及失败路径验证。
4. `session_mcp` 能力广告与上述实现和测试同批交付；验证协议兼容、数据库/JSONL/日志无值与清理隔离。Controller 的选择和派发另行交付。

从声明能力并接入的 Node 开始，显式 MCP 才能进入 Agent Session；Desktop 行为、Node 事件重放、
旧 spec 的空选择和重启中断规则保持不变。

## 核心测试用例

见[显式 Session MCP 用例](../../../test-cases/node/agent-runtime/session-mcp.md)：选择校验与上限、Node
全量解析与 workload view、Setting 值的持久与诊断隔离、能力门禁与失败 code、包租约与 view 生命周期，
以及 Desktop 解析器行为不变。实现尚未合入 desktop 的 main，义务如实标为 `Missing`；
[会话执行用例](../../../test-cases/node/agent-runtime/session-execution.md)的证据保持原范围，不推导本文新义务。
