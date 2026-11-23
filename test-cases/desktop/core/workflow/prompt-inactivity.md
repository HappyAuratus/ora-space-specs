# 工作流 prompt 静默策略

以下用例依据[工作流有限与无限等待决策](../../../../decisions/desktop/core/workflow/20261123-workflow-agent-wait-limit.md)。已有证据来自此前实施的[静默策略](../../../../decisions/desktop/core/workflow/20261008-prompt-inactivity-policy.md)。新策略的直接证据在 Desktop 实施后登记。

## A Silent Waiting Prompt Remains Cancellable And Session Local

**风险与前置状态**　同一连接承载显式 `unbounded` 工作流与默认策略的普通会话。长时间静默可能触发意外重发，或使其他会话也失去超时保护。

**触发与预期**　让两个回合静默跨过 30 分钟及普通会话的重试窗口。无限等待回合应保持单次发送，随后可正常完成。分别触发手动停止、释放 owning stream、工作流停止、provider 断连和宿主关闭，每条路径都须收敛并释放本轮资源。普通会话应独立执行其超时策略。禁止跨会话续期、静默触发节点重试或终止后遗留路由。

| 验证义务 | 证据与边界 | 状态 |
| --- | --- | --- |
| 显式 `unbounded` 长静默后完成或取消，同连接默认会话独立超时 | 原 `wait` 测试使用旧取值。需从冻结图实际传入 `unbounded`，核对发送次数、完成结果和取消确认。 | Missing |
| 按各类终止事件清理显式无限等待回合及订阅者 | 既有生命周期用例覆盖旧策略。新取值的停止、断连、owner 释放及宿主关闭证据待登记。 | Missing |

**决策依据**　D1、D3。

## Every Workflow Turn Uses The Frozen Inactivity Policy

**风险与前置状态**　已有图、含新取值的图和已启动的运行并存。编辑草稿或升级 Desktop 时，后续发送可能误读草稿，非法值也可能被静默降级。

**触发与预期**　从保存、发布、导入和导出的图开始运行，随后编辑草稿并打开运行详情。首轮、追问、节点重试及 Loop/Iteration 的每次实际发送均采用运行的冻结字段。缺失或 `null` 使用默认策略，打开旧图不会补写字段。已发出的回合保留启动时的语义，升级后的下一轮按冻结值解释。禁止因读取、改无关字段或升级而自动选择无限等待。

| 验证义务 | 证据与边界 | 状态 |
| --- | --- | --- |
| 接受缺失、`null`、`timeout`、`wait`、`unbounded`，拒绝未知字符串和 `{"waitMinutes": N}` 并定位节点与字段 | 已有图解析测试 `prompt_inactivity_defaults_and_explicit_policies_survive_graph_parsing` 和 `rejects_invalid_prompt_inactivity_with_node_and_field_context` 覆盖原取值。新增取值及对象拒绝仍缺证据。 | Partial |
| 各轮冻结、保存发布及导入导出保留策略 | `every_human_turn_uses_the_frozen_policy_after_a_draft_edit`、`wait_policy_preserves_real_session_failures_and_frozen_policy_across_retries` 和 `graph-codec.test.ts` 覆盖既有策略。新取值的跨轮、跨版本解释待验证。 | Partial |
| 编辑器提供固定 30 分钟和明确标示需手动停止的无限等待。只读运行详情展示冻结值，有限等待显示 30 分钟 | 既有 `workflow-agent-inactivity-field.test.tsx`、`run-act-inspector.test.tsx` 覆盖旧设置和展示。两种新语义及移除分钟输入待验证。 | Partial |
| 同一在途运行升级时，已发回合不中断，下一次实际发送按冻结值启动新窗口 | 待补升级边界的直接证据。 | Missing |

**决策依据**　D1。

## Only Attributable Progress Renews A Bounded Wait

**风险与前置状态**　已发送 `wait` 回合，同时存在其他会话、同轮多个委托及权限应答。遗漏真实进展会过早取消，把重复或无关更新当作进展会让回合永久挂起。

**触发与预期**　用可控时钟在窗口内注入下列事件。正例从事件到达时刻重新计满 30 分钟，反例保留原截止。工具执行和权限写入期间计时继续。禁止重复事件延长窗口，或父子来源之间共用计划比较状态。

| 信号 | 可续期 | 保留原截止 | 直接证据 |
| --- | --- | --- | --- |
| 模型内容 | 本轮非空消息或思考 | 空内容、其他会话内容 | Missing |
| 计划 | 同一来源计划条目内容、顺序、优先级或状态变化 | 重复计划、仅 `_meta` 变化。交替发送父子计划后分别重复也不续期 | Missing |
| 父工具 | 首次报告状态、状态变化、同一工具的 `content` 或 `rawOutput` 有非空变化 | 重复状态或内容、空输出、仅修改 `title`、`kind`、`locations` | Missing |
| 子代理 | 本次发送仍在等待的具体委托通过父工具 ID 关联，所报更新符合相同进展判据 | 只知道父会话、委托不匹配或已结束、上一轮委托、重复输出 | Missing |
| 权限 | 确认归属本轮父工具，且自动应答成功写入 | 尚在写入、写入失败、子代理或来源不明 | Missing |
| 其他流量 | 无 | 会话级通知、`usage_update`、load 回放、stderr、其他会话、未知或未列入清单的扩展通知 | Missing |

**决策依据**　D2。

## A Bounded Wait Settles Once Before Node Failure Handling

**风险与前置状态**　已发出的有限等待回合带有订阅者和未决权限。静默截止临近时，正常响应、显式取消和进展可能与定时器同时就绪。旧发送尚未收敛就重试会重复产生工具副作用。

**触发与预期**　推进时钟到截止，分别控制进展的到达时刻、终态响应、取消命令和 provider 取消回执。核对 owner、订阅者、节点状态与实际发送次数。禁止双重结算、迟到更新进入下一次发送，或把用户停止记录成静默超时。

| 验证义务 | 证据与边界 | 状态 |
| --- | --- | --- |
| 正常响应与到期同时就绪时正常完成。显式取消先受理时记录用户停止，到期已结算后迟到的取消或响应不能改写终态 | 待补可控时钟下的顺序对照。 | Missing |
| 截止前到达、调度延后处理的有效进展按到达时刻续期。恰在截止和截止后到达均不续期 | 待补三个边界的时间与结果断言。 | Missing |
| 到期取消请求及未决权限。取消终态获确认时允许复用；宽限耗尽则隔离旧发送。旧发送结束或隔离后才交给节点失败策略，owner 和订阅者只结算一次 | 待补回执成功、缺失及迟到输出的直接证据。 | Missing |
| 本轮不因静默重发，保留 stale-load 恢复。非交互节点只按自身失败策略决定重试 | 待分别核对 provider 发送次数、节点尝试次数和失败原因。 | Missing |
| 交互自动首轮失败并释放执行会话，无节点重试、无停靠快照。人类回合保留会话和历史，以 `prompt_timed_out` 异常停靠，不自动重试 | [首轮确认用例](first-turn-confirmation.md)已有两种回合的异常分类。有限等待到期触发两条生产路径的证据待补。 | Partial |
| 可观察策略、会话和发送归属、尝试、窗口、静默时长、运行及待执行工具数、终止原因和取消结果，诊断不含模型内容或工具参数 | `inactivity_diagnostics_record_the_window_tools_and_cancellation_result` 覆盖既有超时诊断。有限等待的完整字段待验证。 | Partial |

**决策依据**　D3。
