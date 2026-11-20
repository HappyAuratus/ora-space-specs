---
status: implemented
---

# 扩展共享 ACP 运行时识别 CodeAgent _subagent/update 通道，复用现有子代理投影与历史模型

共享 ACP 运行时与历史装配新增支持华为 CodeAgentCLI 的 `_subagent/update` 扩展通知通道：
`ora-acp` 保持 provider 中立并按线序透传 `_` 开头的 ACP 扩展通知（`AcpInboundEvent::ExtensionNotification`）；
`ora-agent-runtime` 在连接路由入口识别 `_subagent/update`，校验允许的 `SessionUpdate` 变体、
对子代理局部 `toolCallId` 做作用域隔离并剥离逐帧 `timestamp` 后规范化为带 `_meta["_subagent/update"]`
的 `SessionUpdate`；`ora-history`、`PromptLiveness`、运行诊断、前端只读子代理树以及非交互工作流
输出累加共用同一条子代理来源规则，沿用现有 JSONL schema v2、回放/交接与公共契约，不做存储迁移。
父委托调用在本次尝试中仍未结束时，指向它的子代理帧为父 prompt 窗口续期但不暂停计时（D6）。

本文处于 `implemented`：由 [desktop#783](https://github.com/ora-space/desktop/pull/783)（head `1211708d`）在
`ora-acp`、`ora-agent-runtime`、`ora-history`、`ora-backend`、`@ora/chat` 与 `@ora/app-shell` 实现；
D2 与 D4 依据 `codeagentcli 1.2615.02.05` / ACP `1.2605.00` 启用
`CODEAGENT3_ACP_SUBAGENT_CHANNEL=true` 后的真实带时间戳帧序列锁定，对应核心验证义务与直接证据登记在
[子代理投影、终态缺口与运行诊断](../../../test-cases/node/agent-runtime/child-projection-and-diagnostics.md)，
尚未覆盖的端到端与 `session/load` 排空等缺口如实保留。
真实帧由直连 CodeAgentCLI 的采集脚本（ACP 客户端 `ora-capture`，`codeagent --acp`，stdio 管道）
采得，**没有经过 Ora 的连接层，也没有 `PromptLiveness` 计时**；采集包含完整前台委托、委托开始后
取消、以及新进程 `session/load` 回放三段，没有采到子代理授权请求和嵌套委托。影响 `ora-acp`、
`ora-agent-runtime`、`ora-history`、`@ora/chat`、`@ora/app-shell` 及工作流输出累加边界；不改动
`PromptSessionEvent` / `LoadSessionEvent` 结构，不改变既有 OpenCode `_meta["opencode/child-session"]`
行为，不迁移旧历史文件，也不引入独立 JSONL 记录类型或子代理控制入口。

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
| D2：工具增量中显式 `null` 覆盖旧值 | 限定：`_subagent/update` 外层的 `toolCallId: null` / `agentType: null` 等同缺失，不清除同一任务已知的归因（见 D2 规则 1、3）；内层 `update` 的工具字段仍按前序 D2 处理 |
| D3：子组不推断终态；父回合结束后只把明确完成的前台委托标为「委托已返回结果」，其余标为「后续未知（投影已停止）」 | 修改（扩展）：`_subagent/update` 不依赖 OpenCode 的 `rawOutput.metadata.sessionID`，改由外层 `toolCallId` 关联父流委托工具，并通过 `ora/turn-end-settled` / `settledByTurnEnd` 与 `content`/`rawOutput` 规则区分 provider 完成与回合收束补态。已知限制：实时 CodeAgent 完成帧不带 provider 状态，无法区分后台启动回执（见 D4「已知限制」） |
| D4：有效子来源活动与增量子工具不重置或暂停父 prompt 无活动计时 | 修改：带有效外层 `toolCallId` 的 `_subagent/update`，在它指向的父委托调用于本次尝试中仍为 `pending` 或 `in_progress` 时，为父 prompt 窗口续期但不暂停计时；其余情况仍不续期、不暂停（见 D6） |
| D4：子权限请求的往返不能独立重置父计时 | 继承，CodeAgent 子权限请求例外（未决）：CodeAgent 子代理授权走普通 `session/request_permission`，不带 `_meta["_subagent/update"]`，运行时无法识别其子来源，目前会按父工具续期（见「本决策未解决的问题」） |
| D4：归属、liveness 与运行诊断共用同一来源规则 | 继承，不变：`PromptLiveness`、`ora-history`、`@ora/chat` 诊断与 `SubagentGroup` 共用同一有效来源函数，显式清除、无效或冲突来源退回父流 |
| 不变量 4：子活动不会独立续期或暂停父 Timeout，父委托 `in_progress` 和 Wait 仍有效 | 修改：子活动仍不暂停父 Timeout；仅在 D6 的条件下按父委托为父窗口续期。本 ADR 不变量 5 相应改写 |

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

真实 CodeAgentCLI ACP 帧（`codeagentcli 1.2615.02.05` / ACP `1.2605.00`）确认：
`_subagent/update` 是顶层 ACP `extNotification`（非 `session/update` 包装），其 `params` 顶层
包含 `{ sessionId, taskId, toolCallId, agentType, update, timestamp }`，其中 `params.sessionId`
等于 `session/new` 返回的**父 ACP 会话 ID**（6/6 一致），`params.toolCallId` 等于父流
`Agent`/`Task` 委托工具调用的 `toolCallId`（6/6 一致），每一帧都重复携带 `toolCallId` 与 `agentType`，
没有 `null` 字段。因此 `sessionId` 同时作为 `SessionNotification::new(session_id, update)` 的连接路由键
和只读子代理分组的 `parentId`。

上述事实来自直连采集脚本，Ora 内的路由**没有被真实帧端到端观察过**；路由正确性的证据目前只有
实现中的连接层测试（用真实帧形态构造输入），登记在核心用例里。

在共享运行时的连接事件循环（`run_process_generation`）中，收到 `AcpInboundEvent::ExtensionNotification`
且 `method == "_subagent/update"` 时，必须在同一分支内同步翻译为标准 `SessionNotification` 后再送入
`routes.route_event`，确保翻译不破坏线序；其他未被运行时识别的 `_` 开头扩展通知直接忽略。

翻译边界遵守以下规则：

1. **必填与可选字段校验**：`params` 必须为 JSON 对象；`sessionId`、`taskId` 必须为 trim 后非空字符串；
   可选字段 `toolCallId` 与 `agentType` 若存在则必须为字符串或 `null`；`null` 与空白字符串都等同缺失，
   不清除已知归因；`update` 必须为包含有效 `sessionUpdate` 字段的 JSON 对象。任何校验失败或不受支持
   的帧只记录警告并丢弃，不得断开共享 Agent 进程连接。
2. **允许的 `sessionUpdate` 变体白名单**：仅接受 `agent_message_chunk`、`agent_thought_chunk`、
   `plan`、`tool_call` 与 `tool_call_update`。拒绝 `session_info_update`、`current_mode_update`、
   `config_option_update`、`available_commands_update`、`usage_update` 等会话级状态更新，防止
   子代理进度帧改写父会话标题、模式或用量快照。
3. **剥离易变字段并生成稳定归因**：在目标 `SessionUpdate` 的 `_meta["_subagent/update"]` 写入
   `{ sessionId, taskId, toolCallId?, agentType? }`，刻意不写入外层逐帧 `timestamp`。否则相邻
   文本或思考 chunk 会因 `_meta` 不相等而无法合并，破坏 `ContentChunk` 拼接与 JSONL 紧凑性。
   每个进程代连接按 `(sessionId, taskId)` 维护有界 FIFO 归因缓存（`MAX_CACHED_TASKS = 4096`），
   将先前帧提供的 `toolCallId` 与 `agentType` 补全到同一任务后续省略了可选字段（或为 `null`）的帧上；
   达到容量上限时仅淘汰最早入队的条目，不得整表清空正在运行的任务。缓存只存在于进程代连接内存中：
   条目被淘汰或 Agent 进程重启后，同一任务后续省略这两个字段的帧会失去委托关联（D4、D6 不再适用）
   和卡片标题（回退为固定标题），内容仍按 `sessionId`/`taskId` 进入同一子组。所采版本每一帧都重复
   这两个字段，因此实际不触发这种退化。对于缺少非空 `title` 的子代理 `tool_call`，可按
   `_meta.codeAgentCLI.toolName` 或固定占位标题回退。
4. **内层 `toolCallId` 作用域隔离**：子代理在独立模型轮次执行，网关可能重新从 `call_0`/`call_1`
   编号。翻译层必须按所属 `taskId` 对内层 `update.toolCallId` 做无歧义的长度前缀命名空间编码
   （`subagent:{len}:{taskId}:{innerToolCallId}`，其中 `len` 为 `taskId` 字节长度），防止与父会话
   或兄弟子代理的工具 ID 在 `PromptLiveness`、`HistoryAssembler` 与 UI 键值中碰撞，并消除 `taskId`
   自身含 `:` 时的二义性。
5. **归因命名空间权威性**：翻译层接管 `_subagent/update` 通道的来源归因，插入 `_meta["_subagent/update"]`
   前必须剥离内层载荷携带的其他已知子来源或回合收束内部标记（如 `opencode/child-session`、
   `ora/turn-end-settled`），避免与 `_subagent/update` 冲突导致统一来源规则把该工具退回父流。
6. **找不到路由的帧**：格式合法、但 `sessionId` 匹配不到任何已路由会话的帧，在 `routes.route_event`
   处与普通 `session/update` 一致地静默丢弃，不断开连接、不落盘。

## D3：单一子代理来源规则、已知展平限制与父输出隔离

Rust（`ora_history::child_source_id` / `has_child_source_namespace`）与 TypeScript
（`childSessionSource` / `hasChildSessionNamespace`）各只保留唯一一处子来源判别实现，同时识别
`opencode/child-session` 与 `_subagent/update`：

- `PromptLiveness`、`HistoryAssembler`、`@ora/chat` 运行诊断（`observeDiagnostics`）、
  `SubagentGroup` 分组以及工作流输出提取全部消费同一规则；任何消费端不得再硬编码单个 provider
  命名空间。
- 显式 `null`、缺失必填字段、类型非法或双命名空间冲突一律视作无效来源并退回父流（对应 tool ID
  从子代理归属清除并恢复父工具行为），沿用 `20261020` 的有效来源规则。这里的显式 `null` 指整个
  `_meta["_subagent/update"]` 命名空间为 `null`，与 D2 规则 1 中外层可选字段为 `null` 不同。
- **已知限制（嵌套展平）**：`_subagent/update` 当前未携带子任务之间的父子层次链，解析器统一将
  `parentId` 设为 `sessionId`、固定 `depth: 1`。若 CodeAgent 子代理内部再发起嵌套子代理，它们在
  UI 中以同级只读卡片（`depth: 1`）展平呈现，不递归构造多层树。本次采集没有嵌套样本。
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
父委托工具调用。真实帧与推导得到的父委托工具形态如下：

| 场景 | 父委托 `tool_call` / `tool_call_update` 形态 | 回合结果与标注 |
| --- | --- | --- |
| 实时前台委托完成（实测） | 先发 `tool_call(status: "pending", title: "Task", rawInput: {})`，期间不发 `in_progress`；子代理结束后在 `PromptResponse` 前显式发 `tool_call_update(status: "completed", title: <摘要>, content: [text], rawOutput: 缺失)`，不带任何 provider 状态字段 | 标注「委托已返回结果」 |
| 结构化 `rawOutput`（防御性规则） | 完成调用不带 `content`、只带对象形式的 `rawOutput`，状态写在 `rawOutput.status` 或 `rawOutput.metadata.status`。所采版本只在 provider `session/load` 回放里出现这种形态（`rawOutput: { status: "completed", agentId, agentType, content, ... }`），而 Ora 会排空该回放（见下文），所以它只作为对结构化输出的防御性规则 | `status` 为 `"completed"` 时标注「委托已返回结果」，其他值标「后续未知（投影已停止）」 |
| 中途取消（由规则 3 推导） | 发 `tool_call(status: "pending")` 后收到 `session/cancel`，直接返回 `PromptResponse(stopReason: "cancelled")`，无 `completed` 工具更新。所采样本在出现任何子帧之前就已取消，`cancelled` 之后才到达的子帧没有采到 | 标注「后续未知（投影已停止）」 |
| 回合收束兜底或中间态输出 | 未发 provider 完成更新，仅在 `end_turn` 由运行时/store 收束为 `completed`（带 `ora/turn-end-settled` / `settledByTurnEnd`） | 标注「后续未知（投影已停止）」 |

**`session/load` 回放不进入 Ora 历史**：Ora 的暖加载与冷加载读的是自己的 JSONL，恢复 provider 会话时
`session/load` 期间 provider 回放的全部帧（包括转换后的 `_subagent/update`，以及所采版本把子代理工具
摊平成普通 `session/update` 的回放）都被排空，不落盘、不转发。因此上表第 2 行的判定绑定在存储与组装层
对已持久化工具快照的解析上，而不是绑定在 provider 回放上。

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
   - `content` 非空，或 `rawOutput` 既不是 `undefined` 也不是 `null`（`null` 按没有输出处理）；
   - 若 `rawOutput` 为对象：`rawOutput.metadata` 为对象且含 `status` 键时，以 `rawOutput.metadata.status`
     为准，必须等于 `"completed"`，此时不再看顶层 `rawOutput.status`；否则若顶层含 `status` 键，
     `rawOutput.status` 必须等于 `"completed"`。

**已知限制（后台委托无法区分）**：实时 CodeAgent 完成帧不带任何 provider 状态，规则 3 在实时路径上
实际退化为「`completed`、非回合收束、`content` 非空」。如果 CodeAgent 发起后台委托，它会立刻带着一条
启动回执返回 `completed`，按此规则会被标为「委托已返回结果」，与 `20261020` 否定「把后台委托也标为
委托已返回结果」的取舍相悖。所采版本没有后台委托样本；在 CodeAgent 提供可区分后台启动的状态字段之前，
本 ADR 接受这一限制，不按文本或时长猜测。

## D5：启用开关与兼容性边界

- `_subagent/update` 通道由 CodeAgent 运行环境开关（`CODEAGENT3_ACP_SUBAGENT_CHANNEL=true`）控制；
  未开启时无该扩展帧到达，运行时完全按既有普通 `session/update` 行为工作。
- 沿用 JSONL schema v2 与现有 `PromptSessionEvent` / `LoadSessionEvent` 契约，不迁移历史记录；旧日志
  无 `_meta["_subagent/update"]` 时按普通父流展示。

## D6：父委托未结束时，指向它的子代理帧为父窗口续期但不暂停计时

所采版本的父 `Task` 从 `pending` 直接跳到 `completed`，委托期间不发 `in_progress`。完整前台委托中父流
静默了 62.66 秒，算上子帧，相邻帧的最大间隔只有 19.6 秒。若子代理帧既不续期也不暂停父计时，普通聊天
固定使用 `Timeout`、首个窗口 45 秒，会在子代理给出最终回答之前取消本次尝试；重试后模型从头再委托一次，
子代理及其工具会重复执行。所以本 ADR 修改 `20261020` D4：

1. **续期条件**：一条有效 `_subagent/update` 帧（规范化后带有效 `_meta["_subagent/update"]`），其外层
   `toolCallId`（含按 D2 规则 3 从缓存补全的值）指向的父委托调用在**本次尝试**中已被观察到、且当前状态为
   `pending` 或 `in_progress` 时，`PromptLiveness` 把父 prompt 的无活动窗口重置为一个完整窗口（续期），
   但不把它计入运行中工具，不暂停计时。
2. **不续期的情况**：父委托调用在本次尝试中已结束（`completed`、`failed`，例如后台委托已返回回执）、
   外层 `toolCallId` 缺失或不是本次尝试里认识的父工具、帧格式不合法或来源无效。这些帧照常投递，但不影响
   父计时。重试开始新尝试时清除旧尝试的父委托观察。续期依赖父工具 `toolCallId` 在同一 provider 会话内
   唯一（所采版本未见复用）；若 provider 复用父工具 ID，前一次尝试或前一回合迟到的子帧可能为当前同 ID 的
   父调用续期，届时须改为只认在该父调用 `pending` 之后首次出现的任务。
3. **卡住的子代理仍会超时**：续期只由真实到达的子帧触发；子代理 45 秒（或当次窗口）内没有任何帧时，父
   prompt 照常按静默策略超时和重试。父委托自身发出 `in_progress` 时仍按既有规则暂停计时，暂停优先于续期；
   `Wait` 策略、手动停止、真实错误与取消收敛不变。续期只看子帧是否到达，不判断子代理是否健康，因此有两个
   接受的边界：子代理内单个工具运行超过当次窗口、期间没有任何帧时（所采版本的子工具同样从 `pending`
   直接跳到 `completed`），父 prompt 仍会被取消并重试，子代理和这个工具会重跑；子代理持续发帧（例如陷入
   重试循环）时，父窗口会一直续期，只能手动停止，这与 OpenCode 父委托 `in_progress` 暂停计时时的行为相同。
   这两个边界都由下文的长期契约消除。
4. **子来源的其余活动不变**：不满足上述条件的子代理文本、思考、`plan` 和工具增量，以及其他 provider 的
   投影活动，仍不重置、不暂停父计时。

## 不变量

1. **线序不变**：同一代理进程连接上，`session/update` 与 `_subagent/update` 从读帧到 `routes.route_event`
   保持严格线序，不因扩展翻译通道拆分而发生乱序。
2. **故障与状态边界隔离**：格式非法或包含未授权 `sessionUpdate` 变体的 `_subagent/update` 帧只记日志
   并丢弃，不中断共享连接，也不得修改父会话标题、模式或权限状态。
3. **单一来源权威**：`ora-agent-runtime`、`ora-history`、`@ora/chat` 诊断、UI 子代理分组与工作流输出
   累加消费同一来源判定；显式清除或冲突来源一律归属父流，不按工具标题或前缀猜测。
4. **ID 与计划隔离**：子代理内层 `toolCallId` 在接入前完成任务级作用域隔离，不与父工具或兄弟子代理碰撞；
   `plan` 替换在实时与重载下按同一子来源 ID 生效。
5. **子活动只在父委托未结束时续期**：有效 `_subagent/update` 活动从不暂停父 prompt 计时；只有当它的外层
   `toolCallId` 指向本次尝试中仍为 `pending` 或 `in_progress` 的父委托调用时，才为父窗口续期。其余子活动
   不续期；父委托调用自身的 `in_progress` 仍按既有规则暂停计时。
6. **不伪造终态**：父回合结束不为有效 `_subagent/update` 未决子工具补成 `completed`；不将回合收束补态
   或非 `"completed"` 委托状态伪装为 provider 上报的委托完成。

## 为什么不是这些替代方案

| 方案 | 未采用原因 |
| --- | --- |
| 不启用 `CODEAGENT3_ACP_SUBAGENT_CHANNEL`，维持混排（方案 B） | 子代理工具混在父 `session/update` 里且不带子来源，`20261020` D4 不适用：子工具的 `pending`/`in_progress` 被当作父工具，会续期或暂停父计时，长委托在今天的聊天里正是靠这一点才能跑完，这是它在计时上的优势。未采用是因为 UI 无法把子代理内容折叠成只读卡片、子工具 ID 可能与父工具碰撞、历史里没有子来源可供重载还原；计时上的需要改由 D6 的有条件续期满足 |
| 父委托未结束时，子代理帧暂停父计时 | 暂停后父 prompt 不再有任何超时：子代理卡住、或 provider 漏发完成帧时，父回合会一直挂起，只能手动停止；续期保留了「45 秒无帧即超时」的兜底 |
| 为 CodeAgent 放宽首个窗口或改用 `Wait` | 改的是全局静默策略，也会掩盖父流本身的静默；窗口放到多大都只是推迟问题，子代理更长时照样被取消 |
| 只记为风险，不改计时 | 超过 45 秒的前台委托在普通聊天里必然被取消并重试，子代理及其中的命令重复执行，属于结构性问题，不能等 CodeAgent 侧补发 `in_progress` |
| 为 `_subagent/update` 新增独立 JSONL 记录类型与前后端事件契约 | 破坏 `20261020` 确立的「沿用 schema v2 与 `SessionUpdate` metadata 携带来源」架构，需要双写与版本迁移逻辑，改动面与兼容性维护成本远高于连接边界规范化 |
| 仅在前端按 `Agent`/`Task` 工具名称与时间窗口推断子代理分组 | 无法区分子代理局部 `call_0`/`call_1` 与父工具碰撞，实测完成帧也会改写 `title`；重载后无法稳定还原归属，且无法在 Rust 运行时隔离 `PromptLiveness` 与历史落盘，直接违反 `20261020` D1 |
| 按父 `Task` 是否存在输出直接判定委托返回，不区分回合收束补态 | 回合结束收束会把停在 `pending`/`in_progress` 的父工具补成 `completed` 并保留中间内容，不带 `ora/turn-end-settled` 隔离会导致中止/挂起委托在实时与重载中误报「委托已返回结果」 |

## 风险、本决策未解决的问题与落地顺序

- **长期契约：由 CodeAgent 侧标记父委托运行中**：请 CodeAgentCLI 或 `codeagent-agent` 插件在子代理运行
  期间为父 `Task` 发送 `tool_call_update{status: "in_progress"}`。补发之后既有的暂停规则优先生效，D6 的
  续期规则仍然无害，不需要回退；D6.3 所述长单工具被取消的缺口也随之消失。
- **CodeAgent 子代理授权请求（未决）**：子代理工具需要授权时走普通 `session/request_permission`，不带
  `_meta["_subagent/update"]`，用的大概率是内层未作用域化的 `toolCallId`。运行时无法识别它来自子代理，
  会按父工具在往返结束后续期，与 `20261020` D4「子权限请求的往返不能独立重置父计时」不一致；授权卡片
  也无法关联到对应的子代理卡片。本次采集没有授权样本，待取得真实帧后另行决定识别方式。
- **其余实测兼容观察**：所采版本在 `PromptResponse` 顶层返回 `usage`（同时流中发 `usage_update`）、
  `loadSession` 为 `true`；`session/load` 回放把子代理工具摊平成普通 `session/update`、父委托改用
  `rawOutput.status` 且缺少 `content`。Ora 排空该回放，不影响本 ADR 的判定。
- **本决策未解决的问题（留待后续 ADR）**：
  - `_subagent/update` 的真实多层嵌套链路（`parentTaskId` 或子任务树层级）；
  - 区分后台委托启动回执与前台委托完成的 provider 状态字段（见 D4「已知限制」）；
  - 子代理独立终态事件通道或后台子代理继续运行后的脱离主回合订阅；
  - 对子代理的定向控制（`steer` / 单独取消 / 关闭）；
  - 暖加载跟随实时更新时对 `HistoryBuilder` 入站伪造 `ora/turn-end-settled` 的隔离，以及交互式完成时
    子工具切分父流文本与实时聚合的差异对齐。
- **落地顺序**：
  1. **阶段 1（ADR，`approved`）**：[specs#309](https://github.com/ora-space/specs/pull/309)（合并提交 `9b9fc0d4`）依据真实帧锁定 D2/D4 规则、新增 D6，并在
     [子代理投影、终态缺口与运行诊断](../../../test-cases/node/agent-runtime/child-projection-and-diagnostics.md)
     登记对应核心用例与验证义务。
  2. **阶段 2（实现）**：[desktop#783](https://github.com/ora-space/desktop/pull/783)（head `1211708d`）实现 `ora-acp`、`ora-agent-runtime`、`ora-history`、
     `ora-backend`、`@ora/chat`、`@ora/app-shell` 及中英文文档，并交付真实帧形态用例、FIFO
     缓存淘汰测试与暂停时钟下按真实间隔驱动的 D6 续期及反向对照测试。
  3. **阶段 3（落地，`implemented`）**：将本 ADR 改为 `implemented`，并按 `desktop#783` 的最终实现同步核心用例直接证据与剩余缺口。

从实现合入起，开启 `CODEAGENT3_ACP_SUBAGENT_CHANNEL=true` 的 CodeAgent 会话获得与 OpenCode 一致的只读
子代理卡片与历史持久化；父 prompt 计时上，子代理活动不暂停父计时，只在父委托未结束时续期（D6），这与
OpenCode 投影「子活动不影响父计时、只靠父委托自身的 `in_progress` 暂停」不同，原因是所采 CodeAgent
版本不为父委托发送 `in_progress`。
既有 OpenCode 投影、未启用该开关的普通 `session/update` 流、JSONL schema v2 及 `Timeout`/`Wait` 语义
保持不变。
