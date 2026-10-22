# 工作流节点重试与恢复

依据[工作流节点重试与恢复契约](../../../decisions/cloud/workflow/20261021-node-retry-and-resume-contract.md)
（`approved`，D1–D8）。这些用例覆盖 agent 节点重试策略的发布门禁、可重试失败的有界等待、按尝试隔离的会话
证据、取消与预算耗尽的结算，以及 resume 首期在接口上的明确不可用；调度器整体行为见
[调度串行与恢复](scheduler-serialism-and-recovery.md)，单个尝试内的会话结束见
[工作流会话结束与有界等待](session-end-after-initial-turn.md)。

实现由 [ora-space/circus#460](https://github.com/ora-space/circus/issues/460) 承接，尚未合入 main，cloud 侧义务
全部为 `Missing`，随实现逐项登记直接测试；desktop 的 `engine/retry.rs` 只是策略形状与数值的参考实现，不能代替
Cloud 在真实 PostgreSQL 上的证据。

## The Publish Gate Validates The Retry Policy Instead Of Refusing It

- 风险：门禁继续拒绝 `retry`，作者无法发布重试策略；或者放行非法对象、部分字段，冻结一份作者没看见过的
  策略；缺省值与 desktop 不一致，同一张图在两端行为不同。
- 前置状态：草稿的 agent 节点 `agentConfig.retry` 缺失、为 `null`、为合法对象，或为非对象、缺字段、类型或
  范围不符的值。
- 触发：发布该草稿；运行一份没有 `retry` 的历史快照。
- 必须成立：缺失或 `null` 等于默认策略（启用、2 次、10 秒起）；出现时必须是对象且 `enabled`、
  `maxRetries`（`0..=5`）、`initialDelaySeconds`（`0..=300`）全部合法；非法时按
  `agent_retry_not_an_object` / `agent_retry_enabled_invalid` / `agent_retry_max_retries_invalid` /
  `agent_retry_initial_delay_invalid` 拒绝，带 `nodeId` 与 `field`；未知键忽略；保存草稿仍然宽松；历史快照
  不重新过门禁，按默认策略执行。
- 禁止结果：合法策略被拒绝；缺字段时用默认值补齐后放行；非法值被冻结进快照；历史快照因新门禁无法运行。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 合法对象发布成功，缺失与 `null` 冻结为默认策略的语义 | 真实 PostgreSQL：发布含三种写法的图，按快照执行时读到同一默认策略 | Missing |
| 非对象、缺字段、类型或范围不符各返回命名错误并定位节点与字段 | 发布门禁表驱动测试：每个错误码带 `nodeId`、`field` | Missing |
| 等待时长为 `initialDelaySeconds × 2^(n-1)`，单次不超过 600 秒 | 策略单元测试：边界值（0、300、第 5 次重试）与上限截断 | Missing |
| 没有 `retry` 的历史快照不重新过门禁，失败时按默认策略重试 | 真实 PostgreSQL：直接写入旧形状快照后运行，观察到第 2 次尝试 | Missing |

- 决策依据：D1、D8（历史快照的行为变化，owner 2026-10-21 确认），根决策 D2 的修改项。

## A Retryable Failure Is Replaced By A Committed Bounded Wait

- 风险：可重试的失败直接失败运行；或者不可重试的原因也被重试，掩盖崩溃循环、成倍延长资源占用；
  `retrying` 落到 pass 结尾的"没有失败节点就成功"分支，运行被误结算为 `succeeded`；下游节点在等待期间
  提前就绪。
- 前置状态：运行的一个 agent 节点正在执行，策略启用且预算未耗尽；同一运行还有该节点的下游节点。
- 触发：该尝试的会话结果原因分别为 `agent_failed`、`interrupted`、`cancelled`、`user_ended`、
  `idle_timeout`；Cloud 判定 `session_*_deadline_exceeded`、`node_unavailable`、`workspace_unavailable`；
  调度器在 `retryAt` 之前多次 pass。
- 必须成立：只有 `agent_failed` 使节点进入 `retrying`，带 `attempt`、数据库时钟算出的 `retryAt` 与这次失败
  的原因；判定读原因字符串的闭集，先于状态映射；其余失败原因按既有路径失败，成功原因按既有路径成功；
  等待期间运行保持 `running`、`error` 为空、Workspace 不释放、不产生执行工作项；存在 `retrying` 节点时本轮
  不派发同一运行的其他节点，下游保持未就绪；`retryAt` 之前的 pass 原样返回，绝不进入终态结算。
- 禁止结果：`interrupted`、deadline、宿主或配置类失败被重试；未知原因因"映射为成功"被跳过判定；
  `retrying` 期间运行被结算为 `succeeded` 或 `failed`；下游节点或兄弟 agent 节点在等待期间被派发；等待期间
  Workspace 被声明删除。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `agent_failed` 进入 `retrying`，记录 `attempt`、`retryAt` 与原因 | 真实 PostgreSQL：提交 `agent_failed` 会话结果后检查 `node_states` | Missing |
| 闭集之外的失败原因与 Cloud 自判失败都不重试，按既有路径失败 | 真实 PostgreSQL：逐个原因驱动，检查节点与运行终态、无第 2 次尝试 | Missing |
| `retrying` 期间的 pass 不进入终态结算，运行保持 `running` 且 `error` 为空 | 真实 PostgreSQL：`retryAt` 之前多次 pass，运行状态与版本不变 | Missing |
| 等待期间不派发同一运行的其他节点，下游保持未就绪 | 真实 PostgreSQL：含下游与兄弟 agent 节点的图，等待期间无新执行工作项 | Missing |
| 等待期间 Workspace 不释放 | 真实 PostgreSQL：等待期间无 Workspace 删除声明，失败或取消结算后才出现 | Missing |

- 决策依据：D2、D3，不变量 3、4，根决策 D4 与不变量 5。

## A Waiting Retry Wakes Only From Its Committed Due Time

- 风险：等待依赖进程内定时器，崩溃或换副本后节点永久停在 `retrying`；多副本对到期时刻看法不同，重复启动
  下一次尝试；重启重置预算或等待。
- 前置状态：节点处于 `retrying`，`retryAt` 已提交。
- 触发：`retryAt` 前后的 pass；调度器重启后的首个 pass；两个副本同时 pass；会话结果已提交但进程在提交
  `retrying` 前崩溃。
- 必须成立：只有 `retryAt <= clock_timestamp()` 时才启动下一次尝试，启动沿用既有的 agent 节点启动路径，节点
  回到 `running`、`attempt` 加一、`retryAt` 清空；到期判定只读已提交的 `retryAt`，重启与多副本读到同一时刻；
  并发 pass 由运行级排他事务与版本 CAS 串行化，只启动一次；崩溃后下一次 pass 重新判定，等待只会变长，节点
  不会永久卡住；预算随节点状态提交，重启与重放不重置。
- 禁止结果：到期前启动；到期后永不启动；两个副本各启动一次尝试；重启后 `attempt` 归零或 `retryAt` 被提前。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 到期前不启动、到期后启动且 `attempt` 加一 | 真实 PostgreSQL：调整 `retryAt` 前后各 pass 一次，检查节点状态与执行工作项 | Missing |
| 并发 pass 只启动一次下一次尝试 | 真实 PostgreSQL：并发调用调度 pass，检查执行工作项与 `node_states` 版本 | Missing |
| 重启或崩溃后按已提交状态继续，预算不重置 | 真实 PostgreSQL：新建调度器实例对同一运行 pass；模拟结果已提交、`retrying` 未提交的状态 | Missing |

- 决策依据：D3，不变量 2、5，根决策条件②（无状态调度）与不变量 3。

## Only The Current Attempt Can Settle Its Node

- 风险：按节点唯一的约束让第 2 次尝试无法登记；按节点读取证据时读到第 1 次的失败结果，第 2 次尝试在途时
  节点被误结算；第 1 次尝试的结束命令被投递给第 2 次的会话；被取代尝试的迟到结果改写投影。
- 前置状态：同一（运行，节点）已有第 1 次尝试的执行行、会话结果与可能存在的 `end_session` 命令；第 2 次
  尝试已启动。
- 触发：第 2 次尝试登记执行与回执；第 1 次尝试的结果或回执迟到、重复送达；第 2 次尝试成功回合后放出结束
  命令。
- 必须成立：`execution_work`、`node_executions`、`thread_commands` 按 `workflow_node_attempt` 区分，第 2 次尝试
  能登记；attempt 由 Cloud 从被登记的执行工作项复制，Controller 线上形状不变；会话证据、`end_session` 的
  查找与放出、取消路径的证据读取都限定当前尝试；每个（运行，节点，尝试）至多一条 `end_session`，跨尝试
  投递被拒绝；被取代尝试的迟到结果只写自己的执行行，对投影是空操作。
- 禁止结果：第 2 次尝试因唯一约束登记失败；节点被第 1 次的证据结算；结束命令越过尝试投递；迟到结果推进或
  回退节点状态；线上协议出现 attempt 字段。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 迁移后存量行为 attempt 1，同一节点的第 2 次尝试能登记执行与结束命令 | 真实 PostgreSQL：升级迁移测试 + 两次尝试的登记 | Missing |
| 第 2 次尝试在途时只读当前尝试的证据，节点不被第 1 次结果结算 | 真实 PostgreSQL：第 2 次尝试在途时 pass，节点保持 `running` | Missing |
| `end_session` 按尝试唯一，跨尝试投递被拒绝 | 真实 PostgreSQL：两次尝试各一条命令，跨尝试投递返回冲突 | Missing |
| 被取代尝试的迟到或重复结果不改变投影 | 真实 PostgreSQL：第 2 次尝试后重放第 1 次的结果，`node_states` 不变 | Missing |
| Controller 线上形状不变 | 契约测试：claim、登记与回执的 proto/JSON 不含 attempt | Missing |

- 决策依据：D4，不变量 1、7、9，根决策不变量 3 的扩展，会话结束决策不变量 2 的修改。

## Cancellation Overrides A Waiting Retry

- 风险：取消请求后仍启动新尝试；取消要等到 `retryAt` 才生效；取消与到期唤醒竞争时出现两个结果。
- 前置状态：节点处于 `retrying`，`retryAt` 未到或已到；或节点 `running`。
- 触发：用户请求取消，随后 pass；取消请求与到期唤醒并发。
- 必须成立：取消分支先于重试分支：`retrying` 节点提交为 `cancelled`（清空 `retryAt`、写 `finishedAt`），运行
  结算 `cancelled` 并释放 Workspace，即使 `retryAt` 已到也不启动新尝试；节点 `running` 时走既有的结束会话
  路径；并发时先提交的一方决定结果。
- 禁止结果：取消后出现新的执行工作项；取消等待退避周期；节点与运行的终态不一致。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 等待中取消立即结算 `cancelled` 并释放 Workspace，不启动新尝试 | 真实 PostgreSQL：`retryAt` 已到期时请求取消后 pass，检查终态、无执行工作项与 Workspace 删除声明 | Missing |
| 取消与到期唤醒并发只产生一种结果 | 真实 PostgreSQL：并发取消与 pass，检查节点状态与执行工作项二者择一 | Missing |

- 决策依据：D5，不变量 6，根决策 D6 / D7。

## An Exhausted Budget Fails The Run With The Last Attempt's Evidence

- 风险：预算耗尽后继续重试；最终失败丢掉最后一次的原因，或新增未定义的失败原因。
- 前置状态：节点的 `attempt` 已达 `maxRetries + 1`，或策略 `enabled=false` / `maxRetries=0`。
- 触发：该尝试以 `agent_failed` 结束。
- 必须成立：节点以最后一次尝试的原因提交 `failed`，保留 `attempt`、清空 `retryAt`；运行以同一原因失败并按
  既有路径释放 Workspace；不再有新尝试。
- 禁止结果：超过 `maxRetries + 1` 次尝试；运行 `error` 与最后一次原因不一致；新增失败原因或运行状态。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 总尝试数为 `maxRetries + 1`，最后一次失败结算运行 | 真实 PostgreSQL：`maxRetries=2` 连续失败三次，检查尝试数、原因与终态 | Missing |
| `enabled=false` 与 `maxRetries=0` 首次失败即结算 | 真实 PostgreSQL：两种策略各一次失败，无 `retrying` | Missing |

- 决策依据：D6，不变量 4，根决策 D4 的修改项。

## Resume Is Explicitly Unavailable In The First Phase

- 风险：resume 接口不存在，客户端无法区分"没有能力"和"路径写错"；或接口改写了运行、节点或 Workspace
  状态；页面提供了不能用的入口；授权比取消宽松。
- 前置状态：一个 `failed` 的工作流运行；请求者是发起人、租户管理员、其他成员或其他租户。
- 触发：`POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/resume`，请求体合法、含未知字段或 `mode` 非法。
- 必须成立：接口已注册并进入 OpenAPI；授权与取消相同（无权限 403 `workflow_run_resume_forbidden`，不存在或
  不属于该租户 / 工作流 404 `not_found`）；请求体按最终契约校验（未知字段 400 `unknown_field`，非法 `mode`
  400 `invalid_field_value`）；形式合法的请求一律 501 `workflow_run_resume_unavailable`，不改运行、节点、
  Workspace，也不写幂等记录；运行页不渲染恢复入口。
- 禁止结果：路由 404；合法请求改变任何持久状态；成员绕过授权；页面出现恢复按钮。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 合法请求返回 501 命名错误，运行、节点与 Workspace 不变 | 真实 PostgreSQL：请求前后比较运行行、`node_states` 与 Workspace 声明 | Missing |
| 授权与取消一致，其他成员 403、其他租户 404 | 接口测试：四类请求者 | Missing |
| 未知字段与非法 `mode` 按最终契约拒绝 | 接口测试：请求体校验 | Missing |
| OpenAPI 含该接口与错误码，运行页无恢复入口 | OpenAPI 漂移检查；前端测试断言无入口 | Missing |

- 决策依据：D7，不变量 8，[ora-space/circus#448](https://github.com/ora-space/circus/issues/448) 的 owner 结论。
