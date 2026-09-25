# 持久协调接口与双适配器验收

依据[已实现的后续决策](../../../decisions/controller/persistence/20260922-coordination-store-with-sqlite-and-cloud-adapters.md)。
证据核对 desktop 主干 `aa635a12`（含 desktop#601、#602），`cargo test -p ora-controller` 已在该提交上本地重跑。
SQLite 适配器的原子接受、接管与恢复义务沿用
[本机 Controller 归属核心用例](local-controller-node-ownership.md)中的证据，本文只登记适配器选择、远程适配器
特有的义务与两者共享的行为契约。Cloud 一侧的事务证据见
[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md)。

## Each Deployment Runs Exactly One Persistence Adapter

- **风险**：Cloud 不可用时退回本地写入形成第二个权威，云端部署残留本地状态，或配置组合出不可用的接口。
- **前置状态**：`persistence` 分别为 `sqlite` 与 `cloud` 的部署配置；Cloud 不可达。
- **触发**：以另一种适配器打开配置；云端配置含多个 Node、非法地址或 `api` 段；以云端模式启动、运行并停止。
- **必须成立**：每个适配器只接受自己的持久模式；云端模式只允许一个 Node、拒绝 `api` 段与监听参数；
  云端模式从不创建 `home_directory`、数据库或租约文件；Cloud 不可达时进程保持运行且无写资格。
- **禁止结果**：云端配置打开 SQLite、Cloud 不可达时本地接受或派发、云端部署组合出 JSON 接口。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 适配器互不替代 | `apps/ora-controller/tests/runtime.rs::embedded_owner_reopens_original_operations_and_rejects_overlap` 验证两种适配器互拒对方配置、云端拒绝两个 Node 与非法地址，且云端运行时运行后不创建本地目录 | Covered |
| 云端部署的可执行入口 | `composition.rs::cloud_persistence_serves_no_surface_and_creates_no_local_state` 验证拒绝 `--port`、不监听、Cloud 不可达时记录"lease not acquired"并保持运行，全程无本地状态；尚缺云端配置含 `api` 段被拒的测试 | Partial |

决策依据：D1、D2，不变量 2、6。

## Remote Writes Retransmit Lost Replies Under One Submission Identity

- **风险**：回复丢失后换身份重传导致重复生效，把冲突当作可重试，或把未知结果当作成功／失败。
- **前置状态**：Cloud 适配器的写调用；可控的 gRPC 状态序列。
- **触发**：连续超时后成功；首次即冲突；持续传输错误；各类状态码。
- **必须成立**：只有未知结果被重传且始终使用同一提交身份；冲突等最终裁决不重传；重传耗尽后报告未知；
  状态码按 ADR 映射为冲突、缺失、资格失效、不可用与未知。
- **禁止结果**：重传换身份、冲突被重试、未知被报告为成功或失败。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 同身份重传 | `apps/ora-controller/src/cloud/fault.rs::writes_retransmit_only_unknown_outcomes_under_one_identity` | Covered |
| 分类映射 | `fault.rs::statuses_map_to_contract_classes` | Covered |
| 资格失效后停止写入 | [Watch 信号流用例](../api-boundary/watch-signal-claiming.md#watch-lives-only-under-the-held-epoch)中的 `tests/cloud.rs::a_stale_stream_open_stops_claims_and_writes`（desktop#614，尚未合并）证明打开 `Watch` 被判陈旧后不再领取与登记；由其他写调用（接管、查询结果）的陈旧裁决触发的路径没有直接测试 | Partial |

决策依据：D3，不变量 4。

## Remote Adapter Registers Before Delivery And Takes Over Before Ack

- **风险**：云端模式下 Node 在登记成功前收到命令，或 Controller 在 Cloud 接管提交前确认事件。
- **前置状态**：真实 Cloud、云端模式 Controller 与真实 Node。
- **触发**：Cloud 接受 clone；Controller 领取、登记并派发；Node 回报结果；在登记后强杀 Controller 并重启。
- **必须成立**：命令只来自 `pending_dispatches` 中已登记的执行；`EventAck` 只在 `TakeOverNodeEvent` 成功后发送；
  重启后 Controller 从 Cloud 恢复原执行并接管 Node 重放的原事件；Cloud 只有一次执行。
- **禁止结果**：派发未登记的执行、提前 Ack、重启后创建第二次执行。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 登记与接管顺序 | 协调逻辑的 `take_over` 只在 `take_over_node_event` 成功后产生 Ack，对两种适配器共用；Cloud 侧顺序由 Cloud 集成测试证明；以真实 Cloud 为对端的云端闭环仅在 desktop#602 手工验证 | Partial |
| 原执行恢复 | `apps/ora-controller/src/cloud/mapping.rs::records_rebuild_the_original_command` 验证由 Cloud 记录重建原命令；登记后强杀并恢复的场景仅在 desktop#602 手工验证 | Partial |

决策依据：D1、D2、D4，不变量 3、5。

## Both Adapters Honor One Behavioral Contract

- **风险**：两个适配器对同一业务承诺的实现悄然分化，本机测试通过不代表云端行为正确。
- **前置状态**：同一套行为用例，分别以 SQLite 与 Cloud（真实或行为等价的测试服务端）为权威。
- **触发**：接受幂等与冲突、登记先于派发、接管先于 Ack、查询结果不产生 Ack、缺失与无终态区分、崩溃后恢复。
- **必须成立**：两种适配器对每个用例给出相同的可观察结果。
- **禁止结果**：只对 SQLite 验证、以单元映射测试代替行为验证。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 共享行为套件 | 尚无对两种适配器共同执行的套件；SQLite 行为见本机归属用例，Cloud 行为只有单元与手工证据 | Missing |

决策依据：D4，不变量 1、3、4。
