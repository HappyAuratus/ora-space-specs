# Node 进程运行决策

本目录只作导航，ADR 保留原叶子路径和引用。2026-09-18 按 Eric 要求收紧原 approved 蓝图：
当前交付是 **Linux、可信本机、无 root、BestEffort、独立 host／guardian 和 Node Git**。
implemented 仅表示对应 ADR 收紧后的承诺已实现和验证，不表示全部 process 蓝图完成。

**host／guardian 已被取代。**Node 自己的 Git 改为由 Node 直接执行，见
[Node 直接执行自己的 Git](../git-execution/0-node-runs-git-directly-with-attempt-locks.md)。本目录下 architecture、
control、lifecycle、recovery、io、protocol、persistence、upgrade、integration 的决策整份标为 `superseded`，正文作为
历史记录保留；containment 的决策保持 `approved`，继续定义保证等级和各平台的强纳管方向。下面各节描述的是被取代的
体系，取代者的实现合并之前，desktop 里仍在运行的是这套实现。

本次直接修订既有 ADR 是 Eric 的明确要求；保留未来设计方向和验证缺口，不创建一批空白未来 ADR。
以后恢复延期能力时，在对应叶子追加日期 ADR，继承或修改当前规则，再补直接验收。

## 当前 ADR 状态与范围

| ADR | 状态 | 当前承诺／保留原因 |
|---|---|---|
| [architecture](architecture/0-node-host-and-scope-guardians.md) | superseded | Node／host／guardian 权威分离和独立 Linux 进程组合 |
| [control](control/0-authorized-generations-and-leases.md) | superseded | 主题收敛为可信本机宿主会话控制；不冒充 Controller 授权 |
| [lifecycle](lifecycle/0-durable-run-attempts-and-scope-closure.md) | superseded | 持久尝试与收尾；完整 Start／Close 竞争证据未齐，缺口在用例中 |
| [recovery 根决策](recovery/0-single-authority-and-failure-recovery.md) | superseded | D1 表中的当前行为；完整恢复故障矩阵未齐，缺口在用例中 |
| [rootless bootstrap](recovery/20260917-rootless-guardian-bootstrap-and-reconnect.md) | superseded | 原锁交接、原实例重连、持久宿主绑定和死亡缺口 |
| [guarantees](containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) | approved | 不混淆强／尽力的契约；生产只启用 Linux BestEffort |
| [Linux cgroup](containment/linux/0-cgroup-v2-containment.md) | approved | 获批方向，Strong 后续 ADR 负责实施和验收 |
| [Linux helper](containment/linux/20260917-independent-privileged-helper.md) | approved | 预检／inspect／底层降权已有部分实现，不是 Strong 闭环 |
| [Windows](containment/windows/0-creation-time-job-containment.md) | approved | 新运行体系无该平台 adapter；未来 ADR 负责 |
| [macOS](containment/macos/0-native-best-effort-containment.md) | approved | 新运行体系无该平台 adapter；未来 ADR 负责 |
| [io](io/0-bounded-output-and-fenced-input.md) | superseded | 丢弃／有界内存输出、偏移读取、控制隔离；stdin 关闭 |
| [protocol](protocol/0-independent-local-streams-and-state-recovery.md) | superseded | 有界本机请求／查询；无订阅／结果接管协议 |
| [persistence](persistence/0-independent-journals-and-responsibility-retirement.md) | superseded | 独立 WAL／FULL；完整存储故障矩阵未齐，无退休回收 |
| [upgrade](upgrade/0-version-coexistence-and-safe-migration.md) | superseded | 已知迁移／错误版本拒绝；完整布局与迁移中断矩阵未齐 |
| [integration](integration/0-shared-process-entry-and-resource-handoff.md) | superseded | Node 的 clone 与 Revision Git 经 host；不含插件／Agent 进程与 Backend 切换 |

## 未来 ADR 责任地图

| 未来主题 | 所属叶子／领域 | 当前处理 |
|---|---|---|
| Controller 授权、密钥／令牌、Node 代次、租约、角色与同 UID 隔离 | control | 信任本机 Node，不提供此安全保证 |
| Linux Strong、helper 启动 IPC／持久协调、权限隔离及独立清理 | containment/linux | rootless BestEffort；不自动安装或提权 |
| Windows Job 与 macOS 原生 adapter、身份复用和强杀证明 | containment/windows、containment/macos | 不从旧 crates CI 推导支持 |
| 插件双向 I/O、输入交付未知、终端、背压／日志轮转、持久输出 | io | stdin 关闭，有界易失输出 |
| 事件订阅、快照游标、多观察者、process 完成结果接管 | protocol | 查询原身份，不把 Events 端点当订阅 |
| 结果责任解除、guardian／Scope 退休、GC 和旧身份拒绝压缩 | persistence | 保留记录与 guardian；成本可能持续增长 |
| 自动版本共存、恢复工具和部署回滚工作流 | upgrade | 不兼容拒绝，保留原管理材料 |
| 机器重启、休眠、服务管理器存续、guardian 死亡后独立清理 | recovery／平台 | 缺证据保留未知，不猜成功 |
| 插件接管、Backend 写入入口切换、跨 Node 共享存储 | integration 及所属业务领域 | 不自动迁移旧执行 |
| Controller–Node IPC、持久接管和业务事件 | Node protocol／Controller | 不由 process 协议的 implemented 代替验收 |

未来主题不等于已审批的新方案；启用前仍需新的具体 ADR。
当前已实现能力的存储、并发、文件保护和资源冲突缺口不因范围收紧而成为可忽略的延期项。

## 已确认的范围判断

2026-10-13 specs 治理（总事项 ora-space/circus#49）中 owner 确认：已有实现的 ADR 贴着实现转 `implemented`，缺口
留在核心用例中如实登记。据此：

1. control：当前主题收敛为“可信本机宿主会话控制”，授权／租约由未来 ADR 另议。
2. guarantees 与各平台 containment：保持 `approved`，表示长期方向；当前运行范围只启用 Linux BestEffort。
3. integration：Worktree 已被放弃，受管 Git 的业务入口是 clone 与 Revision，每次 clone 独占目录，原先“同仓库
   不同 worktree 共享元数据”的冲突问题在当前范围内不存在。
4. 无退休回收会保留空闲 guardian 与日志，作为已知限制保留；这不是已实现 GC 的声明。

lifecycle／recovery／persistence／upgrade 的测试缺口是实现验收工作，登记在各自的核心用例中。

## 实施顺序

已交付：Linux adapter → 独立 guardian → 持久 host → Node 变更关联与资源恢复 → 独立 Node 入口。
2026-09-18 首条业务闭环已改为[clone 指定仓库与分支](../repository/0-clone-selected-repository-branch.md)。
随后 clone 契约、Node 执行适配、Controller–Node IPC 与持久协调依次落地；没有按原 Worktree 剩余步骤推进。process 已交付事实及当前实现状态不因目标变化而改变。
未来能力按需求另开 ADR，不再要求全平台、Strong、租约、完整 I/O／GC 先全部完成。

## 故障验证与证据

[核心用例索引](../../../test-cases/node/process/README.md)保留原稳定英文 anchors，并区分
当前义务和未来设计输入；Covered／Partial／Missing 只描述直接证据，未来项不得改称已通过。

本次依据 desktop 的 bdc6354e、66f61234、6d7c7a57、fd18e1de 及既有 host／guardian 实现。
2026-09-17 全量 task test 通过；2026-09-18 重新运行 process-protocol、process-runtime、
process-host、process-guardian、ora-node 测试。平台为本机 Linux，特权 ignored 不算通过。
真实 Node／host／guardian 强杀证据不推导物理断电、服务管理器或其他平台保证。

## 原方案与术语

原 35 项方案仍可从 Git 历史追溯；已实现承诺以各 ADR 当前正文为准，不以历史路线图扩大交付。
术语见[领域词汇表](../../../domain/glossary.md#node-process)；其中长期授权／租约概念不表示本次已启用。
本地 process 协议与 [Controller–Node Protocol](../protocol/0-controller-node-protocol.md)是不同边界。
