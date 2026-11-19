---
status: approved
---

# 扩展共享 ACP 运行时识别 CodeAgent _subagent/update 通道，复用现有子代理投影与历史模型

共享 ACP 运行时与历史装配新增支持华为 CodeAgentCLI 的 `_subagent/update` 扩展通知通道：
`ora-acp` 保持 provider 中立并按线序透传 `_` 开头的 ACP 扩展通知（`AcpInboundEvent::ExtensionNotification`）；
`ora-agent-runtime` 在连接路由入口识别 `_subagent/update`，校验允许的 `SessionUpdate` 变体、
对子代理局部 `toolCallId` 做作用域隔离并剥离逐帧 `timestamp` 后规范化为带 `_meta["_subagent/update"]`
的 `SessionUpdate`；`ora-history`、`PromptLiveness`、运行诊断、前端只读子代理树以及非交互工作流
输出累加共用同一条子代理来源规则，沿用现有 JSONL schema v2、回放/交接与公共契约，不做存储迁移。

本文处于 `approved`（已依据 `codeagentcli 1.2615.02.05` / ACP `1.2605.00` 启用
`CODEAGENT3_ACP_SUBAGENT_CHANNEL=true` 后的真实带时间戳帧序列锁定 D2 与 D4 规则，并已在
[子代理投影、终态缺口与运行诊断](../../../test-cases/node/agent-runtime/child-projection-and-diagnostics.md)
补充对应核心验证义务）。影响 `ora-acp`、`ora-agent-runtime`、`ora-history`、`@ora/chat`、
`@ora/app-shell` 及工作流输出累加边界；不改动 `PromptSessionEvent` / `LoadSessionEvent` 结构，
不改变既有 OpenCode `_meta["opencode/child-session"]` 行为，不迁移旧历史文件，也不引入独立 JSONL
记录类型或子代理控制入口。

## 继承与修改

本决策直接继承并扩展[保留子代理投影来源，以只读详情和真实观察呈现运行状态](20261020-child-projection-history-and-diagnostics.md)，
并沿用它所依赖的[插件拥有模型 adapter](20261017-plugin-owned-model-adapters.md)、
[共享运行时根决策](0-shared-agent-runtime-crate-hosted-by-node.md)与
[静默策略](../../desktop/core/workflow/20261008-prompt-inactivity-policy.md)。同时已在
`20261020` 标题下补充 `> 历史决策：` 提示，不回改其正文其余部分。

| 前序约定（`20261020`） | 处理 |
| --- | --- |
| 宿主拥有进程、会话与授权生命周期，不新增子代理宿主管理实体 | 继承，不变 |
| JSONL 与 Thread 使用同一共享 history，不新增记录类型或迁移旧文件 | 继承，不变；新增来源继续写入 `SessionUpdate` metadata |
| D1：首期只识别 `_meta["opencode/child-session"]`，父子关系只来自显式来源，未知扩展不透明保留 | 修改（扩展）：新增受支持子代理来源 `_meta["_subagent/update"]`（在连接边界从 `_subagent/update` 扩展通知规范化生成）；其他未知 `_` 开头扩展通知不伪造为子代理内容 |
| D1：按 `id`/`parentID` 与正整数 `depth` 连接多层嵌套 | 限定（已知限制）：`_subagent/update` 仅提供外层 `sessionId`/`taskId` 与可选父 `toolCallId`，缺少子任务间父子链，统一按 `depth: 1` 展平为同级只读卡片 |
| D2：文本/思考保留完整 `ContentChunk`、metadata 与注解，禁止跨来源拼接；不同子来源 `plan` 互不覆盖；父回合结束不为有效子来源未决工具生成 `completed` | 继承并细化：生成 `_meta["_subagent/update"]` 时剥离逐帧 `timestamp` 以保证连续 chunk 合并；内层 `toolCallId` 按 `subagent:{len}:{taskId}:{innerToolCallId}` 作用域隔离；实时与历史统一按来源 ID 隔离 `plan` 与未决工具 |
| D3：子组不推断终态；父回合结束后只把明确完成的前台委托标为「委托已返回结果」，其余标为「后续未知（投影已停止）」 | 修改（扩展，已由真实帧锁定）：`_subagent/update` 不依赖 OpenCode 的 `rawOutput.metadata.sessionID`，改由外层 `toolCallId` 关联父流委托工具，并通过 `ora/turn-end-settled` / `settledByTurnEnd` 与 `content`/`rawOutput` 规则区分 provider 完成与回合收束补态 |
| D4：有效子来源活动与增量子工具不重置或暂停父 prompt 无活动计时；归属、liveness 与运行诊断共用同一来源规则 | 继承，不变：`PromptLiveness`、`ora-history`、`@ora/chat` 诊断与 `SubagentGroup` 共用同一有效来源函数，显式清除、无效或冲突来源退回父流 |

## D1：`ora-acp` 保序透传 `_` 开头 ACP 扩展通知

当入站 JSON-RPC 2.0 帧没有 `id` 且 `method` 以 `_` 开头时，`ora-acp` 必须发出
`AcpInboundEvent::ExtensionNotification { method, params }`，投递到与 `SessionUpdate`、
`PermissionRequest` 和 `SessionResponse` 相同的无界入站 FIFO，严格保持线序。非 `_` 开头的未知
通知继续静默丢弃；带有 `id` 的 `_` 开头请求仍按未知客户端方法回复 JSON-RPC `-32601` 而不断开连接。

选择在 `ora-acp` 保留原始 `method` 与 `params`、不直接解析 `_subagent/update` 的原因有两个：

- **传输层与 provider 协议解耦**：ACP 规范将 `_` 前缀保留给任意自定义扩展（`extNotification`）。
  `ora-acp` 只负责 JSON-RPC 帧边界与协议合规，不把 CodeAgent 私有 schema 编码进公共传输包。
- **杜绝乱序竞态**：若扩展通知走独立通道或异步任务转发，父流 `session/update` 中的委托 `tool_call`
  或 `end_turn` 响应可能先于（或晚于）子代理进度到达路由层，导致实时渲染与历史落盘顺序漂移。

## D2：连接路由入口校验、隔离并规范化 `_subagent/update`

真实 CodeAgentCLI ACP 帧（`codeagentcli 1.2615.02.05` / ACP `1.2605.00`）已实测确认：
`_subagent/update` 是顶层 ACP `extNotification`（非 `session/update` 包装），其 `params` 顶层
包含 `{ sessionId, taskId, toolCallId, agentType, update, timestamp }`，其中 `params.sessionId`
等于 `session/new` 返回的**父 ACP 会话 ID**（实测 6/6 一致），`params.toolCallId` 等于父流
`Agent`/`Task` 委托工具调用的 `toolCallId`（实测 6/6 一致）。因此 `sessionId` 确定同时作为
`SessionNotification::new(session_id, update)` 的连接路由键和只读子代理分组的 `parentId`。

在共享运行时的连接事件循环（`run_process_generation`）中，收到 `AcpInboundEvent::ExtensionNotification`
且 `method == "_subagent/update"` 时，必须在同一分支内同步翻译为标准 `SessionNotification` 后再送入
`routes.route_event`，确保翻译不破坏线序；其他未被运行时识别的 `_` 开头扩展通知直接忽略。

翻译边界遵守以下规则：

1. **必填与可选字段校验**：`params` 必须为 JSON 对象；`sessionId`、`taskId` 必须为 trim 后非空字符串；
   可选字段 `toolCallId` 与 `agentType` 若存在则必须为字符串或 `null`（空白字符串按缺失处理）；
   `update` 必须为包含有效 `sessionUpdate` 字段的 JSON 对象。任何校验失败或不受支持的帧只记录
   警告并丢弃，不得断开共享 Agent 进程连接。
2. **允许的 `sessionUpdate` 变体白名单**：仅接受 `agent_message_chunk`、`agent_thought_chunk`、
   `plan`、`tool_call` 与 `tool_call_update`。拒绝 `session_info_update`、`current_mode_update`、
   `config_option_update`、`available_commands_update`、`usage_update` 等会话级状态更新，防止
   子代理进度帧改写父会话标题、模式或用量快照。
3. **剥离易变字段并生成稳定归因**：在目标 `SessionUpdate` 的 `_meta["_subagent/update"]` 写入
   `{ sessionId, taskId, toolCallId?, agentType? }`，刻意不写入外层逐帧 `timestamp`。否则相邻
   文本或思考 chunk 会因 `_meta` 不相等而无法合并，破坏 `ContentChunk` 拼接与 JSONL 紧凑性。
   每个进程代连接按 `(sessionId, taskId)` 维护有界 FIFO 归因缓存（`MAX_CACHED_TASKS = 4096`），
   将首帧提供的 `toolCallId` 与 `agentType` 补全到同一任务后续省略了可选字段的帧上；达到容量
   上限时仅淘汰最早入队的条目，不得整表清空正在运行的任务。对于缺少非空 `title` 的子代理
   `tool_call`，可按 `_meta.codeAgentCLI.toolName` 或固定占位标题回退。
4. **内层 `toolCallId` 作用域隔离**：子代理在独立模型轮次执行，网关可能重新从 `call_0`/`call_1`
   编号。翻译层必须按所属 `taskId` 对内层 `update.toolCallId` 做无歧义的长度前缀命名空间编码
   （`subagent:{len}:{taskId}:{innerToolCallId}`，其中 `len` 为 `taskId` 字节长度），防止与父会话
   或兄弟子代理的工具 ID 在 `PromptLiveness`、`HistoryAssembler` 与 UI 键值中碰撞，并消除 `taskId`
   自身含 `:` 时的二义性。
5. **归因命名空间权威性**：翻译层接管 `_subagent/update` 通道的来源归因，插入 `_meta["_subagent/update"]`
   前必须剥离内层载荷携带的其他已知子来源或回合收束内部标记（如 `opencode/child-session`、
   `ora/turn-end-settled`），避免与 `_subagent/update` 冲突导致统一来源规则把该工具退回父流。

## D3：单一子代理来源规则、已知展平限制与父输出隔离

Rust（`ora_history::child_source_id` / `has_child_source_namespace`）与 TypeScript
（`childSessionSource` / `hasChildSessionNamespace`）各只保留唯一一处子来源判别实现，同时识别
`opencode/child-session` 与 `_subagent/update`：

- `PromptLiveness`、`HistoryAssembler`、`@ora/chat` 运行诊断（`observeDiagnostics`）、
  `SubagentGroup` 分组以及工作流输出提取全部消费同一规则；任何消费端不得再硬编码单个 provider
  命名空间。
- 显式 `null`、缺失必填字段、类型非法或双命名空间冲突一律视作无效来源并退回父流（对应 tool ID
  从子代理归属清除并恢复父工具行为），沿用 `20261020` 的有效来源规则。
- **已知限制（嵌套展平）**：`_subagent/update` 当前未携带子任务之间的父子层次链，解析器统一将
  `parentId` 设为 `sessionId`、固定 `depth: 1`。若 CodeAgent 子代理内部再发起嵌套子代理，它们在
  UI 中以同级只读卡片（`depth: 1`）展平呈现，不递归构造多层树。
- **跨层边界一致性要求**：
  1. **增量省略可选字段不抹除已知归因**：若同一 `taskId` 的早期帧带有 `toolCallId` 或 `agentType`，
     后续帧仅携带必填字段时，不得因整体替换 `_meta["_subagent/update"]` 而丢掉已知委托链接或卡片标题。
  2. **实时与重载的 `plan` 去重键一致**：聊天 store 的 `replacePlan` 与 `HistoryAssembler` 必须统一
     按有效子来源 ID 替换同一子任务的最新计划，使实时面板与重载回放保持相同数量并隔离兄弟任务计划。
  3. **非交互工作流输出排除子代理文本**：`AssistantOutputAccumulator` 与 `completion_snapshot` 必须
     使用同一来源规则跳过带有效子代理归因的 `SessionUpdate`，防止子代理输出混入工作流节点的最终输出
     或破坏结构化输出校验。

## D4：外层 `toolCallId` 关联父委托与「委托已返回结果」判定边界

`_subagent/update` 子组的 `parentId` 指向父流，`toolCallId` 明确指向同一父回合、同一父来源流中的
父委托工具调用。真实 CodeAgentCLI ACP 帧已实测确认以下父委托工具生命周期形态：

| 场景 | 父委托 `tool_call` / `tool_call_update` 真实帧形态 | 回合结果与标注 |
| --- | --- | --- |
| 实时前台委托完成 | 先发 `tool_call(status: "pending", title: "Task", rawInput: {})`，子代理结束后在 `PromptResponse` 前显式发 `tool_call_update(status: "completed", title: <摘要>, content: [text], rawOutput: 缺失)` | 标注「委托已返回结果」 |
| `session/load` 回放或结构化结果 | `tool_call(status: "completed", content: 缺失, rawOutput: { status: "completed", agentId, agentType, content, ... })` | 标注「委托已返回结果」 |
| 中途取消 | 发 `tool_call(status: "pending")` 后收到 `session/cancel`，直接返回 `PromptResponse(stopReason: "cancelled")`，无 `completed` 工具更新 | 标注「后续未知（投影已停止）」 |
| 回合收束兜底或中间态输出 | 未发 provider 完成更新，仅在 `end_turn` 由运行时/store 收束为 `completed`（带 `ora/turn-end-settled` / `settledByTurnEnd`） | 标注「后续未知（投影已停止）」 |

据此锁定关联与「委托已返回结果」判定规则：

1. **只按显式 `toolCallId` 关联**：实测表明完成帧的 `title` 会从 `"Task"` 改写为摘要文本，因此关联
   严格限定在同一父回合、父来源流，按持久化顺序取与 `delegationToolCallId` 相等的最晚一次工具调用，
   禁止按 `title` 匹配，也不接受无关工具偶然返回的 OpenCode `rawOutput.metadata.sessionID`。
   父回合仍在运行或没有持久化终态时不标「投影已停止」。
2. **区分 provider 完成与回合结束补态**：`ora-history` 仅在 `settle_at_turn_end` 兜底将未决父工具
   收束为 `completed` 时写入内部标记 `"ora/turn-end-settled": true`（并在接收不可信入站帧与更新状态时
   剥离同名伪造键），前端 `chat` store 同步在 `settleActiveToolCalls` 写入 `settledByTurnEnd: true`。
3. **「委托已返回结果」充要条件**：当且仅当上述最晚一次匹配委托同时满足以下全部条件时标为「委托已返回结果」，
   其余一律标为「后续未知（投影已停止）」：
   - `status === "completed"` 且未由回合结束兜底补成（`!settledByTurnEnd`）；
   - `content.length > 0 || rawOutput !== undefined`；
   - 若 `rawOutput` 为对象且携带 `rawOutput.metadata.status` 或顶层 `rawOutput.status`（覆盖 `session/load`
     回放等结构化输出），该字段值必须等于 `"completed"`。

## D5：启用开关与兼容性边界

- `_subagent/update` 通道由 CodeAgent 运行环境开关（`CODEAGENT3_ACP_SUBAGENT_CHANNEL=true`）控制；
  未开启时无该扩展帧到达，运行时完全按既有普通 `session/update` 行为工作。
- 沿用 JSONL schema v2 与现有 `PromptSessionEvent` / `LoadSessionEvent` 契约，不迁移历史记录；旧日志
  无 `_meta["_subagent/update"]` 时按普通父流展示。

## 不变量

1. **线序不变**：同一代理进程连接上，`session/update` 与 `_subagent/update` 从读帧到 `routes.route_event`
   保持严格线序，不因扩展翻译通道拆分而发生乱序。
2. **故障与状态边界隔离**：格式非法或包含未授权 `sessionUpdate` 变体的 `_subagent/update` 帧只记日志
   并丢弃，不中断共享连接，也不得修改父会话标题、模式或权限状态。
3. **单一来源权威**：`ora-agent-runtime`、`ora-history`、`@ora/chat` 诊断、UI 子代理分组与工作流输出
   累加消费同一来源判定；显式清除或冲突来源一律归属父流，不按工具标题或前缀猜测。
4. **ID 与计划隔离**：子代理内层 `toolCallId` 在接入前完成任务级作用域隔离，不与父工具或兄弟子代理碰撞；
   `plan` 替换在实时与重载下按同一子来源 ID 生效。
5. **父计时不被子活动保活**：有效 `_subagent/update` 内容、工具与计划不重置或暂停父 prompt 的无活动
   计时；只有父委托调用自身状态参与父计时暂停。
6. **不伪造终态**：父回合结束不为有效 `_subagent/update` 未决子工具补成 `completed`；不将回合收束补态
   或非 `"completed"` 委托状态伪装为 provider 上报的委托完成。

## 为什么不是这些替代方案

| 方案 | 未采用原因 |
| --- | --- |
| 不启用 `CODEAGENT3_ACP_SUBAGENT_CHANNEL`，维持混排 | 子代理事件混入 `session/update` 导致 UI 无法折叠只读卡片，且子工具的 `in_progress` 会错误暂停父 prompt 的无活动计时，违背 `20261020` D4 计时隔离约束 |
| 为 `_subagent/update` 新增独立 JSONL 记录类型与前后端事件契约 | 破坏 `20261020` 确立的「沿用 schema v2 与 `SessionUpdate` metadata 携带来源」架构，需要双写与版本迁移逻辑，改动面与兼容性维护成本远高于连接边界规范化 |
| 仅在前端按 `Agent`/`Task` 工具名称与时间窗口推断子代理分组 | 无法区分子代理局部 `call_0`/`call_1` 与父工具碰撞，实测完成帧也会改写 `title`；重载后无法稳定还原归属，且无法在 Rust 运行时隔离 `PromptLiveness` 与历史落盘，直接违反 `20261020` D1 |
| 按父 `Task` 是否存在输出直接判定委托返回，不区分回合收束补态 | 回合结束收束会把停在 `pending`/`in_progress` 的父工具补成 `completed` 并保留中间内容，不带 `ora/turn-end-settled` 隔离会导致中止/挂起委托在实时与重载中误报「委托已返回结果」 |

## 风险、本决策未解决的问题与落地顺序

- **实测暴露的父工具 `pending` 与无活动超时风险（不改变本 ADR 边界，需外部协作或后续决策）**：
  - 真实帧显示 `codeagentcli 1.2615.02.05` 的父 `Task` 调用仅发送 `status: "pending"`，在子代理执行的
    62.66 秒内从未发送 `status: "in_progress"`，直到子代理完成才直接跳到 `status: "completed"`。
  - 按 `20261020` D4 与[静默策略](../../desktop/core/workflow/20261008-prompt-inactivity-policy.md)，
    子代理帧不刷新父计时，`PromptLiveness` 只对 `in_progress` 暂停计时而对 `pending` 仅重置一次；
    因此在普通聊天（默认 45 秒首窗超时）下，超过 45 秒的子代理委托会触发超时重试/取消（工作流节点可
    显式配置 `promptInactivity: "wait"` 规避）。此问题需要由 CodeAgentCLI 或其插件在子代理运行时
    将父委托切为 `in_progress`，或由后续专门决策评估按外层 `toolCallId` 关联的 `pending` 暂停策略，
    本 ADR 不私自放宽 `PromptLiveness`。
- **其余实测兼容观察**：实测确认 `codeagentcli 1.2615.02.05` 在 `PromptResponse` 顶层返回 `usage`
  （同时流中发 `usage_update`）、`loadSession` 为 `true` 且 `session/load` 回放完整（但父委托使用
  `rawOutput.status` 与缺失 `content`，已纳入 D4 规则）。
- **本决策未解决的问题（留待后续 ADR）**：
  - `_subagent/update` 的真实多层嵌套链路（`parentTaskId` 或子任务树层级）；
  - 子代理独立终态事件通道或后台子代理继续运行后的脱离主回合订阅；
  - 对子代理的定向控制（`steer` / 单独取消 / 关闭）；
  - 暖加载跟随实时更新时对 `HistoryBuilder` 入站伪造 `ora/turn-end-settled` 的隔离，以及交互式完成时
    子工具切分父流文本与实时聚合的差异对齐。
- **落地顺序**：
  1. **阶段 1（ADR，`approved`）**：已补齐真实帧序列锁定 D2/D4 规则，并在
     [子代理投影、终态缺口与运行诊断](../../../test-cases/node/agent-runtime/child-projection-and-diagnostics.md)
     登记对应核心用例与验证义务。
  2. **阶段 2（实现）**：在 `ora-space/desktop` 合入 `ora-acp`、`ora-agent-runtime`、`ora-history`、
     `ora-backend`、`@ora/chat`、`@ora/app-shell` 实现与对应单测/回归测试（含真实帧序列用例与 FIFO
     缓存淘汰测试）。
  3. **阶段 3（落地）**：产品代码合入 `desktop` `main` 后，将本 ADR 改为 `implemented` 并按最终实现同步
     正文与核心用例证据。

从实现合入起，开启 `CODEAGENT3_ACP_SUBAGENT_CHANNEL=true` 的 CodeAgent 会话获得与 OpenCode 一致的只读
子代理卡片、历史持久化与父 prompt 计时隔离；既有 OpenCode 投影、未启用该开关的普通 `session/update`
流、JSONL schema v2 及 `Timeout`/`Wait` 语义保持不变。
