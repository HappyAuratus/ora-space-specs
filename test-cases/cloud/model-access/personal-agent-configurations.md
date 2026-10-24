# 个人 Agent 配置、冻结与授权

决策依据：[个人 Agent 配置](../../../decisions/cloud/model-access/20261017-personal-agent-configurations.md)。
除各节另列提交外，证据固定到 cloud `7a48f34`。多数义务目前只有 store 层证据（真实 PostgreSQL、直接调用核心分发，
未经 HTTP）或单用户 HTTP 证据，按 `Partial` 标记并写明缺口；新增受控模型浏览器联合证据见配置缺口一节，不替代真实供应商或双用户验证。下文 DB 指
[`personal_agent_configurations_db_test.go`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/internal/core/personal_agent_configurations_db_test.go)。既有 OpenCode/个人连接的证据保持原范围；供应商确认不是 Ora 实测证据。

## Plugin configuration must reuse owner-scoped connections without exposing keys

- 风险：插件另建 Key 库，或空间管理员继承他人个人配置权。
- 前置：两用户同空间、管理员、同名不同来源插件，以及可被两插件选择的连接。
- 触发：网页内联创建/选择连接、配置读写、Key 写入和跨用户重试。
- 必须成立：所有连接来自同一用户库；Settings 与秘密接口分开；读取只给允许字段和已配置状态。
- 禁止结果：请求体决定归属、普通 Setting 存 Key、管理员读取个人值、删除插件导致共用连接被删。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 用户/来源隔离 | [`TestPersonalAgentConfigurationHTTP`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/integration/personal_agent_configurations_test.go) 单用户经生产 HTTP 拒绝请求体自报 `userId`、同幂等键重放返回原结果；DB `TestPersonalAgentConfigurationRepairAndOwnership` 中管理员读不到成员配置、写入得到 `not_found`。缺 HTTP 跨用户/管理员矩阵与同名不同来源插件 | Partial |
| 网页秘密通路 | [`editor.test.tsx`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/frontend/src/features/agent-configurations/editor.test.tsx) 核对 Key 不进前端缓存且提交前清空输入；[`TestPersonalAgentConfigurationHTTP`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/integration/personal_agent_configurations_test.go) 核对读取不含凭据引用、上游地址、Key 或令牌，Settings 里传 `apiKey` 被拒。新增[浏览器套件](https://github.com/ora-space/cluster/blob/b270261fe0503b38d3e26ad480d35834ce1bafd2/scripts/personal-agent-readiness-browser.py) 经真实页面及 Gateway/model-gateway 写入合成 Key，并在截图前核对密码框清空；仍缺日志、持久存储与请求缓存的联合核对 | Partial |
| 连接单一事实来源 | [`editor.test.tsx`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/frontend/src/features/agent-configurations/editor.test.tsx) 的内联新建连接写入共享连接库、清除插件选项保留连接库。缺两插件共用连接经轮换/删除的集成验证 | Partial |

## Configuration writes must reject stale revisions without a declaration change

- 风险：只检查 schema 指纹，同一声明下后写覆盖先写，或清除重建产生旧 revision 再次有效。
- 前置：两个编辑器读到同一用户/插件、发行、声明指纹和 revision；另备无配置的首次创建状态。
- 触发：保存/保存、保存/清除竞争，同一幂等请求重放，清除重填后提交旧页面；OpenCode 同时存在全局默认版本。
- 必须成立：不同修改只有一个提交成功，陈旧请求冲突并保留草稿；清除推进 revision，重填不复用旧 revision；合法重放无额外推进。
- 禁止结果：自动重试覆盖胜者、并发建两份默认、清除重置版本、以插件 revision 绕过全局默认 version。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| CAS 与幂等 | DB `TestPersonalAgentConfigurationCASClearAndReplay`：并发首次创建一成一败（`revision_conflict`），清除推进 revision 且重放一致，旧 revision 重填冲突，同幂等键不同输入 `idempotency_conflict`；[`TestPersonalAgentConfigurationHTTP`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/integration/personal_agent_configurations_test.go) 经生产 HTTP 串行覆盖重放、陈旧 409、清除后旧 revision 409 与重填。缺 HTTP 层并发 | Partial |
| 草稿与后续运行 | [`editor.test.tsx`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/frontend/src/features/agent-configurations/editor.test.tsx) 冲突时保留陈旧草稿并要求显式对齐；DB `TestPersonalAgentRunFreezeAndUnavailableAdapter` 运行冻结 revision 1，之后编辑不改变快照。缺“两编辑器冲突 → 胜者 → 运行冻结胜者”的连贯验证 | Partial |
| OpenCode 双资源版本 | DB `TestPersonalAgentOpenCodeUsesGlobalDefaultAndRepairsRemovedSettings`：OpenCode 插件配置不能保存连接/模型（`agent_use_model_default`），读取展示全局默认。缺两种版本互不覆盖的直接断言 | Partial |

## Explicit Claude selection must not change or inherit the global default

- 风险：Claude 误用 OpenCode default，或两个默认记录漂移。
- 前置：已存在 OpenCode 默认值、不同 Claude 配置和工作流 default 槽位。
- 触发：分别编辑两种插件配置并从普通任务与原工作流入口创建运行；包括 Claude 缺配置。
- 必须成立：OpenCode 插件页和全局默认读写同一选择；Claude 独立解析；缺配置明确拒绝。
- 禁止结果：复制 default、切换全局 CLI、缺配置回退其他人/槽位/宿主凭据。
- 验证义务与证据：DB `TestPersonalAgentOpenCodeUsesGlobalDefaultAndRepairsRemovedSettings` 证明 OpenCode 读取全局默认而不复制；`TestPersonalAgentAdmissionRollback`、`TestPersonalAgentInvalidDeclarationCannotUseLegacyFallback` 证明声明缺失或非法时不回退旧路线。新增 [`TestPersonalAgentMissingModelSelectionHTTPAdmissionAndRepair`](https://github.com/ora-space/cloud/blob/80481ce4433080adcb6d15f450cbd6b1b905a4ff/integration/personal_agent_admission_http_test.go) 经 HTTP 核对已有可用全局默认时，Claude 从未选择或清除选择仍拒绝；修复后固定其个人配置 revision，全局默认不变。工作流 default 槽位入口和两种默认的连贯编辑验证仍缺，整体保持 `Partial`。

## Personal model gaps must stay actionable across readiness and admission

- 风险：已保存的 Claude 配置因连接停用、Key 清除或模型撤下而不可用，却显示临时故障或失去修复入口。
- 前置：当前用户拥有有效的 Claude 插件配置、个人连接和模型，部署具备对应运行能力。
- 触发：停用或删除连接、清除 Key、撤下所选模型后读取任务就绪状态或再次尝试绑定运行；另检查未配置插件模型选择的提示。
- 必须成立：就绪投影和最终配置绑定准入给出同一公开原因；连接、凭据和模型问题保留个人连接入口，未配置插件模型选择保留插件设置入口。修复后原任务页恢复就绪并保留未发送草稿；拒绝绑定不留下部分模型绑定或 Agent 配置快照。
- 禁止结果：直接暴露领域内部缺口码、把永久配置缺口统一显示为“稍后重试”、引导用户修改无关默认值或静默回退。
- 新增公开边界证据固定到 cloud `80481ce4433080adcb6d15f450cbd6b1b905a4ff`、cluster `b270261fe0503b38d3e26ad480d35834ce1bafd2`；既有领域/组件证据仍按各自链接固定。

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 当前用户的配置缺口在读取与绑定准入中一致 | [`TestPersonalAgentReadinessAndAdmissionUsePublicModelGaps`](https://github.com/ora-space/cloud/blob/c3ad6761ec3d6353f2c93ad4eac2054ff9d924f6/internal/core/personal_agent_readiness_db_test.go) 使用真实 PostgreSQL：有效配置先可用，再分别停用/删除连接、移除凭据或撤下模型；经 `Store.Public` 读取 readiness，并经绑定准入核对同一公开错误及无残留绑定/快照。未经过 HTTP 适配层 | Covered |
| 提示与处理入口匹配 | [`agent-readiness.test.tsx`](https://github.com/ora-space/cloud/blob/c3ad6761ec3d6353f2c93ad4eac2054ff9d924f6/frontend/src/features/issues/conversation/agent-readiness.test.tsx) 组件测试直接检查连接、凭据、模型及默认模型缺口的文案与个人连接链接；插件缺配置、声明缺失或需修复时链接到插件设置 | Covered |
| 未配置插件选择的完整准入链 | [`TestPersonalAgentMissingModelSelectionHTTPAdmissionAndRepair`](https://github.com/ora-space/cloud/blob/80481ce4433080adcb6d15f450cbd6b1b905a4ff/integration/personal_agent_admission_http_test.go) 使用真实 HTTP/PostgreSQL，在全局默认可用时分别创建“从未配置”和“已清除”状态；任务 readiness、手动 Run、任务提及、任务消息均明确拒绝，公开读取无残留运行/评论/消息；显式保存插件模型选择后恢复准入，Run 固定修复后的个人配置 revision。凭据仅为前置的密文边界替身，不声称真实 Node 执行 | Covered |
| 生产 HTTP 与浏览器联合修复 | [可复跑浏览器套件](https://github.com/ora-space/cluster/blob/b270261fe0503b38d3e26ad480d35834ce1bafd2/scripts/personal-agent-readiness-browser.py) 经生产 UI/HTTP 对未配置插件选择、停用连接、缺 Key、模型撤下四条链逐一验证：原因/入口一致、禁用发送、直接 POST 409 且无新增消息；经链接修复后保留草稿并恢复发送，由当前消息 ID 定位新 Run，校验官方 Claude 0.4.1、正确 Agent 与修复后的模型，收到回复并正常结束。真实 Gateway、Cloud/PostgreSQL、Controller、Sandbox、Node 和官方插件；开发登录、受控模型 API、初始 Git 快照替身。[组件回归](https://github.com/ora-space/cloud/blob/80481ce4433080adcb6d15f450cbd6b1b905a4ff/frontend/src/features/issues/conversation/conversation-panel.test.tsx) 的 “refreshes the selected blocked Agent…” 核对跨窗口修复后的原页恢复和草稿保留。运行与证据格式见[套件说明](https://github.com/ora-space/cluster/blob/b270261fe0503b38d3e26ad480d35834ce1bafd2/docs/testing/personal-agent-readiness.md) | Covered |

## Claude thread permissions must protect the initiator's model access

- 风险：Claude 接入沿用普通 Thread 成员追加权，导致其他成员消耗发起者的个人额度。
- 前置：带个人绑定的 Claude Thread，发起者、其他成员、非发起者租户管理员，以及无绑定历史 Thread 对照。
- 触发：各角色读取、追加、结束；随后撤销成员/账号或模型绑定，进入 ending/ended，再重放旧界面请求。
- 必须成立：以下当前资格与状态由服务器检查，界面据同一能力显示；无绑定历史 Thread 保持原规则。
- 禁止结果：管理员结束权变成追加权、撤销后仍可追加、关闭后无法读取历史、前端禁用而 HTTP 仍受理。

| 场景 | 读取 | 追加 | 结束 |
| --- | --- | --- | --- |
| 可受理状态、有效发起者 | 允许 | 允许 | 允许 |
| 可受理状态、其他有效成员 | 允许 | 拒绝 | 拒绝 |
| 可受理状态、非发起者当前租户管理员 | 允许 | 拒绝 | 允许 |
| 成员或账号资格已失效 | 拒绝 | 拒绝 | 拒绝 |
| 绑定撤销、仍有效的发起者/租户管理员 | 允许 | 拒绝 | 按现有状态允许收尾 |
| ending/ended、当前有效成员 | 允许历史读取 | 拒绝 | 继承 Thread 状态/幂等规则，不重开运行 |

- 验证义务与证据：DB `TestPersonalAgentThreadOwnershipAndPermanentRevocation` 直接核对发起者可追加和结束、非发起者租户管理员可结束不可追加、撤销后重新启用不恢复追加而发起者仍可结束。缺生产 Thread HTTP 的角色/状态/重放矩阵、成员或账号失效，以及浏览器 canAppend/canEnd 与接口一致性。当前 `Partial`。旧 OpenCode 权限证据不能替代。

## Run creation must freeze the complete execution configuration atomically

- 风险：配置编辑或 catalog 更新改变排队运行，失败留下半个运行。
- 前置：合法发行声明和个人配置；并发升级包、编辑 Settings/模型、替换 Key。
- 触发：创建运行、延迟派发、准备重试；提交错误版本/协议/目标/引用。
- 必须成立：发起者、包/target/摘要、声明、adapter 版本、选项 revision 与模型绑定一次冻结；失败原子回滚。
- 禁止结果：客户端自报快照被信任、旧运行重读默认值、事务中访问网络、配置失败仍有可派发运行。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 并发冻结与回滚 | DB `TestPersonalAgentRunFreezeAndUnavailableAdapter` 冻结包摘要、revision 与 Settings，包升级后排队运行仍用原发行；`TestPersonalAgentAdmissionRollback` 拒绝时不留绑定或快照；[`TestDeclaredAgentSessionSnapshotIsPrivateAndImmutable`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/internal/core/declared_agent_startup_db_test.go) 核对实际派发的是冻结配置且 Settings 不进 prompt。缺真实事务交错 | Partial |
| 版本及能力拒绝 | DB 上述两个测试覆盖 `agent_adapter_unavailable`；`TestDeclaredAgentCapabilityRefusalSettlesBeforePrompt` 覆盖能力不符在首轮前结算。缺生产准入接口上的协议不符、`agent_target_unavailable`、`agent_release_conflict` 矩阵 | Partial |
| 共享展示脱敏 | 没有核对 Thread/运行详情响应不泄露个人选项或凭据引用的测试 | Missing |

## Editing and revocation must have distinct effects on existing runs

- 风险：普通配置编辑热切换，或撤销因 DNS 失败/重新启用失效。
- 前置：运行处于 pending/active/idle，并持有可持续流式的授权。
- 触发：编辑选项、改模型、替换 Key、清插件选项、清 Key、停用/删除连接、失去成员资格、结束。
- 必须成立：普通编辑只影响新运行；真正撤销拒绝后续和在途访问；纯降权不依赖出网可用。
- 禁止结果：复活旧绑定、将配置清除误作共用连接删除、回退其他凭据、静默 provider 切换。
- 验证义务与证据：DB `TestPersonalAgentRunFreezeAndUnavailableAdapter` 证明编辑不改冻结快照，`TestPersonalAgentThreadOwnershipAndPermanentRevocation` 证明撤销不复活。缺 PG 授权状态、受控 HTTP/SSE、DNS 不可用与 Node 续期的联合矩阵。当前 `Partial`。

## Upgrades must preserve history and require explicit repair of changed declarations

- 风险：旧编辑器覆盖新配置，升级迁移改写历史输入或丢值。
- 前置：既有 OpenCode 用户/历史运行、旧插件无声明、新包同指纹或不同指纹。
- 触发：数据库升级、旧页面保存、卸载重装、来源或 target 改变、仅排版变化的声明升级（字节不同、语义相同）、新旧运行分别启动。
- 必须成立：旧连接/默认/Key 引用不复制；历史不回填；冲突保留值/草稿并拒绝；仅排版变化也按指纹不匹配处理并要求显式修复，不做兼容猜测；新值只用于新 run。
- 禁止结果：插件迁移脚本在 Cloud 执行、未知字段静默删除、旧包被原地修改、旧绑定升级为新语义。
- 验证义务与证据：[`TestMigration0050PersonalAgentConfigurationsUpgrade`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/integration/personal_agent_migration_test.go) 证明升级保留默认值及其版本、不回填历史；[`TestMigration0060ModelAdapterFreshAndUpgrade`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/integration/node_model_adapter_migration_test.go) 覆盖新建与升级；DB `TestPersonalAgentConfigurationRepairAndOwnership` 证明声明只多一个换行也要求修复。缺部署混合版本和浏览器修复路径。当前 `Partial`。

## Changed declarations must permit explicit repair without overwriting concurrent edits

- 风险：所有指纹不匹配的保存均拒绝，用户无法修复；或修复跳过 CAS、自动丢弃字段并改写在途配置。
- 前置：已保存旧指纹/值/revision，发布的新声明增加必填项或移除字段，旧运行仍持旧快照。
- 触发：旧编辑器保存；重新加载并显式修复；修复过程中另一编辑器保存或管理员再次升级。
- 必须成立：旧保存冲突；重新加载区分新声明与旧值；合法修复按当前发行、新指纹及预期旧 revision 原子提交。
- 禁止结果：只能删除重建、无条件覆盖、隐藏丢字段、默认值自动批准、改写 Key 或已有运行。
- 验证义务与证据：DB `TestPersonalAgentConfigurationRepairAndOwnership` 覆盖普通保存返回 `agent_configuration_repair_required`、单独修复成功、第二次修复 `revision_conflict`；[`editor.test.tsx`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/frontend/src/features/agent-configurations/editor.test.tsx) 覆盖确认后才修复并列出被移除字段。缺修复时缺必填/类型错误、双重竞争和 HTTP 层验证。当前 `Partial`。

## One personal configuration must resolve predictably across spaces with different declarations

- 风险：两个空间选定不同声明的发行，单份配置在空间间来回失配，或被静默回退成某个空间的版本。
- 前置：同一用户的空间 A、B 选定同一插件的不同声明版本（指纹不同），配置按 A 的指纹保存且有效。
- 触发：在 A、B 分别打开个人配置页与创建运行；随后在 B 的发行上下文显式修复，再回 A 创建运行。
- 必须成立：配置页按所在空间解析发行；修复前 A 可创建、B 明确拒绝并说明声明指纹失配；在 B 修复后 B 可创建、A 变为明确拒绝并提示修复；拒绝原因可与其他缺口（权限、安装、凭据）区分。
- 禁止结果：按 latest 或任一空间版本静默回退、两个空间同时声称可用、把失配报成权限或安装错误、修复后另一空间无提示复用旧值启动。
- 验证义务与证据：DB `TestPersonalAgentConfigurationAcrossSpaces` 核对 B 需修复、在 B 修复后 A 转为需修复。缺两空间分别创建运行的拒绝码和 HTTP 层验证。当前 `Partial`。

## Claude model aliases must resolve to the single authorized model

- 风险：辅助或深度调用隐式访问第二模型，兼容标签误判协议。
- 前置：一个 Messages 连接和明确模型；另有 Completions 连接、未授权模型和主/辅助别名配置。
- 触发：启动 Claude、调用主/辅助路径、会话切模型及越权请求。
- 必须成立：实际每个推理请求都使用冻结模型；不兼容配置启动前拒绝；不能满足单模的固定版本不开放。
- 禁止结果：网关任意改写模型、授权第二模型、以同名供应商判断协议或回退直连。
- 验证义务与证据：[`TestProxyRejectsModelMismatchAndSuppressesUpstreamErrors`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/internal/modelgateway/service_test.go) 证明网关拒绝非冻结模型；[`anthropic_test.go`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/internal/modelgateway/anthropic_test.go) 的 `TestPinnedClaudeACPHaikuBetaCombination` 固定实际头组合。缺固定 CLI/adapter 流量证明每个辅助别名都只请求冻结模型。当前 `Partial`。

## Real Claude runs must complete with personal credentials and released resources

- 风险：只验证表单或供应商声明，实际 CLI、成果与撤销未贯通。
- 前置：无部署共享 Key/宿主登录态的真实栈，固定插件/CLI/adapter，用户经网页设置个人连接。
- 触发：真实两轮任务和工具操作、正常结束；独立验证取消/失败及授权撤销。
- 必须成立：同一 Run 的流式/工具调用、成果/Git 身份、历史和 Workspace 释放可核验，敏感材料不进入成果。
- 禁止结果：以模拟器/HTTP 200/另一 Run 替代、把上游 Messages 确认当端到端通过、把正常退出当撤销证据。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 受限协议兼容 | [`anthropic_test.go`](https://github.com/ora-space/cloud/blob/7a48f3403ae3c6daff974b787708a372226012ef/internal/modelgateway/anthropic_test.go)：`TestPinnedCLIClaudeModelBetas`、`TestPinnedClaudeAgentSDKCapabilities`、`TestPinnedClaudeACPHaikuBetaCombination` 回放固定 CLI/SDK 的请求头；`TestClaudeMessagesQueryHeadersAndToolStream` 覆盖 query、头、SSE 工具数据和单次请求不重发；`TestClaudeProtocolRejectsUnknownSurface` 覆盖未知路径/方法/头拒绝；`TestFrozenAnthropicDiscoveryUsesOnlyCurrentGrant` 覆盖发现只用当前授权 | Covered |
| 真实网页正常链路 | 新增[浏览器套件](https://github.com/ora-space/cluster/blob/b270261fe0503b38d3e26ad480d35834ce1bafd2/scripts/personal-agent-readiness-browser.py) 经网页配置连接/合成 Key/模型并驱动官方 Claude 回复、正常结束；模型 API 受控，外部初始 Git 同步用 preset 替身。仍缺真实供应商个人 Key、成果/Git 身份和敏感材料排除的可复跑联合证据 | Partial |
| 隔离与收尾 | 没有双用户并发、错误/取消/撤销与真实 Key 排除的可复跑入库测试 | Missing |
| 旧 OpenCode 回归 | [cluster 固定版本验收脚本](https://github.com/ora-space/cluster/blob/8f39f4219addbd6c788255cea416dff3e7c24644/scripts/opencode-session-acceptance.py) 的 fixture 路径用固定官方包在升级后的栈上跑两种协议的真实 CLI 会话、交付、Git 身份和 Workspace 释放；模型是受控 fixture，不是真实供应商 | Partial |
