# Agent 运行时抽取与 Node 会话执行

依据[已批准的 agent-runtime 根决策](../../../decisions/node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md)。
证据预期来自 desktop Backend 既有测试、`ora-agent-runtime` 测试与 `apps/ora-node` 以 echo agent 夹具的测试；
实现尚未开始，全部为 `Missing`。

## Extracting The Runtime Leaves Desktop Behaviour Unchanged

### 风险

抽取共享 crate 时改变 Desktop 的 Agent 行为或会话 JSONL 格式，破坏已有会话的读取。

### 前置状态

抽取前的 Backend 测试全部通过。

### 触发

把 `crates/backend/src/agent_runtime` 移入 `ora-agent-runtime`，Backend 改为实现五个宿主接口。

### 必须成立

- Backend 既有 Agent 运行时测试在抽取前后都通过，不删除、不弱化。
- `ora-agent-runtime` 不依赖 Backend、SQLite 实现或 Tauri。
- 会话 JSONL 格式不变，抽取前写入的记录可被抽取后读取。

### 禁止结果

为通过编译删除或放宽测试；crate 依赖 `ora-backend`/`ora-db`。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 行为不变 | Backend 既有测试集在抽取 PR 前后同一集合全部通过 | Missing |
| 依赖边界 | 依赖检查：`cargo tree -p ora-agent-runtime` 不含 Backend、`ora-db`、Tauri | Missing |

### 决策依据

agent-runtime D1，不变量 1、2。

## Session Records Reach JSONL Before The Thread

### 风险

Thread 中出现 JSONL 里没有的记录，或崩溃后两边不一致且无法解释。

### 前置状态

Node 运行 echo agent 的会话执行。

### 触发

会话产生多条记录；在追加 JSONL 之后、`append_thread_event` 之前注入崩溃；Node 重启。

### 必须成立

- 每条记录先追加 JSONL，再经 `append_thread_event` 持久化，之后才发送。
- 崩溃后 Thread 至多缺最后一条，JSONL 完整；重启时会话以 `interrupted` 结束，记录已封口。
- 用户消息记录携带 `turn_id`。

### 禁止结果

Thread 有 JSONL 中不存在的记录；重启后恢复出运行中的会话。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 记录顺序 | Node 测试：内存 `SessionLedger` 记录的每条事件都能在 JSONL 中找到且顺序一致 | Missing |
| 崩溃窗口 | Node 测试：注入崩溃后重启，`recover_interrupted` 返回 `interrupted`，JSONL 比 Thread 多至多一条 | Missing |

### 决策依据

agent-runtime D3、D6、D7，不变量 4、7；协议后续决策 D6。

## Sessions Start Only Their Plugin With Identity Confined To Its Process Tree

### 风险

Node 启动了未请求的插件、错误版本的插件，或把 git 身份写进仓库配置影响后续交付。

### 前置状态

Workspace 插件根装有 agent 插件与其他插件。

### 触发

会话以正确版本、错误版本、未安装的插件启动；Agent 进程执行 `git commit`。

### 必须成立

- 只启动指定版本的 agent 插件，在该执行的进程 scope 内；不符或未安装时以 `agent_failed{agent_plugin_unavailable}` 结束。
- Agent 进程树的提交使用 `GIT_AUTHOR_*`/`GIT_COMMITTER_*`，checkout 的 `.git/config` 与全局配置不变。
- 会话期间持有插件租约，结束后释放。

### 禁止结果

启动其他插件；修改 git 配置；会话结束后租约仍被持有。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 插件选择 | Node 测试：版本不符时不启动任何插件进程并以 `agent_plugin_unavailable` 结束 | Missing |
| git 身份隔离 | Node 测试：echo agent 提交后 author/committer 为输入身份，`.git/config` 未变化 | Missing |

### 决策依据

agent-runtime D2、D4、D7，不变量 3、5、8。
