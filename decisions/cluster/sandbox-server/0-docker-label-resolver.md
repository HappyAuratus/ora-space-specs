---
status: approved
---

# Sandbox Server 按 Docker 标签发现沙盒 Node，取代静态路由表

cluster 的 Sandbox Server 目前从静态 JSON 表（`-routes`，示例 `routes.example.json`）把
`ate-target-actor: <atespace>/<sandbox-id>` 解析为 Node 的 WebSocket 地址。本决策新增 Docker
实现的 `route.Resolver`：每次 Controller 建立连接时，按容器标签在 Docker Engine 中查找唯一匹配的
运行中容器，取它在指定 Docker 网络上的 IP 和标签声明的端口作为 Node 地址。Resolver 只读 Docker，
不创建、不停止、不删除容器；容器由 Compose（后续 T8）或人工按标签约定拉起。Controller 看到的
寻址方式、HTTP 状态码与 close code 语义不变，只是“未知沙盒”的来源从 JSON 表换成了 Docker。

本 ADR 处于 `approved`：2026-09-26 Eric 确认 D1–D6 全部按建议采纳，并确定了 T8 的部署形态（见
“对 T8 的约束”）；实现尚未开始，验证义务登记在
[Docker 标签解析核心用例](../../../test-cases/cluster/sandbox-server/docker-label-resolution.md)，
证据均为 `Missing`。涉及 cluster 仓库的
`sandbox-server/internal/route`、`sandbox-server/internal/server` 与 `cmd/sandbox-server`；desktop
与 cloud 的代码、协议和配置格式不变。兼容性边界只有一条：Controller 经 Sandbox Server 连接 Node
时可观察的寻址与错误分类（见[传输决策](../../node/protocol/20260924-frame-channel-with-ipc-and-websocket.md)
D3、D7）保持不变。本 ADR 不设计沙盒生命周期 API（创建、就绪、销毁）与 `SandboxProvider`，也不
承担静态路由表使用者的迁移：静态表保留为无 Docker 环境的实现（D6）。

这是 `decisions/cluster/sandbox-server/` 的根决策；此前 cluster 没有 ADR，Sandbox Server 的连接
语义只记录在 cluster `README.md`。

## 与现有决策的关系

| 已有约定 | 本文处理 |
|---|---|
| 传输 D3：`WsEndpoint` 的请求头原样发送，传输层不认识厂商寻址 | 继承，不变；Controller 配置不因 Docker 改变 |
| 传输 D6：Node 的 WS 入口只能经平台路由（本地为 Sandbox Server）到达 | 继承并落实：Resolver 只使用 Docker 内部网络地址，不使用宿主机端口映射（D2） |
| 传输 D7：路由 `404` → `NotFound`，`502`/`503`/`504` → `Unreachable` | 继承，不变；Docker 状态映射到已有状态码（D3） |
| 传输 D8：Provider 等 Docker 与平台两个真实实现后再提炼 | 继承，不变；本文只做路由发现，是 Docker 实现的第一块，不是 Provider（D4） |
| cluster README：先连 Node 再接受 Upgrade、状态码表、透传语义 | 继承，不变；只替换 `Resolver` 的实现 |

## 决策

### D1：以容器标签声明沙盒身份，不用容器名推导

一个容器是某个沙盒的 Node，当且仅当它带有下列标签：

| 标签 | 含义 | 约束 |
|---|---|---|
| `space.ora.sandbox.atespace` | 所属 atespace | 与 `ParseTarget` 相同：非空、不含 `/` |
| `space.ora.sandbox.id` | sandbox ID | 同上 |
| `space.ora.node.port` | Node WS 监听端口（容器内） | 1–65535 的十进制整数 |

`ate-target-actor: local/demo` 解析时，Resolver 以
`label=space.ora.sandbox.atespace=local` 与 `label=space.ora.sandbox.id=demo` 两个过滤条件列出容器
（包括未运行的），再按 D3 判定。Node 的 WS 路径不进标签：Controller 请求的路径原样转给 Node，
与静态表一致。

为什么用标签：

- 容器名是全局唯一的扁平字符串，Compose 会给它加项目前缀和序号；从名字反推
  `<atespace>/<sandbox-id>` 需要有损规则，改项目名就改变了寻址。标签是显式声明，和 Compose 项目名、
  副本序号无关。
- sandbox ID 与 `NodeId` 是两个身份：前者由路由使用，后者在握手中校验（传输 D3、不变量 5）。标签
  只声明路由身份，Resolver 不读取 Node 配置，也不因为 sandbox ID 与 `NodeId` 不一致而拒绝；两者取
  相同字符串只是 T8 的可读性约定。
- 端口放标签而不固定为约定值：同一镜像可以在不同部署中监听不同端口，Resolver 不需要硬编码 Node 的
  配置默认值。代价是容器必须写三个标签，漏写时 Resolver 以 D3 的“不可用”报错，而不是猜测端口。

### D2：只使用指定 Docker 网络内的地址

Sandbox Server 启动时通过 `-docker-network <name>` 指定沙盒网络。解析结果为
`ws://<容器在该网络上的 IPv4>:<space.ora.node.port>`，不使用容器发布到宿主机的端口，也不使用
其他网络上的地址。

- 这是传输 D6 的部署前提在本地的落实：Node 的 WS 入口不做鉴权，只能被 Sandbox Server 访问。
  Node 容器不发布端口；只有 Sandbox Server（与同网络容器）能到达。
- 因此 Sandbox Server 必须能到达该网络：在 Compose 中作为同网络的容器运行（T8 采用的形态）；
  在 Linux 上的 Docker Engine（含 WSL2 中安装的 Engine）可以直接从宿主机访问 bridge 网络 IP，
  `task sandbox:run` 仍可在宿主机运行，用于调试；Docker Desktop（macOS/Windows）宿主机访问不到
  容器 IP，只能在容器中运行 Sandbox Server。
- 选用 IP 而不是容器 DNS 名：Sandbox Server 在宿主机运行时无法使用 Docker 内置 DNS；IP 来自同一次
  Docker 查询，不会与标签结果不一致。
- 不支持 `wss://`：本地网络内的明文 WS 与静态表中的本地示例一致；TLS 属于平台路由。

### D3：每次连接实时查询，状态到 HTTP 状态码的映射固定

Resolver 不缓存，也不订阅 Docker 事件；每个 Controller 的 Upgrade 请求做一次查询（列表过滤，
必要时再 inspect 一次）。查询受请求 context 与 `-dial-timeout` 共同约束。

| Docker 状态 | Resolver 结果 | HTTP | Controller 分类 |
|---|---|---|---|
| 没有匹配标签的容器 | `ErrUnknownSandbox` | `404` | `NotFound` |
| 恰好一个匹配且 `running`，已接入沙盒网络，端口标签合法 | Node 地址 | 继续拨号 | — |
| 恰好一个匹配，但未运行（`created`/`restarting`/`paused`/`exited`/`dead`） | `ErrSandboxUnavailable` | `503` | `Unreachable` |
| 恰好一个匹配且运行中，但未接入沙盒网络或端口标签非法 | `ErrSandboxUnavailable` | `503` | `Unreachable` |
| 多于一个匹配 | `ErrSandboxUnavailable` | `503` | `Unreachable` |
| Docker API 不可达、超时或返回错误 | 解析错误 | `502` | `Unreachable` |

为什么这样映射：

- 容器存在但未运行，与平台上“沙盒存在但 Node 未就绪”对应。它必须和“未知沙盒”区分：传输 D7
  把 `NotFound` 留给后续 Provider 判断“沙盒是否已被销毁”，把一个暂停或正在重启的容器报成 `404`
  会让未来的 Provider 误以为需要重建。`503` 与 `502`/`504` 同归 `Unreachable`，Controller 按
  `reconnect_ms` 重试，行为与现在 Node 不可达时一致。
- 多个匹配是部署错误。任选其一会把同一 sandbox ID 的连接随机送到不同 Node，由握手的 `NodeId`
  校验兜底也只能事后发现；拒绝并在日志中列出冲突的容器 ID，才能让错误可解释。
- 不缓存：本地联调的连接频率是每个 Controller 每次重连一次，一次 Docker 列表查询是毫秒级；缓存或
  事件订阅会引入“容器已重建但缓存仍指向旧 IP”的窗口，并需要额外的 goroutine 所有权与失效策略。
  容器重启后 IP 可能变化，实时查询天然拿到新地址。
- HTTP 响应体只给简短、稳定的原因（如 `sandbox not running`），Docker 错误原文、容器 ID 与网络名
  只进结构化日志。当前 `server.go` 把 `Resolve` 的错误原文写进响应体，Docker 实现下要改为固定
  文本，避免把基础设施细节暴露给 Controller。

`Resolver` 接口相应改为 `Resolve(ctx context.Context, t Target) (*url.URL, error)`，并新增
`ErrSandboxUnavailable`；server 按错误类别选择 `404`/`503`/`502`。静态实现忽略 ctx，行为不变。

### D4：Resolver 只读 Docker，生命周期留给后续决策

Docker Resolver 只调用列表与 inspect 两类只读 API，不创建、启动、停止或删除容器。Compose（T8）
或人工 `docker run` 按 D1 标签拉起 Node 容器；Resolver 只负责“连接时找到它”。

- 把发现和生命周期分开，T7 可以不等生命周期 API 的设计单独落地和验收；Compose 已经能表达
  “有哪些沙盒”，一键启动不需要 Sandbox Server 的创建 API。
- 沙盒生命周期（创建、就绪查询、可重试销毁）与 `SandboxProvider` 仍按传输 D8 的顺序处理：等
  Controller 需要动态拉起 Node 时，另立本目录的后续 ADR，在 Sandbox Server 上增加生命周期 API，
  创建的容器同样带 D1 标签，本 Resolver 无需改动即可路由它们。
- 就绪不由 Resolver 判断：容器 `running` 不代表 Node 已监听。Node 尚未监听时拨号失败，按现有规则
  返回 `502`，Controller 重试。Compose 可用 healthcheck 控制依赖顺序，但那是 T8 的启动编排，
  不进入路由语义。

### D5：用最小 Engine API 客户端访问 docker.sock，不引入 Docker SDK

Sandbox Server 通过 Unix socket（默认 `/var/run/docker.sock`，`-docker-host` 可覆盖，只接受
`unix://`）以标准库 `net/http` 调用 Docker Engine API 的 `GET /containers/json` 与
`GET /containers/{id}/json`，请求路径固定 API 版本（例如 `/v1.41`，对应 Docker 20.10+），只解码
用到的字段。

- 只需要两个只读端点。官方 `github.com/docker/docker/client` 会把 moby 的大量依赖带进一个目前只依赖
  `coder/websocket` 的小模块，增加 `go.sum`、二进制体积与安全公告面；手写客户端约百行，可用
  `httptest` 在无 Docker 环境下直接测试。
- 固定 API 版本让 Engine 升级不改变响应形状；版本过旧时 Engine 返回错误，按 D3 归为 `502` 并在
  启动自检时报错。
- 启动时做一次 `GET /_ping`：Docker 不可达就拒绝启动，而不是让每个连接都 `502`。运行中 Docker
  失联仍按 D3 逐连接报 `502`，不退出进程。
- 挂载 docker.sock 等同于宿主机 root 权限。这是本地模拟环境可接受的代价：Sandbox Server 只在开发
  机运行，只做只读调用；若以后需要收紧，可以换成只放行 `GET /containers` 的 socket 代理，不影响
  本决策的语义。

### D6：静态表保留为另一种 Resolver，按启动参数二选一

`-routes <file>` 与 `-docker-network <name>` 互斥且必须恰好提供一个：前者使用现有静态表，后者使用
Docker Resolver。README 与 Compose 以 Docker 为默认形态，`routes.example.json` 保留为无 Docker
环境（如 2026-09-25 在云端容器中的手动联调）与 server 单元测试的实现。

保留而不删除，是因为静态表是唯一不依赖 Docker 的联调路径，而传输与中继语义的回归测试不应依赖
Docker。两个实现共享 `Resolver` 接口和 server 的错误映射，不共享数据来源；不做“先查 Docker、
查不到再查静态表”的组合，避免同一 sandbox ID 有两个来源时路由结果取决于查询顺序。

## 不变量

1. 一次解析要么得到唯一一个 Node 地址，要么失败；同一 `<atespace>/<sandbox-id>` 对应多个容器时
   一律拒绝，不任选其一。
2. 没有匹配标签的容器时返回 `404`；存在匹配容器但无法路由时返回 `503`；Docker 自身失败时返回
   `502`。容器存在时永不返回 `404`。
3. Docker Resolver 只调用只读 Docker API，不改变任何容器状态。
4. 解析出的地址只来自 `-docker-network` 指定网络上的容器 IP，不使用宿主机发布端口。
5. 解析结果不缓存：每次 Upgrade 都反映查询时刻的 Docker 状态。
6. Docker 错误详情、容器 ID 与网络名不出现在返回给 Controller 的 HTTP 响应体中。
7. Controller 可观察的寻址方式、请求路径透传、状态码与 close code 语义与静态表实现一致。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 按容器名约定（如 `ora-<atespace>-<id>`）解析 | 名字受 Compose 项目名与序号影响，反推身份需要有损规则；标签显式、稳定（D1）。优势是 `docker ps` 一眼可读，T8 仍可以同时给容器起可读名字 |
| 用宿主机发布端口（`127.0.0.1:<port>`）作为地址 | Node 入口暴露到宿主机，违背传输 D6 的部署前提，且多沙盒需要分配端口；优势是 Docker Desktop 下宿主机运行的 Sandbox Server 也能连通（D2 已承认此限制） |
| 订阅 Docker events 维护缓存 | 需要后台 goroutine 的所有权、断线重订阅与失效策略，还有容器重建后的旧地址窗口；本地连接频率低，实时查询的延迟可忽略（D3） |
| 未运行的容器也返回 `404` | 与“沙盒不存在”混淆，未来 Provider 会据此误判需要重建（D3、传输 D7） |
| 引入官方 Docker Go SDK | 依赖面与体积远超两个只读端点的需要；优势是跟随 API 演进，但固定版本已消除这一需求（D5） |
| 本次同时实现创建/销毁 API | 生命周期需要与平台形态一起提炼（传输 D8），会让 T7 等待 Provider 设计；发现与生命周期可以分开验收（D4） |
| 删除静态表 | 失去无 Docker 的联调与测试路径（D6） |
| Docker 与静态表组合查询 | 同一 ID 两个来源时结果取决于顺序，违背不变量 1 的唯一性（D6） |

## 对 T8（Compose + Taskfile 一键启动）的约束

T8 不属于本决策，但建立在它之上。2026-09-26 Eric 确定 T8 的部署形态：

| 事项 | 决定 |
|---|---|
| Node 镜像 | 由 desktop 提供 Dockerfile，cluster 只引用 |
| Controller 与 Cloud | 都在 Compose 的容器中运行 |
| 默认沙盒 | 默认不起沙盒，按前端请求动态创建 |

由此，本 ADR 为 T8 固定以下接口：

- **两个网络**：沙盒网络（例如 `ora-sandbox`）只接入 Sandbox Server 与 Node 容器；应用网络接入
  Cloud、Controller 与 Sandbox Server。Controller 经应用网络以服务名访问 Sandbox Server，本身不在
  沙盒网络上，因此传输 D6 的“Node 只能经路由到达”在 Compose 中由网络拓扑保证，而不只是约定。
  Sandbox Server 挂载 docker.sock，以 `-docker-network` 指向沙盒网络；只有前端所需的端口发布到
  宿主机 `127.0.0.1`。
- **Node 容器**：不在 Compose 文件中静态声明，由沙盒创建时启动（见下）。每个容器带 D1 三个标签，
  只接入沙盒网络、不发布端口；Node 配置的 `control.listen` 为 `websocket`，`bind` 为
  `0.0.0.0:<port>`，与 `space.ora.node.port` 一致；Node home 使用按 sandbox ID 命名的卷，使“Node 重启
  后重放”场景在容器重建后仍成立。
- **Controller 的端点**：`websocket` 端点的 `url` 指向应用网络上的 Sandbox Server，请求头
  `ate-target-actor: <atespace>/<sandbox-id>` 与容器标签一致。因为沙盒是动态创建的，端点由创建沙盒
  的一方交给 Controller，而不是写在启动时的静态配置里。

“按前端请求动态创建”超出了本 ADR：它需要 Sandbox Server 的创建/销毁 API，以及 Controller 侧接收
新 `NodeTarget` 的入口（传输 D8 的 `SandboxProvider`）。本 ADR 的 D4 不变：Docker Resolver 仍只读，
生命周期 API 创建的容器只要带 D1 标签，就能被本 Resolver 路由。因此生命周期决策从“未解决的问题”
变为 T8 的前置，须在本目录追加 ADR（Sandbox Server 一侧）并与 Controller 一侧的 Provider 决策
一起评审；T8 中与动态沙盒无关的部分（镜像、网络、Cloud/Controller/Sandbox Server 的 Compose 服务、
Taskfile `up`/`down`）可以先落地。

## 本决策未解决的问题

- 沙盒生命周期 API 与 `SandboxProvider`：T8 已确定按前端请求动态创建沙盒，因此这是 T8 的前置，
  在本目录追加 ADR（传输 D8），不在本文决定。
- 就绪探测：是否由 Sandbox Server 提供沙盒级就绪查询，属于生命周期 API。
- 多个 Sandbox Server 共享一个 Docker Engine 时的隔离（例如额外的 `space.ora.sandbox.server` 标签）：
  目前每台开发机只有一个 Sandbox Server，没有需求。

## 落地顺序

1. **接口调整**：`Resolver.Resolve` 增加 ctx，新增 `ErrSandboxUnavailable` 与 `503` 映射，响应体
   改为固定文本。静态实现行为不变，现有 server 测试继续通过，并补 `503` 与响应体不含内部细节的
   测试。完成条件：`task sandbox:check`、`task sandbox:test` 通过。这一步不引入 Docker，评审只看
   错误语义。
2. **Docker Resolver**：最小 Engine API 客户端与标签解析，用 `httptest` 伪造 Engine 覆盖 D3 表的
   每一行；`cmd` 增加 `-docker-network`/`-docker-host`、互斥校验与启动 `_ping`。完成条件：单元测试
   覆盖 D3 全表与不变量 1–6，不依赖真实 Docker。
3. **真实 Docker 验收**：在装有 Docker 的环境中以 `docker run` 按标签启动一个 Node 容器，重跑
   2026-09-25 联调清单中与路由相关的场景（正常路径、未知沙盒、Node 不可达、容器停止、重复标签）。
   完成条件：结果与静态表一致，另有容器停止 → `503` → `Unreachable`；核心用例中需要真实 Docker
   的义务在此补证据。
4. **T8** 与动态沙盒无关的部分在第 3 步之后开始；动态创建等生命周期 ADR 批准后再做。

保持不变的行为：Controller 配置格式、请求头寻址、路径透传、中继的 close code 与 ping/pong 语义，
以及 `-routes` 的静态表用法。从第 2 步开始改变的行为：Sandbox Server 可以不依赖路由文件、直接
路由到带标签的 Docker 容器；从第 1 步开始，解析失败的响应体不再包含错误原文。
