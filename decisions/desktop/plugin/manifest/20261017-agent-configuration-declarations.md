---
status: approved
---

# Agent 配置以包内版本化静态声明发布，声明身份与运行能力分别校验

Agent 插件在包内声明可编辑 Settings 和个人模型访问能力，发布工具从同一份字节生成市场元数据。
宿主负责校验和编辑，插件代码负责 CLI 适配；声明不保存用户值，不授予凭据、进程或网络权限。
本决策为 `approved`，对应能力尚待实现，新增核心验证义务均不因文档存在而视为完成。

范围是 Agent 的声明格式、身份和版本兼容。Cloud 的分发与实例绑定见
[配置元数据分发](../../../cloud/plugin-marketplace/20261017-agent-configuration-metadata.md)，
用户归属见[个人 Agent 配置](../../../cloud/model-access/20261017-personal-agent-configurations.md)，
运行输入见[插件模型 adapter](../../../node/agent-runtime/20261017-plugin-owned-model-adapters.md)。
不改 MCP、Hook、Skill 的配置交付或 Desktop 用户值存储，不增加新的插件 kind。

## 继承与修改

本链前序 [pack 清单决策](20260915-pack-manifest-and-marketplace-visibility.md) 已整体被
[Extension Pack](../extension-pack/0-extension-pack.md) 取代；本文继承其现行替代者，不恢复旧 pack 规则。

| 前序约定 | 本文处理 |
| --- | --- |
| [清单根决策](0-versioned-dual-form-manifest.md) D1–D3、D7：双形态、版本、来源身份、纯解析 | 继承；外部副本及文件摘要校验属于发布/索引/安装，不在 TOML 纯解析器做 I/O |
| [图标决策](20260909-plugin-logo-as-themed-asset.md)：图标失败按缺失降级 | 继承图标规则；模型配置声明不是装饰资产，不能套用图标的静默降级 |
| 根决策 D4–D5：kind、target 和包身份 | 继承；仅为 agent 的个人模型路线增加声明与适配契约要求 |
| [MCP 配置根决策](../category/mcp/0-static-mcp-configuration.md) D2–D3：严格 config shape、Settings、旧 v1 | 继承旧形态；新增独立版本的 Agent shape，不能向旧 v1 塞入未知字段 |
| 根决策不拥有 config 文件内容 | 修改范围：本文拥有 Agent shape 及其发布关联；MCP/Hook 等 shape 仍由各自领域拥有 |

## D1：包内声明是一份事实来源

个人模型路线的 Agent 使用 `assets/config.json` 承载静态声明。它包含以下领域信息；表中名称描述
语义，跨语言序列化字段由共享契约定义，不能由 Cloud、Node 和发布工具各自发明一份解释。

| 信息 | 规则 |
| --- | --- |
| 声明版本 | 显式版本，区别于旧 Settings/MCP/Hook v1；未知版本拒绝，不按“没有声明”处理 |
| Settings | 可省略；存在时复用已有 Setting ID、标题、说明、类型、必填、默认值和顺序语义 |
| 模型访问能力 | 显式要求平台个人模型绑定，列明可消费的模型协议及 adapter 契约版本 |
| 单模型约束 | 首期一个连接与一个明确模型；辅助别名不得成为第二组隐含授权 |

首期 Settings 保留 string、number、boolean 的受限标量，不增加脚本、原生 CLI 文件导入、动态 URL
数据源或自定义页面代码。连接与模型引用由宿主的专用选择控件表达，不是可填写任意用户 ID、URL 或
认证头的普通 Setting。模型协议使用平台契约的精确身份，不能用供应商名猜测协议。

缺少 Settings 的声明仍可合法要求个人模型绑定；不要求作者为展示连接选择器制造一个空 Settings。
Agent shape 不同时携带 MCP transport 或 Hook contribution。未知字段、重复键、非法类型、非法默认
值、无效引用均拒绝；沿用声明不超过 256 KiB、Settings 不超过 128 项的有界解析要求。

声明只规定作者允许的输入。完整 canonical Plugin ID 的 namespace 来自宿主的 Marketplace Source，
不能由声明自报；声明不包含某个用户的连接 ID、真实 Key、临时令牌或预置用户设置。

## D2：发行关联绑定字节，配置指纹不等于插件身份

发布工具从最终包内声明生成静态副本，以该文件原始 UTF-8 字节的 SHA-256 作为声明摘要。副本和包内
文件必须逐字节一致；不得由市场维护者另手写一份同义 JSON。新的个人 Agent 配置以此摘要作为声明
指纹，旧 v1 的指纹算法不改写。重新排版会改变新声明指纹，发布者应避免无语义的声明重打包。

发布关联至少绑定 canonical Plugin ID 的来源/identifier、精确 package version、universal 或精确
target、package SHA-256、声明版本、声明 SHA-256 和静态副本定位。包内声明不包含包自身的摘要，
避免自引用；包摘要及来源关联由发布侧和宿主持有。多个 target 可以共用相同声明字节，但不能仅因
版本号相同就把一个 target 的声明拼到另一个包上。当前目标的声明不同，既有配置须按新指纹重新确认。

`orax.toml` 的 release 形态携带发布关联，installed 形态仍不携带市场下载定位符。文件内容、
归档内普通文件/路径 containment 和摘要一致性由有上下文的消费者校验；清单解析仍是纯函数。
声明定位限定在受控发布条目内，不允许 schema 自己引用网络上的任意资源。

摘要只证明来源关联和字节一致，不构成新增发布者签名或安全背书；既有源信任、包校验和 target 规则
继续适用。模型能力声明也不是模型在线、配置完整或 adapter 已执行成功的证据。

## D3：旧宿主、旧插件和新声明必须有确定的能力结果

清单 resolver、声明版本、adapter 契约版本是三个不同的版本域。不得仅增加旧 resolver 会忽略的
可选字段，就认为旧宿主会执行新的安全规则。发布/安装兼容性检查和运行握手必须共同拒绝不支持的
个人模型路线；声明要求的能力缺失时不得降级到部署 Key、宿主登录态或 Agent 的原生默认 provider。

旧插件无新声明时，保留已有安装/展示含义，但不因此获得新个人配置资格；没有模型绑定的历史执行
和 Echo 不被解释成待补个人配置。既有 OpenCode 的兼容迁移由
[个人配置决策 D5](../../../cloud/model-access/20261017-personal-agent-configurations.md#d5迁移保留旧默认值和历史运行)
限定，不推广成任意插件的隐式回退。

同一身份、版本和 target 的声明/包关联不能由缓存静默替换。升级后的页面须携带其声明指纹保存；
仍持旧指纹的提交失败并保留草稿，不能先删除未知字段再宣称校验成功。支持新声明不改变旧 MCP/Hook
解析、脱敏及生效规则，也不把其本地 store.json 用作 Cloud 个人配置库。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 插件启动后返回 schema | 能动态生成，但配置入口会依赖代码执行、Workspace 和尚未提供的凭据 |
| 市场单独维护 schema | 无需发布工具变更，但两份声明会漂移，无法证明用户编辑的是实际包的契约 |
| 只用版本号或最新 catalog 字段 | 关联短，但遗漏来源、target、包摘要；不能防止旧运行读到新声明 |
| 普通 Settings 加 apiKey 字符串 | 实现简单，但把个人秘密送入普通配置值、返回和存储路径 |
| 全功能模板/JSON Schema 执行环境 | 表达力更强，但扩大求值、下载和兼容面；CLI 行为应留在代码 adapter |

## 不变量

1. 一个 Agent 发布目标只有一个可核验的声明身份，页面副本与包内字节一致。
2. Settings 声明不包含用户值或授予执行权限；来源 namespace 不由插件自报。
3. 不支持的声明/adapter 不能通过忽略字段变成可执行个人模型能力。
4. 新能力不重解释旧 v1、旧用户存储或其他插件 kind 的贡献。

## 验收与落地

先交付共享声明/版本校验及发布工具，再接 Cloud 索引和 Node 安装/握手；消费者都支持之前不能把
新包标为个人模型执行可用。跨语言以同组有效/无效声明和摘要样例验证，不复制另一套业务规则。
核心义务见[Agent 声明](../../../../test-cases/desktop/plugin/manifest/agent-configuration-declarations.md)。
具体 wire 字段和方法由产品契约生成链实现；扩展 Settings 类型、多角色模型或其他 kind 配置不在本期。
