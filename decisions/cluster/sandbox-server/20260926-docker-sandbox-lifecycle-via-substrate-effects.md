---
status: approved
---

# Sandbox Server 用 Docker 实现 Cloud 的 Substrate effects 接口，按 Workspace 创建与终止沙盒

> 待评审关联：[普通停止与管理员强停](../../cloud/operation/20260927-safe-stop-and-administrative-force-stop.md)及[运行时控制同步交付](../../cloud/controller-integration/20260927-fenced-runtime-control-delivery.md)提案要求真实终止和旧实例禁止重建的证据；均为 `proposed`，不据本文规格宣称 cluster 实现已验收。

本地 cluster 不再预先启动 Node：Cloud 的 Workspace 操作走到 sandbox 阶段时，Controller 以 Cloud
已有的 Substrate 最小接口（`GET/PUT /effects/{effectId}`）请求 Sandbox Server；Sandbox Server 用 Docker
为该 Workspace 创建一个运行 `ora-node` 的容器，并带上[根决策](0-docker-label-resolver.md) D1 的路由
标签，使已有的 Docker Resolver 无需改动即可把 Controller 的 WebSocket 送到它。终止时停止并删除容器，
只在 Docker 确认容器已不存在后报告终止。Node 的 home 放在按 Workspace 命名的卷上，沙盒终止后保留，
下一代沙盒以同一 Node 身份继续使用。仓库由 Node 经 Controller–Node 协议 clone，Sandbox Server 不做
任何 Git 操作。

本 ADR 处于 `approved`，2026-09-26 Eric 确认评审要点全部按建议采纳（Node 由 Controller 代报、NodeId 与数据卷按 Workspace 固定、迁移保留旧表并失败在途旧步骤、租户级 clone 保留、不定义 SandboxProvider trait）；实现尚未开始，验证义务登记在[Docker 沙盒生命周期核心用例](../../../test-cases/cluster/sandbox-server/docker-sandbox-lifecycle.md)，证据均为 `Missing`。涉及 cluster 仓库的 Sandbox Server；它与
同一变更中的三份决策一起评审（见“相关决策”）。兼容性边界：Cloud 的 Substrate 接口语义
（`docs/execution-contract.md`“Substrate 最小接口”）以 Cloud 为准，本文只实现、不修改；根决策的标签、
地址与状态码规则不变。本文不设计 suspend/resume、资源配额与镜像拉取。

## 继承与修改

| 根决策（Docker 标签解析）的约定 | 本文处理 |
|---|---|
| D1：以三个标签声明沙盒身份 | 继承，不变；本文创建的容器必须带这三个标签，另加只供生命周期使用的标签（D3） |
| D2：只使用 `-docker-network` 上的容器 IP，Node 不发布端口 | 继承，不变；创建时只接入该网络 |
| D3：解析实时查询、状态码映射固定 | 继承，不变；创建中、已停止的容器按原表返回 `503` |
| D4：Resolver 只读 Docker，生命周期留给后续决策 | 继承并补全：Resolver 仍只读；生命周期由本文的独立组件负责，两者只经标签交汇（D1） |
| D5：最小 Engine API 客户端，不引入 Docker SDK | 继承并扩展：同一客户端增加创建、启动、停止、删除容器与卷的写端点（D6） |
| D6：`-routes` 与 `-docker-network` 二选一 | 继承；生命周期接口只在 Docker 模式下启用，静态表模式不提供 |
| 根决策“对 T8 的约束”：Node 容器由沙盒创建时启动、Node home 用命名卷 | 修改：卷按 Workspace 命名而不是按 sandbox ID（D4），使 stop/start 之后 Node 数据仍在 |

## 决策

### D1：实现 Cloud 已有的 Substrate effects 接口，不另定生命周期 API

Sandbox Server 在独立于中继的监听地址（`-lifecycle-addr`）上提供：

| 方法 | 语义（与 Cloud 契约一致） |
|---|---|
| `GET /effects/{effectId}` | `404` 表示确知没有该 intent；否则返回原请求、`externalId`、`state`、`result`；查询不创建任何东西 |
| `PUT /effects/{effectId}` | 首次先落 journal 再执行；同 ID 同请求幂等，同 ID 不同请求 `409`；成功后原结果保留不变 |

第一版只接受 `sandbox_ensure`、`sandbox_terminate` 与 `workspace_data_delete` 三种 `kind`，与
[运行时 Workspace 生命周期](../../cloud/operation/0-workspace-runtime-follows-desktop-node.md) D1、D3 保留的
effect 一致。其他 `kind`（`storage_*`、`worktree_*`、`plugin_*`）以 `400 unsupported_kind` 拒绝且不写
journal：仓库由 Node 经协议 clone，没有 Project 共享卷、bare repository 或维护 Job，Cloud 不再计划这些
effect。

为什么复用 Cloud 的接口：

- Cloud 已经为它建好了持久 plan、稳定幂等键、epoch 对账（旧 epoch 的 effect 先 `GET` 再登记）与
  generation fencing。另定一套 API，Controller 就要在两种生命周期语义之间翻译，也会与生产 Substrate
  适配器分叉。
- 生产适配器“可以映射到其控制 API，但必须保持语义”；Sandbox Server 正是这样一个适配器，只是后端
  换成 Docker。Controller 面对本地与平台时只差接口地址。
- 独立监听地址而不是与中继共用端口：中继把任何带 `ate-target-actor` 的请求路径原样转给 Node，生命周期
  路径与之共存会让路由依赖“有没有这个请求头”的隐式区分；分开后，生命周期端口在 Compose 中只接入应用
  网络，访问面与 Cloud 契约“受控网络保护”的要求一致。

### D2：sandbox 的外部身份就是 effect ID，由它推出容器名、标签与寻址

Cloud 以 `sandbox_ensure` 的 effect ID 作为 `sandbox_instances.id`，并要求成功结果的
`sandboxInstanceId` 等于该 ID。Sandbox Server 因此直接以该 UUID 作为 sandbox ID：

| 用途 | 值 |
|---|---|
| 成功结果 | `{ "sandboxInstanceId": <effectId>, "nodeId": <D4 的 NodeId> }`，`externalId` = `<effectId>` |
| 容器名 | `ora-sandbox-<effectId>` |
| 路由标签 | `space.ora.sandbox.atespace` = `-atespace`（默认 `local`），`space.ora.sandbox.id` = `<effectId>`，`space.ora.node.port` = Node 监听端口 |
| Controller 寻址 | `ate-target-actor: <atespace>/<externalId>` |

- 容器名是 Docker 保证唯一的稳定外部身份。创建遇到同名冲突即说明上一次执行已经创建过，改为
  inspect 该容器并继续，而不是创建第二个；“执行成功但响应丢失”由此可以按原 ID 恢复，满足 Cloud
  “不能以调用方超时判定不存在”的要求。
- 不另起 sandbox ID：Cloud 已经把 `externalId` 登记为 `substrate_sandbox_id`，Controller 用它寻址；
  两个 ID 相等使日志、容器与数据库三处可以直接对照。

### D3：journal 持久在 Sandbox Server 自己的状态目录，Docker 是对账依据

journal 每个 effect 一个文件，位于 `-state-dir`（Compose 中为命名卷），经临时文件 + fsync + rename
原子替换。状态为 `running → succeeded | failed`，与 Cloud 的有限 Job 语义一致。

- `PUT` 在调用任何 Docker 写 API 之前先写入 `running`；进程在执行中崩溃，重启后 journal 仍说明
  “已开始、结果未知”。
- `GET` 遇到 `running` 时按 D2 的容器名查询 Docker 并据实返回：容器已按期望运行则写回
  `succeeded`；没有容器且没有进行中的 `PUT` 则保持 `running`，由 Controller 重发 `PUT` 继续执行。
  永远不因为“journal 是 running 且进程重启过”就判定失败或成功。
- 同一 effect ID 的执行在进程内串行（按 ID 加锁），不同 effect 可以并发；不使用全局锁，因为容器停止
  可能等待数十秒，不应阻塞其他 Workspace。
- 容器额外带 `space.ora.workspace.id` 与 `space.ora.effect.id` 标签，只供生命周期对账与 D5 的唯一性
  检查使用；Resolver 不读取它们。

### D4：Node 身份按 Workspace 稳定，Node home 放在按 Workspace 命名的卷上

`sandbox_ensure` 为 Workspace `W` 创建容器时：

- NodeId 为 `workspace-<W>`，同一 Workspace 的每一代沙盒相同；每次进程启动的 `NodeIncarnationId`
  由 Node 自己生成，区分各代进程。
- Node home 挂载卷 `ora-workspace-<W>`，卷不存在时创建；`sandbox_terminate` 不删除卷。
- Node 配置由 Sandbox Server 生成并以只读方式放入容器：`controller_id` 取自 `-controller-id`，
  `control.listen` 为 `websocket`（`bind` `0.0.0.0:<port>`，`path` `/ora-node/v1`），其余取自部署提供的
  配置模板。配置中不含任何凭据（传输 D6）。

为什么这样选：

- Cloud 的 stop 保留运行数据、start 分配新 generation；Substrate 文档也要求沙盒释放后 Workspace 数据仍
  可用于创建新沙盒。Node 的执行账本与 clone 结果都在 home 中，卷跟随 Workspace 才能让 start 之后的
  Node 看到之前的 clone 与未确认事件。
- Node home 中的持久状态绑定 NodeId。若每代沙盒换一个 NodeId，同一个 home 会被新身份打开并被 Node
  拒绝（`NodeMismatch`），只能迁移账本。NodeId 跟随 Workspace、incarnation 跟随进程，正好对应协议根决策中
  “稳定身份 + 运行实例身份”的划分。
- 两代沙盒不会同时使用同一个卷：Cloud 在分配新 generation 前要求旧实例已确认终止，Sandbox Server
  另有 D5 的检查；终止确认（D6）要求容器已被删除，旧 Node 进程不可能再写入。

Node 数据库在打开时校验 home 绑定的 NodeId 与 ControllerId（`crates/node-db` 的 `NodeMismatch`、
`ControllerMismatch`），所以 `-controller-id` 同样必须跨代不变。Cloud 侧以 incarnation 区分每代 Node 的
登记，见生命周期决策 D4。

### D5：每个 Workspace 同时至多一个未终止的沙盒容器

`sandbox_ensure` 创建前按 `space.ora.workspace.id` 标签查询：若存在另一个 effect ID 的容器（无论是否
运行），effect 以 `failed`、`result.error = "workspace_sandbox_live"` 结束，不创建新容器。

Cloud 已用 `UNIQUE(workspace_id, generation)` 与“唯一未终止 sandbox”保护替换，这里是第二道防线：两个
容器挂同一个 Workspace 卷会让两个 Node 进程同时写一个 SQLite 账本，后果不可恢复，而 Cloud 的检查只
约束数据库记录，不能约束 Docker 中残留的容器（例如 Sandbox Server 在删除前崩溃）。拒绝而不是替换：
替换需要先证明旧容器已终止，这是 `sandbox_terminate` 的职责，不能在 ensure 中隐式完成。

### D6：创建即启动，终止以“容器已删除”为证据

| effect | 执行 | 成功条件 | 失败 |
|---|---|---|---|
| `sandbox_ensure` | 确认卷 → 以 `-node-image` 创建容器（D2 名称与标签、只接入沙盒网络、不发布端口、挂载卷与配置）→ 启动 | 容器处于 `running` | 镜像不存在、卷或容器创建失败、D5 冲突 |
| `sandbox_terminate` | 按 `sandboxInstanceId` 找到容器并校验其 Workspace 标签与请求一致 → 以 `-stop-timeout` 停止 → 删除容器 | Docker 确认该名称的容器不存在，结果 `{ "terminated": true }` | 停止或删除失败、Workspace 不一致 |
| `workspace_data_delete` | 确认没有带该 Workspace 标签的容器 → 删除卷 `ora-workspace-<W>` | 卷不存在，结果 `{ "removed": true }` | 仍有容器（`workspace_sandbox_live`）、删除失败 |

- ensure 只证明容器在运行，不证明 Node 就绪：Cloud 明确规定 Pod Running 不能代替 Node 协议确认，
  就绪由 Controller 在 node 阶段通过握手与 Node 上报确认。
- 镜像只使用本地已有的 `-node-image`，不在 effect 中拉取：拉取耗时不可控，且镜像由 T8 从 desktop 的
  Dockerfile 构建。镜像缺失是明确失败，便于区分部署问题与沙盒问题。
- `-stop-timeout` 必须大于 Node 的 `shutdown_grace_ms + cleanup_timeout_ms`，使 Node 有机会正常停止并
  记录被中断的 clone（中断失败判定 ADR）；超时后 Docker 强杀，Node 与 git 同生共死的终止策略保证不留
  孤儿写入。
- Cloud 在计划 ensure 时就登记了 sandbox 实例，所以 terminate 可能针对 ensure 从未执行或仍在执行的
  沙盒。terminate 因此与目标 sandbox ID 的 ensure 共用 D3 的按 ID 锁，并先在 journal 中写入该 sandbox
  的终止墓碑；之后任何针对该 sandbox ID 的 ensure 执行都以 `failed`、`sandbox_terminated` 结束，不再
  创建容器。墓碑写入后容器不存在即返回 `terminated: true`，不会有迟到的 ensure 在确认之后重新创建它。
- 无法确认（Docker 报错、超时）时 effect 为 `failed`，Cloud 保持 blocked，不分配新 generation。

## 相关决策

本文只让 Sandbox Server 能按 Workspace 建删沙盒。端到端从前端建出可用的沙盒，还需要同一变更中的：

| 决策 | 归属 | 内容 |
|---|---|---|
| [运行时 Workspace 生命周期](../../cloud/operation/0-workspace-runtime-follows-desktop-node.md) | Cloud | 操作步骤改为 sandbox → node → clone，丢弃 storage/worktree，新增 `workspace_data_delete`，Node 由 Controller 代报 |
| [Workspace 操作与 Node 上报的 gRPC 契约](../../cloud/controller-integration/20260926-workspace-operations-and-controller-reported-nodes.md) | Cloud | `WorkspaceOperationService`、`NodeReportService`，clone 登记扩展到 Workspace |
| [Controller 驱动 Workspace 沙盒](../../controller/node-management/0-controller-drives-workspace-sandboxes.md) | desktop Controller | 调用本文接口，动态维护 `NodeTarget` 与会话，代报 Node |

## 不变量

1. Sandbox Server 的 effects 接口语义与 Cloud 契约一致：先 journal 后执行，同 ID 同请求幂等，不同请求
   `409`，成功结果不改写，`GET` 不创建。
2. 一个 `sandbox_ensure` effect ID 至多对应一个容器；重试与崩溃恢复不会创建第二个。
3. 同一 Workspace 同时至多存在一个沙盒容器。
4. `sandbox_terminate` 只在 Docker 确认容器不存在后报告 `terminated: true`。
5. 本文创建的容器满足根决策的全部路由约束：带三个路由标签、只接入沙盒网络、不发布端口。
6. Workspace 卷不因沙盒终止而删除，只由 `workspace_data_delete` 在没有该 Workspace 容器时删除；同一
   Workspace 的各代沙盒使用同一 NodeId。
7. Sandbox Server 不执行 Git 操作，不注入任何凭据。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 为 Sandbox Server 设计新的 REST 生命周期 API | Cloud 已有带幂等、对账与 fencing 的 effect 语义；另起一套需要 Controller 翻译，并与生产 Substrate 适配器分叉（D1） |
| 实现全部六种 Substrate effect（含 storage/worktree） | 与 desktop“Node 自己 clone”的方向冲突，会在本地复制一条生产不会走的 bare repo + 维护 Job 路径（D1、生命周期决策） |
| 另生成 sandbox ID，不用 effect ID | 多一层映射，且 Cloud 已要求结果中的 `sandboxInstanceId` 等于 effect ID（D2） |
| 卷按 sandbox ID 命名，每代沙盒一个新卷 | stop/start 后 clone 结果与未确认事件丢失，违背 Cloud“stop 保留运行数据”（D4） |
| 每代沙盒一个新 NodeId | 同一 home 被新身份打开，被 Node 以 `NodeMismatch` 拒绝，只能迁移账本（D4）；优势是与 Cloud 原“每进程一个 Node UUID”一致，改由生命周期决策 D4 以 incarnation 对齐 |
| ensure 遇到同 Workspace 残留容器时先删再建 | 等于在 ensure 中隐式终止，绕过 terminate 的确认与 Cloud 的 generation 规则（D5） |
| ensure 等 Node 握手成功才返回 | 把 Node 就绪混入 Substrate 层；Cloud 把就绪留给 node 阶段与 Node 上报（D6） |
| journal 只放内存、以 Docker 为唯一事实 | 进程在“已决定创建、尚未调用 Docker”时崩溃，`GET` 无法区分“从未请求”与“请求过但未执行”，违背 `404` 的“确知没有”（D3） |

## 本决策未解决的问题

- suspend/resume：Substrate 文档有，Cloud 当前操作没有对应步骤，本地也无需求。
- 生命周期接口的调用方认证：本地只靠网络隔离（应用网络、不发布端口）；与 Cloud 的服务身份一起在
  生产适配器决策中处理。
- 资源限制（CPU、内存）：Substrate 由 ActorTemplate 决定，本地暂不限制。

## 落地顺序

1. **effects 接口与 journal**：`-lifecycle-addr`、`-state-dir`，D1 的接口与 D3 的 journal、并发与对账，
   后端以测试替身代替 Docker。完成条件：幂等、`409`、`404`、崩溃恢复（journal `running` 后重启）有
   单元测试。这一步只验证 Cloud 契约语义，不依赖 Docker。
2. **Docker 执行**：扩展根决策 D5 的客户端（卷、容器创建/启动/停止/删除），实现 D2、D4–D6，用伪造
   Engine 覆盖成功、同名冲突恢复、D5 冲突、终止确认与各失败路径。
3. **真实 Docker 验收**：用 Cloud 的 Go 模拟 Controller 或 `curl` 按 effect 调用，验证创建的容器能被
   Docker Resolver 路由并与 Controller 完成握手；terminate 后 Resolver 返回 `404`，再次 ensure 新
   effect 后 Node 以同一 NodeId 读到上一代的 clone。
4. **端到端**：待相关决策批准并实现后，从 Cloud 前端创建 Workspace，经 Controller 走完 sandbox 与
   node 阶段并完成 clone。

保持不变的行为：根决策的路由、状态码与静态表模式。从第 1 步开始，Docker 模式的 Sandbox Server 多一个
生命周期监听地址；从第 2 步开始，沙盒可以由 Controller 按需创建，不再需要预先启动 Node 容器。
