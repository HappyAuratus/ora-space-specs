# Windows Job 归属与退出证据

本文件按实际平台与保证记录验证义务；文档中的机制选择不构成实现通过证据。

## Windows Workloads Join The Job At Creation Without Breakaway

### 风险

创建与纳管窗口或外层 Job 冲突会留下不受控进程；继承管理句柄会改变清理时点。

### 前置状态

真实 Windows 上准备兼容及不兼容外层 Job 环境；业务进程可立即创建后代并尝试 breakaway。

### 触发

通过生产入口创建 Run，检查执行前归属、后代行为及业务进程持有的句柄。

### 必须成立

- 创建时加入目标 Job，业务执行前完成归属；禁止声明范围内后代 breakaway。
- 检查嵌套规则，无法满足请求的能力时拒绝或预先降级；guardian 不在工作负载 Job 中，业务不继承管理句柄。

### 禁止结果

启动后才补分配 Job；忽略外层限制继续宣称强保证；业务持有管理句柄。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 执行前归属和后代约束在真实 Job 环境成立 | 尚无直接证据；需创建同步点与 breakaway 尝试 | `Missing` |
| 嵌套能力不足得到正确处置，句柄继承和 guardian 归属正确 | 尚无直接证据；需 Windows 句柄与外层 Job 检查 | `Missing` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/windows/0-creation-time-job-containment.md) D1、D2 与不变量 1、2。

## Windows Cleanup Verifies Emptiness Despite Handle And Notification Failures

### 风险

Guardian 死亡可能不是最后句柄关闭，通知缺失或 Job 消失也不证明已取得完整退出证据。

### 前置状态

同一 scope 有可独立终止的 A、B；宿主／恢复程序可能额外持有 Job 句柄，通知可延迟或丢失。

### 触发

单独终止 A，关闭 scope 并强杀 guardian；分别保留额外句柄或让 Job 消失后恢复。

### 必须成立

- 先封闭启动入口，再终止并独立核实活动进程归零；单 Run 操作保持 B 存续。
- 每个额外句柄持有者承担清理责任；通知只唤醒核实，Job 消失后的证据不足时保持未知。

### 禁止结果

Guardian 死亡直接当作最后句柄关闭；只凭完成端口通知报告静止；为终止 A 清空 B。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| Run／Scope 清理按实际 Job 归属隔离，入口封闭先于静止判定 | 尚无直接证据；需真实 Windows 并行 Run 测试 | `Missing` |
| 额外句柄、通知缺失和 Job 消失均不制造虚假完成 | 尚无直接证据；需强杀、句柄控制与证据链恢复测试 | `Missing` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/windows/0-creation-time-job-containment.md) D2、D3 与不变量 2、3。
