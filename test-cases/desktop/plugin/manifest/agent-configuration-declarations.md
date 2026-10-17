# Agent 静态声明与发布身份

决策依据：[Agent 配置声明](../../../../decisions/desktop/plugin/manifest/20261017-agent-configuration-declarations.md)。
新增义务状态均为 `Missing`：现有 v1/MCP/包校验测试是回归基线，不证明新 Agent shape 已实现。

## Agent declarations must be strict and free of personal values

- 风险：声明被解释为用户凭据、可执行模板或混合插件贡献。
- 前置：新版 Agent 声明，合法旧 Settings、MCP 和 Hook 包作为对照。
- 触发：发布、索引或安装编译声明，包括未知版本/字段、重复键、超限、非法默认值和混合 shape。
- 必须成立：相同输入在各消费者得到相同有界结果；无 Settings 但有模型能力合法；连接/模型引用和普通标量分开。
- 禁止结果：未知字段被裁剪后接受、普通 apiKey 输入成为秘密接口、读取网络资源或执行表达式。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 新版严格编译和上限 | 生产解析入口的跨语言有效/无效输入矩阵，覆盖 256 KiB/128 项边界 | Missing |
| 不混入凭据和执行权 | 发布与配置接口拒绝个人秘密、混合 contribution、命令/远程 schema 输入 | Missing |
| 旧形态语义不变 | 同批变更对旧 v1/MCP/Hook 生产入口的回归结果 | Missing |

## Published declarations must match the exact package target

- 风险：用户按一份 schema 保存，执行的是另一份包或 target。
- 前置：同 identifier 的不同来源、相同版本的不同 target，以及声明相同/不同的制品。
- 触发：生成副本、变更字节、替换包摘要、改 target、构造包内路径 escape。
- 必须成立：副本来自最终包内字节；关联包含来源、版本、target、包和声明摘要；文件校验在有上下文的消费者完成。
- 禁止结果：按展示名匹配、仅比较版本、同义 JSON 被当相同字节、包中包含自己的 package SHA-256。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 发布与安装身份一致 | 真实打包产物经生产索引/安装校验；逐项篡改关联后的拒绝 | Missing |
| 路径与字节边界 | 普通文件/符号链接/目录穿越/编码/大小用例，经实际文件和归档入口验证 | Missing |

## Unsupported hosts must not silently downgrade personal model access

- 风险：旧宿主忽略新增字段后直连模型或使用共享凭据。
- 前置：新旧清单 resolver、声明版本及 adapter 契约版本的混合组合。
- 触发：安装和启动要求个人模型绑定的插件。
- 必须成立：不支持的新能力明确拒绝；无声明的旧插件不被补造能力；旧 Echo/历史执行不被要求新配置。
- 禁止结果：只用 dependencies 文案替代运行握手、声明失败当未声明、回退宿主登录或任意旧包。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 版本与握手闭合 | 真实发布元数据、生产安装器和运行握手的版本矩阵，拒绝发生在模型调用之前 | Missing |
| 历史兼容 | 无绑定历史/Echo 与明确 OpenCode 迁移路径的独立回归 | Missing |
