# Node 容器：加固布局、两种出网模式、凭据、镜像与资源上限

依据 [Node 容器根决策](../../../decisions/cloud/node-container/0-host-managed-docker-and-two-egress-modes.md)（`approved`）。
下文的 D 编号与不变量编号都指这篇根决策。

截至 cloud `ad324c77`，v2-server 的 Node 是和后端同一用户的本地子进程，没有容器，也没有宿主。下列义务都没有直接证据，
如实标为 **Missing**，由 I4 #1054（宿主的容器管理和网络规则）与 I5 #1055（Node 镜像）补齐。

## Workloads Cannot Read Node Management State Or Reach Docker

- 风险：Agent 或它启动的进程读到项目令牌和 Node 配置，碰到 Docker socket 或控制 socket，借此控制其他容器或宿主。
- 前置状态：一个运行中的 Node 容器，`/run/ora` 里有配置和项目令牌。
- 触发：以工作负载用户（UID 1000）读取 `/run/ora` 和 `/run/ora-control`；查找并连接 Docker socket；尝试写根文件系统；尝试提升权限。
- 必须成立：工作负载读不到 `/run/ora`，进不了 `/run/ora-control`；容器里没有 Docker socket；根文件系统只读；`no-new-privileges`
  生效，capability 只有 Node 需要的最小集合；容器不发布端口，不挂载宿主的其他路径。
- 禁止结果：工作负载读到任何 `/run/ora` 里的文件；容器里能访问 Docker API；工作负载取得 root。
- 验证义务与证据：**Missing** — 没有容器（I5 #1055 确定 capability 集合和可写位置，I4 #1054 按布局创建容器）。
- 决策依据：D2，不变量 2、3。

## Restricted Mode Allows Only Cloud, The Allowlist And DNS

- 风险：企业内网部署里，Agent 访问到数据库、其他机器、宿主本机端口或公网。
- 前置状态：部署配置为 `restricted`，给出 Cloud 入口、白名单和 DNS 解析器。
- 触发：从容器访问 Cloud 入口、白名单地址、DNS 解析器、数据库地址、另一台机器、宿主本机端口、公网地址、IPv6 地址。
- 必须成立：只有 Cloud 入口、白名单和 DNS 解析器能连通；其余全部被拒绝。
- 禁止结果：任何一个不在放行列表里的地址能连通。
- 验证义务与证据：**Missing** — 宿主还没有网络规则（I4 #1054）。
- 决策依据：D3，不变量 4。

## Open Mode Allows The Internet But Never Private Addresses

- 风险：演示或正常网络环境里放开公网时，顺带放开了内网，Agent 碰到数据库或元数据地址。
- 前置状态：部署配置为 `open`。
- 触发：从容器访问公网地址、私有地址、回环地址、链路本地地址（包括 `169.254.169.254`）、CGNAT 地址、宿主本机端口。
- 必须成立：公网可达；私有、回环、链路本地、CGNAT 和组播地址都被拒绝，Cloud 入口、白名单和 DNS 解析器除外。
- 禁止结果：任何内网地址能连通。
- 验证义务与证据：**Missing** — 宿主还没有网络规则（I4 #1054）。
- 决策依据：D3，不变量 4。

## Node Containers Of Different Projects Cannot Reach Each Other

- 风险：同一台机器上，一个项目的 Agent 访问到另一个项目的 Node 容器，跨过项目这条信任边界。
- 前置状态：同一台机器上有两个项目的 Node 容器。
- 触发：从一个容器向另一个容器的地址发起任意连接。
- 必须成立：连接被拒绝（容器间通信关闭）；每个项目有自己的数据卷，一个容器挂载不到另一个项目的数据卷。
- 禁止结果：连接成功；两个项目共用一个数据卷。
- 验证义务与证据：**Missing** — 没有容器（I4 #1054）。
- 决策依据：D3、D5，不变量 5。

## No Node Container Runs Without Network Rules

- 风险：宿主重启、iptables 被清空或规则替换失败后，容器在没有限制的网络上运行。
- 前置状态：一台有运行中 Node 容器的机器。
- 触发：宿主启动时规则安装失败；Cloud 推送新的出网配置时替换失败；规则被外部清空。
- 必须成立：规则不可用时，宿主报告"网络规则不可用"，拒绝启动指令，并停掉已有的 Node 容器；配置替换是原子的。
- 禁止结果：规则缺失或只装了一半时仍有 Node 容器在运行。
- 验证义务与证据：**Missing** — 宿主还没有网络规则（I4 #1054）。
- 决策依据：D3，不变量 6。

## No Long-Lived Credential Other Than The Project Token Enters The Container

- 风险：模型 Key、外部仓库凭据或其他秘密经环境变量、文件或代理配置进入容器，被 Agent 读走。
- 前置状态：部署配置了模型上游的 Key、外部仓库凭据和出网代理。
- 触发：启动 Node 容器；读取容器的环境变量和所有可读文件。
- 必须成立：容器里唯一的凭据是 `/run/ora` 里的项目令牌；代理设置不含凭据；没有任意环境变量注入的入口。
- 禁止结果：容器的环境或文件里出现模型 Key、外部仓库凭据或带凭据的代理地址。
- 验证义务与证据：**Missing** — 没有容器（I4 #1054）。
- 决策依据：D4，不变量 7。

## Containers Start Only Pinned Images Within Resource Limits

- 风险：容器按 tag 启动，不同机器跑了不同的镜像；没有资源上限，一个项目的 Agent 耗尽整台机器的内存或进程数，拖垮其他项目。
- 前置状态：部署配置按 digest 指定 Node 镜像，以及内存、CPU、进程数上限。
- 触发：启动容器；机器上缺这个 digest；容器内的进程超出内存上限或不断 fork。
- 必须成立：容器只按 digest 启动；缺镜像的机器不接收这个项目；每个容器都设置了内存（禁用 swap）、CPU 和进程数上限；
  超出内存上限导致 Node 管理进程退出时，按 A2 D3 处理为中断。
- 禁止结果：按 tag 启动；没有上限的容器；一个容器的资源耗尽让其他容器无法运行。
- 验证义务与证据：**Missing** — 没有容器和镜像（I5 #1055、I4 #1054）。
- 决策依据：D6、D7，不变量 8、9。
