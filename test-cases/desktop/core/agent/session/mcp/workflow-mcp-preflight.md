# 工作流派发前 MCP 预检

## Workflow Mcp Preflight Must Reject Missing Selections Before Dispatch

### 风险

若在已选 MCP 缺失时开始调度，前面的节点可能已经产生副作用。若把配置或能力错误也提前到运行门禁，用户会收到错误的恢复提示。

### 前置状态与触发

冻结图包含启用的 MCP 绑定。分别对 `Pending` 运行调用 start，对可重启的运行（含 `Succeeded`）调用 restart，对可恢复的 `Failed` 或 `Cancelled` 运行保留快照调用 resume。另以 `Running` 运行验证重复命令。

### 必须成立

- 已启用绑定对应的插件缺失、包无效或未提供 MCP 时，调度命令返回 `workflow_mcp_not_found`，可定位节点与 MCP，状态不变。只在未执行的条件分支、循环体或已经成功的节点中出现的已选 MCP 也受检查。
- 未选择或禁用的绑定允许其插件缺失。运行期间卸载依赖后，重复 start 仍返回当前运行，restart 仍返回 `workflow_run_not_restartable`。
- 配置不完整或 Agent 能力不足要在节点 session setup 报错；运行开始后卸载、或在预检和派发之间卸载，由后续 session setup 拒绝已选插件缺失。已有 Session 的恢复保持原选择并 fail closed。
- 错误只含节点 id、节点标题和 MCP id；预检只读。
- 不可恢复的 resume 先按原有规则返回 `workflow_run_not_resumable`，依赖缺失不改变这个结果。

### 禁止结果

预检拒绝后不得派发节点、创建 Session、发送 ACP、改写 worktree 或改变运行状态；不得暴露配置实值，亦不得以部分 MCP 集合继续。

### 验证义务与证据

| 决策与验证义务 | 主线直接证据或缺口 | 状态 |
| --- | --- | --- |
| D1：卸载已选 MCP 后 start 被拒，状态仍为 `Pending` | [移除后启动](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L394)比较完整错误并重新读取 `Pending`；重装后同一运行可以启动 | `Covered` |
| D1：同一 MCP 缺失阻止 restart 和保留快照 resume，原状态不变 | [取消后移除 MCP](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L425)分别调用 restart 和保留快照的 resume，两次拒绝后运行记录完全相同；另两种可调度状态由 [已成功后 restart](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L552) 和 [失败后 resume](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L598) 覆盖：两者都被 `workflow_mcp_not_found` 拒绝，1 秒观察窗口内运行、节点详情、Session 行和新会话记录都与拒绝前相同 | `Covered` |
| D1：不可恢复的运行先按恢复资格拒绝，不先报依赖错误 | [恢复资格优先](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L722)：`Pending` 运行同时缺 MCP 时，resume 返回 `workflow_run_not_resumable` 而非依赖错误，状态仍为 `Pending` | `Covered` |
| D1：卸载后重复 start 和对 Running 的 restart 仍给出原结果 | [移除后重复命令](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L620)断言再次 start 返回 `Running`，对该运行 restart 仍返回 `workflow_run_not_restartable` | `Covered` |
| D1：检查整张图，忽略禁用绑定 | [禁用绑定时启动](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L455)验证禁用绑定被跳过；[未命中的条件分支与不会进入的循环体](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L461)里 `reviewer` 的启用 MCP 仍阻止 start，而依赖齐全时同一张图运行成功、`gate` 走 `else`，没有 `reviewer` 和 `loop` 的节点记录；[已成功节点上的 MCP 阻止 `Failed` resume](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L598)。分支与循环体的遍历只有 e2e 覆盖：`ora-backend` 的 precheck 用例在只查顶层节点时仍通过 | `Covered` |
| D2：包无效及同 ID 非 MCP 均按缺失处理 | [损坏 MCP 包](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L655)和[同 ID 的 Skill 包](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L467)分别断言 start 被拒且运行保持 `Pending` | `Covered` |
| D2：已安装但配置不完整的 MCP 通过预检，随后在 session setup 失败 | [同一次运行的两侧](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L646)：配置不完整的选择通过 start，节点随后在 session setup 以配置不完整失败，不交付 Session 也没有新会话记录；[缺少必填设置的运行获准启动](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L620)和[显式选择对缺失或配置不完整插件报错](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/session_setup/mcp/selection_tests.rs#L85)固定两侧的判断边界 | `Covered` |
| D2：预检之后发生卸载，节点按 setup 错误失败，恢复不会放宽选择 | [预检与派发之间卸载](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L814)：start 返回 `Running` 后在 current-thread runtime 上先卸载包，节点在 session setup 以 `selected MCP plugin official/first is not installed` 失败，不交付 Session 也没有新会话记录。[卸载后恢复既有 Session 按原选择拒绝](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp/recovery.rs#L47)：节点已持有选了两个 MCP 的 Session 后停止它、把草稿缩成只剩一个 MCP 并卸载另一个，经生产 prompt 接口续接时在仍存活的 actor 上被 `mcp_selected_plugin_unavailable`（`plugin_id` 指向被卸载的 MCP）拒绝，ACP 日志不变；关闭并重开同一数据目录、从持久化选择重建 actor 后再续接，得到同样的拒绝。重装后同一个 provider Session 经 `session/load` 恢复原来的完整选择（两个 MCP 加工作流完成工具）。未卸载时[修改草稿后恢复仍使用冻结的选择](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L141)继续覆盖 | `Covered` |
| D4：错误只含节点与 MCP 身份，标题缺省时仍可定位 | [空白标题回退](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/crates/backend/src/workflow/run/operations/precheck_tests.rs#L717)：节点标题只有空白时 `node_title` 回退成节点 id，整份公开错误序列化后只有 `code` 与 `nodeId`/`nodeTitle`/`mcpId`；[保留的设置实值不进入拒绝错误](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/crates/backend/src/workflow/run/operations/precheck_tests.rs#L755)：经生产设置接口写入合成设置值、以 `Retain` 卸载后 start 被拒，公开错误序列化后仍只有 `code` 与三个身份字段，重装后原设置记录原样保留；[比较完整后端错误](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/crates/backend/src/workflow/run/operations/precheck_tests.rs#L396)与[前端按标题和 MCP id 显示](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/packages/app-shell/src/i18n/contract-error.test.ts#L102) | `Covered` |
| D1/D4：拒绝不造成 Session、ACP 或 worktree 副作用 | [拒绝目标依赖时保留未提交文件](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/crates/backend/src/workflow/run/operations/precheck_tests.rs#L485)直接观测 worktree；[被拒的 start 不建 node-run 行](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/crates/backend/src/workflow/run/operations/precheck_tests.rs#L869)；Agent 就绪的 e2e 把工作区建成含已跟踪文件未提交修改和未跟踪文件的真实 checkout，在 start、restart、保留快照的 resume 被拒后 1 秒窗口内反复比较同一份[快照](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L489)（[start](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L633)、[restart](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L663)、[resume](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L716)）：运行与节点详情、完整的 `sessions` 行、fake agent 记录的全部 ACP 会话调用（`session/new`、`session/load`、`session/prompt`，后者由 [record_acp_call](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/fake-agent/acp.rs#L405) 逐次落账）、工作区路径与字节（文件内容、目录和符号链接目标，不含 `.git`，见 [worktree_contents](https://github.com/ora-space/desktop/blob/2509c866e904e1e683f580b3cfa1d600120dd814/e2e/tests/desktop/core/session/tests/workflow_mcp.rs#L457)）以及 `git for-each-ref` 的 refs（含 checkpoint ref）。restart 与 resume 用例先断言拒绝前的 ACP 日志里已有 `session/prompt`，所以向已有 Session 误发 prompt 会进入比较。工作区的观测范围是“工作区路径与字节 + refs”，不包括 HEAD 和 index；1 秒是观察窗口的上限，更慢的泄漏证明不了 | `Covered` |

### 决策依据

[工作流派发前拒绝缺失的已选 MCP](../../../../../../decisions/desktop/core/agent/session/mcp/20261119-workflow-mcp-preflight.md)，D1、D2、D4 及不变量 1–3、5。

## Snapshot Switch Resume Must Validate The Target Graph Before Rollback

### 风险

校验旧图会漏掉目标快照新增的依赖，也会错误阻止用户切到已经修复的快照。先恢复 checkpoint 再报错，会让被拒绝的命令改写工作目录。

### 前置状态与触发

运行处于可 resume 状态；保留可识别的 worktree 内容。用 `snapshot_id` 指定与当前冻结图有不同 MCP 或 Skill 绑定的目标快照并调用 resume。

### 必须成立

- 在回滚（checkpoint 恢复）或任何 worktree 改动之前，从目标图解析所需 Skill，检查启用的 MCP。
- 目标图缺少 MCP 时返回 `workflow_mcp_not_found`，缺少 Skill 时返回对应的 Skill 错误。运行状态、已选快照及 worktree 内容与命令前一致。
- 仅旧快照含有缺失绑定时允许切换到满足依赖的目标快照。

### 禁止结果

只查旧图、忽略新图中的启用绑定、先执行回滚再报依赖错误，或让被拒绝的运行进入 `Running`。

### 验证义务与证据

| 决策与验证义务 | 主线直接证据或缺口 | 状态 |
| --- | --- | --- |
| D3：目标图缺 MCP 或 Skill 时在回滚前拒绝，保留调用前运行和 worktree | [含真实 checkpoint 的两种目标图](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L483)分别拒绝缺失 MCP、Skill，比较公开运行字段（含状态和快照 id）与未提交的 `README.md`；恢复依赖后同一请求切换到目标快照并写回 checkpoint 内容，反证拒绝发生在回滚之前 | `Covered` |
| D3：旧图缺失、目标图已修复时可以切换 | [目标快照去掉缺失的绑定](https://github.com/ora-space/desktop/blob/cd1416f52291a5644bae515f6d346e72c0bc8c43/crates/backend/src/workflow/run/operations/precheck_tests.rs#L743)：取消后卸载旧图绑定的 MCP，再发布不再要求该绑定的目标快照，resume 用 `snapshot_id` 切到 `snapshot-2`，状态从 `Cancelled` 进入 `Running` | `Covered` |

### 决策依据

[派发预检 D3](../../../../../../decisions/desktop/core/agent/session/mcp/20261119-workflow-mcp-preflight.md)及[Skill 会话清单决策 D8](../../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)（`implemented`；Skill 依赖按 Catalog 解析与存储路径存在判定，裸名歧义报 `workflow_skill_ambiguous`）。
