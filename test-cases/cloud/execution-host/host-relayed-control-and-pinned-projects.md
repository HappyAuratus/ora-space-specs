# 多机执行层：宿主转发、断开即中断、项目固定与令牌

依据 [多机执行层根决策](../../../decisions/cloud/execution-host/0-host-relayed-control-and-pinned-projects.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `ad324c77` 的 `v2-server/`（路径相对该目录）。执行宿主、项目放置和项目令牌都还没有实现，
相关义务如实标为 **Missing**；现有测试只覆盖转发链路两端的 Node 协议行为，标为 **Partial**。

## The Host Relays Node Frames Unchanged Over One Authenticated Connection

- 风险：宿主改写、丢弃或乱序转发 Node 的帧，Cloud 看到的执行事件和 Node 记录的不一致；未经认证的程序冒充一台机器连上 Cloud。
- 前置状态：一台机器上有多个 Node 容器；管理员已为这台机器签发宿主令牌。
- 触发：宿主用有效、缺失、已吊销的宿主令牌连接；同一台机器第二次连接；多个 Node 同时收发帧。
- 必须成立：只有带有效宿主令牌的连接被接受；同一台机器的新连接顶掉旧连接；每个 Node 的帧按原顺序、原内容到达对端；
  Cloud 侧仍按 Node 协议完成握手、心跳和显式 ACK。
- 禁止结果：无令牌或已吊销的连接被接受；一台机器同时有两条连接；帧被改写、合并或跨 Node 串路。
- 验证义务与证据：**Partial** — `internal/node/client_test.go` 的 `TestClientExplicitAckAndHeartbeats`、`TestDialRejectsWrongNode`、
  `TestReceiveRejectsChangedIncarnation`、`TestCancellationClosesPartialFrame`。已覆盖：Cloud 侧的 Node 协议客户端在本机
  Unix socket 上的握手、身份校验、心跳、显式 ACK 和半帧处理。仍缺：宿主程序、宿主令牌和多 Node 分路都还没有实现（I4 #1054）。
- 决策依据：D1、D2、D6，不变量 1、2。

## One Congested Node Channel Does Not Stall The Machine

- 风险：一个 Node 发不出去的帧堆满整条宿主连接，同一台机器上其他项目的执行跟着卡住或一起被中断。
- 前置状态：同一台机器上有两个项目的 Node 在执行。
- 触发：让其中一个 Node 的发送队列持续积压，直到超过上限。
- 必须成立：只有积压的那一路被断开，按 Node 退出处理；另一个项目的执行照常进行。
- 禁止结果：整条连接被断开；另一个项目的帧被阻塞到超时。
- 验证义务与证据：**Missing** — 宿主的分路还没有实现（I4 #1054）。
- 决策依据：D2。

## Losing The Host Connection Interrupts, Revokes And Stops On Both Sides

- 风险：Cloud 已经把执行标为中断，Node 却还在跑 Agent、推代码、调模型；或者宿主已停掉容器，Cloud 仍显示执行进行中。
- 前置状态：一台机器上有正在执行的 Node 和预热中的空闲 Node。
- 触发：宿主心跳超时；连接被关闭；同一台机器发起新连接把旧连接顶掉；单个 Node 进程退出。
- 必须成立：宿主断开时，这台机器上所有未结束的执行都标为中断，所有项目令牌都吊销，宿主停掉所有 Node 容器并保留数据卷；
  单个 Node 退出时只影响这一个 Node。
- 禁止结果：断开后仍有执行处于进行中；被吊销前的项目令牌在断开后还能访问 Git 或模型授权；宿主断开后容器仍在运行；
  数据卷被删除。
- 验证义务与证据：**Partial** — `internal/agentruntime/driver_ipc_test.go` 的 `TestDriverIPCDisconnectRequiresStopProofBeforeRebuild`。
  已覆盖：单机下 Node 连接断开时当前轮次结束，确认旧 Node 停止前不重建。仍缺：当前实现把它记为失败而不是中断（见
  [Cloud 单进程用例](../lifecycle/single-process-cloud.md#losing-a-node-host-or-machine-interrupts-only-that-machines-executions)）；
  宿主断开、令牌吊销、宿主停容器都还没有实现（I4 #1054）。
- 决策依据：D3，不变量 5。

## Replayed Events For Ended Executions Do Not Change Business Records

- 风险：Node 重启后重放没确认的事件，Cloud 把已经标为中断的执行改回进行中，或者把迟到的输出追加进已结束的会话。
- 前置状态：执行已被 Cloud 标为中断；Node 的数据库里还有这个执行未确认的事件。
- 触发：Node 在同一个数据卷上重新启动，握手后重放这些事件。
- 必须成立：Cloud 对这些事件照常 `EventAck`，Node 不再重放；业务记录不变。
- 禁止结果：执行状态被改写；会话里多出内容；Cloud 不 ACK，导致 Node 一直重放。
- 验证义务与证据：**Missing** — 没有直接测试；`internal/node` 只测了显式 ACK 本身。
- 决策依据：D3，不变量 6。

## A Project Stays On One Machine Until Its Grace Period Expires

- 风险：同一个项目同时在两台机器上有 Node；机器短暂失联就被迁走，白白重建；或者机器永久丢失后，项目一直不可用。
- 前置状态：多台在线机器，各有容量；一个已固定在机器 M1 上的项目。
- 触发：首次使用项目；M1 断开后在宽限期内重连；M1 超过宽限期仍未重连；管理员停用 M1；机器容量已满；某台机器缺少需要的镜像。
- 必须成立：首次使用时放到有镜像、有容量、项目最少的机器；宽限期内的新执行直接返回"执行机器不可用"；超过宽限期后解除
  固定，下次使用时在别的机器上重建；停用的机器不接收新项目。
- 禁止结果：同一项目在两台机器上都有运行中的 Node；宽限期内被重新放置；新执行在宽限期内排队等待；项目被放到缺镜像或已满的机器。
- 验证义务与证据：**Missing** — 现在只有单机本地子进程，没有放置逻辑（I4 #1054）。
- 决策依据：D4，不变量 4、10。

## A Reconnecting Host Is Reconciled From Cloud Records Only

- 风险：宿主重连后，留着不该在跑的容器，或者删掉了不属于 Cloud 的容器和数据卷。
- 前置状态：宿主断开期间，某些执行被标为中断，某个项目被重新放置到别的机器；机器上另有不带 Cloud 标签的容器。
- 触发：宿主重启或重连，在注册消息里上报本机的容器和数据卷。
- 必须成立：不该在跑的容器被停掉；不再属于这台机器的项目的数据卷被删除；不带 Cloud 标签的容器和数据卷保持不动并记录告警；
  宿主自己没有保存任何执行、项目或调度状态。
- 禁止结果：重连后旧执行恢复运行；误删不属于 Cloud 的资源；宿主依赖本地数据库决定留下什么。
- 验证义务与证据：**Missing** — 宿主还没有实现（I4 #1054）。
- 决策依据：D5，不变量 7。

## A Project Token Reaches Only Its Own Project And Dies With Its Container

- 风险：一个项目的 Node 用令牌访问到别的项目的 Git 或模型授权；容器停止后，旧令牌仍然可用；令牌明文进入日志、checkout 或数据库。
- 前置状态：两个项目各有一个运行中的 Node 容器。
- 触发：用 A 项目的令牌访问 B 项目的仓库和模型授权；A 的容器停止后继续用它的令牌；轮换令牌；令牌文件缺失或格式错误。
- 必须成立：令牌只能访问所属项目；容器停止、通道断开、宿主断开或项目换机器时令牌立即失效；轮换后 Node 下一次请求就用新令牌，
  不用重启；Cloud 只保存令牌的哈希；令牌读不到时请求失败，错误里没有令牌内容。
- 禁止结果：跨项目访问成功；停止后的令牌仍然有效；令牌出现在 checkout、git 配置、日志、journal 或数据库明文里。
- 验证义务与证据：**Missing** — Node 和 v2-server 都还没有改为令牌（circus #1066）；Git 和模型授权的范围检查分别由 A4、A5 的用例细化。
- 决策依据：D6、D7，不变量 8、9。
