---
status: implemented
---

# Sandbox Server 按 Docker 标签发现沙盒 Node，取代静态路由表

> 历史决策：D2 的“Node 只能经 Sandbox Server 到达”与“对 T8 的约束”中“Controller 不在沙盒网络上”已被
> [Controller 接入沙盒网络并以双向 TLS 直连 Node](20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md)
> 修改：Cloud 模式下 Controller 接入沙盒网络，直接认证 Node，不再经过本文的路由。本文的标签解析、状态码映射与
> 静态表仍是 Sandbox Server WS 路由的现行规则，该路由保留给不启用 TLS 的本机传输工具。

Sandbox Server 的 WS 路由把 `ate-target-actor: <atespace>/<sandbox-id>` 解析为 Node 的 WebSocket 地址。
本决策在静态 JSON 表（`-routes`）之外提供 Docker 实现的 `route.Resolver`：每次 Controller 建立连接时，按
容器标签在 Docker Engine 中查找唯一匹配的容器，取它在指定 Docker 网络上的 IP 和标签声明的端口作为 Node
地址。Resolver 只读 Docker，不创建、不停止、不删除容器。Controller 看到的寻址方式、HTTP 状态码与 close code
语义不随数据来源改变。

本 ADR 处于 `implemented`：实现位于 cluster 的 `sandbox-server/internal/route`（`docker.go`、`route.go`）、
`sandbox-server/internal/docker`、`sandbox-server/internal/server` 与 `cmd/sandbox-server`，验证见
[Docker 标签解析核心用例](../../../test-cases/cluster/sandbox-server/docker-label-resolution.md)。兼容性边界：
Controller 经 Sandbox Server 连接 Node 时可观察的寻址与错误分类（见
[传输决策](../../node/protocol/20260924-frame-channel-with-ipc-and-websocket.md) D3、D7）保持不变。沙盒的
创建与销毁由[Docker 沙盒生命周期决策](20260926-docker-sandbox-lifecycle-via-substrate-effects.md)负责。

这是 `decisions/cluster/sandbox-server/` 的根决策。

## 与现有决策的关系

| 已有约定 | 本文处理 |
|---|---|
| 传输 D3：`WsEndpoint` 的请求头原样发送，传输层不认识厂商寻址 | 继承，不变；Controller 配置不因 Docker 改变 |
| 传输 D6：Node 的 WS 入口只能经平台路由（本地为 Sandbox Server）到达 | 继承并落实：Resolver 只使用 Docker 内部网络地址，不使用宿主机端口映射（D2） |
| 传输 D7：路由 `404` → `NotFound`，`502`/`503`/`504` → `Unreachable` | 继承，不变；Docker 状态映射到已有状态码（D3） |
| 传输 D8：Provider 等 Docker 与平台两个真实实现后再提炼 | 继承，不变；本文只做路由发现，不是 Provider（D4） |

## 决策

### D1：以容器标签声明沙盒身份，不用容器名推导

一个容器是某个沙盒的 Node，当且仅当它带有下列标签：

| 标签 | 含义 | 约束 |
|---|---|---|
| `space.ora.sandbox.atespace` | 所属 atespace | 与 `ParseTarget` 相同：非空、不含 `/` |
| `space.ora.sandbox.id` | sandbox ID | 同上 |
| `space.ora.node.port` | Node WS 监听端口（容器内） | 1–65535 的十进制整数 |

`ate-target-actor: local/demo` 解析时，Resolver 以 `space.ora.sandbox.atespace=local` 与
`space.ora.sandbox.id=demo` 两个标签过滤条件列出容器（包括未运行的），再按 D3 判定。Node 的 WS 路径不进
标签：Controller 请求的路径与查询串原样转给 Node。

为什么用标签：

- 容器名受 Compose 项目前缀和序号影响，从名字反推 `<atespace>/<sandbox-id>` 需要有损规则；标签是显式声明，
  和项目名、副本序号无关。
- sandbox ID 与 `NodeId` 是两个身份：前者由路由使用，后者在握手中校验（传输 D3）。标签只声明路由身份，
  Resolver 不读取 Node 配置。
- 端口放标签而不固定为约定值：同一镜像可以在不同部署中监听不同端口。漏写或写错时按 D3 报“不可用”，
  不猜测端口。

### D2：只使用指定 Docker 网络内的地址

Sandbox Server 启动时通过 `-docker-network <name>` 指定沙盒网络。解析结果为
`ws://<容器在该网络上的 IPv4>:<space.ora.node.port>`，不使用容器发布到宿主机的端口，也不使用其他网络上的
地址。

- 这是传输 D6 部署前提在本地的落实：沙盒容器不发布端口，经路由连接的 Node 只能在沙盒网络内到达。
- Sandbox Server 必须能到达该网络：Compose 中作为同网络容器运行；Linux 上的 Docker Engine（含 WSL2 中
  安装的 Engine）可以从宿主机直接访问 bridge 网络 IP，`task sandbox:run` 可在宿主机运行用于调试；Docker
  Desktop（macOS/Windows）宿主机访问不到容器 IP，只能在容器中运行 Sandbox Server。
- 选用 IP 而不是容器 DNS 名：Sandbox Server 在宿主机运行时无法使用 Docker 内置 DNS；IP 与标签来自同一次
  查询，不会彼此不一致。
- 路由只拨 `ws://`：TLS 属于平台路由；启用双向 TLS 的 Node 由 Controller 直连（见标题下的历史决策说明）。

### D3：每次连接实时查询，状态到 HTTP 状态码的映射固定

Resolver 不缓存，也不订阅 Docker 事件；每个 Controller 的 Upgrade 请求做一次列表查询，受请求 context 与
`-dial-timeout` 共同约束。解析失败时 server 在拨号 Node 之前返回。

| Docker 状态 | Resolver 结果 | HTTP | Controller 分类 |
|---|---|---|---|
| 没有匹配标签的容器 | `ErrUnknownSandbox` | `404` | `NotFound` |
| 恰好一个匹配且 `running`，已接入沙盒网络，端口标签合法 | Node 地址 | 继续拨号 | — |
| 恰好一个匹配，但未运行（`created`/`restarting`/`paused`/`exited`/`dead`） | `ErrSandboxUnavailable` | `503` | `Unreachable` |
| 恰好一个匹配且运行中，但未接入沙盒网络或端口标签非法 | `ErrSandboxUnavailable` | `503` | `Unreachable` |
| 多于一个匹配 | `ErrSandboxUnavailable` | `503` | `Unreachable` |
| Docker API 不可达、超时或返回错误 | 解析错误 | `502` | `Unreachable` |

为什么这样映射：

- 容器存在但未运行，对应平台上“沙盒存在但 Node 未就绪”，必须和“未知沙盒”区分：传输 D7 把 `NotFound`
  留给 Provider 判断“沙盒是否已被销毁”，把暂停或重启中的容器报成 `404` 会让 Provider 误以为需要重建。
- 多个匹配是部署错误。任选其一会把同一 sandbox ID 的连接随机送到不同 Node；拒绝并在日志中列出冲突的
  容器 ID，错误才可解释。
- 不缓存：连接频率是每个 Controller 每次重连一次，一次列表查询是毫秒级；缓存会引入“容器已重建但仍指向
  旧 IP”的窗口。
- HTTP 响应体只给固定短语（`unknown sandbox`、`sandbox unavailable`、`sandbox discovery failed`），Docker
  错误原文、容器 ID 与网络名只进结构化日志。

`Resolver` 接口为 `Resolve(ctx context.Context, t Target) (*url.URL, error)`，错误分别包装
`ErrUnknownSandbox`、`ErrSandboxUnavailable` 或原样返回 Docker 错误；静态实现忽略 ctx。

### D4：Resolver 只读 Docker，生命周期由独立组件负责

Docker Resolver 只依赖 `ContainerLister`（列表查询），不创建、启动、停止或删除容器。容器由生命周期组件、
Compose 或人工 `docker run` 按 D1 标签拉起；Resolver 只负责“连接时找到它”。

- 发现与生命周期分开验收：生命周期组件创建的容器同样带 D1 标签，Resolver 无需改动即可路由它们。
- 就绪不由 Resolver 判断：容器 `running` 不代表 Node 已监听。Node 尚未监听时拨号失败返回 `502`，
  Controller 按 `reconnect_ms` 重试。

### D5：用最小 Engine API 客户端访问 docker.sock，不引入 Docker SDK

Sandbox Server 通过 Unix socket（默认 `/var/run/docker.sock`，`-docker-host` 可覆盖，只接受 `unix://`）以
标准库 `net/http` 调用 Docker Engine API，请求路径固定为 `v1.41`（Docker 20.10+），只解码用到的字段。

- 官方 Go SDK 会把 moby 的大量依赖带进一个只需少数端点的小模块；手写客户端可用 `httptest` 与本地 unix
  socket 在无 Docker 环境下测试。
- 固定 API 版本让 Engine 升级不改变响应形状。
- Docker 模式启动时做一次 `GET /_ping`：Docker 不可达就拒绝启动，而不是让每个连接都 `502`。运行中 Docker
  失联仍按 D3 逐连接报 `502`，不退出进程。
- 只接受 `unix://`：Engine API 自身没有认证，TCP 地址会把宿主机 root 权限暴露到网络上。挂载 docker.sock
  本身等同于宿主机 root，这是本地模拟环境接受的代价。

### D6：静态表保留为另一种 Resolver，按启动参数二选一

`-routes <file>` 与 `-docker-network <name>` 互斥且必须恰好提供一个：前者使用静态表，后者使用 Docker
Resolver。静态表在启动时校验每个端点（`ws://` 或 `wss://`、有主机、无查询串），错误的表不能启动。

保留静态表，是因为它是唯一不依赖 Docker 的联调路径，传输与中继语义的回归测试不应依赖 Docker。两个实现
共享 `Resolver` 接口和 server 的错误映射，不做“先查 Docker、查不到再查静态表”的组合，避免同一 sandbox ID
的路由结果取决于查询顺序。

## 不变量

1. 一次解析要么得到唯一一个 Node 地址，要么失败；同一 `<atespace>/<sandbox-id>` 对应多个容器时一律拒绝。
2. 没有匹配标签的容器时返回 `404`；存在匹配容器但无法路由时返回 `503`；Docker 自身失败时返回 `502`。
   容器存在时永不返回 `404`。
3. Docker Resolver 只调用只读 Docker API，不改变任何容器状态。
4. 解析出的地址只来自 `-docker-network` 指定网络上的容器 IP，不使用宿主机发布端口。
5. 解析结果不缓存：每次 Upgrade 都反映查询时刻的 Docker 状态。
6. Docker 错误详情、容器 ID 与网络名不出现在返回给 Controller 的 HTTP 响应体中。
7. Controller 可观察的寻址方式、请求路径透传、状态码与 close code 语义与静态表实现一致。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 按容器名约定（如 `ora-<atespace>-<id>`）解析 | 名字受 Compose 项目名与序号影响，反推身份需要有损规则；标签显式、稳定（D1）。优势是 `docker ps` 一眼可读，容器仍可同时有可读名字 |
| 用宿主机发布端口（`127.0.0.1:<port>`）作为地址 | Node 入口暴露到宿主机，违背传输 D6 的部署前提，且多沙盒需要分配端口；优势是 Docker Desktop 下宿主机运行的 Sandbox Server 也能连通（D2 已承认此限制） |
| 订阅 Docker events 维护缓存 | 需要后台 goroutine 的所有权、断线重订阅与失效策略，还有容器重建后的旧地址窗口；连接频率低，实时查询的延迟可忽略（D3） |
| 未运行的容器也返回 `404` | 与“沙盒不存在”混淆，Provider 会据此误判需要重建（D3、传输 D7） |
| 引入官方 Docker Go SDK | 依赖面与体积远超需要；优势是跟随 API 演进，但固定版本已消除这一需求（D5） |
| 删除静态表 | 失去无 Docker 的联调与测试路径（D6） |
| Docker 与静态表组合查询 | 同一 ID 两个来源时结果取决于顺序，违背不变量 1（D6） |

## 本决策未解决的问题

- 多个 Sandbox Server 共享一个 Docker Engine 时的隔离（例如额外的 `space.ora.sandbox.server` 标签）：每台
  开发机只有一个 Sandbox Server，没有需求。
- 真实 Docker 上的端到端路由没有自动化测试，只有手工验收（见核心用例的证据表）。

## 落地

Resolver 接口带 ctx 并区分 `404`/`503`/`502`，响应体为固定短语；Docker Resolver 以 `httptest` 伪造的 Engine
覆盖 D3 全表；`cmd/sandbox-server` 提供 `-docker-network`/`-docker-host`、互斥校验与启动 `_ping`。保持不变：
Controller 配置格式、请求头寻址、路径透传、中继的 close code 与 ping/pong 语义，以及 `-routes` 的静态表用法。
