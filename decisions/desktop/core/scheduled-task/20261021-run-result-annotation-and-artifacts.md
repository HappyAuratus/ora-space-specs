---
status: proposed
---

# Scheduled Run 携带独立业务结论，运行产物脱离 worktree 生命周期持久化

本 ADR 处于 `proposed`。它在根决策 D7（运行记录与历史）之上，为质量工程方向的 Nightly QA 补齐两件能力：①运行
除执行层状态外还能携带机器可读的业务结论（verdict）；②运行产出的报告/测试结果脱离隔离 worktree 的生命周期持久化，
按 Run ID 索引、随运行历史访问。它不改动执行层状态机与 `failure_reason` 语义，Cloud 侧仅冻结接口边界与设计约束、不在
本地轮实现。涉及模块：desktop 的 scheduled-run domain/repository、runtime、前端运行历史；specs 的后续 ADR 与核心用例。

## 继承与修改

前序决策：根决策 [0-local-and-cloud-execution-ownership.md](0-local-and-cloud-execution-ownership.md)（下称根决策）。

| 约定 | 处置 |
| --- | --- |
| 根决策 D7：运行状态机 `accepted/running/succeeded/failed/cancelled/skipped_busy/skipped_capacity` 与 `failure_reason` 枚举 | 继承，不变；业务结论不写入、不复用这些字段 |
| 根决策 D7：每个任务保留最近 100 条历史、仍带保留 worktree 链接的运行例外 | 继承，不变；产物保留随历史裁剪联动（D3） |
| 根决策 D7：通知是交互辅助、不是运行事实源 | 继承，不变；verdict 不触发通知（另行决策） |
| 根决策 D7 隐含：运行结果只有「执行层状态 + 历史行」两种表达 | **修改**：新增独立业务 verdict 与持久化产物，两者与执行层状态并存 |
| 根决策 D1：本地任务及其运行历史只保存在本机、不上传 Cloud | 继承，不变；本地产物同样只落本机（D5） |

## 决策

### D1：业务 verdict 是独立、可空、机器可读的运行结果字段

`ScheduledRun` 增加 `verdict: Option<ScheduledRunVerdict>`，取值 `pass / warn / fail / blocked / idle /
range_unresolved`，并可携带 `verdict_detail: Option<String>`（受长度约束的自由文本）。verdict 与 `status`、
`failure_reason` 并存且互不推导：`status = succeeded` 且 `verdict = fail` 是合法且重要的状态（执行成功、业务判定失败）。

- 为什么独立字段：质量巡检的「发现 Critical 缺陷」不等于「执行器失败」，复用 `failure_reason` 会污染执行层语义、
  破坏既有故障映射、通知与统计；复用 `status` 则无法表达「执行成功但业务失败」。
- 可空与兼容：旧任务、旧历史无 verdict 时读作「未报告」，列表/详情显示「未报告」，行为不变；旧 reader 忽略新字段。
- 合法性：verdict 只在 `accepted` 之后的运行行上写入；对 `missed`（非运行）不提供 verdict。

### D2：verdict 由执行 Agent 经可信入口写入，末次写入生效

runtime 提供明确入口（repository 方法 `annotate_scheduled_run_verdict(run_id, verdict, detail)`），由执行 Agent 在
会话内通过该入口提交；入口做校验：run 存在且非 `missed`、枚举合法、detail 长度受限，非法输入被拒绝且不改写运行事实。

- 可信来源：只有执行该运行的 runtime/Agent 能写（与运行行关联，不开放任意客户端直写）。
- 冲突处理：同一次运行可多次提交，**末次写入生效**（记录 `updated_at`），便于 Agent 在收尾时覆盖早先的临时结论；
  不接受「部分成功」——一次提交要么整体写入、要么被拒。
- 缺失降级：无提交即「未报告」；不因缺 verdict 影响历史裁剪、保留或通知。

### D3：运行产物持久化到本机独立存储，按 Run ID 索引

本地为定时运行建立独立的产物存储（应用数据目录下、按 `run_id` 分目录），生命周期不依赖隔离 worktree：Agent 在
worktree 内产出 Markdown/JSON/测试报告/日志后，runtime 在运行收尾时把这些产物**登记并搬入**持久化存储，随后
worktree 是否被回收都不影响产物。

- 索引：按 `run_id` 建立产物清单（名称、类型、大小、登记时间），运行历史可列出、打开/下载。
- 上限：单个产物大小、单运行产物数量与总体大小有明确上限；超限或非法路径被拒绝，不静默丢弃。
- 保留：产物随运行历史裁剪联动——被 100 条历史裁剪的运行，其产物一并清理；仍带保留 worktree 链接的运行例外保留。

### D4：路径安全与敏感信息防护

- 路径安全：产物文件名/相对路径不得含 `..`、绝对路径、盘符；解析时不跟随指向存储目录之外的符号链接；拒绝越界。
- 敏感信息：登记与索引不抓取文件内容，仅存元数据；Agent 不得把密钥/token 写入产物（约束写入到 Prompt/使用说明），
  存储不承诺对内容做密钥扫描（Non-goal）。
- 访问：产物读取跟随运行历史可见性，本机访问；不开放任意文件系统路径读取。

### D5：local 与 Cloud 的存储边界

本地：产物与运行历史同机驻留、不自动上传、不跨设备同步（与根决策 D1 一致）。Cloud：产物走 Cloud 侧对象存储，权限与
可见性沿用运行归属/可见性契约，作为未来能力在 Cloud ADR 另行定义；本 ADR 只冻结「Run ID 索引、类型、上限、保留」等
共用语义，本地实现不得预埋上传或同步。

## 不变量

1. `status` 与 `failure_reason` 的语义、映射、通知与历史裁剪不回退。
2. verdict 是独立、可空、机器可读字段；`status = succeeded` 且 `verdict = fail` 可合法并存。
3. verdict 只有执行该运行的 runtime/Agent 能写；非法输入被拒且不改写运行事实；末次写入生效。
4. 运行产物在 worktree 被回收后仍可从运行历史访问。
5. 产物大小/数量受上限约束，越界或路径穿越被拒绝。
6. 本地产物不离开本机；Cloud 产物作为未来能力，不在本地实现。
7. 旧任务与旧历史无 verdict/产物时行为不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 复用 `failure_reason` 表达业务失败 | 实现最快，但污染执行层语义：`failed` 会触发失败通知、进入故障统计，且无法表达「执行成功但业务 FAIL」 |
| 在 `status` 里加业务终态 | 状态机与既有 D6/D7 语义、通知、裁剪全被打乱，`succeeded` 的含义被稀释 |
| 产物留在 worktree、依赖「有改动即保留」 | 无改动即回收、有改动仅临时副本，无索引、无大小上限、无保留策略，不可靠 |
| verdict 驱动自动重试/告警路由 | 有价值但属另一决策面（调度策略、通知语义、费用归属），本次不绑定 |
| 建通用文件资产管理/版本化产物仓库 | 超出定时运行产物的需求，复杂度与安全面大幅扩大 |

## 本决策未解决的问题

- verdict 驱动的自动重试、任务启停或告警路由（另行决策）。
- 运行历史按 verdict 过滤/排序、聚合看板。
- Cloud 侧产物对象存储的凭据、权限、生命周期与费用归属（Cloud ADR）。
- 产物的密钥扫描/内容审计（明确为 Non-goal）。

## 核心测试用例

本 ADR 的核心用例维护在
[`test-cases/desktop/core/scheduled-task/run-result-annotation-and-artifacts.md`](../../../../test-cases/desktop/core/scheduled-task/run-result-annotation-and-artifacts.md)，
至少覆盖：`succeeded + fail` 独立展示；旧任务无 verdict 显示「未报告」且行为不变；非法枚举/detail 超长被拒且不改写事实；
末次写入生效；产物登记后 worktree 回收仍可访问；超限/路径穿越/符号链接越界被拒；产物随历史裁剪联动清理；本地产物
不离开本机。

## 落地顺序

1. **specs**：本 ADR 评审通过标记 `approved`，同步建立核心测试用例（义务初始为 `Missing`）。
2. **#439 实现**：verdict 数据模型 + 可信写入入口 + 校验/兼容 + 历史展示 + 自动化测试（产品 PR，含
   `succeeded + verdict=fail` 场景）。
3. **#440 实现**：产物持久化存储 + Run ID 索引 + 登记/搬移 + 历史查看/下载 + 上限/保留/安全 + 测试。
4. **集成验收**：真实 local 定时任务跑一次最小 Nightly QA，验证 verdict + 产物全链路、重启后历史与产物可读、安全与
   兼容测试。
5. **收尾**：核心用例证据补齐，ADR 转 `implemented`，关闭 #439/#440。