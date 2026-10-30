---
status: implemented
---

# Node 不可用时结束本轮并延后：Cloud 保存退避次数，clone 分轮等待同一执行

云端 Controller 遇到不可用的目标 Node 时，最多保留 2 秒握手宽限，随后以
`retry_wait/node_unavailable` 结束本轮，让出全局普通 operation 处理位置。Cloud 持久保存连续不可推进次数，
从 5 秒开始倍增退避，上限 60 秒。已经登记的 clone 每轮最多等待 5 秒，以 1 秒再领取间隔继续查询同一执行；
从该执行登记起超过 30 分钟仍无结果时，进入 `blocked/clone_result_timeout`；晚到的原执行已知结果由 Cloud
自动接管并继续推进或按原失败预算处理。Node 报告的 Unknown 仍是 `blocked/clone_result_unknown`。

本 ADR 为 `implemented`：本文规则已在下列固定版本实现，测试与部署覆盖仍按核心用例逐项登记，
`implemented` 不表示全部验证义务均为 `Covered`。Cloud 实现已随 [cloud#206](https://github.com/ora-space/cloud/pull/206) 合入
[`1cfaed3`](https://github.com/ora-space/cloud/commit/1cfaed320d653f0f2689e165f98cf3c705f3dcfa)，
Controller 实现与联合测试已随 [desktop#750](https://github.com/ora-space/desktop/pull/750) 合入
[`df694c8`](https://github.com/ora-space/desktop/commit/df694c8836fc70b1c789cc7e8e523d4d5b1b6c66)（最终 PR head [`65fc457`](https://github.com/ora-space/desktop/commit/65fc45714bdcdbcc71f2eb9263b963ab54c6a26a)）。
前期部署镜像构建于 [`9078570`](https://github.com/ora-space/desktop/commit/9078570ad26b644bd5e91d80d310815ebd1e1f0c)，仍保留该 revision 标签；
其生产源码与最终 head 相同，差异仅为 process-runtime `observations.rs` 的 `#[cfg(test)]` 测试夹具，不改变运行行为。
本决策涉及 Cloud 的 operation 持久延后与 Rust Controller 的普通 operation 驱动；不改变 Node 的 Git 执行期限。
D1/D2 在本候选仍使用的 `created_at,id` 排序下修复 #698：A 延后且 `retry_at` 在未来时不再参与领取，B 可以被领取，
A 每轮只占用 worker 最多 2 秒的本地等待。本文规则也适用于
[specs#95 D3](https://github.com/ora-space/specs/pull/95) 的轮转约定，但不依赖它，也不宣称本候选已实现轮转。
对应的验证义务登记在[不可用 Node 延后与 clone 分轮等待核心用例](../../../test-cases/controller/node-management/20261028-defer-unavailable-node-operations.md)，
逐项区分直接测试、部署记录与未覆盖分支；ADR 状态或 PR 测试通过不替代各项覆盖证据，`Partial`/`Missing` 继续保留。

## 背景与约束

[circus#698](https://github.com/ora-space/circus/issues/698) 在 cloud `1dc2a46`、desktop `d913440`、
cluster `28bac76` 上复现：每次使用全新 Compose 项目、空卷和单 Cloud 副本，在租户 A 的 Workspace
clone 阶段使 Node 不可用，再由租户 B 创建 Project。没有工作流、模型调用或 operation 行修改。

| 故障 | B 的 Project 准备耗时 | 同环境无故障基线 | 直接观察 |
| --- | --- | --- | --- |
| 暂停 A 的 Node | 173.5 秒 | 52.0 秒 | B 连续约 62 秒未被领取；A 延后后再次领取，又占约 60 秒 |
| SIGKILL A 的 Node，消失 61 秒后创建 B | 168.6 秒 | 66.5 秒 | B 首次领取很快，后续重试仍被 A 的约 60 秒轮次挡住 |

两次 B 最终均成功；准备总耗时包含快照、插件等正常步骤，不能把全部差值归因于 Node 故障。
多条失联 operation 同时存在时的影响，以及“连接着但不返回结果”的 clone，尚不能由这两次复现证明。

批准本决策时的历史代码仍有同一行为，代码引用基线为 cloud `975336f`、desktop `cab59a3c`：

| 事实 | 当前代码依据 |
| --- | --- |
| 单一全局 lease；候选按 `created_at,id` 取最早一项 | [Cloud `control.go:179–231`](https://github.com/ora-space/cloud/blob/975336fef64dd7677fed545d0bf2013a60c92b4d/internal/core/control.go#L179-L231) |
| 普通 operation 由单个 worker 串行推进；force-stop 有独立任务 | [Controller `operations.rs:38–133`](https://github.com/ora-space/desktop/blob/cab59a3c0ecd4aa31b939f29a5fca76d03a4e64e/apps/ora-controller/src/cloud/operations.rs#L38-L133) |
| `NODE_WAIT=60s`；等待 ready、clone 持续断连会占住本轮 | [常量 `operations.rs:471–474`](https://github.com/ora-space/desktop/blob/cab59a3c0ecd4aa31b939f29a5fca76d03a4e64e/apps/ora-controller/src/cloud/operations.rs#L471-L474)、[等待 `steps.rs:141–261`](https://github.com/ora-space/desktop/blob/cab59a3c0ecd4aa31b939f29a5fca76d03a4e64e/apps/ora-controller/src/cloud/operations/steps.rs#L141-L261) |
| Controller 普通延后固定 5 秒；Cloud 只为 `clone_failed` 专门计算退避，`node_unavailable` 无持久次数或总限 | [Controller 常量](https://github.com/ora-space/desktop/blob/cab59a3c0ecd4aa31b939f29a5fca76d03a4e64e/apps/ora-controller/src/cloud/operations.rs#L33)、[Cloud defer 分支 `control.go:143–165`](https://github.com/ora-space/cloud/blob/975336fef64dd7677fed545d0bf2013a60c92b4d/internal/core/control.go#L143-L165) |
| 已连接的 clone 无结果时可一直循环；现有 unresolved 宽限只覆盖已进入 unresolved 集合的执行 | [Controller `steps.rs:215–261`](https://github.com/ora-space/desktop/blob/cab59a3c0ecd4aa31b939f29a5fca76d03a4e64e/apps/ora-controller/src/cloud/operations/steps.rs#L215-L261) |

[快照 clone ADR](../../cloud/operation/20261016-clone-step-consumes-binding-snapshot.md) D2 明定旧的按创建时间领取，
[Controller 根决策](0-controller-drives-workspace-sandboxes.md) D3 允许串行推进。
60 秒轮内等待与 `node_unavailable` 固定 5 秒不是这两份 ADR 要求的参数。
已规定的 clone 退避来自 [Operation 根决策](../../cloud/operation/0-workspace-runtime-follows-desktop-node.md) D2：
真实已知失败最多三次，前两次等待 5/10 秒，次数由 Cloud 按持久执行记录持有。

## 继承与修改

| 前序约定 | 本文处理 |
| --- | --- |
| 本目录前序 [静态 Node 身份决策](20260926-static-node-identity-proven-before-dispatch.md) D1/D2：先证明身份，已登记执行不改派、不因连接问题判失败 | 继承，不变；本文只改变 Workspace 普通 operation 等待时占用处理位置的时间 |
| [会话重建决策](20260926-rebuild-sessions-from-live-sandbox-list.md)：新 epoch 按 Cloud 存活沙盒列表重建会话 | 继承，不变；延后不移除存活沙盒的会话，不停止后台重连 |
| Controller 根 D3：结束或延后当前普通 operation 后才领取下一项 | 继承串行处理；明确 Node 不可用与 clone 未返回时的短轮次（D1、D3） |
| specs#95 D1：Node 暂时不可达保留原操作，按有限原因延后 | 继承，不变；`node_unavailable` 不设次数耗尽或等待过久的失败终态（D2） |
| specs#95 D3：每次真正交出操作写 `last_claimed_at`，按 `COALESCE(last_claimed_at,created_at),created_at,id` 轮转 | 落地后原样叠加，不是本文的实施前提；该字段不作为退避次数或 clone 总窗口的起点 |
| 快照 clone ADR D2：当前 `created_at,id` 排序 | 继承，不变；D1/D2 通过未来的 retry_at 排除等待项，即可让其他可运行项被领取，不等待轮转落地 |
| Operation 根 D2：已知 clone 失败的三次预算与 5/10 秒退避 | 继承，不变；Node 不可用轮次与等待原执行均不新增 clone 尝试 |
| [最小 clone 契约](../../node/repository/20260918-minimal-clone-execution-contract.md) D6：查询原执行，未知不自动重跑或删除 | 继承，不变；D3 的 timeout 与 Node Unknown 区分，前者晚到的已知结果自动接管，不制造 Node 终态 |
| lease、epoch、operation version、execution 身份、真实结果接管与恢复规则 | 继承，不变；所有写入继续受原围栏保护，旧 epoch 先对账 |
| [安全停止与 force-stop](../../cloud/operation/20260927-safe-stop-and-administrative-force-stop.md) | 继承，不变；不改独立的 force-stop 路径、idle 或终止证据 |

## 决策

### D1：Node 不可用只等 2 秒握手宽限，然后结束本轮

本决策适用于普通 operation 的 Node 就绪检查，以及已派发 clone 等待中发生的 Node 不可达。
quiesce 如需等待目标 Node 就绪，也使用同一宽限；它的停止派发、idle 判定与 Cloud 接受条件不变。

1. 目标 Node 已满足现有握手、身份、初始化与心跳条件时，立即继续，不人为等待。没有可连接目标时直接延后。
2. 有目标但尚未就绪时，从本轮首次观察到不可用起，最多给 2 秒完成现有握手与登记。
   同一轮内的重连、短暂 connected 或重复就绪检查不重新获得宽限，也不分别等待多个 2 秒。
3. 宽限结束仍不能推进，Controller 请求 `retry_wait/node_unavailable`，Cloud 提交 D2 的计数和 `retry_at`，
   Controller 随即结束该轮并继续领取其他可运行 operation，不再在该轮等待 60 秒或等待 `retry_at` 到期。
4. 让出的是普通 operation worker，不是放弃 Controller 的全局 lease。会话重连与心跳继续独立运行。
   Cloud 不可达或 fence 失效时按既有控制连接与租约恢复规则退出，不虚报延后已经提交。

2 秒是本决策的调度政策：允许正在完成的握手跨过短暂连接竞争，同时把每条不可用 operation 的本地等待
从 60 秒限到 2 秒。它不是 Node 冷启动必须完成的期限；启动更慢只会跨轮继续等待。
该值尚无延迟分布实测支撑，不承诺 2 秒足以覆盖所有正常握手，也不包含 Cloud RPC 的网络耗时。

### D2：Cloud 按不可用次数计算 5 秒起、60 秒封顶的退避

Cloud 为每个 operation 持久保存 `node_unavailable_attempts`，初始为 0。每个被接受的
`retry_wait/node_unavailable` 轮次在同一事务把次数加一，并以 Cloud 数据库时间计算未来 `retry_at`：

| 连续不可推进次数 n | 下次可领取前等待 |
| --- | --- |
| 1 | 5 秒 |
| 2 | 10 秒 |
| 3 | 20 秒 |
| 4 | 40 秒 |
| 5 及以后 | 60 秒 |

规则为 `min(5 × 2^(n-1), 60)` 秒；实现须在求幂前限制参与计算的指数，避免长期等待产生算术溢出。
Cloud 决定等待时间，Controller 提交原因，不通过自己给出的秒数覆盖它。计数和 `retry_at` 必须共同提交或回滚；
同一轮的 RPC 重放不得被计作新的不可用轮次，继续沿原 epoch/version 规则处理重复与陈旧写入。

- Controller 重启、重新领取、epoch 换手、手动 retry 或无进度的连接恢复都不清零。
- Cloud 接受一次真实的步骤推进或 operation 完成时，在同一推进事务清零；只领取、刷新心跳或收到 pending
  不算成功推进。其他延后原因既不增加此计数，也不以它计算自己的退避。
- 未到 `retry_at` 的 operation 不参与领取；到期后按当时的领取顺序参与：当前为 `created_at,id`，
  specs#95 D3 落地后为其轮转顺序。两者均不插入优先队列，也不把轮转作为延后生效的条件。
- 不因次数多或不可用持续太久进入 `failed`；保留原 operation、冻结输入、资源和未知执行。
  若已有 clone 执行，其结果等待总窗口另按 D3 判断，达到窗口只进入 `blocked`。
- 此计数独立于持久 clone execution 数。一次 Node 失联、一次让出 worker 都不消耗真实已知失败的三次预算。

5 秒沿用现有首次恢复节奏，倍增减少持续失联项的重复领取；60 秒使长期失联项每分钟至多到期一次，
也把恢复后的额外调度等待限制在最多一个 60 秒退避周期加排队时间。选择 60 而非 120 秒，是为了减少
恢复延迟；它不是已测得的最优参数，不承诺租户级延迟上界。本文不引入 jitter 或新的配置面。

此规则由 Cloud 的 operation 持久列及追加迁移
[`0074_node_unavailable_attempts.sql`](https://github.com/ora-space/cloud/blob/1cfaed320d653f0f2689e165f98cf3c705f3dcfa/internal/core/migrations/0074_node_unavailable_attempts.sql)
承载，并在内部快照中传递恢复所需的计数与执行登记时间；不向公开 API 暴露内部身份。
该列为非负 `bigint NOT NULL DEFAULT 0`，退避计算先限制指数，持久次数到 bigint 上限后饱和。
旧行次数为 0，不从 `version` 或历史错误猜造次数；已有 clone 执行、结果、epoch 与资源绑定不改写。

### D3：clone 每轮等 5 秒，最多自动等待同一执行 30 分钟

已登记 clone 而尚无可接管结果时，Controller 使用固定的 **5 秒本轮结果等待窗口**。窗口从本轮开始等待
该执行时按单调时钟计时，不随 Node 重连、轮询或 pending 重置；D1 的就绪宽限不能额外叠加在窗口后面。
已知结果随时按原规则接管，不必等满窗口。

| 窗口内观察 | 处理 |
| --- | --- |
| Node 暂时不可用，2 秒宽限已尽 | 按 D1/D2 延后；执行仍是原执行 |
| Node 连接正常，5 秒已尽但结果尚未到达 | `retry_wait/clone_result_pending`，Cloud 保存 1 秒后的 `retry_at`，结束本轮 |
| 原执行的总结果窗口已尽，仍无可接管结果 | `blocked/clone_result_timeout`，不再普通领取或重派；保留晚到已知结果的自动接管 |
| `clone_result_timeout` 后原执行的成功结果到达 | Cloud 自动接管并推进，无需人工 retry |
| `clone_result_timeout` 后原执行的已知失败到达 | Cloud 自动接管，按 `clone_failed` 的三次尝试预算继续，无需人工 retry |
| Node 报告 Unknown | 按 clone 契约 D6 保持 `blocked/clone_result_unknown`，不由 timeout 的恢复机制自动解除 |
| 更早已有足够证据按现有契约判为未知、失败或成功 | 继续原分类与接管规则；新窗口不推迟现有 unresolved 处理 |

`clone_result_pending` 是本决策新增的有限延后原因，明确表示“继续等原执行”，不表示 Node 已失败或执行
已经 Unknown。Cloud 的允许原因、内部契约和 Controller 映射已同步扩展；不能用 `clone_failed` 或
`clone_result_unknown` 冒充普通轮次到期，也不应把正常 pending 伪装成通用 `external_failure`。
它使用固定 **1 秒再领取间隔**，不增加或清除 D2 的 Node 不可用计数，不创建新的 clone execution。
该间隔不能为 0：在当前 `created_at,id` 排序下，最老的 pending clone 会立即再被领取，持续占用下一轮，
B 就无法进入；未来 1 秒的 `retry_at` 让 A 暂时离开可领取集合。5 秒本轮等待窗口保持不变。

**总结果窗口为 30 分钟（1800 秒），起点是 Cloud 首次登记当前 clone execution 的持久 `created_at`。**
总窗口使用墙钟经过时间。该时间代表派发前的执行登记，不是 operation 创建时间或实际 Git 开始时间。Cloud 按此时间判断窗口并
在内部快照提供起点或派生截止；Controller 不从本进程启动时间重建起点。排队、断线、退避、重启和
重新领取都计入经过时间，不暂停、不续期。仅经真实已知失败、依原三次预算创建的新 execution 才有新起点。

每次恢复先按原身份查询并接管已经存在的结果，再判断是否仍无结果且超出窗口；Cloud 在持久化
`blocked/clone_result_timeout` 时重新核验，避免把并发到达的已知结果覆盖成超时。该原因是新增的有限原因，
与 `clone_result_unknown` 分别持久保存；Cloud 允许原因、内部契约和 Controller 映射已同步扩展。
没有新证据时，超时只意味着不再占用普通 worker 等待：
不发送新的 `CloneRepository`，不改变 execution/Node/输入，不伪造失败，不清理目录或终止 Node。

`clone_result_timeout` 不关闭原执行的结果交付或接管。原执行的真实已知结果晚到时，Cloud 的结果接管路径
自动识别 timeout 原因并继续处理，不依赖 blocked operation 再被普通领取，也不要求人工 retry：
成功按原规则推进；已知失败按 `clone_failed` 的三次预算及 5/10 秒退避处理，预算耗尽则按原规则失败。
接管继续核验原执行身份与原有围栏，清除的是 timeout 阻塞，不重置执行尝试次数或篡改历史结果。
Node 报告的 Unknown 仍按 clone 契约 D6 保持 `blocked/clone_result_unknown`；不能仅因经过 30 分钟或重新连接，
套用 timeout 的自动恢复分支。手动 retry 也不能重置原执行截止或生成替代执行。

选择 5 秒本轮窗口，是为了比 2 秒握手宽限多留一次短结果交付机会，同时避免一个健康连接的无结果执行
长期占住 worker。30 分钟给一次最长 20 分钟的网络 Git 命令留出 10 分钟结果接管余量，依据是
[最小 clone 契约 D8](../../node/repository/20260918-minimal-clone-execution-contract.md#d8网络-git-采用独立且有上限的预算)。
但网络命令可多次发生；它不是完整 clone 最长耗时的证明。慢 clone 可能仍在执行而 operation 已 blocked，
因此保留原执行并自动接管晚到已知结果是必要条件。这两个窗口已获 owner 接受，不是 #698 的测量结论。

## 不变量

1. 暂时不可达保留原 operation；仅 `node_unavailable` 次数或经过时间不产生失败终态。
2. 不可用宽限和本轮结果窗口不能被重连、轮询或同轮重复检查续期。
3. 同一 pending clone 跨任意多轮仍是同一 execution、目标和冻结输入；让出 worker 不增加 clone 尝试。
4. Cloud 持有退避次数、下次领取时间和总结果窗口依据，Controller 重启或换手不重置它们。
5. 已知结果接管先于未知判断；等待到期不证明 Node 已结束、资源空闲或可以删除。
6. lease、epoch、version fencing、对账、恢复与 force-stop 继续遵守原契约。
7. 本文不依赖领取轮转；未来的 retry_at 在当前排序与 specs#95 D3 排序下都排除尚未到期的等待项。
8. `clone_result_timeout` 后原执行的晚到已知结果由 Cloud 自动接管：成功推进，失败沿原三次预算处理，
   无需人工步骤；Node Unknown 仍是 `blocked/clone_result_unknown`，不因 timeout 恢复机制解除。

## 为什么不是这些替代方案

| 方案 | 取舍与否决理由 |
| --- | --- |
| 只实施 specs#95 D3 轮转 | 能在有限集合里轮转，但 operation 在本轮占住 60 秒期间，新建项的 created_at 仍晚于它的 last_claimed_at；它到期后可再先领取。有 N 条时，新项仍可能多等约 N×60 秒。D1/D2 在当前排序下即可缩短占用并排除未到期项，不等待轮转合并；轮转落地后叠加 |
| Controller 计算退避，Cloud 只存结果 | 若次数只在 Controller 内存，重启或换手就丢失；即使 Cloud 保存次数，算法仍随 Controller 版本变化，次数递增与 retry_at 的决策分开。由 Cloud 统一计算可沿用已知 clone 失败的权威边界 |
| 增加并行 worker | 可提高吞吐，但改变当前领取/版本竞争边界，需要另定所有权与并发安全；也不能单独限制每个无结果执行的等待。本决策保留单 worker |
| 立即零宽限延后 | 最快释放位置，但会把刚好正在完成的正常握手全部推到下一轮；2 秒是显式、有限的折中 |
| `clone_result_pending` 以 0 秒再领取 | 没有额外结果等待，但在创建时间排序下最老项会立即再领，B 无法进入；选择未来 1 秒让出领取机会 |
| 总窗口到期视作 clone_failed 并重新 clone | 无需人工处理，但未知执行可能仍写原目录，会产生第二次副作用，违反 clone 契约 D6 |

## 后果与兼容性

- 不可用项每轮本地等待从 60 秒降至最多 2 秒；连接着但无结果的 clone 每轮最多等 5 秒。
  N 条不可用项仍可能带来约 N×2 秒加 RPC/调度开销，本文不承诺零排队或按租户配额公平。
- 慢启动会跨轮完成；失联恢复可能多等至多 60 秒退避加排队。健康 clone 的结果在让出之后到达时，
  可能增加约最多 1 秒加排队时间，才能在下一次领取后推进。
- Cloud 增加持久计数和有限原因，分轮等待产生更多 claim 和 defer 写入；换来 worker 能服务其他租户。
- N 条慢 clone 同时在途时，新 operation 可能先等待约 N×5 秒的轮内占用，再加 RPC 与调度排队开销；
  本文不把这一估算当作严格的租户级延迟上界。
- 30 分钟到期的 `blocked/clone_result_timeout` 自动接管原执行晚到的已知结果，不需要人工步骤；
  Node Unknown 的 `blocked/clone_result_unknown` 仍保留。两者都不是物理终止或清理授权。
- 产品实现应先部署能接受新原因及持有计数的 Cloud，再部署使用它的 Controller。旧 Controller 的 60 秒
  等待不会被 Cloud 退避单独修复；不把混合版本视为完成本 ADR 的行为。

## 本决策未解决的问题

- 正常握手和大仓库端到端耗时分布尚无本决策的测量，不能据此宣称参数最优；已接受的政策值和
  `clone_result_pending` 命名不再作为待确认事项，timeout 后晚到已知结果的恢复不留给人工处理。
- specs#95/cloud#74 的领取轮转与迁移协调属于对应 owner 的范围；本文不依赖轮转，也不接管其其他领取结算工作。
- 租户配额、多 worker、其他长 Substrate/插件调用的调度，以及未知执行的物理回收，均不在本决策内。
- Cloud 已实现正常 `retry_wait/node_unavailable` 的服务端退避，但 `blocked/node_unavailable` 的参数边界仍有
  独立后续问题：`operationCommand` 对该原因豁免 `retrySeconds` 的 1..3600 校验，blocked 请求可能落入通用
  更新并使用调用方 delay。有限原因白名单不等于所有原因/状态/参数组合均已封闭；本决策不赋予正常
  Controller 路径新的 blocked 延后语义，也不宣称该组合已修复或验证。

## 落地顺序与验证

以下保留原批准的实施与验证义务；Cloud `1cfaed3` 与 Controller PR head `9078570` 已实现本文行为，
Desktop 合并版本和部署覆盖按实际证据登记，不删除未覆盖义务或以状态代替验证。
[核心用例](../../../test-cases/controller/node-management/20261028-defer-unavailable-node-operations.md)按最终决策逐项登记，
同时承担 Controller 轮次、Cloud 持久状态与跨进程结果接管义务，已分别标为 `Covered`、`Partial` 或 `Missing`。
源码证据以 cloud `1cfaed3`、desktop `9078570` 为基线，直接入口见该核心用例的固定源码链接。
受控联合测试使用真实 Rust worker、Go gRPC/Store 与 PostgreSQL，Node 为协议替身；夹具预置步骤并调老
原 execution 的 `created_at`，不能替代真实 30 分钟等待、真实 Git 或下述部署故障时间线。

1. **Cloud 契约和状态**：在当前 `created_at,id` 排序下追加持久计数迁移，接收 pending/timeout 原因，原子计算
   D2 的 retry_at、pending 的未来 1 秒 retry_at，并按原 execution 的 created_at 墙钟时间判断总窗口。
   结果接管路径须能直接恢复 timeout 操作，不依赖它被再次普通领取。完成条件是持久计数、重复/陈旧调用、
   回滚、已知结果竞争及晚到结果自动恢复的直接测试通过；不改变 clone 的三次已知失败预算。
2. **Controller 轮次**：为 Node 就绪、clone 断连与 connected/pending 等待实现 D1/D3；每次领取继续使用
   最新快照、先对账、再推进。完成条件是同一 worker 能在短轮次后处理另一租户，并保持原执行身份。
3. **单元与集成测试**：以受控时钟测试边界，使用真实 Cloud 持久事务和 Rust Controller 的联合测试覆盖
   下表，先验证当前排序；specs#95 D3 落地后在轮转排序下复用这些场景，不把其合并作为本修复验收前置。
   单元替身结果不能替代跨进程持久恢复证据。去掉对应修复时，每项核心断言须失败。

| 场景 | 必须成立；去掉修复后的失败点 |
| --- | --- |
| 当前创建时间排序下 A 的 Node 始终未就绪，B 在 A 本轮期间创建 | 2 秒宽限后 A 进入未来 retry_at 的 retry_wait，被排除后 B 获得 worker，不依赖 D3 轮转；旧实现仍等 60 秒 |
| Node 在宽限前完成握手；以及反复连接/断开却一直不能推进 | 前者继续推进；后者仍在同一 2 秒预算后让出，不被连接事件续期 |
| 连续不可用、Controller 重启/epoch 换手、手动 retry | Cloud 次数和 5/10/20/40/60/60 秒序列保持；旧实现或 Controller 内存计数会回到固定 5 秒 |
| defer 事务回滚、RPC 重放或旧 version/epoch；随后真实推进 | 回滚不改次数/时间，重放不多计，陈旧写入拒绝；成功推进清零；不影响 clone execution 数 |
| 当前排序下 Node 连接正常且永远不返回结果，B 有可推进工作 | 5 秒后 A 以 pending 原因让出，retry_at 是未来 1 秒；此时 B 可被领取，重领仍查询同一 execution、派发一次；旧循环不结束，间隔为 0 则 A 立即再领而 B 无法进入 |
| 原执行登记后 30 分钟墙钟边界、跨重启/退避，或超时提交前已有结果 | 窗口前只分轮等；到期仍无结果才 blocked/clone_result_timeout；已有结果先接管；不新派发、不删除、不以到期消耗 clone_failed 预算 |
| clone_result_timeout 后原 execution 的成功结果晚到 | Cloud 自动接管并推进，无人工 retry，也不依赖 blocked 项再领取；execution 身份和派发次数不变；缺少自动恢复时操作会留在 blocked |
| clone_result_timeout 后原 execution 的已知失败晚到 | Cloud 自动接管并沿 clone_failed 三次预算处理，前两次 5/10 秒退避、第三次 failed，无人工 retry、不清零次数；缺少自动恢复时仍停在 blocked |
| Node 报告 Unknown，随后跨过 30 分钟或重新连接 | 保持 blocked/clone_result_unknown，不改成 timeout、不自动解除、不重派；混用超时恢复分支会使本断言失败 |
| 一次真正 clone 失败与多次 pending/node_unavailable 交错 | 只有真实新 execution 消耗三次预算；已知失败仍走原 5/10 秒及第三次 failed，未知不自动重试 |

4. **沿 #698 方法做修复前后部署复现**：固定每组 Cloud/Controller/Node/cluster 版本，每个场景用新隔离环境
   与空卷，单 Cloud 副本，不跑工作流或模型，不改 operation 行。先测同版本无故障基线；分别在 A 的 clone
   阶段暂停 Node、以及 SIGKILL 并在 Node 消失 61 秒后创建 B。记录 B 创建到首次领取、各轮领取/延后/推进、
   完成耗时，以及 A 每轮占用时间、Cloud 次数/retry_at、clone execution 身份和结果。并列展示修复前、修复后
   与各自无故障基线；保留 #698 的 173.5/168.6 秒和 52.0/66.5 秒作为历史对照，不将其当作新版本固定门槛。
5. **多条不可用 operation**：至少三个不同 Project 各有自己的 Workspace Node，这些 Node 同时失联，
   各自的旧操作持续经历领取与延后轮次。旧项循环仍在进行时，创建健康租户 B 的 Project，再在旧项轮次期间
   加入健康租户 C 的 Project；B/C 必须真实进入同一普通 operation 队列并获得推进。先在当前排序检查所有旧项
   每轮宽限、Cloud 退避与 B/C 的后续推进，
   比较同版本无故障与旧实现；轮转落地后再复用场景，不阻塞当前修复。必须没有每条旧项 60 秒的阻塞；
   报告实际排队时间，不用“最终成功”替代延迟证据。
6. **连接正常但无结果**：让原 clone 保持会话与心跳但不交付结果，观察跨轮执行身份及 B 推进；以原执行
   分别注入 timeout 后晚到的成功和已知失败，验证无人工步骤自动恢复；另一路报告 Node Unknown，验证仍然
   blocked/clone_result_unknown。不能用断连测试替代本场景，也不能把人工预置截止时间的测试冒充真实
   30 分钟部署等待。部署验收若未覆盖这一路，证据保持未覆盖，不宣称完整实现。

本 ADR 按上述固定实现版本登记为 `implemented`；Controller 实现已随 desktop#750 合入 `df694c8`。
Cloud/Controller 两侧直接测试与上述联合路径的实际证据、未覆盖分支及获接受的偏差继续逐项登记，不以
调老时间的联合测试冒充真实部署总窗口。最终运行附件位置与部署版本在核心用例的“证据位置与保留缺口”
登记，未提供时不填写虚构路径或摘要值。保持不变的行为：force-stop、lease、epoch、执行身份与恢复规则。改变的行为：普通 operation
不再因不可用 Node 或尚无结果的 clone 长时间占用处理位置，退避与总窗口依据由 Cloud 持久持有。
