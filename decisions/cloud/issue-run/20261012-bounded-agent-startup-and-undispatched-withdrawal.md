---
status: proposed
---

# 启动意图未登记时允许撤回；已登记的未知执行只请求结束并保留对账责任

Cloud 区分首条提问与启动意图、派发登记、Node 受理和真实记录。未登记的启动意图遇到结束、明确能力拒绝或超时，可原子撤回并声明清理；已登记且尚无启动记录的执行，超时只进入带安全原因的结束等待，保留成果与资源责任。

本文为 `proposed`，尚待 owner 审阅。继承 [IssueRun 根决策](0-agent-run-in-disposable-isolated-workspace.md) 和前序 [续接提案](20261010-resume-run-from-latest-issue-revision.md)，补充根 D3/D6 的启动缺口；续接提案的 Revision 选择与恢复规则不改。Thread 继承 [根决策](../thread/0-durable-agent-thread-with-user-turns.md) 与 [个人模型输入权限](../thread/20261010-personal-model-initiator-controls-input.md)。控制面仍遵循 [派发前登记与同事务接缝](../controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 一个 Run 一个运行 Workspace、派发前登记、冻结输入、先接管再 ACK | 继承 |
| `pending` 与首条 prompt 的声明 | 明确它不证明 Node 受理，清理边界以派发登记为准 |
| 普通 end 为 `202/ending` 的请求 | 继承，终结由后续周期对账完成 |
| Thread `ended` 来自会话终态 | 增加未登记意图的 Cloud 撤回终态；不伪造 Node 结果、历史或收据 |
| 已登记执行的交付与 Workspace 释放 | 继承，超时不跳过这些责任 |

## 决策

### D1：未登记是受事务保护的执行不存在证据

撤回必须与派发登记使用同一事务锁，确认没有 Agent 执行登记、已绑定 work 或已投递命令。控制面撤回未派发 work/命令；业务层结算 Thread、丢弃未执行的 queued 轮次并声明 owned delete。任何一步失败全部回滚。迟到派发不能复活已撤回输入。

普通结束或取消在此边界结算为 `cancelled`，`deliveryState=skipped`，不声明 Agent 成功。明确模型代理能力拒绝使用 `model_proxy_unavailable`，未派发启动超时使用 `agent_start_failed`，两者结算为 `failed`。主 Workspace、历史 Revision 和其他运行不受影响；Project 忙时沿用已有清理重试。

### D2：预算由数据库时间判断，输入不续期

启动等待预算拟定默认 30 分钟，部署可配置。声明前以运行 Workspace 创建完成时间为起点，声明后以 work 时间为起点，登记后以执行登记时间为起点。用户追加轮次、重放和控制连接恢复不延长预算。缺模型代理的已初始化当前 Node 是明确拒绝证据，可在预算前处理；未知或暂时断开的实例不能被当作明确拒绝。

### D3：已登记的未知执行进入可恢复的结束等待

超时后 Thread 为 `ending`，公开读取给出安全 `failureCode=agent_start_failed`，停止接受新轮次。已有用户结束原因保留；没有结束请求时声明一个策略取消。工作、登记、未结算轮次、Workspace 和冻结输入保留，Controller 继续按稳定身份对账与投递结束。

策略超时取消最终经 Node 终态和交付结算后，业务结果为 `failed`；此前已接受的用户结束/取消保持原结果。真实终态之前不能伪造 Node 历史、收据或 Revision，也不能删除 Workspace。迟到的真实记录仍按既有接管顺序保存，终态只结算一次。

## 不变量

1. 登记与撤回只能有一方获得未派发输入的后续责任。
2. 请求超时、缺回复、Controller 断开均不证明已登记执行未发生。
3. 原始模型 Key、临时授权与供应商正文不进入失败原因或证据。
4. 未登记撤回的清理声明与业务终态原子提交；已登记结果继续沿用交付/清理链路。
5. 用户、租户、冻结配置和既有结束原因不被恢复流程扩大或替换。

## 替代方案与代价

| 方案 | 取舍 |
|---|---|
| 所有 pending 继续等待 Node | 简单，但明确无法派发的输入与结束请求可永久悬挂 |
| 把所有 pending 当作未执行并删除 | 收尾快，但会丢失已登记、响应未知的实际执行与成果 |
| 结束前仍启动完整 Agent | 可复用原终态链路，但可能触发用户已请求停止的模型/工具操作，能力拒绝仍无解 |
| 未登记撤回，已登记保留对账 | 增加状态证据和预算策略；保留已接受外部效应的责任，作为本文方案 |

## 未决与落地

owner 需确认普通结束未登记时的 `cancelled/skipped`、30 分钟默认预算及未知执行的安全原因。更强的管理强停/隔离证明、全局 ClaimWork 公平性、公开取消 API 和新结算 SSE 不在本文范围。

批准后再维护对应核心用例；直接验证须覆盖未派发结束、能力拒绝、超时、登记与结束竞态、迟到登记/终态、重复、回滚、Project 忙、正常多轮、冻结输入和租户隔离。随后核验真实 Controller/Node 链路，并按真实交付证据决定实施状态。现有迁移与历史执行结果不重写。
