---
status: proposed
---

# 插件日志：由 Host 按 Plugin ID 独立持久化，而非混入 Ora runtime log

Ora 将插件自身的诊断信息从 Ora runtime log 分离：插件 stdout 继续专用于 Ora Plugin Protocol，Plugin SDK 经 stderr 发送带级别的日志，Host 读取、校验并按 canonical `PluginId` 写入独立 JSONL 日志。Host 自身的插件生命周期、协议及日志设施故障仍写入 Ora runtime log。日志写入不得阻塞插件进程；容量或 I/O 故障造成的丢失必须计数并报告。

本文为 `proposed`，供评审日志归属、数据目录、传输格式、过滤和失败行为；尚未定义已批准的实现契约或核心测试用例。它涉及 `ora-plugin-runtime`、`ora-plugin-lifecycle` 的 storage 与卸载边界、Plugin SDK 及 Host 日志设施。插件此前没有独立日志，本决策不迁移既有 Ora runtime log、不保留旧 SDK stderr 前缀的级别语义，也不迁移插件可能自行存入 `logs/` 的数据。旧插件仍可运行，其 stderr 按 raw 规则处理；Plugin Protocol 帧格式、已安装包目录和 `ora/childprocess/*` 的返回语义不变。日志查看、导出和远端传输不由本决策交付。

## 问题与约束

当前 SDK 在插件运行时把 `console.debug/info/log/warn/error` 以 `[plugin:<level>]` 前缀写到 stderr；`ora-plugin-runtime` 持续读取 stderr，却将每个读取块统一作为 `INFO` 的 `plugin stderr` 事件写入 Ora 主日志。它保证 pipe 被 drain，但级别在 Host 边界丢失，读取块也不等于一条日志。插件诊断与 Host 事实共用日志文件，无法按插件独立管理。

stdout 已承担 Plugin Protocol，不能用作普通日志通道；第三方库及原生代码仍可能直接写入非结构化 stderr。Host 必须在不信任插件内容、不阻塞协议和不要求插件拥有日志文件权限的前提下保留这些诊断。现有 `ora/storage/*` 将 `<data-dir>/plugins/data/<namespace>/<name>/` 映射为插件的逻辑目录，目前只排除 `web-profile/`，因此在该目录下建立 Host 专用 `logs/` 还需要同步修改 storage 边界。

## 决策

### D1：Host 与 Plugin 拥有各自的日志域，stdout 仍只传协议

Ora runtime log 记录 Host 负责的发现、安装、启动、退出、协议和生命周期故障、Host policy 以及 plugin-log sink 故障。Plugin log 记录插件代码和它主动转发的依赖诊断。正常插件诊断只进入 Plugin log，不再默认复制进 Ora runtime log；日志设施的故障则只作为 Host 事实记录在 Ora runtime log。

插件 stdout 只传 Ora Plugin Protocol 帧。SDK logger 和 `console.*` 经 stderr 传输；stderr 是诊断 transport，不意味着 `ERROR`。插件直接写入的 stderr 也被接收，不能要求所有生产者先使用 SDK。这样分离 ownership，同时保留第三方原始输出的诊断能力。

### D2：日志按 canonical Plugin ID 归属，`logs/` 是 Host 保留子树

同一 `<namespace>/<name>` 的插件进程代与安装版本共用 `<data-dir>/plugins/data/<namespace>/<name>/logs/`；活动文件称 `plugin.log`。Plugin ID 由 Host 从已安装身份确定，不读取插件提供的路径或身份。升级后日志继续归属同一身份；卸载选择保留数据时保留日志，选择删除数据时一并删除日志。该目录不属于安装包，也不授予插件额外文件系统权限。

`logs/` 虽位于插件数据目录，却由 Host 独占。`ora/storage/list` 在数据根目录隐藏 `logs`；`list/read/write/remove` 对首段为 `logs` 的路径均拒绝，包括 `logs` 本身和其子孙路径。不能只在列表隐藏：已知路径的直接读写及整目录删除同样必须受保护。日志查看或导出由未来的 Host 入口决定，不借用插件自己的 storage 权限。

`logs/` 从本决策生效起直接成为保留名称，不迁移旧内容。若同名路径已存在且 Host 无法确认它是自己创建的普通目录或文件，sink 初始化失败；Host 不覆盖、不删除该路径，仍按 D7 drain stderr 并报告故障。storage 仍拒绝插件访问该名称，因而此前自行写入 `logs/` 的插件数据将不再通过 storage 可见；这是明确的兼容性边界，而非一次隐式迁移。Host 创建或复用日志路径时不得跟随 symlink/reparse-point，也不能把插件可写的目录内标记当作可信 ownership 证明。

### D3：复用 JSONL event shape，但可信字段只能由 Host 填写

每条持久化记录是一个 JSON object，占一行。稳定顶层字段为 `timestamp`、`level`、`target`、`message`；`method`、`context`、`error` 按需出现。Host 在接收时生成时间戳，并最后写入 `context.plugin_id`，以当前进程绑定的 canonical Plugin ID 覆盖任何同名插件输入。插件可以提供 `target`、`message`、`method`、一般 `context` 及错误详情，但不能决定可信的 Host metadata。缺少 `target` 的结构化记录使用稳定的插件默认 target；raw stderr 使用 `plugin.stderr`。

`request_id`、`trace_id`、`span` 和 `generation` 不接受插件自报。前面三个字段只在未来建立 Host ↔ Plugin correlation contract 后才由 Host 填写；Host 可以附加自己掌握的进程代标识以区分同一 Plugin ID 的先后运行，插件不能伪造该身份。即使插件 payload 含这些保留键，也不能将它们提升为可信顶层字段。日志内容本身仍是不可信的插件陈述；统一 JSONL shape 只复用解析与展示约定，不把插件诊断变成 Host 事实。

Host 对单条记录及字段大小设上限，避免一个 payload 绕过 stderr buffer 上限造成内存或磁盘无限增长。日志可能包含插件主动写出的敏感内容；这项格式决策不承诺对任意第三方输出自动脱敏，未来导出必须沿用 Host 的私有日志访问边界。

### D4：SDK 提供结构化 logger，并将 `console.*` 映射到同一传输

Plugin SDK 在 SDK-owned plugin object 上提供 `plugin.logger`，支持 `TRACE`、`DEBUG`、`INFO`、`WARN`、`ERROR`，可附加 `target`、`method`、JSON `context` 和 `error`，并可派生带默认 target/context 的 child logger。API 不暴露 Host 文件路径、Plugin ID 或可信 correlation 字段。错误对象、不可序列化值及循环引用必须退化为有界、可写入的描述；记录日志本身不能使插件业务方法抛错或污染 stdout。

新 SDK 中的 `console.*` 映射为：

| SDK 调用 | Plugin level |
| --- | --- |
| `console.debug` | `DEBUG` |
| `console.info`、`console.log` | `INFO` |
| `console.warn` | `WARN` |
| `console.error` | `ERROR` |

`console.*` 由同一 logger 发送，使多行文本作为单条结构化消息编码。SDK 对标准 `console.*` 的 stdout 保护只在插件运行入口安装后生效；插件在 SDK 初始化前直接写 stdout 的既有协议风险不由日志 sink 修复。

### D5：stderr 是混合字节流，按版本化 envelope 与 raw fallback 识别

SDK 的结构化记录以一条带换行的 `@ora/plugin-log/v1 ` 前缀加紧凑 JSON object 发送。v1 payload 包含必需的 `level` 与 `message`，可选 `target`、`method`、`context`、`error`；字段类型、大小和级别须校验。SDK 序列化消息内的换行，使一条日志仍占一个 stderr logical record。前缀仅用于识别格式，不是授权凭据；Host 仍按 D3 重写可信 metadata。

Host 对每个 stderr logical record 尝试解析 v1；其它内容一律以 `INFO`、`plugin.stderr` 和原始文本进入 raw fallback，包括旧 SDK 的 `[plugin:<level>]` 文本。Host 不解析该旧前缀，也不承诺保留其原级别。未知版本、格式错误或校验失败的 envelope 同样退化为 raw，不终止插件、不因解析失败丢掉内容；可以另记有界的 Host 格式故障摘要，不能把原始敏感 payload 复制进 Ora 主日志。stderr EOF 时最后一个未换行 fragment 同样处理。

解析按字节增量进行，不能把一次 pipe read 当成一条记录。buffer 与持久化记录均有上限；超长输入拆为有标识和顺序的有界 raw 片段，保留内容而不无限增长内存。无效 UTF-8 必须使用可追溯的有界编码表达，不能在不可逆替换后声称保留了原字节。

### D6：过滤在识别级别之后执行，首版默认 `INFO`

Host 先按 D5 得到记录级别，再应用独立的 Plugin log level 策略，最后交给持久化队列。首版所有插件共用默认 `INFO` 阈值，允许 `INFO`、`WARN`、`ERROR`，过滤 `TRACE` 和 `DEBUG`；该阈值不随 Ora runtime log 的进程级过滤器变化。未来的动态配置和 per-plugin level 可另作决策，但不能改变首版默认值的含义。

raw 与无法识别的 envelope 归为 `INFO`，所以若未来显式设为 `WARN`，这类内容会被策略过滤；这属于有意过滤，不记作 transport 或 sink 丢失。过滤不能提前作用于 stderr 字节，否则结构化记录中的真实 `ERROR` 又会被压成默认 `INFO`。

### D7：队列满或 sink 故障时丢弃新记录、报告数量、继续 drain

stderr reader 不能等待磁盘，也不能等待满队列腾出空间。通过过滤的记录进入容量有限的写入队列；队列满时丢弃**新到达**的记录，按 Plugin ID 与进程代累计数量，继续读取 stderr。首版不按 severity 插队，即拥塞时后来的 `ERROR` 也可能丢失。这保留先入队记录的顺序，且避免日志压力阻塞插件 pipe 或 Plugin Protocol。

sink 初始化或写入失败时，当前进程代的 sink 进入不可用状态，后续 stderr 仍被读取、解析和按过滤策略处理，但待写记录计入丢失；本代不无限重试或积累无界内存，下一个进程代可以重新尝试创建 sink。队列满首次发生、sink 首次故障时 Host 各记一次 `WARN`，恢复或进程代结束时报告相应丢弃数量；不得每条丢失记录都向 Ora 主日志写一次，也不得把插件原文复制过去。已经交给 sink、但 I/O 错误使持久化结果无法判定的记录应报告为结果不确定，不虚报精确的已丢失或已持久化数量。

因此 D5 的“raw 不静默丢弃”保证的是**识别阶段有 fallback**，不是磁盘故障时的无损承诺。正常级别过滤、队列拒收、sink 故障和结果不确定须分别说明。Ora runtime log 自身也可能因同一磁盘故障无法落盘；故障计数至少在本次 Host 运行期可查询，主日志告警是尽力报告，不能宣称故障时仍有绝对持久化证据。日志故障不改变插件的协议状态，不单独杀死插件进程。

### D8：停止先收完 stderr 并释放 sink，卸载再处理数据目录

停止或替换一个插件进程代时，Host 等待进程退出，随后在**有界的整个日志收尾期限内**读取其 stderr 至 EOF，处理末尾 fragment，关闭该代入队入口，完成队列处理、flush 与文件句柄释放。此前“进程已退出”不能等同“日志已完成”；被其它进程继承的 stderr 写端也可能使 EOF 永不出现。读 EOF 或 flush 失败、超时都属于 Host 日志故障；Host 结束该代 reader 与写入入口，报告已知未提交数量及无法确定的剩余内容，不把它改判为 Plugin Protocol 失败，也不无限延长停止。

卸载选择删除数据时，必须在该 Plugin ID 的日志 writer 已释放目录内文件句柄后再移动或删除数据目录。若句柄无法在有界时间内释放，删除数据的卸载应报告清理失败，保留或回滚数据，而不能报告成功却在后台继续写入被删除或已移动的目录；对仍被占用的文件不得强行丢弃 rollback 责任。保留数据的卸载保留现有日志并在不再写入时关闭 sink。该顺序也覆盖异常退出、启动超时和下一进程代启动，避免旧 reader 在新代开始后继续写入同一活动文件而没有归属标识。

### D9：Host-managed child process 的输出由插件选择是否记录

`ora/childprocess/stdout` 和 `ora/childprocess/stderr` 继续把原始流返回给发起进程的插件；Host 不自动将这些流再存为 Plugin log。插件可选择不记录、以哪种级别记录或提取哪些结构化字段。Host 不掌握第三方 CLI 的 stderr severity，自动写入既可能重复，也可能把敏感输出持久化两遍；该选择不影响插件自己的 stderr 始终由 D5 读取。

## 为什么不是这些替代方案

| 方案 | 更好的维度及最终取舍 |
| --- | --- |
| 继续把所有 stderr 写入 `ora.log` | 无须新增 sink，但 Host/Plugin ownership 和 severity 无法分开，仍不能单插件保留与诊断。 |
| 直接把 stderr 当 `ERROR` | 无须解码级别，但 stdout 已被协议占用，正常 `INFO` 也经 stderr，因而会制造误报。 |
| 任意 JSON stderr 直接当结构化日志 | 省去 sentinel，但第三方库输出的 JSON 会被误认为 SDK 记录；版本化 envelope 保留可解释的 raw fallback。 |
| 要求所有 stderr 只输出结构化 envelope | 消除 raw 歧义，但第三方依赖和原生组件可能直接写 stderr，其诊断会丢失。 |
| 让插件自行写文件或写在 versioned 安装包中 | 插件有更直接的控制权，但需要额外文件权限、暴露路径或随升级清除文件，也无法统一 Host 的保留策略。 |
| Host 自动记录 child-process 输出 | 不依赖插件主动转发，但插件本就收到该流，会形成双重记录，Host 也无法决定内容和级别。 |
| 队列满时等待或无限扩大队列 | 可以保留更多日志，但分别会让 stderr pipe 阻塞插件，或让内存随不可信输入无界增长。 |
| 队列满时淘汰旧记录或优先保留 `ERROR` | 更易留下故障末尾与高严重度内容，但打破已排队顺序，需定义跨级别驱逐及统计；首版选择简单可预测的丢弃新记录。 |

## 为什么不能直接沿用已有卸载顺序

当前卸载在进程停止后按同一卷 staging 移动安装包及可选插件数据。引入日志 writer 后，进程退出不再证明数据目录可移动；直接沿用旧顺序可能在 Windows 遇到打开句柄、在其他平台留下 writer 继续写入已移动目录。D8 要求把 stderr 收尾与句柄释放放在 staging 之前；失败时由现有卸载错误和回滚边界承担，不把部分删除包装成成功。

## 不变量

1. 插件 stdout 只承载 Ora Plugin Protocol；stderr 传输所有级别的诊断。
2. Host 事实进入 Ora runtime log，插件诊断默认只进入该 Plugin ID 的 Plugin log。
3. `context.plugin_id`、接收时间和进程代归属由 Host 确定，插件内容不能覆盖可信 metadata。
4. `logs/` 对插件的 `ora/storage/*` 列表与四种操作均不可见、不可访问。
5. Host 不覆盖来源不明的同名日志路径；此类冲突使 sink 初始化失败而不影响 stderr drain。
6. 每个 stderr logical record 经结构化或 raw 路径处理；尾部、超长及无效编码不因解析失败静默消失。
7. 插件日志按独立的默认 `INFO` 策略在识别级别后过滤。
8. stderr reader 不等待日志队列或磁盘；满队列丢弃新记录并计数。
9. 容量拒收、sink 故障及持久化结果不确定均按进程代报告，不转为插件协议错误。
10. 删除插件数据前，该 Plugin ID 的旧进程代 stderr 已收尾且相关日志文件句柄已释放。
11. Host-managed child process 的原始输出不由 Host 自动重复持久化。

## 风险与本决策未解决的问题

- 丢弃新记录可能遗漏拥塞后的 `ERROR`；该代价由不阻塞插件及有界内存约束决定，须在诊断中暴露丢弃数量。磁盘同时使 Ora 主日志失败时，持久化告警也不可靠，不能把 Host 告警当成端到端无损证明。
- 插件、依赖和它启动的外部组件可能主动输出秘密。日志只存于 Host 私有数据目录；脱敏规则与用户查看/导出时的披露边界须在相应功能中另行决定。
- 旧 SDK 的 `[plugin:<level>]` 文本按 raw `INFO` 保存；不提供旧级别的兼容解析，也不迁移此前写入 Ora runtime log 的插件诊断。
- rotation 文件名、阈值、压缩与默认保留窗口只影响资源策略，不改变本文的身份和失败边界，留给后续 ADR；队列容量、单条最大字节数和停止 flush 超时也留给有界实现策略，但不得改变满队列与超时后的行为。
- 动态及 per-plugin level、live tail、查看 UI、support bundle/export、Host ↔ Plugin correlation、远端 sink 不在本次日志落盘闭环内，须分别确定权限和契约。

## 落地顺序与验收

1. **建立目录所有权。** 按 Plugin ID 创建 Host 日志目录，并使 storage 的 root list 与四种操作一致地拒绝 `logs/`。可独立验收同名路径冲突不会被覆盖、sink 失败仍持续 drain，以及 symlink/reparse-point 不越过 Host 日志根；不进行旧数据迁移。
2. **建立传输与格式。** SDK 新 logger、`console.*` 映射、增量 stderr decoder、structured/raw fallback 和 JSONL Host metadata 同时交付。可独立验收分块与换行、超长/无效编码、保留字段覆盖，以及旧前缀按 raw `INFO` 处理；stdout 协议与 child-process 输出语义保持不变。
3. **建立持久化与故障隔离。** 独立默认 `INFO` 过滤、有界队列、丢弃新记录、sink 故障分类和限次 Host 告警组成闭环。可独立验收大量 stderr、队列满或不可写目录不会阻塞插件，日志损失和结果不确定能够区分。从此阶段起，新产生的插件诊断只进入独立 Plugin log，不再写入 Ora 主日志。
4. **闭合生命周期。** stop、异常退出、重启与 retain/delete uninstall 在 EOF、flush、关闭句柄之后完成各自收尾；可独立验收尾部记录与 Windows 打开文件句柄导致的清理失败被如实报告且数据可回滚。只有 ADR 评审通过并标记 `approved` 后，才在镜像 `test-cases` 目录登记上述行为的核心用例与直接证据。
