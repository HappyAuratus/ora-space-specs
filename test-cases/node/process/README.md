# Node 进程运行核心用例目录

本目录与 [process 决策目录](../../../decisions/node/process/README.md) 完全镜像。13 篇根 ADR 已于
2026-09-17 获批；本目录按其责任边界登记 33 个核心用例，每个用例包含风险、场景、验证义务和决策依据。

## 用例导航

| 领域／用例文件 | 风险范围 | 用例数 |
|---|---|---|
| [architecture](architecture/ownership-and-isolation.md) | 运行责任与业务身份隔离 | 1 |
| [control](control/authorization-and-fencing.md) | 授权、租约与实际执行点撤权 | 3 |
| [lifecycle](lifecycle/attempts-and-closure.md) | 启动尝试、收尾与关闭并发 | 4 |
| [recovery](recovery/authority-and-recovery.md) | 独占恢复、失联核实与故障隔离 | 3 |
| [containment/guarantees](containment/guarantees/capability-and-evidence.md) | 实际能力与清理证据 | 2 |
| [containment/linux](containment/linux/cgroup-ownership-and-cleanup.md) | Linux 创建纳管与清理边界 | 2 |
| [containment/windows](containment/windows/job-ownership-and-cleanup.md) | Windows Job 归属与退出证据 | 2 |
| [containment/macos](containment/macos/native-cleanup-and-identity.md) | macOS 原生尽力清理与身份复用 | 2 |
| [io](io/output-and-input-delivery.md) | 输出政策、输入交付与控制可用性 | 3 |
| [protocol](protocol/streams-and-state-recovery.md) | 本地三流、状态衔接与持久接管 | 3 |
| [persistence](persistence/durability-and-retirement.md) | 持久接受、事实汇合与责任退休 | 3 |
| [upgrade](upgrade/coexistence-and-migration.md) | 版本共存、迁移与权威事实保护 | 2 |
| [integration](integration/business-entry-and-resource-handoff.md) | 业务入口、远程句柄与资源交接 | 3 |

## 证据状态

当前新体系尚未实现，全部验证义务为 `Missing`。证据表中的“需……”描述后续取证方式，不是
已有测试入口或测试通过记录。现有 `crates/process/tests/process.rs` 和
`apps/ora-reaper/tests/reaper.rs` 验证旧体系，不能替代独立宿主、Scope guardian、持久接管和
跨实例撤权的直接证据；本次没有执行或新增运行代码测试。

后续只有直接证明某项义务的测试才登记为证据：完整覆盖为 `Covered`，仅覆盖部分路径或较低层
规则为 `Partial`，没有直接证据继续保留 `Missing`。平台、实际能力和强／尽力模式分别记录，
不把 Linux 通过推导为 macOS、Windows 或受限 Linux／gVisor 通过。

## 验证边界

- 通过生产 Client、宿主／guardian 协议和业务 Adapter 验证行为；状态机测试可注入时钟、存储、
  授权和平台事实，但不替代跨进程及真实平台证明。
- 强杀与竞争场景使用显式同步点和外部终止，不用任意 sleep、正常 Drop、正常退出或单进程
  panic 代替。既验证不会错误放行，也验证条件恢复后能继续推进。
- 每条代表性证据记录测试入口、实际平台／能力、运行结果及未验证范围。尚待冻结的机制和默认
  参数按 ADR 明确后再设计具体测试，不在用例中自行补定 schema、算法或超时值。
- 英文二级标题及其 Markdown anchor 是未来代码引用的稳定入口；移动或重命名前同步全部引用。
  ADR、测试职责及证据变化时同步更新本目录；批准方案本身不代表已经满足验收。
