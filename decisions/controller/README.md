# Controller 决策目录

本目录承载 Control Plane / Controller 的架构决策。Controller 是无租户的控制与编排层；在
Server 场景中由管理节点 API Server 负责租户隔离、认证和权威持久化，Controller 通过内部控制
接口访问这些能力；在 Desktop 场景中可以使用本地持久化适配器。

每个叶子目录的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按日期追加。目录按 Controller
的能力边界划分，不预先决定 crate 名称或是否与 API Server 同进程。

| 目录 | 范围 |
|---|---|
| `api-boundary` | Client API、API Server 集成、认证上下文和租户边界适配 |
| `node-management` | Node 注册、能力、选择、拉起、停止和执行位置绑定 |
| `workspace` | Project、Workspace、Task 和 Worktree 意图及状态投影 |
| `session` | Session 业务记录、生命周期和 Node 会话编排 |
| `plugin` | Plugin 期望状态、Node 安装计划和运行状态汇总 |
| `effect` | Effect Scope、Desired State、Generation、协调和恢复编排 |
| `workflow` | Workflow 定义、运行、依赖、恢复和取消 |
| `persistence` | API Server 持久化接口、队列、outbox、lease 和 fencing |
| `protocol` | Client–Controller、Controller–Node 消息与内部 Control API |
| `lifecycle` | Controller 启动、后台运行、优雅停止、重启和升级 |

Node 的实际文件、Git、Agent、Plugin 和资源操作放在 `specs/decisions/node/`；API Server 的
具体租户模型和公开 API 实现不在本目录建立领域决策。

## 已批准的跨仓库依赖

[运行时控制同步交付决策](../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)为 `implemented`，定义 Cloud 场景下 Controller 的服务身份、派发许可、旧执行对账及强停协调义务。业务权限由 Cloud 决策拥有；本文仅导航，不建立第二份权限规则。继续遵守只拨出和过渡监听退役；完整联合故障矩阵仍有缺口，直接证据见[跨进程核心用例](../../test-cases/cloud/controller-integration/fenced-runtime-control-delivery.md)。

[交付执行结算与未受理证明](../cloud/issue-run/20261026-delivery-execution-settlement-and-non-acceptance.md)
为 `approved`：它修改 `node-management` 根决策 D5 和 `session` 根决策 D4/D5 的未完成执行判定，
仅在原身份未受理证明经 Cloud 持久接管后解除对应责任；其余 quiesce 与派发账本规则继续适用。

[不可用 Node 延后与 clone 分轮等待](node-management/20261028-defer-unavailable-node-operations.md)
为 `implemented`：实现基线为 Cloud #206 合入版本 `1cfaed3` 与 Controller #750 PR head `9078570`；
本规格 PR 须等 Desktop #750 先合入，并补其 merge commit，不把 PR head 说成 main 已落地。
Controller 缩短轮内等待，Cloud 持久保存退避次数与有限原因，并自动接管超时后
原执行晚到的已知结果；当前创建时间排序即可实施，不依赖领取轮转合并。
直接验证义务见[对应核心用例](../../test-cases/controller/node-management/20261028-defer-unavailable-node-operations.md)，
按直接测试与部署实际覆盖分别标记 `Covered`、`Partial`、`Missing`；实现状态不消除缺口，受控联合测试调老
执行时间，不替代部署或真实 30 分钟证据。blocked/node_unavailable 无界 retrySeconds 保留为独立后续问题。

