# Agent 配置元数据的分发、版本与安装证据

决策依据：[静态元数据分发](../../../decisions/cloud/plugin-marketplace/20261017-agent-configuration-metadata.md)。
证据固定到 cloud `abc7b31`、desktop `8e497d7`。只有 Node 侧安装核对达到 `Covered`，离线发布一致性仍是 `Missing`，其余为 `Partial` 并写明缺口；没有浏览器端到端证据，也没有 Cloud→Controller→Node 的联合验收。

## Configuration reads must not require agent execution or package downloads

- 风险：首次配置依赖一个无凭据 Agent，或页面请求执行发布者代码。
- 前置：已同步合法/非法声明的 catalog，空间没有运行 Workspace。
- 触发：同步以及浏览、编辑插件个人配置。
- 必须成立：同步只解析受控静态副本；页面读取已持久快照；错误条目无可消费声明，其他条目可用。
- 禁止结果：启动插件、创建沙盒、模型探测、下载整包或通过声明引用任意网络资源。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 静态同步与安全文件读取 | [`catalog_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/internal/pluginmarket/catalog_test.go) 覆盖副本缺失、摘要不符、声明无效、目录冒充副本，[`agent_configuration_catalog_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/integration/agent_configuration_catalog_test.go) 在真实 git 仓库与 PostgreSQL 上同步。缺超 256 KiB、非 UTF-8、schema 版本不一致和符号链接用例；中间目录符号链接在实现里也尚未拒绝（见 ADR D1） | Partial |
| 页面无执行副作用 | 集成测试中包服务器在同步、目录读取、选择、派发全程收到 0 次请求；[`personal_agent_configurations_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/integration/personal_agent_configurations_test.go) 无 Workspace 经生产 HTTP 读写配置。缺对配置读写的进程观察与浏览器流程 | Partial |
| 源快照失败边界 | [`plugins_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/integration/plugins_test.go) 验证整源失败保留旧目录并记录同步错误（既有，非声明专属）；上述集成测试验证含恶意副本的坏条目不拖垮同源其他条目。缺同一条目由有效变无效的连续同步断言 | Partial |

## Selected releases must retain their own declaration after catalog updates

- 风险：catalog 最新 schema 覆盖空间已选择的旧包。
- 前置：两个来源或空间使用同名插件的不同版本/target，用户打开旧页面。
- 触发：同步新版本、管理员升级、旧页面保存；为升级前已排队的运行创建 Workspace 并安装插件。
- 必须成立：声明跟随选定发行快照；保存重新校验关联；排队运行的准备使用原冻结包/声明，无法获取原包则准备失败并收尾。
- 禁止结果：按名称/最新版本合并、丢失旧关联、自动清空字段、选相近 target 或把 Space 新期望装入旧运行冒充原绑定。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 发行与声明持久绑定 | 集成测试证明同步 2.0.0 后已选 1.0.0 的冻结行与摘要不变；[`personal_agent_configurations_db_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/internal/core/personal_agent_configurations_db_test.go) 证明排队后管理员升级，计划项仍是冻结的 1.0.0。缺“同步 → 选择 → 保存 → 创建运行 → 升级 → Workspace 安装”完整交错、原包不可得的准备失败分支，以及同名多来源或两空间不同版本的运行创建 | Partial |
| 旧编辑器的冲突反馈 | [`editor.test.tsx`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/frontend/src/features/agent-configurations/editor.test.tsx)（jsdom）覆盖 revision 冲突保留草稿与显式重新核对。缺服务端发行冲突与声明冲突两个码的任何测试、管理员升级的交错和真实浏览器 | Partial |

## Installation evidence must bind the actual package declaration

- 风险：Cloud 接受正确版本号却携带另一声明的安装结果。
- 前置：固定安装计划、正确包及被篡改的声明/target/结果。
- 触发：Node 安装、结果接管、重试和启动前校验。
- 必须成立：真实包验证声明关联，Cloud 检查结果属于原计划；失败不产生新执行资格。
- 禁止结果：元数据有效等于 installed、结果缺项/重复/错摘要仍成功、重试读取 latest。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 包内与派发关联一致 | Node [`plugins.rs`](https://github.com/ora-space/desktop/blob/8e497d7382441babd19595bfc0c0f61a7511fc31/apps/ora-node/tests/plugins.rs) 用真实 zip 经生产安装器：摘要、重排版、复用已装版本逐项篡改均拒绝，暂存目录为空，租约下旧包磁盘内容不变；Controller 与协议层的关联转发测试。仍无 Cloud→Controller→Node 联合验收，改 target 或身份的篡改靠通用安装测试支撑 | Covered |
| 结果与计划一致 | [`agent_run_boundary_test.go`](https://github.com/ora-space/cloud/blob/abc7b3112ea8c89d0a85b46041a6a5fbcbada561/integration/agent_run_boundary_test.go) 经生产接管接口与 PostgreSQL 验证重复、混合、错版本结果被拒且不留部分变更。这些用例用的是未声明的计划，缺带声明计划的版本，也缺声明计划下的结果缺项与重试沿用原计划 | Partial |
| 离线发布一致性 | 只有发布工具层的字节固定与回读测试（claude-code-agent、opencode-agent 的打包测试），不是离线产物的发布/安装验证；独立的离线产物机制尚未提供 | Missing |

## Space installation authority must not grant access to personal settings

- 风险：管理员用安装权限读取成员 Key，或用户配置把安装失败改成成功。
- 前置：同空间两用户、一个管理员；安装与个人配置分别处于不同状态。
- 触发：读取“我的配置”、保存个人连接、跨用户请求、安装/移除。
- 必须成立：共享安装和个人配置独立授权、独立展示；新建连接复用同一个人入口。
- 禁止结果：跨用户读写、catalog 存个人值、配置成功改变安装状态或反之。
- 验证义务与证据：HTTP+真实 PG 的跨角色矩阵及生产页面状态组合验证，当前 `Partial`：DB 层 `TestPersonalAgentConfigurationRepairAndOwnership` 证明租户管理员读不到成员配置、写成员连接得 `not_found`，单用户 HTTP 证明请求体 `userId` 被拒，前端页面测试证明成员可打开“我的配置”但仍不能安装或移除。缺同空间两用户加管理员的 HTTP 矩阵，缺“保存个人配置前后插件选择不变”与“安装成功但未配置仍被拒绝”的直接断言，也缺生产页面的状态组合。
