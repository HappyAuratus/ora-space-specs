---
status: proposed
---

# 持久协调接口与双适配器：本机 SQLite、云端经 Cloud RPC，Controller 不持有云端权威状态

Controller 的持久化操作经过一层"持久协调接口"：接口按完整原子业务操作定义（接受请求、登记
派发、接管 Node 事件、保存查询结果、恢复读取），不暴露 SQL、表或跨调用的事务。接口有两个
适配器：本机 single-node 场景使用现有 SQLite 实现；云端场景使用 Cloud RPC 适配器，把每个操作
翻译为对 Cloud gRPC 契约的调用，由 Cloud 在 PostgreSQL 事务中提交。适配器在部署时选定，运行时
不切换；云端 Controller 不打开 SQLite，其连接、缓存与在途协调都是可丢弃、可从 Cloud 重建的运行时
状态。第一版接口只覆盖 clone 闭环，与根决策现有的本机规则逐项对应。

本决策处于 `proposed` 状态，基于[根决策](0-local-controller-node-ownership.md)：根决策的本机
规则（单 Controller 归属、派发前落盘、接管后 Ack、原身份恢复）全部继承，只把"由 Controller
自身的 SQLite 持久"改为"由持久协调接口的当前适配器持久"。契约内容由
[Cloud 拥有 Internal Control gRPC 契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)
定义，契约的归属与调用方向由
[Controller 契约方向后续决策](../api-boundary/20260922-cloud-owned-contract-and-controller-dial-out.md)
定义；本文只回答 Controller 内部如何把协调逻辑与持久实现分开、两个适配器各自承诺什么。它不改变
本机 Node 固定绑定 `ControllerId` 的规则，不设计多 worker 租约、fencing 到 Node 指令、沙盒绑定或
Effect 级 operation 的云端协调。

## 继承与修改

| 根决策约定 | 本文处理 |
|---|---|
| D1：Node 固定归属单一 `ControllerId`，连接可替换 | 继承，不变；云端 worker 身份与逻辑控制归属的区分留给 scale 决策，本文不重解释该规则 |
| D2：单一有效会话，本机 Unix socket，握手校验身份与能力 | 继承，不变 |
| D3：数据目录显式注入，独立 SQLite，同一库单一运行所有者 | 修改范围：这些规则只属于 SQLite 适配器；云端部署没有数据目录与租约文件 |
| D3：接受先原子保存输入、身份与待协调责任；派发前持久化 | 继承，改由接口承担：`accept_request`／`record_dispatch` 成功后才报告接受或派发 |
| D4：结果与事件共用接管边界，提交成功后才 `EventAck`；查询结果不构成 Ack 依据 | 继承，改由接口承担：`take_over_node_event` 成功后才 Ack；`record_queried_result` 不返回 Ack 依据 |
| D4：事务失败、提交结果未知、归属冲突时不确认 | 继承并细化："提交结果未知"成为远程适配器的显式错误分类（D3） |
| 不变量 5：本机 SQLite 不被扩展为 Cloud 权威存储 | 继承，不变；SQLite 只服务本机部署，云端不使用 |
| Internal Control Protocol D5：Controller 通过 API Server 适配的 `EffectRepository` 读取 | 不采用其名称；本文的接口是 clone 闭环的持久协调操作，Effect 级适配留给后续 |

## 决策

### D1：接口按原子业务操作定义，读与恢复也经过接口

接口以异步方法表达，每个方法对应一个在权威存储中原子完成的业务边界；下表的方法名用于说明
语义，代码中的命名在落地时固定：

| 方法 | 原子保存 | 成功的意义 | 幂等规则 |
|---|---|---|---|
| `accept_request(request_id, spec)` | 原请求、`operation_id`、`execution_id`、目标 Node、完整输入 | 可以向调用方报告接受 | 同 `request_id` 同输入返回原记录；不同输入冲突 |
| `record_dispatch(execution)` | 派发责任（本机与接受同事务，云端由 Cloud 已接受的工作转为登记） | 可以向 Node 发送该执行 | 同执行同输入幂等；不同输入冲突 |
| `take_over_node_event(session, event)` | 执行结果、事件收据 `(execution, sequence)`、后续责任 | 可以向 Node 发送该序号的 `EventAck` | 同序号同内容幂等；不同内容冲突 |
| `record_queried_result(session, execution, result)` | 由查询得到的 Completed 结果 | 结果可查询 | 同内容幂等；不同内容冲突；不产生 Ack 依据 |
| `original_dispatch(node, operation, execution)` | 无 | 恢复原命令以重传或校验归属 | 缺失为冲突，不补造 |
| `pending_dispatches(node)` | 无 | 重连后查询与重放的依据 | — |
| `operations()`／`operation(execution)` | 无 | 调用方查询；缺失与无终态区分 | — |

接口不暴露 `begin`／`commit`、连接、表名或 SQL；协调逻辑不能通过组合两次调用来"补齐"一个原子
边界。这样两个适配器才能各自用本地事务或一次 RPC 提交实现同一承诺，而不需要跨调用的分布式事务。
接口不是把现有 SQLite 方法逐条包装：现有 `accept_clone` 把接受与派发登记合在一个事务里，本机
适配器继续如此，但接口把它们表达为两个语义，因为云端的接受发生在 Cloud 的业务事务里，Controller
只做登记。

### D2：适配器在部署时选定，运行时不切换，不互为后备

部署配置以枚举表达持久模式：本机模式给出数据目录（SQLite 适配器，继承根决策 D3 的全部文件与
租约规则）；云端模式给出 Cloud 端点与凭据（Cloud RPC 适配器）。启动时只构造一个适配器；云端模式
不创建数据目录、不打开 SQLite、不取文件租约。

Cloud 不可用时，Controller 不能切换到本地 SQLite 代写：那会形成第二个权威状态源，恢复时无法判定
哪一份是事实。正确行为是保持未派发、未确认，按 D3 的分类重试或等待。云端 Controller 的内存缓存、
Node 连接与在途协调可以随进程丢失，新实例只依赖 Cloud 的持久记录（`pending_dispatches`、
`original_dispatch`）与 Node 的原事件重放恢复，不依赖原实例的磁盘。

### D3：错误分类区分冲突、不可用与未知；未知只能同身份重传

接口的错误分类与 Controller 行为：

| 分类 | 含义 | 协调逻辑的行为 | SQLite 适配器 | Cloud RPC 适配器 |
|---|---|---|---|---|
| 冲突 | 身份或内容与权威记录不符 | 不派发、不确认、不重试同一调用 | 事务内校验 | Cloud 返回 `ABORTED`／`FAILED_PRECONDITION`／`INVALID_ARGUMENT` |
| 缺失 | 无该身份的记录 | 与"无终态"区分 | 查询为空 | `NOT_FOUND` |
| 不可用 | 未接受，可重试 | 保持未派发／未确认，稍后重试 | I/O 或 SQLite 错误（事务未提交） | `UNAVAILABLE`、连接失败 |
| 未知 | 可能已提交但回复丢失 | 用同一提交身份重传；返回原结果或冲突后再决定 | 不产生（本地事务要么提交要么回滚；进程崩溃由重启后读取解决） | 截止时间到期、回复前断连 |
| 资格失效 | 租约／epoch 陈旧 | 停止使用该资格，重新获取后再继续 | 不产生 | `FAILED_PRECONDITION` + 陈旧类枚举 |

Cloud RPC 适配器为每个写操作生成稳定的提交身份，并在"未知"后用同一身份重传；协调逻辑不感知
提交身份的存在。SQLite 适配器不需要提交身份，但两者对外的幂等承诺相同（D1 表）。

### D4：两个适配器验证同一行为契约，远程另验证未知结果

接口的行为契约以一套共享的测试套件表达，对 SQLite 适配器与 Cloud RPC 适配器（以真实 Cloud
gRPC 服务端或行为等价的测试服务端为对端）分别执行：接受幂等与冲突、派发登记先于派发、接管先于
Ack、查询结果不产生 Ack、缺失与无终态区分、崩溃后恢复原身份。远程适配器另验证：回复丢失后同身份
重传返回原结果、内容变化后重传被拒绝、Cloud 不可用期间不派发不确认、资格失效后停止提交。

现有以 `WriteGuard`／`WritePoint` 注入 SQLite 事务内故障点的测试缝继续属于 SQLite 适配器；
远程适配器的故障注入发生在测试服务端（提交后丢弃回复、返回 `UNAVAILABLE`、返回陈旧资格）。
真实链路验收（IPC、SQLite、Git、强杀）沿用根决策落地顺序 3，并增加云端模式的等价场景。

### D5：第一版范围是 clone 闭环，本机行为不变

第一版接口只覆盖 D1 表列出的 clone 闭环操作；本机模式的调用方可观察行为、数据目录布局、
schema 与文件租约规则不变，现有本机测试作为 SQLite 适配器的直接证据。云端模式的接受入口由
Cloud 的公开 API 承担，Controller 通过 Cloud 契约的领取与 `Watch` 信号获得工作，再走
`record_dispatch` 及之后的操作；本地与云端共享持久协调语义，不要求请求入口相同。

worktree、sandbox、Effect 级 operation 的云端协调不在第一版：Cloud 现有的 Effect 级方法
（快照、effect 规划与结果）由后续决策决定是并入本接口还是另建接口。

## 不变量

1. 协调逻辑只通过持久协调接口读写持久状态；接口方法各自原子，不存在跨方法的事务组合。
2. 每个部署只有一个适配器；Cloud 不可用时不切换到本地写入，不产生第二个权威状态源。
3. `record_dispatch` 成功先于任何 Node 派发；`take_over_node_event` 成功先于任何 `EventAck`；
   `record_queried_result` 不产生 Ack 依据。
4. 同身份同内容幂等，同身份不同内容冲突；"未知"结果只能用同一提交身份重传，不能换身份。
5. 云端 Controller 的运行时状态可丢弃；新实例仅凭 Cloud 持久记录与 Node 原事件恢复，不创建第二次执行。
6. 本机 SQLite 只服务本机部署；根决策的本机归属、会话与文件规则不因本文改变。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| Controller 独立拥有云端 PostgreSQL | Cloud 不可用时仍可承担已交接的协调；但需要 Cloud→Controller 与 Controller→Cloud 两向可靠交接、两套权威与投影规则，Controller 不能作为无状态 worker 缩放 |
| SQLite 作为云端的本地缓存或 outbox，异步同步到 Cloud | 写入延迟低；但形成两个状态源，实例磁盘丢失即丢失未同步提交，与"运行时状态可丢弃"冲突 |
| 把现有 SQLite 方法逐条包装为 RPC | 改动最小；但接受与派发登记在云端不是同一事务的两半，逐条包装会把本机事务边界强加给 Cloud |
| 接口用 trait object 动态分发 | 调用点更简单；但两个适配器在部署期确定，静态分发即可，且便于分别为两者编译不同依赖 |
| 只用一套 Cloud 适配器，本机也起 Cloud | 一种实现；但本机 single-node 需要无网络、无 Go 进程的可运行形态，这是 minicloud 与 Desktop 场景的硬约束 |

## 本决策未解决的问题

- 多 worker 的租约、分区与到期恢复；fencing 如何同时约束 Cloud 提交与 Node 变更指令。
- 稳定逻辑控制归属与可替换 worker 实例的身份区分；本机固定绑定规则在云端的对应物。
- 新 worker 定位原沙盒、连接原 Node 并恢复原执行的规则；沙盒创建、绑定与回收的持久责任。
- Effect 级 operation 的云端协调接口。
- Cloud 连接的认证凭据与 TLS。

## 落地顺序

1. **接口抽取（只接 SQLite）**：定义接口与错误分类，把现有 `storage`／`takeover`／`operations`
   的 SQL 下沉为 SQLite 适配器，运行时与会话改为异步调用接口。完成条件：本机行为不变，现有
   本机测试全部通过，共享行为契约套件对 SQLite 通过。行为不变：全部本机可观察行为。
2. **Cloud RPC 适配器**：基于锁定的 Cloud 契约生成物实现适配器，含提交身份、未知结果重传、
   错误映射。完成条件：共享套件对远程适配器通过；未知结果与资格失效场景通过。
   从此阶段起改变：云端模式下 `ora-controller` 不打开 SQLite。
3. **拨出任务与部署模式**：部署配置的持久模式枚举、`Watch` 长驻任务、关闭顺序。完成条件：
   云端模式下 clone 闭环以真实 Cloud 为对端跑通，强杀 Controller 后新实例恢复原执行。
4. **进入 `approved` 后**：在 `test-cases/controller/persistence/` 登记两种适配器的共享义务与
   远程特有义务，并同步根决策核心用例中"由 Controller 自身 SQLite 持久"的表述。
