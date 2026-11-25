# 会话选项的送达与失去 provider

决策依据：[会话选项以 session actor 为唯一来源](../../../../../decisions/desktop/core/agent/session/20261125-session-options-single-source.md)（`approved`，尚未实现）。

义务后的标注：**复现**在修复前失败，**回归保护**在修复前已经成立，**新增**是新的行为。已有证据固定到 desktop `59da662c`，只证明现有
实现，实现改动后要重新核对。Desktop 只有一份会话副本，不做界面级的多视图
验证：多视图在 runtime 层用多条加载流、跟随流和应用事件订阅验证，前端用一份会话副本验证单个视图的写入规则。

## Every Session Stream Must Start With The Current Options

### 风险

流没有可靠的起点，视图在流开始时丢弃取回的结果，就可能丢掉唯一的新值。

### 前置状态

- 会话已附着，上次流结束之后当前选项变过。
- 另设当前选项为空集合、会话没有 provider 两种状态。

### 触发

分别打开 prompt 流、空闲时的加载流、回合中跟随的加载流。

### 必须成立

- 有 provider 时，每条流的开头都是 actor 的当前选项，空集合也带。
- 没有 provider 时流不带选项，流结束后视图取一次，得知「没有 provider」。

### 禁止结果

- 已附着会话的 prompt 流不带选项。
- 因选项为空而省略，视图保留旧选项。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 已附着会话的 prompt 流以当前选项开头（复现） | 尚无实现 | `Missing` |
| 空集合也带（复现） | 尚无实现 | `Missing` |
| 加载流以非空的当前选项开头（回归保护） | [a_view_opened_after_a_change_is_told_about_it](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs) | `Covered` |
| 跟随流以非空的当前选项开头（回归保护） | [a_view_opened_after_a_change_is_told_about_it](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs) 的回合中分支 | `Covered` |
| 没有 provider 的加载结束后视图得知「没有 provider」（新增） | 尚无实现 | `Missing` |

### 决策依据

- D3 第 2、5 点。
- 不变量 6。

## Every Option Change Must Be Announced Without Carrying Options

### 风险

只在空闲时通知会漏掉回合中的变化；通知带值又多一个来源。空闲时没有流，通知丢了视图会停在旧值。

### 前置状态

- 视图加载完成，没有流；会话分别空闲、有工作流等后端发起的回合在跑。
- 另有一条正在跟随该回合的流。

### 触发

Agent 推送选项；另在应用事件断开期间推送，再重新订阅。

### 必须成立

- 每次变化都发出应用级广播，只带 Ora 会话 id。
- 没有流的视图取一次，显示新值；有流的视图以流为准。
- 重新订阅后，对正在展示的会话重新取一次，得到新值。
- 广播和查询不访问 Agent，不改变会话状态，不写历史。

### 禁止结果

- 广播带选项。
- 因回合在跑而不广播。
- 用加载代替查询，清掉「已切换模型」标记。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 空闲时 Agent 推送，单个视图经广播和查询显示新值（复现） | 尚无实现 | `Missing` |
| 回合中变化也广播，没有跟随的视图取到新值（新增） | 尚无实现 | `Missing` |
| 有流的视图以流为准（新增） | 尚无实现 | `Missing` |
| 重新订阅后取到新值（新增） | 尚无实现 | `Missing` |
| 广播和查询只读（新增） | 尚无实现 | `Missing` |

### 决策依据

- D3 第 1、3、4、8、9 点。
- 不变量 5、8。

## Fetched Options Must Not Overwrite Newer Stream Options

### 风险

取回的结果和流是两条路，到达先后不定。取回的旧值盖掉流中的新值，或有流时忽略的广播恰是唯一的新值，界面都会停在旧值。

### 前置状态

- 视图正在取当前选项，由用例控制结果返回的时刻。

### 触发

取的期间：收到流中的选项、流开始、又收到广播；以及回合刚结束后选项变化，广播先于流的结束到达视图。

### 必须成立

- 期间从流收到选项或流开始，丢弃这次结果。
- 期间又收到广播，取完再取一次。
- 有流时收到的广播，流结束后补取；流没有送来选项就结束时也补取。

### 禁止结果

- 为判断先后给选项加版本号。
- 有流时收到的广播被直接忽略。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 晚到的取回结果不盖掉流中的选项（新增） | 尚无实现 | `Missing` |
| 取的期间又收到广播会再取（新增） | 尚无实现 | `Missing` |
| 流结束之前收到的广播在流结束后补取（新增） | 尚无实现 | `Missing` |
| 流没有送来选项就结束时补取（新增） | 尚无实现 | `Missing` |

### 决策依据

- D3 第 5 点。
- 不变量 5。

## Losing The Provider Must Turn Selection Into An Intent

### 风险

视图不知道会话已失去 provider，选择器会继续向已停止的会话发设置请求；按选项是否为空判断，又会丢掉只读标签，或把回合中的会话误判为没有 provider。

### 前置状态

- 会话有选项，随后分别：停止、空闲时队列溢出、Agent 进程被替换。插件仍可用。
- 另有一个工作流节点的只读选择器，以及一个回合进行中的会话。

### 触发

失去 provider；用户选模型；再发送一条 prompt 重新附着；另在没有 provider 时直接发设置请求；另在回合、MCP 刷新、标题轮询、附着进行中发查询。

### 必须成立

- 发出广播；查询按是否有通道回答，答「没有 provider」。
- 视图记下这一状态，不清空最后的选项；可设置的选择器只记下次附着的意图，不发设置请求，不报错。
- 只读选择器仍显示最后的模型。
- 重新附着后流开头带选项，视图解除这一状态，意图按附着的规则应用。
- 没有 provider 时到达的设置返回失败。
- 回合、MCP 刷新、标题轮询期间查询立即答当前选项；附着等不读命令的步骤完成后答复。

### 禁止结果

- 停止的会话里选模型报错。
- 被操作占用的通道被当成没有 provider。
- 只读选择器改为显示发现目录里的默认模型。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 停止、溢出、进程被替换都广播，查询答「没有 provider」（新增） | 尚无实现 | `Missing` |
| 可设置的选择器只记意图、不发设置请求（复现） | 尚无实现 | `Missing` |
| 只读选择器保留最后的模型（回归保护） | 尚无直接测试 | `Missing` |
| 重新附着后解除状态（新增） | 尚无实现 | `Missing` |
| 没有 provider 时的设置返回失败（新增） | 尚无实现 | `Missing` |
| 回合、MCP 刷新、标题轮询期间立即答复，附着完成后答复（新增） | 尚无实现 | `Missing` |

### 决策依据

- D1 第 7 点，D3 第 3、4、6 点。

## The Initiating View Must Finish On The Fetched Options

### 风险

返回值不带选项后，选择器过早结束「进行中」会闪回旧值；标记的来源改变后，可能漏记或把 Agent 自己改的模型也记上。

### 前置状态

- 会话有 provider 且至少有一个回合；分别空闲、有流。

### 触发

用户切换模型，通知丢失或晚到；另由 Agent 自己改模型。

### 必须成立

- 空闲时设置成功后主动取一次，得到选项才结束「进行中」，不依赖通知；有流时由流得到。
- 「已切换模型」标记只在用户发起的切换生效时记一次；Agent 自己改的模型不记。

### 禁止结果

- 先闪回旧值再显示新值。
- 同一次切换记两次标记，或不记。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 空闲时成功后主动取一次，通知丢失也不停在旧值（新增） | 尚无实现 | `Missing` |
| 得到选项后才结束「进行中」（回归保护） | [keeps the thumb on the chosen level while the provider answers](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L510) 证明推理强度选择器；模型选择器和由取回结果结束尚无证据 | `Partial` |
| 标记只由用户发起的切换记一次（回归保护） | [marks the thread where the answering model changed](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/chat/tests/store.test.ts#L2489)、[keeps one marker per point in the thread while the model is cycled](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/chat/tests/store.test.ts#L2579) 证明用户切换的标记；Agent 自己改的模型不记尚无直接测试 | `Partial` |

### 决策依据

- D3 第 7 点。
