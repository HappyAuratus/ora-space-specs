---
status: approved
---

# 运行时独占必须贯穿 Cloud、Controller 与 Node，控制身份和旧执行隔离共同验收

Cloud 的独占状态必须成为真实执行入口的约束：Controller 只派发当前被批准并持久登记的工作，Node 在执行入口核验控制绑定，关闭旧入口并确认未完成执行后才允许交接。控制代次是单调递增的操作资格版本；拒绝旧代次新写入称为 fencing，本文使用“旧操作资格隔离”说明其含义。

本 ADR 于 2026-09-28 经用户明确批准，状态为 `approved`，整体实施与验收尚未完成；是本批跨仓库协议与交付义务的权威定义。业务权限与独占状态分别由[使用权限](../workspace/20260927-runtime-workspace-use-permissions.md)、[独占准入](../execution-admission/20260927-exclusive-runtime-control.md)拥有，本文不复制角色策略。公开字段与协议示例表达已批准语义，尚不是已发布接口。核心用例见[真实执行约束与恢复](../../../test-cases/cloud/controller-integration/fenced-runtime-control-delivery.md)，新保障的直接证据按核心用例逐项登记为 `Partial` / `Missing`。

## 现状与继承

本叶子承接[Cloud gRPC 根契约](0-cloud-owned-internal-grpc-contract.md)、[Workspace 操作与 Node 代报](20260926-workspace-operations-and-controller-reported-nodes.md)和[存活沙盒查询](20260926-list-live-sandboxes.md)，保留[保活与 Watch 规则](20260924-control-listener-keepalive.md)。批准时同步 specs 上游 `e2c8dce`，继承[Controller 过渡 JSON 接口与监听器退役](../../controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md)及[minicloud 退役](../../minicloud/runtime/20260928-retire-minicloud.md)：Controller 继续只拨出，不恢复监听；真实云端端到端验收使用 cluster Compose 链路，不使用已退役入口。原代码链接为最初核查基线，开发前仍需检查最新实现。

| 已核查事实 | 当前证据与限制 |
| --- | --- |
| Cloud gRPC holder 为调用者声明的 ControllerId | [server.go](https://github.com/ora-space/cloud/blob/67dcbf95b408559acd5680cf36e84f30e4caad4a/internal/controlgrpc/server.go)明确没有认证；私网/loopback 是现有部署边界，metadata 不是服务身份验证 |
| Rust Controller 已执行 sandbox、Node、clone、quiesce 与 terminate 协调 | [operations/steps.rs](https://github.com/ora-space/desktop/blob/652af715cbca5f50c892299ea815a08896f41b35/apps/ora-controller/src/cloud/operations/steps.rs)；不是 Cloud 模拟器等同于生产 |
| Node 有 Controller 会话与 clone/worktree 消息 | [message.rs](https://github.com/ora-space/desktop/blob/652af715cbca5f50c892299ea815a08896f41b35/crates/node-protocol/src/message.rs)、[session.rs](https://github.com/ora-space/desktop/blob/652af715cbca5f50c892299ea815a08896f41b35/apps/ora-node/src/service/session.rs)；单 Controller 连接不等于单用户写者，尚无本决策的用户控制代次 |
| 本机进程宿主有持久宿主代次 | [进程控制 ADR](../../node/process/control/0-authorized-generations-and-leases.md)仅批准可信本机会话控制，不是 Controller 授权、用户租约或旧业务进程已结束证明 |
| Substrate 有容器与持久卷生命周期约定 | [cluster ADR](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md)；此次未核查 cluster 实现，不能登记为真实边界验收 |

| 前序约定 | 本决策处理 |
| --- | --- |
| Cloud 是云端业务权威，Controller 不访问 PostgreSQL | 继承，不变 |
| proto 属于 Cloud，消费方锁定 commit 并生成客户端 | 继承；新增破坏性要求使用并列版本或明确拒绝旧能力，不能静默重解释 v1 |
| 持久登记后派发，同次提交与执行重试保留身份 | 继承，不变 |
| Node 事件在 Cloud 接管后才确认 | 继承，不变；撤权后仍可回报既有事实 |
| 仅自报 ControllerId、部署内网保护 | 修改：云端新控制能力要求可验证的服务身份与范围绑定 |
| Node 会话与宿主代次隔离旧连接 | 继承，但增加独立的运行时控制绑定；不将旧机制重命名为用户独占 |

## 决策

### D1：各代次表达不同事实

| 标识 | 约束对象 | 不能替代的事实 |
| --- | --- | --- |
| Controller lease epoch | 当前协调者对 Cloud 的写资格 | 谁获准编辑运行时、旧 Node 进程是否退出 |
| Runtime generation 与 sandbox 身份 | 当前运行实例及其资源 | 同一实例内的操作会话切换 |
| Runtime control epoch | 某运行时当前用户或维护会话的操作资格 | 身份认证、Controller 租约、任务结果 |
| execution ID / effect ID | 一次执行责任或外部计划的稳定身份 | 新一轮授权；重试不得因此重建身份 |
| 宿主代次 | 本地 host 与 guardian 的管理绑定 | 用户权限、租约期限和业务任务已终止 |

Controller 消费 Cloud 已限定范围的工作，不复制租户角色判断，也不解析终端用户 token。内部输入只携带执行必需的资源、控制与执行绑定；用户身份及授权决定留在 Cloud。Node 是 Rust 执行程序，不是 JavaScript 的 Node.js。

### D2：控制通道验证服务身份，工作负载不能伪造管理消息

云端 Cloud–Controller 与网络 Controller–Node 管理通道使用部署提供的双向 TLS 身份验证，将可信服务身份绑定到获准的 Controller/Node 及目标运行实例。声明的 ControllerId、NodeId 只作绑定和关联，不能自行建立信任。服务凭据通过部署引用加载，不进入前端、Git、业务表或日志；轮换后重新认证，失效身份不能建立新控制会话。

此项已随 ADR 批准，明确修改当前私网自报身份边界；批准不表示已有实现满足该要求。浏览器不直接连接 Node 或 Substrate。Substrate 仍需独立受限服务身份及 effect/目标范围校验。

执行用户代码的身份不得读取管理凭据、写控制代次日志或调用管理端点。云端部署应将可信管理程序与工作负载分成不同的操作系统权限域，控制数据放在工作负载不可写目录；网络可达性也须限制。若现有 Node home 布局或同用户进程模型不满足，则该执行能力不能按本决策开放，必须交付相应部署隔离。

这不恢复此前延期的整套本机秘密令牌、guardian 租约或跨机器宿主授权方案；可信 Node 到本机 host 的既有信任边界不因本文自动改变。部署管理员、宿主 root 和已被攻陷的管理程序不在此业务权限防护范围内，但普通运行时成员不能被默认视为管理程序。

### D3：派发与撤回共同定义准入顺序

1. Cloud 在短事务中核验成员、使用权限、独占绑定、运行代次、写活动互斥及凭据可用性，持久保存执行身份与固定输入，再允许派发。
2. Controller 在派发新动作前从 Cloud 取得当前许可，不能凭过期快照或缓存续权。Cloud 不可用、资格已撤回或结果未知时不派发新动作；既有执行仍按原身份查询和回报。
3. Node 在本地接受队列与实际变更入口核验目标实例、控制代次和执行身份；持久保存接受责任后才能执行。同执行同输入重传仅返回原状态，不能再次产生副作用。
4. Cloud 撤回时关闭后续准入并保存待办；Controller 请求 Node 封闭旧代次的新执行与交互输入。已开始执行可以收尾，尚未接受的旧动作被拒绝；在途接受结果未知时不得推断其尚未开始。
5. Node 回报旧入口已封闭、已接受执行清单及其结束证据；Cloud 对账无缺口后才允许新持有者。原任务结束回报不因用户失权而被丢弃，但不能借事实回报创建新任务。

“立即拒绝新请求”指成员停用提交之后的 Cloud 授权判定，不声称网络分区下远端瞬时收到撤回。撤回前已经放行的在途请求有独立执行责任，必须查询并纳入交接条件；无论响应多晚，都不能与新操作者并行写入。不能将远端确认延迟掩盖成已经完全撤权。

Node 不以自身墙上时钟重解释 PostgreSQL 租约。它不得离线接收无新许可的顶层操作，网络断开不能使原会话拥有无限执行权；已开始任务的既定执行范围继续收尾。所有未来文件、终端输入、Git、Agent 和插件入口都须接入同一准入边界，旁路入口保持关闭。

### D4：控制绑定持久化并在真实执行点生效

Node 在受保护的本地状态中保存当前运行实例和控制代次的绑定、封闭状态及未完成执行责任。新绑定持久化并完成旧入口关闭后才确认；低代次、同代次不同绑定、错误目标实例均拒绝。重启不重新开放旧代次，不以空内存代表没有活动。

队列里尚未执行的旧指令、旧连接迟到的输入，以及曾被接受但未确定结果的派发都必须重新检查绑定或按原执行身份恢复。不同标识不能引用同一旧许可发起新写入。已经运行的旧进程不靠改变代次停止，正常交接需等待其结束；紧急情形由[强停](../operation/20260927-safe-stop-and-administrative-force-stop.md)终止整个目标执行范围。

若 Node 不可达，只有能够证明旧沙盒不可再写入数据的终止或存储隔离证据才能替代其封闭/结束确认。不能因心跳超时、容器列表暂时缺失或创建新 generation 就开始并行挂载同一可写目录。Substrate 必须阻止迟到 ensure 重新创建被终止的旧实例，并保留稳定 effect 查询结果。

### D5：兼容升级不允许静默退回旧控制路径

Cloud–Controller 契约仍由 Cloud proto 拥有，Controller 锁定可审查的契约 commit；Node 消息按自身协议版本与能力协商演进。身份验证、强制控制绑定等不能被旧调用方忽略的变化，要明确版本化或在握手/入口拒绝不支持的对端，不能仅新增可选字段后默认放行。

上线顺序：先部署能识别新控制规则的执行端和 Controller，再部署 Cloud 的前向迁移与受限入口，验证存量资源的执行责任后开放前端。切换期间先关闭新写准入，存量任务收尾或保持待对账，不要求销毁历史目录。旧的 HTTP/internal 入口也必须复用新授权逻辑或被关闭，不能留下兼容旁路。

回滚前须停止新准入、完成活动收尾并保留新持久状态；旧版本不能把新状态当作空闲继续写入。没有兼容读取与拒绝策略的版本不得连接已升级的运行时。维护窗口和能力拒绝是显式兼容代价，不用放宽约束换取无感升级。

### D6：按责任同步交付与验收

| 组件 / 仓库 | 必须交付 | 完成证据 |
| --- | --- | --- |
| Cloud | 创建者、权限、独占/活动状态、强停、凭据引用状态、插件等待、迁移及审计；服务认证；公开和内部契约 | 真实 PostgreSQL、HTTP、并发竞争、重试与升级测试；生成文件无漂移 |
| Ora 前端 / Cloud frontend | 概况与内容权限、我的运行时、占用提示、获取/续期/释放、强停确认、断线恢复；不做授权界面 | 生成客户端、界面测试及中英文模块文档；前端门禁 |
| Controller / desktop | 经认证连接、消费限定许可、派发前登记与再核验、失权停止新派发、旧执行对账及强停优先协调 | 真实 Cloud 适配器与 Node 对端的契约和恢复测试；禁止内存状态充当权威 |
| Node / desktop | 持久控制绑定、执行入口旧资格隔离、活动跟踪、输入关闭、凭据引用实际解析与运行域隔离 | 真实 Git/文件/进程测试，旧消息和重启不能造成新写入 |
| Substrate / cluster 或部署适配器 | 受限服务身份、目标终止确认、旧实例禁止重建、持久目录挂载和显式删除；管理/工作负载权限隔离 | 实际部署故障与终止证据；仅有 ADR 或 simulator 不算完成 |
| specs | 对应 approved 决策、稳定核心用例及逐项证据 | 按已验证范围标 Covered/Partial/Missing，不将下层间接测试凑为完整证据 |

Cloud PostgreSQL 保存权威业务状态；Node 本地日志承担真实执行责任和恢复证据，两者不是数据库替代关系。云端 Controller 不使用本地 SQLite 另建业务权威；桌面独立运行模式的已有本地持久化不由本文改造。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 只增加 Cloud 数据库锁 | 可以限制 API，却不能阻止迟到命令或仍存活的旧进程 |
| 将 Controller 单连接等同单用户 | 连接简单，但一个 Controller 可以代理许多用户和任务 |
| Node 自己判断租户成员或信任客户端 userId | 省去协调，却形成第二份权限权威或允许身份伪造 |
| 延用未认证 metadata 并宣称控制令牌可信 | 适合原受信开发边界，无法提供本次更强的执行准入保证 |
| 控制代次一变化就宣布旧进程终止 | 状态易实现，但代次检查只能拒绝新操作，不能消除既有进程副作用 |

## 不变量与预期验收

1. 伪造 ControllerId、错误服务身份、跨运行实例命令和直接 Node 访问不能获得执行资格。
2. 派发前记录、响应丢失、撤回同时发生时，只存在一份执行责任；未知结果阻止交接。
3. Node/Controller 重启、旧连接恢复、排队旧指令和迟到输入不能在交接确认后造成新写入。
4. 工作负载不能改控制日志或读取管理凭据；旧进程仍存活时不声称目录安全交接。
5. 真实 sandbox terminate 与迟到 ensure 竞争不会恢复旧写者；不同运行时目录与任务互不影响。
6. 新旧组件混用明确拒绝不支持的能力，不能静默走旧接口；升级与回滚保留持久证据。

## 落地顺序与后续边界

本 ADR 及其引用的同批业务决策已批准，并已建立核心用例，可进入开发。各仓库 PR 明确契约 commit、先后依赖和真实验收范围，不能仅 Cloud 先合入就宣布多人控制完成。第一阶段覆盖已经存在的真实执行路径；尚无完整云端文件/PTY/Agent 产品入口的部分保持不可用。实时共同编辑、跨宿主无损恢复及重建完整本机信任体系不是本次隐含任务。
