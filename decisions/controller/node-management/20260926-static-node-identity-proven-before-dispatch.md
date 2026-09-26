---
status: proposed
---

# 静态 Node 先经握手证明身份，再领取租户级 clone：配错 `node_id` 时工作留在 Cloud 队列，而不是永久挂起

云端持久模式的 Controller 只在本进程内与静态配置的 Node 完成一次握手之后，才向 Cloud 领取租户级 clone
并以该 `NodeId` 调用 `RecordDispatch`。握手证明“配置里的 `node_id` 就是端点背后那个 Node”；证明之前，
已被 Cloud 接受的工作留在 Cloud 的队列里，不登记到任何 Node。这样，把 `node_id` 配错（联调问题 5）不再
留下永久 `pending` 的派发：改正配置或 Node 之后，排队的工作照常派发到正确的 Node。

本 ADR 处于 `proposed`，等待评审；实现已在 desktop 草稿 PR 中，评审通过前不补核心测试用例。涉及 desktop
`apps/ora-controller` 的云端适配器（`cloud/claim.rs`、`cloud/coordinate.rs`）与运行时的静态会话；Cloud
契约、Node、协议消息、SQLite 模式与 Workspace 沙盒路径都不变。兼容性边界：Cloud 中已登记的派发记录，
本文不改写、不迁移任何已有记录。

## 背景：问题 5 在三条路径上的现状

2026-09-25 联调发现：把 `node_id` 改错期间 API 接受的操作永久挂在不存在的 Node 上，改回配置后仍是
`pending`。根因是派发在登记时就以 `NodeId` 绑定目标，而这个身份此前从未被任何握手验证过。按现在的代码
核对三条路径：

| 路径 | 派发目标来自 | 是否先验证身份 | 问题 5 |
|---|---|---|---|
| Workspace 沙盒（云端 + `substrate`） | Cloud 记录的 `sandbox_ensure` 证据 | 是：clone 步骤等该 Node 握手、登记且心跳新鲜后才 `RecordDispatch`；等不到以 `node_unavailable` 延后 | 不存在 |
| 租户级 clone（云端，静态 `nodes`） | 部署配置 | 否：领取到工作就登记到配置的 `NodeId` | 存在，本文修复 |
| SQLite 与 JSON 表面（minicloud） | 部署配置（`api.node_id`） | 否：接受请求即登记派发 | 存在，本文不处理 |

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 根决策 D2：静态 `nodes` 只接收租户级 clone，沙盒 Node 只接收其 Workspace 的工作 | 继承，不变 |
| 根决策 D3：Workspace clone 在 Node 就绪（握手、登记、心跳）后才派发 | 继承，不变；本文把同一原则推广到静态 Node |
| 持久协调双适配器：云端适配器把租户级 clone 派发到唯一配置的 Node | 修改：派发前提增加“该 Node 已在本进程握手证明身份”（D1） |
| Controller 消费 Watch：流建立、信号、续约与无流节拍触发领取 | 继承，并增加一个触发：静态 Node 的第一次成功握手（D1） |
| 协议根 D4：会话绑定身份，断线不判失败；握手时 `node_id` 不符即拒绝会话 | 继承，不变；握手成功就是本文所需的证明 |
| clone 契约 D6：缺证据时不推断执行结果 | 继承，不变；已登记的派发不改派、不判失败（D2） |

## 决策

### D1：租户级 clone 只在静态 Node 握手证明身份后领取

- 运行时的静态 Node 会话在握手成功（`hello_accepted` 的 `node_id` 等于配置）时通知持久适配器。云端
  适配器据此把“静态 Node 已证明”置为真；这个标记在进程内只从假变真，不会撤销。
- 标记为假时，领取直接返回，不调用 `ClaimWork`，也不 `RecordDispatch`；Controller 每次进入等待只记录
  一次日志，说明租户工作留在 Cloud 队列等待握手。
- 标记第一次变真时立即触发一次领取：握手前被接受的工作没有信号会再来，不能等到下一次续约兜底。
- Node 之后断开不撤销证明。部署配置在进程内不可变，已被证明的 `NodeId` 在重连前后指向同一个 Node；
  此时登记的工作与现在“接受时 Node 断开”一样，等它回来即可。

为什么在领取前而不是在派发前拦截：派发的归属在 `RecordDispatch` 提交时就定下来了，Cloud 从此把这项工作
视为“已派发到该 `NodeId`”。在登记之后才发现身份不对已经晚了（见 D2）；不领取则什么都没有发生，工作仍是
Cloud 中尚未派发的队列项，任何一个 Node 身份正确的 Controller 都能领走。

为什么只要求“证明过一次”而不是“当前在线”：问题 5 是身份错误，不是连接可用性。要求在线会让短暂断线也
阻塞领取，而断线期间登记的派发本来就会在重连后由会话交付，没有需要防的错误。

### D2：已登记的派发不改派、不判失败

对已经以某个 `NodeId` 登记、尚无结果的派发，Controller 不把它改派到新的 `NodeId`，也不把它判为失败。
那个 Node 可能已经执行过，只是结果还没被接管；改派会产生两次 clone，判失败会与之后接管到的真实结果
冲突（clone 契约 D6）。

因此 D1 只防止新的错误登记。上线前因配错而登记的派发，以及合法地更换静态 Node 身份时旧 `NodeId` 上尚未
完成的派发，仍保持 `pending`，直到那个身份的 Node 回来。这是已知限制，处理它需要一个明确的“放弃派发”
终态，留给后续决策（见“本决策未解决的问题”）。

### D3：适用范围

- Workspace 沙盒路径不变：它已经满足“先证明、再登记”，`NodeId` 来自 Cloud 的 ensure 证据，Node 不就绪
  时 operation 以可见的原因延后，而不是静默挂起。
- SQLite 模式与 JSON 表面不变：它在接受请求的同一事务里登记派发，要改只能改接受语义；minicloud 即将
  废弃，不在它上面加新行为。
- Cloud 契约不变：Controller 只是晚一点调用已有的 `ClaimWork`。

## 不变量

1. 云端模式中，每个租户级派发登记的 `NodeId` 都已在登记它的 Controller 进程内被一次成功握手证明。
2. 静态 Node 未被证明时，Controller 不调用 `ClaimWork`，Cloud 中的队列项保持未派发。
3. 静态 Node 第一次被证明后，Controller 立即领取一次，不依赖信号或续约。
4. 已登记的派发不改派到其他 `NodeId`，也不因目标身份无法连接而被判失败。
5. Workspace 沙盒派发只在其 Node 就绪后登记（继承根决策 D3）。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 只在 ADR 中记录为已知限制，不改代码 | 配错 `node_id` 仍会把新工作永久挂起，而避免它只需在已有的领取入口加一个条件 |
| Controller 启动时列出发往未配置 Node 的未完成派发并告警 | 能看见问题但不能避免它；而且云端的 `ListPendingDispatches` 同时返回沙盒 Node 的派发，启动时 Controller 还不知道哪些沙盒存活，会误报 |
| 发现发往未配置 Node 的派发时拒绝启动 | 操作者改正配置后反而无法启动，只能改回错误配置，而错误配置永远无法完成这些派发 |
| 自动把旧 `NodeId` 的未完成派发改派到新 `NodeId` | 旧 Node 可能已经执行，改派会重复 clone；违反 D2 与 clone 契约 D6 |
| 要求静态 Node 当前在线才领取 | 把连接可用性和身份混为一谈；短暂断线会无谓阻塞领取（D1） |
| 同时改 SQLite 的接受路径 | 需要改变 JSON 表面“接受即登记”的语义，而 minicloud 即将废弃 |

## 本决策未解决的问题

- 已登记到错误或已退役 `NodeId` 的派发如何收尾：需要一个由操作者显式触发、与 Node 结果区分的
  “放弃派发”终态，涉及 Cloud 契约与公开 API，超出本文范围。
- SQLite 模式的同类问题：随 minicloud 退役一并消失，不单独处理。

## 落地顺序

1. **云端领取前置条件**（desktop 草稿 PR）：静态会话在握手成功时通知持久适配器；云端适配器在领取前
   检查标记，第一次证明时立即领取；以内存假 Cloud 与经 IPC 的假静态 Node 测试“身份不符时不领取，改正后
   立即领取”，并调整既有领取测试，让 Node 握手成为领取的前提。完成条件：`cargo test -p ora-controller`、
   clippy 与格式检查通过。
2. **评审通过后**：状态改为 `approved`，在 `test-cases/controller/node-management/` 补核心用例并登记证据；
   合并实现后改为 `implemented`。

保持不变的行为：Workspace 沙盒路径、SQLite 模式与 JSON 表面、Cloud 契约、Node 与协议。从第 1 步起改变
的行为：云端 Controller 在静态 Node 握手成功之前不领取租户级 clone。
