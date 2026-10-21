# Scheduled Run 结果注解与产物持久化

本用例服务后续 ADR
[20261021-run-result-annotation-and-artifacts.md](../../../../decisions/desktop/core/scheduled-task/20261021-run-result-annotation-and-artifacts.md)
（决策单 ora-space/circus#441，覆盖事项 #439 / #440），覆盖运行结果注解与产物持久化的验证义务。

## Business Verdict Must Be Independent And Machine Readable

- 风险：业务结论复用执行层状态，破坏 `status/failure_reason` 语义、故障映射与通知，或无法表达「执行成功但业务失败」。
- 前置状态：运行已接纳且已收尾（`succeeded/failed/cancelled`）；Agent 最终响应含或不含哨兵。
- 触发：runtime 收尾解析最终 Assistant 轮并写入 verdict；或旧任务运行无 verdict 被读取。
- 必须成立：`verdict` 独立、可空、取值 `pass/warn/fail/blocked/idle/range_unresolved` 且带受限 detail 与独立 `verdict_at`；
  `status=succeeded` 与 `verdict=fail` 合法并存；`skipped_busy/skipped_capacity/missed` 无 verdict；`accepted/running` 无 verdict；
  verdict 在 `RunFinished` 发布前写入（前端刷新即见）；旧运行无 verdict 显示「未报告」且行为不变。
- 禁止结果：verdict 触发失败通知或进入故障统计；复用 `failure_reason`；`verdict_at` 改写 `updated_at`；缺 verdict 时行为回退。
- 决策依据：后续 ADR D1、D2 与不变量 1–3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `succeeded + verdict=fail` 在历史中独立展示，不触发失败语义 | 尚未实现 | `Missing` |
| 哨兵解析：首个匹配生效、多余标记忽略、大小写不敏感规范化 | 尚未实现 | `Missing` |
| 非法枚举 / detail 超长 / 运行不存在（含 `missed`）被拒且不改写事实 | 尚未实现 | `Missing` |
| `skipped_*`/`missed` 强制无 verdict；`accepted/running` 无 verdict | 尚未实现 | `Missing` |
| verdict 先于 `RunFinished` 落库，前端刷新即见最终 verdict | 尚未实现 | `Missing` |
| `verdict_at` 独立，不推进 `updated_at`（不改变保留淘汰排序） | 尚未实现 | `Missing` |
| 旧任务无 verdict 显示「未报告」，历史裁剪/通知/保留不回退 | 尚未实现 | `Missing` |

## Artifacts Must Outlive The Provisioning Worktree

- 风险：产物停留在隔离 worktree，回收后丢失；或未受声明/大小/数量约束而膨胀或越权收集。
- 前置状态：Agent 在固定目录 `.ora-scheduled-artifacts/` 产出白名单类型文件；运行收尾。
- 触发：runtime 登记并 copy 产物；worktree 随后被回收；用户从运行历史打开产物。
- 必须成立：固定目录 + 类型白名单之外的文件不登记；copy 后 worktree 保留判定不变（无改动仍可回收）；按 `run_id` 建立清单可
  列出/打开/下载；单产物 ≤10 MiB、单运行 ≤64 个、总 ≤64 MiB、每任务保留 ≤10 个运行的产物；超限或非法路径被拒绝且可见；清单
  崩溃对账幂等（孤文件清理、缺文件删索引）；产物随 100 条历史裁剪清理，保留 worktree 链接运行例外保留。
- 禁止结果：产物随 worktree 一起被回收；无索引；目录外文件被登记；路径穿越/符号链接越界；无保留策略无限增长。
- 决策依据：后续 ADR D3、D4 与不变量 4、5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 固定目录 + 类型白名单之外的文件不登记 | 尚未实现 | `Missing` |
| 登记后 worktree 被回收，产物仍可从运行历史访问；copy 不改变保留判定 | 尚未实现 | `Missing` |
| 超限（单/总大小、数量）/ 路径穿越 / 符号链接越界被拒绝且可见 | 尚未实现 | `Missing` |
| 清单崩溃对账幂等（重开库后索引与文件一致） | 尚未实现 | `Missing` |
| 产物随历史裁剪清理；保留 worktree 链接运行的产物例外保留；软删除/禁用不即时删 | 尚未实现 | `Missing` |
| 访问跟随运行可见性；索引只存元数据、不索引内容 | 尚未实现 | `Missing` |

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