# 持久接受、事实汇合与责任退休

当前持久接受与事实投影已实现，完整存储故障矩阵仍有缺口。输出持久化及退休用例保留为
未来 ADR 输入，当前不回收记录。旧用例的未来要求不能用当前保留全部记录替代验收。

## Durable Acceptance And Fact Convergence Respect Separate Writers

### 风险

提交前确认、共写另一进程数据库或未持久记录待报告事实，会在崩溃后丢失已接受责任。

### 前置状态

宿主与 guardian 使用独立 SQLite，关键元数据 WAL＋FULL；存在批量接受和待汇合事实。

### 触发

在请求去重／意图／责任提交、运行事实／待报告记录提交及回复／确认处分别强杀并恢复。

### 必须成立

- 各自原子记录所拥有责任，批次提交后才能确认；正常运行通过幂等协议汇合，不直接写对方库或依赖跨库事务。
- 恢复重投汇合不丢失或伪造事实；独占恢复工具才可处理原库，后续清理不伪造原退出结果。

### 禁止结果

回复持久接受但重启找不到责任；重复报告产生冲突事实；恢复程序无独占资格写入。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 关键接受事务与确认顺序跨强杀保持 | `crates/process-runtime/tests/host_state.rs::committed_intent_survives_owner_kill_without_becoming_a_new_attempt` 验证宿主 WAL／FULL 意图强杀后保留；`apps/ora-process-guardian/tests/bootstrap.rs::guardian_commits_before_ready_and_rejects_unauthorized_probes` 验证 guardian 初始化记录晚于自身事务、早于 Ready；`launcher_kill_preserves_original_guardian_and_lost_ready_is_queryable` 验证原启动记录在启动方强杀后可查询。运行时与 `crates/db/src/sqlite_engine_tests.rs::linked_sqlite_includes_wal_reset_fix` 检查链接引擎。另有 apps/ora-process-guardian/tests/bootstrap/runs.rs::run_journal_failures_never_retry_an_accepted_exec 覆盖 Run 日志失败；完整提交强杀、存储故障仍缺，物理断电／结果接管由未来 ADR 验收 | `Partial` |
| 独立写权威与事实汇合幂等，恢复资格受约束 | apps/ora-process-host/tests/service.rs::host_kill_reconnect_stop_and_close_use_original_attempt 已覆盖原 Run 投影重连；完整报告／存储故障组合未齐 | `Partial` |

### 决策依据

[根决策](../../../../decisions/node/process/persistence/0-independent-journals-and-responsibility-retirement.md) D1、D2 与不变量 1。

## Published Output Offsets Never Lead Durable Data

范围：未来持久输出／stdin ADR。当前通用元数据存储失败不卸责仍是现有义务，完整磁盘满／权限故障证据 Missing。

### 风险

先提交可读偏移会暴露不存在的数据；磁盘故障若被当作成功将丢掉未完成责任。

### 前置状态

Guardian 写分段输出并保存已持久可读范围，输入存在不同确认阶段；可注入磁盘满、权限错误和记录损坏。

### 触发

在输出数据持久化与偏移提交之间强杀，在关键元数据／输出／输入记录处注入写入失败，随后恢复条件。

### 必须成立

- 先持久化可靠输出，再提交可读偏移，恢复正确处理未提交尾部；可丢弃日志不冒充可靠保留。
- 失败阻止不可靠接受和确认，保留责任并暴露阻塞；输入保持真实耐久与交付未知边界，条件恢复后继续协调。

### 禁止结果

偏移超出可靠数据；失败自动确认结果或释放资源；清理损坏库后抹除原责任。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 数据／偏移顺序和未提交尾部在崩溃后可核实 | 尚无直接证据；需真实分段文件与元数据提交窗口测试 | `Missing` |
| 磁盘、权限和损坏故障保留责任及未知状态，恢复后推进 | 尚无直接证据；需受控存储失败与生产查询验证 | `Missing` |

### 决策依据

[根决策](../../../../decisions/node/process/persistence/0-independent-journals-and-responsibility-retirement.md) D2、D4 与不变量 2、3。

## Retirement Cannot Revive Old Scope Or Run Requests

范围：未来结果接管／退休 ADR。当前无 GC，以下回收条件不表示已有退休接口。

### 风险

按 TTL 删除详情、去重或退休记录会让旧请求重新执行，或回收仍有结果／资源责任的数据。

### 前置状态

多个 scope 分别处于未收尾、未接管结果、资源未解除关联及全部责任完成状态；留有迟到 Run 请求。

### 触发

请求退休、清理输出与详情、压缩拒绝依据，在重启后投递旧身份及未知 scope 的 Run 请求。

### 必须成立

- 收尾、结果责任与资源关联均处理后才能退休；guardian 事实须已汇合且恢复不再依赖。
- 输出轮转可见缺口，未接管完成记录须接管或显式承担放弃责任后才回收；诊断观察不替代接管。
- 详情回收后仍拒绝旧请求；未知 scope 不隐式创建，ScopeId 永不复用，压缩需要覆盖整个旧身份范围的拒绝依据。

### 禁止结果

单靠 TTL 删除最后拒绝依据；回收尚有责任的数据；旧 Run 重传创建新 scope 或重复运行。

### 验证义务与证据

| 验证义务 | 验证方式与直接证据 | 状态 |
|---|---|---|
| 每类责任未完成均阻止对应数据回收，完成后可退休 | 尚无直接证据；需结果／资源确认与恢复接口测试 | `Missing` |
| 详情回收和依据压缩后旧请求仍拒绝，未知身份不自动创建 | 尚无直接证据；需持久退休、重启与迟到请求测试 | `Missing` |

### 决策依据

[根决策](../../../../decisions/node/process/persistence/0-independent-journals-and-responsibility-retirement.md) D3、D4 与不变量 3。
