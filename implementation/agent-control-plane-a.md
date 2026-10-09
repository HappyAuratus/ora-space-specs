# A 控制面实现与直接验收记录

本记录反映 2026-10-01 的实际实现，不改变任何 ADR 的 proposed/approved/implemented 状态。用户明确授权按当前草案开发。对已批准契约更新原核心用例；Revision 草案的直接证据在此记录，不新建其核心用例。

来源：[分工 specs#58](https://github.com/ora-space/specs/pull/58)、[Cloud #41](https://github.com/ora-space/cloud/pull/41)、[原证据 specs#66](https://github.com/ora-space/specs/pull/66)、[cluster#5](https://github.com/ora-space/cluster/issues/5)、[cluster#7](https://github.com/ora-space/cluster/issues/7)。#41/#66 已合并，后续提交/配套 PR 在原 PR 描述关联。

本轮审查修复为 [Cloud `9ed8414`](https://github.com/ora-space/cloud/commit/9ed841474076831c8ae7444942f1ab1b87df81ee)，包含先前 `5e7c63f`，desktop 配套固定同一提交。后续 Cloud 分支尚未进入 main；直接证据针对该固定依赖。

## 实际实现及差异

| 接缝 | 实际行为 |
|---|---|
| 对象键 | enqueue 时由 Cloud 按 tenant/run/work 固定 bundle/history 键；同 work 重试不变，失败后新 work 使用新键 |
| v1 上传请求 | 新增可选 `GrantRevisionUploadRequest.checksums`（field 3），map 的键为固定对象键，值为小写 SHA-256；缺省保留旧请求。checksum 被签入临时 PUT 头，不改变持久输入。Go/Rust 同源生成，Buf breaking 与旧 wire 兼容测试 |
| 对象不可覆盖 | 所有 PUT（包括旧请求）签入 `If-None-Match: *`；仍有效授权不能改写已验证对象。412 仅说明首次对象存在，Cloud 仍校验声明大小和摘要 |
| 取消与插件恢复 | Controller 生命周期取消不落交付终态；单项插件失败释放原维护绑定，旧选择的迟到失败恢复最新 pending 且清除旧错误 |
| 部分上传恢复 | Go Node 替身在首次 PUT 前持久固定 bundle/JSONL 字节与声明；取消和双替身重启重用原内容，终态证据持久化后清理对象副本。兼容旧 pending 日志的 `Result: null`，授权仍不落盘 |
| 校验范围 | 按最新草案分工，Cloud 在事务外对私网 S3 HEAD 检查存在、Content-Length、存储 SHA-256；Git bundle 与 JSONL 语义由 Node 承担，Cloud 不下载解释 |
| 原 Node 证据 | 成功声明保留原样；明确对象不匹配另记 Cloud verification_failed，无 Revision；网络故障不接管、不 ACK，可重放 |
| 原子接管 | 外部校验后重查租约、身份、固定输入、运行状态、maintenance binding、准入和当前 Node；原结果、收据、Revision、业务写同事务。已提交重放不重新访问 S3 |
| A/B phase | A 当前 schema 尚无 B 的 delivering/releasing phase；使用已登记交付、已结束会话、非终态 run 与上述绑定围栏。B 的 phase 转换由同事务钩子接入 |
| deliverySettled | changed 成功 `saved`，无变更 `unchanged`，均含 revisionId/元数据；失败 `failed/reason`；无存储一次性 `skipped/object_store_unconfigured`，不派发交付 |
| 健康信息 | 数据库实际探测；objectStore 仅 configured/unconfigured，不代表 S3 在线，不返回 endpoint 或凭据 |
| 迁移 | 仅新增 0027；不改已发布迁移，保留旧插件 effect、原执行输入、命令顺序及 Node 证据 |
| 部署 | 用户指定 `rustfs/rustfs:latest` 替代 MinIO；真实验收 digest `sha256:8cc9801755448b71a786705ce76692c77e14936cccd87cf2fc31842e58f4d1ff`，RustFS v1.0.0。latest 可变化，脚本每次打印实际 digest |

## 直接证据

| 义务 | 代表性证据 | 状态 |
|---|---|---|
| changed/unchanged、缺失/摘要/大小错误、回滚与重复接管 | `TestRevisionS3VerificationAndAtomicTakeover`，真实 PostgreSQL/RustFS | Covered |
| 已登记对象拒绝 checksum/旧版有效授权覆盖、移除条件不能绕过签名 | `TestRevisionStoredObjectsRejectLiveGrantOverwrite`，真实 PostgreSQL/RustFS | Covered |
| 网络故障不 ACK、I/O 后租约与状态围栏 | `TestRevisionNetworkFailureRemainsReplayable`、`TestRevisionLeaseAndStateAreRecheckedAfterExternalVerification`，真实 PG + 故障注入 HTTP | Covered |
| 错误 checksum、过期拒绝、同键刷新 | `TestS3RejectsIncorrectChecksumAndExpiredGrant`，真实 RustFS | Covered |
| Go 替身交付、首对象真实 PUT 后取消、部分上传后双重启重用快照、授权刷新、事务回滚后终态重放、释放后 Revision 保留 | `TestSimulatorDeliversRevisionToS3AcrossRestart`，真实 gRPC/PG/Git/RustFS；重启前改变 checkout 内容以检出重新生成快照 | Covered |
| 旧 pending 日志仍可重放 | `TestAgentJournalWithoutTerminalResultRemainsPending`，原磁盘日志 `Result: null` 读回 | Covered |
| 上传响应丢失后的条件重试 | `TestRevisionUploadReconcilesAnAmbiguousCreatedObject` 直接覆盖 503 后 412 的上传流程；真实对象保护与不匹配拒绝分别由上述 RustFS 集成测试覆盖 | Covered |
| 单项失败后新选择收敛 | `TestPluginFailedResultReleasesMaintenanceForNewSelection`，真实 PostgreSQL + Node 证据替身，覆盖失败前/后移除期望 | Covered |
| 公网地址对 sandbox 可达、私网校验、授权不入库 | `TestS3PublicGrantIsUsableFromSandboxNetwork`，独立 sandbox 网络 PUT | Covered |
| 五钩子同事务成功/失败/重复/回滚，无存储 skipped | Thread、Revision、`agent_hook_atomicity_test.go`、`agent_workspace_failure_hook_test.go` | Covered |
| 新库及 0023/0024/0026 升级、重复迁移 | `agent_plugin_upgrade_test.go`、`agent_control_upgrade_test.go`、`revision_upgrade_test.go` | Covered |
| 真实生产 Rust Controller/Node 交付、ACK/日志、重启 | 仅生成协议兼容及消费端构建证据，未执行生产 relay 交付 | Partial |
| B IssueRun phase/业务投影/API/SSE/界面 | A 提供事务钩子，业务实现尚需 B | Partial |
| D 真实 Agent/插件运行与产物 | Go echo/Git 替身仅证明 A；真实 Agent 未验收 | Partial |

cluster 的 `task agent:acceptance` 强制 PostgreSQL、S3 和 sandbox 网络，不能跳过或用内存成功结果通过。完整 Go 格式/lint、PG+S3 测试、race、构建、协议及 OpenAPI/前端检查的最终输出在配套 PR 记录。未关闭或合并 cluster#5/#7；A 的直接闭环不等于 M1–M4 整体完成。

2026-10-01 对固定 Cloud `9ed8414` 的最终门禁均通过：`task check`（lint 0 issues；integration 273.840s）、`task test:race`（integration 574.014s）、`task build`；真实存储与 sandbox 网络下的 `task agent:acceptance`（integration 73.742s）通过，无跳过、失败或 race 报告。desktop `63132df9` 的协议三项兼容测试、clippy 与两消费端全目标构建通过，完整 [CI](https://github.com/ora-space/desktop/actions/runs/36741470877) 的 Windows/Linux/macOS Rust、frontend、Tauri、e2e 和 ci-ok 均通过。

## 2026-10-09 对 Cloud main 的重新验收（cluster#9 阶段 0）

[Cloud #44](https://github.com/ora-space/cloud/pull/44) 合并后改动了交付路径，此前只在 PostgreSQL 上验证。对 Cloud main
[`482477d`](https://github.com/ora-space/cloud/commit/482477d7459b) 重跑 cluster 的 `task agent:acceptance`（真实
PostgreSQL 17.11、RustFS 1.0.1 `sha256:1803faef57627e2d9c2e7d89d655d712ddded5389040054987163043fecb6a3c`、
独立 sandbox 网络）：`task test:revision` 全部 37 个测试通过，无跳过、无失败（integration 47.9s）。

- desktop 的 `third_party/cloud` 从 `9ed8414` 升到 `482477d`；两者之间 `proto/` 无变化，`task proto:check`
  重新生成无漂移。
- Cloud 的 Backend CI 以固定 digest 的 RustFS 运行上述真实 S3 测试（`REQUIRE_S3=1`、`REQUIRE_S3_SANDBOX=1`），
  不再跳过；`internal/objectstore` README 与交付测试头注释中“G-037 未与真实 S3 联调”的过时说明已删除。
- cloud#44 自述的两处 ADR 差异按实现改 ADR（仍为 proposed）：Revision D4 增加“与输入矛盾的声明在接管前
  拒绝”一步；identity-access D2 改为会话启动时解析身份，运行输入携带的身份优先。
