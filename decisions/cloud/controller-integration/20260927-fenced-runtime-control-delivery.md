---
status: implemented
---

# 运行时控制由 Cloud 授权、Controller 派发、Node 在真实入口持久隔离旧资格

Cloud 持久登记限定工作；Controller 派发前取得当前许可；Node 在接受及首次变更处验证目标、
控制绑定与稳定执行身份，关闭旧入口并回报未完成责任。更改代次只拒绝新动作，不等于停止旧进程。

后端路径已实现，业务权限由[使用权限](../workspace/20260927-runtime-workspace-use-permissions.md)
及[独占准入](../execution-admission/20260927-exclusive-runtime-control.md)拥有。三处 UI 和跨仓库
未覆盖故障矩阵由[后续 ADR](../execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)
承接；核心证据见[交付用例](../../../test-cases/cloud/controller-integration/fenced-runtime-control-delivery.md)。
真实 TLS/Node 与真实沙盒的局部验证不等同于全部故障及升级组合已完成。

## 继承与修改

继承[Cloud gRPC 根契约](0-cloud-owned-internal-grpc-contract.md)、
[Workspace 操作与 Node 代报](20260926-workspace-operations-and-controller-reported-nodes.md)、
[存活沙盒查询](20260926-list-live-sandboxes.md)及保活规则。Controller 继续只拨出，旧 JSON
监听/minicloud 不恢复。原受信私网自报 metadata 边界已改为双向 TLS 和范围绑定。

## 决策

### D1：不同代次和身份分别负责

| 标识 | 权威与用途 |
| --- | --- |
| 用户及当前成员 | Cloud 认证/授权 |
| Controller lease epoch | 当前协调者写入资格 |
| Workspace runtime generation / sandbox ID | 一个运行实例及其持久目标 |
| runtime control epoch / session ID | 一次用户或维护操作资格 |
| Node / Host incarnation | 一次执行程序实例 |
| execution / effect ID | 固定输入的持久副作用责任 |

Controller 不复制角色政策，不直接访问 PostgreSQL，不解析终端用户 token。稳定执行 ID 不代表
新的授权，也不能复用旧许可换 ID 建第二次任务。

### D2：管理通道认证服务身份，工作负载无权修改管理事实

Cloud–Controller 和 Controller–Node 管理链路验证双向 TLS 身份与目标绑定；声明 ID 只用于
一致性核查。部署私钥通过管理引用加载，工作负载身份不能读私钥、写控制日志或借管理端点获得
资格。浏览器不直接连接 Node/Substrate。本机可信 host 边界没有因此变成完整租户授权系统。

### D3：派发、撤回与事实回报构成同一责任链

Cloud 事务先核验权限、控制、运行实例与冲突活动，再保存固定执行输入。Controller 新派发前
取得当前许可，Cloud 不可用或许可失效时不发起新动作。Node 先持久接受再执行，同身份同输入
重传只返回已有状态。

撤回关闭新准入，Controller 将 input-closed 绑定送达 Node；未接受旧动作拒绝，已接受工作按
原范围收尾。Node 回报封闭状态与未完成 execution IDs，Cloud 对账结清后才交接。响应丢失
保留未知状态，不凭缓存放行；用户失权不使系统丢弃原执行的结束事实。

### D4：绑定持久化，强停需覆盖迟到资源

Node 保存运行目标、控制代次/版本、关闭状态与责任；低代次、同代次不同绑定、错误 incarnation
或目标拒绝。接受队列和真实变更处均校验，重启不凭空内存开放旧入口。

紧急终止使用独立 effect permit，期限不超过 10 秒且不越过 Controller 租约；执行端以持久
runtime fence/tombstone 限制旧 ensure。Cloud 只接管同时证明 `terminated` 与 `lateEnsureFenced`
的终止结果。不可达、心跳超时或新 generation 本身不允许同目录并行写入。

### D5：不支持控制能力的对端与旧旁路明确拒绝

Cloud proto 为契约源，Controller 固定可审查版本并生成。Node 显式协商 RuntimeControl 等能力；
受控执行不能静默退回无绑定请求。生产旧管理入口关闭，历史责任仍可查询。前向迁移保留历史
owner、执行和审计引用，不能以删除新持久状态实现回滚。

### D6：业务权威与执行证据分属其 owner

Cloud 的 HTTP/gRPC/PostgreSQL 测试证明授权与责任转换；Node 的本地持久测试证明真实入口隔离；
Sandbox Server 的持久 fence 和部署验证证明目标终止边界。注入 acknowledgement 或 simulator
不构成实际进程停止证据。UI、接受结果丢失、旧队列、证书轮换和版本混用/回滚的完整矩阵保持
后续 approved 验收范围。

## 不变量

未开放的能力仍须遵守同一入口约束，历史静态凭据或私网 metadata 不能作为备用控制路径。

1. 服务身份与运行范围共同验证，metadata 不授予信任。
2. 派发前有持久责任，未知接受阻止交接。
3. 旧资格与重复请求不能建立新写入责任。
4. 工作负载不能读写管理材料；代次不能证明旧进程已终止。
5. 终止确认包括迟到 ensure 围栏，不能只改数据库状态。
6. 兼容入口明确拒绝缺能力对端，升级不丢弃持久证据。
