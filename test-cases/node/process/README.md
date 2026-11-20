# Node 进程运行核心用例目录

本目录与 [process 决策目录](../../../decisions/node/process/README.md) 完全镜像。

host／guardian 体系已被 [Node 直接执行 Git 决策](../../../decisions/node/git-execution/0-node-runs-git-directly-with-attempt-locks.md)
取代。architecture、control、lifecycle、recovery、io、protocol、persistence、upgrade、integration 九个子目录下的
26 个用例只服务被取代的决策，已经删除，子目录为保持镜像而保留。取代者的用例见
[Node 直接执行 Git 的核心用例](../git-execution/attempt-locks-and-stop-evidence.md)。

containment 中 guarantees、linux、windows 三份决策保持 `approved`，它们的 8 个用例继续保留。macOS 的决策已被
[Ora Desktop 只支持 Windows 和 Linux](../../../decisions/desktop/platform/0-windows-and-linux-only.md) 取代，它的 2 个用例
只服务该决策，已经删除，`containment/macos` 子目录为保持镜像而保留。

## 用例导航

| 领域／用例文件 | 风险范围 | 用例数 |
|---|---|---|
| [containment/guarantees](containment/guarantees/capability-and-evidence.md) | 实际能力与清理证据 | 2 |
| [containment/linux](containment/linux/cgroup-ownership-and-cleanup.md) | Linux 创建纳管、部署权限、无特权尽力清理边界 | 4 |
| [containment/windows](containment/windows/job-ownership-and-cleanup.md) | Windows Job 归属与退出证据 | 2 |

## 证据状态与当前范围

containment 的用例描述 Strong 纳管和各平台纳管的长期方向，生产只有 Node 直接执行 Git 的 BestEffort。它们原有的
证据来自 `process-runtime` 和 `ora-process-helper` 的测试，这些代码已随 host／guardian 一起删除，8 个用例的义务
现在都没有直接证据，状态均为 Missing，作为将来强纳管落地时的验收输入保留。

Covered 表示义务有直接证据；Partial 保留组合或平台缺口；Missing 没有直接证据。特权测试被忽略时不算通过。

## 验证边界

- 强杀与竞争场景使用显式同步点和外部终止，不用任意 sleep、正常 Drop、正常退出或单进程 panic 代替。
- 每条代表性证据记录测试入口、实际平台／能力、运行结果及未验证范围。
- 英文二级标题及其 Markdown anchor 是代码引用的稳定入口；移动或重命名前同步全部引用。
