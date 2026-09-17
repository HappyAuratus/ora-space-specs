# 运行责任与业务身份隔离

本文件验证进程运行体系的跨进程责任边界。具体撤权、恢复和资源放行分别由相应领域用例继续约束。

## Runtime Responsibility Survives Node Instance Replacement

### 风险

把 Node 内存或进程身份当作运行身份，会在实例更替时重复执行，或丢失已经接受的责任。

### 前置状态

一个 Execution 已关联多个 Run；Scope A 采用允许接管的保留策略，Scope B 承担无冲突工作。宿主与两个 guardian 独立运行。

### 触发

暂停或强杀原 Node，授权替代实例恢复查询和管理；在 A 的事实汇合处制造暂时故障。

### 必须成立

- 恢复原 Execution／Scope／Run 关联；宿主、guardian 与业务层继续分别提供管理意图、运行事实和业务解释。
- A 的事实暂缺保持未知并继续协调，不能靠重启业务恢复；B 的无冲突运行与 I/O 继续。

### 禁止结果

用新 Node 实例身份替换旧 RunId；把宿主未收到回复当成未执行；把 guardian 退出码直接等同业务成功；连带终止 B。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 运行身份和接受责任跨 Node 实例保持稳定 | 尚无直接证据；需独立 Node／宿主／guardian 与持久记录验证 | `Missing` |
| 事实缺失不越过权威边界推断未执行或业务成功 | 尚无直接证据；需在事实汇合与业务结果解释边界注入失败 | `Missing` |
| Scope A 的局部故障不终止无冲突的 Scope B | 尚无直接证据；需并行运行真实进程并观察 B 的持续副作用 | `Missing` |

### 决策依据

[根决策](../../../../decisions/node/process/architecture/0-node-host-and-scope-guardians.md) D1、D2 与不变量 1–3。
