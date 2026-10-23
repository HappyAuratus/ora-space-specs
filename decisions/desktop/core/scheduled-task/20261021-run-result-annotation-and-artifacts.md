---
status: implemented
---

# Scheduled Run 携带独立业务结论，运行产物脱离 worktree 生命周期持久化

本 ADR 处于 `implemented`：verdict（desktop#718）与运行产物（desktop#719）已合入 main；运行级接线、终态拒绝与读取侧链接安全的
直接证据由 desktop#728（`0ff3baf2`）/ #731（`0ae6849a`）补齐。按 circus#502 的 owner 结论完成的真实 local Nightly QA 已通过：
desktop main `bea390eb`、agent `local/ora-space.claude`、run `72d91c84`，哨兵解析为 `verdict=pass`、产物 `report.md`（46 字节）
采集成功、重启 Desktop 后 verdict 与产物仍可读。它在根决策 D7 与紧邻的后续 ADR
`20261020-worktree-ownership-atomic-commit.md` 之上，为质量工程方向的 Nightly QA 补齐两件能力：①运行除执行层状态外还能携带
机器可读的业务结论（verdict）；②运行产出的报告/测试结果脱离隔离 worktree 的生命周期持久化，按 Run ID 索引、随运行历史访问。
它不改动执行层状态机与 `failure_reason` 语义；Cloud 侧仅冻结接口边界与设计约束、不在本地轮实现。涉及模块：desktop 的
scheduled-run domain/repository、runtime、前端运行历史；specs 的后续 ADR 与核心用例。

## 继承与修改

前序决策：根决策 [0-local-and-cloud-execution-ownership.md](0-local-and-cloud-execution-ownership.md)（下称根决策）与紧邻
后续 [20261020-worktree-ownership-atomic-commit.md](20261020-worktree-ownership-atomic-commit.md)（下称 20261020）。

| 约定 | 处置 |
| --- | --- |
| 根决策 D7：运行状态机与 `failure_reason` 枚举 | 继承，不变；业务结论不写入、不复用这些字段 |
| 根决策 D7：每任务 100 条历史、保留 worktree 链接的运行例外 | 继承，不变；产物保留随历史裁剪联动（D3） |
| 根决策 D3：调度器不得自行拼装未版本化 Prompt | 继承，不变；本 ADR 的 runtime 只**解析** Agent 最终响应，不注入任何 prompt 内容（哨兵是用户写在任务 prompt 里的可选约定） |
| 根决策 D4：运行结束无改动即删、有改动保留（10 个上限） | **修改**：产物固定目录及其提交内容不计入 worktree 改动判定（D3），故「只写了产物」的运行仍判无改动、正常回收；其余保留判定不变 |
| 根决策 D8：`full_auto_in_worktree` 授权与 worktree 非沙箱 | 继承，不变；本 ADR 不扩大权限 |
| 20261020 D1/D3：归属在任务提交事务原子建立、retention 先清链再分类 | **修改**：产物清理挂在历史裁剪与链接例外上（D3）；归属事务与清链顺序本身不变 |
| 根决策 D7 隐含：运行结果只有「执行层状态 + 历史行」两种表达 | **修改**：新增独立业务 verdict 与持久化产物 |
| 根决策 D1：本地数据只驻留本机 | 继承，不变；本地产物同样只落本机（D5） |

## 决策

### D1：业务 verdict 是独立、可空、机器可读的运行结果字段

`ScheduledRun` 增加 `verdict: Option<ScheduledRunVerdict>`（`pass / warn / fail / blocked / idle / range_unresolved`）、
`verdict_detail: Option<String>`（受限自由文本）与 `verdict_at: Option<i64>`（独立的注解时间戳，不复用 `updated_at`，以免影响
worktree 保留淘汰排序）。落点覆盖 domain、`ora_contracts` 与 TS `packages/contracts` 的契约、迁移（`verdict` / `verdict_detail` /
`verdict_at` 三列可空，旧数据读作「未报告」）。

- **`status`×verdict 组合**：只有 `succeeded` 可携带 verdict（执行完整走完、Agent 有最终轮可解析）；`failed` / `cancelled` /
  `timeout` 的执行层原因已由 `failure_reason` 表达、且无 `EndTurn` 解析点，不提供 verdict；`skipped_busy` / `skipped_capacity`
  与 `missed` 无运行体、强制无 verdict；`accepted` / `running` 尚无 verdict。`status = succeeded` 且 `verdict = fail` 是合法
  且重要的状态（执行成功、业务失败）。
- **`custom` 不进入首版枚举（owner 结论，2026-10-21）**：无界自由值难聚合、难校验；Agent 需要非标准结论时写入
  `verdict_detail` 自由文本。以后需要再扩枚举、另行决策。
- 为什么独立字段：质量巡检的「发现 Critical 缺陷」不等于「执行器失败」，复用 `failure_reason` 污染执行层语义，复用 `status`
  无法表达「执行成功但业务失败」。

### D2：verdict 由 runtime 在 `succeeded` 收尾时解析 Agent 最终响应并单次写入

Agent 是 ACP 子进程，没有 desktop repository 通道，因此不存在「Agent 直接调用 repository 入口」的路径。写入权在 runtime：仅在
运行以 `EndTurn` 正常结束（映射 `succeeded`）后，runtime 从**持久会话历史**读取最终 Assistant 轮，解析文档化哨兵并调用
`annotate_scheduled_run_verdict` 单次写入；随后才发布 `RunFinished`。`failed / cancelled / timeout` 无 `EndTurn`，不写入
verdict（执行层 `failure_reason` 已充分表达，见 D1 组合矩阵）。

- **哨兵契约（冻结）**：最终 Assistant 轮文本按 LF 归一后匹配
  `QA_VERDICT: <pass|warn|fail|blocked|idle|range_unresolved>`（标记名与取值大小写不敏感、行尾空白忽略、以换行为界不跨行、取
  **第一个**匹配，fence 代码块内与被拒绝的示例文本也参与匹配）；可选紧随一行 `QA_DETAIL: <自由文本>`，detail 长度上限
  `MAX_SCHEDULED_RUN_VERDICT_DETAIL_CHARS = 2000`（字符数）。无匹配或格式非法 → 「未报告」，不报错、不改写运行事实。
- **单次写入**：一个 `succeeded` 运行只在收尾解析一次、写入一次（不是多提交模型）。枚举非法、detail 超长、运行不存在或不处于
  可注解状态时被拒且不改写运行事实。
- **prompt 落点**：哨兵是**用户写在任务 prompt 里的可选约定**（属于用户自备模板，产品侧不存在内置注入点）；runtime 只解析、
  不注入，遵守根决策 D3。未采用该约定的任务 → 恒「未报告」。

### D3：运行产物经固定目录 + 类型白名单声明，copy 到本机独立存储

- **采集时机（冻结）**：仅在 `succeeded` 收尾时采集产物，顺序为「终态 transition → verdict 写入 → 产物采集 → `publish_finished`
  → `janitor.run_finished`」（采集在 janitor 丢弃 worktree 之前；采集失败记入被拒清单、不改写运行事实）。`failed / cancelled /
  timeout` 与启动恢复中被结算为 `failed` 的中断运行**不采集**（其 worktree 随既有清理回收，产物不保留——这是接受的取舍）。
- **声明机制（冻结哪些文件算产物）**：Agent 只能通过 worktree 根下的固定目录 `.ora-scheduled-artifacts/` 声明产物；类型白名单
  闭集 `markdown / json / txt / log / xml / html`。固定目录之外的任何文件不登记——这是 #440「防非授权文件收集」的落点。与
  workflow 交互式节点产出物声明（circus#426）是不同领域，不复用其机制，仅复用路径安全工具。
- **声明载体**：与 D2 哨兵同源——`.ora-scheduled-artifacts/` 目录约定是**用户写在任务 prompt 里的可选约定**，runtime 只回收该目录、
  不注入 prompt；未采用该约定的任务 → 恒无产物。
- **搬入方式与改动判定**：**copy**（跨卷安全，源文件留在 worktree）；固定目录本身从 worktree 改动判定中排除——具体到
  `git_worktree_has_changes`：`status_entries` 的全部条目（未跟踪、已暂存、已修改、已删除）与 committed diff 里位于
  `.ora-scheduled-artifacts/` 下的路径一律忽略（即便 Agent 把产物暂存或提交进分支也忽略），故「只写了产物」的运行仍判无改动、正常回收、不占用 10 个保留上限。
- **索引与崩溃对账**：按 `run_id` 建产物清单，字段为已登记产物（名称、类型、大小、登记时间）与**被拒清单**（名称 + 拒绝原因，
  供历史展示「N 个产物被拒绝」）。落盘顺序：先 copy 到存储内临时文件 → 校验大小 → 原子 rename 到最终名 → 再写清单条目
  （清单后置）。崩溃对账（启动 sweep）：无清单条目的存储内文件（含临时文件）按孤文件清理、清单指向但文件缺失/大小不符则删
  索引项；「rename 后、写清单前崩溃」丢失该单个产物，是**接受的取舍**（不留残缺数据、不产生孤清单），恢复幂等。
- **上限（冻结数值）**：单产物 ≤ 10 MiB（所有白名单类型统一，不按类型分档）、单运行 ≤ 64 个、单运行总 ≤ 64 MiB；**产物保留按每任务 ≤ 10 个运行**（独立于 worktree 保留的**跨任务全局 10**），
  按运行 `updated_at` 最旧优先淘汰，与 100 条历史裁剪联动。超限或非法路径记入**被拒清单**并随运行历史可见（不静默丢弃）。
- **清理时机**：随 100 条历史裁剪删除对应运行产物（裁剪只删索引行 → 事务提交后再删文件，失败由启动 sweep 兜底）；保留
  worktree 链接的运行其产物随链接例外保留——**链接例外优先于每任务 10 个运行上限**（链接清除后该运行产物恢复可裁）；任务
  软删除不即时删，其产物由启动 sweep 对账清理（任务已软删除或已不存在 → 删除该任务全部运行产物）；**禁用任务的产物保留**，
  不被 sweep 清理——禁用可随时恢复，其存储已受每任务 ≤ 10 个运行上限约束，清理会让「临时禁用 + 重启」静默丢失产物。

### D4：路径安全复用既有工具，敏感信息按可判定口径约束

- **路径安全**：复用 `crates/utils/src/path` 既有校验，并补 Windows ADS、大小写、NFC 归一与硬链接越界；产物名/相对路径不得含
  `..`、绝对路径、盘符；不跟随指向存储目录之外的符号链接。威胁模型以「Agent 在非沙箱 worktree 内可写任意路径」为前提。
- **敏感信息（owner 结论）**：**不做内容密钥扫描**。可判定口径：Ora 自身不往产物存储写入任何凭据（模型 Key、token、环境变量
  等）；产物是 Agent 写出的用户内容，暴露面靠固定目录 + 类型白名单 + 大小上限 + 只留本机 + 跟随运行可见性来约束。依据：worktree
  不是沙箱（根决策 D8），Agent 本就能把内容写到任意位置；规则扫描漏报率高、只带来虚假安全感。

### D5：local 与 Cloud 的边界

- 本地：产物与 verdict 只落本机、不自动上传、不跨设备同步（继承根决策 D1）。
- Cloud：verdict **枚举与本地共用**（见 D1，即 #439「Cloud 与 local 共用同一 verdict 契约」）；Cloud 侧的 verdict 写入通道、
  产物存储、权限、生命周期与费用归属**由 circus#455 承接**（需要时再出 Cloud ADR），不阻塞本 ADR（IssueRun ≠ ScheduledRun，
  需显式定义）。

## 不变量

1. `status` 与 `failure_reason` 的语义、映射、通知与历史裁剪不回退。
2. verdict 是独立、可空、机器可读字段；`status = succeeded` 且 `verdict = fail` 可合法并存。
3. verdict 只有 runtime 在收尾时能写；单次收尾单次写入；非法输入被拒且不改写运行事实；`verdict_at` 独立于 `updated_at`。
4. 运行产物在 worktree 被回收后仍可从运行历史访问；产物固定目录排除于 worktree 改动判定。
5. 产物大小/数量受冻结上限约束，越界或路径穿越被拒绝且可见。
6. 本地产物与 verdict 不离开本机；Cloud 侧写入通道与产物存储由 circus#455 承接，不在本地实现。
7. 旧任务与旧历史无 verdict/产物时行为不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 复用 `failure_reason` 表达业务失败 | 实现最快，但污染执行层语义：`failed` 触发失败通知、进入故障统计，且无法表达「执行成功但业务 FAIL」 |
| 在 `status` 里加业务终态 | 状态机与 D6/D7 语义、通知、裁剪全被打乱，`succeeded` 含义被稀释 |
| 给 Agent 提供专用 verdict 工具/命令 | 结构化更可靠，但需在 agent runtime 增加工具接线与授权面；对「每次运行只写一次结论」的单一用途成本高于收益（作为哨兵漂移时的升级路径保留） |
| 暴露独立公开端点由前端/外部直写 | 破坏「只有 runtime 能写」的信任边界，打开任意客户端写入 |
| 产物留在 worktree、依赖「有改动即保留」 | 无改动即回收、有改动仅临时副本，无索引、无大小上限、无保留策略 |
| 产物用 move 而非 copy | 跨卷 `fs::rename` 不可用，且搬走会改变 worktree 保留判定 |
| verdict 驱动自动重试/告警路由 | 属另一决策面（调度策略、通知语义、费用归属），本次不绑定 |
| 建通用文件资产管理/版本化产物仓库 | 超出定时运行产物的需求，复杂度与安全面大幅扩大 |

## 本决策未解决的问题

- verdict 驱动的自动重试、任务启停或告警路由（另行决策）。
- 运行历史按 verdict 过滤/排序、聚合看板。
- Cloud 侧 verdict 写入通道与产物存储、权限、生命周期、费用归属（circus#455 承接，见 D5；IssueRun ≠ ScheduledRun 需显式
  定义）。
- 哨兵解析若在实践中 Agent 不守约、格式漂移，可升级为专用结构化工具（另立决策）。

## 核心测试用例

本 ADR 的核心用例维护在
[`test-cases/desktop/core/scheduled-task/run-result-annotation-and-artifacts.md`](../../../../test-cases/desktop/core/scheduled-task/run-result-annotation-and-artifacts.md)，
覆盖：verdict 独立于执行层状态、只有 `succeeded` 可携带、先于 `RunFinished` 单次写入；哨兵解析规则与「未报告」降级、runtime
不注入 prompt；非法枚举/detail 超长/非可注解运行被拒且不改写事实；`verdict_at` 不推进 `updated_at`；产物只在 `succeeded`
收尾且早于 janitor 采集、固定目录与白名单闭集、采集失败进被拒清单；产物目录不计入改动判定；超限/路径越界被拒且可见；落盘
顺序与崩溃对账；每任务 10 个运行上限与链接例外；历史裁剪、软删除任务的 sweep 清理与禁用任务保留；Ora 不往产物存储写凭据；
本地产物与 verdict 不离开本机。

## 落地顺序

1. **specs**：本 ADR 评审通过并合并，核心测试用例建立（义务初始 `Missing`），`domain/glossary.md` 补「业务 verdict / 运行产物」术语。
2. **#439 实现**：verdict 已合入 main（desktop#718）。
3. **#440 实现**：产物持久化已合入 main（desktop#719）。
4. **证据补齐**：运行级接线与终态拒绝（desktop#731）、读取侧链接安全（desktop#728）已合入 main，对应义务标 `Covered`。
5. **收尾**：真实 local Nightly QA 已完成并写入核心用例证据（desktop main `bea390eb`、agent `local/ora-space.claude`、run `72d91c84`、`verdict=pass`、产物 `report.md` 46 字节、重启后可读），本 ADR 转 `implemented`。
