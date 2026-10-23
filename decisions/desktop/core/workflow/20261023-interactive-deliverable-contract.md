---
status: proposed
---

# 交互式节点产出物声明：先做确定性 files 校验，criteria 与运行时清单分阶段

交互式节点的完成目前只由本轮终态和输出契约决定，不知道节点「应该产出什么」。本决策给节点增加可选的、随冻结图保存的产出物声明：`agentConfig.completion.files` 声明期望存在（或必须变更）的文件，完成时由后端对照运行 workspace 与节点启动时持久化的基线做确定性校验，缺失或未变更时拒绝完成并列出原因。criteria（自然语言验收项 + 模型判定）与运行时验收清单不在本期：前者需要单独决定模型来源、凭据、成本与降级，后者依赖 agent→host 的声明信号，分别留给后续决策。

本决策为 `proposed`。它只在冻结图里增加可选字段、在完成路径复用现有 blocker 与 `WorkflowNodeIncomplete`，不新增命令、错误码或数据库迁移；无声明节点的行为保持不变；`Loop`/`Iteration` 区域与 `agent 自报完成`不在范围内。

## 继承与修改

继承[交互式节点完成契约](20261021-interactive-node-completion.md)（`implemented`），并沿用[节点运行时编排决策](0-node-runtime-orchestration.md)的 per-run 门与失效事件边界。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 完成只认本轮 `EndTurn`，其余终态阻塞 | 继承，不变 | 仍是第一道门禁 |
| 快照在 park 写入 `payload.completion`，投影只读快照 | 继承，不变 | 快照字段与结构不变 |
| 已完成快照的 blockers 是权威，`prepare` 只做一次性修复 | 修改 | 文件校验不写快照，在 `prepare` 完成时按现场 workspace 判定；快照权威规则只作用于回合与输出契约 blockers |
| 错误优先级：状态 → turn key → digest → blockers | 继承，不变 | 文件 blockers 汇入第 4 步的 blockers 列表 |
| `force` 只覆盖 blockers，二次确认并审计 | 继承，不变 | 文件 blockers 同样可被 force 覆盖并留痕 |
| 无 turn key 的 `turn_unknown` 只能 force | 继承，不变 | 文件校验不改变该规则 |
| 投影 `completion { ready, blockers, warnings }` | 继承，不变 | `ready` 仍只反映回合/契约；文件缺失在完成被拒时返回 |
| 区域 interactive 行为不变 | 继承，不变 | 本期只覆盖外层平图 |

## 决策

### D1：声明随冻结图保存，v1 只接受 `files`

`agentConfig` 增加可选对象：

```json
"completion": {
  "files": [{ "path": "reports/*.md", "mustChange": true }]
}
```

- `files` 为 1–32 条；`path` 非空、不超过 512 字节、必须是 workspace 内的相对路径或受限 glob；`mustChange` 缺省为 `false`。
- 校验拒绝绝对路径、`..`、空段与反斜杠；路径用 `ora_utils::path::StrictRelativePath` 校验字面量部分。
- 未知子字段直接报图校验错误，不做静默忽略：避免拼写错误导致声明空转。
- `criteria`、`checklist` 在 v1 是**不支持的字段**，出现在图里即校验失败并给出可读原因；它们由后续决策定义后再加入。
- 声明随图冻结，运行只读冻结图；不改数据库 schema。

选择随图保存，是因为产出去向属于作者的节点设计，必须和 prompt、输出契约一起版本化；运行时可变配置会让同一 run 在不同时刻得出不同判定。

### D2：文件校验在完成时按现场 workspace 判定，不写入快照

`prepare` 组装完成结果时，在门外执行文件校验：

1. 从冻结图取 `completion.files`；
2. 用运行 workspace 的文件快照判断「存在」：至少一个匹配路径；
3. 读取节点启动时持久化的 worktree 基线，计算变更集合；`mustChange: true` 要求至少一个匹配路径在变更集合内；
4. 失败项转成 blockers：`file_missing`、`file_unchanged`，params 带 `path`（原始声明）。

文件判定**不写入 `payload.completion` 快照**。workspace 是可变的：用户看到缺失项后可能手动补文件再点完成，如果判定在 park 时就固化，快照会在此时变陈旧，而 #351 的规则是已存快照 blockers 权威、不自动刷新。完成时重新判定让「看到缺失 → 修复 → 重试」直接生效，也避免投影为了显示文件状态去读 workspace。

代价是 `completion.ready` 不包含文件判定，用户只有在点完成时才会看到缺失项；这是可接受的，因为缺失项来自一次明确的完成动作，且卡片会用 `WorkflowNodeIncomplete` 的 blockers 渲染。

### D3：路径与 glob 语义取最小集合

- 匹配对象是相对运行 workspace 根的 POSIX 路径；
- 支持 `*`（不跨 `/`）、`**`（跨目录）、`?`；不支持 `{}`、`[]`、转义字符与取反；
- 大小写敏感，不做 Unicode 归一化；
- 匹配只读 workspace 内的文件，不跟随逃出 workspace 的符号链接。

最小集合是为了让语义可解释、跨平台一致；复杂 glob 需要单独定义并承担性能与安全面，不在本期。

### D4：基线缺失时 `mustChange` 降级为 warning

节点启动时持久化的基线文件可能不存在（升级前节点、基线写入失败）。此时：

- 存在性校验仍执行；
- `mustChange` 无法判定，记 warning `file_check_unavailable`，不作为 blocker。

这沿用现有 `completion_file_changes` 的保守行为：没有基线时报告没有变更，而不是伪造失败。把不可判定当阻塞会让升级前节点无法完成，属于误伤。

### D5：criteria 与运行时清单分阶段

v1 不实现 `criteria` 的模型判定，也不实现运行时验收清单：

- criteria 需要一次模型调用。desktop 没有直接的模型通道，现有先例是失败诊断：用节点自己的 executor 起一次 one-shot agent session 再解析回答。judge 必须决定模型来源（节点本身还是独立的 judge 配置）、凭据、计费、人工同意、超时与失败降级，并明确不能放进完成事务或 per-run 门内。这些是产品决策，另开 ADR，前置条件是本期 files 校验落地。
- 运行时清单要求 agent 在首轮声明交付物清单、人在卡片上确认、结果存进 node run payload；它依赖 agent→host 的声明信号，与 #427 的完成请求信号同源，由 #427 或其后继承接。
- 因此 v1 在图校验层直接拒绝这两个字段，而不是接受后静默忽略。

### D6：契约与 UI 复用现有形状

- `WorkflowNodeCompletionBlockerKind` 增加 `FileMissing`、`FileUnchanged`；`WorkflowNodeIncomplete { node_id, blockers[] }` 不变，blocker 的 `params.path` 携带声明的路径，前端按 kind 本地化。
- 不新增命令、错误码或投影字段；`force` 仍覆盖 blockers 并写审计。
- inspector 增加 files 列表编辑（路径 + `mustChange` 开关），中英文案同时提供；`workflow-mock` 的图校验与 `workflow-transfer`、graph codec 的测试同步覆盖新字段。

### D7：兼容性

- Rust 的 `WireAgentConfig` 增加可选 `completion` 字段，旧图解析为 `None`；不 bump 图的 `schemaVersion`。
- TypeScript 的 `WorkflowAgentConfig` 增加 `completion`，`normalizeWorkflowAgentConfig` 显式规范化它（不依赖 `...config` 透传）；未知字段仍按现有策略保留。
- 旧编辑器保存会丢失该字段：这是不 bump schemaVersion 的已知代价，由同一版本的 inspector 同步规避；如果要求跨版本保真，再单独决定是否 bump。
- 不改数据库 schema，不迁移既有 run。

## 不变量

1. 只有声明了 `completion.files` 的节点才做文件校验；无声明节点行为与 #351 完全一致。
2. 文件判定只在完成时按现场 workspace 执行，不写入快照，不读会话历史。
3. 文件 blockers 走第 4 步的 blockers 门禁，`force` 可覆盖并审计。
4. 路径必须落在运行 workspace 内，拒绝绝对路径、`..` 与逃出 workspace 的符号链接。
5. 基线缺失时 `mustChange` 只产生 warning，不阻塞完成。
6. v1 拒绝 `completion.criteria` 与运行时清单字段，不静默忽略。
7. 不新增命令、错误码、投影字段或数据库迁移。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 在 park 时算文件 blockers 并写快照 | 用户补文件后快照陈旧，要么违反 #351 的快照权威规则，要么引入文件专属刷新规则；完成时判定更简单且行为可解释 |
| 用 LLM 判定文件是否存在 | 存在性是确定事实，模型只增加成本与不稳定 |
| 直接接 model-gateway 做 criteria | desktop 没有该通道，且模型来源、凭据与计费需要单独决策 |
| 每次完成默认跑 judge | 成本与误判高，且无法离线使用 |
| 支持完整 glob（`{}`、`[]`、取反、转义） | 语义、性能与跨平台一致性的面都变大，先用最小集合 |
| bump `schemaVersion` 并迁移旧图 | 可选字段不需要迁移，代价是全体旧图与编辑器都要处理新版本 |
| 在投影里实时读 workspace 显示文件状态 | 轮询成本高，且读取投影保持纯 payload 是现有契约 |

## 落地顺序

1. **契约与校验**：contracts 增加两个 blocker kind；application 解析并校验 `completion.files`；`prepare` 执行文件判定并汇入 blockers；单测覆盖存在/缺失/变更/未变更/glob/基线缺失。
2. **编辑器与兼容**：inspector 的 files 编辑、`workflow-runtime` 类型与规范化、`workflow-mock` 校验、`workflow-transfer` 与 graph codec 测试、中英文案。
3. **文档**：`docs/workflows/workflow-runs.md` 与 `.zh.md` 补充声明、判定时机与降级。
4. **后续**：criteria 与 judge 另开 ADR；运行时清单随 #427 的信号能力推进。

## 风险、未解决问题与验收

- **投影不显示文件缺失**：`ready` 只反映回合与输出契约，文件缺失在完成被拒时返回；需要在 inspector 文案里说明，避免用户以为按钮可用就一定能完成。
- **glob 语义有限**：复杂匹配后续扩展，扩展时保持 v1 语义不变。
- **跨平台路径**：大小写敏感、POSIX 分隔符；Windows 上大小写不同的路径不会匹配，需在文档写明。
- **旧编辑器丢字段**：不 bump schemaVersion 的已知代价。
- **基线缺失**：`mustChange` 降级为 warning，无法保证「必须变更」。
- **不在本决策范围**：criteria、judge、运行时清单、agent 自报完成、区域 interactive。

验收：

- 声明 `files` 的节点：缺失或未变更时完成返回 `WorkflowNodeIncomplete` 并列出 `file_missing` / `file_unchanged`；补齐后重试可完成；`force` 仍可覆盖并审计；
- 无声明节点行为不变；正常 `EndTurn` 完成与 #351 一致；
- 基线缺失时存在性仍判定，`mustChange` 给出 `file_check_unavailable` warning；
- 图里出现 `criteria` / 运行时清单字段时校验失败并给出可读原因；
- inspector、mock、transfer、codec 与中英文档同步。
