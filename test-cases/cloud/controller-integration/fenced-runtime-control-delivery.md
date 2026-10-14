# 真实执行入口、服务身份与跨进程恢复

已实现服务承诺仍由原 ADR 拥有；下文的 UI 与完整联合故障矩阵改由[后续验收 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)承接。保留本文件与英文标题作为稳定引用，Partial/Missing 不因后端 implemented 而自动提高。

依据[已实现后端的跨进程交付决策](../../../decisions/cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)。本文件是跨仓库验证义务的权威入口，各组件测试提供直接证据，不复制租户角色政策。
新增义务按直接证据登记 Partial/Missing。Cloud PostgreSQL 为业务权威；Node 本地受保护日志保存执行与恢复责任。Controller 继续只拨出，minicloud 与过渡 JSON 监听已退役，真实云端闭环在 cluster Compose 或等价真实部署环境验收。

## Management Identity And Workload Isolation Prevent Forged Control

- 风险：伪造 ControllerId/NodeId 或从用户代码读取管理凭据、篡改控制日志，绕过 Cloud 操作权。
- 前置状态：Cloud–Controller、网络 Controller–Node 配置双向 TLS 及目标范围；管理程序和工作负载属于不同操作系统权限域。
- 触发：可信身份连接，错误/失效身份、身份与目标不符、直接 Node 访问、工作负载读取凭据或修改日志、访问管理端点。
- 必须成立：只有认证且范围匹配的管理身份建立控制；声明 ID 不授予信任。工作负载读写管理材料和端点被拒，秘密不进入前端、业务表、日志或代码目录；可信本机宿主机制不被宣称完整用户授权。
- 禁止结果：metadata 自报身份代替认证、同 UID 可写控制日志仍开放操作、以 localhost 或单连接为授权证明。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 双向服务认证及资源绑定 | 真实 TLS 对端的正确、错误、失效身份与目标错配测试 | Partial |
| 工作负载权限隔离与秘密边界 | 实际沙盒以工作负载身份尝试读写管理文件、访问端点，检查日志与持久目录 | Partial |

- 决策依据：D1、D2，不变量 1、4。

- 本次直接证据：**Partial** — [Cloud internal/controlgrpc/tls_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/controlgrpc/tls_test.go)；[desktop crates/node-transport/tests/mtls.rs](https://github.com/ora-space/desktop/blob/cf166325d393e1b81a6b3f2fde7e6026d13cb128/crates/node-transport/tests/mtls.rs)；[cluster 真实验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/runtime-control-acceptance.py) 实际 UID 1000 拒绝读取密钥、恢复日志及写管理目录。
- 仍缺：失效证书、全部范围错配和工作负载直接伪造入口的真实联合矩阵仍缺；生产轮换与旧目录迁移尚缺。 验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。

## Revocation Reconciles Inflight Acceptance Before A New Operator Is Admitted

- 风险：Cloud 已撤权，但 Controller 缓存继续派发，或接受响应丢失导致交接遗漏执行。
- 前置状态：旧操作许可存在，分别有未派发、派发中、Node 已接受但响应丢失及已开始执行。
- 触发：释放/到期/停用与派发竞争；Cloud 或 Node 不可达；Controller 重启后查询。
- 必须成立：Cloud 保存许可和执行身份后才放行；Controller 新派发前重新核验；Node 持久接受后执行。撤回后关闭旧新输入，旧执行按稳定身份查询与回报；未知接受不能当未开始，全部核对完成前不得交接。
- 禁止结果：离线凭缓存新建执行、重试变换身份、因用户失权丢弃结束事实、声称远端瞬时撤权。
- 验证义务与证据：**Partial** — 需真实 Cloud gRPC/PostgreSQL、Controller、Node 接受边界的故障注入和恢复，检查只有一份持久执行责任及无交接窗口写入。
- 决策依据：D3，不变量 2、3。

- 本次直接证据：**Partial** — [Cloud integration/runtime_control_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_control_test.go) 的未结束责任阻止交接；[desktop crates/node-db/src/tests/repository.rs](https://github.com/ora-space/desktop/blob/cf166325d393e1b81a6b3f2fde7e6026d13cb128/crates/node-db/src/tests/repository.rs) 的持久关闭与接受/启动两道检查。
- 仍缺：真实 Cloud gRPC–Controller–Node 接受响应丢失与故障恢复联合矩阵仍缺。 验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。

## Stale Queued Commands And Restored Connections Cannot Write After Handoff

- 风险：只在接收时检查，排队旧命令、迟到输入或重启恢复再次产生副作用。
- 前置状态：Node 持久保存运行实例和控制绑定；旧代次有排队动作、既有运行进程和未完成执行责任。
- 触发：封闭旧入口，分别重启 Node/Controller、恢复旧连接、重传旧代次或同代次不同绑定、错误实例，然后完成安全交接。
- 必须成立：接受队列及真实变更入口都校验绑定；低代次、同代次错绑定、错实例被拒。重启不开放旧资格；既有进程结束或真实终止/存储隔离确认前不交接，同执行同输入重传返回原状态。
- 禁止结果：空内存当空闲、控制代次代替运行代次/Controller 租约、改代次就宣称进程结束、换 execution ID 重用旧许可。
- 验证义务与证据：**Partial** — 需真实 Node 文件、Git、进程和队列测试，结合 Cloud 交接，断言旧字节不能造成新写入；现有宿主代次测试不证明用户控制绑定。
- 决策依据：D1、D4，不变量 3–5。

- 本次直接证据：**Partial** — [Cloud integration/runtime_connection_recovery_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_connection_recovery_test.go) 证明可用性恢复不复活旧 SID；[desktop crates/node-db/src/tests/repository.rs](https://github.com/ora-space/desktop/blob/cf166325d393e1b81a6b3f2fde7e6026d13cb128/crates/node-db/src/tests/repository.rs) 证明旧资格重开后不能启动或建目录。
- 仍缺：旧字节队列、真实 Git/进程和 Cloud 交接的联合故障注入仍缺。 验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。

## Protocol Upgrade And Rollback Never Restore An Unfenced Path

- 风险：旧组件忽略新增可选字段，或回滚把新持久状态解释为空闲。
- 前置状态：旧新 Cloud、Controller、Node 混用；历史运行时与未完成执行存在；契约由 Cloud proto 拥有。
- 触发：能力握手、新控制请求、升级切换、旧入口调用、恢复扫描与回滚连接。
- 必须成立：明确版本化或拒绝不支持的对端；全部旧入口复用新准入或关闭。执行端与 Controller 能识别规则后再开放 Cloud/前端；切换期间暂停新准入，保留并核对既有责任。Controller 不恢复监听或 minicloud，云端不新增 SQLite 业务权威。
- 禁止结果：缺字段默认放行、手改生成文件、回滚删除新状态、以下层或旧 minicloud 测试宣称真实云端闭环完成。
- 验证义务与证据：**Partial** — 需组件版本组合、生成契约漂移、旧 HTTP/internal 路径及真实部署升级/回滚测试，分别登记各仓库与契约 commit。
- 决策依据：D5、D6，不变量 6；同批停止决策的迟到 ensure 验收联合执行。

- 本次直接证据：**Partial** — [Cloud integration/repository_credentials_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/repository_credentials_test.go) 的生产旧 clone 410；TLS 无明文回退；生成契约固定提交并检查漂移。
- 仍缺：各组件版本组合与真实部署回滚矩阵仍缺；无安全配套的新能力明确拒绝。 验收归属：[后续 ADR D4](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md#d4联合故障矩阵要求真正跨过所声称的边界)。
