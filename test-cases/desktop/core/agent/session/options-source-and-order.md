# 会话选项的来源、顺序与在途设置

决策依据：[会话选项以 session actor 为唯一来源](../../../../../decisions/desktop/core/agent/session/20261125-session-options-single-source.md)（`approved`，尚未实现）。

义务后的标注：**复现**在修复前失败，**回归保护**在修复前已经成立，**新增**是新的行为。已有证据固定到 desktop `59da662c`，只证明现有
实现，实现改动后要重新核对。宿主的顺序义务在 runtime 层由用例控制 ACP 帧的先后验证。

## The Last Option Frame Read From The Agent Must Win

### 风险

设置的答复和 Agent 的推送若走不同的路，晚到的旧值会盖掉新值，界面显示的模型和 Agent 实际使用的不一致。

### 前置状态

- 会话有 provider，当前选项是 A。
- 分别处于：回合中、空闲、标题轮询进行中、MCP 刷新进行中。

### 触发

设置选项，由用例控制 Agent 先答复 B 再推送 C，或先推送 C 再答复 B；前端再交换答复和推送到达的先后。

### 必须成立

- 当前选项是宿主最后读到的那一帧的值，各处处理队列的阶段都一样。
- 正在跟随的加载流、prompt 流和之后的加载显示同一份选项。
- 标题轮询期间收到的推送，不在下一回合开头重放；下一回合开始后当前选项仍是之后的设置答复。

### 禁止结果

- 按到达前端的先后，或按命令先于事件的处理次序决定最终值。
- 旧推送经命令队列或延后重放，排到更晚的答复之后。
- 视图按用户点击的值或设置的返回值写入会话副本。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 同一回合内两种次序，runtime 的流和之后的加载是最后的值（复现） | 尚无实现 | `Missing` |
| 前端：答复和推送到达的先后交换，单个视图的结果不变（复现） | [an option change the agent reports while following a turn wins over an earlier choice](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/chat/tests/store.test.ts#L403) 只证明答复先到、推送后到；推送先到、答复晚到没有测试 | `Partial` |
| 空闲时旧推送先到、答复后到，两帧在 actor 下一次取输入之前都已就绪（复现） | 尚无实现 | `Missing` |
| 标题轮询进行中推送先到、答复后到，两帧在 actor 下一次取输入之前都已就绪（复现） | 尚无实现 | `Missing` |
| 标题轮询期间的推送不在下一回合重放（复现） | 尚无实现 | `Missing` |
| MCP 刷新进行中推送与答复相继到达（新增） | 尚无实现 | `Missing` |
| 答复在标题轮询或 MCP 刷新期间到达也被采用（回归保护） | [a_change_answered_during_a_title_poll_is_kept](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L193)、[a_change_answered_during_an_mcp_refresh_is_kept](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L217) | `Covered` |

### 决策依据

- D1 第 2、8 点，D2。
- 不变量 1、2、5。

## Pushes Older Than A Model Intent Answer Must Not Undo It

### 风险

建立或恢复会话时，意图答复之前的旧推送若在答复之后处理，用户首轮选的模型会在界面上被撤销，之后的加载也得到旧值。

### 前置状态

- 调用方带模型意图，Agent 提供该模型。
- 分别由 `session/new` 建立会话、由 `session/load` 恢复会话、切换 Agent。
- 旧推送 A 分别落在：短收集窗口内；收集窗口之后、意图答复之前；恢复答复之后、意图答复之前。

### 触发

Agent 推送 A，再答复意图为 B，随后发送第一条 prompt 并加载会话。

### 必须成立

- 当前选项是 B，之后的加载得到 B。
- 旧推送不随第一条 prompt 送给视图。

### 禁止结果

- 只丢弃暂存的更新，而仍留在事件队列里的旧推送照样被采用。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 收集窗口之后的旧推送不被采用，之后的加载得到意图答复的值（复现） | 尚无实现 | `Missing` |
| 收集窗口之后的旧推送不随第一条 prompt 送给视图（回归保护） | 尚无直接测试 | `Missing` |
| 恢复答复之后、意图答复之前的旧推送不被采用（复现） | 尚无实现 | `Missing` |
| 收集窗口内的旧推送不撤销意图答复，不随第一条 prompt 送出（回归保护） | [a_snapshot_older_than_the_applied_model_does_not_undo_it](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/setup_options_tests.rs#L171) | `Covered` |

### 决策依据

- D2 第 2、3 点。

## Frames From A Replaced Binding Must Not Describe Its Replacement

### 风险

会话被重建后采用旧绑定的答复，界面会显示替代会话并没有使用的模型。

### 前置状态

- 恢复的会话首轮失败、即将重建，期间有一个设置在途。
- 替代会话报告 B，用户设置的是 A。

### 触发

旧绑定的答复 A 分别在替代会话的回合中和回合后到达。

### 必须成立

- 当前选项、所有流和之后的加载都是 B。
- 设置返回成功，视图得到 B；A 不在替代会话上重发，也不报错。

### 禁止结果

- 旧绑定晚到的答复或推送改变当前选项。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 旧绑定晚到的答复不改变当前选项，之后的加载是替代会话的选项（回归保护） | [a_late_answer_does_not_describe_the_session_that_replaced_its_own](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L241) | `Covered` |
| 重建时设置返回成功，视图得到替代会话的选项（回归保护） | [a_late_answer_does_not_describe_the_session_that_replaced_its_own](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L241) 证明晚到的答复返回成功；重建时立即收尾、视图由流得到选项尚无证据 | `Partial` |
| 旧绑定晚到的推送不改变当前选项（回归保护） | 尚无直接测试 | `Missing` |

### 决策依据

- D1 第 3 点与在途设置的收尾，D2 第 4 点。
- 不变量 3。

## Configuration Calls Must Report Only Success Or Failure

### 风险

返回值若仍被当作当前选项，就又多一个写入者；失败被当成成功，用户会以为模型已经生效。

### 前置状态

- 会话有 provider。
- Agent 分别：照所选的值生效；答复别的值；返回 ACP 错误；超时之后才答复；两个设置同时在途。

### 触发

设置选项，成功后立即打开一个新视图。

### 必须成立

- 契约的返回值不带选项，视图不按所选的值写入会话副本，选择器可以在「进行中」显示所选的值。
- 成功返回时 actor 已处理完答复，之后打开的视图看到这次改动。
- Agent 没有照所选的值生效时返回成功，视图显示 Agent 报告的值；返回 ACP 错误时失败，当前选项不变。
- 超时返回失败；绑定不变时，之后到达的答复仍被采用并送达视图。
- 在途的多个设置不串行，最终值由 Agent 答复的先后决定。

### 禁止结果

- 前一个设置没有答复时挡住后一个设置。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 成功返回之后打开的视图看到改动（回归保护） | [a_view_opened_after_a_change_is_told_about_it](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L165) | `Covered` |
| ACP 错误返回失败且当前选项不变（回归保护） | [reports an unreachable model selection instead of silently keeping the old value](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/chat/tests/store.test.ts#L2449)、[returns the thumb to the reported level when the request fails](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L582) 只证明前端在请求失败时保留原选项；宿主未覆盖 | `Partial` |
| 未照所选值生效时返回成功并显示 Agent 的值（回归保护） | [adopts the agent's answer to a model selection over the requested value](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/chat/tests/store.test.ts#L2414) 只证明前端采用返回的值；宿主未覆盖 | `Partial` |
| 超时后的答复被采用并送达视图（新增） | 尚无实现 | `Missing` |
| 多个在途设置不串行（回归保护） | [lets a newer pick overtake one still in flight without snapping back](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/packages/app-shell/src/features/chat/thought-level-selector.test.tsx#L550) 只证明前端在前一个没有答复时发出后一个；宿主和按答复先后取值未覆盖 | `Partial` |
| 返回值不带选项，前端不用它写入（新增） | 尚无实现 | `Missing` |

### 决策依据

- D1 第 4–6、8 点。
- 不变量 4。

## In Flight Changes Must Settle When The Binding Is Released

### 风险

绑定释放后答复找不到路由。没有人收尾的设置会一直等到超时；读队列的某个循环不认识设置的答复，当前选项还会少一次更新。

### 前置状态

- 会话有 provider，一个设置在途，Agent 还没有答复。

### 触发

分别：切换 Agent（代表 `Stop` 命令）、prompt 流的接收方放弃、会话被重建、事件队列溢出、Agent 进程被替换；另在每一处读会话事件队列的循环中让答复到达。

### 必须成立

- `Stop` 命令、接收方放弃、会话被重建返回成功。
- Agent 失联或失控（溢出、断连、进程被替换、不活跃超时、取消无应答、刷新 MCP 失败）返回失败。
- 释放之后没有仍在等待的设置，不等到超时。
- prompt、取消时的排空、回合取消的预读、空闲排空、标题轮询、MCP 刷新、恢复会话、建会话的收集窗口中，凡可能遇到在途设置的，都能识别答复并完成设置。

### 禁止结果

- 绑定释放后设置仍等待答复。
- 溢出后返回成功，而当前选项没有更新。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 切换 Agent 时返回成功（新增） | 尚无实现 | `Missing` |
| 用户停止会话时返回成功（回归保护） | [a_change_answered_after_the_session_stopped_still_succeeds](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L277) 证明停止后到达的答复返回成功；不等答复就收尾尚无证据 | `Partial` |
| 接收方放弃时返回成功（新增） | 尚无实现 | `Missing` |
| 会话被重建时不等答复返回成功（新增） | 尚无实现 | `Missing` |
| 队列溢出时返回失败（新增） | 尚无实现 | `Missing` |
| Agent 进程被替换时返回失败（新增） | 尚无实现 | `Missing` |
| 各读队列的循环识别设置答复（新增） | 尚无实现 | `Missing` |

### 决策依据

- D1 在途设置的收尾。
- 不变量 7。

## Session Setup Answers Must Enter The Same Source

### 风险

刷新 MCP 的答复被忽略，或启动、切换的返回值晚于流中的更新写入，都会留下旧值。

### 前置状态

- 会话有 provider；另有一个刚启动的会话和一个刚切换 Agent 的会话。

### 触发

刷新 MCP 的 `session/load` 答复带选项 B；启动或切换的返回值在流送来更新的选项之后才到达前端。

### 必须成立

- 刷新的答复被采用，当前选项是 B；不带选项的答复不改变当前选项。
- 启动和切换的返回值按一次取回的结果处理，不盖掉流中更新的选项。

### 禁止结果

- 返回值盖掉更新的选项。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 刷新 MCP 的答复带选项时被采用（新增） | 尚无实现 | `Missing` |
| 刷新 MCP 的答复不带选项时不改变当前选项（回归保护） | [a_change_answered_during_an_mcp_refresh_is_kept](https://github.com/ora-space/desktop/blob/59da662c81c973b4b2e81d4b1d2cdb16db6f1c5d/crates/agent-runtime/src/restored_sessions/set_config_tests.rs#L217) | `Covered` |
| 启动、切换的返回值不盖掉流中的更新（新增） | 尚无实现 | `Missing` |

### 决策依据

- D1 第 2、8 点。
