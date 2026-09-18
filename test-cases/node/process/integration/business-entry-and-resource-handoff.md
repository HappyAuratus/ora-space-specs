# 业务入口、远程句柄与资源交接

当前范围为 Linux 独立 Node 受管 Git。插件、角色授权、租约、Backend 入口切换及跨 Node 协调
是未来 ADR 输入，不是当前交付。共享仓库冲突集合仍待确认，不能将该风险整体延期。

## Remote Handle Release Does Not Cancel Accepted Operations

当前只承诺请求取消不撤销已接受责任；下文 Session 租约、只读角色与高层句柄属于未来 ADR。

### 风险

把远程句柄当作本地进程所有权，会在 Drop 或取消等待时意外终止运行；只读资格也可能越权。

### 前置状态

Session／Scope／Run 已建立，另有只读观察者；启动／关闭处于接受但尚未完成阶段。

### 触发

分别释放句柄、取消等待、停止 session 续租、显式终止运行，并由观察者尝试变更。

### 必须成立

- Drop 仅释放本地句柄，取消仅影响等待，显式停止和整体租约到期各按自己的规则处置。
- 只读观察不能变更；生产 Client 区分接受／完成，并暴露保证、未知事实、输出完整性、控制权失效与清理结果。

### 禁止结果

释放临时句柄隐式关闭 scope；取消等待撤销接受责任；只返回退出码掩盖收尾未知；观察者获得控制权。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 释放、取消、显式终止和租约处置具有独立可观察行为 | 尚无直接证据；需真实远程句柄与受控租约测试 | `Missing` |
| 只读资格被限制，Client 保留接受／完成及未知状态 | 尚无直接证据；需公共 Client 接口和宿主会话测试 | `Missing` |

### 决策依据

[根决策](../../../../decisions/node/process/integration/0-shared-process-entry-and-resource-handoff.md) D1。

## Business Attempts Use One Durable Process Entry Across Retries

### 风险

Git 直接 spawn、辅助命令旁路或 Adapter 自动换 ID 会绕开纳管并重复业务副作用。

### 前置状态

真实 Git 多步骤操作与测试插件通过注入 Adapter 接入；Node 持久保存 Execution、Scope、各命令 RunId 关联。

### 触发

丢失启动回复，在 Git 已修改仓库但结果未保存时强杀 Node，接管插件保留运行；同时检查业务辅助命令和新旧入口迁移。

### 必须成立

- 每条变更命令在启动前持久关联业务身份；只读命令在 host 保留 Run 身份，重传保留原 RunId；主进程及直接辅助命令走唯一新入口，生产无隐式无纳管后端。
- Git 对账实际仓库，插件核实已交付命令与会话，Node 决定新的业务尝试；运行层不解释业务成功。
- 基础设施引导不能成为业务旁路，新旧体系同一尝试不双跑，旧运行明确停止交接。

### 禁止结果

超时自动换身份再次执行 Git；把插件管道恢复当成业务会话已恢复；用 fake 证明生产启动链路。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| Git／插件真实入口及辅助命令统一纳管，关联先于副作用 | apps/ora-node/tests/standalone.rs::node_kill_and_immediate_restart_wait_for_old_git_cleanup 验证生产 Git 及恢复；变更 Run 派发前持久关联，只读 Run 仅由 host 记录；插件与 Backend 切换延期 | `Partial` |
| 回复丢失和业务副作用后崩溃仍恢复原尝试，按领域对账 | apps/ora-node/tests/standalone.rs::node_kill_and_immediate_restart_wait_for_old_git_cleanup 及 apps/ora-process-host/tests/service.rs::lost_reply_executes_once_and_stalled_output_does_not_block_control；Node 与 host 各自故障路径已测，完整组合未齐 | `Partial` |
| 插件接管不重复已交付命令，迁移不双跑 | 未来插件／Backend 切换 ADR 验收输入 | `Missing` |

### 决策依据

[根决策](../../../../decisions/node/process/integration/0-shared-process-entry-and-resource-handoff.md) D2 与不变量 1、3。

## Conflicting Resources Wait For Scope Cleanup And Business Reconciliation

### 风险

按路径分配资源或失权即解锁，会让新工作与旧进程／外部服务同时写共享仓库。

### 前置状态

两个 worktree 共享仓库元数据，另有无冲突资源；scope 已持久关联占用，包含多个 Run 和可能的插件外部任务。

### 触发

Node 强杀后立即重启，分别出现租约失效、直接退出、单 Run 完成、强静止、尽力完成、清理失败及外部任务未结束。

### 必须成立

- 冲突判断由 Git／资源协调领域拥有；占用关联先于启动，scope 关闭、按实际保证清理和必要对账完成后才持久释放。
- 强模式需静止证据；尽力模式按已选政策完成清理与对账后可自动恢复，但保留残留风险。已知残留／失败均不得放行。
- 单 Run 完成不释放开放 scope 共同占用，无冲突工作可独立推进；共享协调域只有一个有效分配权威，跨 Node 共享存储也不能各自授权冲突工作。

### 禁止结果

租约过期或数据库锁释放即放行；不同 worktree 路径即判无冲突；本地静止抹掉外部服务责任；条件满足后永久卡住。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 持久资源关联与领域冲突集合防止重复分配，单 Run 不提前释放 | apps/ora-node/src/tests/local_recovery.rs 覆盖资源预留和重开；共享仓库元数据的完整冲突政策仍未确认 | `Partial` |
| 两种保证分别要求清理和对账，失败阻塞且恢复后可推进 | apps/ora-node/tests/standalone.rs::guardian_loss_keeps_conflicting_recovery_unknown_but_allows_other_work 覆盖 BestEffort 缺证据不放行；Strong 与外部服务协调由未来 ADR 负责 | `Partial` |
| 同一协调域单一有效权威，无冲突工作继续 | apps/ora-node/src/tests/local_recovery.rs::cleanup_failure_does_not_block_unrelated_work_or_release_conflicting_resources 及真实 guardian 丢失测试覆盖局部准入；并行共享仓库协调仍有缺口，跨 Node 延期 | `Partial` |

### 决策依据

[根决策](../../../../decisions/node/process/integration/0-shared-process-entry-and-resource-handoff.md) D3、D4 与不变量 2、3。
