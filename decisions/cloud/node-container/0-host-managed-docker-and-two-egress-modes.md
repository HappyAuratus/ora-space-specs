---
status: proposed
---

# Node 容器隔离与出网：执行宿主直接调 Docker，一个项目一个加固容器，出网分 restricted 和 open 两种模式，取代 sandbox-server

每个项目的 Node 跑在一个由[执行宿主](../../../domain/glossary.md#execution-host)（下称“宿主”）直接通过 Docker Engine API
创建的容器里，不再经过 sandbox-server。容器加固沿用 sandbox-server 里有效的做法：Node 管理进程用 root，Git 和 Agent 等
工作负载用 UID 1000；配置和项目令牌放在私有 tmpfs `/run/ora`，工作负载读不到；工作负载碰不到 Docker socket 和管理目录；
每个项目一个数据卷，NodeId 固定。所有 Node 容器接在宿主专门建的 bridge 网络上，容器之间互相不通；宿主在 `DOCKER-USER`
链和 `INPUT` 链里装好规则。出网分两种模式，由部署配置统一选择：`restricted` 只放行 Cloud 入口和管理员白名单；`open`
另外放行公网，但内网地址照样拒绝。项目就是信任边界：同一个项目里的会话共用一个容器。镜像按 digest 固定，Agent CLI 打进
镜像；每个容器默认带内存、CPU 和进程数上限。

本 ADR 处于 `proposed`，供 owner 评审，是新建领域 `cloud/node-container` 的根决策。前提是
[Cloud 单进程根决策](../lifecycle/0-single-process-cloud-on-enterprise-intranet.md)（下称“A1”）和
[多机执行层根决策](../execution-host/0-host-relayed-control-and-pinned-projects.md)（下称“A2”），依据是 circus #1043 的
Q2 结论。读取的版本为 cluster `0b8d913`（`docs/sandbox/sandbox-server.md`）、cloud `ad324c77`（`v2-server/internal/node`）、
desktop `dcff8afe`（`ora-node` 的配置）。

范围与兼容性边界：

- **涉及**：宿主程序的容器管理和网络规则（cloud），Node 镜像的构建和离线分发（I5 #1055、I9 #1059）。
- **不在本文**：宿主怎么连 Cloud、项目放在哪台机器、令牌怎么签发，见 A2；Git 和模型代理的授权范围见 A4、A5。
  Node 进程内部怎么管理子进程（host/guardian 或 O5 的 ora-process）不在本文，本文只规定容器给它的边界。
- **演示约束**：继承 A1 D7。演示结束前，演示走本地 Node 子进程路径，没有容器隔离，本文的规则不影响演示；I10 #1060
  切到内嵌宿主后，演示部署用 `open` 模式。
- **不做迁移**：v1 沙盒的数据卷不迁移，项目在新的容器里重建（A1 D3、A2 D4）。

## 现状与继承

| 前序约定或现状 | 本文处理 |
| --- | --- |
| A1 D4 第 1 条：Agent 执行的代码碰不到 Cloud 的数据库、其他项目和凭据；容器网络只能访问 Cloud 入口和白名单 | 继承，具体规则见 D3、D4 |
| A1 D4 第 4 条：项目是信任边界，一个项目一个 Node 容器 | 继承，并写明接受的代价（D5） |
| A2 D5：宿主只做起停容器、转发帧、上报清单，不保存状态；按 Cloud 打的标签识别容器和数据卷 | 继承。本文规定标签和容器布局（D1、D2） |
| A2 D6：项目令牌由宿主写进容器的私有配置目录 | 继承，位置是 `/run/ora`（D2） |
| sandbox-server：管理进程 root、工作负载 UID 1000、`/run/ora` 0600 tmpfs、工作负载碰不到 Docker socket 和管理卷、每个 Workspace 一个卷、NodeId 固定、只用本地镜像、可选内存上限 | 继承这些加固；单位从 Workspace 改为项目（D2、D6、D7） |
| sandbox-server：effect journal 与对账、Cloud 签发的短期许可、代次围栏和墓碑、每代 SPIFFE 证书、WebSocket 路由加双向 TLS | 全部不要（D1） |
| sandbox-server：`node.env` 把任意环境变量注入容器，常用于 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY` | 修改：不再提供任意环境注入，Key 不进容器（D4） |
| cluster 沙盒网络：非 internal 的 bridge，可以出网，不做出网白名单 | 修改：两种出网模式，默认拒绝（D3） |

## 决策

### D1：宿主直接调 Docker Engine API，不保存容器状态

- 宿主通过本机 Docker socket 调 Docker Engine API（Podman 的兼容 API 也可以），自己创建、启动、停止和删除 Node 容器和数据卷。
  sandbox-server 和 cluster 的沙盒路由退役。
- 宿主给每个容器和数据卷打上 Cloud 的标签：`space.ora.project`（项目 ID）、`space.ora.start`（A2 D6 里这一次启动的标识）。
  A2 D5 的上报和对照只认这些标签；没有标签的资源宿主不碰。
- 起停都是幂等的：同一个项目已经有运行中的容器时，再收到启动指令就报告"已在运行"，不创建第二个；停止一个已经不存在的
  容器算成功。宿主不写 journal、墓碑或许可记录，状态以 Docker 里实际有什么为准。
- **宿主需要的权限**：访问 Docker socket，以及改 iptables（root 或 `CAP_NET_ADMIN`）。这两项都等于这台机器的 root 权限，
  所以执行机器应当专用于跑 Node。单机部署时宿主内嵌在 Cloud 进程里（A2 D2），Cloud 进程因此也拥有这些权限。

为什么丢掉 sandbox-server 的 effect 机制：它是为了让"Cloud 发出的建删指令在任何一点崩溃都能续上"，A2 D3 已经定了断开即
中断、宿主不保存状态，重连后由 Cloud 按清单对齐。剩下的需求只是"同一个项目不能同时有两个容器"，在一台机器上由宿主按标签
检查就够了，跨机器由 A2 D4 的"一个项目只固定在一台机器上"保证。

### D2：容器布局

| 位置 | 内容 | 谁能读写 |
| --- | --- | --- |
| `/run/ora`（tmpfs，0700，属 root） | Node 配置、项目令牌、Cloud 入口的 CA | 只有 Node 管理进程；文件 0600 |
| `/run/ora-control`（宿主上的私有目录，以 bind mount 挂入，0700，属 root） | Node 控制协议的 Unix socket | Node 管理进程和宿主 |
| `/var/lib/ora`（数据卷 `ora-project-<项目 ID>`） | Node 和进程管理的账本（root 私有），以及工作负载目录（UID 1000） | 账本只有 root；工作负载只能写自己的目录 |
| `/tmp` | tmpfs，有大小上限 | 工作负载 |
| 其余根文件系统 | 镜像内容，只读 | 所有进程只读 |

- **NodeId 固定**为 `project-<项目 ID>`，每次启动都用同一身份打开同一个数据卷。
- **进程身份**：Node 管理进程用 root，Git 和 Agent 等工作负载用 UID/GID 1000（Node 配置的 `process.workload_uid`）。
- **能力**：去掉全部 capability，只加回 Node 管理进程切换用户和管理子进程所需的最小集合；设置 `no-new-privileges`；不用
  `--privileged`，不挂载 Docker socket、宿主的其他目录或设备。最小集合由 I5 #1055 按 Node 的实际需要确定，写进核心用例。
- **不发布端口**：容器只通过 `/run/ora-control` 的 Unix socket 和宿主通信，Node 访问 Cloud 走 D3 的出网规则。

为什么根文件系统只读：Agent 和插件的写入都应当落在数据卷或 `/tmp`，只读根文件系统让"改坏镜像里的工具"不可能，也保证同一个
digest 的镜像在每台机器上行为一致。代价是有的 Agent CLI 可能默认往别处写缓存，需要在镜像里把它们指到数据卷；I5 实测后，
确实需要的可写位置以数据卷子目录的方式加入。

### D3：网络：每台机器一个 bridge 网络，宿主装规则，默认拒绝

- 宿主启动时创建（或接管）bridge 网络 `ora-nodes`，关闭容器间通信（`com.docker.network.bridge.enable_icc=false`），
  关闭 IPv6。所有 Node 容器只接这一个网络。
- 宿主在 iptables 里装一条自己的链 `ORA-NODES`，从 `DOCKER-USER`（转发到别处的流量）和 `INPUT`（发给本机的流量）
  跳转过去，源接口都是 `ora-nodes` 的 bridge。规则按顺序：
  1. 已建立的连接放行；
  2. 发给本机的一律拒绝（宿主不在网络上提供任何服务给容器）；
  3. 放行到 Cloud 入口（部署配置给出的地址和端口）；
  4. 放行到管理员白名单（地址或 CIDR 加端口）；
  5. 放行到 DNS 解析器（部署配置给出的地址，53 端口）；
  6. `restricted`：其余全部拒绝。`open`：拒绝私有、回环、链路本地、CGNAT 和组播地址（包括 `169.254.169.254` 这类元数据
     地址），其余放行。
- **两种模式由 Cloud 的部署配置决定**，宿主在注册时从 Cloud 拿到模式、Cloud 入口、白名单和 DNS 解析器（A2 D2），所有机器一致；
  配置变化时 Cloud 推给在线的宿主，宿主原子地替换规则。
- **装不上规则就不启动容器**：宿主启动或配置更新时规则装失败，宿主报告"网络规则不可用"，不接受任何启动指令，已有容器停掉。
- 容器里看到的 Cloud 入口主机名由宿主通过容器的 hosts 映射到部署配置的地址，TLS 仍按这个主机名校验。

为什么 `open` 也拒绝内网：`open` 是给演示和正常网络环境用的，让 Agent 可以装包、查资料；但 A1 D4 第 1 条的底线不分模式——
数据库、其他机器和宿主本机永远碰不到。为什么白名单只收地址不收域名：宿主按域名放行就得自己解析并持续跟踪解析结果，Agent 还能
借 DNS 重绑定绕过；企业内网的包镜像和代理地址通常是固定的，第一版只收地址足够（域名留在未解决问题）。

反例：`restricted` 下 Agent 执行 `curl http://10.0.0.5:5432`（数据库）→ 拒绝；`open` 下执行 `curl https://registry.npmjs.org` → 放行；
`open` 下执行 `curl http://169.254.169.254/` → 拒绝；任一模式下访问同一台机器上另一个项目的容器 → 拒绝（icc 关闭）。

### D4：容器里没有长期凭据

- 容器里唯一的凭据是 A2 的项目令牌，只在 `/run/ora`。模型 Key 和外部仓库凭据都不进容器（A1 D4 第 2、3 条）。
- 不提供 sandbox-server 那种任意环境变量注入。Agent CLI 访问模型只用 A5 的会话级临时令牌，由 Node 交给 Agent。
- 需要经代理出网时，代理地址作为出网配置的一部分（`HTTPS_PROXY`、`NO_PROXY`）由 Cloud 下发、宿主写进容器环境；这些值
  不能包含凭据，代理本身必须在白名单里。

为什么去掉环境注入：v1 用它把 `ANTHROPIC_API_KEY` 这类 Key 直接给 Agent，正是 A1 D4 要禁止的"Key 进沙盒"；保留一个能放
任意值的入口，就无法保证这一条。

### D5：隔离边界是项目

一个项目一个 Node 容器；同一个项目里不同用户的会话共用这个容器，每个会话有自己的工作目录。

接受的代价：同一个项目里，一个用户的 Agent 在技术上能读到其他会话的目录、同项目其他会话的临时模型令牌，以及项目令牌
（A2 D7）。团队成员本来就共享这个项目的仓库，所以项目就是信任边界。跨项目的隔离靠三点：每个项目一个容器、一个数据卷；
容器之间网络不通（D3）；项目令牌只能访问本项目（A2 D6）。

被否定：一个会话一个容器。它让同项目的会话也互相隔离，但每次新会话都要建容器、clone、装 Agent，等于回到 v1"每次运行
一个一次性沙盒"的成本（#1023：运行 Workspace 从建到删约 65 秒）。

### D6：镜像按 digest 固定，Agent CLI 打进镜像

- Node 镜像包含 Node、进程管理程序、Deno 和支持的 Agent CLI（OpenCode、Claude Code 等），在构建时准备好；v2 现在由
  `task v2:node:prepare` 准备的资源改为镜像构建的一部分（I5 #1055）。
- Cloud 的部署配置按 digest 指定 Node 镜像；宿主只按 digest 启动容器，不用 tag。
- **分发**：离线安装包里带上镜像，各台机器 `docker load` 装入（I9 #1059）。企业有自己的镜像仓库时，也可以配置宿主按 digest
  从那里拉取。不自建镜像仓库。
- 宿主在注册时上报本机有哪些 Node 镜像 digest，Cloud 不把项目放到缺镜像的机器上（A2 D4）。
- **升级**：部署配置换成新 digest 后，新启动的容器用新镜像；正在运行的容器不受影响，空闲回收后下次启动时自然换上。

为什么 Agent CLI 打进镜像：离线环境里装不了；每次启动再装会让首条输出变慢（v2 现在同一项目第一次执行要多等约一分钟安装 Agent）。

### D7：每个容器默认带资源上限

- 每个 Node 容器默认设置内存上限（同时禁用 swap）、CPU 上限、进程数上限（防止 fork 炸弹）和 `/tmp` 大小上限。默认值由
  部署配置给出，管理员可以调整；没有配置时用保守的默认值（例如 4 GiB 内存、2 个 CPU、4096 个进程）。
- 宿主按"本机可分配的内存和 CPU ÷ 每个容器的上限"计算容量，上报给 Cloud（A2 D2、D4）。
- 超出内存上限时，内核杀掉容器里的进程；Node 管理进程被杀等于 Node 退出，按 A2 D3 处理为中断。

数据卷的磁盘用量第一版不设硬上限（默认的 Docker 卷驱动不支持配额），宿主上报每个数据卷的用量，由管理员处理；见未解决问题。

## 被取代的 v1 决策

继承 A1"被取代的 v1 决策"中沙盒一组的安排：下列 ADR 在 I11（#1061）删除 sandbox-server 时标为 `superseded`，取代者为本文
（模型身份一份同时由 A5 取代），在那之前状态不变。

| v1 ADR | 被本文哪一条取代 |
| --- | --- |
| [`cluster/sandbox-server/0-docker-label-resolver.md`](../../cluster/sandbox-server/0-docker-label-resolver.md) | D1（宿主按标签识别自己的容器） |
| [`cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md`](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) | D1（不要 effect、许可、围栏和墓碑） |
| [`cluster/sandbox-server/20260928-agent-run-sandbox-environment.md`](../../cluster/sandbox-server/20260928-agent-run-sandbox-environment.md) | D3（两种出网模式）、D4（不再任意注入环境） |
| [`cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md`](../../cluster/sandbox-server/20261010-model-client-identity-replaces-shared-model-key-injection.md) | D4，以及 A5（模型访问改为项目令牌加会话临时令牌） |
| [`cluster/sandbox-server/20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md`](../../cluster/sandbox-server/20261014-controller-joins-sandbox-network-and-dials-nodes-over-mutual-tls.md) | D2（只经本机 Unix socket，不发布端口）、A2 D1 |
| [`cluster/sandbox-server/20261016-controller-connects-through-sandbox-websocket-router.md`](../../cluster/sandbox-server/20261016-controller-connects-through-sandbox-websocket-router.md) | D2、A2 D1（不要 WebSocket 路由） |

## 不变量

1. 一台机器上同一个项目最多有一个 Node 容器。
2. 工作负载进程（UID 1000）读不到 `/run/ora` 里的任何文件，也碰不到 Docker socket 和 `/run/ora-control`。
3. Node 容器不发布端口，不挂载宿主上除 `/run/ora-control` 对应目录以外的路径。
4. 两种出网模式下，Node 容器都访问不到私有、回环和链路本地地址，除非它是 Cloud 入口、白名单或 DNS 解析器。
5. 不同项目的 Node 容器之间网络不通。
6. 网络规则没有装好时，这台机器上没有运行中的 Node 容器。
7. 容器里没有模型 Key 和外部仓库凭据；唯一的凭据是项目令牌。
8. Node 容器只按 digest 启动镜像。
9. 每个 Node 容器都设置了内存、CPU 和进程数上限。

## 为什么不是这些替代方案

| 方案 | 优势 | 否决理由 |
| --- | --- | --- |
| 复用 sandbox-server | 已实现、已验收，加固已经做过 | 它的 effect、许可、围栏、证书和路由都是为 v1 的"恰好一次"和 mTLS 设计的，A2 已经不需要；留着它就多一个服务和一套状态目录（#1043 Q2 E1） |
| 一个会话一个容器 | 同项目的会话也互相隔离 | 每次新会话都要建容器、clone、装 Agent，回到 v1 一次性沙盒的成本（D5） |
| 网络策略交给企业防火墙，宿主不装规则 | 宿主不需要 `NET_ADMIN` | 企业防火墙通常管不到同一台机器上容器之间、容器到本机的流量；`open` 模式下内网拒绝也没人保证（D3） |
| 白名单按域名 | 管理员配置更方便 | 宿主要持续解析并跟踪变化，Agent 能借 DNS 重绑定绕过；第一版只收地址（D3） |
| 保留环境变量注入 | Claude Code、Codex 现在就能用 Key 直接跑 | 违反 A1 D4"Key 不进沙盒"；改由 A5 的临时令牌（D4） |
| rootless Docker 或 user namespace 重映射 | 容器逃逸后拿不到宿主 root | 宿主仍要改 iptables；Node 的进程管理在 user namespace 下的行为还没验证。留在未解决问题 |
| gVisor、Kata 这类强隔离运行时 | 内核攻击面更小 | 企业机器上不一定能装，性能和兼容性要另测；第一版先用标准 Docker 加固 |

## 本决策未解决的问题

1. **白名单按域名**：第一版只收地址或 CIDR。需要按域名放行时另写决策，至少回答解析由谁做、多久刷新、怎么防重绑定。
2. **数据卷磁盘配额**：默认卷驱动不支持配额，第一版只上报用量。需要硬上限时，由后续决策选择支持配额的存储方式。
3. **更强的隔离**：rootless Docker、user namespace 重映射、gVisor 等，等企业部署环境明确后再决定。

## 落地顺序

1. **I5（#1055）Node 镜像**：把 v2 的 Node 资源搬进镜像构建，确定能力最小集合和只读根文件系统下需要的可写位置。完成条件：
   用镜像在本机起一个容器，跑通一次对话；工作负载读不到 `/run/ora`。
2. **I4（#1054）宿主的容器管理和网络规则**：D1 的起停和标签、D3 的规则、D7 的上限。默认关闭，演示路径不变。完成条件：两种模式
   各跑一遍 D3 的反例；规则装失败时不启动容器。
3. **I9（#1059）离线包**：带上镜像，按 digest `docker load`。
4. **I10（#1060）**：演示切到内嵌宿主加容器，用 `open` 模式。从这一步开始，演示的 Agent 跑在容器里。
5. **I11（#1061）**：删除 sandbox-server，按上一节标记 v1 ADR。
