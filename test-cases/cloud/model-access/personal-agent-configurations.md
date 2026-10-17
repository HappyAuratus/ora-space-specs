# 个人 Agent 配置、冻结与授权

决策依据：[个人 Agent 配置](../../../decisions/cloud/model-access/20261017-personal-agent-configurations.md)。
以下新增验证义务均为 `Missing`。既有 OpenCode/个人连接的 Covered/Partial 证据保持原范围，
不能推导 Claude、新 schema 或 adapter 路线已实现；供应商确认不是 Ora 实测证据。

## Plugin configuration must reuse owner-scoped connections without exposing keys

- 风险：插件另建 Key 库，或空间管理员继承他人个人配置权。
- 前置：两用户同空间、管理员、同名不同来源插件，以及可被两插件选择的连接。
- 触发：网页内联创建/选择连接、配置读写、Key 写入和跨用户重试。
- 必须成立：所有连接来自同一用户库；Settings 与秘密接口分开；读取只给允许字段和已配置状态。
- 禁止结果：请求体决定归属、普通 Setting 存 Key、管理员读取个人值、删除插件导致共用连接被删。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 用户/来源隔离 | 真实 PG + HTTP 的跨用户、管理员、canonical ID 与幂等矩阵 | Missing |
| 网页秘密通路 | 生产页面和 Gateway/model-gateway 联合验证，含前端缓存/错误/日志不留 Key | Missing |
| 连接单一事实来源 | 两插件共用连接、一次轮换/删除与各引用状态的集成验证 | Missing |

## Explicit Claude selection must not change or inherit the global default

- 风险：Claude 误用 OpenCode default，或两个默认记录漂移。
- 前置：已存在 OpenCode 默认值、不同 Claude 配置和工作流 default 槽位。
- 触发：分别编辑两种插件配置并从普通任务与原工作流入口创建运行；包括 Claude 缺配置。
- 必须成立：OpenCode 插件页和全局默认读写同一选择；Claude 独立解析；缺配置明确拒绝。
- 禁止结果：复制 default、切换全局 CLI、缺配置回退其他人/槽位/宿主凭据。
- 验证义务与证据：三入口/默认值读写的生产 HTTP + PG 和浏览器行为测试，当前 `Missing`。

## Run creation must freeze the complete execution configuration atomically

- 风险：配置编辑或 catalog 更新改变排队运行，失败留下半个运行。
- 前置：合法发行声明和个人配置；并发升级包、编辑 Settings/模型、替换 Key。
- 触发：创建运行、延迟派发、准备重试；提交错误版本/协议/目标/引用。
- 必须成立：发起者、包/target/摘要、声明、adapter 版本、选项 revision 与模型绑定一次冻结；失败原子回滚。
- 禁止结果：客户端自报快照被信任、旧运行重读默认值、事务中访问网络、配置失败仍有可派发运行。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 并发冻结与回滚 | 真实事务交错测试核对评论/交互/Run/绑定及实际派发内容 | Missing |
| 版本及能力拒绝 | 生产准入接口的协议/target/声明/安装准备矩阵 | Missing |
| 共享展示脱敏 | Thread/运行详情 HTTP 只含允许元数据，不泄露个人选项、上游地址或凭据引用 | Missing |

## Editing and revocation must have distinct effects on existing runs

- 风险：普通配置编辑热切换，或撤销因 DNS 失败/重新启用失效。
- 前置：运行处于 pending/active/idle，并持有可持续流式的授权。
- 触发：编辑选项、改模型、替换 Key、清插件选项、清 Key、停用/删除连接、失去成员资格、结束。
- 必须成立：普通编辑只影响新运行；真正撤销拒绝后续和在途访问；纯降权不依赖出网可用。
- 禁止结果：复活旧绑定、将配置清除误作共用连接删除、回退其他凭据、静默 provider 切换。
- 验证义务与证据：PG 授权状态 + 真实受控 HTTP/SSE、DNS 不可用与 Node 续期的联合矩阵，当前 `Missing`。

## Upgrades must preserve history and require explicit repair of changed declarations

- 风险：旧编辑器覆盖新配置，升级迁移改写历史输入或丢值。
- 前置：既有 OpenCode 用户/历史运行、旧插件无声明、新包同指纹或不同指纹。
- 触发：数据库升级、旧页面保存、卸载重装、来源或 target 改变、新旧运行分别启动。
- 必须成立：旧连接/默认/Key 引用不复制；历史不回填；冲突保留值/草稿并拒绝；新值只用于新 run。
- 禁止结果：插件迁移脚本在 Cloud 执行、未知字段静默删除、旧包被原地修改、旧绑定升级为新语义。
- 验证义务与证据：真实新建/升级数据库、部署混合版本和浏览器修复路径测试，当前 `Missing`。

## Claude model aliases must resolve to the single authorized model

- 风险：辅助或深度调用隐式访问第二模型，兼容标签误判协议。
- 前置：一个 Messages 连接和明确模型；另有 Completions 连接、未授权模型和主/辅助别名配置。
- 触发：启动 Claude、调用主/辅助路径、会话切模型及越权请求。
- 必须成立：实际每个推理请求都使用冻结模型；不兼容配置启动前拒绝；不能满足单模的固定版本不开放。
- 禁止结果：网关任意改写模型、授权第二模型、以同名供应商判断协议或回退直连。
- 验证义务与证据：固定 CLI/adapter + 受控网关对真实请求的模型/协议核验及拒绝矩阵，当前 `Missing`。

## Real Claude runs must complete with personal credentials and released resources

- 风险：只验证表单或供应商声明，实际 CLI、成果与撤销未贯通。
- 前置：无部署共享 Key/宿主登录态的真实栈，固定插件/CLI/adapter，用户经网页设置个人连接。
- 触发：真实两轮任务和工具操作、正常结束；独立验证取消/失败及授权撤销。
- 必须成立：同一 Run 的流式/工具调用、成果/Git 身份、历史和 Workspace 释放可核验，敏感材料不进入成果。
- 禁止结果：以模拟器/HTTP 200/另一 Run 替代、把上游 Messages 确认当端到端通过、把正常退出当撤销证据。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 受限协议兼容 | 明确路径/方法/query/头的合成供应商测试，覆盖未知项拒绝、SSE、工具数据和不自动重发 | Missing |
| 真实网页正常链路 | 固定版本、同组资源的两轮/成果/身份/释放证据，实际 Key 经网页受控入口写入 | Missing |
| 隔离与收尾 | 双用户并发、错误/取消/撤销、临时令牌及真实 Key 的安全排除核验 | Missing |
| 旧 OpenCode 回归 | 升级后的两协议默认模型与真实任务回归，不仅引用升级前记录 | Missing |
