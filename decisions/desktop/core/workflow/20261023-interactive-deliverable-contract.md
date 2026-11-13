---
status: implemented
---

# 交互式节点产出物声明：先做确定性 files 校验，criteria 与运行时清单分阶段

在本决策之前，交互式节点的完成只由本轮终态和输出契约决定，不知道节点「应该产出什么」。本决策给节点增加可选的、随冻结图保存的产出物声明：`agentConfig.completion.files` 声明期望存在（或必须变更）的文件，完成时由后端对照运行 workspace 与节点启动时持久化的基线做确定性校验，缺失或未变更时拒绝完成并列出原因。criteria（自然语言验收项 + 模型判定）与运行时验收清单不在本期：前者需要单独决定模型来源、凭据、成本与降级，后者依赖 agent→host 的声明信号，分别留给后续决策。

本决策为 `implemented`。实现见 [ora-space/desktop#765](https://github.com/ora-space/desktop/pull/765)，Cloud 发布门禁见 [cloud#169](https://github.com/ora-space/cloud/pull/169)（合并提交 [`e35b23c1`](https://github.com/ora-space/cloud/commit/e35b23c1aa18d3abc27ba13e6a65b204718fb732)）。它在冻结图里增加可选字段，并把声明该字段的图提升到 `agentConfig.schemaVersion: 4`；完成路径复用现有 blocker 与 `WorkflowNodeIncomplete`，不新增命令、错误码或数据库迁移。Cloud 的发布门禁拒绝该字段，无声明节点的行为保持不变；`Loop`/`Iteration` 区域与 `agent 自报完成`不在范围内。核心测试用例与证据状态见[交互式节点产出物声明用例](../../../../test-cases/desktop/core/workflow/interactive-deliverable-contract.md)。

## 继承与修改

继承[交互式节点完成契约](20261021-interactive-node-completion.md)（`implemented`），并沿用[节点运行时编排决策](0-node-runtime-orchestration.md)的 per-run 门与失效事件边界；对 [Cloud 调度根决策](../../../cloud/workflow/0-cloud-side-scheduler-owns-workflow-execution.md) D2 的字段去向作出修改（见 D8）。

| 前序约定 | 处理 | 当前规则 |
| --- | --- | --- |
| 完成只认本轮 `EndTurn`，其余终态阻塞 | 继承，不变 | 仍是第一道门禁；后继[首轮确认决策](20261102-first-turn-confirmation.md)在其上增加自动首轮的 `turn_unconfirmed`，本决策不改写该规则 |
| 快照在 park 写入 `payload.completion`，投影只读快照 | 继承，并扩展 | 快照字段与结构不变；文件 blockers 不写快照；基线不可用时 warning 在 park 或完成前的修复中写入既有 `warnings` |
| 已完成快照的 blockers 是权威，`prepare` 只做一次性修复 | 修改 | 文件 blockers 在每次 `prepare` 完成时按现场 workspace 重算，不写入快照；快照 blockers 仍权威，只作用于回合与输出契约 |
| 错误优先级：状态 → turn key → digest → blockers | 继承，不变 | 文件 blockers 汇入第 4 步的 blockers 列表，排在输出契约 blockers 之后 |
| `force` 只覆盖 blockers，二次确认并审计 | 继承，并澄清 | 文件 blockers 可被 `force` 覆盖并留痕；`force` 不能替代后继决策的首轮确认 |
| 无 turn key 的 `turn_unknown` 只能 force | 继承，不变 | 文件校验不改变该规则 |
| 投影 `completion { ready, blockers, warnings }` | 继承，不变 | `ready` 不包含文件判定；后继首轮确认按既有 gate 决定首轮是否 ready |
| 区域 interactive 行为不变 | 继承，不变 | 本期只覆盖外层平图 |
| Cloud 调度根决策 D2：agent 配置首期只执行 `executorSlot`、`roleId`、`prompt`，延后字段由发布门禁按名拒绝；未知字段忽略 | 修改 | `completion` 加入按名拒绝的延后字段，任何非 null 值都拒绝；`agentConfig.schemaVersion` 只执行 `3`，其他出现的版本拒绝，缺省或 null 的旧文档保持原行为（D8） |

本决策早于[首轮确认决策](20261102-first-turn-confirmation.md)。后继决策在完成门禁上增加 `turn_unconfirmed`、异常停靠分类与 `confirmInitialTurn`；本决策的文件判定既不替代也不放宽这些规则。当前完成顺序仍是状态 → turn key → digest → blockers；`confirmInitialTurn` 只解除 `turn_unconfirmed`，不解除文件 blockers；`force` 只覆盖非确认类 blockers，不能把首轮正常结束记成强制完成。因此自动首轮即使以 `EndTurn` 结束且输出契约通过，也不会因本决策变为 ready。异常停靠的跨 crate 证据由[首轮确认用例](../../../../test-cases/desktop/core/workflow/first-turn-confirmation.md#abnormal-parking-is-distinct-from-awaiting-user)维护，本文不重复登记。

## 决策

### D1：声明随冻结图保存，v1 只接受 `files`

`agentConfig` 增加可选对象 `completion`，v1 只允许 `files` 一个子字段：

- `files` 为 1–32 条；每条含 `path`，可含 `mustChange`；
- `path` 非空、不超过 512 UTF-8 字节，必须是 workspace 内的相对路径或受限 glob；字面量部分用 `ora_utils::path::StrictRelativePath` 校验；
- `mustChange` 缺省为 `false`；显式 `null` 不是「未声明」，按类型错误拒绝；
- 绝对路径、`..`、空段与 `.` 段、反斜杠、控制字符、平台保留名、盘符前缀、超长路径段与过深目录都被拒绝；
- 未知子字段直接报图校验错误，不做静默忽略：避免拼写错误导致声明空转；
- `criteria`、`checklist` 在 v1 是**不支持的字段**，出现在图里即校验失败并给出可读原因；它们由后续决策定义后再加入；
- **声明 `completion` 的图必须把 `agentConfig.schemaVersion` 写成 `4`**；`schemaVersion: 3` 同时带 `completion` 是非法组合，解析直接拒绝。desktop 接受 3 与 4，也接受缺省或 `null` 版本的旧图；其他出现的版本拒绝并给出可读原因；
- 声明随图冻结，运行只读冻结图；不改数据库 schema。

选择随图保存，是因为产出去向属于作者的节点设计，必须和 prompt、输出契约一起版本化；运行时可变配置会让同一 run 在不同时刻得出不同判定。

### D2：文件 blockers 在完成时按现场 workspace 判定，不写入快照

`prepare` 组装完成结果时，在门外执行文件校验：

1. 从冻结图取 `completion.files`；
2. 用运行 workspace 的当前文件观测判断「存在」：至少一个声明匹配到 workspace 内的普通文件；
3. 读取节点启动时持久化的 worktree 基线，比较内容；`mustChange: true` 要求至少一个匹配路径在变更集合内；
4. 失败项转成 blockers：`file_missing`、`file_unchanged`，`params.path` 是原始声明（保留作者写下的 glob，不展开成实际路径）。

文件 blockers **不写入 `payload.completion` 快照**；基线不可判定时产生的 warning 例外（见 D4）。workspace 是可变的：用户看到缺失项后可能手动补文件再点完成，如果判定在 park 时就固化，快照会在此时变陈旧，而交互式完成契约的规则是已存快照 blockers 权威、不自动刷新。完成时重新判定让「看到缺失 → 修复 → 重试」直接生效，也避免投影为了显示文件状态去读 workspace。补齐文件后重试同一完成入口即可；如果节点仍停在自动首轮，重试请求同时携带 `confirmInitialTurn`。

文件 blockers 与输出契约 blockers 汇入同一列表，排在状态、turn key、digest 检查之后。`force` 可覆盖文件 blockers 并沿用现有 `completion_forced` 审计；`confirmInitialTurn` 不解除文件 blockers，`force` 也不能替代首轮确认。`completion.ready` 不包含文件判定，用户只有在点完成时才会看到缺失项；这是可接受的，因为缺失项来自一次明确的完成动作，且步骤卡片会用 `WorkflowNodeIncomplete` 的 blockers 与 `warnings` 渲染。

### D3：路径与 glob 语义取最小集合

- 匹配对象是相对运行 workspace 根的 POSIX 路径；
- 支持 `*`（不跨 `/`，可匹配空串）、`**`（必须是完整路径段，跨零个或多个目录）、`?`（匹配一个字符）；
- 不支持 `[]`、`{}`、反斜杠（因此没有转义语法）；只拒绝以 `!` 开头的取反模式，出现在路径中间的 `!` 按字面量处理；`**` 只能作为完整路径段，嵌在其他段内的 `**`（如 `a/**b`、`***`）拒绝；
- 大小写敏感，不做 Unicode 归一化；
- 匹配只认 workspace 内实际存在的普通文件；目录、gitlink 与逃出 workspace 的符号链接都不算匹配；读取前先做 canonical root 的包含性检查，不读取逃逸链接的内容；
- 声明数量和路径长度在图校验时受限，不依赖运行期截断。

最小集合是为了让语义可解释、跨平台一致；复杂 glob 需要单独定义并承担性能与安全面，不在本期。

### D4：基线覆盖声明文件，缺失或不可判定时保守降级

节点启动时持久化的基线不是只含 Git 默认可见文件：它包含普通 Git-visible 快照，并额外捕获被声明但被 Git 忽略、或因无关大文件导致完整快照不可用的声明文件。基线记录自己的观测范围：

- **完整基线**同时支持文件门禁与完整 `file_changes` 展示；
- **仅声明基线**只覆盖 `completion.files`，足以判定这些声明的存在与变更，但**不能**用来生成完整 `file_changes`；否则未纳入基线的普通文件会被伪造成新增；无法证明覆盖范围的旧版本化缓存按仅声明范围处理，不带版本的旧纯 map 仍按已知的 Git-visible 全量范围处理；
- 自动（非交互）节点只使用普通 Git-visible 基线，不因图上残留的 `completion` 声明改变自己的文件差异。

基线可能缺失或损坏；升级前的旧缓存也可能缺少证明覆盖范围或旧二进制内容的信息。存在性只来自当前 workspace 的观测，基线只参与变更比较。只有确实无法判定的声明才降级：

- 存在性校验仍执行，缺失时仍是 `file_missing`；
- 基线缺失/损坏时，该 `mustChange` 声明无法判定；旧基线把二进制文件的旧内容存成 `null`、当前仍是二进制时也无法证明。两种情况都记录 warning `file_check_unavailable`，不作为 blocker；
- 基线缺失/损坏在 park 时已确定的，warning 随 park 快照与状态迁移同一次原子提交写入 `payload.completion.warnings`，投影可见；park 后才损坏、或完成前才发现不可解析的，`prepare` 使用既有 guarded `Pending` 快照修复改善提前投影；该修复是 best-effort，失败只记录日志，不影响最终完成；
- 旧基线中可读的文本等已知内容仍按内容正常比较；旧 `null` 二进制不伪造成新增，只有无法证明的项才产生 warning；
- 文本缓存与流式摘要表示同一份字节时按内容摘要比较，不因缓存预算或表示形式把未变更文件判成变更。

park 时已知事实的 warning 属于 park 快照，与 `Running → Pending` 同一次原子提交。完成前才发现的 warning，`prepare` 仍用既有 guarded `Pending` 快照修复改善提前投影；这次修复是 best-effort，失败只记录日志，并只影响提前投影。准备结果在存在文件 warning 或 force 审计时携带对应 payload，最终完成事务将其与完成状态一起提交；因此完成后的 warning 不依赖 prepare 修复是否成功，最终事务写入失败时按既有错误返回、不报告成功。文件 blockers 不在完成快照中，不会被这条补丁固化；该机制不新增 DTO、命令或数据库 schema。缓存预算只影响基线/观测的表示与覆盖范围（内联文本或流式摘要、完整或仅声明），不削弱 `files` 的存在性检查与 `mustChange` 判定；一个无关超大文件也不能让已存在的声明文件变成缺失，或让可用的变更基线失效。这沿用现有 `completion_file_changes` 的保守行为：没有可靠基线时报告没有变更，而不是伪造失败或伪造变更。把不可判定当阻塞会让升级前节点无法完成，属于误伤。

### D5：criteria 与运行时清单分阶段

v1 不实现 `criteria` 的模型判定，也不实现运行时验收清单：

- criteria 需要一次模型调用。desktop 没有直接的模型通道，现有先例是失败诊断：用节点自己的 executor 起一次 one-shot agent session 再解析回答。judge 必须决定模型来源（节点本身还是独立的 judge 配置）、凭据、计费、人工同意、超时与失败降级，并明确不能放进完成事务或 per-run 门内。这些是产品决策，另开 ADR；本期 files 校验是它的前置条件。
- 运行时清单要求 agent 在首轮声明交付物清单、人在卡片上确认、结果存进 node run payload；它依赖 agent→host 的声明信号，与完成请求信号同源，由后续决策承接。
- 因此 v1 在图校验层直接拒绝这两个字段，而不是接受后静默忽略。

### D6：契约与 UI 复用现有形状

- `WorkflowNodeCompletionBlockerKind` 增加 `FileMissing`、`FileUnchanged`；`WorkflowNodeIncomplete { node_id, blockers[] }` 不变，blocker 的 `params.path` 携带声明的路径，前端按 kind 本地化；
- 错误传输 DTO 为 blocker 增加可选 `params.path`；原有回合 blocker 保留 `message`，无路径时不发送 `params`。不新增命令、错误码或完成投影字段；`force` 仍覆盖文件等非确认类 blockers 并写审计；
- inspector 增加 files 列表编辑（路径 + `mustChange` 开关），声明第一条时把版本写为 4，删到空时移除声明并回到版本 3；中英文案同时提供，非法路径就地提示；
- 步骤卡片渲染 `file_missing` / `file_unchanged` 与 `file_check_unavailable` warning；
- `workflow-mock` 的图校验、`workflow-transfer` 的导入校验与 graph codec 往返同步覆盖新字段，非法原值保留到校验器而不是在规范化时被丢弃。

### D7：兼容性与版本升级

- `agentConfig.schemaVersion` 提升到 4：声明了 `completion` 的图必须写 4；desktop 解析接受 3/4，也接受缺省版本的旧图；小于 3、大于 4、非整数版本拒绝；`3` 携带 `completion` 视为非法组合并拒绝。
- Rust 的 `WireAgentConfig` 增加可选 `completion` 与版本校验；旧图（版本 3、无 `completion`）行为不变。
- TypeScript 的 `WorkflowAgentConfig` 与 `normalizeWorkflowAgentConfig` 显式规范化 `completion` 与版本，不依赖 `...config` 透传；`workflow-mock` 的校验同步接受 3/4。
- 不改数据库 schema，不迁移既有 run。
- **旧版 desktop 的残留风险**：已经发布的客户端不认识版本 4，会忽略版本并丢弃 `completion`，节点按旧行为完成。这一点无法回溯修复，缓解措施是：新 desktop 拒绝「3 + completion」的非法组合；Cloud 发布门禁拒绝 `completion` 与版本 4（D8）；发布说明要求使用支持该字段的 desktop 版本。旧编辑器保存会丢弃 `completion`，版本号按旧客户端的写法保存；保存后的图不再含声明，不会被误认为已通过校验，作者需要在新编辑器里重新声明。

### D8：Cloud 发布门禁拒绝该字段，规范 schema 记为 desktop-only

- Cloud 首期的可执行子集不含 interactive 完成语义，`agentConfig.completion` 在 Cloud 没有对应判定。发布门禁**拒绝任何非 null 的 `completion`**，拒绝问题点名字段 `completion`，不静默剥离；`completion: null` 与缺省视为没有声明，保持旧文档行为。
- `agentConfig.schemaVersion` 只执行数字 `3`：`4`、`1`、`2`、字符串或其他出现值都拒绝，问题点名 `agentConfig.schemaVersion`；缺省或 `null` 是版本检查引入前的旧文档，保留原行为。版本检查先于字段清单，因此同时带版本 4 与 `completion` 的图会在同一问题列表中先列版本问题、再列字段问题，两者都会返回。
- 资产导入与发布共用同一个图校验门禁，两个入口得到相同问题码与节点/字段定位，且都不产生快照。
- 规范 schema（由 specs 拥有，见 cloud 调度根决策 D2）在 agent 配置字段去向里记录：`completion` 是 desktop-only 字段，与 `schemaVersion: 4` 绑定；Cloud 若要支持，需要自己的后续决策并同步门禁与字段表。
- 这是对 Cloud 调度根决策 D2 字段去向的修改，记录在本文「继承与修改」，根决策标题下以历史决策注明；根决策正文与状态不变。
- Cloud 的明确拒绝门禁必须先于开放该字段的编辑器上线；本期编辑器仍只为 Desktop 保存该声明，Cloud 支持其执行需要后续决策。Cloud 侧已验证的实现与证据见[发布与启动门禁用例](../../../../test-cases/cloud/workflow/publish-and-start-admission.md#publishing-refuses-an-unsupported-node-or-configuration-and-names-it)。

## 实现与 approved 方案的差异

以下边界在 approved 方案中没有展开或表述不同，按最终实现记录：

- `mustChange` 缺省为 `false`，但显式 `null` 被拒绝；`approved` 方案只规定缺省值，最终实现把显式错误值与省略区分开。
- 节点启动基线扩展为「普通快照 + 声明文件的定向捕获」，并记录完整/仅声明范围；仅声明基线不驱动完整 `file_changes`。`approved` 方案只要求基线存在。
- `file_check_unavailable` 的写入时机从「park 时」扩展为「park 时已知的原子写入；完成前才发现的先由 `prepare` 做 best-effort 提前投影修复，最终 warning 快照与 force 审计随完成事务提交」。旧基线中无法证明的二进制内容也在完成检查时降级为该 warning。
- Cloud 的版本兼容按实际实现明确：缺省/`null` 版本保留旧文档行为，只执行数字 `3`，其余出现值都拒绝。
- `force` 与后继首轮确认的关系由[首轮确认决策](20261102-first-turn-confirmation.md)扩展：文件 blockers 仍可 force，但存在 `turn_unconfirmed` 时 force 不能替代确认；本决策不覆盖首轮规则。
- 路径校验还拒绝 `.`/空段、控制字符、平台保留名、盘符前缀、过深目录与超长段；这是 `StrictRelativePath` 的既有边界，`approved` 方案只举了绝对路径、`..`、空段与反斜杠。

## 不变量

1. 只有声明了 `completion.files` 的节点才做文件校验；无声明节点行为与既有完成契约完全一致。
2. 文件判定只在完成时按现场 workspace 执行，不写入快照，不读会话历史。
3. 文件 blockers 走第 4 步的 blockers 门禁；`force` 可覆盖文件等非确认类 blocker 并审计，但不能替代首轮确认。
4. 路径必须落在运行 workspace 内，拒绝绝对路径、`..` 与逃出 workspace 的符号链接；目录与 gitlink 不算文件。
5. 基线缺失或损坏时存在性仍判定，`mustChange` 只产生 `file_check_unavailable` warning，不阻塞完成。
6. 旧基线中不可证明的二进制内容不产生虚假变更，也不产生虚假的「未变更」结论。
7. 基线观测范围被区分：仅声明基线不生成完整 `file_changes`。
8. v1 拒绝 `completion.criteria` 与运行时清单字段，不静默忽略。
9. 声明 `completion` 的图是 `agentConfig.schemaVersion: 4`；3 与 4 混用、以及其他不支持的版本都被拒绝。
10. Cloud 发布门禁拒绝非 null `completion` 与任何出现的非 3 版本；缺省/null 的旧文档保持原行为，带声明的图不会在 Cloud 上不经检查就完成。
11. 文件声明与缓存预算不互相削弱：无关大文件、ignored 文件、文本/摘要表示差异都不能改变 `files` 或 `mustChange` 的真实结论。
12. 不新增命令、错误码、投影字段或数据库迁移；版本提升只改变图的解析契约。
13. 文件 warning 与 force 审计随最终完成事务提交；`prepare` 修复只改善提前投影，完成后的可见性不依赖它，最终事务写入失败时按既有错误返回，不报告成功。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 在 park 时算文件 blockers 并写快照 | 用户补文件后快照陈旧，要么违反快照权威规则，要么引入文件专属刷新规则；完成时判定更简单且行为可解释 |
| 用 LLM 判定文件是否存在 | 存在性是确定事实，模型只增加成本与不稳定 |
| 直接接 model-gateway 做 criteria | desktop 没有该通道，且模型来源、凭据与计费需要单独决策 |
| 每次完成默认跑 judge | 成本与误判高，且无法离线使用 |
| 支持完整 glob（`{}`、`[]`、取反、转义） | 语义、性能与跨平台一致性的面都变大，先用最小集合 |
| 不 bump `schemaVersion`，只靠字段可选 | 声明会被旧 desktop 静默丢弃，节点不经检查就完成，与 D1「不静默空转」矛盾；bump 让新读者能识别版本并拒绝非法组合 |
| 在投影里实时读 workspace 显示文件状态 | 轮询成本高，且读取投影保持纯 payload 是现有契约 |
| 只捕获普通 Git-visible 文件作为基线 | 声明 ignored 输出时永远判为缺失，或把预先存在的 ignored 文件伪造成新增；定向捕获声明文件才让 `mustChange` 有意义 |
| 用 `git status` 生成文件变更 | 会漏掉未跟踪目录中的文件与已完成文件的内容比较，也让“没有基线”与“空工作区”混淆 |

## 风险、未解决问题与验收

- **投影不显示文件缺失**：`ready` 不包含文件判定，只反映回合/输出契约与后继首轮确认门禁；文件缺失在完成被拒时返回，由 inspector 文案与步骤卡片说明，避免用户以为按钮可用就一定能完成。
- **判定与提交之间文件可能变化**：文件校验在 `prepare` 门外执行，与现有 turn key/digest 比对一致；提交阶段不复核文件，提交前用户改动文件会让这次判定短暂过期，重试即可。
- **glob 语义有限**：复杂匹配后续扩展，扩展时保持 v1 语义不变。
- **跨平台路径**：大小写敏感、POSIX 分隔符；Windows 上大小写不同的路径不会匹配，需在文档写明。
- **旧客户端的残留风险**：旧 desktop 不识别版本 4，会静默丢弃 `completion`；由 Cloud 门禁拒绝、新 desktop 拒绝非法组合、发布说明三层缓解，但无法回溯修复。
- **基线不可判定**：缺失、损坏或旧二进制内容无法证明时，`mustChange` 降级为 warning，无法保证「必须变更」；只声明基线也不提供完整文件差异。
- **不在本决策范围**：criteria、judge、运行时清单、agent 自报完成、区域 interactive。首轮确认与异常停靠由后继决策维护。

验收：

- 声明 `files` 的节点：缺失或未变更时完成返回 `WorkflowNodeIncomplete` 并列出 `file_missing` / `file_unchanged` 与原始声明；补齐后重试可完成；`force` 仍可覆盖文件 blockers 并审计，首轮停靠时确认只解除 `turn_unconfirmed`；
- 无声明节点行为不变；正常完成与既有完成契约一致；
- 基线缺失或损坏时存在性仍判定，`mustChange` 在快照 `warnings` 里给出 `file_check_unavailable`；旧二进制内容不伪造变更；
- 声明 ignored 或大文件时基线覆盖它们，无关大文件或仅声明基线不伪造 `file_changes`，也不削弱 `mustChange`；
- 图里出现 `criteria` / 清单字段时校验失败并给出可读原因；
- `schemaVersion: 3` 携带 `completion`、或版本不在 desktop 支持范围，解析拒绝并给出可读原因；
- Cloud 发布带非 null `completion` 或非 3 版本的图被拒绝，原因点名节点与字段；资产导入同样拒绝；版本 3 且无声明照常发布；
- inspector、mock、transfer、codec 与中英文档同步。
