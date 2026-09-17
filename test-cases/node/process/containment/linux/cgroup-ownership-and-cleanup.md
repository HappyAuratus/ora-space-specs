# Linux 创建纳管与清理边界

本文件按实际平台与保证记录验证义务；文档中的机制选择不构成实现通过证据。

## Linux Workloads Enter Their Delegated Cgroup Before Execution

### 风险

先 spawn 后迁移或委派权限错误会产生逃离纳管的后代，无法给出强静止证据。

### 前置状态

真实 Linux 部署提供声明的 cgroup v2 子树；另准备缺少委派权限或能力的容器／gVisor 环境。

### 触发

通过生产入口启动可立即派生后代、后台化及尝试迁出的工作负载，并分别请求强模式和允许降级模式。

### 必须成立

- 业务执行前进入正确子树，后代继承归属，在声明权限范围内不能迁出；只管理获委派范围。
- 能力不足按请求拒绝或启动前降级；支持等价委派部署，不由 systemd 是否存在替代能力判断。

### 禁止结果

业务已执行才迁移；写入委派外 cgroup；普通 Linux 通过即宣称 gVisor 支持。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 业务执行与派生前已纳管，声明权限阻止迁出 | 尚无直接证据；需真实进程、权限与创建同步点验证 | `Missing` |
| 委派边界、等价部署及受限环境拒绝／降级正确 | 尚无直接证据；需记录内核、实际委派与容器／gVisor 配置的独立验证 | `Missing` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/linux/0-cgroup-v2-containment.md) D1、D3 与不变量 1、3。

## Linux Cleanup Requires Closed Launch Paths And An Empty Subtree

### 风险

仅观察空组、发送 kill 或 guardian 死亡，不能排除随后启动或残留工作负载。

### 前置状态

Scope 内 A、B 两个 Run 分别有后代，guardian 在工作负载子树之外；可并发启动并强杀 guardian。

### 触发

先单独终止 A，再关闭 scope；分别在清理中强杀 guardian、停止宿主服务并恢复核实。

### 必须成立

- 单 Run 清理不清空 B；关闭先封闭入口，再终止并核实目标子树为空。
- Guardian 死亡后由宿主／恢复流程定位清理，不能假设自动清空；声明存续的 guardian 不因外部管理器停止宿主被连带清理。

### 禁止结果

空组快照之后继续启动；用 scope 总组冒充 Run 隔离；kill 返回成功或 guardian 死亡即报告静止。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| Run 隔离与封闭入口后核实空子树共同成立 | 尚无直接证据；需真实后代和启动／清理竞争测试 | `Missing` |
| Guardian 强杀仍可定位清理，外部服务停止遵守存续政策 | 尚无直接证据；需实际部署管理器与恢复流程验证 | `Missing` |

### 决策依据

[根决策](../../../../../decisions/node/process/containment/linux/0-cgroup-v2-containment.md) D2、D3 与不变量 2、3。
