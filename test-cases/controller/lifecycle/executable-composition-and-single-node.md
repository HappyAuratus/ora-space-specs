# Controller 组合与托管 Node 生命周期验收

依据[已批准 ADR](../../../decisions/controller/lifecycle/0-executable-composition-and-single-node.md)。
证据基于 [desktop#595](https://github.com/ora-space/desktop/pull/595) 的
`e22bfdd063032d22532c2a007320072b374aaed9`；该 PR 尚未合并。已核对测试源码与绿色 CI，
本次未重跑 Linux 进程测试。下表区分已有直接证据与仍缺少的异常路径断言。

## Invalid Composition Leaves Deployment State Untouched

- **风险**：错误配置先占租约或改写 Node 部署。
- **前置状态**：既有 Controller／Node 配置、状态与用户文件。
- **触发**：相对配置路径、目录重叠、缺失 single_node、多 Node、目标不匹配、身份／endpoint 不一致。
- **必须成立**：全部输入校验在获取租约前拒绝；Node 配置只读；不部署 guardian 或启动 host。
- **禁止结果**：半初始化数据库、配置改写、借启动修复未知状态。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 校验无副作用 | `apps/ora-controller/tests/api.rs` 覆盖未知目标、缺失 single_node、相对 Node 路径、目录重叠及未知文件保护；`repository_single_node.rs` 覆盖归属不符且不创建状态；其他输入及配置只读断言待补 | Partial |

决策依据：D1、D2，不变量 1、4。

## Readiness Failure Never Opens The Api

- **风险**：接管未知 Node 或未就绪即受理。
- **前置状态**：活跃 endpoint，或托管 Node 无法在 ready_timeout_ms 内就绪。
- **触发**：启动 `--single-node`。
- **必须成立**：已有 endpoint 接受连接则拒绝；自有 Node 就绪后才启动 API 与重连；
  就绪超时发送终止信号并失败退出。
- **禁止结果**：接管未知 Node、超时仍受理请求、启动第二个 Node。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 活跃 endpoint 保护 | `apps/ora-node/tests/standalone/repository_single_node.rs::single_node_composition_hosts_and_retires_its_node` 覆盖活跃 endpoint 拒绝第二个托管入口且原 Node 存活 | Covered |
| 就绪与失败清理 | 待补受控延迟 endpoint，验证受理时机、超时终止、退出和租约释放 | Missing |

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
| 进程组可达性 | 待补 Controller 消失后向原进程组发送停止信号并验证 Node 收到 | Missing |

决策依据：D2，不变量 2。

## Shutdown And Node Failure Preserve Accepted State

- **风险**：关闭竞态、超时强杀跳过收尾或 Node 消失后继续接受。
- **前置状态**：有在途请求和 Git 的托管组合，可控制 Node 退出和收尾延迟。
- **触发**：正常停止、stop_timeout_ms 到期或 Node 意外退出。
- **必须成立**：先停 API 并有限等待在途请求，再停会话、SIGTERM 托管 Node 并等待，最后释放租约；
  超时告警退出不升级 SIGKILL；Node 意外退出使 Controller 停止受理并失败退出，不改写已接受记录。
- **禁止结果**：提前释放租约、强杀 Node、自动重启 Node、删除已接受记录或继续接受无法派发的请求。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 正常顺序与收尾 | `repository_single_node.rs::single_node_composition_hosts_and_retires_its_node` 验证正常退出前托管 Node 已退出；缺在途请求、活动 Git 收尾及各关闭阶段顺序断言 | Partial |
| 停止超时 | 待补延迟收尾验证告警、有限退出且无 SIGKILL | Missing |
| Node 意外退出 | 待补杀死托管 Node 后 Controller 失败退出、停止受理且原记录可恢复 | Missing |

决策依据：D2、D3，不变量 3、5。
