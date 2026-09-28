# 普通生命周期活动保护与管理员强停

依据[已批准的停止与强停决策](../../../decisions/cloud/operation/20260927-safe-stop-and-administrative-force-stop.md)。新保障均为 `Missing`；旧生命周期证据见[独立运行时生命周期](workspace-runtime-lifecycle.md)，不替代强停验收。

## Ordinary Stop Restart And Delete Never Bypass Control Or Activity Protection

- 风险：有使用权限就能误停他人工作，或项目删除成为占用旁路。
- 前置状态：运行时分别被他人会话持有、存在活动、结果未知或已空闲；项目含多个运行时。
- 触发：创建者和管理员普通停止、重启、删除独立运行时或删除项目，与新准入并发。
- 必须成立：普通操作检查使用权限、独占和活动；活动/未知/他人占用返回冲突，无部分删除。关闭准入后原子确认 idle；重启使用同一持久意图，停止确认后才启动；main 不可单独删除。
- 禁止结果：管理员普通操作绕过占用、前端拼接停止与启动、idle 与 terminate 间插入新执行。
- 验证义务与证据：**Missing** — 需真实 PostgreSQL 生命周期/准入竞争及执行端 idle 证据；项目删除覆盖全部运行时并断言无部分副作用。
- 决策依据：D1，不变量 1。

## Administrative Force Stop Is Explicit Audited And Scoped To One Runtime

- 风险：既有行政停止静默变为杀进程，强停没有原因或误终止同项目其他成员工作。
- 前置状态：目标被普通成员占用且有活动；同项目另一个运行时也运行；另有普通成员和跨租户管理员。
- 触发：强停请求、并发重复请求、同键不同输入；原发起管理员离开或关闭页面。
- 必须成立：只有目标租户当前管理员可接受新强停，要求原因、目标版本和幂等键；界面明确全部目标工作及未落盘风险。接受后持久记录独立终止责任，撤回普通资格；仅终止目标，已有责任不随发起人离开丢失。
- 禁止结果：普通成员强停、跨租户终止、改写任务 actor、依赖弹窗授权、关闭页面取消责任、改变旧 administrative_stop 语义。
- 验证义务与证据：**Missing** — 需真实 HTTP、数据库、前端及两运行时进程测试，完整检查审计和重试只有同一责任。
- 决策依据：D2、D5，不变量 2、6。

## Force Stop Can Be Accepted During Inflight Work Without Reviving Old Resources

- 风险：create/clone/plugin 卡住让强停不可用；晚到 ensure 在强停后重新创建旧写者。
- 前置状态：目标有未完成普通 operation、execution 或 effect；另一个场景为项目普通 operation 属于其他运行时。
- 触发：登记强停，丢失 ensure/terminate 响应，重启 Controller，再交付迟到成功和旧控制指令。
- 必须成立：强停意图可登记，不删除或伪造在途记录；目标不再派发普通新动作。按稳定身份恢复并覆盖可能晚到的旧资源；仅目标被终止，其他运行时 operation 不被中断。
- 禁止结果：腾空项目唯一约束、重执行未知动作、晚到结果重新开放准入、旧 ensure 恢复可写实例。
- 验证义务与证据：**Missing** — 需真实 Controller/Substrate 的终止与 ensure 竞争、崩溃恢复和磁盘写入检查；Cloud simulator 不构成生产边界证明。
- 决策依据：D3、D4，不变量 3、4、6。

## Unconfirmed Termination Blocks Handoff And Persistent Data Survives Stop

- 风险：数据库状态代替终止证据，强停误删保存代码或未知任务被伪装成功。
- 前置状态：运行时有已保存代码、Node 日志、活动进程及未知执行；终止结果暂不可达。
- 触发：普通停止、强停、重启、接管、删除数据；恢复后查询原终止 effect 并重新启动。
- 必须成立：未确认时处于终止中/待对账，禁止重启、交接和删数据。确认后保留代码及执行引用，已完成结果原样保留、被中断结果有明确证据；新启动挂载同一数据。显式删除在终止后执行，审计与清理引用保留。
- 禁止结果：仅超时就 stopped、凭新 generation 允许同目录并行写入、强停删卷、承诺内存或磁盘损坏无损恢复。
- 验证义务与证据：**Missing** — 需真实沙盒/持久卷、进程及 PostgreSQL 的停止/强停/启动/删除与节点不可达测试，分别检查业务状态、磁盘和进程。
- 决策依据：D4、D5，不变量 4、5。
