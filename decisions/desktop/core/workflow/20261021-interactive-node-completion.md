---
status: proposed
---

# 交互式节点完成契约：以本轮终态门禁推进，暂停可继续，强制完成可审计

交互式 Agent 节点每轮结束后停在 `Pending`。本决策把「完成」从一个人工动作改为由本轮终态支撑的门禁：只有本轮以 `EndTurn` 结束、且声明的输出契约通过时才允许完成；`Cancelled`、截断、拒答、缺失或未知都视为未完成，必须继续对话或显式强制完成。完成判定在 per-run 锁内以持久状态和 turn key 复核，park 快照进入读取投影；会话消息不被拦截，由后端在每个回合前注入步骤边界。

本决策为 `proposed`。它改变 `completeWorkflowNode` 的公开语义（新增 `WorkflowNodeIncomplete`、`WorkflowNodeTurnChanged` 与 `force`），涉及 backend 的完成/停靠路径、contracts 的完成请求、application 的节点映射和 app-shell 的运行视图；数据库只读写既有 `payload` 列，不改 schema、冻结图字段或本地文件布局。正常 `EndTurn` 后的完成行为保持不变；`Loop`/`Iteration` 区域内的 interactive 行为不在本决策范围，仍按既有区域边界处理。升级前已停靠且无快照的节点按 D3 的一次性物化处理。

## 继承与修改

本决策继承[节点运行时编排决策](0-node-runtime-orchestration.md)，并沿用[迭代节点](20260915-iteration-composite-runtime.md)、[prompt 静默策略](20261008-prompt-inactivity-policy.md)与[while-Loop](20261013-while-loop-with-durable-round-scopes.md)的边界。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 根 D2：同一 run 的状态变化串行并事务性提交，等待不跨越 per-run 门 | 继承，不变 | 历史读取与快照计算在门外；`completeWorkflowNode` 与 park 的状态迁移在门内只做持久状态比较和单行事务 |
| 根 D5：foreach 区域禁止 interactive；while 接受配置但无整链保证 | 继承，并澄清 | 本契约只覆盖外层平图；区域行为不变，证据边界仍见[区域 HITL 用例](../../../../test-cases/desktop/core/workflow/stateless-recovery-and-hitl.md#interactive-nodes-inside-a-region-park-and-resume-through-existing-paths) |
| 根 D7：失效事件只标识 run，观察者重查持久状态 | 继承，不变 | 完成快照进入节点读取投影；前端轮询或失效事件后重查 |
| 迭代 D5：foreach 执行前拒绝 interactive 成员 | 继承，不变 | 无变化 |
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

节点正在 `Running` 时不能完成，沿用现有 `workflow_node_not_awaiting_input`，不新增 `turn_in_progress` blocker。声明 `outputContract=Structured` 的节点在 **park 计算快照时**按 schema 校验，失败把 `structured_invalid` 放进 blockers；`Text` 要求本轮最终输出非空（`output_missing`）；无契约（`None`）只校验回合终态，空输出只记非阻塞 warning。编辑器目前只能声明 `Structured`，`Text` 只用于旧快照或导入图。

`HistoryIntegrity` 不是 `Complete` 时，park 无法确认本轮是否收敛，统一判 `turn_unknown` 并把完整性缺口记为 warning；用户可以显式 `force`，但系统不自动 ready。

选择只认 `EndTurn`，是因为截断、停止和拒答都意味着产出不完整；把 `MaxTokens` / `MaxTurnRequests` 当作自然结束会让暂停后重启的问题在 token 截断时静默复发。`None` 不检查空输出，是为了不误伤只改文件、最终回复为空的既有节点。

### D2：用 `{ userSeq, endedSeq }` 唯一标识本轮

「本轮」指最后一条用户消息之后、其后的 `TurnEnded` 之前的部分。turn key 是二元组：

```json
{ "userSeq": 12, "endedSeq": 13 }
```

`userSeq` 是最后一条用户消息的 `HistoryLine.seq`，`endedSeq` 是它之后那条 `TurnEnded` 的 `seq`；两者共同唯一标识本轮。park 时从历史推导并随快照落盘（`payload.completion.turnKey`）。若最后一条用户消息之后没有 `TurnEnded`，判 `turn_unknown`，不生成 turn key，也不得读取上一轮的 `EndTurn` 或上一轮的 assistant 文本。

**本轮最终 assistant 文本**固定为：在 `(userSeq, endedSeq]` 窗口内按 `seq` 取最后一条 assistant 消息的文本；可能为空；不取「最后一条非空」，也不把更早的进度说明拼进来。没有收尾消息时就是空，交给 `Text` / `Structured` / `None` 的规则处理。现有的 `assistant_output_from_history` 取最后一条非空 assistant 文本，不能复用到这条路径。

事件流不带 seq（`PromptSessionEvent` 没有该字段），`record_settled` 也不是可用的历史写入点；因此 turn key 与最终文本只能从历史读取，不能由流累积推导。`complete` 的 `prepare` 必须从同一历史重新推导二元组并比对，不一致返回冲突，不做完成。布尔值「是否存在 `TurnEnded`」无法当版本比对，也无法区分两轮之间的错配；因此这里使用二元组，而不是布尔标记。

### D3：首轮与人类回合共用同一个 park 函数，一次事务写快照与状态

两个 park 入口必须调用同一个函数：executor 的首轮停靠（`pauses_interactive_node` 分支）与人类回合的 cleanup。这个函数**自己获取 per-run 门**，调用点不得已经持有该门（门不可重入）。每次调用：

1. 在门外读一次历史：按 `seq` 排序，定位最后一条用户消息、它之后的 `TurnEnded`、`(userSeq, endedSeq]` 窗口内的最后一条 assistant 文本（D2）与 `HistoryIntegrity`。
2. 计算 blockers 与 warnings：D1 的 stop reason 映射；`Structured` / `Text` 校验；`HistoryIntegrity` 非 `Complete` 判 `turn_unknown`。
3. 生成快照：

```json
{
  "completion": {
    "turnKey": { "userSeq": 12, "endedSeq": 13 },
    "stopReason": "end_turn | cancelled | ...",
    "blockers": [{ "kind": "turn_stopped", "params": {} }],
    "warnings": [],
    "outputDigest": "sha256:...",
    "provisional": false
  }
}
```

`outputDigest` 是本轮最终 assistant 文本的摘要，用来在 `complete` 时确认读到的仍是同一份文本。快照**不存 `structuredOutput`**；`Structured` 的结构化值由 `prepare` 在门外用 digest 校验过的文本重跑 schema 得到（纯 CPU，不进入锁内）。

4. 获取 per-run 门，用同一个 `Immediate` 事务提交：guarded `UPDATE workflow_node_runs SET status = 'pending', payload = <JSON merge>`，只允许 `status = 'running'`，并保留 `error_detail` / `ai_diagnosis` / `file_changes` 等全部既有 key。一条语句同时写 `payload.completion` 和状态；观察者只会看到「`Running` + 旧 payload」或「`Pending` + 快照」，不会出现 `Pending` 无快照。guard 拒绝时什么都不写，不产生孤儿快照；因此不需要失败标记或离开 `Pending` 时的清理规则。投影只在 `Pending` 读取 `completion`，节点进入终态后该字段不再参与展示。

新 run 的写入是原子的；若仍出现 `Pending` 且无 `completion`，只可能是升级前的旧 run。旧 run 的 lazy materialize 由 `get` 与 `complete` 的 `prepare` 共用同一个函数：读一次历史、计算并回填；两条路径都先读历史、再获取 per-run 门做 payload-only guarded Pending update，guard 拒绝就丢弃结果并按当前状态投影。`get` 只对无快照的 `Pending` 节点做一次，成功回填后不再重复；`prepare` 在完成前确保快照存在。

读取投影（`GetWorkflowRun` 的节点状态）只解析 `payload.completion`，不读会话历史、不跑 schema。原因有两条：`map_node_run` 是纯字段映射，拿不到 `sessions_root` 与历史；run detail 以 1.5 秒轮询，实时读历史或跑校验的成本与缓存都无法接受。不新增 `check` 接口，`completeWorkflowNode` 仍是唯一提交权威，投影只是同一份快照的只读视图。

### D4：人类回合的 cleanup 先等 actor 收敛，再 park

`WorkflowSessionTurns::prompt` 只负责转发事件与会话清理，不再承担“累积文本作为快照输入”的职责。人类回合结束时，admission 必须携带 `run_id`、`sessions_root`、冻结图与会话 id；cleanup 在流结束后**先等待 owning prompt 的 actor 收敛**（有界等待 `TurnEnded` 写入），再调用 D3 的 park 函数。

这样避免一个已知竞态：流被 drop 时 actor 可能还在收尾，cleanup 立即读历史会看不到 `TurnEnded`，把实际以 `EndTurn` 结束的一轮写成 `turn_unknown`，而且之后没有路径修复。规则是：

- 有界等待内看到 `TurnEnded`：按正常路径写快照；
- 等待超时仍没有 `TurnEnded`：写 `provisional: true` 的 `turn_unknown` 快照；
- 之后 `get` / `prepare` 若在同一个 `userSeq` 之后看到 `TurnEnded`，重新计算并替换这条 provisional 快照（payload-only guarded Pending update），完成修复；替换前投影显示 `turn_unknown`。

cleanup 与首轮 executor 都通过 D3 的函数获取 per-run 门；门内只做一条 guarded UPDATE，不读历史、不跑 schema。cleanup 必须幂等，不能覆盖已经提交的非 provisional 快照。

### D5：`completeWorkflowNode` 在锁内做权威比对，force 必须留痕

`completeWorkflowNode` 在 per-run 锁内重校验：run 仍 `Running`、节点仍 `Pending`、completion claim 仍持有、turn key 与 prepare 记录一致。不满足且没有 `force` 时返回 `WorkflowNodeIncomplete { node_id, blockers[] }`；turn key 不一致返回 `PublicError::WorkflowNodeTurnChanged { node_id, expected, actual }`，前端刷新运行后再试，不把它伪装成 blocker。

`complete` 不在锁内重算 schema。`prepare` 在门外读取 D2 定义的同一窗口文本、比对快照的 `outputDigest`，并用这段文本重跑 `Structured` schema 得到结构化值（纯 CPU）；同一文本与同一纯函数的结果确定，若与 park blockers 不一致，以 prepare 结果为准、刷新快照并返回冲突。锁内只比对 turn key、状态与 claim。

无快照的旧节点由 `prepare` 先做一次 lazy materialize，再走同一路径。payload 合并必须在同一个 `Immediate` 事务内完成，并保护全部既有 key；不能整体覆盖 `error_detail` / `ai_diagnosis` / `file_changes`。`completion_forced` 作为 payload 的独立 key 记录，不与 `completion` 混用；`completion_forced.reason` 只记 blockers 快照，不混入用户自由文本。

`turn key` 比对是防御性一致性检查：`claim_node_for_completion` 已经在同一把锁下阻止 `prepare` 之后插入新回合；真正要处理的是 park 事务失败后的恢复、provisional 快照修复和升级前无快照。structured 校验失败时 `force` 不写 `structured_output`，raw output 仍保存；下游引用 `{node}.structured_output` 按变量缺失处理，由现有变量模板校验报错，不引入静默降级。

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

1. 只有本轮 `EndTurn` 才能完成；截断、停止、拒答、缺失与未知一律阻塞。
2. 完成使用的是本轮输出；本轮没有终态时不读取上一轮输出。
3. 本轮最终 assistant 文本取 `(userSeq, endedSeq]` 窗口内按 `seq` 的最后一条 assistant 消息，可能为空，不取最后一条非空。
4. executor 首轮与人类 cleanup 共用同一个 park 函数；快照与 `Running → Pending` 在同一个 `Immediate` 事务提交，新 run 不会出现 `Pending` 无快照。
5. park 与 `prepare` 使用同一份历史、同一组输入；turn key、文本与 stop reason 都从历史读取，不用事件流累积。
6. 每次交互回合读取一次整文件历史；这是本决策接受的成本，不把历史读取放进 1.5 秒的读取投影。
7. `Structured` / `Text` 校验在 park 计算快照时完成，blockers 用于展示；`prepare` 在门外用同一段文本重跑 schema 得到结构化值，锁内不重算。
8. `completeWorkflowNode` 是唯一提交权威；UI 的 ready/blockers 只是同一份快照的预览。
9. `force` 必须二次确认并留痕；结构化失败时 force 不写 `structured_output`。
10. 用户消息不被拦截或改写；每个 workflow 人类回合都带步骤边界。
11. 正常 `EndTurn`、无契约空输出、既有平图行为保持不变。
12. 区域 interactive 行为不变；本契约只覆盖外层平图。
13. `Pending` 跨重启保留；在途 `Running` 仍按既有失败边界处理。
14. cleanup 在流 drop 后先等 actor 收敛；超时写 provisional `turn_unknown`，并可由 `get` / `prepare` 修复。

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
| 在读取接口里实时读历史算 blockers | 每个 Pending 节点每 1.5 秒读一次整文件，轮询成本和缓存无法接受 |
| park 分两次写快照与状态，再配失败标记/孤儿清理 | 两次写会引入「Pending 但快照缺失」的窗口和额外规则；单事务 guarded UPDATE 已足够 |
| 用布尔值“本轮有 TurnEnded”当 turn key | 不能唯一标识回合，无法发现两轮之间的错配 |
| 用事件流累积文本/stop reason 作为快照输入 | 事件流没有 `HistoryLine.seq`，无法推导 turn key；且 `prepare` 读历史，两条路径输入不同源 |
| 在 payload 里存 `structuredOutput` | 增加 payload 体积和暴露面，还要定义离开 `Pending` 的删除事务；改为只存 digest，`prepare` 用同一段文本重跑 schema |
| cleanup 在流 drop 后立即写 `turn_unknown` | actor 可能还没写 `TurnEnded`，会把实际 `EndTurn` 的一轮误判且无法修复；应先有界等待，超时用 provisional + 修复 |

## 落地顺序

1. **后端门禁与事实**：D1 的 stop reason/输出校验与 `HistoryIntegrity`、D2 的 `{userSeq, endedSeq}` 与窗口文本、D3 的共享 park 函数与单事务写入、D4 的 cleanup 等待/开门、D5 的 complete/force 审计与 `WorkflowNodeTurnChanged`；升级前旧 run 的 lazy materialize 随本阶段上线。
2. **前端入口**：D7 的 trailing slot、当前步骤卡片、按钮/徽标与中英文案；`WorkflowNodeTurnChanged` 时刷新运行。
3. **验证与文档**：单测矩阵、重启 restore/rebuild 集成测试、e2e 两条复现路径、前端测试；同步 `docs/workflows/workflow-runs.md` 与 `.zh.md` 的 Interactive nodes / Recovery after restart。
4. **后续能力**：`completion.files` / `criteria`、运行时验收清单、LLM judge 与 agent 自报完成另开 issue；真正无法继续时的「重新运行本步骤」也另开 issue，不阻塞本决策。

## 风险、未解决问题与验收

- **每次交互回合读一次整文件历史**：turn key 依赖 `HistoryLine.seq`，事件流不提供，park 必须读历史；这是本决策接受的成本，长会话是 O(会话) 的读取。后续若 runtime 能提供本轮的事件 id/文本，可再降，但不能改变“park 与 prepare 同源”的要求。
- **provisional 快照修复**：流 drop 且等待超时会先写 `turn_unknown`；`get` / `prepare` 在同一个 `userSeq` 之后看到 `TurnEnded` 时替换。修复窗口内界面显示未完成，用户仍可 `force`。
- **升级前的 Pending 节点**：新 run 的 park 是单事务，不会出现 `Pending` 无快照；旧 run 由 `get` 与 `prepare` 共用的 lazy materialize 处理，两条路径幂等。
- **`structuredOutput` 不落 payload**：避免体积和暴露面；`prepare` 用 digest 校验过的文本重跑 schema（纯 CPU）。如果 schema 校验耗时异常，后续再单独优化，不把结果塞回投影。
- **per-run 门不可重入**：park 函数自己获取门；executor 与 cleanup 调用点都不得持有门。`get` 的 lazy materialize 同样先读历史、再获取门写 payload。
- **UI 卡片挂载**：`ChatView` 目前只有 `contextBar` 和 `composerActions`，需要新增可选 trailing slot；这是共享组件改动，实现时要保证普通聊天不受影响。
- **区域 HITL**：本决策不改变 foreach 的拒绝规则，也不补齐 while 区域内的整链证据；该缺口仍由现有区域用例记录。
- **不在本决策范围**：`completion.files` / `criteria`、运行时验收清单、LLM judge 与 agent 自报完成均另开 issue，可依赖本决策。

验收：

- 停止生成 → 重启 → 点完成：返回 `turn_stopped`，不调度下一节点；「继续当前步骤」在 `session/load` 成功/失败→rebuild 两条路径都能得到一轮回复；continue 失败时明确报错、节点保持 `Pending`、仍可强制完成；
- `MaxTokens` / `MaxTurnRequests` / `Refusal` / 未知 stop reason、以及 `HistoryIntegrity` 非 `Complete` 同样不能直接完成；
- 本轮最终文本按 `(userSeq, endedSeq]` 窗口取最后一条 assistant 消息；空文本不会被早先的进度说明掩盖；
- 正常 `EndTurn` 后完成与现在一致；无契约节点空输出仍可完成，只显示 warning；
- 首轮 executor park 与人类 cleanup 都经过同一个函数；`payload.completion` 与 `Running → Pending` 同事务提交，新 run 不会出现 `Pending` 无快照；
- `{userSeq, endedSeq}` 从历史推导；complete 重新推导并比对，不一致返回 `WorkflowNodeTurnChanged`；
- `Structured` / `Text` 校验在 park 完成，blockers 与 complete 同源；`outputDigest` 不一致时完成返回冲突；`structuredOutput` 不落 payload；
- 无快照旧节点的 `get` 与 `complete` 两条 lazy materialize 路径都幂等；
- cleanup 在流 drop 后先有界等待；超时写 provisional `turn_unknown`，后续 `get` / `prepare` 能修复；
- 点击前能看到 blockers；`Running` 时快照被忽略；
- 区域 interactive 行为不变；核心用例在 `approved` 后补到 `test-cases/desktop/core/workflow/interactive-node-completion.md`。
