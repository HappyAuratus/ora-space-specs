---
status: proposed
---

# 交互式节点：agent 显式完成信号驱动可完成，人工确认后推进

交互式 Agent 节点目前只要本轮以 `EndTurn` 结束且 blockers 为空就视为 ready；但 agent 可能用 `EndTurn` 结束一个「向用户提问」的回合，界面于是把提问当成完成。本决策给交互式 agent 节点增加一个由 host 提供的显式完成信号：workflow 节点会话额外获得一个只在该会话可用的完成工具，agent 认为本步骤完成时调用它；host 把「当前 turn 的完成请求」写进完成快照，只有当前 turn 的请求被接受后节点才可完成，人仍在步骤卡片上确认（或按节点配置自动接受）。没有信号时 `EndTurn` 不再让节点 ready；首轮 gate、异常停靠与 `force` 的既有语义继续适用。

本决策是 `proposed`，只定义 host 侧契约与行为，不定义 agent 插件如何实现工具调用；`Loop`/`Iteration` 区域、非交互节点、Cloud 与既有 workflow 包格式不在范围内。

## 继承与修改

继承[交互式节点完成契约](20261021-interactive-node-completion.md)（下称 #385 ADR）、[首轮确认与异常停靠](20261102-first-turn-confirmation.md)与[交互式节点产出物声明](20261023-interactive-deliverable-contract.md)。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| #385 D1：只有本轮 `EndTurn` 且无 blocker 才能完成 | 修改 | 对 `completionSignal: agent` 的节点，还要求当前 turn 的 agent 完成请求；人仍可继续对话或 force |
| #385 D5：完成校验按状态 → turn key → digest → blockers | 扩展 | 增加「当前 turn 的 agent 请求」校验与 `confirmAgentCompletion` 请求字段；`force` 仍只覆盖非确认类 blocker |
| 首轮 ADR D1：自动首轮写 `turn_unconfirmed` | 继承 | 首轮 gate 继续适用；agent 请求与 `turn_unconfirmed` 同时存在时，一次人工确认解除两者 |
| 首轮 ADR D6：`#427` 定义 agent 请求的 requester、权限、幂等与审计 | 落实 | 本决策定义信号载体、请求生命周期、确认与审计；不复用 `confirmInitialTurn` 冒充 agent 请求 |
| #728 方向 2：只有 agent 显式完成信号能让回合变成「可完成」 | 采纳 | 信号缺失时节点保持「等待参与」，不显示普通完成 |
| 产出物声明 D2–D4：文件 blockers 在 `prepare` 按现场 workspace 判定 | 继承 | 信号不能越过文件 blockers；补齐文件后仍需确认 |

## 决策

### D1：完成信号由 host 注入的 MCP 工具承载

workflow 执行器为每个 interactive agent 节点会话额外注入一个 host 提供的 MCP server（本地 loopback HTTP，带一次性 session token），暴露一个工具：

```text
ora_complete_workflow_step()
  -> { "accepted": true }
```

- 工具只在 interactive agent 节点会话中注入；非交互节点、普通对话会话、Cloud 执行不注入。
- 工具调用由 host 的 MCP server 接收，按 token 解析出 session，再解析出该 session 绑定的 `WorkflowNodeRun`；只接受当前处于 `Running` 且绑定该 session 的交互节点。
- 工具不读写用户文件、不执行命令，只记录「当前 turn 的完成请求」，与 agent 是否真的产出无关；是否推进仍由完成后校验决定。
- 信号是显式协议调用，不解析 assistant 文本，不做语义推断。
- 节点配置新增 `completionSignal: "agent" | "end_turn"`，默认 `agent`。`end_turn` 是遗留模式，只有显式配置或升级前冻结的图才使用。

工具调用不阻塞 turn；agent 可以在调用后继续输出，也可以在调用后直接结束本轮。

### D2：`ready` 只由当前 turn 的已接受请求驱动

完成快照新增：

```json
"agentSignal": { "turnKey": { "userSeq": 3, "endedSeq": 4 }, "requestedAt": "..." }
```

- 对 `completionSignal: agent` 的节点，`ready` 额外要求 `agentSignal.turnKey` 等于当前快照的 `turnKey`；没有信号或信号属于旧 turn 时 `ready = false`。
- `turnKey` 在 turn 结算后才确定；工具在 turn 进行中收到的请求先记为待绑定，park 时绑定到刚结算的 turn。turn 结算后到达的旧请求按 `stale_turn` 丢弃，不能用于下一轮。
- 缺少当前 turn 信号时，普通完成返回 `WorkflowNodeIncomplete`，blocker 为新增的 `agent_completion_pending`；`confirmInitialTurn` 不能解除它。
- `agent_completion_pending` 允许 `force` 覆盖并写强制审计，作为 agent 不调用工具时的唯一人工出口；这与首轮 `turn_unconfirmed` 不同——后者必须显式确认。
- 信号只证明「agent 认为完成」，不替代 turn key、digest、输出契约与文件 blockers 的校验。

### D3：人工确认与可配置的自动接受

步骤卡片按信号状态展示：

| 状态 | 徽标 | 主操作 | 次操作 |
| --- | --- | --- | --- |
| 有当前 turn 信号 | 等待确认 | 确认并完成（`confirmAgentCompletion: true`） | 继续当前步骤 |
| 无信号 | 等待参与 | 继续当前步骤 | 仍然完成（force，二次确认） |
| 首轮且无信号 | 等待参与 | 继续当前步骤 | 仍然完成（force） |
| 首轮且有信号 | 等待确认 | 确认并完成（同时带 `confirmInitialTurn: true`） | 继续当前步骤 |

- 人工确认是一次普通完成，不写 `completion_forced`；payload 记录 `confirmation: { by: "human", at }`。
- 请求字段新增可选 `confirmAgentCompletion: bool`，只解除 `agent_completion_pending`；与 `force` 同时为 true 返回 `invalid_request`，与首次确认字段可以同时出现。
- 节点配置 `autoAcceptAgentCompletion: false` 默认；为 true 时，已接受的信号直接完成节点，`confirmation.by = "auto"`，仍不做 force 审计。
- 重复确认幂等；确认时信号已被新 turn 覆盖则返回 `WorkflowNodeTurnChanged`。

### D4：请求生命周期、幂等与并发

- 每个 `(node_run, turn)` 只接受第一个请求；重复调用返回 `accepted: true` 且不改变快照。
- 非绑定 session、非 interactive 节点、节点不在 `Running`、turn 已结束后调用，返回结构化错误，不改变任何状态。
- 请求在 turn 结束前到达、park 发生于同一 turn 时绑定；park 后到达的请求属于旧 turn，返回 `stale_turn`。
- 请求后用户继续对话：新 turn 没有信号，节点仍不可完成；旧信号不跨 turn 复用。
- 节点崩溃/超时/取消导致失败清理时，丢弃未绑定的请求；不写入完成快照。

### D5：审计与 payload

完成后的 `payload.completion` 增加：

```json
"agentSignal": { "turnKey": { "userSeq": 3, "endedSeq": 4 }, "requestedAt": "..." },
"confirmation": { "by": "human", "at": "..." }
```

- `completion_forced` 只在 `force` 实际覆盖 blocker 时出现，形状沿用首轮 ADR D3；`agent_completion_pending` 被 force 覆盖时记录在 `blockers`。
- 不新增 `completion_confirmed` 一类第二个审计键；`confirmation` 只描述本次确认的来源。
- 不记录工具调用的原始文本或用户自由文本。

### D6：兼容与迁移

- 升级前冻结的图没有 `completionSignal` 字段：按 `end_turn` 遗留语义执行，不追溯改变已停靠节点。
- 升级后新建或重新保存的交互节点默认 `agent`；编辑器在节点配置中明示，允许显式改回 `end_turn`。
- 旧快照没有 `agentSignal`：读取投影按 legacy 处理；`prepare` 只在新语义节点上重新物化。
- 不支持工具调用的 agent：节点不会 ready，用户可以继续对话或 force；不在本决策内做 agent 探测或自动降级。是否给这类 agent 一个显式开关由后续决策处理。
- `completionSignal` 是 wire 契约的增量；旧客户端不认识它时按 `end_turn` 处理，不能因为不认识而误开新语义。

## 不变量

1. `completionSignal: agent` 的节点，没有当前 turn 的已接受请求时 `ready = false`，普通完成返回 `WorkflowNodeIncomplete`。
2. 信号只对 owning session 的当前 turn 生效；旧 turn、其他 session、非 interactive 节点一律拒绝，不改变状态。
3. 信号不绕过状态/claim、turn key、`outputDigest`、输出契约或文件 blockers。
4. 人工确认是普通完成，不写 `completion_forced`；force 覆盖 `agent_completion_pending` 时按既有形状写审计。
5. 首轮 gate 与信号同时适用；一次确认可以同时解除 `turn_unconfirmed` 与 `agent_completion_pending` 中的可解除部分。
6. 自动接受只在节点显式配置时启用，并记录 `confirmation.by = "auto"`。
7. 重复请求幂等；park 之后的请求返回 `stale_turn`。
8. `completionSignal: end_turn` 的节点与升级前冻结图保持 #385 语义不变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| structured output 里的完成字段 | 不覆盖 text 契约节点；用户 schema 可能 `additionalProperties: false`，加保留字段会破坏校验；无法在 turn 中途表达 |
| assistant 文本中的完成标记 | 需要解析文本并剥离标记，模型可能漏写或误写；与「不解析文本判断意图」的既有边界冲突 |
| 自定义 ACP session update | 需要每个 agent 插件实现新的协议消息，存量 agent 不支持；无法跨 agent |
| CLI 命令 | agent 需要拿到 run/node 上下文、凭据和可执行路径，权限面最大；沙盒与出网边界复杂 |
| 继续只靠 `EndTurn`，用文案提示 | 不解决「提问回合被当成完成」的误判，只是把问题留给用户点击 |

## 落地顺序

1. **ADR 评审**：本决策从 `proposed` 评审到 `approved`，同步核心用例。
2. **契约**：`NodeCompletionRequester` 增加 `Agent`；节点配置增加 `completionSignal` / `autoAcceptAgentCompletion`；完成请求增加 `confirmAgentCompletion`；新增 blocker kind `agent_completion_pending`。
3. **后端**：workflow 会话注入完成 MCP server 与 token；工具调用绑定 session→node→turn；完成快照/投影/`prepare` 支持 `agentSignal` 与 `confirmation`；完成门禁与 force 审计按本决策。
4. **前端**：步骤卡片按信号状态展示确认/继续/force；节点配置暴露 `completionSignal` 与自动接受；中英文案同步。
5. **验证**：MCP 工具调用、重复/跨 turn/非绑定拒绝、首轮组合、force 覆盖、自动接受、legacy 节点与旧快照、崩溃清理。

## 风险、未决问题与验收

- **工具注入面**：需要 host 运行一个本地 MCP server 并给每个会话发 token；权限校验必须只认 token 绑定的 session，不能让同一台机器的其他进程冒充。
- **agent 不调用工具**：没有信号时节点不可普通完成，只能继续或 force；这是有意的安全默认，但会让不遵守提示词的 agent 体验变差。是否按 agent 能力自动降级留给后续决策。
- **信号时序**：工具调用可能早于/晚于 turn 结束；park 必须把请求绑定到刚结算的 turn，晚到的旧请求丢弃。
- **与首轮 gate 的组合**：首轮有信号时仍要 `confirmInitialTurn`；一次确认同时带两个字段，错误优先级要固定（形状检查 → 状态/claim → turn key → digest → blockers）。
- **兼容**：升级前冻结图保持 `end_turn`，不追溯；编辑器默认 `agent` 会让新图获得新语义。

验收：

- 调用完成工具后，当前 turn 的 park 快照带 `agentSignal`，卡片显示「确认并完成」；确认后节点完成，payload 记录 requester=Agent 与 confirmation.by=human，无 `completion_forced`。
- 不调用工具时（`completionSignal: agent`），普通完成返回 `agent_completion_pending`；`confirmInitialTurn` 不能解除；force 可以覆盖并写审计。
- 重复调用幂等；旧 turn、其他 session、非 interactive 节点、park 后的请求都被拒绝且不改变状态。
- 首轮有信号时，确认同时带 `confirmInitialTurn`；没有信号时首轮仍按原 gate 等待确认。
- 自动接受配置下，信号直接完成并记录 `confirmation.by = "auto"`。
- `completionSignal: end_turn` 的节点、升级前冻结图与旧快照行为不变。
