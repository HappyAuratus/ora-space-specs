# Start 输入 kickoff 取值

依据[工作流执行根决策](../../../decisions/cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md)（D2）。

`[start, input]` 与模板 `{{#start.input#}}` 读的是这一次运行的 kickoff 文本，不是编辑器里保存的默认提示词：运行
提交的覆盖值优先，没有时回退到冻结快照里 Start 节点 authored 的提示词，两者都没有时该根不写入变量池。这条链已随
[cloud#95](https://github.com/ora-space/cloud/pull/95) 合并进 main（合并提交
[`fda6e1d`](https://github.com/ora-space/cloud/commit/fda6e1d642f055ade47daf08da3cc76b96dbd8e2)），只覆盖直接运行与
manual workflow IssueRun 两条入口；`@` Confirm 入口把平台 `prompt` 映射到 kickoff 的接线随 cloud#71 到位后补，因此
与该入口相关的义务暂记 `Missing`。下面的状态是逐条核对 cloud#95 的实现与测试后登记的：有直接证据的记 `Covered`，
只覆盖了义务一部分的记 `Partial` 并写清缺哪一部分。

## The Start Input Kickoff Resolves From The Run Override Or The Authored Prompt

- 风险：Start 变量池把编辑器里的默认提示词当成 kickoff 固定写入，用户在运行里覆盖的 kickoff 文本没有随运行持久化，
  于是 `[start, input]` 与 `{{#start.input#}}` 在 Cloud 读到默认文本、在 desktop 读到用户实际输入的文本，同一条
  condition 走出不同分支；或反过来，把「未提交 kickoff」错当成空字符串，让 `exists` / `not_exists` 答错。
- 前置状态：直接运行与 manual workflow IssueRun 都能创建 `workflow_runs` 行；运行输入对象存在 `workflow_runs.input`；
  冻结快照里的 Start 节点可能带 authored 提示词，也可能不带；`@` Confirm 入口在 cloud#71 前不会创建 WorkflowRun。
- 触发：提交带 kickoff 覆盖值的运行；提交不带覆盖值、但快照里 authored 提示词非空的运行；两者都没有的运行；覆盖值
  为空字符串、为 `null`、为非字符串类型；作者声明了一个名为 `input` 的输入变量；读取一条在本语义落地之前创建的历史
  运行行。
- 必须成立：`[start, input]` 选择器比较与 `{{#start.input#}}` 期望值模板读同一份取值——先读运行行上的覆盖值，再回退
  快照里的 authored 提示词，两者都没有时该根不写入变量池；空字符串是值（present-empty）而不是未设置；保留键只接受
  字符串，非字符串在启动门禁被拒；作者声明的同名变量被 kickoff 覆盖；历史运行按其快照冻结时的文本求值、不回填。
- 禁止结果：把运行提交的 kickoff 丢掉、固定返回 authored 文本；把 `null`／缺键与空字符串混为一谈；把非字符串覆盖值
  静默当成合法值放进变量池；两者都没有时把该根写成空字符串；把历史行里既有的字符串 `input` 键当成变量值而不是
  kickoff 读出来。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| `[start, input]` 与 `{{#start.input#}}` 按「覆盖值 → authored → 根缺失」取值 | 真实 PostgreSQL：覆盖值同时驱动选择器比较与期望值模板，覆盖值与 authored 不同，读错来源会走另一分支；未覆盖时读 authored | Covered |
| 未设置与空字符串区分，`exists` / `not_exists` / `empty` / `not_empty` 据此回答 | 真实 PostgreSQL：两者都无时 `not_exists` 命中、`empty` 不把缺失当空串；空串覆盖值与空 authored 提示词是 present-empty | Covered |
| 保留键 `input` 只接受字符串，非字符串在启动门禁被拒；作者声明同名变量被覆盖 | 真实 HTTP + 单元：非字符串覆盖值 400 `invalid_field_value`（`field=input`）；声明了 `input` 变量的图不因类型判断卡死 | Covered |
| 历史行按其快照冻结时的文本求值，既有键按新语义重新解释 | 真实 PostgreSQL：无键历史行回退 authored；既有字符串 `input` 键被读作 kickoff；既有非字符串键回退 authored | Partial |
| `@` Confirm 入口把平台 `prompt` 映射到同一 kickoff | 真实 HTTP + PostgreSQL：确认后的运行行写入 `executionInput["input"]`，Start 输出读到它 | Missing |

- 决策依据：D2（Start `input` 根在运行时的取值来源）；范围拆分依据 [circus#287](https://github.com/ora-space/circus/issues/287)。
- desktop 参考实现（`8909f07`）一侧：**Covered（取值链）** — `kickoff_input.or_else(|| graph.start_node().instruction)`
  在 [`crates/application/src/workflow_run/handlers.rs:148`](https://github.com/ora-space/desktop/blob/8909f07/crates/application/src/workflow_run/handlers.rs#L148)
  解析，`Some(text)` 写进 `workflow_runs.input` 并 mirror 到 `{start_id}.input`，`None` 时不写这个池值；`mirror_run_input_into_pool`
  在更新运行时按同一条规则维护，历史运行保留创建时写的文本。Cloud 的「直接运行／manual／`@` Confirm 三个入口共用门禁」
  与「历史行的保留键重新解释」是 Cloud 自己的边界，参考实现侧没有对应入口或迁移，按 `Missing` 记。
- 本次直接证据：**前三行 Covered，第四行 Partial，第五行 Missing** — 取值链的共用解析函数是
  [`internal/core/workflow_scheduler.go`](https://github.com/ora-space/cloud/blob/fda6e1d642f055ade47daf08da3cc76b96dbd8e2/internal/core/workflow_scheduler.go)
  的 `WorkflowStartKickoff`（先读 `input["input"]` 的非 `nil` 字符串，再回退 `data["input"]`，都没有返回未设置；导出给
  dev simulator 共用，避免模拟 trace 与真实 Start 输出漂移）。第一行的直接证据是
  [`internal/core/workflow_scheduler_db_test.go`](https://github.com/ora-space/cloud/blob/fda6e1d642f055ade47daf08da3cc76b96dbd8e2/internal/core/workflow_scheduler_db_test.go)
  的 `TestWorkflowAdvanceStartInputUsesTheStoredKickoff`：覆盖值 `"run override"` 与 authored `"authored prompt"` 不同，
  同一条图里的 `[start, input]` 比较与 `{{#start.input#}}` 模板都按覆盖值选分支，Start 输出也是覆盖值；`TestWorkflowAdvanceSucceedsLinearRun`
  未提交覆盖值、读回 authored `"review the diff"`；单元层
  [`internal/core/workflow_scheduler_test.go`](https://github.com/ora-space/cloud/blob/fda6e1d642f055ade47daf08da3cc76b96dbd8e2/internal/core/workflow_scheduler_test.go)
  的 `TestWorkflowStartOutputMatchesDesktopVariableRoots` 表驱动逐例钉住「覆盖值胜出／未提交回退 authored／显式 `null` 回退／
  空串覆盖值保留／两者都无根缺失」，HTTP 集成
  [`integration/workflow_run_test.go`](https://github.com/ora-space/cloud/blob/fda6e1d642f055ade47daf08da3cc76b96dbd8e2/integration/workflow_run_test.go)
  的 `TestWorkflowRunLifecycle` 断言运行行保留 kickoff、Start 输出读到 `"review the diff"` 而 `branch` 仍是 `"dev"`。第二行由
  `TestWorkflowAdvanceConditionOnUnsetKickoffReadsAbsent`（两者都无，`not_exists` 命中、`exists` 不命中）与
  `TestWorkflowAdvanceConditionOnEmptyAuthoredPrompt` / `TestWorkflowAdvanceConditionNotEmptyOnEmptyAuthoredPrompt`
  （authored 空串是 present-empty，`empty` 命中、`not_empty` 走 else）钉住，`TestWorkflowStartOutputMatchesDesktopVariableRoots`
  的「空串覆盖值保留」用例补上覆盖值一侧的空串语义。第三行的单元证据是
  [`internal/core/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/fda6e1d642f055ade47daf08da3cc76b96dbd8e2/internal/core/workflow_admission_test.go)
  的 `TestRequireWorkflowRunInput` 新用例（字符串覆盖值放行、显式 `null` 视为未提供、对象覆盖值被拒）与
  `TestRequireWorkflowRunInputKickoffShadowsInputVariable`（声明了 `input` 数字变量的图，字符串 kickoff 仍被放行、数字
  覆盖值被拒），HTTP 层由 `TestWorkflowRunLifecycle` 的非字符串拒绝断言补上（400 `invalid_field_value`，`field=input`）。
- 仍缺：**第四行「既有字符串 `input` 键的历史行被重新解释为 kickoff」没有直接用例**——现有证据只覆盖「无键历史行回退
  authored」（`TestWorkflowStartOutputMatchesDesktopVariableRoots` 的「未提交回退」与 `TestWorkflowAdvanceSucceedsLinearRun`），
  而「旧行已经带着字符串 `input` 键时，该键被读作 kickoff」这个重新解释边界（[circus#289](https://github.com/ora-space/circus/issues/289)
  检视里明确要求登记的）还没有一条用例钉住，因此按 `Partial` 记。**第五行整条 `Missing`**：`@` Confirm 在 cloud#71 合并前
  不会创建 WorkflowRun，平台 `prompt` 到 `executionInput["input"]` 的映射、以及「作者声明了 `prompt`（平台字段不注入）或
  资产禁用了该字段时回退 authored」的边界都没有可执行的运行可验证。
