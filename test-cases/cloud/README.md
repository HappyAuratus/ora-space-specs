# Cloud 核心测试用例

本目录与 `specs/decisions/cloud/` 完全镜像。Cloud 各领域的 ADR 进入 `approved` 后，才在对应叶子
目录登记需要长期维护的租户与 owner 隔离、认证授权、聚合约束、幂等、并发、恢复、协议和升级用例。

占位目录只用于稳定领域路径，不表示已有决策或测试证据。核心用例必须按业务风险组织，并使用
`Covered`、`Partial` 或 `Missing` 如实标记直接证据。

## 已批准的多人协作运行时控制

以下核心用例依据 2026-09-28 批准的六份 ADR 建立。已补充已有真实路径的代码及直接证据；23 个稳定标题保持不变，新增义务按 Partial/Missing 登记。六份 ADR 仍为 approved：真实插件执行器、受控凭据解析/重绑、完整产品入口与故障/升级矩阵尚缺，不能据数据库或模拟器宣布整体完成。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| workspace | [使用权限与历史创建者](workspace/runtime-workspace-use-permissions.md) | 内容越权、归属误判、升级保真、独立运行时隔离 |
| execution-admission | [独占与安全交接](execution-admission/exclusive-runtime-control.md) | 并发获取、会话内冲突、到期、撤权与收尾 |
| operation | [普通停止与强停](operation/safe-stop-and-administrative-force-stop.md) | 活动保护、目标范围、迟到创建、终止及持久数据 |
| controller-integration | [跨进程执行约束](controller-integration/fenced-runtime-control-delivery.md) | 服务身份、旧指令、接受结果未知、重启和协议兼容 |
| plugin-marketplace | [管理员选择与维护等待](plugin-marketplace/admin-selection-and-idle-maintenance.md) | 管理员权限、维护独占、持久 pending、期望变更 |
| project | [凭据连续性](project/repository-credential-continuity.md) | 成员离开、受控团队绑定、静态回退、历史引用 |

真实端到端证据采用 cluster Compose 或等价真实部署链路，遵守 minicloud 与 Controller 过渡监听退役。核心用例使用英文二级标题作为稳定代码引用，实施时按义务逐项补充直接测试和对应仓库提交。

## 本轮提交与依赖

| 仓库 | 提交 | PR | 交付职责 |
| --- | --- | --- | --- |
| Cloud | fe33eb49633028f3fd466de966d3a5754d126557 | [#37](https://github.com/ora-space/cloud/pull/37) | PostgreSQL 权威、使用权限、控制、强停、pending 与凭据冻结 |
| desktop | 9b788efbaa388908e4068e6f92a3da48439d4ec0 | [#632](https://github.com/ora-space/desktop/pull/632) | 精确锁定 Cloud；Controller 派发、Rust Node 持久绑定及权限隔离 |
| cluster | 3ce153f322ca31dd402cb1704d5d01e3dcc86d7e | [#6](https://github.com/ora-space/cluster/pull/6) | 管理认证、迟到创建屏障、真实 Git/进程/存储验收 |

集成顺序为 Cloud 契约及数据库 → desktop Controller/Node → cluster 部署，再核查本库证据。三个实现 PR 分别面向上游 main；不直接合并 specs #56。真实验收八组检查已通过，定位信息和可重跑脚本在 cluster docs/runtime-control.md。完整测试的最终结果与环境限制在各实现 PR 更新。
