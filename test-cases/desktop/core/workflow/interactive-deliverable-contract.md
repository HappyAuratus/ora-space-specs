# 交互式节点产出物声明

本文件是[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md)的核心用例。决策已 `approved`，desktop 与 Cloud 的实现尚未开始，以下义务全部为 `Missing`，由承接实现的运行补齐证据。

## A Declared File That Is Missing Or Unchanged Blocks Completion

### 风险

节点声明了应产出的文件，agent 没有写出或没有改动就被完成，下游节点拿到空结果；或者用户补齐文件后仍被陈旧判定挡住。

### 前置状态

- 一个交互式 Agent 节点已 park 在 `Pending`，本轮 `EndTurn`、输出契约满足；
- 冻结图里声明了 `completion.files`，含 `mustChange: false` 与 `mustChange: true` 的条目；节点启动时的 worktree 基线存在。

### 触发

用户点击完成；随后补齐或改动文件后再次点击完成；或在 `force` 下点击完成。

### 必须成立

- 没有任何匹配路径存在时返回 `WorkflowNodeIncomplete`，blocker `file_missing`，`params.path` 是原始声明；
- `mustChange: true` 而没有匹配路径在相对基线的变更集合内时，blocker `file_unchanged`；
- 文件 blockers 在 `prepare` 完成时按现场 workspace 判定，不写入 `payload.completion`；补齐文件后重试即可完成，不需要重新 park；
- 文件 blockers 与回合、输出契约 blockers 汇入同一列表，错误优先级仍是状态 → turn key → digest → blockers；
- `force` 二次确认后可覆盖文件 blockers，并在审计里记录。

### 禁止结果

- 文件缺失或未变更时无 `force` 完成；
- 把文件判定写进快照，使补齐文件后仍被陈旧 blocker 挡住；
- `force` 覆盖文件 blockers 而不留审计。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 缺失与未变更分别产生 `file_missing` / `file_unchanged`，params 带原始声明 | — | `Missing` |
| 补齐文件后重试可完成，快照不含文件 blockers | — | `Missing` |
| `force` 覆盖文件 blockers 并审计 | — | `Missing` |
| 无声明节点的完成行为与 #351 一致 | — | `Missing` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D2、D6；[交互式节点完成契约](../../../../decisions/desktop/core/workflow/20261021-interactive-node-completion.md) D1。

## Paths Stay Inside The Workspace And Glob Semantics Stay Minimal

### 风险

声明路径逃出运行 workspace，读取或判定了不属于本次运行的文件；glob 语义在不同平台上得出不同结论。

### 前置状态

- 冻结图里的 `completion.files` 含绝对路径、`..`、空段、反斜杠，或含 `{}`、`[]`、取反；
- workspace 内有指向 workspace 外的符号链接；文件名大小写与声明不同。

### 触发

加载或校验该图；完成该节点。

### 必须成立

- 绝对路径、`..`、空段与反斜杠在图校验时被拒并给出可读原因，字面量部分经 `StrictRelativePath` 校验；
- 只支持 `*`（不跨 `/`）、`**`、`?`；其他 glob 语法在图校验时被拒；
- 匹配大小写敏感，不做 Unicode 归一化；逃出 workspace 的符号链接不算匹配；
- `files` 为 1–32 条、`path` 不超过 512 字节，越界被拒。

### 禁止结果

- 跟随符号链接读取 workspace 外的文件并据此判定存在；
- 把不支持的 glob 语法当字面量或静默忽略。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 非法路径与不支持的 glob 在图校验时被拒 | — | `Missing` |
| `*` / `**` / `?` 的匹配语义与大小写敏感 | — | `Missing` |
| 逃出 workspace 的符号链接不算匹配 | — | `Missing` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D1、D3，不变量 4。

## A Missing Baseline Degrades mustChange To A Visible Warning

### 风险

升级前节点或基线写入失败时，`mustChange` 无法判定：当成阻塞会让节点永远无法完成，静默放行又让作者以为已检查。

### 前置状态

- 节点声明了 `mustChange: true` 的文件，但节点启动时的基线不存在。

### 触发

节点 park；用户点击完成。

### 必须成立

- 存在性校验仍执行，缺失时仍是 `file_missing`；
- `mustChange` 不产生 blocker，park 派生快照时在 `payload.completion.warnings` 写入 `file_check_unavailable`，投影可见；完成成功后该 warning 仍在。

### 禁止结果

- 因基线缺失阻塞完成；
- 基线缺失时不留任何可见痕迹。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 基线缺失时存在性仍判定、`mustChange` 不阻塞 | — | `Missing` |
| `file_check_unavailable` 在 park 快照的 warnings 里且投影可见 | — | `Missing` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D4。

## Unsupported Declarations And Version Mismatches Are Refused, Not Ignored

### 风险

拼写错误、未定义的 `criteria` / 清单字段或版本错配被静默忽略，节点看似声明了验收却不经检查就完成；带声明的图在 Cloud 上不经检查运行。

### 前置状态

- 图里出现 `completion.criteria`、`checklist` 或未知子字段；
- `agentConfig.schemaVersion: 3` 同时带 `completion`，或版本大于 `4`；
- 同一份带 `completion` 的图被发布到 Cloud。

### 触发

desktop 加载、保存或启动该图；Cloud 发布或导入该图。

### 必须成立

- desktop 对 `criteria`、`checklist` 与未知子字段报图校验错误并给出可读原因；
- 声明 `completion` 的图必须是 `schemaVersion: 4`；`3 + completion` 与大于 `4` 的版本被拒；旧图（版本 3、无 `completion`）行为不变；
- TypeScript 规范化显式处理 `completion` 与版本，`workflow-mock`、`workflow-transfer`、graph codec 对 3/4 两个版本往返一致；
- Cloud 侧的拒绝见 [publish-and-start-admission.md](../../../cloud/workflow/publish-and-start-admission.md) 的 `agentConfig.completion` 一行，且在 desktop 发布可写 `completion` 的编辑器之前上线。

### 禁止结果

- 静默丢弃不支持的声明字段后照常完成；
- 接受 `3 + completion` 或未知的更高版本；
- desktop 编辑器先于 Cloud 门禁开放 `completion`。

### 验证义务与证据

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| `criteria` / `checklist` / 未知子字段被拒 | — | `Missing` |
| 版本 3/4 的合法与非法组合按 D7 判定 | — | `Missing` |
| mock、transfer、codec 对新字段与版本往返一致 | — | `Missing` |
| Cloud 门禁先于编辑器发布上线 | 发布记录或 PR 合并顺序 | `Missing` |

### 决策依据

[交互式节点产出物声明](../../../../decisions/desktop/core/workflow/20261023-interactive-deliverable-contract.md) D1、D5、D7、D8 与落地顺序。
