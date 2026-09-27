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

## 多人协作运行时控制提案（2026-09-27）

以下六份 ADR 均为 `proposed`，本次只交付规格供评审，不表示新权限或并发控制已实现。先评审并修正决策，明确标记 `approved` 后再维护核心测试用例和开始开发；实现与真实边界验收完成后才标记 `implemented`。

基线：Cloud main `67dcbf9`（PR #33 已合并）；desktop `652af71`；specs main `459b43d`，本批建立在仍为 Open 的 [PR #55](https://github.com/ora-space/specs/pull/55) `a81c263` 上。PR #55 的成员管理规则是依赖，不能据此认为多人操作控制已完成。新 PR 面向 main，评审时须区分依赖变更与本次新增提案。

| 建议阅读顺序 | ADR | 决策归属 |
| --- | --- | --- |
| 1 | [运行时使用权限](workspace/20260927-runtime-workspace-use-permissions.md) | 创建者、管理员、概况/内容边界、权限矩阵、历史归属、同类产品比较 |
| 2 | [独占操作与执行准入](execution-admission/20260927-exclusive-runtime-control.md) | 会话、写活动、续期、断线、撤回、正常接管与状态转换 |
| 3 | [普通停止与管理员强停](operation/20260927-safe-stop-and-administrative-force-stop.md) | 生命周期冲突、紧急终止、真实结束证据及数据保留 |
| 4 | [跨进程协议与同步交付](controller-integration/20260927-fenced-runtime-control-delivery.md) | 控制身份、旧操作资格隔离、组件责任、兼容升级及跨仓库验收 |
| 5 | [空间插件维护](plugin-marketplace/20260927-admin-selection-and-idle-maintenance.md) | 管理员选择、逐运行时维护独占与持久等待 |
| 6 | [仓库凭据连续性](project/20260927-repository-credential-continuity.md) | 团队/个人/未知引用、成员离开、验证重绑与执行端依赖 |

### 评审边界与默认值

- 本批采用“普通成员使用自己的运行时、管理员使用同空间全部运行时；操作仍需独占”的第一阶段方案，不建立成员授权关系或授予/撤销功能，不实现实时共同编辑。
- 60 秒独占有效期、20 秒续期，以及云端管理通道双向 TLS 和管理/工作负载权限隔离，是本次提出的技术默认值与交付要求，随 ADR 评审，不伪称已有决策或现状。
- 当前没有完整云端文件/终端/Agent 使用入口；Substrate 此次只有规格核查、真实插件执行器与按项目凭据解析也有实现缺口。具体能力须在执行端和部署配套完成后开放。
- 各 ADR 的“预期验收”用于评审系统承诺，不是正式核心用例或通过证据。批准后才在既有镜像领域目录维护核心用例，新保障没有直接证据时标为 `Missing`。
- 旧 ADR 的关联提示不改变其状态和当前规则；本批未重命名旧文件、未改写迁移、未修改任何产品仓库代码。
