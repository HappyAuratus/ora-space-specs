# 插件模型 adapter：输入、隔离与收尾

决策依据：[插件拥有 CLI 适配](../../../decisions/node/agent-runtime/20261017-plugin-owned-model-adapters.md)。
证据固定到 desktop `f43f52fc`、claude-code-agent `8115695`、opencode-agent `b0e72d5`。Node 侧测试
走真实 Node 入口和插件协议，但 CLI 是回显 fixture；adapter 侧测试用插件自带的宿主模拟器；唯一跑
固定版本真实 adapter 与 CLI 的是 Claude 的命令行扫描测试。因此多数义务为 `Partial`。
[既有模型代理用例](model-proxy.md)只证明其已列范围，不覆盖新的 adapter 契约。

## Adapter startup must apply the frozen binding before the first prompt

- 风险：Agent 已启动但未应用配置便发首轮，或启动错包/错模型。
- 前置：固定发行/声明/adapter 版本、安装租约、模型绑定和可观察的 CLI。
- 触发：准备与启动；各阶段注入错摘要、缺能力、授权拒绝、provider 初始化失败。
- 必须成立：身份匹配、领取授权、应用上下文、就绪、prompt 顺序成立；失败停止启动并撤销已领取授权。
- 禁止结果：ready 标记单独当证明、首轮先于配置、使用 latest/共享凭据重试、部分准备资源遗留。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 生产启动顺序 | [`apps/ora-node/tests/agent_session.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/tests/agent_session.rs)：`a_configured_adapter_receives_its_grant_only_in_the_startup_context`、`an_unconfirmed_adapter_never_receives_a_prompt`（未确认已应用配置就没有 ACP 和 prompt）、`a_configuration_the_installed_package_cannot_prove_never_gets_a_grant`、`a_plugin_of_another_version_ends_the_session_without_starting_it`；[Claude `model_route_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/model_route_test.ts) 的 “a platform start spawns the adapter on its route and sends it nothing”。CLI 是回显 fixture，没有观察真实 CLI 首轮前的输入 | Partial |
| 失败收尾 | [`apps/ora-node/tests/agent_session.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/tests/agent_session.rs) 上述测试在网关核对 grant 领取与撤销、插件进程退出；[`apps/ora-node/src/model_proxy/tests.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/src/model_proxy/tests.rs) `adapter_route_revokes_a_grant_that_differs_from_the_frozen_configuration` 核对撤销且不留运行目录。缺按阶段的故障矩阵（provider 初始化失败，以及各阶段的租约、续期和目录） | Partial |

## Runtime context must not leak credentials through durable channels

- 风险：adapter 上下文或错误把长期 Key、短期令牌或客户端私钥写入历史。
- 前置：带可检测合成秘密的授权服务，普通/错误/取消路径和真实日志/账本输出。
- 触发：控制输入、插件握手/provider 初始化、CLI 输出、续期、封口与成果保存。
- 必须成立：长期 Key 仅在凭据服务；短期令牌只在本执行敏感运行通路；公开 CA 与私钥分开。非秘密冻结身份/选项/版本可在控制账本中保留，并与执行输入一致。
- 禁止结果：令牌成为 prompt/Settings/控制账本/文件字面值，CLI 错误原样公开。
- 验证义务与证据：[`apps/ora-node/tests/agent_session.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/tests/agent_session.rs) `a_configured_adapter_receives_its_grant_only_in_the_startup_context` 核对令牌不进历史、Thread 和插件环境；[`apps/ora-node/src/model_proxy/tests.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/src/model_proxy/tests.rs) `adapter_route_hands_the_token_only_to_the_startup_context`；[`empty_environment_does_not_inherit_host_authority`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/crates/process/tests/environment.rs)；[Claude `model_route_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/model_route_test.ts) 的令牌只在启动环境、错误只带码不带原文、帧与日志不含令牌和代理地址；[Claude `argv_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/argv_test.ts) 用固定版本真实 adapter 与 CLI 扫描 `/proc`，断言令牌不在任何命令行参数里；[OpenCode `model_route_test.ts`](https://github.com/ora-space/opencode-agent/blob/b0e72d58add1a111e4b9596265251e6e6bed6ffd/tests/model_route_test.ts) 同类检查。缺运行目录与成果的负向扫描、访问权限检查，以及“账本里的冻结配置足以校验重试”的正向核对。当前 `Partial`。

## Concurrent adapters must not share provider state or discovery results

- 风险：两个用户同 cwd 被缓存或共享连接合并，产生额度/模型串用。
- 前置：同插件、相同 cwd 的两个不同执行上下文，个人连接、模型或 revision 不同。
- 触发：并发启动、发现模型、进行轮次；期间编辑一个人的配置或升级包。
- 必须成立：进程/连接/目录和缓存各自归属明确；旧执行保持冻结；新执行使用新 revision。
- 禁止结果：共享进程来回 providers/set、cwd-only 缓存、全局环境突变或热切换在途 provider。
- 验证义务与证据：没有并发测试。[Claude `model_route_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/model_route_test.ts) 与 [OpenCode `model_route_test.ts`](https://github.com/ora-space/opencode-agent/blob/b0e72d58add1a111e4b9596265251e6e6bed6ffd/tests/model_route_test.ts) 拒绝客户端切换 provider 或重新认证，只证明单进程内路由不可变，不证明执行之间隔离。当前 `Missing`。

## Project and ambient settings must not override platform model routing

- 风险：配置目录隔离仍被项目 env、登录文件或更高优先级规则覆盖。
- 前置：固定支持版本，checkout 中有冲突配置，宿主环境有其他 provider/登录态，包含特殊字符输入。
- 触发：初始化 adapter、读取项目配置、首轮与辅助调用。
- 必须成立：所有请求仍走授权代理和冻结模型；无可靠覆盖能力时准备失败；原项目文件不变。
- 禁止结果：删除用户文件、继承长效凭据、修改平台权限、把字符串当 shell/原生配置程序执行。
- 验证义务与证据：[Claude `model_route_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/model_route_test.ts) 的 “every session setup carries the route's settings…”、“session options cannot choose another model, CLI or environment”、“the compiled adapter runs only its bundled CLI on a platform route”；[Claude `argv_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/argv_test.ts) 证明真实 CLI 收到的会话设置含平台代理地址；[OpenCode `model_route_test.ts`](https://github.com/ora-space/opencode-agent/blob/b0e72d58add1a111e4b9596265251e6e6bed6ffd/tests/model_route_test.ts) 的 “inherited OpenCode configuration variables cannot reach the route”。这些只核对生成的设置和环境，缺真实 CLI 在 checkout 有冲突配置时的出站观察和特殊字符输入。当前 `Partial`。

## Aliases and discovery must remain within the single-model authorization

- 风险：模型选择器或辅助别名访问第二模型，发现过程先使用未授权凭据。
- 前置：Claude 单模型绑定以及无绑定的配置页面；adapter 声明可选发现能力。
- 触发：展示配置、运行内发现、主/辅助调用、会话模型切换和不支持的发现端点。
- 必须成立：配置页用个人目录；运行内发现有上下文；真实调用只到冻结模型，未支持的能力不冒充成功。
- 禁止结果：影子运行、原生默认模型回退、用配置选项绕过 Gateway 比对、扩大授权模型集合。
- 验证义务与证据：[Claude `model_route_test.ts`](https://github.com/ora-space/claude-code-agent/blob/8115695883a9682f5d7816766afd9755b13a74cc/tests/model_route_test.ts) 的 “a frozen context becomes one pinned Messages route”（别名钉到冻结模型）、“a client can only ever select the frozen model”、“every model selector the adapter reports offers only the frozen model”、“model discovery on a platform route answers the frozen model without a probe”；[OpenCode `model_route_test.ts`](https://github.com/ora-space/opencode-agent/blob/b0e72d58add1a111e4b9596265251e6e6bed6ffd/tests/model_route_test.ts) 同类检查。缺真实 CLI 或受控上游上的请求核对（含辅助调用）和网页无启动副作用的验证。当前 `Partial`。

## Revocation and termination must close the adapter resource lifecycle

- 风险：取消后仍续期/推理，或清理另一执行的目录。
- 前置：运行与续期进行中，另一执行并发存在。
- 触发：准备失败、普通撤销、已验证 ending、主动结束、Node 正常停止与重启。
- 必须成立：无新调用，续期停止、授权撤销、所管理进程结束后清理本执行目录和租约；ending 保留原原因。
- 禁止结果：重新解析当前配置复活授权、恢复旧令牌、跨执行清理、以正常退出证明孤儿后代已回收。
- 验证义务与证据：[`apps/ora-node/src/model_proxy/tests.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/src/model_proxy/tests.rs) `renewable_access_keeps_token_in_memory_and_revokes_at_end`、`denied_renewal_notifies_the_conversation_to_stop`、`close_cancels_the_scheduled_renewal`；[`apps/ora-node/tests/agent_session.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/tests/agent_session.rs) 中用户结束后撤销授权、插件进程退出。缺 mTLS/真实网关、Node 停止与重启、跨执行清理和孤儿进程的覆盖。当前 `Partial`。

## Another compatible plugin must use the same platform integration seam

- 风险：所谓通用接口仍要求每个插件修改 Cloud UI/Node 授权分支。
- 前置：新 OpenCode/Claude adapter 路线和一个使用已有协议的最小测试插件。
- 触发：新增测试插件的声明、adapter 与包，完成个人配置和一次授权执行。
- 必须成立：平台无需增加该插件 ID 分支或专用页面；声明、配置、授权、取消、清理经同一接口。
- 禁止结果：测试硬编码特例、只测 JSON 不启动、把测试插件通过当作其他真实 CLI 已支持。
- 验证义务与证据：[`apps/ora-node/tests/agent_session.rs`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-node/tests/agent_session.rs) 中测试插件 `official/ora-space.echo` 走带冻结配置的路线，Node 没有该插件 ID 的分支；唯一的 ID 分支是旧 OpenCode 路线，由 `an_undeclared_plugin_never_takes_the_legacy_model_route` 守住；[`configured_session_waits_for_a_matching_model_adapter`](https://github.com/ora-space/desktop/blob/f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30/apps/ora-controller/tests/workspaces/agents.rs) 核对 Controller 只派给能力匹配的 Node。缺测试插件的 Cloud 个人配置与页面，以及可审计的变更范围证明；不支持协议的拒绝只部分覆盖。当前 `Partial`。
