---
status: implemented
---

# 取得租约后按 Cloud 的存活沙盒列表重建 Node 会话

云端 Controller 首次以某个租约 epoch 推进 operation 之前，调用 Cloud 的 `ListLiveSandboxes`，为列出的每个
sandbox 启动会话，并停止自己持有但未被列出的会话。重启后的 Controller 因此不必等到某个 Workspace 再被领取
operation 就能重连并重新上报 Node。

本 ADR 处于 `implemented`，由 ora-space/desktop#621 落地（`apps/ora-controller/src/cloud/operations/live.rs`），
使用 [ListLiveSandboxes 契约](../../cloud/controller-integration/20260926-list-live-sandboxes.md)（cloud#31），
验证证据见[Controller 驱动 Workspace 沙盒核心用例](../../../test-cases/controller/node-management/workspace-sandbox-driving.md)。
兼容性边界：无（不改配置、持久布局或协议）。

## 继承与修改

| 前序约定（[根决策](0-controller-drives-workspace-sandboxes.md)） | 本文处理 |
|---|---|
| D2：`NodeTarget` 只来自 Cloud 快照与 effect 结果 | 继承；列表也是 Cloud 记录，来源不变 |
| D2：重建只随 operation 发生，重启后要等其 Workspace 再有 operation | 修改：每个新 epoch 先按列表重建（D1），之后仍由每个快照校正 |
| D3：失去租约停止 operation 推进，不停止会话 | 继承，不变 |
| 不变量 7：Controller 从不自行重建沙盒 | 继承，不变；只重建会话，不调用 Substrate |

## 决策

### D1：每个 epoch 先重建一次，失败不阻塞推进

operation 任务在某个 epoch 下第一次启动时调用列表；成功后记住该 epoch，不再重复。失败（Cloud 不可达、陈旧、
旧版 Cloud 不支持）只记录告警，照常领取 operation，下一次唤醒再试。

- 为什么放在 operation 任务里：它与领取属于同一个租约所有者，且在同一任务中先于领取执行，重建与 operation
  轮次之间不会并发修改会话集合。
- 为什么失败不阻塞：每个被领取的 operation 仍会按快照校正自己 Project 的沙盒，列表只是补齐“没有 operation
  的 Workspace”，阻塞推进只会扩大影响。

### D2：列表之外的会话被停止

未被列出的会话属于正在终止、已终止或已被新 generation 取代的 sandbox（例如租约在其他进程手中时发生的变化），
按根决策不变量 1 必须移除；移除走与 terminate 相同的路径，不上报 `disconnected`。

## 不变量

1. 重建只调用 Cloud 的只读列表，不调用 Substrate。
2. 重建后 `NodeTarget` 集合等于 Cloud 列出的 sandbox 集合（静态 Node 冲突的除外）。
3. 列表失败不阻止 operation 推进。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 每次唤醒都重新列出 | 唤醒频繁（每个领取节拍），而列表只在租约易手时才可能变化；快照已持续校正 |
| 列表失败时暂停 operation | 让一个补齐性读取成为推进的前置条件，Cloud 短暂不可达就阻塞全部操作（D1） |
| 只补齐不删除 | 会保留已被终止或取代的 sandbox 的会话，违反根决策不变量 1（D2） |

## 落地

desktop#621：`live.rs` 重建逻辑、Cloud 契约 submodule 指向 cloud#31、测试 Cloud 实现新 RPC、测试
`a_restarted_controller_reconnects_live_sandboxes_without_an_operation`。保持不变：SQLite 模式、静态 Node、
operation 推进顺序。
