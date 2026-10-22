---
status: proposed
---

# Cloud Scheduled Run 的 verdict 由执行端结构化上报，产物在 Revision 之外按运行登记并受独立上限约束

本文承接 [Cloud Scheduled Task 根决策](0-cloud-owned-scheduled-prompt-runs.md) 与
[local 结果注解/产物 ADR](../../desktop/core/scheduled-task/20261021-run-result-annotation-and-artifacts.md)
留给 Cloud 的契约：定时运行如何携带机器可读的业务 verdict，以及运行产物在一次性运行 Workspace
释放后如何存储、索引、访问、保留和计费。Cloud 运行由 IssueRun / Thread 承载，`IssueRun ≠ ScheduledRun`；
本文不新增 IssueRun 阶段，也不改变 Thread 会话模型与 Revision 交付语义，只扩展 `delivering` 的收敛项与
Node 终态上报字段。

本 ADR 处于 `proposed`。它只冻结 Cloud 侧契约，不实现产品代码、真实栈端到端验收、验收开关、发布或
迁移执行；获批后如需实现，另开 Cloud 实现事项与运行单。按 specs 规则，核心测试用例在 ADR 转为
`approved` 后建立，本文暂不维护。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| [Cloud Scheduled Task 根决策](0-cloud-owned-scheduled-prompt-runs.md) D1/D3：每次触发在绑定 issue 下新建 agent IssueRun，复用既有编排；任务历史投影 IssueRun 事实 | 继承，不变：verdict 与产物只附着于定时触发的 IssueRun，不新增第二套运行入口 |
| 同上 D4/D6/D7：60 分钟截止、失败/超时语义、任务历史每任务 100 条且只裁剪历史视图、IssueRun/Thread/Revision 审计事实不被历史裁剪删除 | 继承，不变：100 条裁剪仍只作用于任务历史视图；verdict 是运行事实，产物按本文 D5 单独清理 |
| 同上 D8：Cloud 执行位置在验收前保持禁用 | 继承，不变：本文不解除 Desktop `cloud` 选项 |
| [Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md) D2：Cloud 不解析消息和工具的业务字段，只校验 JSON/`kind`/大小与 `TurnEnded.stopReason` | 继承：Cloud 仍不解析 Thread 内容；verdict 哨兵由 Node 执行端在最终 Assistant 轮解析后，以结构化字段上报 |
| [Thread 根决策](../thread/0-durable-agent-thread-with-user-turns.md) D1/D4 与 Headless 决策：会话记录先持久化、会话终态由 TurnEnded 推导、定时运行空闲窗口为 0 | 继承，不变：verdict 与产物在会话结束后、工作区释放前处理 |
| [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D3：`provisioning → starting → running → delivering → releasing → done` 阶段机 | 继承；**修改**：`delivering` 同时收敛 Revision 交付与定时产物上传，但不新增 phase/status；产物失败不改变 Revision 或运行终态 |
| [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D4/D5：执行结果与 Revision 分别记录，Revision 登记前不释放，交付失败有界重试 | 继承，不变：产物是 Revision 之外的独立对象，不进入 Revision bundle，也不改变交付门禁 |
| [Controller 集成 D1](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)：`AgentSessionEnded{reason, detail}` 是 Agent 会话的终态结果 | **修改**：定时运行可携带可选的 `scheduled_result` 与 `artifact_manifest`；手工运行与既有字段语义不变 |
| [Revision 根决策](../revision/0-cloud-owned-object-store-and-verified-revisions.md) D1–D4：对象键由 Cloud 生成、预签名 PUT、校验大小与 SHA-256 后登记 | 继承并复用对象存储与上传授权机制；产物使用独立的运行级前缀与登记表 |
| [local 结果注解/产物 ADR](../../desktop/core/scheduled-task/20261021-run-result-annotation-and-artifacts.md) D5：local 与 Cloud 只共用 verdict 枚举，Cloud 存储/写入/保留另行定义 | 继承并落定：Cloud 只共用枚举；Node 执行端沿用同一哨兵文本约定作为上报来源，但不共享 Cloud 存储或解析契约；写入通道、产物存储、上限、保留和费用均在本文定义 |
| [个人模型发起者决策](../thread/20261010-personal-model-initiator-controls-input.md) 与发送者权限 | 继承，不变：定时运行仍 `canAppend = false`，产物与 verdict 不开放任何输入通道 |

## 决策

### D1：verdict 词汇与 local 共用，Cloud 的落点是运行级只写结果

Cloud 与 local 共用同一 `pass / warn / fail / blocked / idle / range_unresolved` 枚举，且：

- 新增 `scheduled_run_results`（字段至少：`tenant_id`、`task_id`、`occurrence_id`、
  `issue_run_id`、`verdict`、`verdict_detail`、`verdict_at`）。记录以 `issue_run_id` 为唯一身份，
  `task_id` / `occurrence_id` 是可空关联；100 条历史视图裁剪删除 occurrence 时保留结果行并把关联置空，
  不级联删除运行级 verdict。结果与 `issue_runs.status` 分开，不从执行层状态推导。
- 只有定时触发且会话正常结束（Thread 按 Headless 决策进入空闲并以正常 `TurnEnded` 收尾、IssueRun
  推导为 `completed`）的运行可携带 verdict；`failed`、`cancelled`、`timeout`、`missed`、`skipped_*`
  以及手工 IssueRun 一律没有 verdict。
- detail 为受限自由文本，上限 2000 字符；首版不提供 `custom` 枚举，非标准结论写入 detail。
- `status = completed` 且 `verdict = fail` 合法并存；读取历史时缺少记录显示「未报告」。
- 一个运行最多写入一次 verdict；后续事件、重放或任何外部调用都不能覆盖或清空已写入的事实。

### D2：可信写入者是执行端，Cloud 不解析 Thread 内容

verdict 由 Node 执行端在会话收尾时从 Agent 最终 Assistant 轮解析，经既有 Node→Cloud 终态事件
结构化上报。Cloud 只校验枚举、detail 长度和运行身份，不读取或解析 Thread / 消息 / 工具业务字段，
从而保持 Thread D2 不变。

- **哨兵契约**：与 local 相同，最终 Assistant 轮按 LF 归一后匹配以行为界的
  `QA_VERDICT: <pass|warn|fail|blocked|idle|range_unresolved>`（大小写不敏感、行尾空白忽略），可紧随
  一行 `QA_DETAIL: <自由文本>`；取第一个匹配，非法值/超长/无匹配均为「未报告」，不报错、不改写运行事实。
- **Node 协议扩展**：`AgentSessionEnded{reason, detail}` 增加可选的
  `scheduled_result{verdict, verdict_detail}`；该字段只对定时运行上报，手工运行不设置。Node 不写
  Thread 消息内容，只上报解析后的结构化值。
- **Cloud 写入时机**：`sessionEnded` 钩子在接管会话终态的同一事务内校验并写入 `scheduled_run_results`，
  先于任务历史可见；无匹配、非法值或超长不写入结果行（读取时即「未报告」），不报错、不改写运行事实。
  重放同一终态事件必须幂等。
- **为什么不在 Cloud 解析文本**：Thread D2 已冻结「Cloud 不解析消息业务字段」；把解析放在 Node 执行端
  既复用 local 的哨兵约定，又避免 Cloud 猜测 `ora-history` 的业务 schema。为什么不是 Agent 直写：
  Agent 是 Node 上的子进程，没有可信 Cloud 写入通道；开放公开端点会破坏「只有执行端能写」的边界。

### D3：产物在会话结束后由 Node 声明并上传，Cloud 生成对象键

运行 Workspace 是一次性隔离环境，产物必须在 `releasing` 前离开 Workspace。首版沿用 local 的
`.ora-scheduled-artifacts/` 固定目录与类型白名单，但 Cloud 不扫描 Workspace：

- **声明与采集**：仅在正常 `TurnEnded` 收尾时，Node 执行端在 Workspace 释放前收集固定目录下的
  白名单文件（`markdown/json/txt/log/xml/html`），执行与 local 一致的路径安全、符号链接/硬链接、
  大小、数量、总大小与大小写/NFC 检查；采集结果写入 `AgentSessionEnded` 的
  `artifact_manifest{items[], rejections[]}`。`failed / cancelled / timeout` 不采集。
- **对象键与上传**：Cloud 为每个被接受的条目生成不透明对象键，例如
  `scheduled-runs/{tenantId}/{issueRunId}/{artifactIndex}`，绝不使用用户文件名作为键；新增
  `GrantScheduledArtifactUpload{execution_id, artifacts[{index,size,sha256}]}`，只对处于 `delivering`、
  已登记且目标 Node/epoch 匹配的定时运行签发单对象 PUT 授权，规则复用 Revision 上传授权（`If-None-Match: *`、
  有效期、校验大小与 SHA-256）。Node 不能选择对象位置或覆盖既有对象。
- **收敛与失败**：Cloud 在 `sessionEnded` 钩子为含产物的定时运行登记独立的
  `DeliverScheduledArtifacts` 工作项（不新增 phase），Node 上传后上报
  `ScheduledArtifactsDelivered / Failed`。`delivering` 在 **Revision 与产物上传都收敛**后进入
  `releasing`：产物成功则登记 `scheduled_run_artifacts`；上传失败/超时在
  `artifact_delivery_state = failed` 或被拒清单中可见，并经过固定 10 分钟上限后放行。运行 Workspace
  在产物收敛或窗口到期前保持不释放。产物失败不改变 IssueRun 状态、Revision、通知或交付重试；没有
  产物时该工作项直接完成。对象存储未配置时产物以 `skipped(object_store_unconfigured)` 收敛，不阻断
  运行与释放。
- **上限**：单产物 10 MiB、单运行 64 个、单运行总计 64 MiB；超限/非法类型/非法路径进入被拒清单，
  不静默丢弃，也不阻断运行收尾。上限在 Node 与 Cloud 两侧校验，Cloud 以服务端上限为准。

### D4：产物按运行索引，读取跟随 issue 可见性

- `scheduled_run_artifacts` 至少保存：`tenant_id`、`task_id`、`occurrence_id`、`issue_run_id`、`object_key`、
  `display_name`（相对路径，仅用于展示）、`kind`、`size_bytes`、`sha256`、`created_at`。被拒候选单独存
  `scheduled_run_artifact_rejections`（名称 + 原因）。
- 任务历史读取时一并返回 verdict、产物数量和被拒数量；产物读取/下载只在 Cloud 公开 API 上提供，按
  当前有效租户成员对绑定 issue 的读取权限判定，不存在与无权限统一 `404`。
- 下载使用短期签名 URL，不暴露永久公开对象地址；对象存储凭据与桶配置只留在 Cloud，Controller/Node
  不持有对象存储业务凭据。产物不随 Revision、Thread 或 Issue 评论复制。
- 手工 IssueRun 没有定时产物；同一 IssueRun 的产物只能由该次运行的执行端上报，不能由普通客户端
  直接创建。

### D5：保留、清理与费用按 Cloud 自己的边界定义

- **每任务产物保留**：每任务只保留最近 **10 个含产物的运行**（按 occurrence 的 `scheduled_at` 从新到旧）；
  第 11 个含产物运行登记后，异步删除最旧运行的产物对象与索引。历史视图中被淘汰的运行仍保留运行事实，
  产物显示为「已过期」。「10 个运行」是本文冻结的 Cloud 成本边界，不复用 local 的 worktree 链接例外，
  也不改变根决策 D6 的 100 条历史视图。
- **历史视图裁剪**：100 条裁剪只删除任务侧 occurrence/展示关联；verdict 与 IssueRun/Thread/Revision
  事实保留，artifact 对象与索引按上一段的 10 个运行边界清理。任务软删除保留既有运行事实与产物，
  产物仍受 10 个运行上限约束，直到空间/租户删除或保留策略淘汰；不做即时级联删除。
- **空间删除**：Space/租户删除沿用既有数据生命周期，对象与索引随空间所有权删除；本文不新增
  跨租户搬迁或恢复语义。
- **费用归属**：产物对象存储与流量费用归任务创建者所在的 Space/租户，与模型费用归属一致；首版
  只设每运行上限和每任务 10 个运行上限来约束成本，不新增独立计费项或全局配额。费用明细与配额属于
  后续决策。
- **敏感信息**：不做内容密钥扫描（沿用 local ADR D4 的 owner 结论）。Ora 自身不向产物存储写入模型
  Key、token 或环境变量；产物是 Agent 输出，暴露面由类型白名单、大小上限、运行可见性和短期下载
  授权约束。

### D6：历史投影、事件与 Cloud/local 边界

- 任务历史条目增加 `verdict`（可空）、`verdict_detail`（可空）、`artifact_count` 与
  `rejected_artifact_count`；verdict 只来自 `scheduled_run_results`，缺失即「未报告」；产物数量只统计
  已登记对象，不读取对象内容。
- 产物登记/清理后通过既有 Space 事件流发失效提示；事件只带任务/运行标识，不作为产物事实源。
- Cloud 与 local 只共享 verdict 枚举；哨兵文本格式是用户 prompt 约定，Node 执行端采用它作为上报
  来源，但不构成 Cloud 存储或解析契约的共享。两端不共享存储、ID、索引或同步协议；local 产物不上传
  Cloud，Cloud 产物不同步 local；需要跨端查看时，用户仍需在对应执行端查看。

## 不变量

1. `issue_runs.status` / `failure_reason` 的语义、阶段机、通知与历史保留不回退。
2. verdict 独立、可空、只由执行端结构化上报；只有正常完成的定时运行可携带，单次写入，非法输入不改写运行事实。
3. Cloud 不解析 Thread / 消息 / 工具业务字段；verdict 哨兵解析发生在 Node 执行端。
4. 产物必须在运行 Workspace 释放前完成声明与上传；worktree/Workspace 回收后，已登记产物仍可按 Run ID 读取。
5. 产物对象键由 Cloud 生成；路径穿越、符号链接/硬链接越界、非法类型与超限进入被拒清单且可见。
6. 产物上传/登记失败不改变 IssueRun、Revision、通知或回收语义；`delivering` 不因产物无限期占用。
7. 任务历史 100 条裁剪只作用于视图；verdict 与 IssueRun/Thread/Revision 审计事实不被裁剪删除，产物按 D5 独立淘汰。
8. 产物只随绑定 issue 的读取权限可见；Cloud 与 local 不互相同步或上传。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Cloud 直接解析 Thread 最后一条消息文本 | 违反 Thread D2「Cloud 不解析消息业务字段」，并把 `ora-history` 的 schema 猜测引入 Cloud |
| 给 Agent 开放公开 verdict 写入端点 | 破坏「只有执行端能写」的可信边界，任何客户端都能伪造业务结论 |
| 由 Agent 直接上传产物到对象存储 | Agent 运行在一次性 Workspace，没有 Cloud 对象存储凭据；开放凭据会扩大攻击面 |
| 把产物塞进 Revision bundle / session JSONL | Revision 是 Git 成果与历史归档，产物与 Revision 生命周期和校验语义不同，混入会扩大交付失败面 |
| 产物跟随 Worktree/Workspace 生命周期 | Workspace 释放后丢失，无法按 Run ID 稳定访问 |
| 复用 local 的每任务 10 个运行、随历史裁剪清文件 | Cloud 的 100 条历史视图与对象存储成本边界不同；本文显式定义 Cloud 自己的 10 个运行产物上限 |
| 开放永久公开对象 URL | 绕过 issue 可见性；短期签名 URL 才能跟随读取权限 |
| 对产物做内容密钥扫描 | 漏报率高且只带来虚假安全感；沿用 local owner 结论，靠白名单、上限、可见性与不写凭据约束 |

## 本决策未解决的问题

- 产物对象存储成本明细、租户配额与超额计费；首版只用每运行/每任务上限约束。
- 历史按 verdict 过滤/排序、聚合看板，以及 verdict 驱动的自动重试或告警。
- Cloud 产物是否需要与 Space 导出/备份/迁移联动。
- 哨兵在实践中漂移时是否升级为专用结构化工具（与 local ADR 同一升级路径）。

## 落地顺序

1. **specs**：本 ADR 评审通过并合并，核心用例建立（义务初始 `Missing`）；`domain/glossary.md` 的
   「Scheduled Run Artifact」扩展到 Cloud 稳定存储语义。
2. **Cloud 契约实现**：迁移（结果/产物/被拒表）、Node→Cloud `scheduled_result` 与 artifact manifest
   协议、上传授权与登记、历史读取/下载 API、保留清理任务。
3. **真实栈验收**：用一个真实定时运行验证 verdict + 产物上传 + Workspace 释放 + 历史读取 + 重启/重放；
   验证 10 个运行淘汰、权限与失败降级。
4. **收尾**：核心用例证据补齐，ADR 转 `implemented`，关闭承接事项。
