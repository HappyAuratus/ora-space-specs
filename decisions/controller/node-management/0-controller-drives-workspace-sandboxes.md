---
status: approved
---

# 云端 Controller 驱动 Workspace 沙盒：经 Substrate effects 建删沙盒，动态维护 Node 会话并代报 Cloud

云端持久模式（`persistence.kind = cloud`）的 Rust Controller 在现有 clone 闭环之外，领取并推进 Cloud 的
运行时 Workspace 操作：sandbox 阶段调用 Substrate effects 接口（本地为 cluster Sandbox Server，平台为
Substrate 适配器）创建沙盒，成功后按结果动态加入一个 `NodeTarget` 并建立 WebSocket 会话，握手后代 Node
向 Cloud 登记与上报心跳；clone 阶段向该 Node 派发 clone；terminate 阶段先移除会话再终止沙盒。静态配置的
Node 只继续服务租户级 clone，Workspace 的 Node 全部来自沙盒。

本 ADR 处于 `approved`，是 `controller/node-management` 的根决策；2026-09-26 Eric 确认评审要点全部按建议采纳（Node 由 Controller 代报、NodeId 与数据卷按 Workspace 固定、迁移保留旧表并失败在途旧步骤、租户级 clone 保留、不定义 SandboxProvider trait）；实现尚未开始，验证义务登记在[Controller 驱动 Workspace 沙盒核心用例](../../../test-cases/controller/node-management/workspace-sandbox-driving.md)，证据均为 `Missing`。
涉及 desktop `apps/ora-controller` 的云端适配器、会话管理与部署配置；Node、协议消息与 SQLite 模式不变。
它实现[运行时 Workspace 生命周期](../../cloud/operation/0-workspace-runtime-follows-desktop-node.md)中
Controller 的职责，使用[新增的 gRPC 契约](../../cloud/controller-integration/20260926-workspace-operations-and-controller-reported-nodes.md)，
并回答[传输决策](../../node/protocol/20260924-frame-channel-with-ipc-and-websocket.md) D8 留下的
`SandboxProvider` 问题。兼容性边界：Controller 配置文件格式（部署输入，按仓库约定可直接改形态）。
本文不涉及 minicloud（即将废弃）、任务与交互执行准入，也不改变租户级 clone 闭环。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 传输 D3：端点是 `NodeEndpoint` enum，传输层不认识厂商 | 继承，不变；动态端点仍是 `WebSocket(WsEndpoint)` |
| 传输 D8：Provider 输出 `NodeId` 与 `NodeEndpoint`，等 Docker 与平台两个实现再定 trait | 修改：Provider 边界就是 Cloud 的 Substrate effects HTTP 接口，两个实现都在接口另一侧，Controller 只需一个 HTTP 客户端，不定义 trait（D1） |
| 持久协调双适配器：云端适配器“派发到唯一配置的 Node” | 修改范围：该规则只适用于租户级 clone；Workspace 的 Node 由沙盒动态决定（D2） |
| Controller 消费 Watch：信号驱动领取、断流退回周期 | 继承，扩展到 `OperationAvailable`（D3） |
| 协议根 D4：会话绑定身份，断线不判失败 | 继承，不变；断线只影响 Node 上报的连接状态（D4） |
| 归属 D1、D2：Node 绑定单一 `ControllerId`，单一有效会话 | 继承，不变；沙盒创建时注入的 `controller_id` 必须是本 Controller 的 |

## 决策

### D1：Substrate effects 接口就是 Provider 边界，Controller 只有一个 HTTP 客户端

Controller 的云端配置增加 `substrate` 段：

| 字段 | 含义 |
|---|---|
| `effects_url` | Substrate effects 接口的基地址（本地为 Sandbox Server 的 `-lifecycle-addr`） |
| `router_url` | Node WebSocket 路由的基地址与路径，如 `ws://sandbox-server:18000/ora-node/v1` |
| `atespace` | 拼接 `ate-target-actor` 的 atespace |
| `request_timeout_ms` | 单次 effect 调用期限 |

Controller 调用 `PUT /effects/{effectId}` 执行 Cloud 计划的 effect，重试与对账前先 `GET`；effect ID、
请求体都来自 Cloud 的 plan，Controller 不改写。

- 为什么不定义 `SandboxProvider` trait：Docker 与 Substrate 两个真实实现已经存在，但它们的差异被
  Substrate effects 接口吸收了——Cloud 契约明确“生产适配器可以映射到其控制 API，但必须保持语义”。
  Controller 面对的是一个协议，而不是两个库；trait 只会有一个实现。若将来某个平台无法提供该接口，
  再在 Controller 中增加适配层。
- effect 的超时不等于失败：`request_timeout_ms` 到期后 Controller 以同一 ID `GET` 查询，按 Cloud 契约
  “不能以调用方超时判定不存在”处理；查询也失败时 `DeferOperation`，保留 plan。

### D2：云端模式的 Node 集合由 Workspace 沙盒动态决定

Controller 在内存中维护“当前 generation 的沙盒 → `NodeTarget`”映射，来源只有 Cloud 的快照与 effect
结果：

| 事件 | Controller 行为 |
|---|---|
| `sandbox_ensure` 成功（或领取到已有 sandbox 的 operation） | 以结果 `nodeId` 与 `WsEndpoint{ url: router_url, headers: { ate-target-actor: <atespace>/<externalId> } }` 加入 `NodeTarget`，启动会话 |
| 进程启动或重新取得租约 | 从 Cloud 快照重建全部未终止 sandbox 的 `NodeTarget`，不依赖本地状态 |
| `terminate` 阶段计划 `sandbox_terminate` 之前 | 停止该 Node 的会话并移除 `NodeTarget`，再调用 Substrate |

- 两种来源分工而不混用：静态 `nodes` 只接收租户级 clone（现有规则，至多一个），沙盒 Node 只接收其
  Workspace 的工作。沙盒的 NodeId 形如 `workspace-<uuid>`（cluster D4），与静态配置冲突时 Controller
  拒绝加入该 `NodeTarget` 并告警，而不是二选一：同一 `NodeId` 两个来源时无法判断哪一个是事实（同双
  适配器决策“不做回退”的理由）。SQLite 模式与 `--single-node` 不变。
- 先停会话再终止沙盒：会话仍在时终止，Controller 会把沙盒消失看成断线并按 `reconnect_ms` 重连一个
  注定不存在的目标，还会向 Cloud 报告 `disconnected` 干扰 quiesce 之后的状态。
- 会话的连接失败分类沿用传输 D7；`NotFound` 在这里有了明确含义：沙盒在 Cloud 看来未终止、Substrate
  却找不到，Controller 记录告警并继续重连，不自行重建——重建只能由 Cloud 的新 operation 决定。

### D3：Workspace 操作由 `ClaimOperation` 领取，每个 operation 串行推进

Controller 在持有租约时，与 clone 请求并行地领取 Workspace 操作：流建立时、`OperationAvailable` 到达时、
每次续期后与流不可用时按 `claim_interval_ms`。每个 operation 由一个任务按步骤推进：

| 步骤 | Controller 动作 | 推进条件 |
|---|---|---|
| sandbox | `PlanEffect` → Substrate `PUT` → `RecordEffectResult` → 加入 `NodeTarget` | effect 成功 |
| node | 等待会话握手 → `RegisterNode` → `ReportNodeStatus(connected, initialized)` | Cloud 接受推进（Node 已登记且心跳新鲜） |
| clone | `RecordDispatch` → 向 Node 派发 `CloneRepository` → 接管结果 | 成功则推进；失败 `DeferOperation`；未知保持 blocked |
| quiesce | 停止向该 Node 派发新工作 → `ReportNodeIdle` | Cloud 接受 idle 证据 |
| terminate | 停会话、移除 `NodeTarget` → `PlanEffect` → Substrate `PUT` → `RecordEffectResult` → `EndNode` | 终止被确认 |
| cleanup | `PlanEffect(workspace_data_delete)` → Substrate `PUT` → `RecordEffectResult` | 删除被确认 |

- 同一 operation 串行、不同 operation 并发：Cloud 保证每个 Project 至多一个进行中的 operation，并发只
  发生在不同 Project 之间。任务的所有权与取消跟随租约：失去租约立即停止全部 operation 任务与由它们
  启动的 Substrate 调用，但不停止 Node 会话（会话的去留由 D2 与新的持有者决定）。
- 旧 epoch 的 effect 在推进前先按 ID 向 Substrate `GET` 并 `RecordEffectResult`，满足 Cloud 的
  `reconcile_required` 规则。

### D4：代报 Node 状态以会话为准

- 握手成功后，Controller 用 `hello_accepted` 中的 `node_id` 与 `node_incarnation_id` 调用
  `RegisterNode`；`node_id` 与 `NodeTarget` 不符时握手本身已失败（传输 D3、不变量 5）。
- 会话存续期间每 10 秒 `ReportNodeStatus(connected)`；Node 发来的应用层心跳与 Controller 的上行心跳
  共同维持会话，Controller 只在会话确实存活时上报。会话结束时上报 `disconnected`；重连后若
  incarnation 变化，先 `EndNode` 旧行再 `RegisterNode` 新行。
- 上报失败（Cloud 不可达）不影响会话与执行：Cloud 会因心跳过期把 Workspace 视为不可用，这是正确的
  保守结果；Controller 在 Cloud 恢复后继续上报。

### D5：idle 依据派发账本，quiesce 期间不再派发

quiesce 阶段 Controller 先把该 Node 标记为“不接受新派发”，再检查该 Node 上是否还有尚无终态的派发
（含状态未知）：没有则上报 `idle=true`，否则 `idle=false`。标记在 operation 失败或 Workspace 重新开放
准入时撤销。Controller 是 Node 唯一的派发者，所以它的账本覆盖 Node 上全部可能的活动；先停派发再检查，
消除“检查通过后又派发了一个”的竞争。

## 不变量

1. 云端模式中每个 `NodeTarget` 都对应 Cloud 中当前 generation 的一个未终止 sandbox，端点只由
   Substrate 结果与部署配置推出。
2. 静态 `nodes` 只接收租户级 clone，沙盒 Node 只接收其 Workspace 的工作；两者的 NodeId 不重合；SQLite
   模式不调用 Substrate。
3. Controller 只执行 Cloud 计划的 effect，不改写 effect ID 或请求体；超时后先查询，不判定不存在。
4. 终止沙盒之前，该沙盒的 Node 会话已停止。
5. 只在会话存活时上报 `connected`；Node 登记使用握手中得到的身份。
6. 上报 idle 前，Controller 已停止向该 Node 派发。
7. 失去租约时停止全部 operation 推进；Controller 从不自行重建沙盒。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 定义 `SandboxProvider` trait，Docker 与 Substrate 各一个实现 | 两者都在 Substrate effects 接口背后，Controller 侧只会有一个实现（D1） |
| Controller 本地持久保存沙盒与端点 | 与 Cloud 形成两份事实；快照已包含全部未终止 sandbox，重启可重建（D2） |
| 云端模式取消静态 `nodes`，租户级 clone 也改派到沙盒 | 租户级 clone 没有 Workspace，也就没有沙盒可派；它是否退役由生命周期决策留待后续（D2） |
| 找不到沙盒时 Controller 自行重建 | 绕过 Cloud 的 generation 与终止确认，可能产生两个写同一数据卷的 Node（D2、cluster D5） |
| Node 自己向 Cloud 上报 | 需要沙盒内凭据与出站通道（生命周期决策 D4） |

## 本决策未解决的问题

- 任务与交互执行（`execution_tickets`）的派发：Controller 目前只派发 clone。
- 多 Controller worker 与 `NodeAssignment`。
- Substrate effects 接口的调用方认证（平台适配器）。

## 落地顺序

1. **契约锁定与客户端**：锁定包含新服务的 Cloud 契约 commit；实现 gRPC 客户端与 Substrate HTTP 客户端，
   配置增加 `substrate` 段。
2. **动态会话**：D2 的 `NodeTarget` 映射与会话启停，D4 的代报。以伪造 Substrate 与 Cloud 测试会话随
   sandbox 出现、消失与租约交接。
3. **操作推进**：D3 的步骤任务与 D5 的 idle；用 Cloud 的 PostgreSQL 集成环境与伪造 Substrate 验证
   create_workspace、stop、start、delete_workspace 全流程。
4. **联调**：与 cluster Sandbox Server（Docker）端到端，从 Cloud 前端创建 Workspace 完成 clone。

保持不变的行为：SQLite 模式、`--single-node`、租户级 clone 闭环（仍派发到静态 Node）、传输与协议。从
第 3 步开始，配置了 `substrate` 段的云端 Controller 按需创建沙盒。
