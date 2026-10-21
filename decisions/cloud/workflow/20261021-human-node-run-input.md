---
status: proposed
---

# 人工节点以运行级结构化输入继续：有界等待、追加式输入行，与 agent 会话解耦

本文是[工作流执行根决策](0-cloud-side-scheduler-owns-workflow-execution.md)（`approved`）的后续决策，
承接其 D4 留下的「后续的交互式运行控制决策」。本文只固定首期 HITL 中 **`human` 节点**的契约：
运行进入 `awaiting_input` 后，由发起人或租户管理员通过运行级输入接口提交一份结构化输入；输入先追加
落库，调度器在下一次重算中消费它、把该节点结算为 `succeeded`，运行回到 `running` 继续推进。
等待必须有界（新配置 `workflow.hitl_wait_timeout`，默认 24 小时），取消或超时都进入终态，不允许运行
永久停在 `awaiting_input`。`human` 节点不创建 agent 会话，也不经过 Controller，因此单回合会话不变量
不变。interactive Agent 不在本期范围，等 specs#177 与 #351 收敛后另定。

**状态与影响面**：本文当前为 `proposed`。owner 已在
[ora-space/circus#449](https://github.com/ora-space/circus/issues/449) 拍板选 B：先定契约 ADR，暂不实现；
九项默认值全部按建议。本文评审通过前不维护核心用例；变为 `approved` 后，在同一运行内按最终决策补
`test-cases/cloud/workflow/`。**本期产品行为不变**：发布门禁继续拒绝 `human` 节点和
`interactive: true`，`workflow_runs.status` 的 CHECK 里保留 `awaiting_input` 作为没有写入者的保留值；
公开 OpenAPI 与前端按 #449 的结论移除该状态（运行 ora-space/circus#450），实现本文时再由引入写入者的
同一变更把它加回公开枚举。**兼容边界**：新增输入接口、输入行与运行上的等待列；不改图 schema 的既有
节点、不改 Controller 契约、不改取消接口已有的响应与错误码。**非目标**：不实现 HITL、不放开发布门禁、
不改单回合 agent 会话、不定义指定处理人/通知渠道、不定义 interactive Agent 的后续回合。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D1：执行归属归 Cloud 侧调度器，节点终态只来自已提交状态 | 继承，不变：等待进入与退出都由调度器在重算里提交，输入请求只追加一条输入行 |
| 根决策 D2：规范图 schema 由 specs 拥有；`human` 首期被发布门禁拒绝 | 继承门禁结论，扩展 schema：`human` 节点声明 `humanConfig.inputSchema`，并声明 `output` 变量根；实现本文时发布门禁只放行 `human`，仍拒绝 `interactive: true` |
| 根决策 D4：HITL（`human`、`awaiting_input`）延后，去向「后续的交互式运行控制决策」；当时把 HITL 描述为需要 Thread 的对话式交互 | **承接并修改**：本文就是该后续决策，但 `human` 节点不再走 Thread 对话，改为运行级结构化输入；需要后续人类回合的 interactive Agent 仍延后 |
| 根决策 D4/D6：取消是请求加有界终态，取消写入者纳入首期 | 继承并扩展：实现本文后 `awaiting_input` 也接受取消；已记录的取消请求优先于未消费的输入与等待超时 |
| 根决策 D7：首期只保证资源释放 | 继承：等待不是终态，不释放运行 Workspace；取消或超时进入终态后走既有释放路径，等待上限同时是 Workspace 保留上限 |
| 根决策 D8：不新增 Controller 契约 | 继承，不变：人工输入不经过 Node/Controller，不新增工作项、claim 或回执字段 |
| 根决策 D3：WorkflowRun 是执行聚合，IssueRun 只是调用记录 | 继承：输入行归属 WorkflowRun；IssueRun 只获得一条 Issue 活动，不是线程回合 |
| [会话结束后续决策](20261019-workflow-session-end-after-initial-turn.md) D1/D5：工作流 agent 会话是单回合的，节点终态仍只来自会话结果 | 继承，不变：`human` 节点不创建 agent 会话，因此不需要后续回合；interactive Agent 才会改变会话生命周期，不在本文 |
| 取消证据（运行 #347、cloud#120）与 S5 释放状态机（cloud#119） | 继承：取消路径继续按 `cancelled` 结算节点与运行，释放事实按既有 `workspace_release_*` 列登记 |

## 决策

### D1：交互通道是运行级输入接口加运行页

**规则**：输入走 `POST /api/v1/tenants/:tid/workflows/:wfid/runs/:rid/inputs`，请求体是
`{"nodeId": "<图节点 id>", "input": { ... }}`，`nodeId` 与 `input` 都必填；请求必须带平台既有的
`Idempotency-Key` 头（1–200 字符）。接口只追加一条输入行并返回 202，响应体是
`{"resource": {"id", "runId", "nodeId", "actorId", "input", "recordedAt"}}`。运行页用同一个接口提交，
不另开写入路径。

**为什么是运行级接口而不是 Issue 线程回复**：Issue 线程回复只覆盖 IssueRun，直接运行的 WorkflowRun
没有 Issue 可回；它还会把运行输入混进讨论时间线，让「讨论」和「运行控制」共用一种写入。运行级接口
同时服务直接运行和 IssueRun，权限与状态都以运行为准。**为什么不是专用对话会话**：专用会话需要引入
Thread、轮次与跨回合的会话生命周期，正是 interactive Agent 才需要的东西；`human` 节点只需要一次结构化
输入，用会话承载会把首期范围重新放大。

**为什么返回 202 而不是直接返回节点终态**：请求只完成「记录输入」；消费输入、结算节点与推进运行由
调度器在下一轮重算完成（D2）。这与取消接口「记录请求、调度器反应」的分工一致，也保证请求事务不携带
调度推进。

### D2：等待写入 `awaiting_input`，输入被调度器消费后回到 `running`

**规则**：运行上的等待事实由两个新列共同承载：`awaiting_input_at timestamptz` 与
`awaiting_input_node_id text`，两者同时写入、同时清空，CHECK 保证同为空或同非空。`human` 节点成为
就绪节点时，调度器在同一个事务里：

1. 把该节点在 `node_states` 里置为 `running`（节点状态词表不新增值，等待是运行级状态）；
2. 写入 `awaiting_input_at = now()`（数据库时钟）与 `awaiting_input_node_id`；
3. 把运行 `status` 置为 `awaiting_input`，版本 +1，并为关联的 IssueRun 记一条 Issue 活动（D8）。

输入行提交后，下一次调度重算在同一事务里消费它：把该节点置为 `succeeded`、`output` 为通过校验的输入
对象、写 `finished_at`，清空两个等待列，把运行放回 `running` 并继续按图推进；如果这是最后一个节点，
同一次推进直接进入 `succeeded`。**HTTP 请求不写运行状态、不写 `node_states`、不推进图**；它只追加
D6 的输入行。调度器崩溃后重启，下一轮重算读到的仍是同一条已提交输入，消费一次且只消费一次。

**状态转移**（实现本文后）：

| 起始 | 事件 | 结果 |
|---|---|---|
| `running` | `human` 节点就绪 | `awaiting_input`，等待列成对写入 |
| `awaiting_input` | 输入行被消费 | `running`（或图结束时的 `succeeded`） |
| `awaiting_input` | 取消请求 | `cancelled`（D4） |
| `awaiting_input` | 没有输入且超过等待上限 | `failed`（D3） |

**禁止**：在输入请求事务里直接把节点记为成功；在 `awaiting_input` 下派发其他节点；同一运行同时有两个
等待列指向不同节点；等待期间运行被投影成终态。

### D3：等待有界，默认 24 小时，到期按失败结算

**规则**：新增配置 `workflow.hitl_wait_timeout`，默认 **24 小时**，按部署配置，本期不做逐运行覆盖。
窗口从 `awaiting_input_at` 起算，用数据库时钟比较，与调度周期、进程时钟和副本数量无关。窗口到期且
该等待仍没有被消费的输入行时：

- 该 `human` 节点以命名原因 `hitl_wait_timeout` 失败，节点 `error` 记录同名原因；
- 运行进入 `failed`，`error` 记录 `human input wait timed out`；
- 运行 Workspace 按根决策 D7 的既有终态收尾声明删除，释放路径与失败运行相同。

**已接受的输入优先于超时**：只要在窗口内提交了输入行，即使调度器在窗口之后才重算，也消费输入而不是
判超时；超时只在该等待没有任何已接受输入时成立。**取消优先于超时**：已记录的取消请求按 D4 结算
`cancelled`，不再写超时失败。

**为什么必须有界**：等待不是终态，运行 Workspace（以及运行本身）会被保留；没有上限就会重演「永久
`running`」的另一条形态。24 小时的依据：覆盖跨时区的一次人工审核（一个工作日加缓冲），同时把
Workspace 保留限制在一天量级；更长或更短的需求由部署配置调整，不把具体数字写进实现。**为什么按失败
而不是跳过**：跳过会把「没有人处理」伪装成「已批准」，节点后续分支可能因此执行错误路径；超时按失败
结算，让图和运行都停在可解释的终态。

### D4：写入权限与取消一致：发起人加租户管理员

**规则**：输入请求的授权谓词与取消完全相同：运行带冻结模型绑定时，只有该运行的发起人
（`created_by`）与租户的当前管理员可以提交，其他活跃成员只读并得到 `403 workflow_run_input_forbidden`；
运行没有冻结绑定时沿用 Issue 线程的成员规则（实际创建路径总会冻结默认槽位的绑定）。权限从当前
PostgreSQL 状态读取，不信任请求里的 actor 字段。授权检查先于状态、节点匹配与输入校验，未授权的调用者
不能从 409/400 的差异推断运行是否在等待。

**取消请求也接受 `awaiting_input`**：实现本文后，`awaiting_input` 进入可取消状态集合。取消请求记录后，
调度器在下一次重算把等待中的 `human` 节点结算为 `cancelled`（没有会话可停止），运行结算 `cancelled`，
未消费的输入行保留为历史，但**不能把运行从终态复活**。取消请求与输入请求的竞争以数据库提交顺序为准：
取消先提交则之后的输入得到 `409 run_not_awaiting_input`；输入先提交但尚未消费时取消到达，取消仍决定
运行终态，该输入不产生节点成功。

**为什么与取消一致**：两者都是对运行的控制写入，都是发起人与管理员的责任边界；如果输入允许任意成员
写入，取消却只允许发起人/管理员，权限模型就会出现一个更弱的旁路。指定处理人留到后续决策，不影响
本谓词。

### D5：节点声明 JSON Schema，发布期只接受受支持的子集

**规则**：`human` 节点的配置为 `humanConfig.inputSchema`，是一个 JSON Schema 对象；`data.instruction`
仍是展示给处理人的提示文本。输入在写入前按 schema 校验；发布门禁在实现本文后同时校验 schema 本身。

**支持的子集**（闭合集合，不静默忽略未知关键字）：

- 顶层与每个嵌套对象都必须是 `"type": "object"`，且必须显式写 `"additionalProperties": false`；
- 对象关键字：`type`、`properties`、`required`、`additionalProperties`（只允许 `false`）；
- 数组关键字：`type: "array"`、`items`（一个受支持的子 schema）；
- 标量：`type` 为 `string` / `number` / `integer` / `boolean`，可用 `enum` 限定取值；
- `required` 必须是 `properties` 中已声明属性名的数组。

出现其他关键字（例如 `oneOf`、`$ref`、`patternProperties`、`format`、`minLength`）时，发布门禁以
`human_schema_unsupported` 拒绝并指出节点与字段；schema 本身不是对象、顶层不是 object、缺少
`additionalProperties: false`、`required` 指向未声明属性时，以 `human_config_invalid` 拒绝。这样门禁
不会接受一份「看起来有约束、实际被实现忽略」的 schema。

**输入期校验**：请求体中的 `input` 必须是通过 schema 的 JSON 对象；未知字段、缺失必填、类型不符、
`enum` 不匹配都拒绝，返回 `400 workflow_input_invalid`，响应体含
`{"problems": [{"pointer": "<相对 input 的路径>", "reason": "<required|type|unknown_field|enum>"}]}`，
按 pointer 升序排列。`input` 序列化后的 UTF-8 字节数上限为 **64 KiB**，超过时返回
`400 content_too_large`，不进入 schema 求值；这与 agent Thread 单条内容的上限一致。

**为什么用 JSON Schema 而不是自由文本**：自由文本无法在写入时判断「输入是否满足节点承诺」，只能把
校验推迟到下游节点运行时，错误会出现在离输入很远的地方。受支持子集覆盖首期需要的必填、类型、未知
字段与大小限制，剩余关键字在发布期明确拒绝，不制造「配置了但没生效」的假象。

### D6：输入是追加式行，`node_states` 只存最终结果

**规则**：新增追加式表 `workflow_run_inputs`，每行至少包含：`id`、`tenant_id`、`workflow_id`、
`workflow_run_id`、`node_id`、`actor_id`、`idempotency_key`、`input jsonb`、`recorded_at timestamptz`。
表只插入、不更新、不删除；一个（运行，节点）的等待至多接受一行输入（首期 DAG 每个节点至多执行一次），
唯一约束 `(workflow_run_id, node_id)` 固定这一点。每次等待最多只有一行，后续不同幂等键的提交返回
`409 workflow_input_already_recorded`；同一个 `Idempotency-Key` 加同一请求由平台既有幂等记录返回原响应，
不产生第二行。运行页与详情读取的是输入行与 `node_states` 的组合：历史与 actor 来自输入行，节点的最终
输出只来自 `node_states[nodeId].output`。

**为什么追加而不是覆盖**：输入是「谁在什么时候给出了什么」的审计事实，覆盖会丢失 actor 与时刻，也让
幂等重放无法判定。**为什么 `node_states` 不存历史**：`node_states` 是运行推进的状态投影，节点终态必须
只有一个来源；把输入历史塞进去会让同一事实出现两份，并在恢复重算时产生分叉。

### D7：首期只做 `human` 节点，人工节点不创建 agent 会话

**规则**：本契约只覆盖 `human` 节点。 `human` 节点不派发 Node、不创建 AgentSession、不占用执行器槽位
或模型绑定、不写 `thread_commands`；它只是调度器图推进中的一个等待点，输入就是它的输出。因此本文不需要
改变单回合会话不变量，也不需要 Controller 契约。实现本文时，发布门禁只放行 `human`，`interactive: true`
的 agent 节点继续拒绝。

**interactive Agent 不在本期**：它需要同一会话的后续人类回合、desktop 的完成契约（specs#177）与
#351/#426/#427 的收敛；这些依赖未定时实现会返工。interactive Agent 将来若需要改变「单回合、Cloud
主动结束」的生命周期，必须另开决策，本文不预设其形状。

### D8：等待时发 Issue 活动，不是线程回合

**规则**：WorkflowRun 关联了 workflow IssueRun 时，进入等待的同一次事务里追加一条 Issue 系统活动，
动作名 `run.awaiting_input`，正文包含节点 id、节点显示名与等待截止时间（`awaiting_input_at` 加配置
窗口）。输入被消费、运行回到 `running` 时，不新开线程回合、不写 `thread_commands`、不改变 IssueRun 的
`status`；IssueRun 继续按既有投影保持 `running`。直接运行（没有 IssueRun）只在运行页展示等待与截止时间。

**为什么发活动**：Issue 是参与者实际查看的地方；等待输入时没有活动，参与者只能靠轮询运行页才知道有人
在等。**为什么不算线程回合**：线程回合属于 agent 会话，`human` 节点没有会话；把输入记成回合会伪造一条
不存在的人类对话，破坏 Thread 的语义与统计。

### D9：公开契约现在不含 `awaiting_input`，实现写入者时再加回

**规则**：按 #449 的结论（由运行 #450 承接），首期公开 OpenAPI、前端状态类型与文案不包含
`awaiting_input`；`workflow_runs.status` 的 CHECK 保留该值作为保留值，取消接口继续按不可取消处理。
实现本文的那次变更在引入第一个写入者的同时，把 `awaiting_input` 加回公开枚举、前端状态类型与运行页
文案，并按本 ADR D1/D2 的接口契约发布。存储 CHECK 不提前删除，避免实现前后需要第二次迁移。

**为什么现在移除**：没有写入者的公开枚举会被使用者读成「已支持」；契约应只声明实现能兑现的状态。
**为什么实现时再加回**：状态枚举与写入者必须在同一次变更里出现，否则又回到「枚举存在但没人写」的
误导状态。

## 不变量

1. 输入只在运行处于 `awaiting_input`、且请求的 `nodeId` 等于 `awaiting_input_node_id` 时被接受；一个等待
   至多接受一行输入。
2. 等待不能永久存在：每个等待要么被输入消费、要么被取消请求、要么被有界超时终结。
3. 输入请求只追加输入行，不写运行状态、不写 `node_states`、不推进图；节点结算只由调度器提交。
4. 已提交的输入不会被超时覆盖；已记录的取消请求不会被未消费的输入或节点结果复活。
5. 输入行不可变；`node_states` 只存节点最终结果与终态时间，不存输入历史。
6. `human` 节点不创建 AgentSession、不写 `thread_commands`、不占用执行器槽位与模型绑定，单回合会话
   不变量不变。
7. 输入权限与取消一致，从当前数据库状态判断；未授权调用者不能从错误差异推断运行状态。
8. 公开契约只有在存在写入者时才声明 `awaiting_input`；存储保留值不算公开支持。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| Issue 线程回复作为输入通道 | 直接运行的 WorkflowRun 没有 Issue；IssueRun 的讨论与运行控制混在同一时间线；权限仍需按运行重算，等于多一层映射 |
| 专用对话会话承载 `human` 节点输入 | 需要引入 Thread、轮次与会话生命周期，正是本期明确延后的 interactive Agent 能力；单次结构化输入不需要对话承载 |
| 不写 `awaiting_input`，从节点行推导等待状态 | Cloud 的列表、筛选与配额读取以运行状态列为准；推导会让每个读取点重复实现同一判断，也无法在运行行上表达等待截止时间 |
| 无上限等待，只能人工取消 | 运行与 Workspace 会被无限期保留；没有人处理时系统没有可解释的终态，重演「永久 `running`」的缺陷 |
| 超时后跳过该节点继续执行 | 把「没有人处理」伪装成「已批准」，下游分支可能基于不存在的输入执行；失败是可解释的终态，跳过不是 |
| 直接覆盖 `node_states` 保存最新输入 | 丢失 actor、时刻与幂等证据，无法审计也无法判断重复提交；恢复重算时同一事实有两个来源 |
| 只收自由文本，不声明 schema | 无法在写入时校验节点承诺，错误推迟到下游；调用方也无法从契约知道需要什么字段 |
| 首期同时实现 `human` 与 interactive Agent | interactive Agent 依赖 specs#177、#351、#426、#427；在完成契约与缺陷未收敛前实现，必然返工，且会改动单回合会话不变量 |
| 让 `human` 节点创建一条 agent 会话等待输入 | `human` 节点没有 agent 任务、模型调用与 Workspace 产出，创建会话只会引入无意义的执行与收尾路径，并把 HITL 与 agent 会话生命周期绑死 |

## 本决策未解决的问题

- **运行页如何渲染 schema**：本期只定“提交与 API 相同形状的 JSON 对象”；把受支持子集渲染成表单控件
  （字段顺序、必填提示、数组编辑）属于产品实现，由实现运行在 UI 侧决定，不改变本契约。
- **指定处理人与通知渠道**：本期只支持发起人与租户管理员；指定处理人、邮件/IM 通知留给后续决策。
- **interactive Agent 的后续回合与完成契约**：依赖 specs#177、#351、#426、#427，另开决策；本文不定义
  同一会话多回合的传输与结束语义。
- **逐运行覆盖等待上限**：本期只有部署级配置；如果将来需要按工作流或节点覆盖，另开决策，不在本文预设
  字段。
- **等待期间的 Workspace 复用策略**：本期保持运行 Workspace 存活到终态（D3 的上限约束保留时长）；
  是否在等待时挂起或重建 Workspace 属于资源优化，另行评估。
- **重试/循环导致同一节点再次等待**：首期 DAG 每个节点至多执行一次，`(workflow_run_id, node_id)` 唯一；
  将来支持重试/循环时，输入行需要引入 attempt/等待代次键，届时另开决策。

## 落地顺序

- **阶段 0（本文）**：以 `proposed` 提交 specs PR，经 reviewer 评审、owner 批准后改为 `approved`，并在同一
  运行内补 `test-cases/cloud/workflow/hitl-run-input.md`，覆盖等待进入、输入校验、幂等、权限、取消竞争、
  超时、Issue 活动与恢复重算。完成条件：ADR 与核心用例合并；产品行为不变。
- **阶段 1（实现，后续运行）**：新增迁移（`awaiting_input_at`、`awaiting_input_node_id`、
  `workflow_run_inputs`）、输入接口与校验、调度器等待/消费/超时、取消对 `awaiting_input` 的扩展、Issue
  活动、运行页表单与文案、发布门禁放行 `human`（仍拒绝 `interactive: true`）。完成条件：核心用例在
  cloud 侧有直接测试证据；ADR 改为 `implemented` 并同步实际实现的偏离。
- **阶段 2（interactive Agent，后续决策）**：在 specs#177 与 #351 收敛后，定义同一会话的后续人类回合、
  人工完成与完成声明；届时才需要修改单回合会话不变量。
- **保持不变**：首期节点白名单（除新增 `human`）、agent 会话的单回合生命周期、Controller 线上契约、
  取消与释放的既有结算、IssueRun 与 WorkflowRun 的投影关系。
