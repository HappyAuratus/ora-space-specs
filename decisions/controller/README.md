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

