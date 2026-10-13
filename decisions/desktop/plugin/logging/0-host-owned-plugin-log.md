---
status: implemented
---

# 插件日志：由 Host 按 Plugin ID 独立持久化

插件诊断由 Host 从 stderr 解码，按 canonical `PluginId` 写入独立 JSONL 日志；stdout 保留给
Plugin Protocol。Host 的生命周期、协议和日志设施故障进入 Ora runtime log。日志压力或磁盘故障
允许损失诊断，但不能阻塞插件协议，也不能把不确定的写入宣称为成功。

本决策已实现于 `ora-plugin-runtime`、`ora-plugin-lifecycle` 和 Plugin SDK。核心证据见
[插件日志用例](../../../../test-cases/desktop/plugin/logging/host-owned-plugin-log.md)，其中跨 Host
竞争、时钟回拨等尚未直接验证的边界仍保留 `Partial` / `Missing`。本决策不迁移历史 Ora 日志，
不承诺任意第三方输出自动脱敏，也不提供 Plugin Protocol 日志读取接口。

## 决策

### D1：Host 与 Plugin 拥有各自的日志域

插件的结构化日志、`console.*` 和原始 stderr 默认只进入 Plugin log。Host 只在自身日志中报告
生命周期事实及有界的日志设施故障摘要，不复制插件 payload。stderr 是诊断通道，不代表 ERROR。

### D2：日志位于独立 Host 管理根目录，活动文件只有一个 writer

活动文件为 `<home_directory>/plugins/logs/<namespace>/<name>/plugin.log`。升级和进程换代共用
该 Plugin ID 的目录；路径由安装身份确定。storage API 的根在 `plugins/data/`，不能到达同级
logs tree。该 API 隔离不等同于对具有直接文件或子进程权限的 Agent 的操作系统级隔离。

logs root、各级子目录、活动文件必须是预期的普通对象；链接、reparse-point、类型冲突或无法
验证的路径使 sink 不可用，原对象保持原样。Host 取得独立的排他 writer lock 后才能打开活动文件。
writer 未真正释放时，新代仍可运行，但不能同时写同一文件；删除卸载也不能通过释放检查。

### D3：JSONL 中的可信身份与接收时间由 Host 填写

记录保留 `timestamp`、`level`、`target`、`message`，以及可选 `method`、`context`、`error`。
Host 清除保留 context 键，再写入 `plugin_id`、每次 Host 启动新建的 `host_session_id` 及本次
启动的 `generation`。插件自报的 correlation 与编码标记不能变成可信 metadata。

接收时间使用 Ora 本地时钟、带偏移的 RFC 3339。成功入队的记录按接收顺序写入，不能用墙上时间
证明因果顺序。插件提供的级别、消息和普通字段始终是插件陈述，不证明 Host 调用或授权发生。

### D4：SDK logger 和五个 console 方法共用 stderr 传输

`plugin.logger` 支持 TRACE、DEBUG、INFO、WARN、ERROR 和 child logger；不暴露 Host 文件句柄
或物理路径。`console.debug` 对应 DEBUG，`console.info/log` 对应 INFO，`console.warn/error`
分别对应 WARN/ERROR。标准运行入口在首个协议帧前安装接管，成功后不在 stop 或回调结束时恢复。
接管失败则不进入协议运行。

SDK 对复杂值、错误对象、循环引用和 sink 故障作有界退化，不向业务回调抛出日志异常；写入完整
带换行 envelope，处理短写。静态 import 求值、提前缓存的旧方法、独立 worker/子进程、其它
console 方法及直接 stdout 写入不属于五个方法的接管保证。

### D5：版本化 envelope 与 raw fallback 共存

结构化记录使用 `@ora/plugin-log/v1 ` 前缀及紧凑 JSON。Host 校验字段、级别、大小与嵌套深度，
前缀只识别格式，不赋予信任。无法识别、未知版本或无效 envelope 按 INFO / `plugin.stderr`
处理；旧 SDK 前缀不恢复原级别。

reader 按字节增量组帧，处理 EOF 尾部。超长内容分片，无效 UTF-8 以可追溯编码保留；不把一次
pipe read 当成一条日志，也不把截断 envelope 当完整结构化事件。解码缓冲和持久化记录均有界。

### D6：每个 Plugin ID 的持久级别是落盘阈值

默认 INFO，设置由 Host 配置存储拥有，与插件 data tree 和 Ora runtime log level 独立。
Host 先解码，再按过滤时的有效阈值决定是否入队，记录保留原级别。设置持久化成功后发布给运行中
的进程代，不要求重启；已入队记录不追溯删除。级别控制不减少插件实际写向 stderr 的字节。

设置与卸载由同一 Plugin ID 的操作锁串行协调；写入失败不报告成功。升级、停用和保留数据卸载
保留设置；删除数据卸载清除设置，清除失败回滚安装包、数据与日志树。

### D7：队列和 sink 故障隔离于插件协议

队列同时按条数和总字节数限流。reader 不等待磁盘或队列空间；满时拒收新记录并计数，包括后来
的 ERROR，不改变先入队顺序。sink 初始化或写入失败后本代不反复重试，仍持续 drain。

策略过滤、队列拒收、确定未写和 I/O 结果不确定分别说明。首次容量/设施故障作限次 Host 告警，
收尾报告计数；部分写入或 flush 失败不能虚报精确已落盘数。告警本身也是尽力落盘，不构成磁盘
故障下的无损保证。日志故障不改变 Plugin Protocol 状态。

### D8：日志收尾共用截止时间，卸载先确认 writer 释放

进程退出后，stderr EOF、队列 drain、flush 共用一个截止时间，reader 只消耗其中一部分。超时
撤销 reader 入队能力，分别报告未知 pipe 残余、尚未提交记录和不确定写入。blocking writer
不能因异步等待取消而被认为已结束：它继续持有文件及排他权，直至实际退出。

删除数据卸载先确认 writer 释放，再以 staging/回滚处理安装包、data tree、日志树和级别设置。
未释放 writer 或清理失败时不能报告成功；保留数据卸载保留日志。日志收尾不以诊断清理为由额外
终止进程树，也不无限延长停止。

### D9：Host-managed child 输出由插件选择是否记录

`ora/childprocess/stdout` 和 `ora/childprocess/stderr` 将原始流返回发起插件。Host 不自动再存
一份；只有插件主动转发到 logger 的内容才按其选定级别进入 Plugin log。

### D10：残缺尾部保留原文件，后续记录从新活动文件开始

取得 writer lock 后有界检查活动文件尾部。空文件和换行结尾文件继续 append；未换行文件使用
不覆盖的唯一恢复名称保留，再新建活动文件。恢复失败使 sink 不可用，不截断或覆盖历史。
恢复文件属于同一日志树，随卸载 retain/delete。flush 不承诺掉电持久化。

## 与 approved 文本的同步

实际 writer 是持有排他文件锁的 blocking task；超时后继续由该 writer 保有资源，而不是强行
中止磁盘 I/O。收尾结果显式区分 EOF、writer 释放和排队数量，以满足有界 stop 与确定所有权。
原文本中以“现有实现没有……”叙述的约束已由上述实际规则取代；既有测试缺口不作为未实现功能。

## 不变量

1. stdout 只传协议，stderr 承载所有级别诊断。
2. Host 事实与插件诊断分属 Ora runtime log 和 Plugin log。
3. 插件不能覆盖 Host 生成的身份与接收时间。
4. 日志归属独立 logs tree，storage API 不能到达它。
5. SDK 只写日志，不暴露物理文件；该承诺不扩大为 OS 文件隔离。
6. 解码失败有 raw fallback，尾部、超长和无效字节可追溯。
7. 独立持久阈值默认 INFO，过滤不改写 severity。
8. reader 不等待日志队列或磁盘，满队列拒收新记录。
9. 确定损失与不确定 I/O 分开报告，不变成插件协议错误。
10. 删除日志树前必须确认 writer 释放。
11. child-process 原始输出不被 Host 自动重复持久化。
12. 同一活动文件只有一个 writer，旧 writer 未释放时新代 sink 不可用。
13. 每条记录包含 Host session、进程代和本地偏移接收时间。
14. 收尾共用截止时间，未知剩余量不能被报告为零。
15. 重开文件不能拼接残缺尾部或覆盖历史。

## 后果与边界

接受拥塞时丢弃新记录，以保住协议活性和内存上限。日志可能含插件主动写出的敏感内容，读取、
导出、远端 sink、rotation/累计磁盘额度和更强进程隔离需各自后续决策；本决策只保证单条和
内存有界。验收边界与直接证据持续维护于核心用例，不把单一 writer 测试等同于跨 Host 故障演练。
