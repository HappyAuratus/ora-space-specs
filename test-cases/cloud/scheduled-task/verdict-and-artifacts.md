# Cloud Scheduled Task Verdict 与产物

本文件登记 [Cloud Scheduled Run verdict 与产物 ADR](../../../decisions/cloud/scheduled-task/20261022-run-verdict-and-artifacts.md)
的核心验证义务。实现尚未合入 main，义务如实标为 `Missing`；每项落地完成时按直接证据更新状态。

## Verdict Must Come From The Executor Only For Scheduled Runs

- 风险：任何客户端都能伪造业务结论，或手工 IssueRun 被误打上 verdict，或执行层状态与业务 verdict 相互污染。
- 前置状态：一个定时运行已接纳并开始执行，会话执行输入 `AgentSessionSpec.trigger = scheduled`；手工 IssueRun 无该字段。
- 触发：Node 在会话收尾解析最终 Assistant 轮的哨兵文本，经 `AgentSessionEnded.scheduled_result` 结构化上报；Cloud 在
  `sessionEnded` 钩子同一事务内写入 `scheduled_run_results`。
- 必须成立：只有当执行输入 `trigger = scheduled` 且该轮正常 `TurnEnded` 收尾、IssueRun 推导为 `completed` 的运行才携带
  verdict；Cloud 只校验枚举、detail 长度与运行身份，不解析 Thread 内容；`scheduled_run_results` 以 `issue_run_id` 唯一，
  单次写入，重放同一终态事件幂等，后续事件、重放或外部调用不能覆盖或清空已写入事实；`status = completed` 且
  `verdict = fail` 可合法并存，旧任务/无注解显示「未报告」。
- 禁止结果：给手工 IssueRun 写 verdict、从 `issue_runs.status` 推导 verdict、Cloud 解析 Thread 文本、重复写或清空 verdict、
  无效值/超长改写运行事实。
- 决策依据：本 ADR D1、D2 与不变量 2、3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 只有 `trigger = scheduled` 且正常收尾的运行才写 verdict，单次写入、重放幂等 | 尚未实现 | `Missing` |
| Cloud 只校验枚举/长度与运行身份，不解析 Thread；`status=completed` 与 `verdict=fail` 并存 | 尚未实现 | `Missing` |
| 无注解/旧任务显示「未报告」，非法输入不改写运行事实 | 尚未实现 | `Missing` |

## Artifact Collection Must Be Triggered And Path/Size Bounded

- 风险：手工运行被采集产物、非法路径/符号链接越界/硬链接被收进产物、超限文件被静默丢弃。
- 前置状态：定时运行输入 `trigger = scheduled`，工作区含 `.ora-scheduled-artifacts/` 下的白名单文件与若干非法/超限项目。
- 触发：Node 在正常 `TurnEnded` 收尾时于 Workspace 释放前收集固定目录。
- 必须成立：只在 `trigger = scheduled` 且正常收尾时采集；`failed / cancelled / timeout` 不采集；只收白名单类型
  （`markdown/json/txt/log/xml/html`）并做与 local 一致的路径安全、符号链接/硬链接、大小、数量、总大小与大小写/NFC
  检查；采集结果写入 `AgentSessionEnded.artifact_manifest{items[], rejections[]}`，超限/非法进入被拒清单，不静默丢弃，
  不阻断运行收尾。
- 禁止结果：采集手工运行、跨目录/越界文件被收进、超限静默丢弃、产物失败改变 IssueRun/Revision/通知/回收语义。
- 决策依据：本 ADR D3、D4 与不变量 4、5。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 仅定时触发且正常收尾时采集，失败/取消/超时不采集 | 尚未实现 | `Missing` |
| 白名单、路径安全与大小/数量/总大小上限，越界入被拒清单 | 尚未实现 | `Missing` |
| 采集/上传失败不改变执行、Revision 交付或回收语义 | 尚未实现 | `Missing` |

## Artifact Delivery Must Respect The Non-Renewable Deadline

- 风险：迟到回执绕过已结算保留、重复 PUT 制造重复登记、部分失败整批回退、截止随重派延长。
- 前置状态：含产物运行登记了 `DeliverScheduledArtifacts` 工作项与 `artifact_delivery_deadline`；上传可能跨越截止。
- 触发：Node 上传、`ScheduledArtifactsDelivered / Failed` 回执、截止结算、重派与重启。
- 必须成立：`artifact_delivery_deadline` 以运行身份唯一、不可续期（登记时刻 + 10 分钟），重派沿用同一截止；成功登记在锁定
  `artifact_delivery_state` 的同一事务内按数据库时间校验 `now < artifact_delivery_deadline`，到期（含恰好到期）只确认回执、
  不登记对象；登记按条目逐项结算（全部成功 `delivered`、部分 `partial`、全部失败 `failed`，失败项入被拒清单）；重复 PUT 在
  `If-None-Match: *` 下得 `412` 不视为失败，登记前逐对象 HEAD 复核 size/SHA-256，一致按「已存在」幂等登记；已上传未登记孤
  对象由前缀 GC 兜底清理。
- 禁止结果：截止后登记对象、用超时扫描与回执先后代替绝对截止、重复登记、整批失败丢弃成功项、依赖 PUT 时点的一次性校验。
- 决策依据：本 ADR D3 与不变量 11、12。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 10 分钟截止唯一、不可续期，重派沿用同一截止 | 尚未实现 | `Missing` |
| 到期（含恰好到期）只确认回执不登记，绝对截止不依赖扫描先后 | 尚未实现 | `Missing` |
| 重复 PUT 412 幂等、登记前逐对象复核、部分成功逐项结算 | 尚未实现 | `Missing` |

## Artifact Reads Must Follow Issue Visibility

- 风险：产物泄漏给无关成员、永久公开对象 URL 绕过可见性、被拒数量无法按运行归属鉴权。
- 前置状态：产物已登记并建立按 Run ID 的索引；成员资格可能变化。
- 触发：任务历史读取、产物下载、`404` 判定。
- 必须成立：产物读取/下载只在 Cloud 公开 API 上提供，按当前有效租户成员对绑定 issue 的读取权限判定，不存在与无权限统一
  `404`；Cloud 鉴权后通过私有端点读取对象，按登记大小/SHA-256 校验，上限 10 MiB，返回前再次核验权限与登记；浏览器不取得对象存储地址或凭据；`scheduled_run_artifact_rejections` 携带 `tenant_id`/`issue_run_id`，被拒数量按运行归属投影；
  读取权限、下载授权与每任务淘汰都经 `issue_run_id → issue` 定位，不依赖 `occurrence_id`。
- 禁止结果：暴露对象存储地址/凭据、未校验字节或内联渲染 HTML、把产物权限放宽到任务创建者之外、被拒清单缺运行身份导致无法鉴权投影。
- 决策依据：本 ADR D4、D6 与不变量 8、10。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 读取/下载跟随 issue 可见性，不存在与无权限统一 `404` | 尚未实现 | `Missing` |
| 私有端点读取与登记大小/SHA-256 校验（10 MiB 上限），缺失/不符返回 `404 artifact_unavailable`、未配置存储返回 `503` | 尚无已合并的直接证据 | `Missing` |
| 读取期间权限撤销、登记变更/淘汰后不返回字节，外部 I/O 不占数据库事务 | 尚无已合并的直接证据 | `Missing` |
| 原始字节响应使用安全 basename 的 attachment、octet-stream、nosniff 与 private no-store，HTML 也不内联执行 | 尚无已合并的直接证据 | `Missing` |
| 前端仅提供同源下载路径，浏览器与契约不取得对象地址/凭据或 `url` / `expiresAt` | 尚无已合并的直接证据 | `Missing` |
| 被拒清单按 `issue_run_id` 归属投影，不依赖 `occurrence_id` | 尚未实现 | `Missing` |

## Retention Must Stay Bounded Per Task And Independent Of Occurrence Trim

- 风险：产物无限增长，或 occurrence 裁剪破坏排序/归属/鉴权，或裁决事实被历史裁剪误删。
- 前置状态：任务已积累多个含产物运行；部分 occurrence 已被 100 条历史裁剪删除。
- 触发：第 11 个含产物运行登记、100 条历史裁剪、任务软删除、空间删除。
- 必须成立：每任务只保留最近 10 个含产物的运行（按产物行冻结的 `scheduled_at` 从新到旧），第 11 个登记后异步删除最旧
  产物对象与索引；排序与淘汰只依赖 `scheduled_run_artifacts` 行自带的 `task_id`/`scheduled_at`，不依赖 occurrence；裁剪
  occurrence 时把 `scheduled_run_results`/`scheduled_run_artifacts`/`scheduled_run_artifact_rejections` 的 `occurrence_id` 置空、
  不级联删除，`task_id`/`scheduled_at`/`issue_run_id` 保持不变；裁决与 IssueRun/Thread/Revision 审计事实不被裁剪删除。
- 禁止结果：依赖被裁剪的 occurrence 排序、把保留边界扩散到 IssueRun 审计事实、裁决随历史裁剪删除。
- 决策依据：本 ADR D5 与不变量 7、10。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 每任务 10 个含产物运行上限，第 11 个淘汰最旧对象与索引 | 尚未实现 | `Missing` |
| 排序/淘汰只依赖产物行冻结事实，occurrence 裁剪置空引用不级联删除 | 尚未实现 | `Missing` |
| 裁决与 IssueRun/Thread/Revision 审计事实不受历史裁剪影响 | 尚未实现 | `Missing` |

## Artifact Directory Must Stay Out Of The Revision Snapshot

- 风险：产物目录随 `git add -A` 进入交付 ref，跳过 issue 可见性与 Cloud 代理附件下载，破坏 10 个运行保留边界。
- 前置状态：仓库包含 `.ora-scheduled-artifacts/`（含已跟踪/已提交的同名目录）；定时运行产出产物。
- 触发：Node 交付快照以临时 index 执行 `git add -A`，随后推送交付 ref。
- 必须成立：`.ora-scheduled-artifacts/` 是 Ora 保留目录名，Node 快照对所有运行一律排除（不按 `trigger` 区分），产物文件
  不进入交付 ref 的最终提交；已跟踪/已提交的同名目录此后不再随快照更新，历史提交中的既有副本不删除、不回改。
- 禁止结果：产物随快照进入 Git 交付、绕过 issue 可见性与 Cloud 代理附件下载、指定仅定时运行排除。
- 决策依据：本 ADR D3 与不变量 9；对 Node Revision 根决策 D2 的修改。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 交付快照排除 `.ora-scheduled-artifacts/`，产物不进入交付 ref | 尚未实现 | `Missing` |
| 排除对所有运行生效（不只定时运行），已跟踪同名目录不再随快照更新 | 尚未实现 | `Missing` |