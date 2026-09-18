# 实际能力与清理证据

本文件按实际平台与保证记录验证义务；文档中的机制选择不构成实现通过证据。


## 当前验收范围（2026-09-18）

当前范围是 Linux rootless BestEffort 及内核请求选择；Strong 正向、macOS／Windows、受限部署
与外部服务协调由未来 ADR 负责。保留下文跨保证用例作为长期风险地图，Partial 不等于当前生产支持 Strong。

## Containment Guarantees Are Fixed Before Business Execution

### 风险

先执行后降级、按环境名称猜能力或在失败后降低标准，会破坏资源恢复承诺。

### 前置状态

环境分别具备和缺少强纳管能力，请求策略分别为必须强、优先强可降级、明确尽力；另有已接受但启动未知的 Run。

### 触发

创建 scope 并启动 Run；随后使强模式清理失败、重试到限，并重传未知启动请求。

### 必须成立

- 必须强且能力不足时副作用前拒绝；允许降级时在业务执行前确定、保存并公开实际保证。
- Scope 内 Run 遵守同一契约；失败、期限或次数不能事后改变保证，启动未知不能换实现再次运行。

### 禁止结果

以 Linux／容器／gVisor 名称断言强能力；静默降级；强清理失败后改称尽力成功。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 三类请求按实际能力在执行前选择保证或拒绝 | `crates/process-runtime/tests/admission.rs::containment_selection_respects_required_and_explicit_guarantees` 的受控能力测试通过（Linux 上运行）；`linux_best_effort.rs::rootless_admission_exit_and_idempotency` 另验证真实无特权 adapter 拒绝 RequireStrong、PreferStrong 选择 BestEffort；各受限部署仍需验证 | `Partial` |
| 保证跨重启与错误保持，未知启动不会切换后端重跑 | apps/ora-process-host/tests/service.rs::host_kill_reconnect_stop_and_close_use_original_attempt 覆盖原尝试／BestEffort 事实恢复；Strong 正向和完整错误组合不在现有证据内 | `Partial` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D1 与不变量 1、2。

## Cleanup Results Never Exceed Their Evidence Boundary

### 风险

把尽力清理、超时或本地进程退出当成完整静止，会放行仍可能受到写入影响的资源。

### 前置状态

分别准备强静止证据、尽力清理完成、已知残留、清理失败与无法核实的状态；插件还可调用外部服务持续写入。

### 触发

请求关闭并查询结果，Node 据此判断资源恢复；推进等待期限和清理重试次数。

### 必须成立

- 强静止同时要求声明范围内完整退出证据及入口封闭；尽力完成保留不能排除残留的事实。
- 已知残留、失败和无法核实保持未完成／未知；本地证据不替代外部服务对账。符合尽力政策且对账完成后允许恢复。

### 禁止结果

超时生成完成；尽力完成改名为强静止；把本地清理视作外部任务已取消。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 三类清理结果与证据一一对应，失败不伪造完成 | `crates/process-runtime/tests/lifecycle.rs::cleanup_failure_recovers_without_erasing_exit_evidence`、`contradictory_observations_remain_blocked_until_valid_evidence_arrives` 验证尽力完成、错误和矛盾事实；`crates/process-runtime/tests/facts.rs::exit_status_is_refined_and_never_erased_by_less_precise_observations` 验证退出事实单向细化；`linux_best_effort.rs::direct_exit_keeps_background_descendants_pending_until_stopped` 另验证真实 Linux 后代存活时不能完成，完成只输出 BestEffortComplete；强模式与完整故障矩阵仍缺失 | `Partial` |
| 外部副作用保留独立责任，尽力恢复满足政策与对账后可推进 | 尚无直接证据；需外部写入服务和 Node 资源协调验证 | `Missing` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D2、D3 与不变量 2、3。
