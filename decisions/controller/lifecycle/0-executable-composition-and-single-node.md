---
status: implemented
---

# Controller 可执行入口的组合：配置文件承载部署状态，启动参数承载组合，`--single-node` 托管生命周期独立的 Node

`ora-controller` 的启动输入分为两类：配置文件保存部署状态（数据目录、受保护目录、
`ControllerId`、Node 列表、会话与重连参数、时区、API 目标 Node），启动参数保存本次进程的组合
方式（监听 Transport、是否托管 Node）。`--single-node` 让 Controller 启动配置指定的 `ora-node`，
等待其 IPC endpoint 就绪后按既有本机 IPC 连接；Node 与 Controller 同属一个进程组，但 Controller
崩溃不连带终止 Node，正常关停时向 Node 发送终止信号并等待其完成受管 Git 收尾。process host 与
guardian 不由 Controller 部署或启动。

本决策处于 `implemented` 状态：desktop#595（`7b7fea96`）实现了配置与启动参数分工、`--single-node`
托管、关闭顺序与启动器职责调整，直接证据登记在
[生命周期核心用例](../../../test-cases/controller/lifecycle/executable-composition-and-single-node.md)。
本文覆盖 `apps/ora-controller` 的入口、配置校验、Node 子进程生命周期与
关闭顺序，以及 minicloud 开发启动器的职责调整。它不改变 Controller 持久接管、Node IPC 协议、
process host／guardian 的部署规则，也不引入进程监督、自动重启或多 Node 编排。API 监听的
Transport 参数含义由[Client API 决策](../api-boundary/0-controller-hosted-client-api.md)定义。

## 继承与修改

| 来源与约定 | 本文处理 |
|---|---|
| [本机 Controller 归属及持久接管](../persistence/0-local-controller-node-ownership.md)：Node 在私有 Unix socket 监听、Controller 主动连接、断连不取消执行、两侧可独立重启 | 继承，不变；托管只改变谁启动 Node 进程，不改变会话、归属或恢复规则 |
| 同决策：数据目录显式注入、与 Node／host／guardian 状态目录隔离 | 继承，不变；`--single-node` 不放宽目录重叠检查 |
| [minicloud 根决策](../../minicloud/runtime/0-local-web-client-embedded-controller.md) D3：开发启动器正常停止先让 Node 收尾，再停止 host 与 guardian | 修改：Node 的启动与收尾由 `ora-controller --single-node` 承担，host 与 guardian 仍由启动器承担；停止顺序不变 |
| [Node host 与 scope guardian 架构](../../node/process/architecture/0-node-host-and-scope-guardians.md)：guardian 按内容哈希部署到私有目录，host 以 create／recover 模式启动 | 继承，不变；这些是 Node 执行环境的前置条件，Controller 不复制其部署逻辑 |
| 现有 `ora-controller <absolute-config-file>` 位置参数入口 | 修改：改为 `--config <absolute-path>` 加显式组合参数；配置文件内容不承载组合参数 |

## 决策

### D1：配置文件与启动参数分工

| 输入 | 内容 | 校验时机 |
|---|---|---|
| `--config`（必填，绝对路径） | 数据目录、受保护目录、`controller_id`、`nodes`、`session`、`reconnect_ms`、`timezone`、`api.node_id`、可选 `single_node` 段 | 打开数据库前完成全部结构与目录重叠校验 |
| `--transport`／`--host`／`--port`／`--socket` | 本次进程的监听方式 | 打开数据库前拒绝非法组合 |
| `--single-node` | 本次进程是否托管 Node | 要求配置存在 `single_node` 段、`nodes` 恰有一项且等于 `api.node_id`；不满足即拒绝启动 |

配置文件里不出现 `listen` 之类的组合字段，启动参数里不出现数据目录之类的部署状态。这样同一份
配置可以被独立 Controller、托管 Node 的开发组合和未来的服务部署复用，而组合差异只体现在
进程启动命令上。所有校验在获取数据库租约前完成，避免因参数错误留下半初始化状态。

### D2：`--single-node` 托管 Node 的进程语义

`single_node` 段包含 `node_executable`（`ora-node` 绝对路径）、`node_config`（Node 配置文件绝对
路径）、`ready_timeout_ms` 与 `stop_timeout_ms`。Controller 启动前只读取 Node 配置做一致性交叉
检查：`ipc.controller_id` 等于本 Controller 的 `controller_id`，`ipc.endpoint` 等于 `nodes[0].endpoint`。
不一致即拒绝启动，不修改 Node 配置，也不为 Node 生成配置。

启动顺序为：完成上述校验并探测 endpoint（已有进程接受连接则拒绝启动，不接管来历不明的 Node）→
打开 Controller 状态 → 在 Controller 自身进程组内启动 `ora-node <node_config>`（不新建会话），
标准输出／错误继承给 Controller 的终端 → 在 `ready_timeout_ms` 内等待 `nodes[0].endpoint` 可连接 →
启动 API 监听与既有重连循环。就绪超时视为启动失败：Controller 向 Node 发送终止信号并退出，
不进入受理状态。

崩溃隔离不依赖新会话：进程组成员关系不会在父进程死亡时传递信号，Controller 被强杀时 Node 及其
受管 Git 继续运行，已接受的执行按既有规则由重启后的 Controller 恢复。保留同一进程组是为了让
启动器或运维按进程组发出的停止信号同时到达两者——尤其是 Controller 已经崩溃、只剩 Node 的情况，
否则 Node 会成为没有任何持有者知道其 PID 的孤儿。Node 意外退出时，Controller 记录原因并执行
正常关停后以失败退出：托管组合只对“Controller 与它启动的 Node”成立，Node 消失后继续受理只会
产生无法派发的接受记录，交给启动器或运维处理更明确。

Node 需要 process host 已在其配置的 `process.host_directory` 内运行，guardian 已按哈希部署；
这是 `--single-node` 的前置条件，Controller 不启动 host、不部署 guardian、不校验其版本。
原因是这两者属于 Node 执行环境的部署与版本共存规则，复制到 Controller 会形成第二套编排。

### D3：关闭顺序与超时

正常关停按下列顺序执行，每一步完成后才进入下一步：

1. 停止 API 受理（不再接受新请求，等待在途请求在有限超时内结束）。
2. 关闭 Controller 到 Node 的会话；已接受的执行不受影响。
3. 仅在 `--single-node` 下向 Node 发送 `SIGTERM`，等待至多 `stop_timeout_ms`；Node 在此期间
   完成既有的受管 Git 收尾。
4. 释放数据库租约并退出。

`stop_timeout_ms` 到期时 Controller 记录告警并继续退出，不升级为 `SIGKILL`：强杀 Node 会绕过
它的受管进程收尾，而残留进程的遏制责任属于 host／guardian；启动器如需升级信号，由它在停止
host 前自行决定。先停 API 再停会话再停 Node，是为了保证不会出现“已向调用方报告接受、但 Node
已被要求退出”的窗口。

### D4：minicloud 开发启动器的职责收缩

启动器保留初始化、构建、guardian 部署、host 启动／恢复、Vite 启动与整体停止；不再直接启动
`ora-node` 与 server，改为启动
`ora-controller --config <controller.json> --single-node --transport tcp --host 127.0.0.1 --port <port>`。
启动器写入的 Controller 配置包含 `single_node` 段，其中 `node_config` 指向启动器生成的
`node.json`。停止时先停 Vite，再向 Controller 发送终止信号并等待（Controller 内部完成 Node 收尾），
最后停止 host 与 guardian。第二个启动器对同一数据目录的独占拒绝、状态路径必须位于启动器
目录内、端口必须为回环等既有规则不变。

## 不变量

1. 配置文件只承载部署状态，启动参数只承载组合方式；两者的非法值都在获取数据库租约前拒绝。
2. `--single-node` 启动的 Node 与 Controller 同进程组；Controller 崩溃不终止 Node 或其受管 Git，进程组级停止同时到达两者。
3. 正常关停顺序固定为 API 受理 → Node 会话 → 托管 Node → 数据库租约；不向 Node 升级 `SIGKILL`。
4. Controller 不生成、不修改 Node 配置，不部署 guardian，不启动 host。
5. 托管 Node 意外退出时 Controller 不继续受理，且不清空或改写已接受记录。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 在 Controller 进程内以任务形式运行 Node | 少一个进程，但 Controller 崩溃即丢失 Node 的受管 Git 进程归属，违反“断连不取消执行”与两侧独立重启 |
| 以新会话（`setsid`）完全脱离 Node | 隔离最强，但 Controller 崩溃后没有任何持有者知道 Node 的 PID：启动器按进程组停止到不了它，下一次启动因 endpoint 仍活跃被拒绝 |
| Controller 同时部署 guardian 并启动 host | 一条命令拉起全栈，但复制了 host／guardian 的版本共存与恢复规则，形成第二套编排 |
| Controller 监督并自动重启 Node | 对长期部署有价值，但重启策略、退避和状态判定超出当前范围；先由启动器或运维决定 |
| 把 `listen`／`single_node` 全放进配置文件 | 少几个参数，但同一部署配置无法在不同组合方式间复用，且启动器需要为每种组合改写文件 |

## 本决策未解决的问题

- 非 Linux 平台的托管方式（Windows 没有等价的进程组信号语义）。
- 多 Node 部署下的托管与目标选择。
- 托管 Node 的自动重启策略与日志路由。
- 服务化部署（systemd、容器）中 host／guardian 与 Controller 的启动依赖表达。

## 落地结果

- 入口参数与 `--single-node` 均由 desktop#595 实现；Controller 持久接管测试、独立进程验收与
  minicloud 真实链路测试在新入口上通过，`apps/ora-controller/tests/composition.rs` 以 Node 替身
  覆盖就绪超时、停止超时与 Node 意外退出等真实 Node 难以稳定制造的路径。
- 与最初提案的偏离：提案阶段设想以 `setsid` 让托管 Node 完全脱离，实现时发现 Controller 崩溃后
  没有任何持有者知道 Node 的 PID，启动器按进程组停止到不了它；最终采用 D2 记录的同进程组方案，
  崩溃隔离由"进程组关系不在父进程死亡时传递信号"保证。
- 核心用例中仍为 `Partial` 的两项如实保留：在途 HTTP 请求的有限等待、启动器自身 Vite →
  Controller → host 顺序的自动化断言（目前为手工验收）。
