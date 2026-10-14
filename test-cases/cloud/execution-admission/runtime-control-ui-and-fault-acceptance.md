# 运行时控制 UI 与联合故障验收

依据[后续 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)。
三处 UI 为 approved 待交付义务；后端局部证据保留在各领域原用例，不重复标记为完整 UI/故障证明。

## Operator Controls Must Recheck Authority After Disconnection

- **风险：** acquiring 或旧 session 被界面视为有效，恢复时盲目重放写入。
- **前置状态：** 同用户两页面、占用/无权限、续期将到期、网络断开或成员失效。
- **触发：** 获取、续期、释放、断开及恢复。
- **必须成立：** held 确认后才开放交互，断开停发，恢复重查；释放后等待责任结清，冲突与权限提示区分。
- **禁止结果：** 第二页面共享旧资格、离线自动重放、到期立即显示空闲。
- **验证义务与证据：** Missing：真实浏览器两页面、响应延迟/丢失与服务端状态联合验证。
- **决策依据：** D1。

## Force Stop Confirmation Must Preserve Scope and Pending Responsibility

- **风险：** 误强停、接受即显示终止、响应丢失制造第二意图。
- **前置状态：** 当前管理员/普通成员/跨租户用户，两个运行时及一个在途任务。
- **触发：** 普通停止、确认强停、响应未知、关闭页面与重查。
- **必须成立：** 展示具体目标和未保存风险，原因/版本/稳定幂等输入完整；服务确认终止前保持 pending。
- **禁止结果：** stop 自动强停、重新请求换键、扩大目标、自动启动/夺锁/删卷。
- **验证义务与证据：** Missing：浏览器确认与真实服务意图/目标对象联合验收。
- **决策依据：** D2。

## Runtime Views Must Separate Shared Summaries From Restricted Content

- **风险：** 我的运行时或管理员过滤让其它成员获得受限详情。
- **前置状态：** 创建者、普通他人、管理员、unknown creator 及已降级用户。
- **触发：** 列表、详情、直接链接、重试、角色变化及事件刷新。
- **必须成立：** 安全概况共享，服务权限决定内容/操作，失效后清除受限视图；不出现未批准共享授权。
- **禁止结果：** 共享卡片带命令/路径/输出，按钮状态充当权限或 unknown creator 对成员开放。
- **验证义务与证据：** Missing：真实角色矩阵浏览器路径；HTTP 局部证据不替代此义务。
- **决策依据：** D3。

## Unknown Execution and Termination Must Block Handoff Across Real Fault Boundaries

- **风险：** 接受丢失/重启/旧队列/迟到 ensure 后出现第二写者，升级丢失执行或创建者证据。
- **前置状态：** ADR D4 各行所述多实例、未知接受、在途执行、软删除历史或新旧组件组合。
- **触发：** 精确故障注入与恢复后交接/强停/升级/装卸。
- **必须成立：** 每行均核对稳定责任、真实文件/进程/容器/卷及允许结果；未确认继续阻止交接。
- **禁止结果：** 注入 Node 确认被当作实际终止、模拟器替代真实装卸、修改版本恢复旧实例或抹去缺口。

### 验证义务与证据

以下每行对应 ADR D4 的独立义务。源码基线为 cloud `b82834e`、desktop `cf166325`、cluster
`aef10f3`；路径分别以仓库根目录为准。`Partial` 仅表示列出的直接局部证据，最后一列仍是完成
该行的必要条件；用例导航本身不构成证据。

| D4 义务 | 现有直接局部证据 | 状态 | 尚缺的边界 |
| --- | --- | --- | --- |
| 同会话多入口与多运行时隔离 | desktop `crates/node-db/src/tests/repository.rs::runtime_mutex_spans_identities_and_started_work_survives_closure` 检查互斥和关闭后既有责任；cluster `scripts/runtime-control-acceptance.py` 创建两个真实 Node/Git/卷并核对写入隔离 | Partial | 已开放入口之间实际进程竞争与单目标处置；未开放文件/PTY/任意 shell、完整 Agent 入口由 ADR 未解决问题明确承接 |
| 成员撤权与在途任务 | cluster `scripts/runtime-control-acceptance.py` 的成员停用步骤检查 HTTP/SSE、续期、新准入与 Node 控制撤回 | Partial | 停用与已接受/未知接受任务、新请求及事件投递同时竞争，真实任务收尾不丢责任 |
| 插件角色与租户组合 | cloud `integration/plugin_pending_test.go::TestPluginSelectionRequiresAdminAndWaitsForControlThenExecutor` 直接验证管理员门控和忙时持久等待；前端控件测试不作为服务端拒绝零副作用的证据 | Partial | 跨租户与选择、移除、角色变化及重放的联合零副作用矩阵 |
| 多实例与页面独占 | cloud `integration/runtime_control_test.go::TestStoppedRuntimeControlConcurrentAcquireAndPageSeparation` 与 `TestRuntimeControlRequiresAcknowledgementAndRetainsExpiredResponsibility` 检查同步获取、确认门禁和到期责任 | Partial | 多个真实 Cloud 服务实例、独立连接的续期/到期竞争和响应丢失 |
| 派发/撤回与未知接受 | cloud `integration/runtime_control_test.go::TestRuntimeControlRequiresAcknowledgementAndRetainsExpiredResponsibility`；desktop `crates/node-db/src/tests/repository.rs::runtime_mutex_spans_identities_and_started_work_survives_closure` 检查接受后责任保留 | Partial | 真实 gRPC–Controller–Node 接受前后丢响应、分区与重启，不重复执行且未知阻止交接 |
| 旧消息与重连 | cloud `integration/runtime_connection_recovery_test.go::TestNodeReconnectRestoresAvailabilityButNeverOldPageQualification` 验证恢复不复活旧页面资格 | Partial | 真正排队旧字节经旧连接/节点重启后的文件、Git 和进程零新写入 |
| 管理身份与工作负载 | cloud `internal/controlgrpc/tls_test.go::TestManagementMutualTLSAndBoundControllerIdentity`；desktop `crates/node-transport/tests/mtls.rs::management_tls_authenticates_both_peers_and_refuses_workload_certificate`；cluster 脚本以 UID 1000 验证管理文件不可读写 | Partial | 失效证书、所有范围错配、生产轮换与工作负载伪造管理入口的真实组合 |
| 强停与迟到 ensure | cloud `integration/runtime_force_stop_test.go::TestForceStopRegistersDuringLifecycleAndFencesLateEnsureWithoutRewritingHistory`；cluster `sandbox-server/internal/lifecycle/runtime_fence_test.go` 验证持久 fence 和旧 ensure | Partial | 真实 ensure/terminate 同时竞争、响应丢失、协调器崩溃及磁盘核对 |
| 持久数据与未知终止 | cloud `integration/runtime_force_stop_test.go::TestRestartIsOneDurableIntentAndNeverClonesAgain`；同文件 `TestForceStopRegistersDuringLifecycleAndFencesLateEnsureWithoutRewritingHistory` 的 `early-take` 409 直接验证未确认终止不能接管；cluster 脚本核对目标容器终止、卷保留及另一运行时内容 | Partial | Node 不可达与停止、强停、重启、显式删除的组合；未确认不能交接或删卷 |
| 历史升级与回滚 | cloud `integration/runtime_upgrade_test.go::TestRuntimeUpgradeOnlyBackfillsUniqueCreationEvidenceAndKeepsUnknownReferences` 核对唯一/冲突/缺失创建证据、operation 快照和重复迁移 | Partial | 软删除/在途 execution/effect 的全表快照、新旧组件组合与实际部署回滚 |
| 插件与最新期望 | cloud `integration/plugin_intent_recovery_test.go::TestChangedPluginIntentReconcilesUnknownOldResultBeforeRemoval` 以 PostgreSQL 和显式 Node 替身核对稳定执行重放及唯一后续维护 | Partial | 真实 Node 下载、安装/移除、迟到结果、未知响应、目标删除与维护/用户竞争 |

### 决策依据

[后续 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md) D4、D5 与“本决策未解决的问题”。
三处 UI 与矩阵分别判断；不能从局部 `Partial` 推导整行 `Covered`。
