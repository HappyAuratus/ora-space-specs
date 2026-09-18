# Node 进程运行核心用例目录

本目录与 [process 决策目录](../../../decisions/node/process/README.md) 完全镜像。13 篇根 ADR 已于
2026-09-17 获批；连同后续 Linux helper 与无特权 Guardian 启动／重连决策，本目录登记 36 个核心用例，每个用例包含风险、场景、验证义务和决策依据。

## 用例导航

| 领域／用例文件 | 风险范围 | 用例数 |
|---|---|---|
| [architecture](architecture/ownership-and-isolation.md) | 运行责任与业务身份隔离 | 1 |
| [control](control/authorization-and-fencing.md) | 授权、租约与实际执行点撤权 | 3 |
| [lifecycle](lifecycle/attempts-and-closure.md) | 启动尝试、收尾与关闭并发 | 4 |
| [recovery](recovery/authority-and-recovery.md) | 启动资格交接、独占恢复、失联核实与故障隔离 | 4 |
| [containment/guarantees](containment/guarantees/capability-and-evidence.md) | 实际能力与清理证据 | 2 |
| [containment/linux](containment/linux/cgroup-ownership-and-cleanup.md) | Linux 创建纳管、部署权限、无特权尽力清理边界 | 4 |
| [containment/windows](containment/windows/job-ownership-and-cleanup.md) | Windows Job 归属与退出证据 | 2 |
| [containment/macos](containment/macos/native-cleanup-and-identity.md) | macOS 原生尽力清理与身份复用 | 2 |
| [io](io/output-and-input-delivery.md) | 输出政策、输入交付与控制可用性 | 3 |
| [protocol](protocol/streams-and-state-recovery.md) | 本地三流、状态衔接与持久接管 | 3 |
| [persistence](persistence/durability-and-retirement.md) | 持久接受、事实汇合与责任退休 | 3 |
| [upgrade](upgrade/coexistence-and-migration.md) | 版本共存、迁移与权威事实保护 | 2 |
| [integration](integration/business-entry-and-resource-handoff.md) | 业务入口、远程句柄与资源交接 | 3 |

## 证据状态与当前范围

2026-09-18 随 ADR 范围收紧同步本目录。当前是 Linux 可信本机 rootless BestEffort，
真实 host／guardian／Node Git 已接通；架构、有界输出、本机查询协议和 bootstrap ADR 标为
implemented，其余状态及保留原因见[决策索引](../../../decisions/node/process/README.md)。

当前直接证据覆盖独立 Node／host 强杀恢复、原身份去重、持久绑定与旧通道拒绝、guardian
丢失不伪造清理、局部恢复、有限输出及停止隔离。2026-09-18 重跑 process-protocol、
process-runtime、process-host、process-guardian、ora-node 测试通过；基线 fd18e1de。
本次是文档范围与证据维护，没有实现代码变更。

Controller 授权／租约、stdin、订阅与结果接管、退休回收、Strong、Windows／macOS 新 adapter
及自动版本部署用例保留原 anchor，明确标为未来 ADR 输入，证据仍 Missing／Partial。
正文中“必须成立”对未来项只表示启用能力时的验收要求，不是当前 API 承诺。
现有 Start／Close 并发、存储失败、迁移中断和共享仓库资源边界缺口仍属当前工作，不靠延期抹除。

Covered 表示收紧后明确义务有直接证据；Partial 保留组合或平台缺口；Missing 没有直接证据。
特权测试 ignored 不算通过，不把旧 process/reaper 的平台 CI 当成新独立运行体系验收。
核心用例数量仍为 36（范围说明不是新增用例）；英文标题因稳定引用保留，不从其旧名推导新功能。

## 验证边界

- 通过生产 Client、宿主／guardian 协议和业务 Adapter 验证行为；状态机测试可注入时钟、存储、
  授权和平台事实，但不替代跨进程及真实平台证明。
- 强杀与竞争场景使用显式同步点和外部终止，不用任意 sleep、正常 Drop、正常退出或单进程
  panic 代替。既验证不会错误放行，也验证条件恢复后能继续推进。
- 每条代表性证据记录测试入口、实际平台／能力、运行结果及未验证范围。尚待冻结的机制和默认
  参数按 ADR 明确后再设计具体测试，不在用例中自行补定 schema、算法或超时值。
- 英文二级标题及其 Markdown anchor 是未来代码引用的稳定入口；移动或重命名前同步全部引用。
  ADR、测试职责及证据变化时同步更新本目录；批准方案本身不代表已经满足验收。
