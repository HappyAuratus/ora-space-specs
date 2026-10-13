---
status: implemented
---

# Workspace 创建与启动增加 plugin 步骤并由 Node 执行；IssueRun 的运行 Workspace 由 Cloud 发起建删

`create_project`、`create_workspace` 与 `start` 在 `node`/`clone` 之后增加 `plugin` 步骤：Cloud 在进入该步骤时
把 Space 期望安装的插件集合快照进 operation，Controller 以一次 Node 执行（`InstallPlugins`）完成安装，
结果逐个写回 `workspace_plugin_instances`。插件步骤不再计划 Substrate effect。IssueRun 的运行 Workspace
（isolated）复用 `create_workspace`/`delete_workspace`，由 Cloud 以系统身份发起，不能被公开 API 操作。

本 ADR 处于 `implemented`，继承 [运行时 Workspace 生命周期根决策](0-workspace-runtime-follows-desktop-node.md)
与 [具体默认分支决策](20260926-require-concrete-default-branch.md)。插件目录、选择状态与 fan-out 的修改
见 [plugin-marketplace 后续决策](../plugin-marketplace/20260928-node-executes-plugin-installs.md)，
运行 Workspace 的业务语义见 [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md)。
兼容性边界：已有 operation、effect 与实例行保持原记录；`step` 与 `kind` 的 CHECK 只增加取值。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根 D1：create_project/create_workspace 为 sandbox → node → clone → done，start 为 sandbox → node → done | 修改：三者在 done 前增加 `plugin`（D1） |
| 根 D1：clone 成功后开放准入并进入 `ready` | 修改：`plugin` 步骤完成后才开放准入（D1） |
| 根 D2：Workspace 的 clone 复用 `clone_executions` 与收据规则 | 继承；plugin 步骤以同样的派发前登记与接管规则登记 Node 执行（D2） |
| 根 D5：quiesce 的 idle 证据来自 Controller 派发账本 | 继承；账本包括 Agent 会话与交付执行（D4） |
| 根不变量 1：新 effect 只有三种沙盒/数据 effect 及插件 effect | 修改：不再计划 `plugin_ensure`/`plugin_delete`，只剩三种（D3） |
| 默认分支 D2：isolated Workspace 的 `baseRef: HEAD` 指 Project 默认分支 | 继承；运行 Workspace 即 isolated |

## 决策

### D1：步骤表

| Operation | 新步骤 |
|---|---|
| create_project | sandbox → node → clone → plugin → done |
| create_workspace | sandbox → node → clone → plugin → done |
| start | sandbox → node → plugin → done |
| stop / administrative_stop / delete_workspace / delete_project | 不变 |
| install_plugin / remove_plugin | plugin → done（执行方式改变，见 D2） |

- `start` 也经过 `plugin`：Workspace 停止期间 Space 的插件选择可能变化，Node 的安装是幂等的，已装的
  同版本插件立即成功，因此代价只是一次执行往返。
- 插件步骤结束后才开放准入：Agent 会话需要的 agent 插件必须已装好，准入早于插件会让 IssueRun 在插件
  尚未安装时启动。

### D2：plugin 步骤是一次 Node 执行，逐个插件报告结果

- 进入 `plugin` 步骤的事务中，Cloud 计算期望集合并写入 operation 的 `request.plugins`：
  create/start 为 Space 中 `desired_state = installed` 的全部插件；install_plugin 为该插件；remove_plugin 为
  待移除插件。每项包含 canonical ID、版本与 release 载荷（URL、SHA-256、targets），与原 `plugin_ensure`
  载荷相同。集合为空时步骤直接推进。
- Controller 用 `RecordDispatch`（`operation_id` = 该 operation）登记一个 `InstallPlugins` 或 `RemovePlugins`
  执行，输入必须等于 `request.plugins`，目标 Node 必须是该 Workspace 当前 generation 的 Node；否则冲突。
- 执行结果逐插件给出 `installed{version} | removed | failed{reason}`；接管结果的事务把每项写回
  `workspace_plugin_instances` 并重算 Space 聚合（沿用 plugin-marketplace D1 的聚合规则）。
- 所有插件都有结果后步骤推进，**单个插件失败不阻止 Workspace 就绪**：失败记录在实例上，依赖该插件的
  消费者（例如 Agent 运行需要其 agent 插件）自行判断。执行本身失败（`Interrupted`、Node 不可达）按 clone
  步骤同样的规则 defer 并以新执行重试；结果未知保持 `blocked`。

为什么不让单个插件失败阻塞：一个损坏的插件 release 会让整个 Space 的所有新 Workspace 都无法就绪，影响面
远大于缺一个插件；插件失败已经在 Space 插件页可见。

### D3：插件不再是 Substrate effect

`plugin_ensure`/`plugin_delete` 不再被计划。已有的这两类 effect 行保留为历史；迁移时仍处于 `plugin` 步骤且
未终态的 install_plugin/remove_plugin operation 回到 `plugin` 步骤的开始，按 D2 重新执行（插件安装是幂等的，
重做是安全的）。Sandbox Server 与生产 Substrate 都不需要实现插件 effect。

### D4：运行 Workspace 由 Cloud 以系统身份建删

- IssueRun 的派发事务创建运行 Workspace（`kind = isolated`、`issue_run_id` 非空）与其 `create_workspace`
  operation；进入 `releasing` 的事务创建 `delete_workspace` operation。两者的幂等身份是
  `(issue_run_id, kind)`，由 Cloud 生成，不经过公开幂等键。
- 公开 API 对运行 Workspace 的 start/stop/delete 返回 `404`：它的生命周期只属于 IssueRun。运行 Workspace
  也不出现在 Workspace 列表与运行时列表中，不参与 Space 插件的 fan-out 与聚合；它的插件只在自己的 `plugin`
  步骤安装。
- `delete_workspace` 的 quiesce 规则不变：Controller 只有在该 Node 上没有尚无终态的派发（包括 Agent 会话、
  交付与插件执行）时才报告 idle。IssueRun 保证只在交付结束后创建删除操作，因此正常路径下 quiesce 一次通过；
  若仍有未终态执行（例如交付被放弃时的会话结果未知），quiesce 失败并按原规则重试。

## 不变量

1. Workspace 在 `plugin` 步骤完成后才开放准入。
2. plugin 步骤的执行输入等于进入步骤时写入 operation 的期望集合。
3. 单个插件的失败只影响其实例状态，不阻止 Workspace 就绪。
4. 新计划的 effect 只有 `sandbox_ensure`、`sandbox_terminate`、`workspace_data_delete`。
5. 运行 Workspace 只能由其 IssueRun 发起创建与删除。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保留插件为 Substrate effect，由 Sandbox Server 执行 | 与现有契约一致；但插件安装是 Node home 内的文件与校验逻辑，desktop 已有安装器，Substrate 侧要重写一份并理解插件格式 |
| 每个插件一个 Node 执行 | 结果粒度自然；但一次 Workspace 创建产生 N 次派发往返与登记，且步骤需要跟踪 N 个执行的完成 |
| 插件在 clone 之前安装 | 插件与仓库无依赖可并行；但步骤仍是串行的，且放在 clone 后让“代码与插件都就绪”成为单一的准入条件 |
| 运行 Workspace 走公开 API 建删 | 复用现有入口；但运行 Workspace 不属于用户操作的对象，公开可删会绕过 Revision 交付 |

## 本决策未解决的问题

- 插件运行时配置（MCP 配置、密钥）的下发：第一版只安装，不配置。
- 插件更新（版本变化）时对运行中 Agent 会话的影响。

## 落地

迁移增加步骤与运行 Workspace 关联列，改写步骤表；`request.plugins` 快照（同时记录每个插件的 `desiredRevision`，
写回时据此判断结果是否仍对应当前选择）与接管写回以 PostgreSQL 集成测试覆盖（空集合直接推进、部分失败仍就绪、
执行失败重试）；desktop Node 执行安装，Go simulator 保留用于 Cloud 测试。保持不变：
sandbox/node/clone/quiesce/terminate/cleanup。开始改变：Workspace 就绪前安装 Space 插件，插件不再经 Substrate。
