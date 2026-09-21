# Host 管理的插件日志

决策依据：[插件日志 ADR](../../../../decisions/desktop/plugin/logging/0-host-owned-plugin-log.md)。
ADR 已批准，本文记录长期验证义务；不表示实现已完成。证据来自现有测试源码检查，本次文档变更未运行测试。
`Partial` 表示已有直接但不完整的证据，`Missing` 表示尚无对应直接证据，不能以相邻测试替代。
下文 Rust 测试名称相对于所列源码文件；源码路径相对于代码仓库根目录。

## Log Ownership Must Survive Path Conflicts and Competing Writers

- **风险：** 新目录覆盖用户文件、路径链接越界，或多个 writer 交错写入同一文件。
- **前置状态：** logs root、namespace、插件目录或活动文件存在类型冲突或链接；另一个进程代或 Host 已持有活动文件。
- **触发：** 创建 sink 或在同一 Ora home 启动另一 writer。
- **必须成立：** 仅在安全验证路径且取得排他写入权后追加；冲突对象原样保留，sink 不可用仍持续 drain、报告丢失，协议继续运行；storage API 无法访问 logs tree。
- **禁止结果：** 覆盖或删除冲突文件，跟随链接写出根目录，两个 writer 同时写同一活动文件，因 sink 不可用终止插件。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 路径类型冲突不改写原对象且不阻塞 drain | `crates/plugin-runtime/src/plugin_log/pipeline_tests.rs`：`a_conflicting_log_path_fails_the_sink_but_keeps_draining` | Covered |
| logs root 与所有子级链接/reparse-point 均被拒绝 | `crates/plugin-runtime/src/plugin_log/sink.rs`：`symlinks_at_any_level_are_refused`；仍需根目录及 Windows reparse-point 证据 | Partial |
| 跨进程代、跨 Host 排他写入及失败隔离 | 尚无直接竞争测试 | Missing |
| 通过生产 storage 接口验证日志树不可达 | 尚未登记对应直接证据 | Missing |

- **决策依据：** D2、D7、D8，不变量 4、5、12。

## Persisted Metadata Must Distinguish Host Facts From Plugin Claims

- **风险：** 插件伪造身份、关联或分片标记；Host 重启后进程代计数重复，日志归属混淆。
- **前置状态：** 插件 payload 包含伪造的保留字段，Host 使用非 UTC 时区；同一插件跨 Host session 启动。
- **触发：** 接收结构化或 raw 记录并落盘。
- **必须成立：** Host 覆盖身份、清除未授权 correlation 与诊断标记，写入 session/generation；接收时间为带本地偏移的 RFC 3339；普通字段始终是插件陈述，墙上时钟回拨不改变队列顺序。
- **禁止结果：** 接受插件自报的可信 metadata，跨 session 混淆进程代，把 `method`/`target` 当作调用或授权事实。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 身份覆盖、correlation 保留键清除及时间序列化 | `crates/plugin-runtime/src/plugin_log/record.rs`：`structured_records_carry_host_identity_and_never_promote_reserved_keys`；尚不覆盖新增 session 与全部诊断标记 | Partial |
| Host 重启后 session/generation 不混淆 | 尚无直接证据 | Missing |
| 实际 pipeline 使用本地接收时钟，回拨仍按接收顺序写入 | 尚无注入回拨时钟的生产路径证据 | Missing |

- **决策依据：** D3，不变量 3、13。

## Console Redirection Must Remain Installed After Initialization Failure

- **风险：** 初始化或退出路径恢复 console，导致普通诊断污染 stdout 协议。
- **前置状态：** 标准 SDK 入口执行前后均有 console 调用，第三方依赖调用全局方法，初始化可能失败。
- **触发：** 安装接管、执行回调、初始化失败或 stop。
- **必须成立：** 五个约定方法在业务回调前接管，安装后直到进程退出均走 logger；第三方的同环境全局调用一致；接管失败不进入协议运行。初始化前、缓存旧方法和其它运行环境不被虚报为已保护。
- **禁止结果：** 接管成功后因初始化失败恢复 stdout，序列化异常抛入业务回调，SDK 短写或并发写入造成自身 envelope 交错。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 五个方法映射且 stdout 仅含协议帧 | `packages/plugin-sdk/tests/logger.test.ts`：`console.* maps onto the plugin logger and stdout carries only protocol frames` | Covered |
| 复杂值与 sink 失败不抛入业务 | 同文件：`hostile values never throw and degrade to bounded descriptions`、`a failing sink is absorbed rather than thrown into plugin code` | Covered |
| 初始化前边界、失败后持续接管、第三方调用与 stop | 尚无覆盖这些生命周期路径的直接证据 | Missing |
| SDK 短写和并发写入的完整 envelope 顺序 | 尚无故障注入证据 | Missing |

- **决策依据：** D1、D4、D5。

## Mixed Stderr Must Be Decoded With Bounded Memory and Explicit Loss

- **风险：** read 边界被误当日志边界，未知格式被丢弃，编码膨胀或队列拥塞导致无界内存。
- **前置状态：** 混合 v1、未知版本、旧前缀、超长输入、无效 UTF-8 和未换行尾部；队列满或 sink 失败。
- **触发：** 分块接收、EOF、持续输入或写入故障。
- **必须成立：** 按实际流解码，无法识别的内容按 raw INFO；超长/无效编码保留有界可追溯表示；队列拒收新记录并持续 drain；过滤、确定丢失、结果不确定分别计数；总字节占用有界，告警不复制 payload。
- **禁止结果：** 无界缓冲、把截断 envelope 当作完整事件、把过滤算丢失、对不确定写入虚报精确成功或丢失数。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| read 边界、structured/raw、EOF 尾部 | `crates/plugin-runtime/src/plugin_log/pipeline_tests.rs`：`persists_structured_and_raw_records_independently_of_read_boundaries` | Covered |
| 分片与无效字节可追溯 | `crates/plugin-runtime/src/plugin_log/record.rs`：`fragments_and_invalid_bytes_are_preserved_traceably` | Covered |
| 满队列拒收新记录并区分策略过滤 | `crates/plugin-runtime/src/plugin_log/pipeline_tests.rs`：`a_full_queue_drops_the_newest_records_and_counts_them` | Covered |
| 编码膨胀、嵌套深度及队列总字节界限 | 尚无完整边界测试 | Missing |
| 部分写入、flush 失败、告警限次与未知结果计数 | 路径冲突测试只覆盖初始化失败，不能证明这些义务 | Missing |

- **决策依据：** D5、D7，不变量 6、8、9。

## Log Level Changes Must Be Persistent Isolated and Immediately Effective

- **风险：** 一个插件的设置影响其它插件；设置失败却已改变有效值；卸载与配置更新竞争残留状态。
- **前置状态：** 多个插件处于运行或停止状态，默认或已配置不同级别，配置存储可能失败。
- **触发：** 通过 Host 接口调整级别、并发更新、重启或 retain/delete uninstall。
- **必须成立：** 默认 INFO，仅目标身份变化；过滤时使用当前有效阈值，已入队记录不被追溯删除；持久化失败不报告成功且有效值与存储一致；retain 保留、delete 清除，失败可恢复。
- **禁止结果：** 改写日志 severity，影响 Ora runtime log 级别，插件自报配置身份，删除失败后丢失原设置。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 默认、身份隔离与重开持久化 | `crates/plugin-lifecycle/src/log_level_tests.rs`：`levels_default_to_info_persist_and_stay_per_identity` | Covered |
| 运行中设置送达及过滤阈值生效 | 同文件：`launch_receives_the_reserved_directory_and_a_live_level`；`crates/plugin-runtime/src/plugin_log/pipeline_tests.rs`：`applies_the_live_threshold_as_a_floor_without_restart`；仍缺真实 Host 到插件落盘的整链证据 | Partial |
| retain/delete 处置与清除失败报告 | `crates/plugin-lifecycle/src/log_level_tests.rs`：`uninstall_disposition_decides_whether_the_level_survives`、`delete_uninstall_reports_a_failed_level_clear`；失败后的整体回滚仍需证明 | Partial |
| 设置写入失败、并发设置及卸载竞争的一致性 | 尚无完整故障/竞争证据 | Missing |

- **决策依据：** D6、D8，不变量 7。

## Timed Out Teardown Must Fence Old Writers Before Reuse or Deletion

- **风险：** 旧 pipe 不关闭或文件 I/O 卡住，stop 返回后旧 writer 仍写新代文件或已移动的目录。
- **前置状态：** 子进程继承 stderr，队列有待提交记录，旧 writer 可能阻塞。
- **触发：** stop、启动超时、异常退出、Host 关闭、新代启动或 delete uninstall。
- **必须成立：** reader 与 writer 共用一个截止时间；到期撤销旧代入队并报告未知残余；未释放 writer 的清理责任和排他权继续存在。新代可运行但 sink 不可用；释放后仅后续代重试。删除必须确认无 writer，失败保留或回滚日志、数据、设置。
- **禁止结果：** 每阶段重新获得完整超时，取消等待即认定句柄释放，新代复用未完成 writer，删除成功后后台继续写，stop 因日志无限等待。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| stderr 不关闭时最终有界返回 | `crates/plugin-runtime/src/plugin_log/pipeline_tests.rs`：`teardown_is_bounded_when_stderr_never_closes`；该测试不能证明单一截止时间或 writer 隔离 | Partial |
| 真正阻塞 writer 下的代际排他、新代 drain 与最终释放 | 尚无直接证据；当前 `pipeline.rs::finish` 超时后放任 blocking writer 完成，需补齐所有权管理 | Missing |
| retain/delete 日志目录处置 | `crates/plugin-lifecycle/src/log_directory_tests.rs`：`uninstall_disposition_decides_whether_the_log_tree_survives` | Covered |
| writer 未释放时禁止删除，失败全量回滚 | 同文件：`delete_uninstall_fails_and_rolls_back_while_the_log_is_open` 仅覆盖 Windows 打开句柄场景，未覆盖真实 writer 与设置共同回滚 | Partial |
| 启动失败及 Host 关闭复用同一收尾约束 | 尚无对应直接集成证据 | Missing |

- **决策依据：** D8，不变量 10、12、14。

## Reopening After Partial Writes Must Preserve History Without Joining Records

- **风险：** 崩溃留下半条 JSONL，下一次 append 与其拼接，损坏新日志或覆盖历史。
- **前置状态：** 活动文件为空、以换行结束或含未换行尾部；恢复名称可能已存在，恢复可能失败。
- **触发：** 取得排他写入权后重新创建 sink。
- **必须成立：** 有界检查尾部；完整文件正常追加；残缺文件用不覆盖的唯一恢复名称原样保留后新建活动文件；失败则 sink 不可用。恢复文件随卸载保留/删除，不宣称 flush 提供掉电持久化。
- **禁止结果：** 截断历史，覆盖恢复文件，直接拼接尾部，未取得排他权就移动文件。
- **验证义务与证据：**

| 验证义务 | 代表性证据 | 状态 |
| --- | --- | --- |
| 完整文件创建、复用与追加 | `crates/plugin-runtime/src/plugin_log/sink.rs`：`creates_every_level_reuses_and_appends` | Covered |
| 残缺尾部恢复、名称冲突与恢复失败隔离 | 尚无直接证据 | Missing |
| 恢复文件参与 retain/delete 事务 | 尚无直接证据 | Missing |

- **决策依据：** D10，不变量 15。

## Managed Child Output Must Be Logged Only When the Plugin Forwards It

- **风险：** Host 自动复制 child-process 输出，导致重复落盘或持久化插件未选择记录的敏感内容。
- **前置状态：** 插件通过 `ora/childprocess/*` 启动外部工具。
- **触发：** 工具产生 stdout/stderr，插件选择转发部分内容到 logger。
- **必须成立：** 原始输出正常返回发起插件；仅主动转发的内容按对应级别进入 Plugin log。
- **禁止结果：** Host 自动追加一份 child-process 原始输出，或把该输出混入 Ora runtime log。
- **验证义务与证据：** `Missing`：需要通过生产 child-process 接口验证未转发零落盘、转发一次仅一条；现有 logger 单元测试不能替代此证据。
- **决策依据：** D9，不变量 11。
