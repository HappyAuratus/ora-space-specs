# 首轮确认与异常停靠

本文件是[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md)的核心用例。实现尚未落地，证据状态按现状记为 `Missing`，落地运行补齐。

## First Automatic Turn Awaits Explicit Confirmation

### 风险

workflow 自动发起的首轮以 `EndTurn` 正常结束，但 agent 实际上在向用户提问；界面把它当成 ready，用户一点就跳过回答。

### 前置状态

- 交互式 Agent 节点由 workflow 自动发起首轮；
- 本轮 `EndTurn`、历史完整、输出契约通过、没有其他 blocker。

### 触发

首轮 park；用户点击完成。

### 必须成立

- 快照写入 `turnOrigin = "workflow"` 与 `turn_unconfirmed`，`ready = false`；
- 步骤卡片显示「确认并完成」+「继续当前步骤」，右下角按钮禁用；
- 不带 `confirmInitialTurn` 的完成返回 `WorkflowNodeIncomplete`，列出 `turn_unconfirmed`；
- 带 `confirmInitialTurn` 的完成成功，且不写 `completion_forced`；
- 只带 `force` 被拒绝。

### 禁止结果

- 首轮自动回合在未确认时直接推进；或 `force` 把 `turn_unconfirmed` 当成可覆盖的异常 blocker。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 首轮自动 park 写 `turn_unconfirmed` 且不 ready | 待实现运行补快照/投影测试 | `Missing` |
| 未确认完成被拒、确认完成成功且无 force 审计 | 待实现运行补完成路径测试 | `Missing` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D1、D2。

## Abnormal Parking Is Distinct From Awaiting User

### 风险

超时、截断、拒答、未知、会话崩溃等异常与「等待参与」共用同一状态，用户可以在没有确认的情况下推进。

### 前置状态

- 交互式节点停靠，`stopCause` 为 `prompt_timed_out` / `stream_dropped` / `session_crashed` / `unknown`，或 stop reason 为截断/拒答/未知；
- 另一节点由用户主动停止（`user_stopped`）。

### 触发

park 与读取投影。

### 必须成立

- 快照写入 `parkKind = "abnormal"` 与 `stopCause`；用户主动停止写 `awaiting_user` + `user_stopped`；
- 异常停靠显示异常原因，默认不可推进，只能继续或 force（二次确认）；
- 用户主动停止显示「等待参与」，继续或 force 均可；
- 右下角按钮只在 ready 时启用。

### 禁止结果

- 异常停靠显示为「等待参与」；或没有 force 就能推进。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 异常与用户停止的 `parkKind` / `stopCause` 分类 | 待实现运行补 park/投影测试 | `Missing` |
| 异常默认不可推进、只能 force | 待实现运行补完成/UI 测试 | `Missing` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D4、D5。

## Explicit Confirmation Is A Normal Completion

### 风险

把首轮确认当成强制完成，写入 `completion_forced`，污染审计语义。

### 前置状态

- 快照只有 `turn_unconfirmed`，没有其他 blocker。

### 触发

带 `confirmInitialTurn: true` 的完成请求。

### 必须成立

- 走普通完成路径，不写 `completion_forced`；
- 完成后的 payload 保留 `turn_unconfirmed`、turn key 与 `outputDigest`；
- `confirmInitialTurn` 不越过状态/claim、turn key、digest、输出契约或文件 blockers。

### 禁止结果

- 确认请求写入强制审计；或越过其他 blocker。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 确认完成不写 `completion_forced` | 待实现运行补 payload 测试 | `Missing` |
| 确认不越过其他 blocker | 待实现运行补优先级测试 | `Missing` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D2、D3。

## Forced Completion Audits Only Covered Blockers

### 风险

`force` 对没有 blocker 的快照也写审计，或审计不记录 `stopCause` 与实际覆盖的 blocker。

### 前置状态

- 快照含非 `turn_unconfirmed` 的 blocker；或快照已 ready。

### 触发

带 `force: true` 的完成请求。

### 必须成立

- 实际覆盖至少一个 blocker 时才写 `completion_forced`；
- 审计包含 `blockers`、`stop_cause`、`without_turn`、`output_source`；
- 快照已 ready 时按普通完成处理，不写空审计；
- `force && confirmInitialTurn` 返回 `invalid_request`。

### 禁止结果

- 空审计；或 `force` 覆盖 `turn_unconfirmed`。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| force 审计形状与覆盖范围 | 待实现运行补完成路径测试 | `Missing` |
| 互斥请求返回 `invalid_request` | 待实现运行补请求校验测试 | `Missing` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D2、D3。

## Legacy Snapshots Keep Their Previous Semantics

### 风险

升级时给旧快照追溯加 `turn_unconfirmed`，把可能由人类发起的停靠误判成首轮。

### 前置状态

- 升级前已停靠、没有 `turnOrigin` 的 `Pending` 节点。

### 触发

升级后读取/完成该节点。

### 必须成立

- 旧快照不追溯加 `turn_unconfirmed`；
- legacy 物化默认 `turnOrigin = "human"`；
- 旧客户端不带 `confirmInitialTurn` 时不能完成首轮停靠，必须升级；
- 徽标沿用 legacy，不追溯改成异常。

### 禁止结果

- 批量回填或改写旧 payload；或把旧快照当首轮自动 gate。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 旧快照不追溯、物化默认 human | 待实现运行补兼容测试 | `Missing` |
| 旧客户端请求被正确拒绝 | 待实现运行补请求测试 | `Missing` |

### 决策依据

[首轮确认与异常停靠决策](../../../../decisions/desktop/core/workflow/20261102-first-turn-confirmation.md) D7。
