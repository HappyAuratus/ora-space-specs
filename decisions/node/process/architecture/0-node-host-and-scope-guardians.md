---
status: implemented
---

# 独立宿主协调意图，Scope Guardian 拥有运行事实

当前 Linux 可信本机路线使用独立 host 和每 Scope 一个 guardian。Node 管业务执行与资源对账，
host 管持久意图和协调，guardian 管进程、输出与收尾。宿主重启不重新启动原 Run。

本文于 2026-09-18 按已交付范围收紧并标为 implemented；不表示全部 process 蓝图完成。
旧 Backend／插件仍走原入口，不自动迁移活进程；文件系统兼容边界由 upgrade 决策约束。

## 决策

### D1：事实按所有者分离

Node 保存 Execution 及变更尝试的 Scope／Run 关联；host 保存创建、启动、停止、关闭意图及查询投影；
guardian 独立保存接受记录和运行事实。退出码不等于业务成功，投影陈旧不等于尚未执行。
一个 Execution 可有多个 Run，Scope 不固定等于业务执行或 worktree 路径。

### D2：独立进程承担故障责任

每 Scope 的 guardian 独立 session、数据库及锁，不依赖 Node 或 host 的 Drop。
host 故障可重连原 guardian；guardian 丢失不能由替代实例重新执行旧身份，也不宣称已清理。
Node 为 Git 注入 owner 退出清理策略，该策略不等于 Controller 租约。

### D3：模块按责任组合

protocol 拥有共享消息；client 只依赖协议而不依赖 runtime；runtime 拥有 host、guardian、
平台和持久化实现；两个薄 app 负责部署入口。Node 的 ManagedGitRunner 实现 gitlancer 的
GitRunner，gitlancer 不依赖 Node 或 process。通用 OS／路径能力归 ora-utils。

### D4：明确本次范围调整

原 approved 蓝图中的 Controller 授权、租约、双向 I/O、订阅、结果接管回收、插件及全平台
集成不再是本 ADR 的完成条件，由本目录索引列出的未来 ADR 分别负责。
当前选择可信本机、Linux BestEffort、有限输出和轮询查询；不新增安全沙盒或资源调度配额。

## 不变量

1. 重连不更换原 Scope／Run 身份或生成第二次尝试。
2. 业务解释、管理意图和运行事实各有所有者，不用跨库原子事务伪装共享权威。
3. 同一次业务尝试不同时交给新旧执行体系。

## 取舍与后果

继续依附 Node 内存成本较低，但无法在 Node 强杀后独立收尾；单宿主直接持有所有进程会扩大
宿主故障范围。每 Scope 一个 guardian 增加成本；当前尚无退休回收，不能宣称长期资源占用有界。

## 验收

真实 Node 强杀重启、host 强杀重连、guardian 丢失后的局部阻塞已验证。
见[核心用例](../../../../test-cases/node/process/architecture/ownership-and-isolation.md)。
Strong、服务管理器下存续、物理断电及非 Linux 不在本次实现声明内。
