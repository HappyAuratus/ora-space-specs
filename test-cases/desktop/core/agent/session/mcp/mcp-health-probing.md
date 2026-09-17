# Host 侧 MCP 运行健康探测

本文件用例依据
[Host 持有 MCP 运行健康决策](../../../../../../decisions/desktop/core/agent/session/mcp/20260916-mcp-health-probing.md)
登记。该 ADR 已评审通过（`approved`），实现尚未开始，全部证据暂为 `Missing`；落地 1a/1b 阶段时
必须按证据状态回填本文件。

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
| stdio 探测失败（spawn 失败、超时）后会话仍发送完整 `mcpServers` 且 setup 成功 | 尚无；实现未开始 | `Missing` |
| 横幅只提示不阻塞 prompt，成员范围限于该 Session Effective MCP Set | 尚无 | `Missing` |
| 分辨率失败仍按 D7 全量失败，探测结果不进入资格判定 | 尚无 | `Missing` |

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
| 更新、revision 变化、卸载或失去资格后旧身份结果立即失效并停止呈现 | 尚无；实现未开始 | `Missing` |
| workspace-context 成员在卡片保持 `Unknown(context_missing)`，「重新检测」不伪造占位 `cwd` | 尚无 | `Missing` |
| 会话探测使用真实绝对 `cwd`，结果不回写全局卡片 | 尚无 | `Missing` |

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
| stdio/HTTP 成功与失败矩阵（spawn 失败、提前退出、握手失败、超时、`tools/list` 失败、401、5xx、非 MCP 响应）有组件级证据 | 尚无；实现未开始 | `Missing` |
| 同一身份并发触发只执行一次探测且结果共享 | 尚无 | `Missing` |
| 安装、保存、会话建立不被探测阻塞；「重新检测」等待 single-flight 后查询到新结果 | 尚无 | `Missing` |
| stdio 握手或超时后探测进程被确认回收 | 尚无 | `Missing` |

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
| `mcp_http_unauthorized` 场景下查询、事件、日志与 UI 不含 Key 原值与响应体 | 尚无；实现未开始 | `Missing` |
| 配对日志与该次 Session 发送日志可关联，字段 secret-free | 尚无 | `Missing` |
| 健康错误码族封闭且与既有 setup 稳定码无碰撞 | 尚无 | `Missing` |

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
| 卡片三个事实分离呈现，配置不完整插件不展示健康行 | 尚无；实现未开始 | `Missing` |
| 工作流横幅只含白名单内 Unhealthy 成员；Explicit 空集合无补探测、无横幅 | 尚无 | `Missing` |
| 新增文案无 `MCP Ready` 或「会话内已生效」表述 | 尚无 | `Missing` |

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
| 重启后卡片与横幅不显示任何旧健康状态，全部回到 `Unknown(not_probed)` | 尚无；实现未开始 | `Missing` |
| 会话建立复用内存中身份匹配的结果，身份不匹配视为 `Unknown(not_probed)` | 尚无 | `Missing` |

### 决策依据

- Host 侧 MCP 健康探测决策 D1、D4：状态只在内存，重启后重新探测。
- 该决策不变量 7。
