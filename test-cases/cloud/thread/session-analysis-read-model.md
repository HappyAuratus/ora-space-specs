# 会话分析：授权读取与归档保真

决策依据：[会话分析只读视图](../../../decisions/cloud/thread/20261016-session-analysis-read-model.md)（`implemented`）。
直接代码证据固定在 [Cloud PR #69](https://github.com/ora-space/cloud/pull/69) 的合并提交
[`7c9cd8551f6b39b752265d4ba60f8aba2a33701a`](https://github.com/ora-space/cloud/tree/7c9cd8551f6b39b752265d4ba60f8aba2a33701a)。
`Covered` 仅表示该提交在所列自动化测试边界内直接覆盖义务；部署验收独立记录，不能替代证据缺口。
真实 PostgreSQL、受控 HTTP 对象服务和前端组件分别证明各自边界；测试中的 Revision 为合成登记，
不能证明 Node 上传、真实 S3 与浏览器贯通。

## Session reads must preserve current tenant scope without provisioning identities

- 风险：跨租户或已删除运行被列出，查看页面隐式创建身份或绑定别名。
- 前置状态：存在 Agent 与非 Agent Run、有效/停用成员、删除记录和已验证的身份别名。
- 触发：读取列表，按项目、Agent、状态、时间和游标筛选，或用未知身份读取历史。
- 必须成立：仅返回当前租户可见 Agent IssueRun；筛选与倒序分页稳定；未知/重复/空/畸形查询被拒绝；读取不写入身份或运行。
- 禁止结果：跨租户游标泄漏、非 Agent 会话混入、GET 触发 JIT 建号或新别名绑定。
- 决策依据：D1、不变量 1、3。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 列表范围、精确筛选、倒序游标及只读身份 | [Core 列表测试][list]：`TestSessionListScopeAndFilters`、`TestSessionListDescendingCursorBoundaries`、`TestSessionListRejectsInvalidFiltersAndDoesNotProvisionReaders`，真实 PostgreSQL | Covered |
| HTTP 查询白名单、非成员/跨租户/删除/非 Agent 拒绝 | [HTTP 测试][http]：`TestSessionDashboardListHTTPAndStrictQueries`、`TestSessionDashboardReadAuthorizationAndHistoryWiring` | Covered |
| 既有 Huawei 别名只读解析和冲突拒绝 | [历史读取测试][history-db]：`TestSessionReadIdentityResolvesHuaweiAliasesWithoutWriting`、`TestSessionReadIdentityRejectsHuaweiAliasConflict` | Covered |

## Archive reads must reauthorize after verified external IO

- 风险：任意对象读取、读取期间撤权后仍返回归档、损坏归档被信任、网络 I/O 长时间占用数据库事务。
- 前置状态：Agent Run 已登记未过期 Revision，包含对象路径、大小和 SHA-256；读取期间可撤权或更改登记。
- 触发：历史接口通过 Cloud 私有存储端点读取对象。
- 必须成立：外部读取前后核验当前成员、Issue/Run 及原登记；仅读取登记的 session.jsonl；校验字节大小/摘要，限制 32 MiB、100000 条及读取时限；取消传播；原始物理顺序与字段保留。
- 禁止结果：浏览器指定对象键、签名或存储 URL 出现在公开结果、重定向携带签名、损坏归档被当作完整 Thread、持数据库事务访问对象存储。
- 决策依据：D2、不变量 1–3。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 正常读取无写入、不跨 HTTP 持有数据库连接，结果不泄漏对象定位 | [历史读取测试][history-db]：`TestSessionHistoryReadsRegisteredArchiveWithoutWritesOrLongTransaction`，真实 PostgreSQL 单连接与受控 HTTP | Covered |
| 读取前拒绝越权/过期/跨范围对象；读取后拒绝撤权、登记替换或删除 Run | [历史读取测试][history-db]：`TestSessionHistoryRejectsUnauthorizedScopeAndUnavailableBeforeStorage`、`TestSessionHistoryRechecksMembershipAndRegistrationAfterStorage` | Covered |
| 私有端点签名、大小/摘要/超限、重定向与取消 | [对象读取测试][object-read]：`TestReadVerifiedPrivateEndpointAndByteVerification`、`TestReadVerifiedRejectsUnavailableInvalidAndOversizedObjects`、`TestReadVerifiedDoesNotFollowRedirectOrLeakTransportDetails`、`TestReadVerifiedCancellationStopsBodyRead` | Covered |
| 合法 UTF-8 JSONL、原始顺序/数字和记录数边界 | [JSONL 测试][history-parse]：`TestSessionHistoryRecordsPreservesOriginalOrderAndNumbers`、`TestSessionHistoryRecordsRejectsInvalidJSONL`、`TestSessionHistoryRecordsBoundAndCancellation` | Covered |
| 成功响应的 `private, no-store` 与 30 秒读取超时 | 实现已有设置；上述测试未直接断言缓存头或超时期限 | Missing |
| 真实 Node 登记到 S3 读取、浏览器多身份撤权的全链路 | 上述自动化证据分别覆盖合成登记、受控对象 HTTP 及组件撤权，未贯通真实 Node、S3 与浏览器撤权 | Partial |

## Session projections must retain source boundaries and unknown observations

- 风险：尾部 Thread 被当作完整归档、record.seq 被当作公开游标、未知时间/用量被补成零、恶意 JSON 被执行。
- 前置状态：归档包含重复 seq、未知类型和缺失时间；另一运行只有分页 Thread，归档也可能校验失败。
- 触发：打开会话、加载更早页、切换时间轴与完整事件、刷新失败或撤权。
- 必须成立：归档物理记录保留、投影按最终版本重放；Thread 使用公开 seq；来源与不完整状态显式；无时间事件可查看，原始 JSON 按文本展示；未知测量与真实零不同。
- 禁止结果：校验失败静默降级、撤权后继续展示缓存、用数据库创建时间/相邻间隔伪造事件时间或耗时、把文本数当模型 HTTP 次数。
- 决策依据：D3、不变量 4。
- 验证义务与证据：

| 义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 重复版本、未知记录/时间、公开游标及真实零保真 | [适配器测试][adapter]：`replays the last physical seq version and keeps the original rows without mutation`、`preserves unknown time in timeline rows and validates timezone and calendar dates`、`uses public cursors independently of repeated archive seq and never invents submission time`、`retains raw payloads failure and measured zero while calculating coverage-aware durations` | Covered |
| 100000 条归档投影不会耗尽函数参数栈 | [适配器测试][adapter]：`projects an archive at the server record limit without exhausting the argument stack`；仅证明投影完成，不证明整页渲染性能 | Covered |
| 归档错误不降级、尾部历史明确分页、撤权后清缓存 | [页面组件测试][sessions]：`does not silently substitute a Thread for an invalid archive`、`loads older Thread pages without pretending they are a full archive`、`hides cached content after membership is revoked on refresh` | Covered |
| 完整事件可搜索、未知时间可达、原始 JSON 与缺失测量展示 | [时间轴组件测试][timeline]：`shows full payload, result, timing and JSON without treating absent measurements as zero`、`finds thoughts, lifecycle and unknown-time records while retaining raw replaced versions` | Covered |
| 真实浏览器完整交互及大归档渲染性能 | 组件和投影测试不测真实浏览器全链路、帧率或内存峰值 | Missing |

[list]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/internal/core/session_list_test.go
[http]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/integration/session_dashboard_test.go
[history-db]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/internal/core/session_history_db_test.go
[object-read]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/internal/objectstore/read_test.go
[history-parse]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/internal/core/session_history_test.go
[adapter]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/frontend/src/features/session-data/adapter.test.ts
[sessions]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/frontend/src/features/sessions/sessions.test.tsx
[timeline]: https://github.com/ora-space/cloud/blob/7c9cd8551f6b39b752265d4ba60f8aba2a33701a/frontend/src/features/session-timeline/session-timeline.test.tsx
