---
status: implemented
---

# 本机 Controller 持久接管执行，Node 固定归属单一 Controller

本机首版中，一个 Node 只归属一个持久 `ControllerId`，同一时刻只有一个有效控制会话。
Controller 先持久保存派发责任，再请求 Node 执行；结果持久接管后才确认。两侧可以独立重启，
断连不取消已受理执行，也不产生新执行身份。

本 ADR 处于 `implemented`：Node 侧归属记录在 desktop 的 `crates/node-db`（`ControllerMismatch`），Controller 的
本机持久协调是 `apps/ora-controller` 的 `SqliteStore`，验证见
[本机归属与结果接管用例](../../../test-cases/controller/persistence/local-controller-node-ownership.md)。本决策覆盖 Linux
本机 clone 协调，不接 Desktop UI，不切换 Backend 写入入口，不引入密钥、令牌、证书或 Cloud 权威存储。持久化接口
后来抽象为 `CoordinationStore`，本文规则由 SQLite 适配器承担（见
[持久协调双适配器](20260922-coordination-store-with-sqlite-and-cloud-adapters.md)）。

## 依据与继承

这是 Controller 本机持久协调的根决策。

| 来源与约定 | 本文处理 |
|---|---|
| [clone 执行契约](../../node/repository/20260918-minimal-clone-execution-contract.md)：Node 执行、原身份恢复、结果持久化 | 继承，不变；Controller 不在自身环境代跑 Git |
| [Controller–Node Protocol](../../node/protocol/0-controller-node-protocol.md)：统一消息、查询与确认分离 | 采用其本机可靠性语义；帧编码与本机 IPC 见 [FrameChannel 传输](../../node/protocol/20260924-frame-channel-with-ipc-and-websocket.md) |
| 同协议中的 bootstrap credential、mTLS 和多种远端连接器 | 本机首版不采用；归属校验不是认证，远端与信任体系另行决策 |
| Cloud 场景由 Cloud 持久接管（[Cloud 拥有的 gRPC 契约](../../cloud/controller-integration/0-cloud-owned-internal-grpc-contract.md)） | 不在本文范围；本机由 Controller 自身持久存储接管 |
| 现有 Backend | 保留，不迁移其业务入口；本机协调只派发 clone（Worktree 已被放弃，见 [clone 根决策](../../node/repository/0-clone-selected-repository-branch.md) D3） |

## D1：归属固定，连接可替换

`ControllerId` 标识持久协调者，不标识进程或 socket。Controller 首次初始化保存身份，重启沿用；
Node 的 `NodeId` 同样持久，`NodeIncarnationId` 随 Node 重启改变。
一个 Controller 可以管理多个 Node，但一个 Node 只绑定一个 Controller。

归属由部署显式指定并在 Node 持久保存，不能由第一个连接者抢占。重启配置须与原绑定一致；
不同 `ControllerId` 的握手被拒绝，不得提交命令、查询或确认事件。本文不支持重新绑定、
Controller 数据丢失后的自动接管，也不因旧 Controller 长期离线而转移归属。

这防止两个可信部署误用同一 Node，并非防止恶意程序冒用身份。`ControllerId` 不是秘密，
同 UID 的恶意代码、身份伪造和跨机器认证不由此保证。

## D2：单一有效会话，不抢占仍有效的连接

Node 在显式注入的本机 Unix socket endpoint 监听，Controller 主动连接；协议流不使用日志标准输出，
Node 生命周期不依赖 Controller 的 stdin 或进程存活。握手校验版本、绑定身份和实际能力；
Controller 同时校验预期 `NodeId`，只向声明 clone 能力的 Node 派发 clone。

同一 Controller 的重复连接也不能形成两个有效控制会话：已有有效会话时拒绝新会话，
不以“最后连接者获胜”隐式抢占。原连接关闭或经有限的会话超时撤销后才能重连；
撤销后旧连接的未受理消息不能继续派发或确认。超时只撤销连接，不终结业务执行。

执行与会话收发解耦：长时间 clone 不阻塞心跳与连接生命周期。连接和发送队列有界；
慢读、半帧或无法继续发送时可以关闭连接，但未确认结果仍留在持久记录中供重放。
endpoint 由本机权限保护，不增加应用秘密；不覆盖普通文件、符号链接或其他仍在使用的 socket。

## D3：派发责任先持久化，断线沿用原身份

Controller 的数据目录显式注入，不从 HOME、当前目录或 Node 路径推导；本机采用独立 SQLite 存储，
同一库同时只有一个运行所有者，不与 Backend 或 Node 共享数据库。持久保存：

- Controller 身份及目标 Node 关联；
- 原请求、`operation_id`、`execution_id` 和完整 clone 输入；
- 待协调责任、执行结果及事件接管记录。

接受边界先原子保存输入、身份和待协调责任，再报告接受；接受不表示 Node 已收到或 clone 成功。
原请求重传返回原接受记录，身份相同但输入不同则拒绝。重启、断线和派发回复丢失都沿用原身份，
不自动生成新的 clone 尝试。

重连后查询原执行，并允许重传原命令；Node 的持久去重决定是否首次接受或返回原状态。
查询得到 Unknown 不证明从未执行，不能换身份绕过。明确失败后的用户重试才使用新操作／执行，
由 Node 分配新目录。Controller 不删除 Node 残留，也不把 Node 路径解释为本机可访问路径。

## D4：结果和事件共用接管边界，确认只针对原事件

Controller 接收结果时验证会话 Node、原派发身份、原输入及资源事实的关联。历史结果中的
`NodeIncarnationId` 可以与当前会话不同，不得改写历史身份。

查询的 Completed 和主动重放事件共用结果接管逻辑：同执行同内容幂等，内容冲突明确拒绝。
执行事实与调用方可查询状态在同一事务提交；若需后续处理，责任也须持久化，不能只保留内存回调。
首次事件接管另保存原事件精确身份，提交成功后才发送 `EventAck`。

查询结果不提供原事件序号，不能据此构造 Ack，也不能让 Node 停止重放。查询先到时可以先保存
结果；原事件后到仍须完成事件接管，再按原序号确认。确认丢失时重复事件只重新确认，
不再次推进状态或产生第二份后续责任。

事务失败、提交结果未知或结果归属冲突时不确认；断线不把执行改成业务失败。
Controller 重启后能从自身记录及 Node 原事件恢复，不依赖上一个进程的回调。

## 不变量

1. Node 的持久归属不因连接、超时或进程重启改变；同时至多一个有效控制会话。
2. 已受理执行独立于连接存活；重连不另建执行，不自动取消或清理现场。
3. Controller 派发责任先于网络副作用，结果接管提交先于事件确认。
4. 查询、重放及重复确认不会产生重复业务结果，冲突内容不会覆盖原事实。
5. 无密钥的归属校验不被描述为认证；本机 SQLite 不被扩展为 Cloud 权威存储。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 多个 Controller 同时控制一个 Node | 可支持高可用，但需要协调权、分区和确认归属；首版没有该需求 |
| 新连接自动顶替旧连接 | 重连更快，但旧连接在途命令和确认需要额外移交规则；首版显式关闭旧会话后重连 |
| 只在内存记录派发和接管 | 实现更短，但重启会丢失执行关联，无法证明何时可以确认 Node |
| 收到事件就 Ack，随后保存结果 | 降低确认延迟，却可能使唯一结果在两侧崩溃窗口中丢失 |

## 文件兼容与风险

新增 Controller 数据目录必须与 Node／host／guardian 状态隔离；已有未知文件不能当作新库初始化。
Node 新增归属记录需要识别旧 schema、安全迁移并保留旧执行／结果，不清空数据库。
既有无 Controller 归属的历史执行不能仅凭握手被认领或确认；没有匹配原派发记录的结果保持未接管，
不补造 Controller 请求、改变历史身份或删除事件。历史任务导入留待独立决策。

Controller 数据丢失不等于 Node 责任消失；首版可能需要保留未接管事件并暂停受影响任务。
单会话拒绝抢占会使异常连接的重连等待受超时限制，这是避免并行控制的明确代价。

## 后续边界

重新绑定／历史导入、多 Controller 高可用、认证与远端传输、Desktop UI／订阅以及 Backend 入口切换均不由本文
实现。Cloud 场景的持久化与认证分别由 Cloud 契约和运行时控制决策承担。

## 落地

本机 Node 会话（绑定、单会话、断连不取消、重连重放、endpoint 文件保护）、Controller 持久协调（原输入接受、
派发前落盘、查询与事件统一接管、提交后确认）与独立进程验收（真实 IPC、SQLite、HTTPS／SSH Git）已合入。
仍缺的故障组合登记在核心用例中。完成本切片不代表 Client 可观察的完整产品闭环已完成。
