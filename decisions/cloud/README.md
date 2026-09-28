# Cloud 决策目录

本目录承载 Ora Cloud / API Server 的架构决策。Cloud 是多租户业务状态的权威入口，负责身份与
成员授权、Project 和 Workspace 聚合、持久操作、执行准入以及对 Controller 的受限内部接口；
Controller 和 Node 不得绕过 Cloud 建立另一份业务权威状态。

每个叶子目录的第一个 ADR 使用 `0-...md` 作为根决策，后续决策按 `YYYYMMDD-...md` 追加。目录按
Cloud 的领域与协议边界划分，不预先决定 Go package、数据库表或部署进程的拆分方式。

| 目录 | 范围 |
|---|---|
| `api-boundary` | 公开与内部 HTTP API、OpenAPI、分页、错误契约和输入边界 |
| `identity-access` | 外部身份映射、服务与用户凭据、认证上下文和访问判定 |
| `tenancy` | Tenant、Membership、角色、管理员保护和 owner 隔离 |
| `project` | Project 聚合、Repository 意图、凭据引用和 Project 级资源状态 |
| `workspace` | Workspace 业务身份、归属和生命周期意图：协作 Space（`collab_workspaces`）与运行时工作区（`workspaces`）两类聚合 |
| `operation` | 持久操作、幂等、版本、重试、状态转换和恢复证据 |
| `execution-admission` | Access、Execution Ticket、准入关闭、idle 证据和活动保护 |
| `controller-integration` | Controller lease、内部快照、effect plan、epoch fencing 和接管边界 |
| `persistence` | PostgreSQL schema、migration、事务顺序、约束和兼容性 |
| `lifecycle` | Cloud 启动、健康检查、优雅停止、配置、升级和部署边界 |
| `plugin-marketplace` | 插件市场目录快照同步、工作区插件选择状态、fan-out 与自闭合 effect 载荷 |

Controller 的调度与恢复策略放在 `specs/decisions/controller/`，Node 的 Git、文件、进程、Agent、
Plugin 和本地执行职责放在 `specs/decisions/node/`。跨层决策放在拥有业务不变量的一侧，并从其他
层引用，不复制成多份权威规则。

## 已批准的多人协作运行时控制（2026-09-28）

以下六份 ADR 于 2026-09-28 经用户明确批准，均为 `approved`，可以作为跨仓库开发依据。已补充核心用例，新保障的直接证据均为 `Missing`；批准不表示新权限或并发控制已经实现。实现与真实边界验收完成后才标记 `implemented`。

最初核查基线：Cloud main `67dcbf9`（PR #33 已合并）、desktop `652af71`、specs PR #55 `a81c263`。批准时 [PR #55](https://github.com/ora-space/specs/pull/55) 已合并，本分支已同步 specs 上游 `e2c8dce`；[PR #56](https://github.com/ora-space/specs/pull/56)仍面向 main，不直接合并。成员管理与多人操作控制分别验收，历史代码链接不能证明新增保障已落地。

上游已合入 [Controller 过渡 JSON 接口与监听器退役](../controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md)及[minicloud 退役](../minicloud/runtime/20260928-retire-minicloud.md)，由 desktop #627 落地。开发继续遵守 Controller 只拨出和浏览器经 Cloud 接入；真实端到端联调由 cluster Compose 栈承担，不恢复旧监听、DTO 或 minicloud。

| 建议阅读顺序 | ADR | 决策归属 |
| --- | --- | --- |
| 1 | [运行时使用权限](workspace/20260927-runtime-workspace-use-permissions.md) | 创建者、管理员、概况/内容边界、权限矩阵、历史归属、同类产品比较 |
| 2 | [独占操作与执行准入](execution-admission/20260927-exclusive-runtime-control.md) | 会话、写活动、续期、断线、撤回、正常接管与状态转换 |
| 3 | [普通停止与管理员强停](operation/20260927-safe-stop-and-administrative-force-stop.md) | 生命周期冲突、紧急终止、真实结束证据及数据保留 |
| 4 | [跨进程协议与同步交付](controller-integration/20260927-fenced-runtime-control-delivery.md) | 控制身份、旧操作资格隔离、组件责任、兼容升级及跨仓库验收 |
| 5 | [空间插件维护](plugin-marketplace/20260927-admin-selection-and-idle-maintenance.md) | 管理员选择、逐运行时维护独占与持久等待 |
| 6 | [仓库凭据连续性](project/20260927-repository-credential-continuity.md) | 团队/个人/未知引用、成员离开、验证重绑与执行端依赖 |

### 实施边界与默认值

- 本批采用“普通成员使用自己的运行时、管理员使用同空间全部运行时；操作仍需独占”的第一阶段方案，不建立成员授权关系或授予/撤销功能，不实现实时共同编辑。
- 60 秒独占有效期、20 秒续期，以及云端管理通道双向 TLS 和管理/工作负载权限隔离，已随本批 ADR 批准；执行端、协议和部署仍需实现与验证。
- 当前没有完整云端文件/终端/Agent 使用入口；Substrate 此次只有规格核查、真实插件执行器与按项目凭据解析也有实现缺口。具体能力须在执行端和部署配套完成后开放。
- 各 ADR 的验收要求已落实到[核心用例索引](../../test-cases/cloud/README.md)。按可独立判断的义务维护 `Covered`、`Partial`、`Missing`，既有成员或 clone 测试不作为新独占保障的完整证据。
- 旧 ADR 保留原状态；本批的继承表明确修改后的目标规则。产品实现尚未同步，原测试证据的适用边界需单独保留；本次未重命名旧文件、未改写迁移、未修改任何产品仓库代码。
