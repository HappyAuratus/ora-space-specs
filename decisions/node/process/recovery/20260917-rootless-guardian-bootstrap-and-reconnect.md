---
status: implemented
---

# 无特权 Guardian 交接原锁，宿主重连原实例而不补启旧 Scope

当前 Linux rootless 路线先提交 Scope 创建责任，再把已持有的稳定锁交给独立 guardian。
回复丢失后查询原实例，旧 Scope 不启动替代 guardian。本文于 2026-09-18 同步可信本机、
无秘密令牌的最终实现并标 implemented；不包含 Strong、Controller 授权或 guardian 死亡后收养。

## 继承与修改

| 前序约定 | 当前规则 |
|---|---|
| 恢复资格不等于业务静止 | 继承；锁空闲不能授权资源放行 |
| guardian 独立于启动方存续 | 继承；独立 session 和原锁交接 |
| 原 approved 要求秘密管理凭据 | 修改；Eric 已选择可信本机、移除令牌，保留 UID／私有路径及持久宿主绑定 |
| guardian 死亡后封闭而不接管 | 继承；身份证据不足时清理未知，不按旧 PID 补杀 |
| 全平台、服务管理器与机器重启验收 | 明确交由未来部署／平台 ADR，不是本次 Linux 普通进程边界 |

继承[恢复根决策](0-single-authority-and-failure-recovery.md)，控制、协议和存储分别由同目录主题 ADR 约束。

## 决策

### D1：显式状态目录，拒绝覆盖旧文件

host 接收专用绝对 state_dir，不读取 HOME 或业务 cwd；guardian 接收 Scope 定位及已打开资源。
host.lock／host.sqlite 与 scopes 下每个 Scope 的 guardian.lock／guardian.sqlite 分别保留。
管理 socket 与 output 目录属于已知布局；output 目录不表示持久输出已实现。
拒绝符号链接、错误权限／身份、未知布局及不支持的文件系统；不清空目录或自动换路径。
稳定锁不随数据库迁移或普通维护替换。

### D2：持久登记之后交接锁，Ready 不等于 Run 成功

host 提交原 Scope／guardian 创建意图与启动责任，持锁通过专用 bootstrap 通道传递身份及描述符。
guardian 检查继承锁 inode 与原目录绑定，提交初始化 journal、建立端点后报告 Ready。
不先释放再让 child 竞争；不因 Ready 丢失重复执行 guardian 启动。
工作负载 exec 前关闭非预期管理描述符；guardian 不依赖父进程 Drop。
Run 接受另有持久 journal，业务执行前记录尝试，不开放绕过日志的 spawn。

### D3：绑定原 guardian，实际执行点拒绝旧宿主

OS 对端 UID 和私有路径限定可信部署；Scope／guardian 身份防止错接，不是秘密凭据。
host 在原稳定锁下持久推进代次；guardian 提交新的宿主实例／代次绑定后才确认。
同绑定重传幂等，低代次和同代次不同实例拒绝；三旧通道及排队变更在实际执行点检查绑定。
此机制不是 Controller 的 Node 授权或租约。

### D4：失联不补启，失效不伪造清理

端点不可达或超时继续核实原实例，不自行删除端点、重启 guardian 或原 Run。
guardian 强杀丢失 pidfd／回收权后，磁盘 PID／start time 不自动成为新的信号权威。
原 Scope 不由替代 guardian 开放，host 可保留历史事实和 Unavailable；这不是已观测 Closed。
缺证据时保留未知，不能报告清理完成或恢复冲突业务。

### D5：明确现有组合与未来边界

真实 host 已提供协调和查询；独立 Node 已接入 Git。旧 Backend／插件保持原入口。
未来 ADR 负责服务管理器部署、机器重启身份恢复、非 Linux、guardian 死亡后的独立清理、
凭据与租约、退休回收；本次不授权自动安装服务或提权。

## 不变量

1. 原锁交接不制造独占空窗，回复丢失不创建第二个 guardian。
2. 原身份、启动未知和关闭方向不因重连重置。
3. Ready、管理绑定、业务结果及清理证据互不替代。
4. guardian 身份证据丢失不转换为旧数字 PID 信号权限。

## 取舍与验收

连接失败就重启更容易恢复表面可用性，却可能重复业务；根据旧 PID 补杀可能误伤。
选择保留未知，代价是人工处置前相关资源可能持续阻塞。
真实 app 的锁交接、启动方强杀、Ready 丢失、持久接管、旧通道失效、Run 去重和 guardian
死亡缺口已有直接证据，见[核心用例](../../../../test-cases/node/process/recovery/authority-and-recovery.md)。
测试不声称覆盖任意机器断电时序或同 UID 恶意代码。
