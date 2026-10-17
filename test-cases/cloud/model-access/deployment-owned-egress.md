# 部署方共享出网：模型转发、资产下载、解析与策略切换

决策依据：[部署方出口与显式解析 ADR](../../../decisions/cloud/model-access/20261017-deployment-owned-egress-and-explicit-resolution.md)
（`approved`）。以下都是新增/改变的验证义务，当前没有对应实现的直接证据，统一为 `Missing`。
表中的场景是验收要求，不是已经执行的测试。原公网、凭据、冻结和撤销的已有证据继续见
[个人模型访问](personal-model-access.md)；这些证据不能替代企业策略的组合验证。

## Public deployments must retain the existing HTTPS and trusted DNS policy

- 风险：企业扩展悄悄改变默认公网规则或启用系统解析。
- 前置状态：未配置策略路径，或显式 `mode: public`，已有公网连接、有效 grant 及 HTTPS 工作流资产源。
- 触发：保存连接、两协议请求、资产取包和解析故障。
- 必须成立：原公网 HTTPS/DoH 与数字 bootstrap 保持；全部答案校验、数字拨号、原 hostname TLS、无代理和无重定向。
- 禁止结果：默认允许私网、系统/fake-IP 回退，或未启用 enterprise 却要求用户白名单。
- 决策依据：D1、D4、D5。
- 验证义务与证据：`Missing`；需要无配置与 public 配置两条真实保存/受控 TLS 转发及资产下载回归，两调用方使用同一部署 DoH 配置，双栈混合保留地址与 DoH 故障均拒绝。

## Enterprise targets must match an exact canonical host and effective port

- 风险：仅在拨号检查，或 URL/别名/端口规范化差异绕过许可。
- 前置状态：enterprise 仅允许 `model.example.invalid:443`；已有一条不在新许可内的历史冻结绑定。
- 触发：保存/更新连接及使用历史绑定请求；测试无凭据的受控域名解析。
- 必须成立：大小写、单末尾根点与默认/显式 443 等价；同 authority 不同 base path 可用；需要准入的保存解析失败不提交新版本；每次请求重新检查。
- 禁止结果：通配/后缀/别名匹配、8443 自动放行、userinfo/query/fragment 或数字 IP literal 绕过；在 SQL 事务中等待 DNS。
- 决策依据：D1、D2、D5。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 共享规范化和精确匹配 | 配置与保存/转发边界覆盖根点、端口、Unicode、畸形 host、IP literal 与不同路径 | Missing |
| 未列出的所有目标均拒绝 | 未许可 host 的公网（含 `7.x` 借用公网编号夹具）、RFC1918/ULA；断言 DNS/上游零调用 | Missing |
| 保存与转发独立且一致 | 真实保存接口、版本/幂等冲突、DNS 失败不提交；旧绑定不能绕过收紧许可 | Missing |

## Revocation must remain available when egress admission is unavailable

- 风险：DNS 或策略拒绝阻止停用，导致旧绑定不能被永久撤销；纯停用成为修改目标的旁路。
- 前置状态：连接有有效绑定，之后 host 被移出许可、地址进入禁区、DNS 故障或 D6 调用入口关闭。
- 触发：分别提交纯 `enabled=false`、删除连接、清除 Key，以及重试、并发冲突和重新启用。
- 必须成立：身份/归属/资源版本/幂等约束继续校验；纯停用接受原样历史配置且零 DNS/上游调用；三种降低权限操作均持久撤销绑定；重新启用重新准入且旧绑定永久失效。
- 禁止结果：因旧 URL 不再符合准入而拒绝纯停用、关闭调用时一并关闭撤销、借停用附带改 URL/模型、幂等重放重新执行出口检查或复活旧授权。
- 决策依据：D2、D6、不变量 6、7；根决策 D1–D4。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 降低权限独立于准入 | 真实 HTTP/PostgreSQL/Gateway 覆盖三种操作与四种不可准入状态，断言永久撤销且 DNS/上游零调用 | Missing |
| 纯停用只降低权限 | 历史 URL 原样可停用，夹带修改配置拒绝，跨用户/版本冲突仍拒绝；启用后旧绑定继续无效 | Missing |
| 幂等不受之后网络变化影响 | 保存成功后改变 DNS/策略，合法相同重试返回原结果；原有幂等冲突仍成立 | Missing |

## Private exceptions must never override infrastructure and special address denies

- 风险：白名单或宽 CIDR 使模型代理访问 cluster 数据库、宿主或元数据服务。
- 前置状态：同 target 分别省略/开启 `allowPrivate`，可选限定 CIDR；完整双栈 topology。
- 触发：受控 DNS 或静态映射返回私网、特殊地址或混合 A/AAAA。
- 必须成立：私网默认拒绝；显式允许后仍逐地址检查目标 CIDR、绝对禁区及保留类别；mapped IPv6 同义。
- 禁止结果：跨 target 合并 CIDR、只检查首个答案、跳过坏答案、只支持 IPv4 的宿主拒绝。
- 决策依据：D1、D3。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 私网许可与可选 CIDR 独立 | RFC1918/ULA 的缺省拒绝、显式通过、CIDR 缺省和越界；另一个 target 许可无效 | Missing |
| 不可覆盖拒绝 | 两调用方在白名单+allowPrivate+覆盖 CIDR 下，cluster 网络、宿主接口/网关/VIP、loopback、link-local 仍拒绝 | Missing |
| 其他保留类别和全答案检查 | unspecified/multicast/documentation/benchmark/转换地址，A 允许+AAAA 拒绝及反向组合，mapped IPv6 | Missing |
| 拓扑来源完整 | Docker 静态/动态池、默认桥、多宿主/双栈及声明 NAT 合并；缺来源/空表/未知池时启动不 ready | Missing |

## Enterprise resolution must use only the configured resolver or static mapping

- 风险：隐藏的系统、公网回退或 CNAME/附加记录绕过地址检查。
- 前置状态：显式 TCP resolver 数字地址，或 target 非空 staticAddresses；系统 DNS 返回一个可计数诱饵。
- 触发：A/AAAA 查询、TCP 超时、SERVFAIL、NODATA、畸形响应、CNAME 链和静态映射请求。
- 必须成立：动态查询只发往显式 resolver，按原 target 校验最终全部地址；一族 NODATA 可接受，错误不可忽略；静态映射零 DNS。
- 禁止结果：DNS 失败改走系统/公网、静态失败改动态、DNS 控制通路放开同地址模型 HTTP，或从 additional 区取拨号地址。
- 决策依据：D3、D4。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 显式解析及无回退 | 真实受控 TCP DNS 服务与系统诱饵计数，超时/失败/单族错误拒绝，NODATA 语义 | Missing |
| 有界协议校验 | CNAME 深度/循环、响应关联、超长/畸形/附加记录；地址验证发生在任何 dial 前 | Missing |
| 静态映射与 TLS 身份 | 无 resolver 的全静态配置、混合配置、静态全地址检查；HTTP Host/SNI/证书仍为原域名 | Missing |

## Verified dialing and connection reuse must preserve the current target boundary

- 风险：DNS 重绑定、长连接、HTTP/2 合并、代理或重定向绕过已经通过的检查。
- 前置状态：真实受控双栈 DNS/TLS 上游，有效授权，已建立 HTTP/1 或 HTTP/2 连接。
- 触发：新请求、新拨号、TTL 到期地址变化、证书错误、3xx 和环境代理配置。
- 必须成立：完整验证后只拨数字地址；每请求准入；TTL 到期先重验，peer 不在新集合或新集合含禁区时不复用；原 hostname TLS 不变。
- 禁止结果：验证后系统再次解析、跨 authority/策略池复用、TLS 降级、跟随跳转、向环境代理发送认证头。
- 决策依据：D3、D5。
- 验证义务与证据：`Missing`；需计数真实 DNS/dial/HTTP 边界，覆盖新 dial、keep-alive、HTTP/2、TTL 变更、错误证书、3xx、HTTP/SOCKS 代理与有限脱敏错误；纯匹配函数单测不够。

## Policy activation must fence old streams pools and inconsistent services

- 风险：新策略生效后旧 Gateway/池仍发送，或应用失败静默恢复宽许可。
- 前置状态：旧快照已加载且存在活跃 SSE、HTTPS 资产下载与空闲连接；新快照移除目标或收紧 CIDR/topology。
- 触发：预检失败、仅改文件、单服务重启、摘要不一致、受控应用成功/失败及显式回滚。
- 必须成立：文件变动不热生效；预检失败留旧当前策略；切换由 handler 调用开关封入口，旧流/全部池停止；两端当前启动实例确认同代激活后新策略才生效；启动失败保持关闭，纯停用/撤销仍可用。
- 禁止结果：新旧策略混合开放、healthz 非 200 但直连仍发送、只关 idle 池、旧实例/旧激活重放开放、自动回退 public/旧配置、自动重发中断的生成。
- 决策依据：D1、D3、D6。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 严格 schema 与部署快照一致 | 缺文件、版本/字段/重复键、空 CIDR、冲突开关、摘要不一致、缺 topology 的双服务启动 | Missing |
| 受控切换全链路 | 部署入口实际停旧 Gateway 并取消 SSE/资产下载/两端连接池，故障注入证明入口关闭；未完整取包的任务包级失败且零文档结果，成功后旧地址零调用 | Missing |
| handler 与资产入口直接拒绝 | 在启动 closed、摘要不一致、半激活、激活失败时，使用有效旧 grant 直连 runtime、请求 grant 新建/续期、复用连接及触发后台资产 GET；模型为安全 503、资产为包级失败，无业务 DNS/上游调用，纯停用/撤销成功 | Missing |
| 单端重启与确认失败 | 分别自动重启 Cloud/Gateway；新实例保持 closed，对端下次模型请求或资产 GET 识别新实例/超时并关闭、取消两类在途及清池；重新受控激活前上游零新增调用 | Missing |
| 激活身份与准备状态 | 用户/Node 调管理通道及重放旧代次/旧启动实例激活均失败；控制预检可读 prepared，半激活不能伪报模型 ready，双端确认后才可调用 | Missing |
| 回滚与拓扑变更 | 显式完整快照回滚、新增宿主/动态池先封出口并重新应用；禁止隐含旧许可复活 | Missing |

## Current policy must constrain frozen bindings without reviving revoked grants

- 风险：冻结被误读为永久网络许可，或策略重新允许后复活已撤销凭据。
- 前置状态：有效冻结绑定和另一条已永久撤销的绑定；之后修改个人 URL/Key/槽位或收紧部署策略。
- 触发：旧运行请求、清除/停用/删除连接，再重新允许目标和启用连接。
- 必须成立：有效旧绑定仍用原 URL/模型/Key 并受当前出口限制；准入失败不改写绑定；恢复许可只适用于仍有效的授权；永久撤销永不复活。
- 禁止结果：旧运行自动换新 Key/URL/槽位、把临时网络拒绝误写为永久撤销、部署配置读取或注入真实 Key 给 Cloud/Node。
- 决策依据：D5、D6；根决策 D1–D4 与槽位后续 ADR。
- 验证义务与证据：`Missing`（企业策略组合）；需真实数据库+Gateway 贯通修改/撤销/恢复许可矩阵、原绑定字段核对和解密边界断言，不能只引用原永久撤销数据库测试。

## Model forwarding and asset downloads must share one deployment policy

- 风险：资产下载器保留硬编码公网 DoH/独立白名单而无法取内网包，或借签名 query、对象存储/池复用绕过共同准入。
- 前置状态：Cloud 与 Gateway 加载同一 `deployment-egress.json` 并激活；受控 HTTPS 模型与资产源返回双栈地址，目录含带签名 query 的合法 `.orax` URL 和 object key。
- 触发：模型 POST 与后台资产 GET 分别访问白名单内/外目标，更换部署 resolver/静态映射，返回混合禁区地址、3xx、错误证书或超限包，并在取包期间应用新策略。
- 必须成立：同一 host:port 集合、目标私网/CIDR、DNS 与拓扑规则用于两者；资产原路径/签名 query 被保留，模型 baseUrl 仍拒绝 query；无个人模型认证头进入资产 GET。资产大小/摘要/事务与包级失败语义保持，切换关闭同时停止两类出网。
- 禁止结果：只有模型遵守策略、资产默认放行未许可公网/7.x、使用系统或写死 Cloudflare DNS、忽略 AAAA 坏答案、cluster HTTPS 源特例、把拒绝的 HTTPS 改为 object-key 取包、泄漏 query/Key 或取消后留下部分导入。
- 决策依据：D1、D3–D7、不变量 1、5、8、9；[工作流资产 ADR D3–D5](../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)。
- 验证义务与证据：

| 义务 | 直接证据要求 | 状态 |
| --- | --- | --- |
| 两者使用同一集合与解析来源 | 同一部署快照的真实 Cloud Fetch/Gateway HTTP 贯通：许可内私网成功，未许可公网（含 7.x）/私网/错误端口拒绝；更换一个 resolver/静态映射同时影响两者，记录 DNS 目标而非只测共享函数 | Missing |
| 全答案与不可覆盖拒绝 | 两入口分别覆盖 A/AAAA 混合禁区、cluster/宿主/loopback/link-local、TTL 变化；新池与复用池均无禁止外连 | Missing |
| URL 与凭据边界不同但准入一致 | 合法签名 query 下载成功、模型 query 拒绝；服务端断言原路径/query 且资产 GET 无模型认证头，日志/失败信息不含签名 URL；别名/重定向/错误证书均拒绝 | Missing |
| 资产原完整性与失败边界 | 真实 Fetch+PostgreSQL 验证 16 MiB/10–30–60 秒、摘要失败先于解析、包级失败零文档结果及下次触发重试；导入 revision 围栏不被策略切换绕过 | Missing |
| 内部对象存储不是 HTTPS 旁路 | object key 只走固定私有客户端且不能携带 host；同一 cluster 服务若作为 HTTPS URL 则被拒，无自动降级 | Missing |
| 共同策略切换与激活 | 后台资产任务直接经过 closed/对端确认检查，移除任一 target 后两调用方旧池均不可用；取消未完整取包的任务无部分导入，旧 grant/单端重启不能开放模型或资产出口 | Missing |
