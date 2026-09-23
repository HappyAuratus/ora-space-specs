---
status: implemented
---

# 持久协调接口与双适配器：本机 SQLite、云端经 Cloud RPC，Controller 不持有云端权威状态

Controller 的持久化操作经过持久协调接口 `CoordinationStore`：接口按完整原子业务操作定义（接管 Node
事件、保存查询结果、恢复原派发、列出无结果派发、读取结果），外加适配器自身与权威协调的长驻任务
`serve`；它不暴露 SQL、表或跨调用的事务。自己受理请求的权威另外实现 `CloneIntake`（接受请求、列出与
读取 operation）。接口有两个适配器：本机 single-node 使用 `SqliteStore`，同时实现两者；云端使用
`CloudStore`，把每个操作翻译为 Cloud gRPC 契约的调用，由 Cloud 在 PostgreSQL 事务中提交，它不实现
`CloneIntake`。适配器在部署时选定、运行时不切换；云端 Controller 不打开 SQLite，其连接、租约 epoch 与
在途协调都是可丢弃、可从 Cloud 重建的运行时状态。第一版只覆盖 clone 闭环。

本决策处于 `implemented` 状态，基于[根决策](0-local-controller-node-ownership.md)：本机规则（单 Controller
归属、派发前落盘、接管后 Ack、原身份恢复）全部继承，只把"由 Controller 自身的 SQLite 持久"改为"由当前
适配器持久"。接口抽取、部署模式与泛型运行时由 desktop#601 落地，Cloud 适配器与领取循环由 desktop#602
落地（均已合入；原计划单独提交的 desktop#599 并入 #601）。契约内容见
[Cloud 拥有 Internal Control gRPC 契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)，
归属与调用方向见
[Controller 契约方向后续决策](../api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)。
验证证据登记在
[持久协调双适配器核心用例](../../../test-cases/controller/persistence/coordination-store-adapters.md)。
本文不改变本机 Node 固定绑定 `ControllerId` 的规则，不设计多 worker 租约、fencing 到 Node 指令、沙盒
绑定或 Effect 级 operation 的云端协调。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：Node 固定归属单一 `ControllerId`，连接可替换 | 继承，不变 |
| D2：单一有效会话，本机 Unix socket，握手校验身份与能力 | 继承，不变 |
| D3：数据目录显式注入，独立 SQLite，同一库单一运行所有者 | 修改范围：这些规则只属于 SQLite 适配器；云端部署不创建数据库、不取文件租约 |
| D3：接受先原子保存输入、身份与待协调责任；派发前持久化 | 继承，由各适配器的受理路径承担：SQLite 在 `accept_request` 同一事务内登记派发；云端由领取循环经 `RecordDispatch` 登记，成功后 Node 会话才可见 |
| D4：结果与事件共用接管边界，提交成功后才 `EventAck`；查询结果不构成 Ack 依据 | 继承，由接口承担：`take_over_node_event` 成功后才 Ack；`record_queried_result` 不产生 Ack 依据 |
| D4：事务失败、提交结果未知、归属冲突时不确认 | 继承并细化："提交结果未知"成为远程适配器的显式错误分类（D3） |
| 不变量 5：本机 SQLite 不被扩展为 Cloud 权威存储 | 继承，不变 |

## 决策

### D1：接口按原子业务操作定义，派发登记属于适配器自己的受理路径

`CoordinationStore`（两个适配器都实现）：

| 方法 | 原子保存 | 成功的意义 | 幂等规则 |
|---|---|---|---|
| `take_over_node_event(session, event)` | 执行结果与事件收据 `(execution, sequence)` | 可以向 Node 发送该序号的 `EventAck` | 同序号同内容幂等；不同内容冲突 |
| `record_queried_result(session, operation, execution, result)` | 查询得到的 Completed 结果 | 结果可读取 | 同内容幂等；不同内容冲突；不产生 Ack 依据 |
| `original_dispatch(session, operation, execution)` | 无 | 恢复原命令以重传或校验归属 | 缺失或归属不符为冲突，不补造 |
| `pending_dispatches(node)` | 无 | 重连后周期查询的依据：该 Node 上**尚无结果**的派发 | — |
| `result(execution)` | 无 | 读取与权威无关的终态 `ExecutionOutcome`；`None` 是无结果或未知身份，不是失败 | — |
| `serve(shutdown)` | — | 适配器与其权威的长驻协调：云端维持租约、领取并登记派发，停止时释放租约；本机无事可做 | — |

`CloneIntake`（仅 `SqliteStore`）：`accept_request(request_id, spec)` 原子保存原请求、`operation_id`、
`execution_id`、目标 Node、完整输入与派发责任，同请求同输入返回原记录、不同输入冲突；`operations()`／
`operation(execution)` 供过渡 JSON 接口展示，缺失与无终态区分。

接口不暴露 `begin`／`commit`、连接、表名或 SQL，协调逻辑不能组合两次调用来"补齐"一个原子边界。

与提案相比，`record_dispatch` 不是接口方法：本机的接受与派发登记本来就是同一事务，云端的登记只发生
在适配器自己的领取循环里，协调逻辑与会话从不需要单独调用它。`accept_request` 也不是在云端返回配置
错误的方法，而是云端适配器根本不实现的 `CloneIntake`：云端部署因此在类型上无法组合出 JSON 接口，
而不是组合出一个只会报错的接口。两个适配器都通过泛型静态分发使用，运行时与服务的类型在构造时固定。

### D2：适配器在部署时选定，运行时不切换，不互为后备

部署配置以 `persistence` 枚举选择模式：`{ kind: sqlite }` 使用 `home_directory` 内的 SQLite 与文件租约
（继承根决策 D3 的全部文件与租约规则），并要求 `api` 段；`{ kind: cloud, endpoint, claim_interval_ms }`
使用 Cloud 适配器，拒绝 `api` 段，且只允许恰好一个配置的 Node（云端工作项不携带目标 Node）。启动时只
构造一个适配器；每个适配器只接受自己的持久模式。云端模式不创建数据库、不取文件租约、不创建
`home_directory`；gRPC 通道惰性连接，Cloud 不可达不是启动失败。

Cloud 不可用时 Controller 不切换到本地 SQLite 代写：那会形成第二个权威状态源。正确行为是保持未派发、
未确认，按 D3 的分类重试或等待。新实例只依赖 Cloud 的持久记录（`pending_dispatches`、`original_dispatch`）
与 Node 的原事件重放恢复，不依赖原实例的磁盘。

### D3：错误分类区分冲突、不可用、未知与资格失效

| 分类 | 含义 | 协调逻辑的行为 | SQLite 适配器 | Cloud 适配器 |
|---|---|---|---|---|
| 冲突 | 身份或内容与权威记录不符 | 不派发、不确认、不重试同一调用 | 事务内校验 | Cloud 裁决冲突或非法输入 |
| 缺失 | 无该身份的记录 | 读取为 `None`；要求事实存在的写入为冲突 | 查询为空 | `NOT_FOUND` |
| 不可用 | 未提交，可重试 | 保持未派发／未确认，稍后重试 | I/O 或 SQLite 错误（事务未提交） | Cloud 不可达或拒绝 |
| 未知 | 可能已提交但回复丢失 | 不确认；之后的重试依赖业务幂等 | 不产生 | 写入的回复丢失 |
| 资格失效 | 租约 epoch 陈旧或未持有 | 停止写入，重新获取后再继续 | 不产生 | `FAILED_PRECONDITION`，或尚未取得租约 |

Cloud 适配器为每次写调用生成一个提交身份，回复丢失时以同一身份重传（至多三次，见 api-boundary D3）；
协调逻辑不感知提交身份。仍未知时向上报告"未知"，**之后由协调逻辑发起的重试使用新的提交身份**，其
安全性来自接口的业务幂等：事件接管与查询结果按"同执行同内容"幂等，已提交的派发登记使请求离开
`queued`，下一次领取不会再为它创建执行，而登记的执行由 `pending_dispatches` 交给会话。这偏离了提案中
"未知只能用同一提交身份重传"的表述：提交身份覆盖的是单次调用内的回复丢失，跨调用的重复由业务身份
承担，两层叠加，而不是让协调逻辑持有并传递提交身份。

### D4：两个适配器验证同一业务承诺，远程另验证未知结果

两个适配器都必须满足 D1 表中的承诺：接受幂等与冲突、派发登记先于派发、接管先于 Ack、查询结果不产生
Ack、缺失与无终态区分、崩溃后恢复原身份。远程适配器另需验证：回复丢失后同身份重传、冲突不重试、
Cloud 不可用期间不派发不确认、资格失效后停止写入。

最终实现没有一套对两个适配器共同执行的测试套件：SQLite 适配器由现有本机测试直接证明；Cloud 适配器
由错误映射、同身份重传与契约映射的单元测试、配置与组合测试，以及以真实 Cloud 为对端的手工端到端验证
证明；Cloud 侧事务由 Cloud 的 PostgreSQL 集成测试证明。共享套件与自动化远程对端仍是缺口，如实登记在
核心用例中。

### D5：第一版范围是 clone 闭环，本机行为不变

本机模式的调用方可观察行为、数据目录布局、schema 与文件租约规则不变；`pending_dispatches` 在 SQLite
上同步收敛为"无结果的派发"。云端模式的接受入口是 Cloud 公开 API，Controller 经领取与登记获得工作，
Controller 不提供过渡 JSON clone 接口。worktree、sandbox、Effect 级 operation 的云端协调不在第一版。

## 不变量

1. 协调逻辑只通过持久协调接口读写持久状态；接口方法各自原子，不存在跨方法的事务组合。
2. 每个部署只有一个适配器；Cloud 不可用时不切换到本地写入，不产生第二个权威状态源。
3. 派发登记成功先于任何 Node 派发；`take_over_node_event` 成功先于任何 `EventAck`；`record_queried_result`
   不产生 Ack 依据。
4. 同身份同内容幂等，同身份不同内容冲突；未知结果从不被当作成功或失败，也不产生 Ack。
5. 云端 Controller 的运行时状态可丢弃；新实例仅凭 Cloud 持久记录与 Node 原事件恢复，不创建第二次执行。
6. 本机 SQLite 只服务本机部署；云端部署不创建任何本地持久状态。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Controller 独立拥有云端 PostgreSQL | Cloud 不可用时仍可承担已交接的协调；但需要两向可靠交接、两套权威与投影规则，Controller 不能作为无状态 worker 缩放 |
| SQLite 作为云端的本地缓存或 outbox，异步同步到 Cloud | 写入延迟低；但形成两个状态源，实例磁盘丢失即丢失未同步提交 |
| 把现有 SQLite 方法逐条包装为 RPC | 改动最小；但接受与派发登记在云端不是同一事务的两半，逐条包装会把本机事务边界强加给 Cloud |
| 云端也实现 `accept_request`，返回配置错误 | 接口只有一个 trait；但错误要在运行时才暴露，JSON 接口能被组合出来却永远不可用 |
| 协调逻辑持有并传递提交身份 | 跨调用重试也能命中提交记录；但每个调用点都要感知远程适配器细节，而业务幂等已覆盖跨调用重复 |
| 接口用 trait object 动态分发 | 调用点更简单；但适配器在部署期确定，静态分发即可 |

## 本决策未解决的问题

- 两个适配器共同执行的行为契约套件，以及以真实 Cloud 为对端的自动化测试。
- 多 worker 的租约、分区与到期恢复；fencing 如何同时约束 Cloud 提交与 Node 变更指令。
- 稳定逻辑控制归属与可替换 worker 实例的身份区分；多 Node 的云端派发目标。
- 新 worker 定位原沙盒、连接原 Node 并恢复原执行的规则；沙盒的持久责任。
- Effect 级 operation 的云端协调接口；Cloud 连接的认证与 TLS。

## 落地结果

- 接口抽取、部署模式与泛型运行时（desktop#601）：SQLite 收敛为 `SqliteStore` 适配器、`CoordinationStore`
  接口、运行时／服务／API 对接口泛型、`persistence` 枚举；本机测试全部保持通过。
- Cloud 适配器与领取循环（desktop#602）：`pending_dispatches` 收敛、拆出 `CloneIntake`、错误分类补齐不可用／
  未知／资格失效；租约维持、周期领取与派发登记、事件接管、查询结果、恢复读取、错误映射与同身份重传；
  云端模式不打开 SQLite、不提供 JSON 接口。
- 与提案不同的最终决定：`record_dispatch` 不是接口方法、`CloneIntake` 取代云端的配置错误（D1）；
  跨调用重试依赖业务幂等（D3）；尚无共享行为套件（D4）。
