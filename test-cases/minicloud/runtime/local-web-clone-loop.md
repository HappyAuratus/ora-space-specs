# minicloud 本机 Web 闭环验收

依据[已批准的根 ADR](../../../decisions/minicloud/runtime/0-local-web-client-embedded-controller.md)。
本页登记 minicloud 新入口的直接验证义务。应用尚未实现，证据均为 Missing；
已有 Controller／Node 测试作为下层基础，不代替真实 HTTP、Vite proxy 和页面链路。

## Local Proxy Reaches The Embedded Controller

- **风险**：浏览器绕过 proxy、监听范围扩散，或 HTTP handler 另建执行器。
- **前置状态**：显式配置本机 server、Vite proxy、Controller 与独立 Node。
- **触发**：浏览器经 Vite 地址提交仓库 URL 和分支。
- **必须成立**：前后端只监听回环；请求经 proxy 到内嵌 Controller，再通过本机 IPC 到 Node。
  不要求登录或令牌，Git 仍由 Node 的既有受管链路执行。
- **禁止结果**：监听所有接口、直连后端成为必要条件、server 直接执行 Git，或新增认证／安全体系。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 部署与 proxy 链路 | 启动真实 Vite／server，检查绑定地址并通过 proxy 完成请求，不附带应用凭据 | Missing |
| 内嵌协调与真实执行 | 真实 HTTP → Controller → Node → host／guardian → HTTPS Git clone，结果可查询 | Missing |

决策依据：D1、D2、D3，不变量 1、2。

## Durable Acceptance Survives Lost Http Responses

- **风险**：HTTP 先报告接受，或回复丢失导致第二次 clone。
- **前置状态**：真实 server 与 Controller 存储；可中断 HTTP 回复并注入接受写失败。
- **触发**：重复原请求、同请求身份改变输入、接受写失败，或持久接受后丢失回复再重传。
- **必须成立**：持久接受先于成功回复与派发；相同请求返回原执行，冲突拒绝且不覆盖原输入。
  接受只表示协调责任已保存，不表示 Node 已执行完成。
- **禁止结果**：内存接受、写失败后派发、每次 HTTP 重试生成新执行、将 Accepted 展示为成功。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接受与失败顺序 | 通过 HTTP 注入真实 Controller 接受写失败，验证无成功接受、无 Node 派发 | Missing |
| 回复丢失与幂等 | 截断接受回复后重传原身份，查询原操作并验证没有新增 clone 执行 | Missing |
| 输入冲突 | 同身份不同仓库／分支返回冲突，原操作及结果保持不变 | Missing |

决策依据：D2、D4，不变量 2、3。

## Polling Distinguishes Results From Unavailability

- **风险**：前端把无结果、断网或 Unknown 当作失败，或自行生成执行事实。
- **前置状态**：已接受、尚无终态、成功、失败的操作，以及暂时不可用的 HTTP 或 Node 连接。
- **触发**：打开列表、轮询结果、刷新页面，随后恢复连接。
- **必须成立**：列表来自 Controller；未完成／未知／查询失败不伪装成业务失败。
  成功展示原 Node 路径和实际 commit；失败展示原原因及残留事实。
  刷新恢复原记录，不触发新命令；卸载页面时释放轮询和请求生命周期。
- **禁止结果**：浏览器持有唯一任务事实、自动新执行重试、伪造进度／成功或后台泄漏轮询。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 展示与生命周期 | 通过生产前端接口及类型化操作 handler 验证状态、查询失败、恢复和卸载行为 | Missing |
| 刷新恢复 | 浏览器经真实 proxy 提交并刷新，重新读取同一接受记录及结果 | Missing |

决策依据：D4，不变量 2、3、4。

## Server Restart Retains Original Execution Responsibility

- **风险**：浏览器断连取消执行，或 server 重启丢失协调事实／提前确认结果。
- **前置状态**：真实 server 内嵌 Controller、独立 Node 和 Git；操作已持久接受。
- **触发**：中断 HTTP、关闭页面、正常停止或强杀 server，再用同一数据目录启动。
- **必须成立**：Node 已接受执行不因调用方退出而取消；server 重开原 Controller 身份及记录，
  恢复查询和结果接管，只在持久接管后确认原事件。
- **禁止结果**：重启新建执行、丢失已接受记录、结果未提交即 Ack、HTTP 层补造完成事实。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 断连与正常关闭 | 实际 HTTP 断连／server 停止后，原 Node clone 继续完成并可在重启后查询 | Missing |
| 强杀与接管恢复 | 强杀 minicloud server 后从原存储恢复，验证原身份、结果与唯一 clone Run | Missing |

决策依据：D2、D4，不变量 3、4。

## Embedded Ownership Preserves Existing State

- **风险**：内嵌运行时与独立 Controller 双写，或新入口覆盖旧目录／切换 Backend 写入路径。
- **前置状态**：显式注入各状态目录，已有 Controller 数据、独立运行所有者及用户文件。
- **触发**：启动 minicloud、配置重叠目录、同时运行独立 Controller，或使用未知已有状态文件。
- **必须成立**：单运行所有者、目录隔离、未知文件拒绝初始化；有效历史原样恢复。
  server 不另建任务数据库，也不迁移 Desktop／Backend 记录或用户 checkout。
- **禁止结果**：清库、覆盖用户文件、双运行时并发写同库、重复协调实现或隐式 Backend 切换。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 组合入口保护 | 经 minicloud 启动入口验证重复所有者、路径重叠及未知文件拒绝，文件保持不变 | Missing |
| 既有记录复用 | 由独立 Controller 接受的记录可在其停止后通过 minicloud 查询和继续协调 | Missing |

决策依据：D1、D2、D3，不变量 2、5。
