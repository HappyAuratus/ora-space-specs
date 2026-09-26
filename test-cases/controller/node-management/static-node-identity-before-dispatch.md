# 静态 Node 先证明身份再领取租户级 clone

依据[静态 Node 身份决策](../../../decisions/controller/node-management/20260926-static-node-identity-proven-before-dispatch.md)。
本文件登记云端 Controller 只在静态 Node 握手证明身份后才领取租户级 clone 的承诺。证据来自 `ora-space/desktop`
的 `apps/ora-controller`（desktop#620）：`tests/cloud.rs` 以内存假 Cloud（真实契约的测试桩）与经 IPC 的假静态
Node 驱动运行时，假 Node 的身份可在运行中切换。

## Tenant Work Is Registered Only After The Static Node Proves Its Identity

### 风险

配置的 `node_id` 与端点背后的 Node 不符时，Controller 仍领取工作并以错误的 `NodeId` 登记派发；派发一经登记
就不能改派，工作因此永久挂起（联调问题 5）。

### 前置状态

云端 Controller 持有租约，配置了一个静态 Node；Cloud 队列中有已接受、尚未派发的租户级 clone。

### 触发

静态 Node 握手时声称的身份与配置不符并被拒绝；之后端点背后的 Node 以配置的身份握手成功。

### 必须成立

- 静态 Node 未在本进程内握手成功之前，Controller 不调用 `ClaimWork`，也不调用 `RecordDispatch`。
- 身份不符的握手被拒绝后 Controller 继续重连，工作留在 Cloud 队列中。
- 第一次成功握手立即触发一次领取，不依赖信号、续约或无流节拍。
- 证明不因之后断线而撤销。

### 禁止结果

以未经握手证明的 `NodeId` 登记派发；证明之前领取工作；证明之后要等下一次续约才领取。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未证明时不领取、不登记 | 假 Node 声称其他身份，经多次被拒握手与多个领取节拍后，断言完整调用序列只有获取租约与打开流 | Covered：`tests/cloud.rs` `tenant_work_waits_until_the_configured_node_proves_its_identity`（去掉领取前置条件后失败） |
| 证明后派发到正确的 Node | 同一测试中假 Node 改为配置的身份后，排队的工作被登记 | Covered：同上 |
| 第一次证明立即领取 | `claim_interval_ms` 为 1 小时且不发信号时，握手后排队工作按序登记，整段调用序列相等 | Covered：`tests/cloud.rs` `work_queued_before_the_node_handshake_is_claimed_once_it_completes` |
| 断线不撤销证明 | 证明后断开 Node，再入队工作，断言仍被领取 | Missing |

### 决策依据

静态 Node 身份决策 D1，不变量 1–3。

## Registered Dispatches Are Never Reassigned

### 风险

为了收尾发往错误或已退役 `NodeId` 的派发，Controller 把它改派到另一个 Node 或直接判失败，导致重复 clone，
或与之后接管到的真实结果冲突。

### 前置状态

Cloud 中存在以某个 `NodeId` 登记、尚无结果的派发，而当前配置的静态 Node 是另一个身份。

### 触发

Controller 以新的静态 Node 配置启动并完成握手。

### 必须成立

- 旧 `NodeId` 的派发保持原样，既不改派也不判失败。
- 新的静态 Node 只接收之后领取并登记的工作。

### 禁止结果

以新 `NodeId` 重新登记旧派发；因目标身份不可连接而写入失败结果。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 不改派、不判失败 | 假 Cloud 预置发往旧 `NodeId` 的未完成派发，以新配置运行后断言该记录不变且没有针对它的写调用 | Missing：只有代码路径保证（会话只查询自己 `NodeId` 的未完成派发） |

### 决策依据

静态 Node 身份决策 D2，不变量 4；clone 契约 D6。
