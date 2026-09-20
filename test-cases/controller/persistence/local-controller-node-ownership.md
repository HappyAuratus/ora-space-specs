# 本机 Controller 归属与持久接管

依据[已批准的根决策](../../../decisions/controller/persistence/0-local-controller-node-ownership.md)。
以下登记新会话与协调层的验证义务；现有 Node clone、SQLite 或消息 codec 测试不能替代
Controller–Node 真实通信证据。当前已有真实本机 IPC 与持久接管实现，以下分别记录直接证据及仍缺失的故障组合。

## Node Ownership Survives Connections And Restarts

- **风险**：首个连接者抢占归属，或重启时更换身份并误认领旧任务。
- **前置状态**：部署显式指定 Controller A，Node 保存其归属；另有 Controller B。
- **触发**：A／Node 分别重启，B 抢先连接，或启动配置与持久绑定不一致。
- **必须成立**：ControllerId、NodeId 和归属保持；Node 重启只改变 incarnation；B 不得提交、查询或确认。
  一个 Controller 可连接多个分别绑定给它的 Node，不要求一对一数量关系。
- **禁止结果**：按首个连接、PID、endpoint 或离线时长重置归属；将无秘密身份校验称为认证。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 显式归属及重启保持 | `ora-node-db::controller_binding_preserves_unclaimed_history_and_survives_restart`、Node `repository_ipc.rs` 与 `repository_controller.rs` 覆盖持久归属、不同 owner 及两侧重启 | Covered |
| 握手绑定与派发能力 | `ora-controller/tests/session.rs` 验证错误 Node 或缺少 clone 能力时拒绝派发且保留原接受记录；尚缺版本不符与多个绑定 Node 隔离的会话证据 | Partial |

决策依据：D1、D2，不变量 1、5。

## Only One Live Session Can Control A Node

- **风险**：重复连接抢占，或撤销后的在途消息仍能派发／确认。
- **前置状态**：同一 Controller 已有有效会话；Node 正在执行长时间 clone。
- **触发**：建立第二连接，关闭或超时撤销旧连接，再重连；发送半帧、停止读取或使发送队列达到上限。
- **必须成立**：有效会话存在时拒绝第二会话；撤销后旧会话未受理消息失效，新会话才可接入。
  clone 不阻塞心跳和连接生命周期；缓冲及等待有界；断连只影响会话，未确认结果持久保留。
- **禁止结果**：最后连接者抢占、两条连接同时控制、连接超时取消业务执行、背压静默丢失终态。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 单会话及撤销边界 | `local_ipc_enforces_owner_and_replays_original_clone_after_restart` 覆盖重复连接拒绝；`busy_clone_keeps_heartbeats_and_revokes_timed_out_queued_command` 在 HTTPS 暂停期间令排队 clone 超时，验证新会话查询仍为 Unknown 且原已接受 clone 完成 | Covered |
| 有界收发与执行解耦 | `repository_session.rs` 的真实 socket 测试覆盖长 clone 心跳、排队受理超时、半帧超时、持续查询但不读取回复造成背压后的断连与原结果重放 | Covered |
| endpoint 文件保护 | `ora-utils/tests/local_ipc.rs` 覆盖普通文件、符号链接、活动 socket 拒绝及失效 socket 恢复；Node IPC 重启测试覆盖部署调用 | Covered |

决策依据：D2，不变量 1、2。

## Durable Acceptance Precedes Dispatch And Retry

- **风险**：先派发后落盘，或重启／回复丢失导致第二次 clone。
- **前置状态**：独立 Controller SQLite 和真实 Node；可在接受事务、派发及回复边界中断。
- **触发**：重复请求、同身份不同输入、事务失败、Controller 强杀、派发回复丢失及 Unknown 查询。
- **必须成立**：原输入、目标 Node、操作／执行身份和待协调责任原子持久接受后才派发或报告接受。
  原请求重传返回原接受记录，冲突输入被拒绝；重启只查询／重传原执行。Unknown 不证明从未执行。
  已知失败后的明确新重试才创建新身份及新目录，旧残留保留。
- **禁止结果**：内存接受、接受等于完成、恢复另建身份、自动删除残留、Controller 本地代跑 Git。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 原子接受和幂等 | `ora-controller/tests/persistence.rs` 覆盖 SQLite 写失败、重开、重复／冲突输入；尚缺接受失败与真实网络派发的联合同步点 | Partial |
| 原执行恢复 | `repository_controller.rs` 独立 Controller 在 Ack 丢失后强杀，重开验证原关联及唯一 Run；`ora-controller/tests/session.rs` 验证 Unknown 每连接只重传原命令一次；尚缺派发回复丢失强杀组合 | Partial |

决策依据：D3，不变量 2、3。

## Queries And Replayed Events Share One Result Takeover

- **风险**：查询和事件重复推进状态，或由查询回复编造 Ack。
- **前置状态**：Node 有原终态及未确认事件；当前 incarnation 可与结果中的历史 incarnation 不同。
- **触发**：Completed 查询和事件分别先到，重复交付；再提供错误 Node、错误派发归属及同身份不同结果。
- **必须成立**：两条路径统一接管同一结果；历史身份保留，同内容幂等，冲突不覆盖、不确认。
  查询可保存结果，但不推导事件序号；实际收到原事件后另完成精确事件接管与确认。
- **禁止结果**：按当前 incarnation 改写历史、查询停止重放、重复推进状态或重复创建后续责任。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 乱序与幂等接管 | `takeover_is_atomic_in_both_delivery_orders_and_conflicts_never_ack` 覆盖真实 SQLite 两种到达顺序、重复／重开及查询无 Ack；真实 Node IPC 测试覆盖确认后查询 | Covered |
| 原归属与冲突拒绝 | 同一 takeover 测试覆盖历史 incarnation、request 关联与结果内容冲突；尚缺错误 Node／能力及所有派发身份组合 | Partial |

决策依据：D4，不变量 3、4。

## Acknowledgement Follows Durable Takeover Across Crashes

- **风险**：提前 Ack 使 Node 删除唯一交付证据，或 Controller 崩溃丢失后续处理责任。
- **前置状态**：真实独立 Controller、Node、host／guardian 和 Git；结果等待交付。
- **触发**：接管事务失败或结果未知；在提交前后、Ack 发送前后强杀 Controller；Node 在结果落盘后投递前强杀。
- **必须成立**：未确定持久提交时不 Ack；已提交结果、事件去重及必要后续责任可恢复。
  Ack 丢失后重放原事件，只重复确认，不重复业务推进；Node 重启不改变原执行或事件身份。
  连接断开不取消已受理 clone，也不无依据地报告业务失败。
- **禁止结果**：收到即 Ack、依赖内存回调补写、重连重新 clone、只用正常 drop 或 fake 证明强杀恢复。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接管提交与确认顺序 | SQLite receipt 写失败回滚；`killed_controller_before_commit_replays_without_losing_node_responsibility` 在独立子进程的生产会话／真实 SQLite 接管事务内注入提交前暂停，确认无 Ack 后 SIGKILL，重开结果为空，再由正常 Controller 程序恢复；`independent_controller_replays_durable_takeover_after_lost_ack_and_kill` 覆盖提交后 Ack 丢失强杀 | Covered |
| 双侧独立恢复 | `repository_controller.rs` 与 `repository_ipc.rs` 分别覆盖真实 HTTPS 下 Controller／Node 强杀；尚缺 SSH 与全部中断位置组合 | Partial |

决策依据：D3、D4，不变量 2、3、4。

## New Ownership Preserves Existing Files And Unclaimed History

- **风险**：新库或归属迁移覆盖现有状态，握手隐式导入历史任务。
- **前置状态**：显式注入各数据目录；已有未知文件、旧 Node schema、未确认历史结果和用户 checkout。
- **触发**：配置路径重叠、两个 Controller 打开同库、升级 Node、尝试确认没有 Controller 派发记录的结果。
- **必须成立**：数据目录隔离、Controller 库单运行所有者；未知文件拒绝初始化。
  识别旧 schema 后安全迁移，保留原执行／结果／文件；无匹配派发责任的历史保持未接管。
  Controller 数据丢失不解除 Node 责任，不自动重绑定。
- **禁止结果**：清库、改历史身份、补造请求、删除无法认领的事件、隐式切换 Backend 写入入口。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 路径和库所有权 | Controller persistence 测试覆盖注入路径、未知文件和同库独占；尚缺 Controller 部署目录重叠及跨进程竞争专项 | Partial |
| 迁移及未认领历史保护 | Node schema v3→v4 测试保留原记录并拒绝认领；v1／v2 回归保留；尚缺 Controller 数据丢失后历史事件的真实会话联合验收 | Partial |

决策依据：D1、D3、文件兼容与风险，不变量 1、5。
