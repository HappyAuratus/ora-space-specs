---
status: superseded
---

# 本机控制与输出分流，使用有界 MessagePack 请求和查询恢复

> 已被取代：本决策由 [Node 直接执行自己的 Git](../../git-execution/0-node-runs-git-directly-with-attempt-locks.md) 取代，原因：host 退役后不再有 Node 与 host 之间的本机协议。当前系统承诺以取代者为准；取代者的实现合并之前，desktop 里仍在运行的 host／guardian 以本文为历史依据。

当前 Linux Node—host 使用控制和输出两个端点；host—guardian 保留 Control／Events／I/O
通道绑定，但尚不提供事件订阅。查询原身份及历史事实恢复管理，不以通知流作为唯一事实来源。
本文于 2026-09-18 收紧为已交付本机请求协议后标 implemented；Controller–Node IPC 不在本层。

## 决策

### D1：分离控制和输出，不承诺已实现三条业务流

host 的 host.sock 承担创建、启动、查询、停止和关闭，host-io.sock 承担输出读取。
guardian 三通道校验同一宿主绑定；Events 端点存在不等于有订阅、游标或事件回放。
可信本机使用 OS 对端身份和私有路径，无秘密令牌或 Controller 租约。

### D2：长度前缀与 MessagePack 明确边界

帧长、输出分块及连接任务有界。Node—host、host—guardian 分别定义消息及 wire 版本；
未知版本及不兼容输入在副作用前拒绝，不从可解码推断语义兼容。
当前不提供无限列表查询或分页 API，不将两套消息合并为全局业务协议。

### D3：查询区分意图、历史事实与当前协调状态

回复区分持久接受与实际完成。host 返回 last_observed 以及协调状态，断线时历史 Running
不当作当前存活证明。调用方按原 Scope／Run 查询，不能因失联分配新 Run 重做副作用。
输出通过偏移读取，错误和截断显式返回。

### D4：重连不恢复旧宿主资格

guardian 先持久保存新宿主绑定，旧通道和排队请求在执行点拒绝；同绑定重传幂等。
Node—host 仍是可信调用方模型，不将宿主代次包装成 Node 授权。

## 不变量

1. 错误版本或绑定不产生运行副作用。
2. 输出等待不阻塞控制，通道连接不等于业务成功。
3. 查询及重连不再次执行原尝试，历史投影不充当新鲜证据。

## 取舍与未来 ADR

轮询比订阅简单，代价是查询开销和延迟。快照—订阅原子衔接、游标与重同步、完成结果接管确认、
多观察者、stdin 流、Windows Named Pipe／macOS transport 由未来协议 ADR 定义和验收。
当前无回收，不因“读过结果”删除责任；Node 业务终态 EventAck 是另一层协议，不等同于
process 结果退休协议。

## 验收

协议边界测试、真实 host 错误版本／对端拒绝、回复丢失只执行一次、输出阻塞控制隔离、
guardian 三旧通道及迟到字节拒绝已有证据。见[核心用例](../../../../test-cases/node/process/protocol/streams-and-state-recovery.md)。
