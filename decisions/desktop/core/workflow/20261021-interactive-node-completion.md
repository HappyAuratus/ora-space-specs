---
status: proposed
---

# 交互式节点完成契约：以本轮终态门禁推进，暂停可继续，强制完成可审计

交互式 Agent 节点每轮结束后停在 `Pending`。本决策把「完成」从一个人工动作改为由本轮终态支撑的门禁：只有本轮以 `EndTurn` 结束、且声明的输出契约通过时才允许完成；`Cancelled`、截断、拒答、缺失或未知都视为未完成，必须继续对话或显式强制完成。完成判定在 per-run 锁内以持久状态和 turn key 复核，park 快照进入读取投影；会话消息不被拦截，由后端在每个回合前注入步骤边界。

本决策为 `proposed`。它改变 `completeWorkflowNode` 的公开语义（新增 `WorkflowNodeIncomplete`、`WorkflowNodeTurnChanged`、`WorkflowNodeOutputChanged` 与 `force`），涉及 backend 的完成/停靠路径、contracts 的完成请求、application 的节点映射和 app-shell 的运行视图；数据库只读写既有 `payload` 列，不改 schema、冻结图字段或本地文件布局。正常 `EndTurn` 后的完成行为保持不变；`Loop`/`Iteration` 区域内的 interactive 行为不在本决策范围，仍按既有区域边界处理。升级前已停靠且无快照的节点只在 `prepare` 做一次性物化。

## 继承与修改

本决策继承[节点运行时编排决策](0-node-runtime-orchestration.md)，并沿用[迭代节点](20260915-iteration-composite-runtime.md)、[prompt 静默策略](20261008-prompt-inactivity-policy.md)与[while-Loop](20261013-while-loop-with-durable-round-scopes.md)的边界。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 根 D2：同一 run 的状态变化串行并事务性提交，等待不跨越 per-run 门 | 继承，不变 | 历史读取与快照计算在门外；`completeWorkflowNode` 与 park 的状态迁移在门内只做持久状态比较和单行事务 |
| 根 D5 末段：foreach v1 禁止区域内 interactive，停靠与续算机制保留 | 继承，不变 | 区域行为不变；证据边界见[区域 HITL 用例](../../../../test-cases/desktop/core/workflow/stateless-recovery-and-hitl.md#interactive-nodes-inside-a-region-park-and-resume-through-existing-paths) |
| 迭代决策的 D5 继承行与 D7：foreach 结构校验拒绝 interactive 成员，while 未作同样拒绝 | 继承，并澄清 | foreach 仍拒绝；while 的解析接受不构成本契约的整链保证 |
| while D1：Loop body 接受 interactive 配置 | 继承，并澄清 | 本契约只覆盖外层平图，不补齐区域内的完成/恢复证据 |
| 根 D7：失效事件只标识 run，观察者重查持久状态 | 继承，不变 | park 或修复提交后由 transition sink 发布一次 run invalidation；前端重查 |
| 静默策略 D2：每个 Agent turn 使用冻结策略 | 继承，不变 | 人类回合继续沿用节点冻结的 `promptInactivity` |
| while D6：重启时 Running 轮按 `interrupted_by_restart` 失败 | 继承，不变 | `Pending` 保留；在途 `Running` 仍失败，不自动续算 |
| 完成只由人工按钮触发，且不检查回合终态 | 修改 | 完成由本轮终态与声明契约共同决定；`force` 需确认并审计 |

## 决策

### D1：完成门禁只认本轮 `EndTurn`，其余终态一律未完成

只有最后一轮的 stop reason 是 `EndTurn` 才能完成。stop reason 映射固定为：

| stop reason | blocker | 是否允许完成 |
| --- | --- | --- |
| `EndTurn` | 无 | 是 |
| `MaxTokens` / `MaxTurnRequests` | `turn_truncated` | 否 |
| `Cancelled` | `turn_stopped` | 否 |
| `Refusal` | `turn_refused` | 否 |
| 缺失或未知 | `turn_unknown` | 否 |

`缺失` 在首轮仍由 `SessionEndedWithoutStopReason` 失败，不进入停靠；只有人类回合或历史无法界定本轮时才映射为 `turn_unknown` 并停靠。

**交互式节点对所有 terminal stop reason 停靠**：`EndTurn`、截断、`Cancelled`、`Refusal`、未知都写快照并停在 `Pending`，对应 blocker 让「完成」不可用，但用户仍可继续对话或显式 `force`。非交互节点保持现有语义：`Refusal` 与未知仍失败节点，不进入本契约的停靠路径。这条修改了 `pauses_interactive_node` 的集合，使 `turn_refused` / `turn_unknown` 在首轮也可达。

节点正在 `Running` 时不能完成，沿用现有 `workflow_node_not_awaiting_input`，不新增 `turn_in_progress` blocker。声明 `outputContract=Structured` 的节点在 **park 计算快照时**按 schema 校验，失败把 `structured_invalid` 放进 blockers；`Text` 要求本轮最终输出非空（`output_missing`）；无契约（`None`）只校验回合终态，空输出只记非阻塞 warning。编辑器目前只能声明 `Structured`，`Text` 只用于旧快照或导入图。

`HistoryIntegrity` 不是 `Complete` 时：若历史里仍能找到本轮 `TurnEnded`，快照写入 `userSeq` / `turnKey` / `stopReason` / `outputDigest`，但 blocker 仍是 `turn_unknown`，并加 `history_integrity_gap` warning；找不到 `TurnEnded` 时这些字段为 `null`。两种情况都只能显式 `force`，系统不自动 ready。

选择只认 `EndTurn`，是因为截断、停止和拒答都意味着产出不完整；把 `MaxTokens` / `MaxTurnRequests` 当作自然结束会让暂停后重启的问题在 token 截断时静默复发。`None` 不检查空输出，是为了不误伤只改文件、最终回复为空的既有节点。

### D2：用 `{ userSeq, endedSeq }` 唯一标识本轮

「本轮」指最后一条用户消息之后、其后的 `TurnEnded` 之前的部分。turn key 是二元组：

```json
{ "userSeq": 12, "endedSeq": 13 }
```

`userSeq` 是最后一条用户消息的 `HistoryLine.seq`，`endedSeq` 是它之后那条 `TurnEnded` 的 `seq`；两者共同唯一标识本轮。park 时从历史推导并随快照落盘（`payload.completion.turnKey`）。若最后一条用户消息之后没有 `TurnEnded`，判 `turn_unknown`，**快照不生成 turn key**（`turnKey: null`），也不得读取上一轮的 `EndTurn` 或上一轮的 assistant 文本。

**本轮最终 assistant 文本**固定为：在 `(attemptStart, endedSeq]` 窗口内按 `seq` 取最后一条 assistant 消息的文本；可能为空；不取「最后一条非空」，也不把更早的进度说明拼进来。`attemptStart` 见下段；无重试时等于 `userSeq`。没有收尾消息时就是空，交给 `Text` / `Structured` / `None` 的规则处理。现有的 `assistant_output_from_history` 取最后一条非空 assistant 文本，不能复用到这条路径。

**重试后的最终文本**：`Retrying` 会重置 executor 的 accumulator；历史里被放弃尝试的 assistant 消息也会保留，并由 `settle_cancelled_prompt` 写入自己的 `TurnEnded`。因此最终文本不能简单取窗口内最后一条 assistant 消息，而要取 `attemptStart` 之后的最后一条：`attemptStart = max(userSeq, endedSeq 之前最后一条 TurnEnded 的 seq)`。这样被放弃尝试的文本不会混入最终输出，和 executor 的 `Retrying` 重置一致。

事件流不带 seq（`PromptSessionEvent` 没有该字段），`record_settled` 也不是可用的历史写入点；因此 turn key 与最终文本只能从历史读取，不能由流累积推导。`complete` 的 `prepare` 必须从同一历史重新推导二元组并比对，不一致返回冲突，不做完成。布尔值「是否存在 `TurnEnded`」无法当版本比对，也无法区分两轮之间的错配；因此这里使用二元组，而不是布尔标记。

### D3：首轮与人类回合共用同一个 park 函数，一次事务写快照与状态

两个 park 入口必须调用同一个函数：executor 的首轮停靠（`pauses_interactive_node` 分支）与人类回合的 cleanup。这个函数**自己获取 per-run 门**，调用点不得已经持有该门（门不可重入）。每次调用：

1. 在门外读一次历史：按 `seq` 排序，定位最后一条用户消息、它之后的 `TurnEnded`、`(attemptStart, endedSeq]` 窗口内的最后一条 assistant 文本（D2）与 `HistoryIntegrity`。
2. 计算 blockers 与 warnings：D1 的 stop reason 映射；`Structured` / `Text` 校验；`HistoryIntegrity` 非 `Complete` 判 `turn_unknown`。
3. 生成快照；没有 turn key 时字段为 `null`：

```json
{
  "completion": {
    "userSeq": 12,
    "turnKey": { "userSeq": 12, "endedSeq": 13 },
    "stopReason": "end_turn | cancelled | ...",
    "blockers": [{ "kind": "turn_stopped", "params": {} }],
    "warnings": [],
    "outputDigest": "sha256:...",
    "provisional": false
  }
}
```

`outputDigest` 是本轮最终 assistant 文本的摘要，用来在 `complete` 时确认读到的仍是同一份文本；`userSeq` 在有最后一条用户消息时总是写入；没有终态时 `turnKey`、`stopReason`、`outputDigest` 都是 `null`。快照**不存 `structuredOutput`**；`Structured` 的结构化值由 `prepare` 在门外用 digest 校验过的文本重跑 schema 得到（纯 CPU，不进入锁内）。

4. 获取 per-run 门，用同一个 `Immediate` 事务提交：guarded `UPDATE workflow_node_runs SET status = 'pending', payload = <JSON merge>`，只允许 `status = 'running'`，并保留 `error_detail` / `ai_diagnosis` / `file_changes` 等全部既有 key。一条语句同时写 `payload.completion` 和状态；观察者只会看到「`Running` + 旧 payload」或「`Pending` + 快照」，不会出现 `Pending` 无快照。guard 拒绝时什么都不写，不产生孤儿快照；因此不需要失败标记或离开 `Pending` 时的清理规则。提交成功后由 transition sink 发布一次 run invalidation。投影只在 `Pending` 读取 `completion`，节点进入终态后该字段不再参与展示。

读取投影（`GetWorkflowRun` 的节点状态）只解析 `payload.completion`，**不读会话历史、不跑 schema、也不做修复**。原因有两条：`map_node_run` 是纯字段映射，拿不到 `sessions_root` 与历史；run detail 以 1.5 秒轮询，实时读历史或跑校验的成本与缓存都无法接受。不新增 `check` 接口，`completeWorkflowNode` 仍是唯一提交权威，投影只是同一份快照的只读视图。

**修复/物化事务（Pending → Pending）**：升级前的 `Pending` 旧节点或 `provisional` 快照只在 `prepare` 修复一次。`prepare` 读一次历史，能推导出 turn key 就用同一函数重算；随后用 payload-only guarded UPDATE（只允许 `status = 'pending'`）写回 `payload.completion`，不改状态，提交后发布 invalidation。推不出来就保持 `turn_unknown`，由用户决定 `force`。投影对无快照的 `Pending` 节点显示「回合未知」，不显示 ready，也不触发任何历史读取。

这与上面的 park 事务是两种形状：park 是 `Running → Pending` 并写 payload；修复只写 payload、不改状态。两者共用同一个快照计算函数。

### D4：人类回合的 cleanup 先等 actor 收敛，再 park

`WorkflowSessionTurns::prompt` 只负责转发事件与会话清理，不再承担“累积文本作为快照输入”的职责。人类回合结束时，admission 必须携带 `run_id`、`sessions_root`、冻结图与会话 id；cleanup 在流结束后**先等待 owning prompt 的 actor 收敛**（有界等待 `TurnEnded` 写入），再调用 D3 的 park 函数。

这样避免一个已知竞态：流被 drop 时 actor 可能还在收尾，cleanup 立即读历史会看不到 `TurnEnded`，把实际以 `EndTurn` 结束的一轮写成 `turn_unknown`，而且之后没有路径修复。规则是：

- 有界等待内看到 `TurnEnded`：按正常路径写快照；
- 等待超时仍没有 `TurnEnded`：写 `provisional: true` 的 `turn_unknown` 快照；
- 之后 `prepare` 若在同一个 `userSeq` 之后看到 `TurnEnded`，重新计算并替换这条 provisional 快照（payload-only guarded Pending update），完成修复；替换前投影显示 `turn_unknown`。

cleanup 与首轮 executor 都通过 D3 的函数获取 per-run 门；门内只做一条 guarded UPDATE，不读历史、不跑 schema。cleanup 必须幂等，不能覆盖已经提交的非 provisional 快照。

### D5：`completeWorkflowNode` 按固定优先级校验，force 只覆盖 blockers

校验顺序与错误固定为：

1. run 不是 `Running`、节点不是 `Pending`、completion claim 不持有 → `workflow_node_not_awaiting_input`（现有错误），**`force` 不能越过**；
2. turn key 不一致 → `WorkflowNodeTurnChanged { node_id, expected, actual }`，前端刷新运行后再试；
3. `outputDigest` 不一致 → `WorkflowNodeOutputChanged { node_id, expected, actual }`，前端刷新运行后再试；
4. 快照 blockers 非空且没有 `force` → `WorkflowNodeIncomplete { node_id, blockers[] }`。

只有第 4 步的 blockers 可以被 `force` 覆盖。`force` 必须由用户二次确认，并在 payload 记录 `completion_forced` 与 blockers 快照；`completion_forced.reason` 只记 blockers 快照，不混入用户自由文本。

`complete` 不在锁内重算 schema。`prepare` 在门外读取 D2 定义的同一窗口文本、比对快照的 `outputDigest`，并用这段文本重跑 `Structured` schema 得到结构化值（纯 CPU）；同一文本与同一纯函数的结果确定，若与 park blockers 不一致，以 prepare 结果为准并刷新快照，再按刷新后的 blockers 返回 `WorkflowNodeIncomplete`（防御性路径）。锁内只比对 turn key、状态与 claim。

没有 turn key 的 `turn_unknown` 快照只能 `force`。`force` 的“当前产出”固定为：`attemptStart` 之后的最后一条 assistant 消息文本（可能为空），其中 `attemptStart = max(userSeq, 历史末尾之前最后一条 TurnEnded 的 seq)`；没有 TurnEnded 时按最后一条用户消息。不计算 `outputDigest`，不写 `structured_output`，并在 `completion_forced` 记录 `without_turn: true` 与 `output_source: "last_assistant_message"`。若 `prepare` 在修复时找到了 `TurnEnded`，则按正常 turn key 路径完成，不需要 force。

无快照的旧节点由 `prepare` 先做一次 lazy materialize，再走同一路径。payload 合并必须在同一个 `Immediate` 事务内完成，并保护全部既有 key；不能整体覆盖 `error_detail` / `ai_diagnosis` / `file_changes`。`completion_forced` 作为 payload 的独立 key 记录，不与 `completion` 混用。

structured 校验失败时 `force` 不写 `structured_output`，raw output 仍保存；下游引用 `{node}.structured_output` 按变量缺失处理，由现有变量模板校验报错，不引入静默降级。

### D6：会话消息不被拦截，每个回合注入步骤边界

不拦截、不改写任何用户消息，不按词表解释「继续」。每个 workflow 人类回合在 prompt 前加一个本地化的 `<workflow_step_boundary>`：

- 当前步骤与直接后续节点（可能多个，也可能没有）；
- 只完成本步骤，不执行后续节点；
- 用户要求进入下一步时，提示用户在工作流界面确认本步骤完成。

文案由后端从冻结图渲染，`record_prompt` 保持用户原文，未来 CLI 或其他客户端自动生效。边界块写「本步骤的范围与完成标准」，不写「系统会阻止你执行后续节点」这类运行时无法兑现的承诺，也不写死 Desktop 的按钮名。首轮自动回合已有完整 handoff，保持不变。

### D7：UI 只暴露一个推进动作，卡片只展示状态

会话消息列表末尾保留一张当前步骤卡片（本决策不做每轮一张）：ready 时显示「完成并进入下一步：<直接后续节点；末节点不显示下一步>」和「继续对话」；未完成时写明原因（已停止 / 被截断 / 结构化输出不符合 / 回合未知），主操作「继续当前步骤」，次操作「仍然完成（采用当前产出）」，后者二次确认。

右下角按钮与卡片共用同一套 `ready` / `blockers`，是唯一推进入口；卡片不提供第二个主操作。「继续当前步骤」是用户明确点击的动作，直接发送一条带边界与未完成原因的作用域指令，不需要额外确认；只有强制完成需要确认并留痕。徽标区分「等待参与」和「已暂停（上一轮未正常结束）」，原因覆盖停止、截断、中断/超时；act card 不再直接渲染原始 `cancelled`。运行列表不派生 paused，只保留 Theater 节点徽标。

### D8：重启由 runtime 处理 provider 恢复，工作流层不预设降级

`Pending` 在重启后保留。provider 会话层由 runtime 负责：`attach` 先试 `session/load`，不支持或失败就 `rebuild_provider_session`，下一条 prompt 带 transcript 交接；工作流层看不到 “restore failed”，因此不预设工作流层的「重新运行本步骤」。

验收固定为三条：

1. 重启后 `session/load` 成功、失败→rebuild 两条路径，「继续当前步骤」都能得到一轮回复（真实 agent 集成测试）；
2. 如果 continue 的 prompt 失败：界面明确报错，节点保持 `Pending`，「仍然完成」仍可用，不静默丢回合；
3. 如果测试发现真正无法继续（Ora 会话/历史丢失等），另开依赖本决策的后续 issue，定义「重新运行本步骤」的触发、机制（新 attempt / 新 node run）、部分输出来源与测试；不阻塞本 ADR。

重启时仍在 `Running` 的节点按 while D6 的失败边界处理，不由本决策新增自动续算。

## 不变量

1. 正常完成只认本轮 `EndTurn`；截断、停止、拒答、缺失与未知一律阻塞，`force` 是显式例外。
2. 完成使用的是本轮输出；本轮没有终态时不读取上一轮输出。
3. 本轮最终 assistant 文本取 `(attemptStart, endedSeq]` 窗口内按 `seq` 的最后一条 assistant 消息，可能为空，不取最后一条非空；`attemptStart = max(userSeq, endedSeq 之前最后一条 TurnEnded 的 seq)`，被放弃尝试的文本不进入输出。
4. 交互式节点对所有 terminal stop reason 停靠；非交互节点保持现有失败语义。
5. executor 首轮与人类 cleanup 共用同一个 park 函数；快照与 `Running → Pending` 在同一个 `Immediate` 事务提交，新 run 不会出现 `Pending` 无快照。
6. park 与 `prepare` 使用同一份历史、同一组输入；turn key、文本与 stop reason 都从历史读取，不用事件流累积。
7. 每次交互回合读取一次整文件历史；读取投影不读历史、不跑 schema、不做修复。
8. `Structured` / `Text` 校验在 park 计算快照时完成，blockers 用于展示；`prepare` 在门外用同一段文本重跑 schema 得到结构化值，锁内不重算。
9. 错误优先级固定：状态/claim → `workflow_node_not_awaiting_input`；turn key → `WorkflowNodeTurnChanged`；digest → `WorkflowNodeOutputChanged`；只有 blockers 可被 `force` 覆盖。
10. 无 turn key 的 `turn_unknown` 只能 force；force 的文本固定为 `attemptStart` 之后的最后一条 assistant 消息，可能为空。
11. `completeWorkflowNode` 是唯一提交权威；UI 的 ready/blockers 只是同一份快照的预览。
12. `force` 必须二次确认并留痕；结构化失败时 force 不写 `structured_output`。
13. 用户消息不被拦截或改写；每个 workflow 人类回合都带步骤边界。
14. 非交互节点与正常 `EndTurn` 的行为保持不变；交互式节点在 Refusal/未知时停靠是本决策有意的行为变化。
15. 区域 interactive 行为不变；本契约只覆盖外层平图。
16. `Pending` 跨重启保留；在途 `Running` 仍按既有失败边界处理。
17. cleanup 在流 drop 后先等 actor 收敛；超时写 provisional `turn_unknown`，只在 `prepare` 修复。
18. `HistoryIntegrity` 非 `Complete` 时，即使找到 `TurnEnded` 也判 `turn_unknown`；字段可填但 blocker 不变。
19. 快照总是存 `userSeq`；provisional 修复按 `userSeq` 匹配。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 只阻塞 `Cancelled` | 改动最小，但截断、拒答和旧回合输出仍会静默推进，暂停后重启的问题只修一半 |
| 按词表拦截「继续/下一步」 | 意图有歧义，带附件、代码块和未来 CLI 都会误判；猜错会推进节点或吞消息 |
| 每次完成都让 LLM judge 判“工作是否完成” | 自由文本没有可判定目标，成本与误判高；应作为独立能力，另开 issue |
| 新增只读 `checkWorkflowNodeCompletion` 接口 | 多一套校验实现，容易出现 check 说 ready、complete 拒绝的分叉 |
| 每轮结束都插一张卡片 | 重启和多轮后积累多张状态互相矛盾的卡片；本决策只保留当前一张 |
| 无契约节点也要求输出非空 | 会误伤只改文件、最终回复为空的既有节点，破坏“正常完成行为不变” |
| 工作流层再做一条「不能恢复→重新运行本步骤」 | 正常路径 runtime 总会 rebuild，工作流层看不到 restore failed，无法观测也无法测试；真正不可恢复应另开能力 |
| 在读取接口里实时读历史算 blockers/做修复 | 每个 Pending 节点每 1.5 秒读一次整文件，成本不可接受，且崩溃后永远修不好；修复只在 `prepare` 做一次 |
| park 分两次写快照与状态，再配失败标记/孤儿清理 | 两次写会引入「Pending 但快照缺失」的窗口和额外规则；单事务 guarded UPDATE 已足够 |
| 用布尔值“本轮有 TurnEnded”当 turn key | 不能唯一标识回合，无法发现两轮之间的错配 |
| 用事件流累积文本/stop reason 作为快照输入 | 事件流没有 `HistoryLine.seq`，无法推导 turn key；且 `prepare` 读历史，两条路径输入不同源 |
| 在 payload 里存 `structuredOutput` | 增加 payload 体积和暴露面，还要定义离开 `Pending` 的删除事务；改为只存 digest，`prepare` 用同一段文本重跑 schema |
| cleanup 在流 drop 后立即写 `turn_unknown` | actor 可能还没写 `TurnEnded`，会把实际 `EndTurn` 的一轮误判；应先有界等待，超时用 provisional，由 `prepare` 修复 |
| `force` 越过状态或 claim | 会把已取消/已完成的节点重新提交；状态与 claim 必须先于 blockers 拒绝 |
| `turn_unknown` 也自动 ready | 没有终态就没有可验证的文本与 turn key；只能显式 force |
| 把重试前被放弃尝试的文本并入最终输出 | 与 executor 的 `Retrying` 重置不一致，会把失败尝试的半截内容当成本轮交付 |
| `HistoryIntegrity` 非 `Complete` 但找到 `TurnEnded` 就判 ready | 历史本身有缺口，无法确认收敛；保守判 `turn_unknown`，只能 force |

## 落地顺序

1. **后端门禁与事实**：D1 的 stop reason/输出校验与 `HistoryIntegrity`、D2 的 `{userSeq, endedSeq}` 与窗口文本、D3 的共享 park 函数与单事务写入、D4 的 cleanup 等待、D5 的优先级/force 审计与两个冲突错误；旧 run 的 `prepare` 物化随本阶段上线。
2. **前端入口**：D7 的 trailing slot、当前步骤卡片、按钮/徽标与中英文案；`WorkflowNodeTurnChanged` / `WorkflowNodeOutputChanged` 时刷新运行。
3. **验证与文档**：单测矩阵、重启 restore/rebuild 集成测试、e2e 两条复现路径、前端测试；同步 `docs/workflows/workflow-runs.md` 与 `.zh.md` 的 Interactive nodes / Recovery after restart。
4. **后续能力**：`completion.files` / `criteria`、运行时验收清单、LLM judge 与 agent 自报完成另开 issue；真正无法继续时的「重新运行本步骤」也另开 issue，不阻塞本决策。

## 风险、未解决问题与验收

- **每次交互回合读一次整文件历史**：turn key 依赖 `HistoryLine.seq`，事件流不提供，park 必须读历史；这是本决策接受的成本，长会话是 O(会话) 的读取。读取投影不读历史，修复只在 `prepare` 做一次。
- **重试边界**：被放弃尝试的 assistant 消息与 `TurnEnded` 留在历史里；最终文本从 `attemptStart` 之后取，和 executor 的 `Retrying` 重置一致。
- **provisional 快照修复**：流 drop 且等待超时会先写 `turn_unknown`；只有 `prepare` 在同一个 `userSeq` 之后看到 `TurnEnded` 时替换。修复窗口内界面显示未完成，用户仍可 `force`。
- **旧节点只在完成时物化**：升级前的 `Pending` 节点在 `prepare` 前显示「回合未知」；用户点击完成后由 `prepare` 一次性回填，不在 1.5 秒轮询里读历史。
- **`turn_unknown` 的 force**：没有 turn key/digest，只能显式 force；文本取 `attemptStart` 之后的最后一条 assistant 消息，可能为空，不写 `structured_output`，审计 `without_turn`。
- **交互式 Refusal/未知停靠**：修改 `pauses_interactive_node` 的集合，使交互式节点在拒答/未知时也停在 `Pending`；非交互节点保持现有失败语义。这是有意的行为变化，需在实现运行里验证。
- **`structuredOutput` 不落 payload**：避免体积和暴露面；`prepare` 用 digest 校验过的文本重跑 schema（纯 CPU）。
- **per-run 门不可重入**：park 函数自己获取门；executor 与 cleanup 调用点都不得持有门。
- **UI 卡片挂载**：`ChatView` 目前只有 `contextBar` 和 `composerActions`，需要新增可选 trailing slot；这是共享组件改动，实现时要保证普通聊天不受影响。
- **区域 HITL**：本决策不改变 foreach 的拒绝规则，也不补齐 while 区域内的整链证据；该缺口仍由现有区域用例记录。
- **不在本决策范围**：`completion.files` / `criteria`、运行时验收清单、LLM judge 与 agent 自报完成均另开 issue，可依赖本决策。

验收：

- 停止生成 → 重启 → 点完成：返回 `turn_stopped`，不调度下一节点；「继续当前步骤」在 `session/load` 成功/失败→rebuild 两条路径都能得到一轮回复；continue 失败时明确报错、节点保持 `Pending`、仍可强制完成；
- `MaxTokens` / `MaxTurnRequests` / `Refusal` / 未知 stop reason、以及 `HistoryIntegrity` 非 `Complete` 同样不能直接完成；交互式 Refusal/未知会停靠，非交互仍失败；
- 本轮最终文本按 `(attemptStart, endedSeq]` 窗口取最后一条 assistant 消息；重试前的被放弃文本不进入输出；空文本不会被早先的进度说明掩盖；
- `HistoryIntegrity` 非 `Complete` 时字段形状按 ADR：能找到 `TurnEnded` 就填 `userSeq` / `turnKey` / `stopReason` / `outputDigest`，但 blocker 仍是 `turn_unknown`；快照总是存 `userSeq`，provisional 修复按它匹配；
- 正常 `EndTurn` 后完成与现在一致；无契约节点空输出仍可完成，只显示 warning；
- 首轮 executor park 与人类 cleanup 都经过同一个函数；`payload.completion` 与 `Running → Pending` 同事务提交，新 run 不会出现 `Pending` 无快照；
- 状态/claim 不满足返回 `workflow_node_not_awaiting_input`，force 不能越过；turn key 不一致返回 `WorkflowNodeTurnChanged`；digest 不一致返回 `WorkflowNodeOutputChanged`；只有 blockers 可被 force 覆盖；
- 无快照旧节点与 provisional 只在 `prepare` 修复一次；`get` 不读历史；
- `Structured` / `Text` 校验在 park 完成，blockers 与 complete 同源；`structuredOutput` 不落 payload；
- cleanup 在流 drop 后先有界等待；超时写 provisional `turn_unknown`，`prepare` 能修复；
- park/修复提交后发布一次 run invalidation；
- 区域 interactive 行为不变；核心用例在 `approved` 后补到 `test-cases/desktop/core/workflow/interactive-node-completion.md`。
