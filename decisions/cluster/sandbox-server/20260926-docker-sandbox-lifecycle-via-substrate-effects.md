---
status: implemented
---

# Sandbox Server 用 Docker 实现 Cloud 的 Substrate effects 接口，按 Workspace 创建与终止沙盒

> 历史决策：D1 的“生命周期接口只靠网络隔离”与 D4 的“Node 监听明文 `websocket`”已被后续决策修改。
> [普通停止与管理员强停](../../cloud/operation/20260927-safe-stop-and-administrative-force-stop.md)及
> [运行时控制同步交付](../../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)要求生命周期接口
> 认证 Controller、校验 Cloud 短期许可并持久化运行时围栏；
> [Controller 接入沙盒网络并以双向 TLS 直连 Node](20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md)
> 把 Node 监听改为双向 TLS。本文其余规则不变。

Cloud 的 Workspace 操作走到 sandbox 阶段时，Controller 以 Cloud 已有的 Substrate 最小接口
（`GET/PUT /effects/{effectId}`）请求 Sandbox Server；Sandbox Server 用 Docker 为该 Workspace 创建一个运行
`ora-node` 的容器，并带上[根决策](0-docker-label-resolver.md) D1 的路由标签。终止时停止并删除容器，只在
Docker 确认容器已不存在后报告终止。Node 的 home 放在按 Workspace 命名的卷上，沙盒终止后保留，下一代沙盒
以同一 Node 身份继续使用。仓库由 Node 经 Controller–Node 协议 clone，Sandbox Server 不做任何 Git 操作。

本 ADR 处于 `implemented`：实现位于 cluster 的 `sandbox-server/internal/lifecycle`（`lifecycle.go`、`effects.go`、
`journal.go`、`locks.go`）与 `sandbox-server/internal/docker`，验证见
[Docker 沙盒生命周期核心用例](../../../test-cases/cluster/sandbox-server/docker-sandbox-lifecycle.md)。兼容性边界：
Cloud 的 Substrate 接口语义（`docs/execution-contract.md`“Substrate 最小接口”）以 Cloud 为准，本文只实现、
不修改；根决策的标签、地址与状态码规则不变。本文不设计 suspend/resume、资源配额与镜像拉取。

## 继承与修改

| 根决策（Docker 标签解析）的约定 | 本文处理 |
|---|---|
| D1：以三个标签声明沙盒身份 | 继承，不变；本文创建的容器带这三个标签，另加只供生命周期使用的标签（D3） |
| D2：只使用 `-docker-network` 上的容器 IP，Node 不发布端口 | 继承，不变；创建时只接入该网络 |
| D3：解析实时查询、状态码映射固定 | 继承，不变；创建中、已停止的容器按原表返回 `503` |
| D4：Resolver 只读 Docker | 继承并补全：Resolver 仍只读；生命周期由本文的独立组件负责，两者只经标签交汇 |
| D5：最小 Engine API 客户端，不引入 Docker SDK | 继承并扩展：同一客户端增加创建、启动、停止、删除容器与卷的写端点（D6） |
| D6：`-routes` 与 `-docker-network` 二选一 | 继承；生命周期接口只在 Docker 模式下启用（`-lifecycle-addr` 要求 `-docker-network`） |

## 决策

### D1：实现 Cloud 已有的 Substrate effects 接口，不另定生命周期 API

Sandbox Server 在独立于中继的监听地址（`-lifecycle-addr`）上提供：

| 方法 | 语义（与 Cloud 契约一致） |
|---|---|
| `GET /effects/{effectId}` | `404` 表示确知没有该 intent；否则返回原请求、`externalId`、`state`、`result`；查询不创建任何东西 |
| `PUT /effects/{effectId}` | 首次先落 journal 再执行；同 ID 同请求幂等，同 ID 不同请求 `409`；成功后原结果保留不变 |

只接受 `sandbox_ensure`、`sandbox_terminate` 与 `workspace_data_delete` 三种 `kind`，与
[运行时 Workspace 生命周期](../../cloud/operation/0-workspace-runtime-follows-desktop-node.md)保留的 effect 一致。
其他 `kind` 以 `400 unsupported_kind` 拒绝且不写 journal。effect ID、`projectId`、`workspaceId` 与终止目标
`sandboxInstanceId` 必须是 UUID，否则 `400` 且不写 journal。请求以键序无关的规范形式保存，只调整键序的重试
仍是同一请求。执行失败的 `PUT` 返回 `503` 与失败条目，同一请求再次发送会重新执行。

为什么复用 Cloud 的接口：

- Cloud 已经为它建好了持久 plan、稳定幂等键、epoch 对账与 generation fencing。另定一套 API，Controller 就要在
  两种生命周期语义之间翻译，也会与生产 Substrate 适配器分叉。
- 生产适配器“可以映射到其控制 API，但必须保持语义”；Sandbox Server 正是这样一个适配器，只是后端换成
  Docker。Controller 面对本地与平台时只差接口地址。
- 独立监听地址而不是与中继共用端口：中继把任何带 `ate-target-actor` 的请求原样转给 Node，生命周期路径与之
  共存会让路由依赖请求头的隐式区分。

### D2：sandbox 的外部身份就是 effect ID，由它推出容器名、标签与寻址

Cloud 以 `sandbox_ensure` 的 effect ID 作为 `sandbox_instances.id`，并要求成功结果的 `sandboxInstanceId`
等于该 ID。Sandbox Server 因此直接以该 UUID 作为 sandbox ID：

| 用途 | 值 |
|---|---|
| 成功结果 | `{ "sandboxInstanceId": <effectId>, "nodeId": <D4 的 NodeId> }`，`externalId` = `<effectId>` |
| 容器名 | `ora-sandbox-<effectId>` |
| 路由标签 | `space.ora.sandbox.atespace` = `-atespace`（默认 `local`），`space.ora.sandbox.id` = `<effectId>`，`space.ora.node.port` = `-node-port` |

- 容器名是 Docker 保证唯一的稳定外部身份。创建遇到同名冲突即说明上一次执行已经创建过，改为 inspect 该
  容器并接管（未运行时启动它），而不是创建第二个；“执行成功但响应丢失”由此可以按原 ID 恢复。
- 不另起 sandbox ID：两个 ID 相等使日志、容器与数据库三处可以直接对照。

### D3：journal 持久在 Sandbox Server 自己的状态目录，Docker 是对账依据

journal 每个 effect 一个文件，位于 `-state-dir`（Compose 中为命名卷），经临时文件 + fsync + rename + 目录
fsync 原子替换。状态为 `running → succeeded | failed`，与 Cloud 的有限 Job 语义一致；失败条目只带有界错误码，
Docker 细节只进日志。

- `PUT` 在调用任何 Docker 写 API 之前先写入 `running`；进程在执行中崩溃，重启后 journal 仍说明“已开始、
  结果未知”。
- `GET` 遇到 `running` 且没有进行中的 `PUT` 时，按 D2 的容器名查询 Docker 并据实返回：ensure 的容器已按期望
  运行、终止墓碑已写且容器已不存在时提升为 `succeeded`；其余保持 `running`，由 Controller 重发 `PUT` 继续
  执行。有 `PUT` 持锁时，`GET` 直接返回 journal 当前内容，不等待可能耗时整个停止超时的容器停止。永远不因为
  “journal 是 running 且进程重启过”就判定失败或成功。
- 同一 effect ID 的执行串行（按 ID 加锁）；同一 Workspace 的 effect 也串行（按 Workspace 加锁），使 ensure、
  terminate 与数据删除的 Docker 调用不交错。不同 Workspace 可以并发，不使用全局锁，因为容器停止可能等待
  数十秒，不应阻塞其他 Workspace。
- 容器额外带 `space.ora.workspace.id` 与 `space.ora.effect.id` 标签，只供生命周期对账与 D5 的唯一性检查；
  Resolver 不读取它们。

实际决策与偏离：原决策只按 effect ID 加锁，并让 terminate 与目标 ensure 共用按 sandbox ID 的锁。实现改为
effect 锁加 Workspace 锁：同一 Workspace 的 effect 本来就不应并发改变 Docker 状态（D5），按 Workspace 串行同时
覆盖了 terminate 与目标 ensure 的互斥，以及运行时围栏的读改写（见标题下的历史决策说明）。

### D4：Node 身份按 Workspace 稳定，Node home 放在按 Workspace 命名的卷上

`sandbox_ensure` 为 Workspace `W` 创建容器时：

- NodeId 为 `workspace-<W>`，写入 Node 配置的 `node.identity`（要求 home 绑定该身份），同一 Workspace 的每一代
  沙盒相同；每次进程启动的 `NodeIncarnationId` 由 Node 自己生成。
- Node home 挂载卷 `ora-workspace-<W>`（挂载点 `-node-data-dir`），卷不存在时创建；`sandbox_terminate` 不删除卷。
- Node 配置由 Sandbox Server 按部署模板生成，经 `ORA_NODE_CONFIG` 环境变量放入容器：`control.controller_id`
  取自 `-controller-id`，`control.listen` 监听 `-node-port` 与 `-node-path`，其余取自模板。配置中不含任何凭据。

为什么这样选：

- Cloud 的 stop 保留运行数据、start 分配新 generation。Node 的执行账本与 clone 结果都在 home 中，卷跟随
  Workspace 才能让 start 之后的 Node 看到之前的 clone 与未确认事件。
- Node home 中的持久状态绑定 NodeId 与 ControllerId（`crates/node-db` 的 `NodeMismatch`、
  `ControllerMismatch`）。若每代沙盒换一个 NodeId，同一个 home 会被 Node 拒绝；因此 NodeId 跟随 Workspace、
  incarnation 跟随进程，`-controller-id` 也必须跨代不变。
- 两代沙盒不会同时使用同一个卷：Cloud 在分配新 generation 前要求旧实例已确认终止，Sandbox Server 另有 D5 的
  检查；终止确认（D6）要求容器已被删除。

### D5：每个 Workspace 同时至多一个未终止的沙盒容器

`sandbox_ensure` 创建前按 `space.ora.workspace.id` 标签查询：若存在另一个容器（无论是否运行），effect 以
`failed`、`workspace_sandbox_live` 结束，不创建新容器。

Cloud 已用 `UNIQUE(workspace_id, generation)` 与“唯一未终止 sandbox”保护替换，这里是第二道防线：两个容器挂
同一个 Workspace 卷会让两个 Node 进程同时写一个 SQLite 账本，后果不可恢复，而 Cloud 的检查只约束数据库记录，
不能约束 Docker 中残留的容器。拒绝而不是替换：替换需要先证明旧容器已终止，这是 `sandbox_terminate` 的职责。

### D6：创建即启动，终止以“容器已删除”为证据

| effect | 执行 | 成功条件 | 失败码 |
|---|---|---|---|
| `sandbox_ensure` | 检查终止墓碑 → 同名容器存在则接管 → D5 检查 → 确认卷 → 以 `-node-image` 创建容器（D2 名称与标签、只接入沙盒网络、不发布端口、挂载卷与配置）→ 启动 | 容器处于 `running` | `sandbox_terminated`、`workspace_sandbox_live`、`sandbox_scope_mismatch`（同名容器属于其他 Workspace）、`node_image_missing`、`container_not_running`、`docker_failure` |
| `sandbox_terminate` | 按 `sandboxInstanceId` 找到容器并校验其 Workspace 标签 → 写终止墓碑 → 以 `-stop-timeout` 停止 → 删除容器 | Docker 确认该名称的容器不存在，结果 `{ "terminated": true, "lateEnsureFenced": true }` | `sandbox_scope_mismatch`、`docker_failure` |
| `workspace_data_delete` | 确认没有带该 Workspace 标签的容器 → 删除卷 `ora-workspace-<W>` | 卷不存在，结果 `{ "removed": true }` | `workspace_sandbox_live`（仍有容器或卷在使用中）、`docker_failure` |

- ensure 只证明容器在运行，不证明 Node 就绪：就绪由 Controller 在 node 阶段通过握手与 Node 上报确认。
- 镜像只使用本地已有的 `-node-image`，不在 effect 中拉取：拉取耗时不可控。镜像缺失是明确失败，便于区分
  部署问题与沙盒问题。
- `-stop-timeout` 必须大于 Node 的 `shutdown_grace_ms + cleanup_timeout_ms`，使 Node 有机会正常停止并记录被
  中断的 clone；超时后 Docker 强杀。单次 `PUT` 的执行另受 `-stop-timeout` 加一分钟的上限约束，且与调用方
  连接解耦，调用方断开不会让 Docker 操作半途放弃。
- terminate 可能针对 ensure 从未执行或仍在执行的沙盒。墓碑在第一次破坏性调用之前写入，即使容器不存在也
  写；之后任何针对该 sandbox ID 的 ensure 都以 `sandbox_terminated` 失败，不再创建容器。墓碑永不删除，
  `lateEnsureFenced` 表示这一保证已经成立。
- 无法确认（Docker 报错、超时）时 effect 为 `failed`，Cloud 保持 blocked，不分配新 generation。

## 相关决策

| 决策 | 归属 | 内容 |
|---|---|---|
| [运行时 Workspace 生命周期](../../cloud/operation/0-workspace-runtime-follows-desktop-node.md) | Cloud | 操作步骤为 sandbox → node → clone，新增 `workspace_data_delete`，Node 由 Controller 代报 |
| [Workspace 操作与 Node 上报的 gRPC 契约](../../cloud/controller-integration/20260926-workspace-operations-and-controller-reported-nodes.md) | Cloud | `WorkspaceOperationService`、`NodeReportService`，clone 登记扩展到 Workspace |
| [Controller 驱动 Workspace 沙盒](../../controller/node-management/0-controller-drives-workspace-sandboxes.md) | desktop Controller | 调用本文接口，动态维护 `NodeTarget` 与会话，代报 Node |

## 不变量

1. Sandbox Server 的 effects 接口语义与 Cloud 契约一致：先 journal 后执行，同 ID 同请求幂等，不同请求
   `409`，成功结果不改写，`GET` 不创建。
2. 一个 `sandbox_ensure` effect ID 至多对应一个容器；重试与崩溃恢复不会创建第二个。
3. 同一 Workspace 同时至多存在一个沙盒容器。
4. `sandbox_terminate` 只在 Docker 确认容器不存在后报告 `terminated: true`；终止墓碑写入后，同一 sandbox ID
   不会再被创建。
5. 本文创建的容器满足根决策的全部路由约束：带三个路由标签、只接入沙盒网络、不发布端口。
6. Workspace 卷不因沙盒终止而删除，只由 `workspace_data_delete` 在没有该 Workspace 容器时删除；同一
   Workspace 的各代沙盒使用同一 NodeId。
7. Sandbox Server 不执行 Git 操作；Node 配置中不含凭据。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 为 Sandbox Server 设计新的 REST 生命周期 API | Cloud 已有带幂等、对账与 fencing 的 effect 语义；另起一套需要 Controller 翻译，并与生产 Substrate 适配器分叉（D1） |
| 实现全部六种 Substrate effect（含 storage/worktree） | 与 desktop“Node 自己 clone”的方向冲突，会在本地复制一条生产不会走的 bare repo + 维护 Job 路径（D1） |
| 另生成 sandbox ID，不用 effect ID | 多一层映射，且 Cloud 已要求结果中的 `sandboxInstanceId` 等于 effect ID（D2） |
| 卷按 sandbox ID 命名，每代沙盒一个新卷 | stop/start 后 clone 结果与未确认事件丢失，违背 Cloud“stop 保留运行数据”（D4） |
| 每代沙盒一个新 NodeId | 同一 home 被新身份打开，被 Node 以 `NodeMismatch` 拒绝，只能迁移账本（D4） |
| ensure 遇到同 Workspace 残留容器时先删再建 | 等于在 ensure 中隐式终止，绕过 terminate 的确认与 Cloud 的 generation 规则（D5） |
| ensure 等 Node 握手成功才返回 | 把 Node 就绪混入 Substrate 层；Cloud 把就绪留给 node 阶段与 Node 上报（D6） |
| journal 只放内存、以 Docker 为唯一事实 | 进程在“已决定创建、尚未调用 Docker”时崩溃，`GET` 无法区分“从未请求”与“请求过但未执行”，违背 `404` 的“确知没有”（D3） |
| 全局锁串行所有 effect | 一个 Workspace 的容器停止会阻塞所有其他 Workspace（D3） |

## 本决策未解决的问题

- suspend/resume：Substrate 文档有，Cloud 当前操作没有对应步骤，本地也无需求。
- 资源限制（CPU、内存）：Substrate 由 ActorTemplate 决定，本地暂不限制。
- 真实 Docker 中断 clone 的终止路径没有自动化证据（见核心用例的证据表）。

## 落地

effects 接口、journal、按 effect 与按 Workspace 的锁、崩溃对账以伪造 Engine 覆盖；Docker 客户端增加卷与容器
的写端点；Compose 以 `-lifecycle-addr`、`-state-dir`（命名卷）、`-node-image`、`-node-config` 启动 Sandbox
Server。保持不变：根决策的路由、状态码与静态表模式。开始改变：Docker 模式的 Sandbox Server 多一个生命周期
监听地址，沙盒由 Controller 按需创建，不再预先启动 Node 容器。
