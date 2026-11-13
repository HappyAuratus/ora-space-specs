# 实际能力与清理证据

本文件按实际平台与保证记录验证义务；文档中的机制选择不构成实现通过证据。


## 当前验收范围（2026-09-18）

实现这些义务的 host、guardian 和 `process-runtime` 已随[Node 直接执行 Git 决策](../../../../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)删除，desktop 里
现在没有按请求选择或上报纳管保证的实现；Node 自己的 Git 固定为 BestEffort（该决策 D8）。下文用例作为长期风险地图
和将来 Strong 纳管落地时的验收输入保留，原有证据随代码删除，状态均为 Missing。

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
| 三类请求按实际能力在执行前选择保证或拒绝 | 尚无直接证据：原有的 `process-runtime` 准入测试随被取代的实现删除，desktop 里现在没有按请求选择纳管保证的入口；需在新的纳管实现上验证三类请求和各受限部署 | `Missing` |
| 保证跨重启与错误保持，未知启动不会切换后端重跑 | 尚无直接证据：原有的 host 服务测试随被取代的实现删除；需覆盖 BestEffort 与 Strong 的事实恢复和完整错误组合 | `Missing` |

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
| 三类清理结果与证据一一对应，失败不伪造完成 | 尚无直接证据：原有的 `process-runtime` 生命周期、事实细化和 Linux 尽力清理测试随被取代的实现删除。Node 自己的 Git 在“停止证据不成立就不报告已停止”上的证据登记在 [git-execution 用例](../../../git-execution/attempt-locks-and-stop-evidence.md)，那里只有 BestEffort 一种结果，不替代本行的三类结果 | `Missing` |
| 外部副作用保留独立责任，尽力恢复满足政策与对账后可推进 | 尚无直接证据；需外部写入服务和 Node 资源协调验证 | `Missing` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D2、D3 与不变量 2、3。
