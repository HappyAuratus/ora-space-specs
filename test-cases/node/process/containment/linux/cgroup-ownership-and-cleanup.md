# Linux 创建纳管与清理边界

本文件按实际平台与保证记录验证义务；文档中的机制选择不构成实现通过证据。


## 当前验收范围（2026-09-18）

host、guardian、`process-runtime` 和特权 helper 已随[Node 直接执行 Git 决策](../../../../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)删除，desktop 里现在没有
Linux 纳管实现；Node 自己的 Git 靠进程组和尝试锁收尾，保证等级是 BestEffort。下文全部用例作为将来 Linux 强纳管
落地时的验收输入保留，原有证据随代码删除，状态均为 Missing；不从 Node 直接执行 Git 的测试推导纳管保证。

## Rootless Cleanup Retains Captured Identities Without Claiming Strong Containment

### 风险

把 session 扫描当成完整纳管，或向过期 PID／进程组发信号，会漏清理或伤及无关业务。

### 前置状态

普通 Linux 用户不配置 helper、cgroup 或独立账号；两个 Run 各有后代，其中一个后代在被捕获后调用 `setsid`。

### 触发

请求强／允许降级／尽力模式；直接进程退出后观察后代，通知后升级强制停止，并重传已接受的启动。

### 必须成立

- 无特权路线只选择 BestEffort，RequireStrong 在执行前拒绝；重传不重复执行。
- 已捕获身份在 session 变化后仍受管理，停止一个 Run 不伤及邻居，通知成功不生成完成证据。
- 直接退出不释放尚未完成的责任；只有新扫描与已捕获成员退出均满足时才报告尽力完成。
- 无法核实或信号失败继续保留责任；发现前逃离的后代不在完整性承诺内，不能输出强静止。

### 禁止结果

要求用户配置 root helper 才能使用尽力模式；沿历史 PPID 猜归属；向复用身份错发信号；以正常 Drop 代替崩溃恢复证明。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 普通用户准入、重传、直接退出后后代收尾及 Run 隔离 | 尚无直接证据：原有的 `process-runtime` Linux 尽力清理测试随被取代的实现删除 | `Missing` |
| 已捕获成员脱离 session 后仍可停止，通知不冒充退出 | 尚无直接证据：原有的测试随被取代的实现删除。Node 自己的 Git 对脱离会话的后代靠尝试锁给出停止证据（见 [git-execution 用例](../../../git-execution/attempt-locks-and-stop-evidence.md)），那是另一种机制，不替代本行 | `Missing` |
| 复用身份不被误伤，观测／资源失败保留责任并可恢复 | `crates/utils/tests/linux_process.rs` 仍验证过期 proc 观测拒绝、原 pidfd 退出和非 UTF-8 名称，但使用这些辅助函数的纳管实现已经删除，它不再是本义务的证据；实际数字 PID 复用、描述符耗尽及失败恢复注入尚缺 | `Missing` |
| 所有者强杀后仍能定位、清理与恢复 | 尚无持久所有权的直接证据；Drop 测试不替代该义务 | `Missing` |

### 决策依据

[Linux 根决策](../../../../../decisions/node/process/containment/linux/0-cgroup-v2-containment.md) D3 与
[纳管契约](../../../../../decisions/node/process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) D1、D2。

## Privileged Helper Rejects Untrusted Deployment Authority

### 风险

把可被业务修改的配置、符号链接或同 UID 委派当成特权管理依据，会扩大 root 操作范围或允许工作负载迁出。

### 前置状态

管理员提供 helper 配置和专用 cgroup v2 子树；另准备共用身份、root 业务身份、可写配置及符号链接。

### 触发

通过部署预检入口加载配置，再由正式管理通道尝试创建工作负载。

### 必须成立

- 配置来自 root 控制的普通文件，逐级拒绝不可信祖先和符号链接，管理与业务身份分离。
- 未知版本、字段、非 cgroup 文件系统拒绝；预检成功不授予 Strong、不声称收尾完成。
- 管理通道读取请求前校验内核对端 UID；有界帧和期限限制单连接资源，检查回复不能被解释为启动授权。
- 实际启动重新验证权限和管理资格，业务无权选择 root 身份、任意 PID 或委派外目标。

### 禁止结果

预检成功冒充真实纳管证明；工作负载操控 helper 执行任意 root 命令；降权失败后仍执行业务代码。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 部署身份和可信配置门禁拒绝不安全输入 | 尚无直接证据：`ora-process-helper` 和 `process-runtime` 的 helper 测试随实现一起删除；`crates/utils/tests/trusted_path.rs` 只验证底层的可信路径打开，不替代本义务。特权文件加载、完整权限矩阵和正向部署同样待验证 | `Missing` |
| 管理通道先认证、有界处理且退出释放连接 | 尚无直接证据：原有的管理通道测试随 helper 一起删除；需覆盖 UID 拒绝、严格请求、过大帧、半包超时、监听退出，以及 root 端点权限／清理、连接饱和及跨身份进程 | `Missing` |
| 真实特权启动限制目标、执行前降权且拒绝迁出 | 尚无直接证据：原有的启动门禁测试和从未执行的特权验收入口随 helper 一起删除；`crates/utils/tests/child_identity.rs` 只验证底层的身份切换，不替代本义务 | `Missing` |

### 决策依据

[独立 Helper 决策](../../../../../decisions/node/process/containment/linux/20260917-independent-privileged-helper.md) D1、D2。

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
| 业务执行与派生前已纳管，声明权限阻止迁出 | 尚无直接证据：原有的 root／cgroup 验收入口从未执行，已随 helper 一起删除；需派生后代和创建同步点验证 | `Missing` |
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
