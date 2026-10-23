# Agent 静态声明与发布身份

决策依据：[Agent 配置声明](../../../../decisions/desktop/plugin/manifest/20261017-agent-configuration-declarations.md)。
证据固定到 desktop `8e497d7`、cloud `abc7b31`。Rust 与 Go 共用同一份有效/无效声明向量
（[`agent-configuration-contract.json`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/crates/plugin-config/tests/fixtures/agent-configuration-contract.json)，
27 例，与 cloud `internal/pluginmarket/testdata/` 字节一致），这是跨语言一致性的主要证据；发布工具（Deno）只钉了摘要。
没有任何一项义务拿到完整的“所需直接证据”，缺口逐项写明；现网旧 Desktop 不在当前树中，其行为只有按旧编译器规则的推断。

## Agent declarations must be strict and free of personal values

- 风险：声明被解释为用户凭据、可执行模板或混合插件贡献。
- 前置：新版 Agent 声明，合法旧 Settings、MCP 和 Hook 包作为对照。
- 触发：发布、索引或安装编译声明，包括未知版本/字段、重复键、超限、非法默认值、混合 shape 和非 agent kind 携带 Agent shape。
- 必须成立：相同输入在各消费者得到相同有界结果；无 Settings 但有模型能力合法；连接/模型引用和普通标量分开。
- 禁止结果：未知字段被裁剪后接受、普通 apiKey 输入成为秘密接口、读取网络资源或执行表达式。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 新版严格编译和上限 | 共享向量经 Rust [`agent/tests.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/crates/plugin-config/src/agent/tests.rs) 与 Go [`agentconfiguration_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/internal/pluginmarket/agentconfiguration_test.go) 消费，覆盖未知版本、未知成员、重复键、默认值类型；Rust 另有 256 KiB 与 128/129 项边界。缺 Go 侧的 256 KiB/128 项边界，发布脚本不做完整编译 | Partial |
| 不混入凭据和执行权 | 向量拒绝个人引用、`secret` 类型、混合 MCP/Hook shape；[`personal_agent_configurations_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/integration/personal_agent_configurations_test.go) 经生产 HTTP 拒绝 Settings 里的 `apiKey` 与请求体 `userId`；[`install.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/crates/plugin-manager/src/install.rs) 覆盖 mcp 类携带 Agent shape 整包失败。缺发布接口拒绝秘密、`agent` 类混合声明与 hook/skill 等类携带 Agent shape 的包级用例，也没有命令或远程 schema 输入的专门用例 | Partial |
| 旧形态语义不变 | `agent/tests.rs` 钉死旧 v1 指纹算法不变、旧 Settings 与 MCP 仍走旧 shape，[`service.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/crates/plugin-config/src/service.rs) 钉死 legacy 指纹，既有 MCP/Hook 套件同批通过。新增测试里没有 Hook 旧 shape 的直接对照，依赖既有套件 | Covered |

## Published declarations must match the exact package target

- 风险：用户按一份 schema 保存，执行的是另一份包或 target。
- 前置：同 identifier 的不同来源、相同版本的不同 target，以及声明相同/不同的制品。
- 触发：生成副本、变更字节、替换包摘要、改 target、构造包内路径 escape。
- 必须成立：副本来自最终包内字节；关联包含来源、版本、target、包和声明摘要；文件校验在有上下文的消费者完成；配置指纹与完整性摘要是同一个原始字节 SHA-256，仅排版差异也产生新指纹。
- 禁止结果：按展示名匹配、仅比较版本、同义 JSON 被当相同字节、包中包含自己的 package SHA-256。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 发布与安装身份一致 | Desktop 安装校验与计划测试（[`install.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/crates/plugin-manager/src/install.rs)、`planned.rs`）、Node 经真实 zip 与生产安装器的 [`plugins.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/apps/ora-node/tests/plugins.rs)（摘要、重排版、复用已装版本、租约下旧包不动），Cloud 同步的摘要不符、副本缺失、声明无效。缺“发布工具真实产物 → Cloud 同步 → Node 安装”的端到端用例，篡改只覆盖字节、版本和缺失，未覆盖改来源或 target | Partial |
| 路径与字节边界 | 清单 `copy` 路径的 `..`、绝对路径用例，目录冒充副本，非法 UTF-8 与尾随 JSON。缺符号链接、超大副本和归档入口路径穿越的声明相关用例 | Partial |

## Unsupported hosts must not silently downgrade personal model access

- 风险：旧宿主忽略新增字段后直连模型或使用共享凭据。
- 前置：新旧清单 resolver、声明版本及 adapter 契约版本的混合组合；旧 Desktop（现网版本）、新 Desktop 与 Cloud/Node 宿主。
- 触发：在各类宿主上安装和启动要求个人模型绑定的插件。
- 必须成立：不支持的新能力明确拒绝；旧 Desktop 对新 shape 包按宿主表得到确定结果：`mcp`/`hook` 类整包失败、`agent` 等其余类安装成功但声明判无效、无法激活，不产生部分安装也不静默可用；新 Desktop 编译 Settings、模型能力无本地效果且本地执行资格不变；同一包在 Desktop 与 Cloud/Node 各自按决策 D3 的宿主表表现；无声明的旧插件不被补造能力；旧 Echo/历史执行不被要求新配置。
- 禁止结果：只用 dependencies 文案替代运行握手、声明失败当未声明、回退宿主登录或任意旧包、把整包失败弱化为忽略字段的安装成功、把激活拒绝弱化为静默可用。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 版本与握手闭合 | Controller 不向无匹配能力的 Node 派发（[`agents.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/apps/ora-controller/tests/workspaces/agents.rs)）、Cloud 能力不符在首轮 prompt 前结算、Node 对无法证明的配置不发授权也不启进程（[`agent_session.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/apps/ora-node/tests/agent_session.rs)）。缺 resolver × 声明版本 × adapter 版本的矩阵，Node `start` 的不支持分支无直接测试，Node 侧用 stub catalog | Partial |
| 宿主结果矩阵 | 新 Desktop 编译 Settings 且本地资格不变（`service.rs`、`install.rs`），Cloud/Node 见上行，mcp 类整包失败。缺旧 Desktop（无可钉住的旧版本）、hook 等类携带 Agent shape 的直接用例，以及同一包在 Desktop 与 Cloud/Node 的对照 | Partial |
| 历史兼容 | Node 无冻结配置时仅 OpenCode 走遗留路线、其余拒绝；OpenCode 迁移（[`personal_agent_migration_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/integration/personal_agent_migration_test.go)）。缺 Echo/无绑定历史执行的专门回归，`agent_declaration_required` 无测试 | Partial |
