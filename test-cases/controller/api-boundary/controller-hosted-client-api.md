# Controller API 边界验收

依据[根决策](../../../decisions/controller/api-boundary/0-controller-hosted-client-api.md)、
[Cloud 拥有契约的后续决策](../../../decisions/controller/api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)与
[过渡 JSON 接口退役决策](../../../decisions/controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md)。
Cloud 契约的消费与调用方向证据核对 desktop 主干 `aa635a12`（含 desktop#598、#601、#602）；监听器退役后的证据
核对 [desktop#627](https://github.com/ora-space/desktop/pull/627)（已合并），`cargo test -p ora-controller` 与 Node standalone 测试已在该 PR 的提交上本地重跑。
Cloud 服务端一侧的证据见
[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md)。

根决策 D2／D3 的 Transport 与过渡 JSON 接口已按退役决策删除，原用例“Transport Validation Preserves Private
State”与“Api Results Follow Durable Controller Records”随被测行为一起移除。本机受理语义（持久提交后返回、
幂等、冲突、缺失与无终态区分）由[持久协调双适配器核心用例](../persistence/coordination-store-adapters.md)
经 `CloneIntake` 承担。

## Controller Executable Opens No Listener

- **风险**：Controller 仍为调用方打开端口，或监听参数被静默忽略、在拒绝前留下状态。
- **前置状态**：SQLite 或云端持久模式的部署配置，未初始化的数据目录。
- **触发**：以任一持久模式启动；传入 `--port`、`--transport`、`--socket` 等旧监听参数。
- **必须成立**：两种模式都不绑定 TCP 端口或 Unix socket；旧监听参数与相对配置路径在获取数据库租约前拒绝，
  不创建 Controller 状态；本机工作只经 `ControllerHandle`／`CloneIntake` 受理。
- **禁止结果**：可执行入口接受并忽略监听参数、为调用方开放任何受理通道、拒绝前创建数据库。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 旧参数拒绝且无副作用 | `apps/ora-controller/tests/composition.rs::refused_composition_leaves_no_state_behind` 以真实可执行入口验证 `--port`、`--transport`、`--socket` 与相对配置路径均失败退出且不创建 Controller 状态；`apps/ora-controller/src/cli.rs::flags_map_to_explicit_hosting` 验证解析层拒绝 `--port` | Covered |
| 任何模式都不监听 | 组合与 Node standalone 测试以 `ora-controller coordinating locally`／`coordinating through cloud` 日志判断就绪，不再读取监听地址；没有直接断言进程未绑定任何 socket 的测试 | Partial |
| 本机受理只经内嵌句柄 | `apps/ora-controller/tests/service.rs::local_composition_refuses_before_state_and_holds_an_exclusive_owner` 经 `SqliteStore` 受理后启停组合，读回原记录；`apps/ora-node/tests/standalone/repository_single_node.rs::single_node_composition_hosts_and_retires_its_node` 经同一所有者受理后由托管组合派发并接管真实 clone | Covered |

决策依据：退役决策 D1、D2，不变量 1–3。

## Generated Cloud Contracts Preserve The Boundary

- **风险**：desktop 使用的契约与 Cloud 锁定版本不一致，Controller 为 Cloud 打开监听，错误在多处被解释，或
  Controller 复制租户授权。
- **前置状态**：Cloud 拥有 proto；desktop 以 `third_party/cloud` submodule 锁定契约 commit，提交
  `ora-controller-proto` 客户端生成物；Controller 以云端持久模式部署。
- **触发**：移动 submodule 指针或修改生成物后运行检查；以云端持久模式启动并尝试打开监听；Cloud 返回各类
  gRPC 状态；Controller 发起调用。
- **必须成立**：submodule 位于锁定 commit、`proto/` 无本地修改、重新生成无 diff；云端部署不打开任何监听；状态码只在 Cloud 适配器内映射一次；每次调用只携带 `x-ora-controller-id` 声明，不含凭据
  或租户字段；无法作为 ASCII metadata 传输的 `ControllerId` 在启动时拒绝。
- **禁止结果**：本仓库保留 proto 副本或手改生成物、Cloud 拨入 Controller、协调逻辑解析 gRPC 状态或消息
  文本、Controller 做租户筛选。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 锁定与生成漂移 | `task proto:check`（`task lint:crates` 与 `.github/workflows/pr-tests.yml` 执行，CI 先 `task proto:init`）校验 gitlink、无本地修改与重新生成无 diff；lint／breaking 由 Cloud CI 承担 | Covered |
| 拨出且不监听 | `apps/ora-controller/tests/composition.rs::cloud_persistence_serves_no_surface_and_creates_no_local_state` 以真实可执行入口验证云端模式不建本机状态、Cloud 不可达时持续运行且无写资格；监听参数的拒绝见上一用例 | Covered |
| 错误映射唯一 | `apps/ora-controller/src/cloud/fault.rs::statuses_map_to_contract_classes` 覆盖冲突、缺失、资格失效、不可用与未知分类；尚无以真实 Cloud 服务端返回的状态驱动适配器的自动化测试 | Partial |
| 身份声明与租户边界 | `cloud.rs::every_request_names_the_controller` 与 `non_ascii_controller_ids_are_configuration_errors` 覆盖 metadata 声明与启动拒绝；契约不含租户字段仍只依赖评审 | Partial |

决策依据：后续决策 D1–D4，其不变量 1–5。
