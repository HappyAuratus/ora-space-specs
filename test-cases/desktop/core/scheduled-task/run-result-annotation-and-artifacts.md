# Scheduled Run 结果注解与产物持久化

本用例服务后续 ADR
[20261021-run-result-annotation-and-artifacts.md](../../../../decisions/desktop/core/scheduled-task/20261021-run-result-annotation-and-artifacts.md)
（决策单 ora-space/circus#441，覆盖事项 #439 / #440），覆盖运行结果注解与产物持久化的验证义务。

## Business Verdict Must Be Independent And Machine Readable

- 风险：业务结论复用执行层状态，破坏 `status/failure_reason` 语义、故障映射与通知，或无法表达「执行成功但业务失败」。
- 前置状态：运行已接纳（非 `missed`），Agent 在会话内提交业务结论。
- 触发：Agent 经可信入口写入 verdict；或旧任务运行无 verdict 被读取。
- 必须成立：`verdict` 独立、可空、取值 `pass/warn/fail/blocked/idle/range_unresolved` 且带受限 detail；`status=succeeded`
  与 `verdict=fail` 合法并存；旧运行无 verdict 显示「未报告」且行为不变；`missed` 不提供 verdict。
- 禁止结果：verdict 写入触发失败通知或进入故障统计；复用 `failure_reason`；缺 verdict 时行为回退。
- 决策依据：后续 ADR D1、D2 与不变量 1–3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `succeeded + verdict=fail` 在历史中独立展示，不触发失败语义 | 尚未实现 | `Missing` |
| 非法枚举 / detail 超长被拒且不改写运行事实 | 尚未实现 | `Missing` |
| 末次写入生效；同运行多次提交最终取最后一次 | 尚未实现 | `Missing` |
| 旧任务无 verdict 显示「未报告」，历史裁剪/通知/保留不回退 | 尚未实现 | `Missing` |

## Artifacts Must Outlive The Provisioning Worktree

- 风险：产物停留在隔离 worktree，回收后丢失；或未受大小/数量/保留约束而膨胀。
- 前置状态：Agent 在 worktree 产出 Markdown/JSON/测试报告/日志；运行收尾。
- 触发：runtime 登记并搬移产物；worktree 随后被回收；用户从运行历史打开产物。
- 必须成立：worktree 被回收后产物仍可访问；按 `run_id` 建立清单可列出/打开/下载；单个与总体大小、数量受上限；超限或
  非法路径被拒绝；产物随 100 条历史裁剪联动清理，保留 worktree 链接的运行例外保留。
- 禁止结果：产物随 worktree 一起被回收；无索引；路径穿越/符号链接越界；无保留策略无限增长。
- 决策依据：后续 ADR D3、D4 与不变量 4–6。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 登记后 worktree 被回收，产物仍可从运行历史访问 | 尚未实现 | `Missing` |
| 超限（大小/数量）/ 路径穿越 / 符号链接越界被拒绝 | 尚未实现 | `Missing` |
| 产物随历史裁剪清理；保留 worktree 链接运行的产物例外保留 | 尚未实现 | `Missing` |

## Local Artifacts Must Stay On The Device

- 风险：本地产物被意外加入 Cloud 同步或上传。
- 前置状态：local 定时运行产出产物。
- 触发：登记、读取、重启。
- 必须成立：产物只落本机、不自动上传、不跨设备同步；重启后历史与产物仍可读。
- 禁止结果：把本地产物写入账号数据 API 或 Cloud 上传路径。
- 决策依据：后续 ADR D5 与不变量 6、7。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 产物存储不调用 Cloud 上传/同步；重启后历史与产物可读 | 尚未实现 | `Missing` |
| 本地产物随运行历史驻留本机，与 D1 数据驻留一致 | 尚未实现 | `Missing` |