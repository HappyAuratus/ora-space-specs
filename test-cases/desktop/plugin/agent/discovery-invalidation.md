# 发现结果的缓存与失效

决策依据：[Agent 插件发现契约](../../../../decisions/desktop/plugin/agent/0-discovery-contract.md)（`approved`）。Desktop 一侧的修订号、失效事件和界面缓存已实现，证据取自 desktop `339aa926`，回滚的证据取自 desktop `346a9916`；插件侧的缓存和等待尚未实现。

## Skill Catalog Changes Must Invalidate Discovered Commands

### 风险

Effect 退役后没有进程替换来清缓存。失效漏掉一处，用户会看到已卸载的 Skill，或看不到新装的。

### 前置状态

- 插件以完整发现输入为键缓存，缓存尚未过期；界面也有未过期的结果。
- 依次：安装、改名、升级、原地编辑、卸载 Skill；再以相同路径、名称和描述装回内容不同的版本。两次变化之间有的有发现请求，有的没有。
- 另有一次 Available 集合变化而没有 Catalog 提交，例如包里的 `SKILL.md` 被删除后又恢复。
- 原地编辑、改名、删除各有一次在写入 Catalog 时失败。

### 触发

每次变化提交或被 Desktop 观察到之后，发起一次需要命令的发现。

### 必须成立

- 每次提交的 Skill Catalog 变更、每次观察到的 Available 集合变化都推进修订号，包括清单恢复旧值和清单不变的原地编辑；修订号在 Desktop 进程内单调递增。
- 原地编辑、改名、删除在写入 Catalog 之前已经在磁盘上换上或移走包；写入失败而回滚时，这些暴露过中间状态的回滚推进一次修订号。
- 同一时点发布 `AgentModelsInvalidated`。
- 界面与 Desktop 的事件流断开后重新连上时，对所有 Agent 需要命令的发现结果失效一次。
- 变化之后的发现不返回变化之前的结果。
- 失效不重启 Agent，不影响进行中的 turn。

### 禁止结果

- 用清单摘要判断是否推进，使恢复旧值命中旧缓存。
- 没有暴露到磁盘的回滚推进修订号：变更在动到包之前就失败，或新建在写入失败时移除一个还没有 Catalog 记录、发现读不到的包。
- 未生效的变更推进修订号。
- 插件把发现结果写进 host 存储或文件。
- 失效本身启动发现进程。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 每种已提交的变更都推进修订号并在同一时点发布事件 | [`tests.rs`](https://github.com/ora-space/desktop/blob/339aa926/crates/backend/src/skill_revision/tests.rs) 的 `each_committed_change_advances_the_revision`、`an_import_advances_the_revision`、`a_skill_plugin_advances_the_revision`、`a_change_invalidates_what_every_agent_offered` | `Covered` |
| 读清单时观察到 Available 集合变化，推进修订号 | [`tests.rs`](https://github.com/ora-space/desktop/blob/339aa926/crates/backend/src/skill_revision/tests.rs) 的 `noticing_another_set_of_available_skills_advances_the_revision` | `Covered` |
| 暴露过中间状态的回滚推进一次，没有暴露的回滚和未生效的变更不推进 | [`tests.rs`](https://github.com/ora-space/desktop/blob/346a9916/crates/backend/src/skill_revision/tests.rs) 的 `an_edit_rolled_back_after_its_swap_advances_the_revision`、`a_rename_rolled_back_after_its_swap_advances_the_revision`、`a_delete_rolled_back_after_its_move_advances_the_revision`、`a_create_rolled_back_leaves_the_revision_alone`、`a_change_that_does_not_take_effect_leaves_the_revision_alone`、`a_create_is_counted_once_it_is_written` | `Covered` |
| 事件流重连后，所有 Agent 需要命令的结果失效一次，只取模型的不失效 | [`use-app-events.test.ts`](https://github.com/ora-space/desktop/blob/339aa926/packages/app-shell/src/state/hooks/use-app-events.test.ts) 的 `refreshes discovered commands when Skills change, or may have while disconnected` | `Covered` |
| 插件缓存不落盘 | 插件侧尚未实现 | `Missing` |
| 卸载后装回同名不同内容的 Skill，不返回卸载前的结果 | [`tests.rs`](https://github.com/ora-space/desktop/blob/339aa926/crates/backend/src/skill_revision/tests.rs) 的 `a_skill_plugin_advances_the_revision` 证明卸载和以同一投影装回都推进修订号；插件侧缓存未实现，端到端没有覆盖 | `Partial` |
| 失效不重启 Agent、不影响进行中的 turn | 尚无直接证据 | `Missing` |

Available 集合依赖 Skill 插件的启用状态时，启用或禁用同样推进修订号并发布事件。目前 Available 集合不依赖启用状态，Desktop 也不轮询，这条义务目前不适用，不列入证据表。

### 决策依据

- D6 第 1、2、4、7 点，以及 D6 开头“下一次需要命令的发现不得返回变化之前的结果”（暴露过中间状态的回滚）。
- 不变量 9。绕过 Ora 直接修改 Skill 目录不在范围内。

## Results Requested Before A Skill Change Must Not Be Shown After It

### 风险

变化之前发出的发现可能在变化之后才完成。把它写回缓存，等于取消刚发生的失效，旧列表会留到有效期结束。

### 前置状态

- 一个需要命令的发现请求正在进行，完成时机可控。
- 界面分别正在显示和没有显示该结果。

### 触发

提交一次 Skill Catalog 变更，再让原请求完成。

### 必须成立

- 原请求的结果不作为变化之后的结果展示或缓存。
- 正在显示的界面可以立即用新修订号重新发现；没有显示的到下次需要时再发现。

### 禁止结果

- 原请求晚于新请求完成时覆盖新结果。
- 没有界面需要结果时启动发现进程。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 变化前的请求后完成时不展示、不缓存 | [`plugin-lifecycle.test.ts`](https://github.com/ora-space/desktop/blob/339aa926/packages/app-shell/src/state/data/plugin-lifecycle.test.ts) 的 `shows no commands from before a Skill change that interrupts a running refresh`、`shows no commands from before a Skill change, while refreshing or after a failed refresh` 只证明进行中的刷新被打断、刷新失败时不再展示旧命令；旧请求随后成功完成的路径没有用例 | `Partial` |
| 后到的旧结果不覆盖新结果 | 尚无直接证据 | `Missing` |
| 没有界面需要时不启动发现 | 尚无直接证据 | `Missing` |

### 决策依据

- D6 第 4、7 点。
- 不变量 9。

## Model Only Results Must Not Hide Commands Or Follow Skill Changes

### 风险

工作流编辑器和对话选择器面向同一 Agent 和 Workspace。共用缓存会让编辑器得到的 `None` 挡住选择器的命令；反过来，装一个 Skill 会让编辑器为每个 Agent 各起一个发现进程。

### 前置状态

- 工作流编辑器以不需要命令的方式列出多个 Agent 的模型，结果未过期。
- 同一 Agent 和 Workspace 的对话选择器需要命令。

### 触发

先打开编辑器再打开选择器，另按相反顺序再做一次；随后在编辑器打开时提交一次 Skill Catalog 变更。

### 必须成立

- 编辑器的结果不满足选择器，选择器自己发现并得到命令。
- 选择器的结果可以满足只取模型的请求。
- Skill 变化不使只取模型的结果失效，编辑器、定时任务校验和 SDK `agent/list_models` 都不受影响。
- 不需要命令时，Skill 输入和修订号不参与插件和 Desktop 的缓存键。

### 禁止结果

- 选择器在有效期内读到编辑器写入的 `None`。
- Skill 变化使编辑器为各 Agent 重新起发现进程。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 只取模型的结果不满足需要命令的请求 | [`use-workflow-agent-models.test.tsx`](https://github.com/ora-space/desktop/blob/339aa926/packages/app-shell/src/state/hooks/use-workflow-agent-models.test.tsx) 的 `never lets a models-only answer hide the commands` | `Covered` |
| 需要命令的结果可以满足只取模型的请求 | [`use-workflow-agent-models.test.tsx`](https://github.com/ora-space/desktop/blob/339aa926/packages/app-shell/src/state/hooks/use-workflow-agent-models.test.tsx) 的 `answers the editor from a chat's discovery that is still good` | `Covered` |
| Skill 变化后编辑器不重新发现 | [`use-workflow-agent-models.test.tsx`](https://github.com/ora-space/desktop/blob/339aa926/packages/app-shell/src/state/hooks/use-workflow-agent-models.test.tsx) 的 `does not ask the editor's agents again when Skills change after the %s asked`；[`plugin-lifecycle.test.ts`](https://github.com/ora-space/desktop/blob/339aa926/packages/app-shell/src/state/data/plugin-lifecycle.test.ts) 的 `refreshes only the commands of one agent when Skills change` | `Covered` |

### 决策依据

- D1 第 3 点，D6 第 2、3、5 点。
- 不变量 14。

## Missing Commands Must Be Cached While Failures Are Not

### 风险

不缓存 `None`，不上报命令的 Agent 每次展示都再起一个进程；缓存失败，一次偶然的错误会在有效期内让模型列不出来。

### 前置状态

- 使用可控时钟和会统计启动次数的 Agent。
- Agent 分别不上报命令、发现报错。

### 触发

在有效期内重复发起同一发现，过期后再发起一次。

### 必须成立

- `None` 与其他结果一样按有效期复用，过期后重新发现。
- 失败不缓存，下一次请求重新发现。

### 禁止结果

- 把失败当作 `None` 或空模型列表缓存。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `None` 在有效期内不再启动进程，过期后重新发现 | 尚无直接证据 | `Missing` |
| 失败后下一次请求重新发现 | 尚无直接证据 | `Missing` |

### 决策依据

- D6 第 6 点。
- 不变量 3。
