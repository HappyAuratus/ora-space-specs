---
status: proposed
---

# Cloud 持有对象存储配置并签发限定上传授权，Revision 在 Cloud 校验对象后才登记

对象存储（S3 兼容）的端点、桶与访问凭据是 Cloud 的启动参数；Controller 与 Node 都不持有对象存储凭据。
Revision 交付时，Controller 向 Cloud 申请本次交付的**上传授权**（针对固定对象键、限定时长的预签名 PUT），
原样转交 Node；Node 直接上传增量 bundle 与会话 JSONL，报告对象键、大小与 SHA-256。Cloud 在数据库事务
之外用自己的凭据校验对象存在且摘要一致，然后在事务中登记 Revision。Revision 记录最终版本与对象位置，
是 IssueRun 释放运行 Workspace 的前提。

本 ADR 处于 `proposed`，是新建领域 `cloud/revision` 的根决策。Revision 的 Git 语义（内部快照、累计增量
bundle、无改动时不生成 bundle）由 [Node revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md)
承担；何时交付、失败如何重试由 [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) D5
承担；gRPC 形状见 [gRPC 后续决策](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)。
兼容性边界：新增配置项与表，不改变已有数据。本文不设计 bundle 过期、恢复（从 Revision 重建 Workspace）
与推送到用户远端。

## 决策

### D1：对象存储是 Cloud 的启动配置，凭据按基础设施引用注入

Cloud 配置新增 `object_store` 段，均可由 `CLOUD_OBJECT_STORE_*` 环境变量覆盖：

| 字段 | 含义 |
|---|---|
| `endpoint` | S3 兼容端点 URL（当前 cluster 按用户指定使用 RustFS） |
| `region` | 签名区域 |
| `bucket` | Revision 对象所在的桶 |
| `path_style` | 是否使用 path-style 寻址（当前 RustFS 联调开启） |
| `access_key_id_file`、`secret_access_key_file` | 凭据文件路径；凭据值不进入配置文件、日志或数据库 |
| `upload_grant_ttl` | 单个上传授权的有效期，proposed 默认 15 分钟 |
| `public_endpoint` | 可选；签发给 Node 的 URL 所用端点，用于 Node 与 Cloud 看到的地址不同的部署 |

- 缺少 `object_store` 时 Cloud 仍可启动，但 Agent 运行在进入 `delivering` 时以 `deliveryState = skipped`
  直接进入释放，并在健康检查的依赖列表中报告对象存储未配置。这让不需要 Revision 的开发环境不被阻塞，
  同时不把“未保存”伪装成“已保存”。
- 配置错误（端点不可达、桶不存在）不阻止启动，只让交付失败并按 IssueRun D5 重试；Cloud 不在启动时
  创建桶。

为什么放在 Cloud：Cloud 是 Revision 的权威登记方，也是唯一有能力校验对象的一方；Controller 不持凭据，
符合“Controller 无租户、不持有业务权威”的边界，也少一个需要轮换 S3 凭据的进程。

### D2：对象键由 Cloud 按运行决定，Node 不能选择写入位置

每个交付工作项（一次交付尝试）对应一组固定对象键：

```text
revisions/{tenantId}/{runId}/{deliveryWorkId}/revision.bundle
revisions/{tenantId}/{runId}/{deliveryWorkId}/session.jsonl
```

- 键在 Cloud 放出交付工作项时生成并写入工作项的输入（[controller-integration D6](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)
  的 `execution_work`）。此时执行 ID 尚不存在（它由 Controller 在 `RecordDispatch` 时生成），所以键使用
  工作项 ID；每次重试是新的工作项，不会覆盖上一次尝试的对象，失败尝试的残留对象可按前缀清理。
- 预签名 URL 只对单个键、`PUT` 方法和 `upload_grant_ttl` 有效，并要求 `x-amz-checksum-sha256` 头，
  由 S3 在写入时校验内容与 Node 声明的摘要一致。

### D3：上传授权不属于执行输入，可以随时重新申请

- 上传授权（URL、需要附带的头、过期时间）由 Controller 在向 Node 下发交付执行之后调用
  `GrantRevisionUpload(execution_id, checksums?)` 获取，并以非持久消息交给 Node；Node 过期后可请求新的授权。
- 当前实现以可选 checksum map 接收 Node 预先算出的对象 SHA-256，签入 PUT 头；map 只含固定输入中的键，
  属于本次授权请求，不属于持久执行输入。旧 v1 请求仍可解码且保留缺省授权响应。
- 授权不写入执行登记、提交记录、Node 账本或日志；它是短期持有者凭据。执行输入只包含对象键，因此同一
  执行重传时输入不变，幂等规则不受授权刷新影响。
- 只有交付执行尚无结果、且运行处于 `delivering` 时 Cloud 才签发授权；否则 `FAILED_PRECONDITION`
  之外的业务拒绝统一为 `ABORTED`/`CONFLICT`。

### D4：Revision 在对象校验后登记，校验与事务分离

Node 报告交付结果（`delivered{finalCommit, baseCommit, ref, bundle?{key, size, sha256}, history{key, size, sha256}}`
或 `unchanged{...}` 或 `failed{reason}`）后，Cloud 按以下顺序处理：

1. 事务外：对每个声明的对象调用 `HEAD`（带 checksum 模式），确认存在、大小与 SHA-256 与声明一致。
2. 事务内：接管交付事件（收据规则同 clone），写入 `revisions` 行，运行进入 `releasing`。
3. 校验不一致或对象不存在：接管原 Node 事件，在独立 Cloud 验证结论中记 `failed{verification_failed}`，
   同事务调用失败的 deliverySettled，按 IssueRun D5 重试新交付；不登记 Revision。原 Node 成功声明与原始事件
   保留，不以 Cloud 推断覆盖；网络失败不接管、不 ACK，可重放。

`revisions` 表：`id`、`tenant_id`、`run_id`、`workspace_id`、`project_id`、`repository_url`、`base_commit`、
`final_commit`、`revision_ref`、`bundle_key/bundle_size/bundle_sha256`（无改动时为空）、
`history_key/history_size/history_sha256`、`created_at`、`expires_at`（第一版为空）。

- 无文件改动（`unchanged`）也登记 Revision，`final_commit = base_commit` 且没有 bundle；与“保存失败”区分。
- 为什么校验放在事务外：仓库规则禁止在事务中进行 HTTP 调用；先校验再写入，最坏情况是校验通过后事务失败，
  Controller 以同一提交身份重试即可，对象不变。
- 为什么由 Cloud 而不是 Controller 校验：Cloud 登记 Revision 就是在承诺“这些对象存在且可恢复”，承诺方应
  亲自验证；Controller 转述 Node 的“已上传”正是 Revision 方案否定的交付证据。

当前实现说明：外部 HEAD 后再次检查租约、执行身份/输入、非终态 run、maintenance binding、Workspace 准入及
当前 Node/sandbox，再把原结果、收据、验证结论、Revision、业务 hook 同事务提交；重复接管不重新访问存储。
A 现有 schema 不含 B 的 delivering/releasing phase，phase 仍由 B 的调用方事务与 hook 维护。相关直接验收及
草案差异见[实现记录](../../../implementation/agent-control-plane-a.md)；本 ADR 仍为 proposed。

### D5：公开读取只暴露元数据

`GET .../runs/{rid}` 的结果中包含 Revision 元数据（最终 commit、是否有改动、对象大小、`deliveryState`）；
第一版不提供下载或 diff 预览接口，对象 URL 不出现在公开 API 中。

## 不变量

1. 对象存储凭据只在 Cloud 进程内；Controller、Node、数据库与日志中没有对象存储凭据，也不持久化上传授权。
2. Node 只能写入 Cloud 为本次交付尝试（交付工作项）指定的对象键。
3. Revision 只在 Cloud 亲自校验对象存在且 SHA-256 一致后登记。
4. 同一交付执行的输入在授权刷新后不变。
5. 无改动的运行也有 Revision 记录，且没有 bundle。
6. 未配置对象存储时交付明确记为 `skipped`，不登记 Revision。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| S3 配置放在 Controller，由它签发授权 | Cloud 少一个依赖；但 Controller 要持有并轮换凭据，且 Cloud 仍需凭据才能校验 |
| 把 S3 凭据下发给 Node | 最简单；沙盒获得整个桶的写权限，违反“Node 不持凭据” |
| bundle 经 Controller–Node 长连接传回，由 Cloud 写入对象存储 | 无需预签名；但大文件会阻塞控制与 Thread 消息，Revision 方案已否决 |
| 预签名 URL 放进执行输入 | 省一次调用；URL 过期后重传同一执行会变成“输入不同”，破坏幂等 |
| 只信任 S3 的 checksum 头，不做 HEAD | 少一次请求；但无法区分“Node 声称上传了”与“对象确实存在”，例如上传请求从未发出 |

## 本决策未解决的问题

- bundle 与会话 JSONL 的过期与清理、失败尝试的残留对象清理任务。
- 从 Revision 恢复 Workspace（续接运行）。
- 下载、diff 预览与推送到用户远端（需要另行授权）。
- 生产对象存储的凭据轮换与加密策略。

## 落地顺序

1. 配置与对象存储客户端（签发、HEAD 校验），以真实 S3 兼容服务（当前 RustFS）的集成测试覆盖签发 URL 只能写指定键、
   checksum 不一致被拒、HEAD 校验失败不登记。
2. `revisions` 迁移与交付接管事务；`GrantRevisionUpload`。
3. cluster Compose 增加 S3 兼容服务（当前 RustFS）与 Cloud 的 `object_store` 配置。

保持不变：已有配置与 API。开始改变：Agent 运行保存 Revision。
