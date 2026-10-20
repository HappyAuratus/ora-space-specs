# 子代理投影、终态缺口与运行诊断

本文件守住子代理来源保真、父回合结束后的终态缺口与委托结果关联、可见子活动与父 prompt 计时的
隔离，以及诊断时间来源。决策依据：[保留子代理投影来源](../../../decisions/node/agent-runtime/20261020-child-projection-history-and-diagnostics.md)。
实现尚未合入 desktop 的 main，下列义务目前都是 `Missing`；[静默策略用例](../../desktop/core/workflow/prompt-inactivity.md)
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
| 显式来源进入 history 且分组稳定 | 真实帧经生产 runtime 写入会话 JSONL，再由生产 history/load 组装；比较实时、运行中重开与暖/冷加载的子组归属与内容 | Missing |
| 无来源与异常关系降级 | 缺 metadata、缺父、自环、未知扩展的负例不串组、不递归、不丢内容；旧历史不被补造来源 | Missing |

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
| 完整块与 metadata 保真 | 真实帧经生产组装与 JSONL 往返后，文本/思考的外层与内容层 metadata、annotations 逐项一致 | Missing |
| 追加与替换语义 | 相同/缺失 messageId、显式 null、整体替换与晚到结果的负例按 D2 规则归属，重载后同一快照 | Missing |

- 决策依据：D2，不变量 2。

## An Ended Parent Turn Must Not Complete A Child That Never Reported A Terminal State

- 风险：父回合结束把未上报终态的子组收束为完成或取消，把未知结局伪装成已知终态；忽略已返回的
  委托结果，把常见的前台委托说成子代理还没结束；反过来把委托返回当成子组终态；实时与重载结论
  不一致；旧的未知子组在新 prompt 后复活为进行中。
- 前置状态：父回合中已收到两个子组的内容与未完成工具，两个子组都没有自己的终态事件；其中一组
  对应的委托工具已 `completed` 且 `rawOutput.metadata.sessionID` 等于该组 id（分别覆盖
  `status=completed` 与 `status=running`），另一组的委托工具失败、取消或没有可关联的
  `sessionID`；分别准备父回合完成、失败与取消三种结束；另有一个回合中途崩溃、没有持久化回合
  终态的历史文件。
- 触发：父回合结束；随后暖/冷重载该会话；同一会话再发起新 prompt。
- 必须成立：关联到已返回委托的子组显式标为「委托已返回结果」，其余子组标为「后续未知（投影已
  停止）」；两者都不显示为进行中，也不显示为完成、失败或已取消；标注只由已持久化的父回合终态、
  委托结果与缺少子组终态推出，重载后相同；`status=running` 的委托不显示为子代理已结束；新 prompt
  不把旧子组改判为进行中；该标注不释放资源，也不作为取消、steer 或 close 的入口；没有持久化回合
  终态的崩溃历史不标「投影已停止」，也不补成完成、失败或取消。
- 禁止结果：把父回合结束当作子组 completed 或 cancelled；把委托返回当作子组 completed 或
  cancelled；忽略已返回的委托结果，把前台委托标成未知；实时显示进行中而重载显示完成（或相反）；
  按时长自行补终态；把未知标注当作健康判断；重载后把旧子组挂到新回合；从错误文本或 tool ID
  前缀补出关联。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 委托已返回与未知的标注选择 | 三种父结束方式 × 已返回（`completed`/`running`）、失败、关联缺失，实时与暖/冷重载都给出同一标注 | Missing |
| 标注不越界 | 两种标注都不产生完成、失败、取消或进行中；未完成工具不显示成功；不释放资源 | Missing |
| 持久化来源与回合隔离 | 从 JSONL/Thread 重载不新增或丢失子组；旧子组不因新 prompt 复活为进行中；崩溃历史不标停止 | Missing |

- 决策依据：D2、D3，不变量 3、7。

## A Delegation Result Must Be The Only Link Between A Child Group And Its Launcher

- 风险：按工具标题、tool ID 前缀、`<subagent sessionID=…>` 文本或时间先后把委托调用与子组配对；
  并发委托交叉配对；续接同一子会话的多次委托生成重复子组；缺少 metadata 时沿用相邻关系。
- 前置状态：同一父回合内两个并发委托、一个二层嵌套委托、同一子会话被再次委托（`sessionID`
  续接），以及一个没有 `rawOutput.metadata.sessionID` 的子组；前台与后台委托各一组。
- 触发：委托工具返回；父回合结束；暖/冷重载。
- 必须成立：只有 `rawOutput.metadata.sessionID` 等于子组来源 id 时建立关联；并发与嵌套在各自
  来源流内配对，不交叉；同一子组对应多次委托时以最晚一次的结果判定；`status` 只说明委托是否已
  返回，不作为子组终态；缺少或无效 metadata 时不建立关联，也不从文本或顺序推断。
- 禁止结果：按标题、ID 前缀、结果文本或时间先后配对；并发或嵌套交叉配对；同一子会话的多次委托
  生成重复子组；把 `status=running` 标成子代理已结束。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 关联只来自 provider 字段 | 真实帧经生产 store 完成并发、嵌套、续接与缺失 metadata 的配对，结果与 `rawOutput.metadata.sessionID` 一一对应 | Missing |
| 不产生生命周期结论 | 配对与 `status` 不产生完成、失败或取消；重载后关联不变 | Missing |

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
| 子来源更新不续期父计时 | 生产 runtime 与暂停时钟，经真实 ACP transport 观察子文本、工具增量与子权限往返后父 prompt 仍按原窗口推进；重试后旧来源不残留 | Missing |
| 来源继承与父工具边界 | 省略 metadata 的增量沿用已知子来源；父委托 `in_progress` 的暂停计时仍成立 | Missing |

既有 `ora_agent_runtime::connection::prompt_policy_tests::child_updates_do_not_rearm_the_parent_prompt`
只证明不同 ACP session 的更新不会唤醒父 prompt，不覆盖同一会话中带子来源 metadata 的更新，不作为
本用例证据。

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
| 观察时间与记录时间可区分 | 实时、运行中重开、暖/冷加载分别核对时间来源与标注；重放流不刷新最近实时活动 | Missing |
| 状态、静默与待处理请求不越界 | 未上报状态的 provider、长时间静默、question 缺能力与权限往返的负例都保持未知或缺口，不制造健康与等待结论 | Missing |

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
| 真实界面与重载一致 | 真实 Desktop/浏览器经生产 store 展开并发嵌套、缺失来源与旧历史，实时与重载的内容、来源、终态标注一致 | Missing |
| 降级与只读边界 | 循环、缺父、未知 metadata 与历史缺口不崩溃不丢内容；界面没有新增控制入口 | Missing |

- 决策依据：D1、D3，不变量 1、3、6。
