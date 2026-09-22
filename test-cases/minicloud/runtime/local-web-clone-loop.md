# minicloud 本机 Web 闭环验收

依据[已批准的根 ADR](../../../decisions/minicloud/runtime/0-local-web-client-embedded-controller.md)。
入口迁移依据[后续 ADR](../../../decisions/minicloud/runtime/20260921-controller-hosted-server.md)。证据已核对 [desktop#595](https://github.com/ora-space/desktop/pull/595) 的 `e22bfdd063032d22532c2a007320072b374aaed9` 测试源码及绿色 CI；PR 尚未合并，本次未重跑 Linux 进程测试。稳定用例标题保留，正文与证据以 ora-controller 新入口为准。
已有 Controller／Node 测试作为下层基础，不代替真实 HTTP、Vite proxy 和页面链路。
Eric 已明确不要求完整浏览器引擎验收；页面行为以 DOM 测试验证，网络与执行链路以真实 HTTP／Vite proxy 验证。
此范围调整不免除回复丢失、正常关闭、所有权交接及目录保护等非浏览器验证义务。

## Local Proxy Reaches The Embedded Controller

- **风险**：浏览器绕过 proxy、监听范围扩散，或 HTTP handler 另建执行器。
- **前置状态**：显式配置本机 ora-controller API、Vite proxy 与独立 Node。
- **触发**：浏览器经 Vite 地址提交仓库 URL 和分支。
- **必须成立**：前后端只监听回环；请求经 proxy 到 ora-controller，再通过本机 IPC 到 Node。
  不要求登录或令牌，Git 仍由 Node 的既有受管链路执行。
- **禁止结果**：监听所有接口、直连后端成为必要条件、API 层直接执行 Git，或新增认证／安全体系。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 部署与 proxy 链路 | `minicloud_vite_proxy_reaches_real_https_clone_and_restart` 启动真实本机 Vite／ora-controller，经 proxy 无凭据提交；CLI 测试验证回环默认值；启动器显式回环参数仍缺自动化断言 | Partial |
| 内嵌协调与真实执行 | Node standalone `repository_minicloud.rs` 两个测试覆盖真实 HTTP／Vite proxy → ora-controller → Node → host／guardian → HTTPS Git，查询实际 commit／目录并验证唯一 Run | Covered |

决策依据：D1、D2、D3，不变量 1、2。

## Durable Acceptance Survives Lost Http Responses

- **风险**：HTTP 先报告接受，或回复丢失导致第二次 clone。
- **前置状态**：真实 ora-controller API 与持久存储；可中断 HTTP 回复并注入接受写失败。
- **触发**：重复原请求、同请求身份改变输入、接受写失败，或持久接受后丢失回复再重传。
- **必须成立**：持久接受先于成功回复与派发；相同请求返回原执行，冲突拒绝且不覆盖原输入。
  接受只表示协调责任已保存，不表示 Node 已执行完成。
- **禁止结果**：内存接受、写失败后派发、每次 HTTP 重试生成新执行、将 Accepted 展示为成功。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接受与失败顺序 | `repository_minicloud.rs` 持有真实 SQLite writer lock，HTTP 接受失败返回 503、列表为空；释放锁后同请求接受并最终只有一个 Run | Covered |
| 回复丢失与幂等 | `repository_minicloud.rs` 真实 TCP 代理收到上游 202 后只交付一个 body 字节，客户端解码失败；强杀重启后重传原输入，返回原执行且最终唯一 Run；`App.test.tsx` 覆盖回复丢失后刷新并重传原输入 | Covered |
| 输入冲突 | `apps/ora-controller/tests/api.rs` 同身份改分支返回 409，列表与重启查询保留原记录 | Covered |

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
| 展示与生命周期 | `App.test.tsx` 覆盖 pending、成功 commit／路径、失败残留、查询失败保留成功事实和卸载 abort；`automatically resumes polling after an outage without submitting again` 验证恢复后自动显示原结果、清除告警且不提交命令 | Covered |
| 刷新恢复 | `App.test.tsx` 卸载／重挂载恢复 session storage 中的原身份；真实 Vite proxy 测试验证原记录读取；完整浏览器引擎点击和刷新不在本次验收要求内 | Covered |

决策依据：D4，不变量 2、3、4。

## Server Restart Retains Original Execution Responsibility

- **风险**：浏览器断连取消执行，或 Controller 重启丢失协调事实／提前确认结果。
- **前置状态**：真实 ora-controller、独立 Node 和 Git；操作已持久接受。
- **触发**：中断 HTTP、关闭页面、正常停止或强杀 Controller，再用同一数据目录启动。
- **必须成立**：Node 已接受执行不因调用方退出而取消；Controller 重开原 Controller 身份及记录，
  恢复查询和结果接管，只在持久接管后确认原事件。
- **禁止结果**：重启新建执行、丢失已接受记录、结果未提交即 Ack、HTTP 层补造完成事实。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 断连与正常关闭 | `repository_minicloud.rs` 暂停 HTTPS，确认 Git 创建 .git 并以 pidfd 固定存活身份；SIGTERM 非托管 Controller 后验证正常退出且原 Git 仍存活，重启后得到原结果及唯一 Run；HTTP 测试另覆盖待协调记录恢复 | Covered |
| 强杀与接管恢复 | `repository_minicloud.rs` 接受后与完成后分别强杀独立 Controller，重传／查询原执行，HTTPS 拒绝访问时仍恢复原结果，Node 唯一 Run | Covered |

决策依据：D2、D4，不变量 3、4。

## Embedded Ownership Preserves Existing State

- **风险**：内嵌运行时与独立 Controller 双写，或新入口覆盖旧目录／切换 Backend 写入路径。
- **前置状态**：显式注入各状态目录，已有 Controller 数据、独立运行所有者及用户文件。
- **触发**：启动 minicloud、配置重叠目录、同时运行独立 Controller，或使用未知已有状态文件。
- **必须成立**：单运行所有者、目录隔离、未知文件拒绝初始化；有效历史原样恢复。
  API 层不另建任务数据库，也不迁移 Desktop／Backend 记录或用户 checkout。
- **禁止结果**：清库、覆盖用户文件、双运行时并发写同库、重复协调实现或隐式 Backend 切换。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 组合入口保护 | Controller `http_acceptance_is_idempotent_and_survives_service_restart` 通过 Service::start 验证重复所有者、非法目标／托管配置、重叠目录拒绝且不创建目录、未知状态文件拒绝且逐字节保留 | Covered |
| 既有记录复用 | 同一 HTTP 测试用独立 Controller 公共接口接受原请求，验证持有所有权时 Controller API 组合拒绝启动；释放独立所有者后 minicloud 打开原目录，通过 HTTP 重传／查询及正常重启保留原 operation／execution 和输入 | Covered |
| 开发权限与发布权限分离 | `ora-utils` 的 `private_path`／`trusted_path` 测试分别在 debug、release 运行，前者允许原有宽松权限且不 chmod，后者拒绝；两者仍拒绝符号链接及不符所有者 | Covered |

决策依据：D1、D2、D3，不变量 2、5。

## Launcher Shutdown Preserves Hosted Node Cleanup

- **风险**：启动器保留第二套 Node 编排，或在 Node 收尾前停止 host／guardian。
- **前置状态**：启动器运行 host、ora-controller --single-node 和 Vite，存在活动 Git。
- **触发**：正常停止启动器或仅强杀 Controller。
- **必须成立**：正常顺序为 Vite → Controller（含 Node 收尾）→ host → guardian；崩溃后可收回残留 Node；既有状态与 checkout 不被清理或迁移。
- **禁止结果**：重复启动 Node、提前停止 host、残留不可控进程、清库或改写用户文件。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 启动器完整关闭与崩溃回收 | PR #595 报告真实启动器手工验收；`apps/ora-node/tests/standalone/repository_single_node.rs` 直接证明 Controller 正常收回托管 Node，但未覆盖完整启动器顺序和活动 Git 收尾，仍待补自动化证据 | Partial |

决策依据：入口迁移 ADR D1、不变量 3、4；生命周期 ADR D3、D4。
