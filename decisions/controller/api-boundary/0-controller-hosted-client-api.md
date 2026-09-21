---
status: proposed
---

# Controller 可执行入口承载 Client API：显式 Transport、过渡 JSON 接口与 proto 定义的 Cloud 契约

`ora-controller` 可执行入口自身承载面向调用方的 API 监听，`apps/minicloud/server` 不再作为独立
程序存在。监听 Transport 由启动参数显式选择：`tcp`（`--host`／`--port`）或 `unix`（Controller
数据目录内的私有 socket），默认只监听回环地址，不默认监听所有网络接口。现有 clone 提交／查询的
JSON HTTP 接口作为过渡契约随入口迁移；Go Cloud（API Server）与 Controller 之间的正式契约以本仓库
拥有的 `.proto` 定义，运行在同一监听器上的 gRPC over HTTP/2 之上，两侧均从同一份 proto 生成代码，
不再人工对齐字段。浏览器不直接使用 gRPC。

本决策处于 `proposed` 状态，覆盖 `apps/ora-controller`、被移除的 `apps/minicloud/server`、
`ora-contracts` 中的过渡 DTO，以及 Cloud↔Controller 契约的定义与生成方式。它不定义认证、TLS、
租户边界、Internal Control API 的具体消息集合、PTY 或其他流式执行交互，也不改变
Controller–Node 协议。可执行入口的启动参数拆分、`--single-node` 与进程生命周期由
[可执行入口组合决策](../lifecycle/0-executable-composition-and-single-node.md)承担。

## 继承与修改

| 来源与约定 | 本文处理 |
|---|---|
| [本机 Controller 归属及持久接管](../persistence/0-local-controller-node-ownership.md)：持久接受、派发前落盘、接管后 Ack、Node 本机 IPC | 继承，不变；API 层只调用 `ControllerHandle`，不触碰 SQLite、会话或重连实现 |
| [minicloud 根决策](../../minicloud/runtime/0-local-web-client-embedded-controller.md) D2：独立 `apps/minicloud/server` 内嵌 Controller | 修改：不再有独立 server 程序，API 由 `ora-controller` 可执行入口承载；协调规则唯一实现的要求不变 |
| minicloud 根决策 D3：server 拒绝非回环监听 | 修改：回环成为可执行入口的默认值，非回环监听需显式 `--host`；minicloud 开发部署继续只用回环，见[minicloud 后续决策](../../minicloud/runtime/20260921-controller-hosted-server.md) |
| [Client–Controller Contract 提案](../../desktop/protocol/0-client-controller-contract.md)：异步接受、幂等重传、查询 | 继承其语义；本文仍不引入 Subscription、Worktree 业务或 Desktop IPC |
| [Internal Control Protocol 提案](../protocol/0-internal-control-protocol.md)：API Server 是权威存储，Controller 不接数据库 | 继承；本文回答其未决项中的“Transport 选择”，不定义其消息集合 |
| 代码仓库 `AGENTS.md` 的 contracts 单一来源规则 | 继承并按边界解释：浏览器／Desktop DTO 的来源是 `ora-contracts`，Cloud↔Controller 契约的来源是 `.proto`；两者不互相生成，也不各自维护对方的副本 |

## 决策

### D1：`ora-controller` 是 API 的唯一宿主，库保持与传输无关

`ora-controller` crate 的库部分（`ControllerRuntime`、`ControllerHandle`）不依赖 HTTP、gRPC 或
任何监听器；API 层是可执行入口内的适配模块，只把请求转换为 `ControllerHandle` 的调用并把
结果投影为契约 DTO。接受、幂等、冲突、缺失与暂时不可用的语义和错误映射与 minicloud 现有
实现相同：接受只在 Controller 持久提交后返回，缺失记录与“已接受但无终态”明确区分，持久化
失败映射为暂时不可用而不是 clone 失败。

派发目标 Node 来自部署配置 `api.node_id`，必须是 `nodes` 中已配置的 Node；调用方不指定 Node，
也不引入调度器。这样做是为了让 minicloud 的“目标 Node 来自显式部署配置”规则在迁移后原样成立。

同一 Controller 数据目录仍只允许一个运行所有者；API 宿主与协调运行时同进程，关闭时先停止
API 受理，再关闭 Node 会话，最后释放数据库租约。API 断连或调用方退出不取消 Node 已接受的执行。

### D2：Transport 由启动参数显式选择，默认回环

监听 Transport 是可执行入口的组合参数，不是持久部署状态，因此放在命令行而不是配置文件：

| 参数 | 适用 | 规则 |
|---|---|---|
| `--transport tcp` | 默认 | `--host` 默认 `127.0.0.1`，`--port` 默认 `4820`；不使用 4317／4318（OTLP 默认端口）或 minicloud 的 Vite 端口 |
| `--transport unix` | 本机调用方 | `--socket` 必须是绝对路径且直接位于 Controller 数据目录内，沿用 Node endpoint 的私有 socket 创建与恢复规则（不覆盖普通文件、符号链接或仍在使用的 socket） |
| 参数组合 | 全部 | `unix` 携带 `--host`／`--port`，或 `tcp` 携带 `--socket`，启动即拒绝；Transport 用枚举表达，不接受布尔开关 |

非回环 `--host` 允许启动，但当前没有认证，入口必须记录明确警告；这是部署约束，不是安全保证。
认证落地后非回环监听改为要求凭据配置，该收紧留给认证 ADR。后续新增 Transport（例如 Windows
named pipe）以枚举变体加入，同样不提供隐式网络默认值。

### D3：JSON HTTP 接口是过渡契约，不是 Cloud 契约

现有 `POST /api/clones`、`GET /api/clones`、`GET /api/clones/{execution}` 及其 DTO 随入口迁入
`ora-controller`，DTO 从 `ora_contracts::minicloud` 更名为 `ora_contracts::controller_api`，
浏览器类型继续由 ts-rs 生成。它的消费者只有 minicloud client（经 Vite proxy）和测试；
Go Cloud 不使用该接口，浏览器经 Go Cloud 接入后该接口连同 DTO 一起删除，不做长期兼容承诺。

保留它是为了让迁移阶段的行为可以被现有真实链路测试逐字验证，而不是为了给 Cloud 提供第二套
契约。

### D4：Cloud↔Controller 契约由本仓库拥有的 proto 定义，双方生成代码对齐

Cloud 与 Controller 之间使用 gRPC over HTTP/2，与 D2 的监听器共用同一进程和同一 Transport
（`tcp` 上为明文 h2c，`unix` 上为 h2；TLS 留给认证 ADR）。选择 gRPC 而不是自定义 h2 协议或
JSON 流，是因为跨 Rust／Go 的双向流、取消、截止时间、流控和消息边界已经由它定义，自研只会
得到一个不完整的复制品。

契约来源与生成规则：

- `.proto` 放在代码仓库 `proto/ora/controller/v1/` 下，由 Controller（服务端）拥有；Cloud 仓库
  不复制 `.proto`，以固定的 git tag 作为生成输入。
- 两侧都提交生成代码：Rust 生成到专用 crate，Go 生成到 Cloud 的内部包；CI 重新生成后
  diff 必须为空。生成器为 `buf`，`buf lint` 约束命名与 package 版本后缀，`buf breaking`
  以主干为基线阻止破坏性修改；破坏性变更只能通过新增 `v2` package 表达，`v1` 保留。
- 错误分类以 proto 枚举携带在 gRPC status details 中，Rust 的 `ora_controller::Error` 到该
  枚举的映射只写一次；Go 侧按同一枚举分支，不解析错误字符串。
- 一次契约变更的流程是：本仓库修改 `.proto` → 生成 → 合并并打 tag；Cloud 仓库更新 tag →
  生成 → 编译。编译失败即不对齐，无需人工比对字段。

单一来源规则按边界成立：浏览器不消费 gRPC，因此 `ora-contracts` 与 `.proto` 描述的是两个
不同边界，各自唯一；禁止为同一 DTO 同时维护 serde 结构、proto message 与手写转换。

契约对齐只证明结构一致；行为一致由 Cloud 仓库的集成测试以真实 `ora-controller --transport unix`
替换其 Controller 替身来验证，该义务在契约稳定后登记。

### D5：调用方向与流式交互的边界

Cloud 场景中 Controller 领取和推进操作的方向（Controller 拨入 API Server 的 `/internal/v1`）
由 Cloud 的 `controller-integration` 决策拥有，本文不改变它。本文的监听器服务于 API Server
向 Controller 发起的调用（当前为过渡 JSON 接口的同类操作，未来为执行交互）以及本机调用方。
单集群部署中 API Server 主动拨号 Controller；Controller 位于 NAT 之后或需要动态寻址时，
改为 Controller 拨出并在同一 gRPC 契约上使用服务端流下发命令，该反转不改变契约，留作未决项。

浏览器面向 Cloud 的接口保持 JSON 与 SSE；PTY 等双向流式交互需要 Controller–Node 协议先具备
流式消息，本文不为其预留 gRPC 服务定义。

## 不变量

1. 面向调用方的 API 只由 `ora-controller` 可执行入口承载；库不依赖任何监听器或序列化传输。
2. API 层不另造执行或持久事实；接受、冲突、缺失和暂时不可用的语义与 Controller 持久记录一致。
3. Transport 由启动参数显式给出，默认回环；非法参数组合在打开数据库前拒绝。
4. Cloud↔Controller 契约唯一来源是本仓库的 `.proto`；两侧生成代码与之无漂移，破坏性修改不进入 `v1`。
5. 同一 DTO 不存在 serde、proto 与手写转换三份并行定义。
6. 调用方断连不取消 Node 已接受的执行，也不改变 Controller 持久状态。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 保留独立 `apps/minicloud/server` | 部署边界更清晰，但它已只剩组合代码；两个可执行入口共享同一运行时会让 Transport、关闭顺序和配置校验出现两份 |
| 默认监听 `0.0.0.0` | 对远端 API Server 更省配置，但当前没有认证；默认暴露与 minicloud 已批准的回环约束冲突 |
| `.proto` 放在 Cloud 仓库或独立仓库 | 放 Cloud 会让服务端跟随调用方的契约；独立仓库使每次接口变更需要三处 PR，当前只有一个消费者 |
| Rust 侧用 `build.rs` 在编译期生成 | 不会漂移，但要求所有贡献者安装 `protoc`；提交生成代码与仓库已有的 ts-rs 漂移检查一致 |
| Cloud↔Controller 用 JSON／NDJSON over HTTP/2 | 可继续用 `ora-contracts`，但需要自定义流生命周期、取消、心跳和背压映射；浏览器并不消费这条边界，复用收益不存在 |
| Cloud↔Controller 用 WebSocket | 真双向，但要求 Gateway 增加升级转发；gRPC 在 Go／Rust 侧的生成与流语义更完整 |
| 浏览器直接使用 gRPC-Web | 全链路统一，但需要代理层且不等价于原生双向流；浏览器契约由 Cloud OpenAPI 拥有 |

## 本决策未解决的问题

- 认证、TLS 与非回环监听的凭据要求，留给认证 ADR。
- Internal Control API 的具体消息集合、领取／提交身份与 fencing 字段，仍由
  [Internal Control Protocol 提案](../protocol/0-internal-control-protocol.md)承担。
- NAT 或动态寻址下由 Controller 拨出的反向连接。
- PTY、Session 等流式执行交互及其所需的 Controller–Node 流式消息。
- 多 Node 部署下的目标选择；当前 `api.node_id` 固定单一目标。

## 落地顺序

1. 入口迁移：`ora-controller` 承载 JSON 接口与 Transport 参数，删除 `apps/minicloud/server`，
   更名过渡 DTO，minicloud 启动器改为启动 `ora-controller`。完成条件：minicloud 现有真实
   HTTP／Vite proxy／强杀恢复测试在新入口上逐项通过，Controller 的持久接管测试不变。
   此阶段调用方可观察行为不变，只有可执行程序名与启动参数改变。
2. proto 契约：Go Cloud 开始调用 Controller 时加入 `proto/`、生成 crate、`buf` 漂移与破坏性
   检查，gRPC 与 JSON 接口共用监听器。完成条件：Cloud 仓库以 tag 生成并编译通过，错误枚举
   在两侧映射一致。
3. 过渡接口退役：浏览器经 Go Cloud 接入后删除 JSON 接口与 `ora_contracts::controller_api`。

进入 `approved` 后，在 `specs/test-cases/controller/api-boundary/` 登记 Transport 拒绝、回环默认、
接口语义与 Controller 记录一致、契约生成无漂移等核心用例，并把 minicloud 用例中指向
`apps/minicloud/server` 的证据改指新入口。
