---
status: approved
---

# minicloud：本机 Web Client 与内嵌 Controller

新建 `apps/minicloud/client` 和 `apps/minicloud/server`，以非生产本机 Web 应用承载
clone 指定仓库与分支的最小闭环及短期需求。client 使用 Vite、React 19、shadcn；
server 是内嵌 Rust Controller 的薄 HTTP 入口，Node 继续作为独立进程运行。

本 ADR 已于 2026-09-20 获 Eric 批准，状态为 `approved`；方向和落地约束已确认，不表示代码已实现。
首版沿用 Linux 本机执行环境，不接入 Desktop、不切换既有 Backend 写入入口，
不迁移用户 checkout 或重建已有 Controller／Node／process 数据。minicloud 不投入生产环境。

## 背景与继承

Node 受管 clone、本机 IPC 与 Controller 持久协调已具备实现；缺少用户提交与观察入口。
当前需要尽快跑通真实业务链路，而不是先承担 Desktop 集成或正式 Cloud 的部署复杂度。

| 依据 | 本决策处理 |
|---|---|
| [clone 根决策](../../node/repository/0-clone-selected-repository-branch.md)及[执行契约](../../node/repository/20260918-minimal-clone-execution-contract.md) | 继承输入、独占目录、Git 内容与恢复规则；首个用户入口改为 minicloud，而不是 Desktop |
| [本机 Controller 归属及持久接管](../../controller/persistence/0-local-controller-node-ownership.md) | 继承持久接受、原执行重传及接管后 Ack；HTTP server 编译并调用同一 Controller 实现 |
| [Client 契约提案](../../desktop/protocol/0-client-controller-contract.md) | 沿用异步接受、查询及幂等意图；不据此引入 Worktree 业务、订阅或 Desktop IPC |
| [正式 Cloud 内部控制提案](../../controller/protocol/0-internal-control-protocol.md) | 不用于 minicloud；不引入 Go API Server、Controller 子进程控制协议或第二套权威数据库，也不改写该提案的正式 Cloud 范围 |

## 决策

### D1：minicloud 是持续承载短期需求的独立应用

目录按 client／server 分开。client 提供输入与展示，server 提供 HTTP 适配及运行时组合。
不依赖 Tauri、Desktop bindings 或旧 Backend 的写入入口；也不为演示在 HTTP handler 内直接执行 Git。
短期新增需求沿用实际业务所有者的接口，不提前引入通用任务框架或完整 Cloud 模型。

### D2：server 内嵌 Controller，复用唯一协调实现

```text
浏览器 → Vite dev server → proxy → minicloud server（Controller）
                                      ↓ 本机 IPC
                                     Node → host／guardian → Git
```

Controller 继续拥有接受、幂等、执行关联、结果接管和恢复规则；HTTP 层只做请求／响应转换与组合。
已有重连、关闭和部署配置逻辑应通过可复用 Controller 运行时提供，不在 minicloud 复制实现。
server 不另建任务状态数据库，不直接操作 Controller 内部表，也不将 Node 内部消息直接当成浏览器契约。
所需提交、列表和结果查询能力由 Controller 提供；只扩展实际消费者需要的接口。

同一 Controller 数据目录只能由一个运行时拥有，不能同时启动独立 `ora-controller` 和内嵌实例。
Controller 身份、数据目录及 Node endpoint 均由部署显式注入；保留已有数据目录隔离和独占规则。
server 关闭会停止 HTTP 受理和本身的 Controller 会话，不因浏览器或 HTTP 断连取消 Node 已接受的执行。

### D3：仅本机访问，不建设认证和安全体系

server 监听回环地址，Vite 同样供本机使用；浏览器通过 Vite proxy 访问 HTTP 入口，
不要求浏览器直连后端端口。端口作为开发配置，不在业务契约中固定。

不增加登录、用户／租户、令牌、权限、TLS、额外安全中间件或生产加固；
不为 minicloud 接入正式 Cloud 的认证与授权系统。这里的本机监听是部署约束，
不是“本机即可信”的安全保证，应用不对恶意本机调用方提供隔离承诺。
不监听所有网络接口、不新增生产部署或公网暴露方案。

不建设安全体系不改变业务正确性要求：输入契约、请求冲突、持久接受、数据目录保护和恢复规则继续生效。
私有仓库凭据仍由 Node 的既有部署提供，不在浏览器收集或另建凭据管理功能。

### D4：首条闭环是提交 clone 与持续查询

用户输入仓库 URL 和分支；目标 Node 来自显式部署配置，不引入调度器。
Controller 持久接受后返回稳定请求／执行关联，HTTP 接受不等于 clone 已完成。
接受响应丢失时用原请求身份重传，同身份不同输入返回冲突，不自动另建执行。

client 首版通过轮询展示操作列表和结果，不引入 WebSocket 或订阅协议。
成功展示 Node 返回的路径与实际 commit；失败展示明确失败及保留残留的事实。
未完成、未知或暂时无法查询不得伪装成失败；页面刷新或 server 重启后可重新读取原接受记录。
列表中的“操作”是 Controller 已接受的 clone 记录，不据此创建新的 Task／Workspace 领域实体。
展示的路径属于 Node，不表示浏览器能够直接打开该路径。

具体 HTTP 路径、DTO 与错误映射在该范围内收敛为最小契约，不要求提前统一正式 Cloud 与 Desktop 的所有协议。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 先接入 Desktop | 最终桌面体验更直接，但增加 Tauri、bindings 与旧入口衔接工作，不符合当前闭环顺序 |
| 只有 CLI 或一次性 clone 页面 | 验证成本更低，但无法承载已明确的短期交互需求与恢复展示 |
| HTTP server 另启 Controller 子进程 | 生命周期隔离更强，但增加内部协议；当前可以直接复用 Rust 库，无须新增一跳 |
| server 自己存任务或执行 Git | 容易得到演示结果，但绕开已完成的协调和进程恢复，产生两套事实 |
| 先建设正式 Cloud 与认证体系 | 适合生产演进，但 minicloud 明确不投产，不承担这些义务 |

## 不变量

1. minicloud 是非生产本机应用，client 经 Vite proxy 访问仅监听回环的 server。
2. 一个 Controller 实现拥有协调规则与持久记录；HTTP 层不另造执行或持久事实。
3. 接受、完成和暂时不可知可区分；重传和刷新不制造第二次 clone。
4. 浏览器断连不取消已接受执行；重开使用原 Controller 数据与执行身份恢复。
5. Desktop、旧 Backend、现有用户文件与既有执行记录保持不变。

## 风险、未解决的问题与落地

server 与 Controller 同进程，共同故障；恢复依靠已有持久接受和 Node 未确认结果，而非 HTTP 连接存活。
非生产用途不意味着已有用户文件可删除或状态库可清空。缺失结果也不能通过重新 clone“修复”。

正式 Cloud、远程访问、认证授权、多租户、生产部署、Desktop 集成与 Backend 切换留给未来 ADR，
不由 minicloud 的实现暗中扩展。首版也不增加取消、自动重试新执行、目录复用或自动清理功能。

落地按三个可独立检视的阶段推进：先提供 Controller 可嵌入运行时及最小查询接口，
再接通本机 HTTP 与 Vite proxy，最后完成 clone 提交／查询页面及刷新、重启后的真实闭环验证。
核心测试义务见[本机 Web 闭环验收](../../../test-cases/minicloud/runtime/local-web-clone-loop.md)。
当前尚无 minicloud 直接验收证据，不将已有 Node／Controller 测试冒充本应用验收。
