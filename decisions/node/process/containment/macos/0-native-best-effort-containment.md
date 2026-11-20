---
status: superseded
---

# macOS 原生执行并明确采用尽力纳管

> 已被取代：本决策由 [Ora Desktop 只支持 Windows 和 Linux](../../../../desktop/platform/0-windows-and-linux-only.md) 取代，原因：owner 于 2026-11-20 决定 Ora Desktop 不再支持 macOS，v0.6.0 是最后一个 macOS 版本；本决策从未实现，新的运行体系一直没有 macOS adapter。当前系统承诺以取代者为准，不再有未来的 macOS 纳管 ADR。正文里链接的核心用例只服务本决策，已随取代一并删除，链接不再有效。

macOS 原生运行本机 Git 与插件，以 guardian、进程组和后代跟踪组合提供经验证的尽力清理。
不以 Linux VM 替代原生执行，也不将进程组清空宣称为全部历史后代退出。

本根决策已于 2026-09-17 获批（`approved`），沿用[保证契约](../guarantees/0-explicit-containment-and-cleanup-evidence.md)。
具体算法和完成判据尚未冻结；它不承诺原生强保证，不将磁盘旧 PID 登记直接视作安全信号目标。

批准确认本文的架构选择与不变量；实现和平台验证仍待完成，文末待冻结事项不因批准视为已解决。

## 当前范围与未来 ADR 责任（2026-09-18）

当前新 process host／guardian 没有该平台 adapter。crates CI 的该平台通过不证明本 ADR 的真实平台纳管和故障恢复已实现。

本 ADR 保留 approved，表示平台方向曾获批准，而不是当前版本交付义务或 implemented。
下文尚未实现的机制、不变量和验收保留为未来 ADR 的设计输入；恢复此路线时，必须在本叶子
追加 ADR，继承已批准方向并明确具体部署、机制、身份与直接证据，再开始宣称平台支持。
不通过删除其全部实质范围把本文标为 implemented。当前最小闭环不等待此路线完成。

## 背景与约束

原生 Git 和插件需要使用本机环境。launchd 的相关清理语义针对同进程组，而 `setsid` 可建立新会话
和进程组，仅依赖这些机制不足以证明全部历史后代退出。参见
[Apple launchd 源文档](https://github.com/apple-oss-distributions/launchd/blob/main/man/launchd.plist.5)
和 [Apple setsid 文档](https://developer.apple.com/library/archive/documentation/System/Conceptual/ManPages_iPhoneOS/man2/setsid.2.html)。

## 决策

### D1：原生尽力模式显式呈现残留风险

启动前声明并保存尽力保证，按所承诺的跟踪与清理政策执行。可以在完成尽力清理和业务对账后
按政策自动恢复，但保留未取得完整退出证明及残留写入可能性的事实。已知冲突进程仍活着或清理
失败时不能报告完成。请求必须强纳管时不能静默使用本模式。

### D2：信号目标必须核实身份

验证创建时分组、后台化、新会话、后代跟踪丢失及 guardian 强杀后的发现。PID 和进程组编号可能
复用，磁盘旧编号不是充分身份依据；发送信号前必须按平台方案核实目标，避免误杀无关进程。

## 未来启用能力时必须保留的底线

1. 本机 Git 和插件原生执行，实际保证可查询且始终标识为尽力。
2. 清理完成以明确平台判据为依据，重试次数或超时不能替代判据。
3. 无法核实的旧 PID／进程组不被盲目作为清理目标。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 使用 Linux VM 获得强纳管 | 可利用 Linux 机制，但改变本机 Git／插件执行环境与本版范围 |
| 进程组为空即宣称强静止 | 判断简单，但 `setsid` 和后台化可能脱离原组 |
| 完全禁止异常后的自动恢复 | 保守，但超出已选择的尽力恢复政策；应依据清理和业务对账决定 |

## 未来 ADR 的落地与验收输入

先冻结尽力完成判据和身份核实，再用真实原生进程验证分组、后台化、跟踪缺口、编号复用及
guardian 强杀。证据须说明未覆盖范围，同时验证清理失败不放行、条件恢复后可继续工作。

## 核心测试用例

[风险与验证义务](../../../../../test-cases/node/process/containment/macos/native-cleanup-and-identity.md)记录本决策的核心用例；
当前直接实现证据为 `Missing`，实现后按验证结果更新。

## 本决策未解决的问题

跟踪算法、身份核实手段和尽力完成判据仍需原生验证；不从 launchd 文档直接推导已完成实现。
受限 Linux／gVisor 的尽力实现归 Linux 决策，不能共用一个未经证明的 Unix 强弱判断。
