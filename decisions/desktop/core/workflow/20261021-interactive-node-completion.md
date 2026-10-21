---
status: proposed
---

# 交互式节点完成契约：以本轮终态门禁推进，暂停可继续，强制完成可审计

交互式 Agent 节点每轮结束后停在 `Pending`。本决策把「完成」从一个人工动作改为由本轮终态支撑的门禁：只有本轮以 `EndTurn` 结束、且声明的输出契约通过时才允许完成；`Cancelled`、截断、拒答、缺失或未知都视为未完成，必须继续对话或显式强制完成。完成判定在 per-run 锁内以持久状态和 turn key 复核，park 快照进入读取投影；会话消息不被拦截，由后端在每个回合前注入步骤边界。

本决策为 `proposed`。它改变 `completeWorkflowNode` 的公开语义（新增 `WorkflowNodeIncomplete` 与 `force`），涉及 backend 的完成/停靠路径、contracts 的完成请求、application 的节点映射和 app-shell 的运行视图；数据库只读写既有 `payload` 列，不改 schema、冻结图字段或本地文件布局。正常 `EndTurn` 后的完成行为保持不变；`Loop`/`Iteration` 区域内的 interactive 行为不在本决策范围，仍按既有区域边界处理。升级前已停靠且无快照的节点按 D3 的一次性物化处理。

## 继承与修改

本决策继承[节点运行时编排决策](0-node-runtime-orchestration.md)，并沿用[迭代节点](20260915-iteration-composite-runtime.md)、[prompt 静默策略](20261008-prompt-inactivity-policy.md)与[while-Loop](20261013-while-loop-with-durable-round-scopes.md)的边界。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 根 D2：同一 run 的状态变化串行并事务性提交，等待不跨越 per-run 门 | 继承，不变 | `completeWorkflowNode` 在门内只比对持久状态、claim 与 turn key；历史、schema、diff 留在 `prepare` 的既有门外边界 |
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

节点正在 `Running` 时不能完成，沿用现有 `workflow_node_not_awaiting_input`，不新增 `turn_in_progress` blocker。声明 `outputContract=Structured` 的节点在完成时按 schema 校验，失败返回 `structured_invalid`，raw 输出仍落库；`Text` 要求本轮最终输出非空（`output_missing`）；无契约（`None`）只校验回合终态，空输出只记非阻塞 warning。编辑器目前只能声明 `Structured`，`Text` 只用于旧快照或导入图。

选择只认 `EndTurn`，是因为截断、停止和拒答都意味着产出不完整；把 `MaxTokens` / `MaxTurnRequests` 当作自然结束会让暂停后重启的问题在 token 截断时静默复发。`None` 不检查空输出，是为了不误伤只改文件、最终回复为空的既有节点。

### D2：以最后一条用户消息界定本轮，turn key 阻止复用旧回合

「本轮」指最后一条用户消息之后、最后一条 `TurnEnded` 之前的部分。park 时从历史推导 turn key（最后一条用户消息之后是否存在 `TurnEnded`），并随快照落盘；`admission` 不写库。若本轮没有终态（流被丢弃、adapter 拒绝、连接断开），判 `turn_unknown`，不得读取上一轮的 `EndTurn` 或上一轮的 assistant 文本。

当前 `last_stop_reason` 会返回历史中任意最后一条 `TurnEnded`，`assistant_output_from_history` 也会取最后一条非空 assistant 文本；两者在「最新一轮没有结束」时都会指向旧回合。暂停后重启的另一个入口正是这里：节点仍是 `Pending`，旧的 `EndTurn` 会让它看起来 ready，并把旧输出当成节点产出。turn key 同时是 `complete` 在锁内比对的一致性版本。

### D3：park 时写完成快照，读取投影只解析快照

park 时计算并持久化：

```json
{
  "completion": {
    "turnKey": "<本轮标识>",
    "stopReason": "end_turn | cancelled | ...",
    "blockers": [{ "kind": "turn_stopped", "params": {} }],
    "warnings": []
  }
}
```

写入节点 run 的 `payload.completion`，**先写快照，再走 `Running → Pending`**，使观察者看到 `Pending` 时一定能读到快照。写入失败时尽力写一个可区分的失败标记（例如 `completion: { "write_failed": "..." }`）；无论标记是否写成，新 run 的无快照 `Pending` 都按 `turn_unknown` 投影，不回退读历史。孤儿快照（快照写入成功但 `Running → Pending` 被守卫拒绝）只允许投影在 `Pending` 时读取；节点离开 `Pending` 时删除 `completion`（`completion_forced` 与 blockers 审计另存）。新回合开始时清掉或覆盖旧快照，`Running` 期间投影必须忽略快照。

读取投影（`GetWorkflowRun` 的节点状态）只解析 `payload.completion`，不读会话历史、不跑 schema。原因有两条：`map_node_run` 是纯字段映射，拿不到 `sessions_root` 与历史；run detail 以 1.5 秒轮询，实时读历史或跑校验的成本与缓存都无法接受。不新增 `check` 接口，`completeWorkflowNode` 仍是唯一权威，投影只是同一份快照的只读视图。

新 run 在创建或启动时写 `payload.completion_contract = 1`。投影据此区分两种「无快照」：新 run 的 `Pending` 无快照视为写失败，直接 `turn_unknown`，不回退读历史；没有该版本的旧 run 才由 backend 命令层做一次 lazy materialize（读一次历史、计算并回填；读失败也写 `turn_unknown` 快照，避免反复重试）。

### D4：人类回合的 stop reason 有两个写入点

`Completed` 事件带 stop reason，但它不是每条结束路径都有：`Cancel { operation_id }`、连接丢失、队列溢出和事件通道关闭只写 `TurnEnded`。因此人类回合必须：

1. 在 `WorkflowSessionTurns::prompt` 观察 `Completed`，用事件里的 stop reason 写快照；
2. 在 cleanup（`attach_cleanup` → `end_human_turn`）里**仅当本轮还没有快照**时读一次历史兜底：有 `TurnEnded` 就按它写，没有就写 `turn_unknown`。

cleanup 分支必须幂等，不能把正常完成的一轮覆盖成 `turn_unknown`。admission 需要携带算快照所需的 `sessions_root`、冻结图与会话 id；当前 `HumanTurnAdmission::Workflow` 只有 node run id 与 `prompt_inactivity`，不足以在 cleanup 重建快照。

只写 `Completed` 会漏断连和放弃流，退回暂停后重启的旧 bug；只写 cleanup 读历史则每轮多一次整文件读。`Completed` 先于 cleanup 到达（actor 先写 `TurnEnded` 再发 `Completed`），因此正常情况下 cleanup 能看到已有快照并跳过读取。

### D5：`completeWorkflowNode` 在锁内做权威校验，force 必须留痕

`completeWorkflowNode` 在 per-run 锁内重校验：run 仍 `Running`、节点仍 `Pending`、completion claim 仍持有、`turn key` 与 `prepare` 记录一致。不满足且没有 `force` 时返回 `WorkflowNodeIncomplete { node_id, blockers[] }`；`force` 必须由用户二次确认，并在 payload 记录 `completion_forced` 与 blockers 快照。`completion_forced.reason` 只记 blockers 快照，不混入用户自由文本。

历史、schema 校验与 worktree diff 继续留在 `prepare_completion` 的门外；锁内只比对版本与状态，不重跑整套校验。`turn key` 比对是防御性一致性检查：`claim_node_for_completion` 已经在同一把锁下阻止 `prepare` 之后插入新回合，它不是主要保护；真正要处理的是快照写失败/延迟和升级前无快照。

structured 校验失败时 `force` 不写 `structured_output`，raw output 仍保存；下游引用 `{node}.structured_output` 按变量缺失处理，由现有变量模板校验报错，不引入静默降级。payload 合并不得覆盖 `error_detail` / `ai_diagnosis`。

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
3. 读取投影只解析 park 快照，不在读取路径读会话历史或执行 schema 校验。
4. `completeWorkflowNode` 是唯一权威；UI 的 ready/blockers 只是同一份快照的预览。
5. `force` 必须二次确认并留痕；结构化失败时 force 不写 `structured_output`。
6. 用户消息不被拦截或改写；每个 workflow 人类回合都带步骤边界。
7. 正常 `EndTurn`、无契约空输出、既有平图行为保持不变。
8. 区域 interactive 行为不变；本契约只覆盖外层平图。
9. `Pending` 跨重启保留；在途 `Running` 仍按既有失败边界处理。

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
| 在读取接口里实时读历史算 blockers | 每个 Pending 节点每 1.5 秒读一次整文件，和“重启后不依赖历史”的目标相反 |

## 落地顺序

1. **后端门禁与事实**：D1–D6（含人类回合的步骤边界）、D3 的 `completion_contract` 版本标记与升级前 lazy materialize、D5 的 complete/force 审计随本阶段上线。
2. **前端入口**：D7 的 trailing slot、当前步骤卡片、按钮/徽标与中英文案。
3. **验证与文档**：单测矩阵、重启 restore/rebuild 集成测试、e2e 两条复现路径、前端测试；同步 `docs/workflows/workflow-runs.md` 与 `.zh.md` 的 Interactive nodes / Recovery after restart。
4. **后续能力**：`completion.files` / `criteria`、运行时验收清单、LLM judge 与 agent 自报完成另开 issue；真正无法继续时的「重新运行本步骤」也另开 issue，不阻塞本决策。

## 风险、未解决问题与验收

- **整文件读历史的成本**：正常轮次由 `Completed` 事件提供 stop reason，只有 cleanup 且没有快照时才读历史；park 快照仍需要本轮最终 assistant 文本，长会话是 O(会话) 的读取。本决策接受这个成本，后续可把本轮文本从 runtime 带出来再降。
- **升级前的 Pending 节点**：由 D3 的一次性 lazy materialize 处理；历史读取失败时写 `turn_unknown`，不反复重试。
- **快照写失败与孤儿快照**：D3 已给出失败标记、只在 `Pending` 读取、离开 `Pending` 清理的规则。
- **UI 卡片挂载**：`ChatView` 目前只有 `contextBar` 和 `composerActions`，需要新增可选 trailing slot；这是共享组件改动，实现时要保证普通聊天不受影响。
- **区域 HITL**：本决策不改变 foreach 的拒绝规则，也不补齐 while 区域内的整链证据；该缺口仍由现有区域用例记录。
- **不在本决策范围**：`completion.files` / `criteria`、运行时验收清单、LLM judge 与 agent 自报完成均另开 issue，可依赖本决策。

验收：

- 停止生成 → 重启 → 点完成：返回 `turn_stopped`，不调度下一节点；「继续当前步骤」在 `session/load` 成功/失败→rebuild 两条路径都能得到一轮回复；continue 失败时明确报错、节点保持 `Pending`、仍可强制完成；
- `MaxTokens` / `MaxTurnRequests` / `Refusal` / 未知 stop reason 同样不能直接完成；
- 正常 `EndTurn` 后完成与现在一致；无契约节点空输出仍可完成，只显示 warning；
- 结构化失败返回可读 blockers，raw output 落库；force 不写 `structured_output`，payload 有 `completion_forced` 审计；
- 点击前能看到 blockers，且与 complete 返回的 blockers 同源；`Running` 时快照被忽略，新回合清掉旧快照；
- 升级前无快照节点按 lazy materialize 处理；快照写失败、孤儿快照的行为有测试；
- 区域 interactive 行为不变；核心用例在 `approved` 后补到 `test-cases/desktop/core/workflow/interactive-node-completion.md`。
