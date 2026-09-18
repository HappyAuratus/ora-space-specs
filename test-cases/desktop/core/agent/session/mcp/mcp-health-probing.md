# Host 侧 MCP 运行健康探测

本文件用例依据
[Host 持有 MCP 运行健康决策](../../../../../../decisions/desktop/core/agent/session/mcp/20260916-mcp-health-probing.md)
登记。该 ADR 已评审通过（`approved`），实现已按 1a（传输、状态、错误码）与 1b（触发、查询、
事件、卡片、横幅、配对日志）落地；下表证据于落地时回填，测试按「生产接口」而非内部实现选择。
两处标为 `Partial` 的项说明见对应单元格。

## Probe Failures Must Not Change Delivery Semantics

### 风险

如果探测失败升级为 setup 失败、把成员移出 Effective MCP Set 或产生部分 `mcpServers`，一次远端
抖动或 API Key 填错会把仍可用的会话整体废掉，运行期事实将劫持根决策已经固定的投递语义。

### 前置状态

- installed snapshot 中存在配置完整的 stdio 与 HTTP MCP 成员。
- 探测已返回 `Unhealthy`（例如架构不符的 `mcp_spawn_failed` 或填错 Key 的
  `mcp_http_unauthorized`）。

### 触发

Host 为该 Session 执行 `session/new` 或 `session/load`。

### 必须成立

- 探测失败不使 `session/new` / `session/load` 失败，不产生部分 `mcpServers`，不改变
  Effective MCP Set 成员。
- 分辨率失败仍按根决策 D7 全量失败；健康结果不参与资格判定。
- 会话横幅在 setup 完成后非阻塞呈现该 Session Effective MCP Set 内的 Unhealthy 成员与稳定码；
  prompt 不被阻塞。

### 禁止结果

- 因探测失败而失败 setup、跳过入选成员或把完整集合降级为空集合。
- 把 `Unhealthy` 解释为配置不完整或资格缺失。
- 横幅阻塞 prompt，或把探测未完成渲染成失败。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| stdio 探测失败（spawn 失败、超时）后会话仍发送完整 `mcpServers` 且 setup 成功 | `plugin::operations::health_tests::probe_failure_does_not_change_session_delivery`（导入不可运行的 stdio MCP，卡片已为 `mcp_spawn_failed`，`SessionSetup::resolve` 仍解析出完整成员）；`session_setup::mcp::health::tests::probe_failure_does_not_change_the_delivered_snapshot` | `Covered` |
| 横幅只提示不阻塞 prompt，成员范围限于该 Session Effective MCP Set | `packages/app-shell` `features/chat/session-mcp-health-banner.test.tsx`（自动会话列出、未选择不出现、空显式集合无横幅、`Unknown` 不视为失败、跳转配置入口、横幅可见时 composer 仍可编辑并发送）；横幅为 `role="status"` 的纯呈现，不参与 prompt admission | `Covered` |
| 分辨率失败仍按 D7 全量失败，探测结果不进入资格判定 | `session_setup::mcp::selection_tests::*`、`session_setup::mcp::tests::fails_closed_when_http_capability_is_missing`、`session_setup::mcp::health::tests::probe_failure_does_not_change_the_delivered_snapshot` | `Covered` |

### 决策依据

- Host 侧 MCP 健康探测决策 D3：探测失败不改变投递语义。
- 该决策不变量 1。

## Health Identity Must Follow Member Revision And Session Cwd

### 风险

如果旧身份的健康结果被当作当前状态呈现，用户会看到已经失效的「可用/不可用」；如果用占位 `cwd`
伪造 workspace 探测，一个 Workspace 的失败会污染另一个 Workspace 的展示。

### 前置状态

- 成员经历 package version 更新、Configuration revision 变化、卸载或失去资格。
- 存在 args 依赖 `{ "context": "workspace" }` 的成员；插件卡片没有 Session `cwd`，会话有真实
  绝对 `cwd`。

### 触发

身份变化、卸载或失去资格后渲染卡片与横幅；安装、保存或「重新检测」触发探测。

### 必须成立

- 健康身份绑定 canonical Plugin ID、精确 package version、Configuration revision 与 transport；
  workspace-context 成员额外绑定该 Session 的绝对 `cwd`。
- 身份不匹配视为 `Unknown(not_probed)`；身份消失后旧结果立即失效，不再呈现。
- workspace-context 成员在无真实 `cwd` 的触发点保持 `Unknown(context_missing)`；会话探测使用
  该 Session 的真实绝对 `cwd`。
- Session 探测结果不写回全局插件卡片。

### 禁止结果

- 呈现与当前身份不匹配的旧健康结果。
- 用占位 `cwd` 伪造探测，把卡片上的 `Unknown(context_missing)` 变成 Healthy/Unhealthy。
- 卸载、更新、revision 变化或失去资格后残留旧健康展示。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 更新、revision 变化、卸载或失去资格后旧身份结果立即失效并停止呈现 | `session_setup::mcp::health::tests::invalidation_clears_plugin_identities_and_publishes`、`session_setup::mcp::health::tests::stale_identity_is_not_presented_as_current`（旧 revision 的结果不再呈现，身份不匹配读作 `Unknown(not_probed)`）、`plugin::operations::health_tests::uninstall_removes_the_health_row`、`session_setup::mcp::health::tests::configuration_incomplete_member_is_not_listed_or_probed`、`plugin::operations::health_tests::incomplete_settings_are_not_probed_until_a_complete_save`（只有新 revision 被呈现）、`plugin::operations::health_tests::reset_configuration_drops_the_health_row`（Reset All 清空 Settings 后旧身份行消失）。package version 更新走同一失效+探测路径（`Plugins::update`），但无独立 marketplace 更新用例 | `Partial` |
| workspace-context 成员在卡片保持 `Unknown(context_missing)`，「重新检测」不伪造占位 `cwd` | `session_setup::mcp::health::tests::workspace_context_member_binds_session_cwd_and_keeps_card_separate`、`plugin::operations::health_tests::workspace_context_mcp_stays_context_missing_on_the_card`（含重新检测）、`packages/app-shell` `features/settings/mcp-health-row.test.tsx`（卡片不为这类成员提供可伪造 cwd 的重新检测） | `Covered` |
| 会话探测使用真实绝对 `cwd`，结果不回写全局卡片 | `session_setup::mcp::health::tests::workspace_context_member_binds_session_cwd_and_keeps_card_separate`（会话身份绑定绝对 cwd；会话探测后卡片身份仍为 `not_probed`） | `Covered` |

### 决策依据

- Host 侧 MCP 健康探测决策 D1、D2：健康身份与成员 revision 对齐，workspace context 绑定 Session
  `cwd`。
- 该决策不变量 3、9、10。

## Probing Must Be Bounded Single-Flight And Non-Blocking

### 风险

探测包含 spawn 或外发 HTTP。如果没有硬超时、并发去重与进程回收，运行期抖动会被放大为会话不可用，
并发触发会重复 spawn 或重复外发，stdio 探测会留下僵尸进程。

### 前置状态

- 安装完成、配置保存成功、会话建立与「重新检测」可能并发指向同一健康身份。
- stdio MCP 进程可能在握手期间挂死。

### 触发

任一触发点发起探测；同一健康身份并发触发；探测达到硬超时。

### 必须成立

- 探测只执行 `initialize` 与 `tools/list`，受硬超时约束，超时上限明显短于 session setup 的
  30 秒预算。
- 同一健康身份的并发触发共享一次探测。
- 安装、保存与会话建立不因等待探测而阻塞；「重新检测」可以等待同一次 single-flight 结束。
- stdio 探测关闭 stdin 并在超时边界内确认进程回收；超时返回 `mcp_probe_timeout`。
- 配置保存成功只触发一次探测，不自动重试；再次探测只来自手动「重新检测」与会话建立时对
  `Unknown` 的补探测。

### 禁止结果

- 探测阻塞触发路径，或在时间上冒充 setup。
- 同一健康身份并发重复 spawn 或重复外发 HTTP。
- 探测后残留僵尸进程。
- 引入 TTL 或后台周期复检。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| stdio/HTTP 成功与失败矩阵（spawn 失败、提前退出、握手失败、超时、`tools/list` 失败、401、5xx、非 MCP 响应）有组件级证据 | `ora-utils` `mcp::tests::stdio_probe_succeeds_for_well_behaved_server`、`stdio_probe_maps_missing_binary_to_spawn_failed`、`stdio_probe_maps_early_exit_to_exited_prematurely`、`stdio_probe_maps_wrong_id_answer_to_handshake_failed`、`stdio_probe_maps_tools_list_error_to_tools_unavailable`、`stdio_probe_maps_hang_to_timeout_and_reaps_promptly`、`http_probe_succeeds_for_json_responses`、`http_probe_accepts_sse_framed_responses`、`http_probe_maps_unauthorized`、`http_probe_maps_forbidden_to_unauthorized`、`http_probe_maps_server_error`、`http_probe_maps_non_mcp_body_to_handshake_failed`、`http_probe_maps_tools_list_error_to_tools_unavailable`、`http_probe_maps_unreachable`、`http_probe_maps_silent_peer_to_timeout`，以及 `mcp::protocol::tests::default_probe_timeout_stays_well_below_a_session_setup_budget`（硬超时上界）等 `mcp::protocol::tests::*` | `Covered` |
| 同一身份并发触发只执行一次探测且结果共享 | `session_setup::mcp::health::tests::concurrent_triggers_share_one_probe`（并发两次触发只产生一次连接）、`session_setup::mcp::health::tests::session_observation_reuses_a_matching_identity` | `Covered` |
| 安装、保存、会话建立不被探测阻塞；「重新检测」等待 single-flight 后查询到新结果 | `plugin::operations::health_tests::importing_a_slow_http_member_does_not_wait_for_the_probe`（安装立即返回，卡片仍为 `Unknown(not_probed)`）、`plugin::operations::health_tests::incomplete_settings_are_not_probed_until_a_complete_save`（保存共用同一异步触发实现）、`plugin::operations::health_tests::save_probes_exactly_once_without_automatic_retry`（保存只启动一次 server、不自动重试）、`session_setup::mcp::health::tests::session_observation_reuses_a_matching_identity`（会话补探测 fire-and-forget）、`packages/app-shell` `features/settings/mcp-health-row.test.tsx`（重新检测等待并呈现新结果） | `Covered` |
| stdio 握手或超时后探测进程被确认回收 | `ora-utils` `mcp::tests::stdio_probe_waits_for_child_exit_after_a_handshake`（成功路径只有在子进程退出后才返回）、`mcp::tests::stdio_probe_maps_hang_to_timeout_and_reaps_promptly`（超时路径 kill + wait 有界） | `Covered` |

### 决策依据

- Host 侧 MCP 健康探测决策 D1、D2、D6：异步触发、有界握手、single-flight、无自动重试。
- 该决策不变量 5、8。

## Probe Surfaces Must Stay Secret-Free

### 风险

探测必须看见真实 env 与 headers 才有效。一旦实值、第三方响应体或本地错误字符串进入日志、事件、
查询响应或 UI，凭据会泄漏；健康错误码若与既有 setup 稳定码碰撞，会破坏对外错误契约。

### 前置状态

- HTTP MCP 的 Setting 含凭据（例如填错的 API Key）；stdio 成员 env 含 Setting 实值。
- 探测失败携带 OS 或第三方错误文本。

### 触发

保存成功后探测失败；查询、AppEvent、配对日志与 UI 呈现探测结果。

### 必须成立

- 探测使用与 ACP 投递相同的 resolver 绑定产物（真实 env/headers），不存在第二条解析路径。
- 查询响应、`McpHealthChanged`、配对日志与 UI DTO 只携带身份、稳定错误码或 Unknown reason；
  原始错误文本只进 operator 日志。
- `McpHealthChanged` 只携带 `plugin_id`；配对日志字段限于成员身份、稳定码、耗时，且可与该次
  Session 的 `sending ACP session configuration` 日志关联。
- 错误码族封闭，不与既有 setup 稳定码碰撞；新增细分码另作决策。

### 禁止结果

- 任何公开表面出现 Setting 实值、凭据、argv、env、headers 或第三方响应体。
- 把本地 OS 错误字符串或响应体送进 UI。
- 配对日志改写既有发送日志的语义。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `mcp_http_unauthorized` 场景下查询、事件、日志与 UI 不含 Key 原值与响应体 | `plugin::operations::health_tests::http_credential_failure_reaches_the_card_without_leaking_the_key`（带 Setting 凭据的 HTTP 成员失败后，安装列表、查询响应、探测响应与日志都不含 Key、`Bearer`、header 名）；分类本身见 `ora-utils` `mcp::tests::http_probe_maps_unauthorized` / `http_probe_maps_forbidden_to_unauthorized`；呈现见 `packages/app-shell` `features/settings/mcp-health-row.test.tsx`（`mcp_http_unauthorized` 码）。入库用例改用快速失败的 HTTP 不可达分支，因为测试环境无法在可信 TLS 端点后伪造真实 401；人工端到端复核时另以真实 HTTPS 端点复现过 401/403/500，分别落到 `mcp_http_unauthorized` 与 `mcp_http_server_error` | `Partial` |
| 配对日志与该次 Session 发送日志可关联，字段 secret-free | `session_setup::mcp::health::tests::session_observation_pairs_secret_free_results`（同一 `session_id`、身份、稳定码、耗时；无实值、凭据、header、端点）；发送边界与探测收敛为单一 seam：`log_session_mcp_request` 变为私有，`session/new` 与两条 `session/load`（prompt attach 与 live refresh）都经 `agent_runtime::record_session_mcp_boundary`，配对与先后由结构保证；`plugin::operations::health_tests::session_health_observation_is_logged_after_the_acp_send_boundary` 断言发送日志先于同 session 的探测结果日志，且去掉观测会让该用例失败 | `Covered` |
| 健康错误码族封闭且与既有 setup 稳定码无碰撞 | `ora-contracts` `mcp_health::tests::serializes_every_error_code_token`、`session_setup::mcp::health::tests::maps_every_probe_error_to_a_closed_product_code`；两族是两个不同的 Rust 类型（`McpHealthErrorCode` 与 `SessionMcpErrorCode`），碰撞在类型层面不可表达 | `Covered` |

### 决策依据

- Host 侧 MCP 健康探测决策 D2、D4、D5、D6：同一解析产物、封闭错误码族、secret-free 呈现与
  配对日志。
- 该决策不变量 4、6。

## Health Presentation Must Stay Orthogonal To Selection And Setup Success

### 风险

健康若并入安装状态或配置完整性的呈现，或被命名为 `MCP Ready`，第三个独立事实会被误读成 setup
成功；工作流会话横幅若出现未选择的成员，会破坏显式选择边界。

### 前置状态

- 普通聊天（`Automatic`）与工作流（`Explicit`，含空集合）会话并存。
- 插件卡片与会话横幅两种呈现面同时可见。

### 触发

渲染插件卡片与会话横幅；建立 Explicit 空集合的工作流会话。

### 必须成立

- 插件卡片以独立第三状态行呈现健康（含错误码与「重新检测」），不改写安装状态与配置完整性，
  不得命名为 `MCP Ready`。
- 配置不完整或已失去资格的插件不被探测，也不展示健康行。
- 会话横幅只呈现该 Session Effective MCP Set 内的 Unhealthy 成员；Explicit 空集合不发起会话
  补探测，不展示 MCP 横幅，发送空 `mcpServers`。
- 新增文案保持 Host 视角，不承诺「会话内已生效」。

### 禁止结果

- 未选择的 MCP 出现在工作流会话横幅。
- 安装/保存触发的探测改变任何会话的 MCP 选择。
- 把健康合并进前两个事实的展示，或把探测未完成渲染成失败。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 卡片三个事实分离呈现，配置不完整插件不展示健康行 | `packages/app-shell` `features/settings/plugin-manager.health.test.tsx`（无 Settings 的合格 MCP 与 required Settings 齐全的 MCP 都展示第三行；`incomplete` 只保留「需要配置」，`unavailable` 只保留「配置不可用」）、`features/settings/mcp-health-row.test.tsx`、`session_setup::mcp::health::tests::configuration_incomplete_member_is_not_listed_or_probed` | `Covered` |
| 工作流横幅只含白名单内 Unhealthy 成员；Explicit 空集合无补探测、无横幅 | `packages/app-shell` `features/chat/session-mcp-health-banner.test.tsx`（未选择不出现；空显式集合无横幅）、`session_setup::mcp::health::tests::explicit_empty_selection_has_no_members_or_backfill`（空集合不发起补探测）、`session_setup::mcp::health::tests::explicit_selection_backfills_only_its_whitelist`（非空白名单只补选中成员） | `Covered` |
| 新增文案无 `MCP Ready` 或「会话内已生效」表述 | `features/chat/session-mcp-health-banner.test.tsx`、`features/settings/mcp-health-row.test.tsx`（健康文案显式声明「不代表该 MCP 已在会话内生效」）；文案源见 `features/settings/translations/plugins.ts` 与 `features/chat/translations.ts` | `Covered` |

### 决策依据

- Host 侧 MCP 健康探测决策 D1、D5：健康是第三个独立事实，探测与选择正交。
- 该决策不变量 2、11、12。

## Health State Must Be Memory-Only

### 风险

如果健康状态被持久化，上一次进程的探测结果会冒充当前健康；重启后展示的旧状态会误导用户对
「现在连不连得上」的判断。

### 前置状态

- 探测已产生 Healthy/Unhealthy 结果。
- Ora 进程重启。

### 触发

重启后打开插件卡片或建立会话。

### 必须成立

- 重启后所有健康状态回到 `Unknown(not_probed)`，界面不显示旧状态。
- 由 D1 触发点重新探测；会话建立直接复用内存中身份匹配的结果。

### 禁止结果

- 从持久化数据恢复健康状态。
- 把历史结果当作与当前身份匹配的状态呈现。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 重启后卡片与横幅不显示任何旧健康状态，全部回到 `Unknown(not_probed)` | `session_setup::mcp::health::tests::restart_starts_from_not_probed`（新建 store 无历史；实现不写任何持久化、无 TTL/周期复检） | `Covered` |
| 会话建立复用内存中身份匹配的结果，身份不匹配视为 `Unknown(not_probed)` | `session_setup::mcp::health::tests::session_observation_reuses_a_matching_identity`、`session_setup::mcp::health::tests::stale_identity_is_not_presented_as_current` | `Covered` |

### 决策依据

- Host 侧 MCP 健康探测决策 D1、D4：状态只在内存，重启后重新探测。
- 该决策不变量 7。
