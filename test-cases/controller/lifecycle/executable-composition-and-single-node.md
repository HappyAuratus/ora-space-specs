# Controller 组合与托管 Node 生命周期验收

依据[已批准 ADR](../../../decisions/controller/lifecycle/0-executable-composition-and-single-node.md)，
其中涉及 API 的约定已由[过渡 JSON 接口退役决策](../../../decisions/controller/api-boundary/20260928-retire-transitional-json-surface-and-listener.md)
修改：可执行入口不再监听，就绪后报告组合完成，关停不再有 API 受理阶段。证据核对 [desktop#627](https://github.com/ora-space/desktop/pull/627)（已合并），
Linux 进程测试已在该 PR 的提交上本地重跑。
`apps/ora-controller/tests/composition.rs` 以子进程 Node 替身（正常监听／永不就绪／忽略 SIGTERM）
在可执行入口层复现真实 Node 难以稳定制造的异常路径；托管真实 Node 的证据仍来自
`apps/ora-node/tests/standalone/repository_single_node.rs`。

## Invalid Composition Leaves Deployment State Untouched

- **风险**：错误配置先占租约或改写 Node 部署。
- **前置状态**：既有 Controller／Node 配置、状态与用户文件。
- **触发**：相对配置路径、旧监听参数、目录重叠、缺失 single_node、多 Node、身份／endpoint 不一致。
- **必须成立**：全部输入校验在获取租约前拒绝；Node 配置只读；不部署 guardian 或启动 host。
- **禁止结果**：半初始化数据库、配置改写、借启动修复未知状态。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 校验无副作用 | `apps/ora-controller/tests/service.rs::local_composition_refuses_before_state_and_holds_an_exclusive_owner` 覆盖缺失 single_node、多 Node 托管、相对 Node 路径、目录重叠及未知文件保护；`composition.rs::refused_composition_leaves_no_state_behind` 以真实入口覆盖相对配置路径、旧监听参数、Node 配置归属不符时拒绝且不创建状态、不启动 Node、Node 配置逐字节不变；`repository_single_node.rs` 断言真实 Node 配置在托管全程只读 | Covered |

决策依据：D1、D2，不变量 1、4。

## Readiness Failure Never Completes The Composition

- **风险**：接管未知 Node 或未就绪即开始派发。
- **前置状态**：活跃 endpoint，或托管 Node 无法在 ready_timeout_ms 内就绪。
- **触发**：启动 `--single-node`。
- **必须成立**：已有 endpoint 接受连接则拒绝；自有 Node 就绪后才启动重连并报告组合完成；
  就绪超时发送终止信号并失败退出。
- **禁止结果**：接管未知 Node、超时仍报告组合完成、启动第二个 Node。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 活跃 endpoint 保护 | `apps/ora-node/tests/standalone/repository_single_node.rs::single_node_composition_hosts_and_retires_its_node` 覆盖活跃 endpoint 拒绝第二个托管入口且原 Node 存活 | Covered |
| 就绪与失败清理 | `composition.rs::readiness_timeout_stops_node_and_never_composes` 以永不就绪的替身验证超时后终止 Node、从不记录组合完成、失败退出且同一身份可重新打开状态（租约已释放）；`composition.rs::normal_stop_orders_phases_and_retires_node` 断言 `managed Node ready` 日志先于 `ora-controller coordinating locally` | Covered |

决策依据：D2。

## Controller Crash Preserves Node Execution

- **风险**：父进程崩溃取消执行，或 Node 脱离进程组无法停止。
- **前置状态**：同进程组 Controller 与托管 Node，已有接受的 Git 执行。
- **触发**：仅强杀 Controller，再恢复协调；另验证进程组停止。
- **必须成立**：Node 与 Git 继续运行，恢复后保留原执行及唯一 Run；进程组停止能到达剩余 Node。
  托管入口遇活跃 endpoint 仍拒绝；先让原 Node 完成并正常退出后，可启动替换组合接管原结果。
- **禁止结果**：自动接管未知 Node、父进程死亡连带取消执行、Node 新建会话脱离进程组。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 崩溃与恢复 | `apps/ora-node/tests/standalone/repository_single_node.rs::single_node_composition_hosts_and_retires_its_node` 固定 Node 身份，强杀 Controller 后原 Node 完成 clone，替换组合接管原执行且唯一 Run | Covered |
| 进程组可达性 | 同一用例断言托管 Node 的 pgid 等于 Controller pid，并在 Controller 被强杀后向该进程组发送 SIGTERM，验证 Node 收到并退出 | Covered |

决策依据：D2，不变量 2。

## Shutdown And Node Failure Preserve Accepted State

- **风险**：关闭竞态、超时强杀跳过收尾或 Node 消失后继续运行。
- **前置状态**：有已接受工作和活动 Git 的托管组合，可控制 Node 退出和收尾延迟。
- **触发**：正常停止、stop_timeout_ms 到期或 Node 意外退出。
- **必须成立**：先停会话，再 SIGTERM 托管 Node 并等待，最后释放租约；超时告警退出不升级 SIGKILL；
  Node 意外退出使 Controller 失败退出，不改写已接受记录。
- **禁止结果**：提前释放租约、强杀 Node、自动重启 Node、删除已接受记录或在 Node 消失后继续运行。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 正常顺序与收尾 | `composition.rs::normal_stop_orders_phases_and_retires_node` 按日志断言 Node 会话 → 托管 Node 的完成顺序且 Node 已退出；`repository_single_node.rs` 在 HTTPS 暂停、Git 活跃时正常停止，验证 Node 完成受管 Git 收尾且 Git 已退出后 Controller 才成功退出，重启后两条已接受记录保持原身份、已完成记录状态不变 | Covered |
| 停止超时 | `composition.rs::stop_timeout_warns_without_sigkill` 以忽略 SIGTERM 的替身验证 Controller 在期限内告警并成功退出，替身仍存活（未 SIGKILL） | Covered |
| Node 意外退出 | `composition.rs::node_exit_stops_composition_and_keeps_records` 经 `SqliteStore` 接受一条 clone 后启动组合并强杀替身，验证 Controller 失败退出并报告原因，重新打开状态后原命令逐字保留且无终态 | Covered |

决策依据：D2、D3，不变量 3、5。
