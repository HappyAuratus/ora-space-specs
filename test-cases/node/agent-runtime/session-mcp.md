# Node 显式 Session MCP

依据[已批准的显式 Session MCP 决策](../../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)。
实现尚未合入 desktop 的 main，下表义务的证据状态一律如实标为 `Missing`；“所需直接证据”按 desktop
未合入变更中的目标测试登记（路径相对 desktop 仓库根目录），ADR 转为 `implemented` 时按合入后的最终
测试逐项核对并回填状态，不以目标变更存在测试替代合入后的核对。
[会话执行用例](session-execution.md)与[子代理投影用例](child-projection-and-diagnostics.md)的证据保持各自
范围，不推导本文新义务。

## The Spec Selection Must Be Explicit Complete And Bounded

### 风险

省略与空选择被区别对待，Node 把缺失选择解释为自动发现或 `Automatic`；乱序、重复或非规范的 ID 与
版本被受理；超过上限的进程扇出或凭据驻留量被接受；非法选择被折叠成会话终态而不是帧校验失败。

### 前置状态

Controller 构造 `start_agent_session` 的 `AgentSessionSpec.mcp`；Node 以 `deny_unknown_fields` 解码
spec，旧 Node 未声明 `session_mcp` 能力。

### 触发

发送省略 `mcp`、空列表、乱序或重复的 `plugin_id`、非严格解析的 ID、`v1.2.0` 或非完整 semver、缺失
`settings`、17 个 Server、settings 紧凑 JSON 超过 64 KiB（键、标点与转义都计入）的选择；重发旧格式
spec。

### 必须成立

- 省略与显式空列表都表示空插件 MCP 集合，Node 从不自动发现、不退回 `Automatic`；空选择不读取无关
  插件。旧格式 spec 继续解析为空选择，空选择不被序列化，旧 `deny_unknown_fields` Node 仍接受。
- `plugin_id` 通过严格 `PluginId` 解析并按原始 ID 的 UTF-8 字节序严格递增、无重复；
  `plugin_version` 满足 canonical semver 往返；`settings` 必填，没有覆盖值时是空对象。
- 一次会话最多 16 个 Server；所有 `settings` map 的紧凑 JSON UTF-8 编码长度之和最多 64 KiB；两项
  上限独立于控制帧上限，超限是 spec 校验错误。
- 非法或超限选择是帧校验失败：Node 以协议错误关闭控制连接（继承 close-code ADR），不受理该执行、
  不启动 Agent。

### 禁止结果

- Node 端补排序、去重或改写非规范 ID；超限选择被受理；缺失 `settings` 被默认成空对象；空选择被
  当成错误或全部插件；省略被当成非空。
- 帧校验失败被折叠为 `agent_failed` 会话终态而不是关闭控制连接。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空与省略等价、旧 spec 兼容 | `crates/node-protocol/tests/protocol/session_mcp.rs` 的 `session_mcp_round_trip_and_old_spec`：旧输入等于空选择、空选择不序列化、分帧往返保留显式 settings | Missing |
| 有序唯一规范、版本 canonical、上限 | 同文件 `session_mcp_rejects_ambiguous_or_unbounded_selection`：重复、乱序、非严格 ID、`v1.2.0`、17 个 Server、值/键/转义分别越过 64 KiB 的选择在编码与接收边界都被拒绝；恰好 16 个 Server 与恰好 64 KiB 被接受 | Missing |
| Controller 发送前镜像客户端规则 | Controller 侧以同样规则校验选择的测试；Controller 的选择和派发由落地顺序第 4 步另行交付，随该交付登记 | Missing |

### 决策依据

D1、D4 失败表第一行，不变量 1、2；close-code ADR 的协议错误关闭。

## The Whole Snapshot Resolves From The Installed Package Or The Setup Fails

### 风险

Node 按非 spec 版本解析、跳过失败成员交付部分快照、把声明默认值或覆盖校验放宽；Controller 提供的
主机命令或 Node 本地路径进入 ACP setup；revision 混入 Setting 值或其摘要。

### 前置状态

Node plugin catalog 已按 `InstallPlugins` 精确版本安装所选 MCP 包，包内含 `assets/config.json`；
spec 携带完整冻结选择。

### 触发

启动带完整选择的会话；其中一项未安装精确版本、包无效、不是 MCP、stdio 命令越界或丢失；一项必填或
transport 引用的值缺失、覆盖未声明或类型错误、对无声明 settings 的包发送覆盖。

### 必须成立

- Node 只从自身 plugin catalog 取得每项的精确版本并持有使用租约；编译包内 `assets/config.json`，
  由共享纯解析器映射到 ACP；Node 的配置来源为 spec，没有 Desktop 的配置 store。
- 任何一项成员失败都拒绝整个快照，绝不跳过插件、绝不交付部分 MCP 集合；包、Setting 与 spec 错误
  在 Agent 启动前检查。
- Settings 使用声明默认值；显式覆盖必须已声明且类型正确；被 transport 引用而不存在的值（含可选但
  无默认且被省略）失败；不猜测值。
- 协议与 ACP setup 不携带 Node 本地路径或 Controller 提供的主机命令；stdio 命令必须是该版本包
  `assets/` 内的普通文件，发送前重新验证 containment。
- Server 名称为 canonical Plugin ID，输出按该 ID 排序；revision 只含身份、版本、配置 revision（冻结
  会话输入为 `0`）与 transport，不散列 Setting 值，也不用于不同会话之间的值相等判断。

### 禁止结果

- 部分快照被交付或失败成员被静默跳过；声明默认值未应用；未声明的覆盖被接受或被静默丢弃；
  revision 含值或值摘要；Controller 主机命令或 Node home 路径进入 ACP setup。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 全量解析或整体拒绝 | `apps/ora-node/src/session/mcp/tests.rs` 的 `rejects_missing_capabilities_and_missing_or_invalid_packages_and_settings`、`maps_optional_referenced_setting_missing_to_full_failure`（可选声明被 transport 引用而未提供时按 `mcp_setting_missing` 整体拒绝，失败携带 `setting_id` 与 transport）、`rejects_an_override_for_a_settings_less_package` | Missing |
| 精确版本与无值 revision | 同文件 `resolves_exact_packages_and_secret_free_revision`：按精确安装版本解析、租约计数、revision 不含值 | Missing |
| 真实 `session/new` 交付 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings`：fixture 校验完整 `session/new` 帧的成员、顺序与默认值应用 | Missing |
| MCP 包经既有安装路径 | `apps/ora-node/tests/plugins/mcp.rs` 的 `installs_mcp_packages_through_the_existing_plan_path`：`kind = "mcp"` 走既有 plan 安装入口与 MCP 包验证 | Missing |

### 决策依据

D2、D5，不变量 2、4；继承 Desktop 的 ACP Session setup 交付与工作流显式选择。

## Setting Values Must Never Persist Or Reach Any Diagnostic

### 风险

Setting 值进入 ledger（含 SQLite WAL、空闲页与备份）、会话 JSONL、Thread 事件、日志、错误链或输入
对象的 `Debug`；重发改写正在使用的内存值；崩溃重启后 Agent 带旧值复活或要求 Controller 重放凭据。

### 前置状态

spec 的 `settings` 含敏感值；Node 使用真实 SQLite ledger；会话运行并产生记录与日志。

### 触发

受理并启动带值会话；重发同一 execution 携带不同值；受理后强杀 Node 再重开数据库；以 TRACE 级日志
运行；制造包、Setting 与能力失败。

### 必须成立

- Setting 值只通过 `start_agent_session` 控制帧进入 Node 内存，成功解析后进入 ACP `session/new`；
  不得进入 ledger（包括 SQLite WAL）、会话 JSONL、Thread、日志、错误链或 `Debug`；ACP 帧诊断对
  `session/new`/`session/load` 的 `mcpServers` 脱敏，不把 ACP 请求体写日志。
- 运行时值与持久输入在首次数据库写入前分开；持久投影只保留 MCP 身份、版本与空 Settings；身份、
  版本及其他原字段参与重复执行比较，Setting 值不参与；重发相同 execution（含不同值）返回既有记录，
  不替换正在使用的内存值、不重启；需要改变值必须创建新 execution。
- 受理后进程崩溃按原规则结算 `interrupted`，不从持久投影启动 Agent，不要求 Controller 重放凭据来
  恢复；历史空选择无需数据迁移。

### 禁止结果

- 任何值、值摘要或可逆编码出现在持久或诊断数据；重发改变运行中会话的值；重启后 Agent 复活或凭据
  被重放；先写入再删除式的“擦除”路径。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 账本无值 | `apps/ora-node/tests/session_ledger.rs` 的 `session_mcp_settings_never_enter_the_durable_ledger`：持久记录等于清空 settings 的投影；重发不同值返回同一记录，身份或版本冲突仍被拒绝；数据库目录全部文件字节扫描无值；重开后恢复投影无值 | Missing |
| 运行时无泄漏 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings`：Shared 与 Separate 两种工作负载下，Node home 全文件递归字节扫描、待发 Thread 事件与记录的 TRACE 日志均无值 | Missing |
| 失败路径无泄漏 | 同文件 `mcp_capability_failures_stop_before_session_new`、`mcp_preflight_failures_start_no_agent`：终态 JSON、日志与文件扫描无值 | Missing |
| `Debug` 与帧诊断脱敏 | `crates/node-protocol/tests/protocol/session_mcp.rs` 的 `session_mcp_debug_redacts_settings`；`crates/acp/src/frame.rs` 的 `mcp_setup_trace_redacts_servers_without_mutating_the_frame`（`session/new` 与 `session/load` 都脱敏且不改原帧） | Missing |
| 崩溃不恢复 Agent | 带非空 MCP 输入的会话强杀后以 `interrupted` 收口、不重启 Agent 的直接断言；无 MCP 输入时的既有义务见[会话执行用例](session-execution.md#durable-session-ledger-preserves-events-commands-and-terminal-state)，不推导本文新义务 | Missing |

### 决策依据

D3，不变量 3、7；协议后续决策“上传授权是唯一不持久化输入”的 MCP 例外。

## Capability Failures Stop Before Session New With Secret-Free Identifiers

### 风险

`session_mcp` 能力未声明仍收到非空选择；Agent 能力（`session/load`、HTTP MCP）在任何 ACP 帧之前被
断言，或缺失时仍发送 `session/new` 使值到达 Agent；失败终态或日志携带 Setting 值或原始错误文本。

### 前置状态

Node 在 `HelloAccepted.capabilities` 广告 `session_mcp`；echo agent 的 ACP `initialize` 响应可控地
声明或省略 `session/load` 与 HTTP MCP 能力。

### 触发

向未声明 `session_mcp` 的 Node 发送非空列表；非空选择且 Agent 未声明 `session/load`；HTTP 插件遇到
未声明 HTTP MCP 能力的 Agent；包、Setting 或 spec 校验失败。

### 必须成立

- `session_mcp` 只与 `AgentSession` 一起广告；它表示 Node 能解析，不保证所选 Agent 支持 MCP；
  Controller 只向声明该能力的 Node 发送非空列表。
- 包、Setting 与 spec 错误在 Agent 启动前检查（无 ACP `initialize`、无插件进程）；能力错误在
  `initialize` 之后、`session/new` 之前检查；任何解析失败都不发送 `session/new`，失败路径没有 MCP
  值到达 Agent；不以健康探测代替 Agent 能力协商。
- 四个失败 code 及语义：未安装精确版本、包无效、不是 MCP、stdio 命令越界或丢失为
  `mcp_plugin_missing`；必填或 transport 引用值缺失、覆盖不合法、值含非法运行时控制字符为
  `mcp_setting_missing`；Agent 未声明 HTTP MCP 为 `mcp_http_capability_missing`；非空选择且 Agent
  未声明 `session/load` 为 `mcp_load_session_missing`。运行时失败以 `agent_failed` 加稳定 detail 结束
  会话，遵循原进程清理和终态顺序。
- MCP setup 失败时 `AgentSessionEnded` 另携带可选的无值 `plugin_id`、`setting_id` 与 `transport`
  （`stdio` 或 `http`），无法确定的字段与其他终态一律省略；`detail` 继续只含稳定 code；包编译或
  I/O 错误不透传原始文本；preflight 与能力失败的日志同样只记录 code 与这些标识。

### 禁止结果

- 能力失败路径出现 `session/new` 或任何值到达 Agent；能力在任何 ACP 帧之前断言；终态、日志或错误
  链含值；非 MCP 终态携带 `plugin_id`、`setting_id` 或 `transport`；以健康探测替代能力协商。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 能力广告与 wire 拼写 | `crates/node-protocol/tests/protocol/session.rs` 的 `accepts_independent_and_combined_execution_capabilities` 固定 `session_mcp` 拼写；`apps/ora-node/tests/standalone/agent_sessions/support.rs` 的生产握手断言真实 Node 进程广告 `SessionMcp`（与 `AgentSession` 同批） | Missing |
| 能力失败停在 `session/new` 前 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_capability_failures_stop_before_session_new`：ACP 方法列表只有 `initialize`，终态为 `agent_failed` + `mcp_load_session_missing` / `mcp_http_capability_missing`，进程清理与租约归零 | Missing |
| preflight 失败不启动 Agent | 同文件 `mcp_preflight_failures_start_no_agent`：`mcp_plugin_missing` / `mcp_setting_missing` 无 ACP 帧、无插件进程 | Missing |
| 失败标识无值 | 上述两测试断言终态携带 `plugin_id` / `setting_id` / `transport` 且其 JSON、日志与文件扫描无值；spec 级失败按帧校验拒绝（`session_mcp_rejects_ambiguous_or_unbounded_selection`） | Missing |

### 决策依据

D4，不变量 5；协议根决策的能力协商与 close-code ADR。

## The Package Lease And The Read-Only View Live Until The Agent Stops

### 风险

会话运行期间 MCP 包被移除，Agent 之后从已删除路径重新 spawn server；Separate 工作负载收到 Node
home 内的命令无法执行；Setting 被写入 view；Agent 停止后租约或 view 残留，或清理越过本次会话。

### 前置状态

选择含 stdio MCP 包；Separate 工作负载身份不能读取 Node 私有 home；会话正在运行。

### 触发

会话运行期间移除插件；会话正常结束或 Node 停止；以 Separate 工作负载启动带 stdio 插件的会话。

### 必须成立

- Node 对每个所选 MCP 包持有使用租约直到 Agent 停止；会话期间活跃插件移除被租约拒绝。
- stdio command 属于被选版本包 `assets/` 内的普通文件；不可写的 MCP 包文件以 hard-link 进入本次
  会话拥有的 package view，交付的是 view 内的 command；view 保留同一版本字节与 assets containment。
- 不修改 Node home 权限、不把 Setting 写入 view；Agent 停止后释放 view 与包租约；会话子目录使用
  独立名称，复用既有会话目录清理，不碰其他会话、工作区或用户文件。

### 禁止结果

- 会话期间包被移除或命令路径失效；Agent 收到 Node home 路径或 Controller 提供的命令；view 可写或
  含 Setting 值；结束后租约或 view 残留；清理越界到其他会话或全局目录。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 租约活到 Agent 停止 | `apps/ora-node/tests/agent_session/mcp.rs` 的 `mcp_reaches_real_acp_session_new_without_persisting_or_logging_settings`：会话期间租约计数包含所选 MCP 包，结束并关闭会话后归零；运行期间 `RemovePlugins` 对被选 MCP 包返回 `plugin_in_use` 的直接断言尚缺 | Missing |
| view 交付与清理 | 同测试 Separate 分支：交付的 command 位于 workload 目录内而不在 Node home 下，会话结束后 workload 目录清空 | Missing |

### 决策依据

D5，不变量 6；`InstallPlugins` 的精确版本安装与包验证。

## Desktop Resolution Behaviour Stays Unchanged Behind The Shared Resolver

### 风险

解析器从 Backend 移到共享 crate 时改变 Desktop 的选择、映射、顺序或错误分类；Desktop 引入 Node 的
spec 来源，或 Node 引入 Desktop 的配置 store、自动选择、刷新或健康探测职责。

### 前置状态

Desktop 原有 MCP 选择、映射与错误分类测试在移动前的基线通过；纯解析器已移至共享 crate，由 Desktop
与 Node 共用。

### 触发

在移动后的代码上运行 Desktop 既有 `mcp` 测试套件与新增用例；执行 stdio/HTTP 映射、能力缺失与
revision 生成。

### 必须成立

- 共享解析器保持 Desktop 原行为：canonical Plugin ID 排序、精确 revision、stdio 与 HTTP 映射、错误
  分类不变；Desktop 既有 `mcp` 测试在移动后原样通过，只允许新增用例。
- Desktop 的自动选择、配置 store、刷新与 Host 健康探测规则及归属不变；Node 侧按 spec 解析，不使用
  Desktop 的配置来源。

### 禁止结果

- 移动引入选择、映射、顺序或错误分类漂移；Desktop 测试被改写以适配新行为；健康探测变成 Node 职责
  或 Node 出现 Desktop 的 store。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 映射与顺序不漂移 | `crates/backend/src/session_setup/mcp/tests.rs` 的 `maps_stdio_and_http_in_canonical_plugin_id_order_with_exact_revisions`、`regenerates_when_package_version_changes_during_resolve`、`resolves_the_plugin_behind_the_agent_identity_the_runtime_supervises`、`resolves_nothing_for_an_agent_no_package_supplies`；Desktop 既有 `mcp` 测试套件在移动后原样通过（基线用例不变，仅新增） | Missing |
| 错误分类不变 | 同文件 `fails_closed_when_http_capability_is_missing`、`fails_before_send_when_load_capability_is_missing_for_a_non_empty_set`、`empty_set_does_not_require_load_capability`、`omits_incomplete_mcp_without_failing_complete_peers` | Missing |
| 值与文件边界 | 同文件 `desired_revision_excludes_setting_values`、`resolver_does_not_create_workspace_files` | Missing |

### 决策依据

D2；继承 Desktop 的 ACP Session setup 交付、工作流显式选择与 Host 健康探测归属。
