# Scheduled Run 结果注解与产物持久化

本用例服务后续 ADR
[20261021-run-result-annotation-and-artifacts.md](../../../../decisions/desktop/core/scheduled-task/20261021-run-result-annotation-and-artifacts.md)
（决策单 ora-space/circus#441，覆盖事项 #439 / #440），覆盖运行结果注解与产物持久化的验证义务。

## Business Verdict Must Be Independent And Machine Readable

- 风险：业务结论复用执行层状态，破坏 `status/failure_reason` 语义、故障映射与通知，或无法表达「执行成功但业务失败」。
- 前置状态：运行已 `succeeded`；Agent 最终响应含或不含哨兵。
- 触发：runtime 在 `EndTurn` 收尾解析最终 Assistant 轮并写入 verdict；或旧任务运行无 verdict 被读取。
- 必须成立：`verdict` 独立、可空、取值 `pass/warn/fail/blocked/idle/range_unresolved` 且带受限 detail 与独立 `verdict_at`；
  `status=succeeded` 与 `verdict=fail` 合法并存；`failed/cancelled/timeout/skipped_*` 与 `missed` 均无 verdict；`accepted/running`
  无 verdict；verdict 在 `RunFinished` 发布前写入（前端刷新即见）；旧运行无 verdict 显示「未报告」且行为不变。
- 禁止结果：verdict 触发失败通知或进入故障统计；复用 `failure_reason`；`verdict_at` 改写 `updated_at`；缺 verdict 时行为回退。
- 决策依据：后续 ADR D1、D2 与不变量 1–3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `succeeded + verdict=fail` 在历史中独立展示，不触发失败语义 | 尚未实现 | `Missing` |
| 哨兵解析：首个匹配生效、多余标记忽略、大小写不敏感规范化 | 尚未实现 | `Missing` |
| 非法枚举 / detail 超长 / 运行不存在被拒且不改写事实 | 尚未实现 | `Missing` |
| `failed/cancelled/timeout/skipped_*` 与 `missed` 均无 verdict | 尚未实现 | `Missing` |
| verdict 先于 `RunFinished` 落库，前端刷新即见最终 verdict | 尚未实现 | `Missing` |
| `verdict_at` 独立，不推进 `updated_at`（不改变保留淘汰排序） | 尚未实现 | `Missing` |
| 旧任务无 verdict 显示「未报告」，历史裁剪/通知/保留不回退 | 尚未实现 | `Missing` |

## Artifacts Must Outlive The Provisioning Worktree

- 风险：产物停留在隔离 worktree，回收后丢失；或未受声明/大小/数量约束而膨胀或越权收集。
- 前置状态：Agent 在固定目录 `.ora-scheduled-artifacts/` 产出白名单类型文件；运行收尾。
- 触发：`succeeded` 收尾时 runtime 登记并 copy 产物；worktree 随后被回收；用户从运行历史打开产物。
- 必须成立：只在 `succeeded` 收尾采集，且采集早于 janitor 丢弃 worktree（`failed/cancelled/timeout` 与中断运行不采集）；
  固定目录 + 类型白名单之外的文件不登记；copy 后 worktree 保留判定不变（无改动仍可回收）；按 `run_id` 建立清单
  （已登记：名称/类型/大小/登记时间）与被拒清单（名称 + 原因），可列出/打开/下载；单产物 ≤10 MiB、单运行 ≤64 个、总 ≤64 MiB、
  每任务保留 ≤10 个运行的产物（独立于 worktree 全局 10、按最旧运行淘汰）；超限或非法路径进入被拒清单且可见；落盘顺序
  copy→临时文件→校验大小→原子 rename→写清单，崩溃对账幂等（孤文件清理、缺文件删索引、半写不残留）；产物随 100 条历史裁剪
  清理，保留 worktree 链接运行例外保留；软删除任务的产物由启动 sweep 清理，禁用任务的产物保留。
- 禁止结果：产物随 worktree 一起被回收；无索引或被拒不可见；目录外文件被登记；路径穿越/符号链接越界；清单指向半写/截断文件。
- 决策依据：后续 ADR D3、D4 与不变量 4、5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 采集早于 janitor 丢弃：只写了产物的运行判定无改动仍回收，但产物先落库 | 尚未实现 | `Missing` |
| `failed/cancelled/timeout` 与启动恢复结算为 `failed` 的中断运行不采集（取舍） | 尚未实现 | `Missing` |
| 固定目录 + 类型白名单之外的文件不登记 | 尚未实现 | `Missing` |
| 登记后 worktree 被回收，产物仍可从运行历史访问；copy 不改变保留判定 | 尚未实现 | `Missing` |
| 超限 / 路径穿越 / 符号链接越界记入被拒清单且可见 | 尚未实现 | `Missing` |
| 落盘顺序（临时文件 + 大小校验 + 原子 rename + 清单后置）崩溃对账幂等（含 rename 后写清单前崩溃的取舍） | 尚未实现 | `Missing` |
| 每任务 ≤10 个运行的产物按最旧运行淘汰，独立于 worktree 全局 10；链接例外优先于 10 上限 | 尚未实现 | `Missing` |
| 产物随历史裁剪清理（先删索引行、事务提交后删文件，失败由启动 sweep 兜底） | 尚未实现 | `Missing` |
| 软删除或已不存在的任务的产物由启动 sweep 清理；禁用任务的产物保留、不被 sweep 清理 | 尚未实现 | `Missing` |
| 旧运行无产物显示「无产物」；访问跟随运行可见性；清单只存元数据 | 尚未实现 | `Missing` |

## Local Artifacts And Verdicts Must Stay On The Device

- 风险：本地产物/verdict 被意外加入 Cloud 同步或上传。
- 前置状态：local 定时运行产出 verdict 与产物。
- 触发：登记、读取、重启。
- 必须成立：产物与 verdict 只落本机、不自动上传、不跨设备同步；重启后历史与产物仍可读。
- 禁止结果：把本地产物写入账号数据 API 或 Cloud 上传路径。
- 决策依据：后续 ADR D5 与不变量 6、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 产物/verdict 存储不调用 Cloud 上传/同步；重启后历史与产物可读 | 尚未实现 | `Missing` |
| 本地产物随运行历史驻留本机，与 D1 数据驻留一致 | 尚未实现 | `Missing` |