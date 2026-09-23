# Controller API 边界验收

依据[根决策](../../../decisions/controller/api-boundary/0-controller-hosted-client-api.md)与
[Cloud 拥有契约的后续决策](../../../decisions/controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)。
入口迁移证据基于 [desktop#595](https://github.com/ora-space/desktop/pull/595) 的
`7b7fea969eba76fb0b8694f07b308f860a061200`，Linux 进程测试已在该提交上本地重跑。Cloud 契约的消费与调用方向
证据核对 desktop 主干 `aa635a12`（含 desktop#598、#601、#602），`cargo test -p ora-controller` 已在该提交上
本地重跑；Cloud 服务端一侧的证据见
[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md)。

## Transport Validation Preserves Private State

- **风险**：非法参数留下半初始化状态，或 socket 恢复覆盖用户文件。
- **前置状态**：部署配置、未初始化数据目录，以及普通文件、符号链接或活跃 socket。
- **触发**：默认 TCP、显式非回环、Unix socket 或不兼容参数组合启动。
- **必须成立**：默认 `127.0.0.1:4820`；显式非回环告警；非法组合在打开数据库前拒绝；
  socket 为数据目录直接子路径且为绝对路径，遵循私有创建与恢复规则。
- **禁止结果**：隐式监听所有接口、参数失败后创建数据库、覆盖普通文件、符号链接或活跃 socket。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 监听与拒绝时机 | `apps/ora-controller/src/cli.rs::flags_map_to_explicit_transports` 验证默认值与参数拒绝；`apps/ora-controller/tests/composition.rs::refused_composition_leaves_no_state_behind` 以真实可执行入口验证非法组合、相对配置路径与相对 socket 均拒绝且不创建 Controller 状态；`composition.rs::non_loopback_listener_warns` 断言 `0.0.0.0` 监听成功并写入告警日志 | Covered |
| Unix 路径保护 | `apps/ora-controller/tests/api.rs::http_acceptance_is_idempotent_and_survives_service_restart` 覆盖越界拒绝、socket 路径为普通文件或符号链接时拒绝且原文件逐字节保留，以及合法 Unix HTTP；`composition.rs::refused_composition_leaves_no_state_behind` 覆盖相对 socket 在租约前拒绝 | Covered |

决策依据：D2，不变量 3。

## Api Results Follow Durable Controller Records

- **风险**：HTTP 适配另造事实，或回复早于持久接受。
- **前置状态**：真实 Controller、Node，可注入存储失败和中断回复的调用方。
- **触发**：提交、重传、冲突、查询缺失／无终态记录，并中断调用方连接。
- **必须成立**：持久提交后才接受；相同请求返回原执行；冲突不覆盖；缺失与无终态区分；
  存储失败为暂时不可用；目标来自配置 `api.node_id`；调用方退出不取消已接受执行。
- **禁止结果**：调用方指定 Node、重复执行、存储失败伪装 clone 失败、断连取消执行。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接受、查询与目标边界 | `apps/ora-controller/tests/api.rs::http_acceptance_is_idempotent_and_survives_service_restart` 覆盖幂等、冲突、缺失／待完成和配置目标；`apps/ora-node/tests/standalone/repository_minicloud.rs` 真实写锁覆盖 503 且无接受记录 | Covered |
| 断连与恢复 | `apps/ora-node/tests/standalone/repository_minicloud.rs::minicloud_http_clone_survives_controller_kill_and_replays_original_intent` 截断 202 回复并重启重传，保留原执行及唯一 Run | Covered |

决策依据：D1、D3，不变量 1、2、6。

## Generated Cloud Contracts Preserve The Boundary

- **风险**：desktop 使用的契约与 Cloud 锁定版本不一致，Controller 为 Cloud 打开监听，错误在多处被解释，或
  Controller 复制租户授权。
- **前置状态**：Cloud 拥有 proto；desktop 以 `third_party/cloud` submodule 锁定契约 commit，提交
  `ora-controller-proto` 客户端生成物；Controller 以云端持久模式部署。
- **触发**：移动 submodule 指针或修改生成物后运行检查；以云端持久模式启动并尝试打开监听；Cloud 返回各类
  gRPC 状态；Controller 发起调用。
- **必须成立**：submodule 位于锁定 commit、`proto/` 无本地修改、重新生成无 diff；云端部署拒绝监听参数且
  不打开任何监听；状态码只在 Cloud 适配器内映射一次；每次调用只携带 `x-ora-controller-id` 声明，不含凭据
  或租户字段；无法作为 ASCII metadata 传输的 `ControllerId` 在启动时拒绝。
- **禁止结果**：本仓库保留 proto 副本或手改生成物、Cloud 拨入 Controller、协调逻辑解析 gRPC 状态或消息
  文本、Controller 做租户筛选。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 锁定与生成漂移 | `task proto:check`（`task lint:crates` 与 `.github/workflows/pr-tests.yml` 执行，CI 先 `task proto:init`）校验 gitlink、无本地修改与重新生成无 diff；lint／breaking 由 Cloud CI 承担 | Covered |
| 拨出且不监听 | `apps/ora-controller/tests/composition.rs::cloud_persistence_serves_no_surface_and_creates_no_local_state` 以真实可执行入口验证云端模式拒绝 `--port`、日志无监听、Cloud 不可达时持续运行且无写资格 | Covered |
| 错误映射唯一 | `apps/ora-controller/src/cloud/fault.rs::statuses_map_to_contract_classes` 覆盖冲突、缺失、资格失效、不可用与未知分类；尚无以真实 Cloud 服务端返回的状态驱动适配器的自动化测试 | Partial |
| 身份声明与租户边界 | `cloud.rs::every_request_names_the_controller` 与 `non_ascii_controller_ids_are_configuration_errors` 覆盖 metadata 声明与启动拒绝；契约不含租户字段仍只依赖评审 | Partial |

决策依据：后续决策 D1–D4，其不变量 1–5。
