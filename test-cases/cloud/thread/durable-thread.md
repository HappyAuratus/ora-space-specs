# 持久 Thread

依据[已批准的 Thread 根决策](../../../decisions/cloud/thread/0-durable-agent-thread-with-user-turns.md)。
本文件登记 Thread 条目的顺序、用户轮次与结束条件。证据预期来自 `ora-space/cloud` 的 `integration` 包与前端组件测试；
实现尚未开始，全部为 `Missing`。

## Thread Entries Are Gap Free And Taken Over Once

### 风险

Node 事件重放产生重复条目，或乱序接管把对话顺序排错；前端的分页游标出现空洞。

### 前置状态

运行处于 `running`，会话执行已登记。

### 触发

`TakeOverThreadEvents` 接管批次；同一批次重放；同序号不同内容；批次首序号出现缺口。

### 必须成立

- 条目 `seq` 在 Thread 内从 1 连续递增，来源包括 Node、用户与系统。
- 重放不产生新条目；同序号不同内容与缺口以 `CONFLICT` 拒绝，整批不写入。
- 条目内容与 Node 记录逐字节相同。

### 禁止结果

重复条目；`seq` 空洞或重排；部分写入的批次。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 连续与幂等 | 集成测试：两个批次、一个重放、一个用户轮次交错写入后，`seq` 为 1..n 且无重复 | Missing |
| 冲突整批拒绝 | 集成测试：批次中一个事件内容不同，响应 `CONFLICT`，条目数与收据数都不变 | Missing |

### 决策依据

Thread D1、D2，不变量 1、2、6。

## User Turns Are Persisted Before Delivery And Settle Exactly Once

### 风险

用户消息因 Controller 重启丢失、被执行两次，或会话结束后仍在新会话中执行。

### 前置状态

Thread 处于 `pending | active | idle`。

### 触发

带 `Idempotency-Key` 追加消息；同一键重试；Thread 进入 `ending` 后追加；会话结束时仍有排队轮次。

### 必须成立

- 条目与 `SubmitUserTurn` 命令在同一事务写入，`turn_id` 由 Cloud 生成。
- 同一 `Idempotency-Key` 返回同一条目；`ending | ended` 时返回 `409 thread_closed`。
- 接管到带该 `turn_id` 的用户消息记录时轮次变为 `delivered`；会话终态事务中仍 `queued` 的轮次变为 `discarded`。

### 禁止结果

命令先于条目存在；同一键产生两个轮次；`discarded` 的轮次之后被执行。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 同事务写入与幂等 | 集成测试：追加消息后条目与命令同时存在；同键重试返回原条目且命令数不变 | Missing |
| 关闭后拒绝 | 集成测试：`ending` 时追加返回 `thread_closed` | Missing |
| 轮次结算 | 集成测试：接管带 `turn_id` 的记录后为 `delivered`；会话终态后剩余轮次为 `discarded` | Missing |

### 决策依据

Thread D3，不变量 3、4。

## Idle Sessions End On Database Time

### 风险

依赖 Controller 或 Node 时钟判断空闲，时钟偏差导致会话过早结束或永不结束。

### 前置状态

Thread 为 `idle`，`idle_since` 已记录。

### 触发

数据库时间越过 `idle_since + thread_idle_timeout`；在此之前用户追加消息。

### 必须成立

- 越过窗口后派发循环把 Thread 置为 `ending` 并放出 `EndSession{idle_timeout}`，只放出一次。
- 窗口内追加消息使 Thread 回到 `active`，不结束。
- Agent 首轮失败后 Thread 同样进入 `idle`，不自动结束。

### 禁止结果

用应用进程时钟判断；重复放出结束命令。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 空闲期满结束 | 集成测试：调整 `idle_since` 使其早于窗口，派发循环运行两次只产生一个 `EndSession` | Missing |
| 追加消息重置 | 集成测试：`idle` 时追加消息后 `thread_state = active` 且无结束命令 | Missing |

### 决策依据

Thread D4，不变量 5。
