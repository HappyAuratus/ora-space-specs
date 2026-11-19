# 子代理投影、终态缺口与运行诊断

本文件守住子代理来源保真、父回合结束后的终态缺口与委托结果关联、可见子活动与父 prompt 计时的
隔离，以及诊断时间来源。决策依据：[保留子代理投影来源](../../../decisions/node/agent-runtime/20261020-child-projection-history-and-diagnostics.md)
（`implemented`）与[扩展共享 ACP 运行时识别 CodeAgent _subagent/update 通道](../../../decisions/node/agent-runtime/20261118-codeagent-subagent-update-channel.md)
（`approved`）。证据路径以产品仓库名开头：`desktop/<文件>::<测试函数>` 是 desktop 的 Rust 测试，
前端测试写文件路径与用例标题。[静默策略用例](../../desktop/core/workflow/prompt-inactivity.md)
与[会话执行用例](session-execution.md)的证据保持各自范围，不推导来源保真或终态缺口已实现。

## Explicit Child Sources Must Reach History Without Merging Across Child Sessions

- 风险：父内容与子内容互相合并；按 depth、工具 ID 前缀或标题猜归属；并发子会话串组；循环或缺失
  父节点导致错误挂载。
- 前置状态：真实 OpenCode v2 轨迹，含并发与二层嵌套子会话，`_meta["opencode/child-session"]` 给出
  id、parentID 与 depth；另有缺 metadata、自环、缺父节点与未知扩展的内容。
- 触发：实时接收；运行中重开；暖/冷加载；同一父回合内多个子会话交错输出。
- 必须成立：只有含非空 id、parentID 和正整数 depth 的来源进入子组，父子关系只由 id/parentID 决定；
  同一父回合按 child id 分组、嵌套按 parentID 连接；无来源的内容留在父流；未知扩展不透明保留；
  缺父、自环或冲突关系降级为可阅读的独立组并标明关系未知；history 组装出的分组与实时一致。
- 禁止结果：按 depth 或工具 ID 前缀猜父、跨子会话合并、把旧历史猜成子内容、循环递归挂载、丢弃未知
  扩展、分组产生新会话或授予控制能力。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 显式来源进入 history 且分组稳定 | `desktop/crates/history/src/projection_tests.rs` 的 `projected_text_keeps_sources_and_annotations_with_same_or_missing_message_ids`、`unidentified_source_switches_preserve_order_through_disk_reload`（生产 assembler 与 `HistoryWriter`/`read_session_history` 的 JSONL 往返）；`desktop/packages/chat/tests/projection.test.ts` 的 `live batching and reload keep colliding child text ids and annotations distinct`（实时与冷加载逐项相等）；`desktop/packages/app-shell/src/features/chat/subagent-replay.test.tsx` 的 `renders nested read-only details from the production store and keeps cold replay attribution`。缺口：没有真实 ACP transport 帧经生产 runtime 写入 JSONL 的端到端用例，也没有“运行中重开”的独立用例；聊天 store 的 plan 去重按完整 metadata 键、history 按有效 child id 保留最新一份，同一子组 metadata 变化时实时可能多一份 plan、重载只剩一份（无对应 TS 测试） | Partial |
| 无来源与异常关系降级 | `projection_tests.rs` 的 `cleared_or_invalid_child_sources_settle_as_parent_tools`（null、空对象、空白 id/parentID、非法 depth，含 JSONL 回读）、`integral_float_depth_keeps_the_same_child_source_as_live_projection`；`projection.test.ts` 的 `cleared or invalid child sources settle as parent tools in live and reloaded turns`；`subagent-items.test.ts` 的 `keeps cycles, missing parents and invalid sources readable`（互环、缺父、非法来源）；`subagent-replay.test.tsx` 的 `does not invent a stopped projection from old EOF-only history`。缺口：没有字面自环用例，未知扩展在渲染层的不透明保留没有单独断言 | Partial |

- 决策依据：D1，不变量 1。

## Text And Thinking Chunks Keep Complete Metadata When Appended

- 风险：丢掉内容层 metadata 或 annotations；相同或缺失 messageId 时跨来源拼接；不同子会话的 plan
  互相覆盖；工具增量的顶层替换复活已清除的值。
- 前置状态：同一父回合内父流、多个子会话与嵌套子会话交错；包含相同/缺失 messageId、完整与部分
  metadata、显式 null、字段整体替换和晚到结果。
- 触发：实时追加文本、思考、plan 与工具增量；持久化后重载。
- 必须成立：文本与思考保存完整 `ContentChunk`，包括外层 metadata、内容层 metadata 与 annotations；
  只有 metadata、注解与消息类型一致时才允许追加文本；工具增量按顶层命名空间替换同名值，省略的键
  保留，显式 null 覆盖旧值，不递归 patch 未定义的嵌套对象；不同子会话的 plan 不互相覆盖；晚到的
  来源或结果更新同一工具快照。
- 禁止结果：以压缩展示为由混合来源或丢内容；用 messageId 跨来源拼接；plan 覆盖父或兄弟 plan；晚到
  结果生成第二个工具快照；因为重载而丢失已保存的 metadata。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 完整块与 metadata 保真 | `projection_tests.rs` 的 `projected_text_keeps_sources_and_annotations_with_same_or_missing_message_ids`（外层与内容层 metadata、annotations，经 serde 往返）、`non_text_chunks_and_thoughts_keep_their_full_metadata`（图像内容块与 thought 外层 metadata）；`desktop/packages/chat/tests/projection.test.ts` 的 `live batching and reload keep colliding child text ids and annotations distinct`（实时与冷加载的 metadata、annotations 相等）。缺口：文本与思考完整 chunk 经 JSONL 文件的逐项往返只有 serde 往返和工具快照的文件往返，没有单独的文本文件往返断言；thought 内容层 metadata 与聊天层非文本块没有直接断言 | Partial |
| 追加与替换语义 | `projection_tests.rs` 的 `late_tool_metadata_replaces_namespaces_and_survives_terminal_refinement`（顶层命名空间替换、显式 null、同一快照）、`delegation_results_survive_a_late_snapshot_and_reload`、`plans_from_distinct_sources_do_not_overwrite_each_other`、`plans_replace_by_source_despite_changing_metadata_through_reload`；`projection.test.ts` 的 `late attribution, opaque replacement and unknown child outcomes survive finalization`、`a source switch does not discard leading whitespace or merge unidentified runs`、`cleared or invalid child sources settle as parent tools in live and reloaded turns` | Covered |

- 决策依据：D2，不变量 2。

## An Ended Parent Turn Must Not Complete A Child That Never Reported A Terminal State

- 风险：父回合结束把未上报终态的子组收束为完成或取消，把未知结局伪装成已知终态；忽略已返回的
  委托结果，把常见的前台委托说成子代理还没结束；反过来把委托返回当成子组终态；实时与重载结论
  不一致；旧的未知子组在新 prompt 后复活为进行中。
- 前置状态：每种委托结果各准备一个已收到内容、含未完成子工具且没有子组终态事件的子组；分别准备
  委托工具 ACP `completed` 且 `rawOutput.metadata.sessionID` 等于子组 id、provider
  `status=completed` 的前台返回，以及 ACP `completed` 但 provider `status=running` 的后台
  返回；另准备委托工具未完成、失败、取消和没有可关联 `sessionID` 的情况。分别让父回合完成、
  失败与取消；另有一个回合中途崩溃、没有持久化回合
  终态的历史文件。
- 触发：父回合结束；随后暖/冷重载该会话；同一会话再发起新 prompt。
- 必须成立：只有最晚一次可关联委托的 ACP 工具状态与 `rawOutput.metadata.status` 均为
  `completed` 的子组显式标为「委托已返回结果」；`status=running`、委托未完成、失败、取消或
  无可关联 `sessionID` 的子组标为「后续未知（投影已停止）」。两者都不显示为进行中、完成、失败或
  已取消；标注只由已持久化的父回合终态、委托结果与缺少子组终态推出，重载后相同；新 prompt
  不把旧子组改判为进行中；该标注不释放资源，也不作为取消、steer 或 close 的入口；没有持久化回合
  终态的崩溃历史不标「投影已停止」，也不补成完成、失败或取消。
- 禁止结果：把父回合结束当作子组 completed 或 cancelled；把委托返回当作子组 completed 或
  cancelled；忽略已返回的委托结果，把前台委托标成未知；实时显示进行中而重载显示完成（或相反）；
  按时长自行补终态；把未知标注当作健康判断；重载后把旧子组挂到新回合；从错误文本或 tool ID
  前缀补出关联。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 委托已返回与未知的标注选择 | `subagent-replay.test.tsx` 的 `keeps missing child terminal state explicit after parent completed in live, warm and cold views`（`failed`、`cancelled` 同组）、`settles a child only from the delegation that names it in live, warm and cold views`、`keeps a background delegation's child on the projection-stopped wording`；`subagent-items.test.ts` 的 `links a child only through the provider's session id`、`requires the provider to report completion, not a settled or failed call`；`desktop/packages/chat/tests/delegation.test.ts` 的 `requires both the ACP completion and the provider's completed result`；`projection_tests.rs` 的 `delegation_results_survive_a_late_snapshot_and_reload`、`parent_end_does_not_complete_unsettled_projected_tools`。缺口：失败、取消、未完成与无可关联 `sessionID` 的委托在真实界面 × 三种父结束下没有逐一断言（由 store 判定单测覆盖）；provider 的 `completed` 结果随不带工具状态的晚到增量到达、回合结束再结算 ACP `completed` 的组合没有直接测试 | Partial |
| 标注不越界 | `subagent-replay.test.tsx` 断言未上报终态的子工具显示 `Outcome unknown` 且不显示 `Completed`，并确认界面没有 steer、取消或关闭子代理的按钮。缺口：两种标注不产生 completed/failed/cancelled 的显式状态断言，资源配置断言没有单独测试 | Partial |
| 持久化来源与回合隔离 | `subagent-replay.test.tsx` 的 `keeps a settled child projection unknown after the same session starts a new prompt`、`keeps an earlier turn's returned delegation after the same child is resumed`、`does not invent a stopped projection from old EOF-only history`（有/无 attribution）；`projection_tests.rs` 的 JSONL 往返用例。缺口：没有把真实 JSONL/Thread 文件重载到聊天 store 的端到端用例，子组数量的显式断言也只在渲染层间接覆盖 | Partial |

- 决策依据：D2、D3，不变量 3、7。

## A Delegation Result Must Be The Only Link Between A Child Group And Its Launcher

- 风险：按工具标题、tool ID 前缀、`<subagent sessionID=…>` 文本或时间先后把委托调用与子组配对；
  并发委托交叉配对；续接同一子会话的多次委托生成重复子组；缺少 metadata 时沿用相邻关系。
- 前置状态：同一父回合内两个并发委托、一个二层嵌套委托、同一子会话被再次委托（`sessionID`
  续接），以及一个没有 `rawOutput.metadata.sessionID` 的子组；前台与后台委托各一组。另准备跨回合
  续接：回合 1 前台委托子会话 X，ACP 和 provider 状态均为 `completed`；回合 2 用相同
  `sessionID` 在后台续接 X，ACP 为 `completed`、provider `status=running`。两个回合各有 X 子组。
- 触发：委托工具返回；父回合结束；暖/冷重载。
- 必须成立：只有**同一父回合**、子组 `parentID` 所在来源流里的委托调用，其
  `rawOutput.metadata.sessionID` 等于子组来源 id 时才建立关联；并发与嵌套在各自来源流内配对，
  不交叉；同一子组对应多次委托时以工具调用首次出现的持久化位置选最晚一次，再读取其最终快照，
  不按结果到达时间或时间戳排序。回合 1 的 X 组在实时及暖/冷重载后仍标「委托已返回结果」，
  回合 2 的 X 组标「后续未知（投影已停止）」；仅 ACP 与
  provider 状态均为 `completed` 才标「委托已返回结果」，`status=running` 即使 ACP 工具已
  `completed` 仍标「后续未知（投影已停止）」；这两种标注都不作为子组终态；缺少或无效 metadata
  时不建立关联，也不按文本或相邻顺序补造关联。
- 禁止结果：按标题、ID 前缀、结果文本或时间戳配对；并发、嵌套或跨回合交叉配对；同一父回合、
  同一子会话的多次委托生成重复子组；把 `status=running` 标成「委托已返回结果」或子代理已结束。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 关联只来自同回合、同来源流中的 provider 字段 | `desktop/packages/chat/tests/delegation.test.ts` 的 `reads only the provider's session id from the delegation result`（空白、非字符串、缺失字段与近义键都不关联）、`requires both the ACP completion and the provider's completed result`；`subagent-items.test.ts` 的 `links a child only through the provider's session id`、`takes the latest call by persisted position, not the newest timestamp`、`keeps a top-level child scoped to the parent stream`；`subagent-replay.test.tsx` 的 `settles a child only from the delegation that names it in live, warm and cold views`（嵌套）、`keeps an earlier turn's returned delegation after the same child is resumed`。缺口：两个并发的顶层委托没有交叉配对的直接断言；晚到结果不改变调用顺序只有位置单测；缺 metadata 的 store/UI 集成用例没有 | Partial |
| 不产生生命周期结论 | `delegation.test.ts` 与 `subagent-replay.test.tsx` 只断言「委托已返回结果 / 后续未知（投影已停止）」两种文案，在实时、暖、冷加载下不变；没有据此生成完成、失败或取消的断言。缺口：没有显式断言两种标注不产生终态，也没有资源相关的断言 | Partial |

- 决策依据：D3，不变量 1、7。

## Projected Child Activity Must Not Rearm The Parent Prompt

- 风险：子文本、思考、工具增量或子权限请求往返回流后重置或暂停父 Timeout；省略 metadata 的增量被
  重新算成父工具后暂停计时；后台子输出掩盖父 prompt 静默。
- 前置状态：父 prompt 使用默认策略或 `wait`，带明确子来源的更新持续到达；包含后续省略 metadata 的
  同一工具增量、子权限请求往返和一次重试。
- 触发：在静默窗口内持续收到子活动；子权限请求往返；provider 失败后重试。
- 必须成立：带子来源的内容照常投递但不重置或暂停父计时；省略 metadata 的增量继承本次尝试已识别的
  来源；子权限请求往返不独立重置父计时；父委托工具自身的 `in_progress` 仍按既有规则暂停计时；
  重试清除旧来源观察；Wait、手动停止、真实错误与取消收敛不变。
- 禁止结果：把子输出或后台日志当作父活动续期；按 tool ID 文本保活；省略 metadata 的增量改算父工具；
  新增父子等待协议或改变权限处理策略。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 子来源更新不续期父计时 | `desktop/crates/agent-runtime/src/prompt_policy_tests.rs` 的 `projected_child_activity_does_not_keep_an_unrelated_parent_prompt_alive`、`projected_child_permission_does_not_rearm_the_parent_prompt`、`parent_permission_rearms_the_prompt_window`（正对照）；`desktop/crates/agent-runtime/src/prompt_liveness.rs` 的 `child_permission_does_not_renew_the_parent_deadline`；`desktop/packages/chat/tests/diagnostics.test.ts` 的 `projected text and metadata-free child tool deltas never advance the parent observation`。缺口：runtime 侧“重试清除旧投影来源观察”没有直接测试 | Partial |
| 来源继承与父工具边界 | `prompt_policy_tests.rs` 的 `projected_child_activity_does_not_keep_an_unrelated_parent_prompt_alive`（省略 metadata 的子工具增量仍不续期）、`running_delegate_pauses_default_policy_until_it_finishes`（父委托 `in_progress` 暂停计时）、`cleared_child_source_keeps_the_delegate_pause_and_settles_on_completion`（显式 null 之后父委托仍暂停、完成照常结算）；`prompt_liveness.rs` 的 `cleared_child_source_keeps_a_running_delegate_paused`、`invalid_child_sources_do_not_end_a_running_pause`（空对象、空白 id/parentID、非法 depth 都回到父工具）、`cleared_child_source_permission_rearms_the_parent_deadline`（清除后按父工具 rearm）；`projection.test.ts` 的 `late attribution, opaque replacement and unknown child outcomes survive finalization`；[desktop/packages/chat/tests/diagnostics.test.ts](https://github.com/ora-space/desktop/blob/d271fb3b94234ab0f5d7f5ae3dcf94c738f104af/packages/chat/tests/diagnostics.test.ts) 的 `diagnostic activity and provider reports use the canonical effective source rule`（有效/无效来源与 retry/compaction）、`explicit null and invalid sources clear child inheritance in a live attempt`（生产 store 中清除后，同 tool id 的省略命名空间增量继续属于父流）；[desktop/packages/app-shell/src/features/chat/run-diagnostics.test.tsx](https://github.com/ora-space/desktop/blob/d271fb3b94234ab0f5d7f5ae3dcf94c738f104af/packages/app-shell/src/features/chat/run-diagnostics.test.tsx) 的 `shows cleared or invalid sources as parent activity and preserves retry: %j`（生产 store → 面板的父/子时间、父工具分组与 retry）。诊断证据固定于 desktop#723 合并提交 `d271fb3b94234ab0f5d7f5ae3dcf94c738f104af`；第一项诊断测试证明与规范解析器一致，独立规则边界由 `projection.test.ts` 的无效来源用例证明 | Covered |

既有 `ora_agent_runtime::connection::prompt_policy_tests::child_updates_do_not_rearm_the_parent_prompt`
只证明不同 ACP session 的更新不会唤醒父 prompt；上表的直接证据覆盖同一会话内带子来源 metadata
的更新，该测试不作为本用例证据。

- 决策依据：D4，不变量 4；继承[静默策略决策](../../../decisions/desktop/core/workflow/20261008-prompt-inactivity-policy.md)，
  不改变其窗口与重试规则。

## Diagnostics Must Separate Observation Time From Recorded History Time

- 风险：把重放的旧事件标成刚发生；把静默当成健康或卡死；切换会话、重载或卸载后留下虚假的活动与
  待处理请求；把任意 metadata.status 当成通用状态。
- 前置状态：同一会话的实时流、运行中重开、暖/冷加载、会话切换、换 provider 与新 prompt；包含未上报
  状态的 provider、缺历史的旧记录和未决权限请求。
- 触发：面板刷新、重新加载、切换会话、定时刷新、权限请求到达与答复、重试。
- 必须成立：展示 provider 实际上报且已识别的状态，缺状态显示未上报/未知；活动时间标明是客户端接收
  时间还是历史记录时间；无法区分重放与实时的加载流不刷新最近实时活动；静默只表示多久未见活动，
  不推导健康、卡死或超时；等待用户只在确有未决请求时显示；question 缺能力时不制造等待状态；切换、
  重载、换 provider 与新 prompt 清除失效观察；定时刷新不生成新活动。
- 禁止结果：用当前时间补最近活动；由静默推出健康或卡死；把未知当作已上报状态；把任意
  metadata.status 当通用状态协议；卸载或切换后仍显示旧的待处理请求。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 观察时间与记录时间可区分 | `diagnostics.test.ts` 的 `live observation clears on reload and remains isolated from another session`、`a load stream that replays an open turn exposes no live observation`；`desktop/packages/app-shell/src/features/chat/run-diagnostics.test.tsx` 的 `does not invent provider health or turn replay into live activity`、`says no parent activity was observed yet for a fresh attempt with a live snapshot`、`keeps the unknown wording while a load replays an open turn`、`reports elapsed silence without creating activity and releases its timer`。缺口：最终实现不展示历史记录时间，重放只显示“未知；重放不是新活动”，与批准文本“明确标注为记录时间”不同（见 ADR 的差异说明） | Partial |
| 状态、静默与待处理请求不越界 | `diagnostics.test.ts` 的 `only recognized provider reports establish state and null clears retries`、`failure and cancellation clear transient reports while a new prompt clears prior observations`；`run-diagnostics.test.tsx` 的 `labels parent, child and unobserved tools separately`、`reports elapsed silence without creating activity and releases its timer` 与自动处理权限的文案。缺口：question 缺能力时不制造等待状态没有代码或测试断言（只有文档说明）；未决权限的“等待”正例与权限往返不进入诊断没有直接断言 | Partial |

- 决策依据：D3，不变量 5。

## The Read-Only Detail Tree Must Degrade Without Losing Content And Stay Read-Only

- 风险：缺父、循环、未知 metadata 或历史缺口导致崩溃或丢内容；只读详情长出取消、steer 或 close
  等控制入口；虚拟滚动与流式更新丢内容。
- 前置状态：包含并发嵌套、缺失来源、未知扩展和旧历史（无来源、缺工具结局）的会话，以及同时进行的
  实时会话。
- 触发：展开/折叠子组与工具详情；流式更新；虚拟滚动；重载；在新 prompt 之后查看旧回合。
- 必须成立：可展开查看文本、思考、工具输入/输出与结果，父流保留父内容；缺状态显示未知、缺历史显示
  缺口；循环、缺父与未知 metadata 不崩溃、不丢内容；只有既有停止入口，没有单独取消、steer 或 close；
  详情分组不创建新会话。
- 禁止结果：丢内容或崩溃；把未知渲染成成功；新增子代理控制入口或改变既有停止语义；把详情树当作
  生命周期结论。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 真实界面与重载一致 | `subagent-replay.test.tsx` 以生产 store 渲染生产组件：`renders nested read-only details from the production store and keeps cold replay attribution`（实时与冷加载摘要相等、展开两层、父内容不混入子组）、`keeps missing child terminal state explicit after parent completed in live, warm and cold views`、`settles a child only from the delegation that names it in live, warm and cold views`。缺口：这是 jsdom 组件级测试，没有真实 Desktop/浏览器或 E2E；思考、工具输入/输出在子卡片中的可见性没有单独断言；虚拟滚动没有覆盖 | Partial |
| 降级与只读边界 | `subagent-items.test.ts` 的 `keeps cycles, missing parents and invalid sources readable`、`connects a late parent and keeps unrelated root content`；`subagent-replay.test.tsx` 断言界面没有 steer、取消或关闭子代理的按钮。缺口：循环与缺父在渲染层“不崩溃、不丢内容”没有直接断言，只读边界按按钮名称正则间接断言 | Partial |

- 决策依据：D1、D3，不变量 1、3、6。

## CodeAgent Subagent Extension Notifications Must Preserve Wire Order, Scope Inner Tool IDs, And Route To Parent Sessions

- 风险：`_subagent/update` 与 `session/update` 走不同通道导致实时与历史顺序错乱；`timestamp` 进入
  `_meta` 阻断 chunk 合并；`call_0`/`call_1` 等内层工具 ID 与父流或兄弟任务碰撞；`taskId` 含 `:` 时
  前缀碰撞；后续省略或置 `null` 的 `toolCallId`/`agentType` 抹除归因；伪造或冲突命名空间越权或改写
  父会话状态；未知扩展或扩展请求断开共享连接；找不到路由的帧污染其他会话或断开连接。
- 前置状态：运行中的 `AcpPeer` 与 `run_process_generation` 连接路由，存在多个活跃会话，CodeAgentCLI
  在启用 `CODEAGENT3_ACP_SUBAGENT_CHANNEL=true` 后并发或顺序发出父 `session/update`、有效
  `_subagent/update`（含省略可选字段或可选字段为 `null` 的续帧、`taskId` 含 `:`、缺 `title` 工具、内层
  携带冲突 `opencode/child-session` / `ora/turn-end-settled`、同一会话的多个子任务），以及含未授权
  `sessionUpdate` 变体或缺失必填字段的非法 `_subagent/update` 帧、`sessionId` 不对应任何会话的合法帧、
  非 `_` 开头的未知通知、带 `id` 的 `_` 开头请求；超过任务缓存上限的连续多任务。帧形态取自真实采集
  （直连采集脚本，非经 Ora），Ora 内路由只能由本用例的连接层测试证明。
- 触发：经 `ora-acp` 读取扩展通知、在 `run_process_generation` 入口同步翻译、路由到目标父会话、
  写入 `HistoryAssembler` 与前端聊天 store 并重载。
- 必须成立：`_` 开头通知按入站 FIFO 保序透传；非 `_` 开头的未知通知丢弃；带 `id` 的 `_` 开头请求回
  JSON-RPC `-32601`，连接保持；`_subagent/update` 仅在合法且属于白名单变体（`agent_message_chunk`、
  `agent_thought_chunk`、`plan`、`tool_call`、`tool_call_update`）时路由至 `params.sessionId` 对应的父
  会话，白名单外的变体不改写父会话的标题、模式和用量；剥离逐帧 `timestamp` 使连续 chunk 合并；内层
  `toolCallId` 编码为单射 `subagent:{len}:{taskId}:{innerToolCallId}`；同 `(sessionId, taskId)` 缓存的
  `toolCallId`/`agentType` 在可选字段省略或为 `null` 时保留，超上限按 FIFO 淘汰最早条目，被淘汰任务的
  后续省略帧失去委托关联与标题但仍进入同一子组；剥离内层冲突子来源与内部标记；非法或会话级状态修改帧
  只记警告并丢弃，不断开连接；找不到路由的合法帧静默丢弃，不断开连接、不落盘、不污染其他会话；同一
  会话内的所有子任务都按 `depth: 1` 挂在父流下，不构造多层树。
- 禁止结果：父子通知乱序、把非法扩展通知或扩展请求当成连接致命错误断开进程、允许子代理修改父会话
  标题/模式/用量、碰撞内层工具 ID、因 `timestamp` 拆散连续文本、外层 `null` 清除已知归因、整表清空
  活跃任务缓存或退回父流、找不到路由的帧进入其他会话、为 `_subagent/update` 推断多层嵌套。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| `_` 开头扩展通知按 FIFO 保序透传与翻译路由 | 经 `AcpPeer` 与 `run_process_generation` 验证 `_subagent/update` 与 `session/update` 线序一致、精确路由到 `params.sessionId` 父会话、非法/越权变体丢弃且不断连（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 未知通知与扩展请求不破坏连接 | 验证非 `_` 开头的未知通知被丢弃、带 `id` 的 `_` 开头请求收到 `-32601` 且连接继续处理后续帧（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 子代理帧不改写父会话状态 | 验证白名单外的 `session_info_update`、`current_mode_update`、`config_option_update`、`available_commands_update`、`usage_update` 经 `_subagent/update` 到达时，父会话的标题、模式与用量快照不变（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 作用域编码、归因保留与权威性 | 验证剥离 `timestamp` 后连续 chunk 合并、长度前缀编码防止含 `:` 碰撞、外层可选字段省略或为 `null` 时保留缓存归因、FIFO 淘汰后省略帧失去关联与标题但留在同一子组、内层冲突命名空间与 `ora/turn-end-settled` 剥离（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 找不到路由的帧静默丢弃且隔离 | 验证 `sessionId` 不对应任何已路由会话的合法帧被静默丢弃，不断开连接、不落盘、不混入其他活跃会话（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 嵌套按 `depth: 1` 展平 | 验证同一父会话下多个 `taskId` 的子组都以 `parentId = sessionId`、`depth: 1` 呈现，在实时与重载下一致（待阶段 2 产品 PR 合入后登记测试引用） | Missing |

- 决策依据：`20261118` D1、D2、D3，不变量 1、2、4。

## CodeAgent Subagent Delegations Must Require Provider Completion, Renew The Parent Only While The Delegation Is Open, And Stay Out Of Workflow Output

- 风险：仅靠回合结束收束 `completed`（带中间输出）或取消/非 `completed` 状态误报「委托已返回结果」；
  `title` 在完成帧变化导致委托关联失效或错配无关 `rawOutput.metadata.sessionID`；结构化 `rawOutput`
  的状态被忽略或被错误字段覆盖；provider 的 `session/load` 回放被写进 Ora 历史；父 `Task` 只发
  `pending` 时，超过首个窗口的前台委托被取消并重复执行；反过来，子代理活动在父委托结束后仍续期父计时，
  或在任何情况下暂停父计时；子代理更新混入工作流节点输出；`plan` 按错误键覆盖或重复。
- 前置状态：真实 `_subagent/update` 轨迹 `s1_complete`（父 `Task` `pending` → 6 条子帧 → 完成帧
  `status: "completed"` 且 `content` 非空、`rawOutput` 缺失、`title` 改写；父流静默 62.66 秒，含子帧时
  相邻最大间隔 19.6 秒）与 `s2_cancel`（`pending` → `cancelled`，没有子帧）；按 `s3_load` 回放形态构造的
  结构化快照（`content` 缺失、`rawOutput.status` 为 `"completed"` 或其他值、`rawOutput` 为 `null`、
  `rawOutput.metadata.status` 与顶层 `status` 同时存在且取值不同）；`in_progress` 带部分输出经
  `end_turn` 收束并标记 `ora/turn-end-settled`；一次 provider `session/load` 回放（含子代理工具摊平帧与
  转换后的 `_subagent/update`）；以及工作流输出累加器。
- 触发：实时完成/取消、回合收束、恢复 provider 会话、暖/冷重载、工作流输出累加、诊断更新与父 prompt
  静默计时。
- 必须成立：显式 `toolCallId` 只匹配同回合父流中同一 ID 的最晚调用（不看 `title` 也不回退到 OpenCode
  `sessionID`）；`s1_complete` 在实时、暖加载与冷加载下均显示「委托已返回结果」，`s2_cancel` 与回合收束
  补态（`ora/turn-end-settled` / `settledByTurnEnd`）均显示「后续未知（投影已停止）」；存储与组装层解析
  已持久化的结构化 `rawOutput` 时，`null` 按没有输出处理，`rawOutput.metadata.status` 存在时优先于顶层
  `status`，只有取值为 `"completed"` 才显示「委托已返回结果」；`session/load` 期间 provider 的回放（含
  转换后的 `_subagent/update`）被排空，不落盘、不转发；带有效外层 `toolCallId` 的子帧在父委托于本次尝试中
  仍为 `pending`/`in_progress` 时为父窗口续期、不暂停计时，按 `s1_complete` 的真实间隔驱动时父 prompt
  不被取消；父委托已结束、外层 id 不认识或帧非法时不续期；子代理超过一个窗口没有任何帧时父 prompt 照常
  超时；重试清除旧尝试的父委托观察；诊断消费统一来源规则；`plan` 按有效子来源 ID 去重替换；工作流
  `AssistantOutputAccumulator` 与 `completion_snapshot` 跳过带有效 `_subagent/update` 归因的更新。
- 禁止结果：回合结束兜底补成 `completed` 的父委托误报已返回结果、信任入站伪造的 `ora/turn-end-settled`、
  结构化 `rawOutput.status="completed"` 漏判或被顶层字段覆盖 `metadata.status`、provider 回放进入 Ora
  历史、子代理帧暂停父计时、父委托结束后子代理帧仍续期、压缩时间间隔使续期测试失去意义、子代理文本或
  工具污染工作流输出。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 真实帧形态下委托返回与收束隔离 | 覆盖 `s1_complete` 在实时/暖/冷加载下标「委托已返回结果」，`s2_cancel` 与 `in_progress` 带部分输出经 `end_turn` 收束（带 `ora/turn-end-settled`）标「后续未知（投影已停止）」，以及伪造标记剥离（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 结构化 `rawOutput` 在存储与组装层的解析 | 覆盖已持久化快照中 `content` 缺失、`rawOutput.status` 为 `"completed"` 与其他值、`rawOutput` 为 `null`、`metadata.status` 与顶层 `status` 并存时以 `metadata.status` 为准（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| `session/load` 回放不进入 Ora 历史 | 覆盖恢复 provider 会话时回放的子代理工具摊平帧与转换后的 `_subagent/update` 被排空，不写 JSONL、不转发给前端（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 父委托未结束时续期、不暂停 | 覆盖父委托 `pending`/`in_progress` 时有效子帧续期；父委托已完成、外层 id 缺失或不认识、帧非法时不续期；子帧不计入运行中工具、不暂停；子代理一个窗口内没有帧时照常超时；重试清除旧观察；其中一条按 `s1_complete` 的真实时间间隔（不压缩）驱动并断言父 prompt 不被取消（待阶段 2 产品 PR 合入后登记测试引用） | Missing |
| 统一来源在诊断、`plan` 与工作流输出生效 | 覆盖 `PromptLiveness` 隔离（含碰撞 ID）、`observeDiagnostics` 继承、`plan` 按子来源 ID 替换，以及 `AssistantOutputAccumulator` / `completion_snapshot` 排除子代理输出（待阶段 2 产品 PR 合入后登记测试引用） | Missing |

已知缺口：CodeAgent 子代理的 `session/request_permission` 不带子来源，目前按父工具续期，与本文件
「Projected Child Activity Must Not Rearm The Parent Prompt」的子权限要求不一致，留待 `20261118`
「本决策未解决的问题」决定后再登记义务；实时完成帧不带 provider 状态，后台委托回执会被标为「委托已返回
结果」，属于 `20261118` D4 的已知限制，不作为本用例的失败。

- 决策依据：`20261118` D3、D4、D6，不变量 3、4、5、6。
