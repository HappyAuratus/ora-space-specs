# Node 核心测试用例

本目录与 `specs/decisions/node/` 镜像。Node 各能力的根决策进入 `approved` 后，再在对应子目录
登记需要长期维护的跨进程、外部副作用、ownership、重试和恢复用例。

## Node 直接执行 Git

[Node 直接执行 Git 决策](../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)为
`implemented`（Linux）；核心用例的证据按最终实现核对，缺口逐项写在用例里。它取代了 host／guardian 体系，
[process](process/README.md) 下只保留 containment 的用例，它们原有的证据随被取代的代码一起删除。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| git-execution | [尝试锁、停止证据与重启收口](git-execution/attempt-locks-and-stop-evidence.md) | 用超时或"查不到"推断已停止、锁泄漏给无关进程、后代仍在写入时上报结果、按过期线索误杀、Node 死后 Git 无人终止、重启后未收口就恢复、升级后旧尝试卡住、clone 留下常驻进程 |

## Git 原生代码链路（内部线）

[Node 从快照 clone 决策](../../decisions/node/repository/20261016-clone-from-git-server-snapshot-and-ora-git.md)为
`implemented`，repository 一行的用例按现有测试登记证据并保留缺口。
[Node 交付推送与 Git 恢复决策](../../decisions/node/revision/20261017-push-delivery-ref-and-restore-from-git-server.md)为
`implemented`，revision 一行的用例按现有测试登记证据并保留缺口。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| repository | [快照 clone 与 `ora-git`](repository/snapshot-clone-and-ora-git.md) | 访问外部仓库、checkout 留下 remote、带证书的 Git 读到 checkout 配置、拉取次数无界、冒充会话、推到别人的分支 |
| revision | [Git 交付与恢复](revision/git-delivery-push-and-restore.md) | 交付 ref 未写入就成功、`403` 被重试、快进失败拖垮交付、恢复访问外部、`401` 让前序永久不可续、缺证书仍启动 |

## 已批准的恢复截止、取消与停止确认

[Node 恢复截止与停止确认决策](../../decisions/node/revision/20261018-complete-restore-deadline-and-cancel-confirmation.md)经 owner 批准为
`approved`。恢复已改为从 Git Server 拉取前序交付 ref，bundle 恢复不再存在；完整截止、取消与停止确认的实现尚未合入 desktop 的 main，
这些义务目前都是 Missing。内部线恢复的证据在[Git 交付与恢复](revision/git-delivery-push-and-restore.md)，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| revision | [恢复截止、取消与停止确认](revision/restore-deadline-and-stop-confirmation.md) | 截止被重置或命令越过截止、执行预算/停止请求/停止确认混同、未知停止证据下交付或复用、上限被重启重置、取消中途打断 Git、启动清理越过原执行交接、BestEffort 当作 Strong |

## 插件模型 adapter

[插件模型 adapter 决策](../../decisions/node/agent-runtime/20261017-plugin-owned-model-adapters.md)于 2026-10-17 经 owner
批准，已实现，状态为 `implemented`。义务多为 Partial：Node 侧测试用回显 fixture 代替真实 CLI，缺口逐项写在
用例里；[既有模型代理用例](agent-runtime/model-proxy.md)的证据保持原范围。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [插件模型 adapter](agent-runtime/plugin-model-adapters.md) | 首轮前准备失败残留授权、上下文泄入账本/日志、CLI 高优先级配置改路由、共享 provider 状态、别名与发现越权、收尾与撤销不收敛 |

## 子代理投影与运行诊断

[子代理投影来源决策](../../decisions/node/agent-runtime/20261020-child-projection-history-and-diagnostics.md)为
`implemented`，核心用例按现有测试登记证据并保留缺口；
[会话执行用例](agent-runtime/session-execution.md)与[静默策略用例](../desktop/core/workflow/prompt-inactivity.md)的
证据保持原范围，不推导来源保真或终态缺口已实现。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [子代理投影、终态缺口与运行诊断](agent-runtime/child-projection-and-diagnostics.md) | 父子内容互串或按 depth 猜归属、同 messageId 跨来源拼接、父回合结束把未知结局伪装成终态、忽略已返回的委托或把委托返回当子组终态、子活动续期父提示、把重放时间当实时活动、只读详情崩溃或长出控制入口 |

## 显式 Session MCP

[显式 Session MCP 决策](../../decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md)于
2026-11-15 经 owner 批准为 `approved`。实现尚未合入 desktop 的 main，这些义务目前都是 `Missing`；
[会话执行用例](agent-runtime/session-execution.md)的证据保持原范围，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [显式 Session MCP](agent-runtime/session-mcp.md) | 空选择被当成全量或自动发现、乱序重复非规范选择被受理、部分快照照常交付、Node 本地路径或 Controller 命令进入 setup、Setting 值进入账本/JSONL/Thread/日志/错误/Debug、重发改写运行中的值、能力缺失仍发 session/new、失败终态或日志携带值、租约提前释放、Separate 工作负载拿到不可执行命令、共享解析器使 Desktop 行为漂移 |

## 同一 checkout 的多个 Agent 会话

[共享 checkout 会话决策](../../decisions/node/agent-runtime/20261115-shared-checkout-agent-sessions.md)于
2026-11-16 经 owner 批准为 `approved`。实现尚未合入 desktop 的 main，这些义务目前都是 `Missing`；
[会话执行用例](agent-runtime/session-execution.md)、[显式 Session MCP 用例](agent-runtime/session-mcp.md)与
[恢复截止与停止确认用例](revision/restore-deadline-and-stop-confirmation.md)的证据保持原范围，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [同一 checkout 的多个 Agent 会话](agent-runtime/shared-checkout-sessions.md) | 结束一个会话波及兄弟、共享目录无界累积会话、恢复在存活兄弟下移动工作区或未知停止证据被绕过、拒绝关闭控制连接或可交付、重发产生第二个终态、拒绝路径持久化 Setting 值、依赖兄弟的交接或特权遍历越出 checkout、深树耗尽描述符、硬链接保护关闭仍遍历、跨 checkout 并发遍历无界、能力缺失或混合版本边界被隐藏 |

## 会话 MCP 刷新与宿主健康

[会话 MCP 刷新与宿主健康决策](../../decisions/node/agent-runtime/20261116-session-mcp-refresh-and-host-health.md)于
Node 侧已实现为 `implemented`。17 条核心用例分别维护直接证据与缺口，Node 侧验收 4、5、6、8、12 为 Passed；
不把 Node 合入或验收通过外推为 Controller/v2 全链完成，也不把默认 ignored 的 root 用例算作 CI 通过。
[显式 Session MCP 用例](agent-runtime/session-mcp.md)、[共享 checkout 会话用例](agent-runtime/shared-checkout-sessions.md)与
[尝试锁用例](git-execution/attempt-locks-and-stop-evidence.md)的证据保持原范围，不推导本文的新义务。

| 领域 | 核心用例 | 重点风险 |
| --- | --- | --- |
| agent-runtime | [会话 MCP 刷新与宿主健康](agent-runtime/session-mcp-lifecycle.md) | 刷新扩大白名单或发送部分列表、旧 generation 回滚目标、Setting 值进入账本/日志/停止记录、准入前结算 Executed 或 MCP 门禁结束 execution、受阻轮次无界等待、刷新失败后以 session/new 重建、同一进程 load 无界累积或升级不重启、在用包版本被删、宿主子进程不持锁被判已停止、跨主机凭 boot_id 确认停止、残留挡住普通后继或放行恢复、未排空升级起不来或旧会话被直接结算、无证据的运维释放、排队探测被判超时或清理被遗弃、HTTP 探测随跳转或代理外发秘密、三层状态互相冒充 |
