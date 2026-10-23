# 会话身份、绑定历史与统一诊断核心用例

本文件依据[会话根决策](../../../../../decisions/desktop/core/agent/session/0-session-identity-bindings-and-diagnostics.md)维护验证义务。
基础自动化证据固定到 desktop `d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116`，对应 [desktop#722](https://github.com/ora-space/desktop/pull/722)。
后端历史异常边界的补充证据固定到合并 commit `914bde1875a42e2368bf9be78018689f9e29659e`，对应 [desktop#725](https://github.com/ora-space/desktop/pull/725)：通过生产 bindings 接口读取真实 SQLite/JSONL 合成夹具；不证明真实 provider 运行、原生降级文案或全部插件入口无调用。
前端批量与生命周期补充证据固定到 desktop `2fe0f91b9c6f1585017817fbc3c9796ed1f2100f`，对应 [desktop#727](https://github.com/ora-space/desktop/pull/727)：通过生产 typed client 与受控 operation handler 验证 UI/查询行为，不证明真实 provider、宿主授权、原生异常或全部插件入口无调用。
宿主边界补充证据固定到 desktop `6cc77e37b3b146655cbb691f03bba532c249bfda`，对应 [desktop#730](https://github.com/ora-space/desktop/pull/730)：真实 SQLite/JSONL、实际扫描及卸载的隔离插件、真实本地 Deno 协议观察器与按平台执行的文件 IO 夹具。Deno 夹具只模拟协议，不调用真实 provider；冷查询阶段暂停普通自动监督器以隔离其启动，运行阶段恢复监督并等待 ACP Ready 后才取协议帧基线。
`Partial` 表示仅部分义务有直接证据，不表示整项已验收。
合并后原生交互证据依据 desktop main `f43f52fcef68a8ec6a1ea28ef7039188b7ec6f30`，见[直接验收证据](https://github.com/ora-space/circus/issues/482#issuecomment-6335994326)。通过 Windows Tauri/WebView2 生产查询消费隔离 SQLite/JSONL 合成夹具，覆盖代表性界面路径及冷启动；不证明真实 provider 运行产生这些记录，不代替下表剩余并发、权限及副作用观测义务。
下表各项剩余缺口统一由 [circus#96](https://github.com/ora-space/circus/issues/96) 承接；证据覆盖以下明确列出的接口与子路径，其余缺口仍由该事项承接。

子代理来源、委托结果、终态缺口和活动观察由根决策“继承与职责”引用的运行诊断 ADR 定义。
面板只定位父时间线的同一份只读详情树，保留其展开行为，不复制第二份内容或组件；本文件只验证组合边界。

## Two agent switches preserve three ordered binding segments

- 风险：只读当前行会丢掉切换前的排障身份。
- 前置状态：同一 Ora 会话 S 和 Workspace W，Meta 记录 A/a，随后两条 AgentSwitched 记录 B/b、C/c；插入 HandoffDelivered，at 与 seq 顺序不一致。
- 触发：打开 S 的绑定时间线并逐段复制 id。
- 必须成立：依 seq 展示 A/a、B/b、C/c 三段，只有 C/c 为当前；Ora id、Workspace 与历史路径不变；复制保留完整不透明 id，时间标明记录来源。
- 禁止结果：按 at 重排；HandoffDelivered 另造一段；截断复制；读取启动插件或改写历史。
- 决策依据：会话根决策 D1、D2。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 段数量、次序、当前标记和完整复制分别断言；查询前后核对身份及无写入、无插件调用。 | Covered | [真实 SQLite/JSONL 查询测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 验证三段顺序、当前唯一及忽略 handoff；[面板测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/chat/session-information.test.tsx) 验证完整复制。合并后原生交互确认普通对话/节点面板/索引三段呈现，并通过系统剪贴板精确核对完整 id。[binding_order_follows_sequence_despite_reversed_timestamps_and_append_order](https://github.com/ora-space/desktop/blob/914bde1875a42e2368bf9be78018689f9e29659e/crates/backend/src/session/binding_anomaly_tests.rs#L108) 补充 at 及物理写入顺序均与 seq 反序时仍按 seq 返回三段、仅末段当前，并通过 assert_projection 深比较查询前后的会话行及 JSONL 字节不变。[真实插件协议观察](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_plugin_tests.rs#L60) 补充受测插件在普通监督器暂停期间冷查询后保持 Stopped（只观察直接 lifecycle/gateway 启动）；普通监督恢复并完成 ACP 握手后，对重复三段、损坏、历史缺失、未创建及不存在会话的生产 bindings 查询，查询结束后等待 250 ms 静默窗口，再与查询前完整协议帧快照深比较相等、generation 不变。观察器的 control canary 先直接证明通信和记录有效，覆盖实际插件帧中的 lifecycle/ACP 调用；不宣称真实 provider 执行。 |

## Repeated plugin bindings remain distinct segments

- 风险：按插件或 id 去重会把再次绑定错误合并。
- 前置状态：Meta 为 A/x，切换至 B/y 后再次切回 A/x；当前行是最后一次 A/x。
- 触发：查询并定位每段。
- 必须成立：保留三个独立且可定位的段，只有末段为当前；即使插件已卸载也保留完整引用与 id。
- 禁止结果：仅按插件、显示名称或 id 合并；将首段 A/x 标为当前。
- 决策依据：会话根决策 D1、D2。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 对重复插件及重复 id 验证三段定位互异、当前标记唯一，并覆盖插件名称不可解析。 | Covered | [bindings_preserve_repeated_segments_and_ignore_handoff_without_starting_plugins](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 断言 A/x → B/y → A/x 的不同 key 和唯一当前段；同文件截断回归断言上限外 B → A 不把 history:0 标当前。合并后原生交互在未注册的 acceptance.fixture / acceptance.second 引用上确认 A/x → B/y → A/x 三段保留，仅末段当前。[实际卸载回归](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L8) 通过生产 scan 先确认完整插件身份，随后实际 uninstall 并确认安装目录和列表项消失；查询前后完整三段投影深相等，历史字节不变。插件已不存在、显示名称不可解析时仍保留原始引用、完整 id、三种独立 key 与唯一当前标记。 |

## Lagging history and concurrent switches never fabricate a stable binding

- 风险：历史落盘滞后或并发切换可能拼出两个当前绑定，或把缺口冒充完整审计。
- 前置状态：历史尾部 A/a，当前行为 B/b 且尚未落盘；另设读取中 B/b 切至 C/c、持续切换、无有效当前绑定三种状态。
- 触发：读取绑定投影，在当前行读取与历史读取之间触发切换。
- 必须成立：落盘滞后时保留 A/a 并补 B/b 活绑定，未知时间明确显示，缺口仍可见；并发视图不一致时有界重读，持续变化返回需重试；无有效当前绑定不将历史尾段标为当前。
- 禁止结果：补造当前时间、旧缺失段或持久 revision；搜索旧同 id 段冒充当前；无限重读；读 IO 阻塞 prompt 或持有调度事务。
- 决策依据：会话根决策 D2、D3。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 控制读取交错，分别验证稳定补段、有界重试、无当前绑定和未落盘旧段永久不可恢复；验证查询不写历史且不持调度事务等待 IO。 | Partial | [binding_tests.rs](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 验证活绑定补段、受控行变化最多两次读取和持续变化返回 Retry。[historical_bindings_survive_unbinding_but_never_resurrect_deleted_sessions](https://github.com/ora-space/desktop/blob/914bde1875a42e2368bf9be78018689f9e29659e/crates/backend/src/session/binding_anomaly_tests.rs#L152) 补充有历史但清空当前 agent_session_id 时保留历史段且全部非当前，并通过 assert_projection 深比较查询前后的会话行及 JSONL 字节不变。[未记录绑定回归](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L113) 直接覆盖中间绑定从未写入 JSONL、被后续当前行覆盖时仅返回可见初段与当前 live，不伪造中间 id，历史不变。[一致读取 seam](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L365) 验证历史回调内独立连接以零 busy timeout 取得 BEGIN IMMEDIATE 并提交；[Unix FIFO 阻塞 IO](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L412) 使用生产 bindings 等待实际文件字节期间，在同一单线程异步执行器完成另一会话持久 rename，释放后取得完整投影。尚缺真实崩溃窗口、真实 provider prompt 并发及调度事务的直接观测；writer slot 可用不证明没有任何 SQLite 只读事务。 |

## Unavailable histories remain distinguishable from absent sessions

- 风险：把不可读历史解释为从未绑定会掩盖丢失数据。
- 前置状态：分别准备历史缺失、权限不可读、损坏记录、Gap、不支持 schema、身份或 seq 矛盾、资源上限截断，以及会话已删除但残留 JSONL。
- 触发：单独查询每种状态。
- 必须成立：缺失或不可读明确不可用；损坏、Gap、矛盾、截断明确不完整，不支持版本明确不支持；仅展示可验证归属和次序的部分及已知当前绑定；删除会话明确不存在。
- 禁止结果：静默跳过后声称完整或尚未创建；从残存文件重建会话；无限读取；跨会话拼接记录。
- 决策依据：会话根决策 D2、D3。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 逐状态断言结果及降级文案；验证有限读取、可验证部分保留和删除会话无恢复副作用。 | Partial | [binding_tests.rs](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 覆盖缺失、外来身份、损坏、不支持 schema、8 MiB 截断保留历史并补 live；[面板测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/chat/session-information.test.tsx) 验证不可用不误称未创建。合并后原生交互确认缺失历史提示不可用并保留 live/时间未知，删除会话保留 Ora id 并提示不存在。[gaps_and_conflicting_sequences_preserve_only_verified_prefixes](https://github.com/ora-space/desktop/blob/914bde1875a42e2368bf9be78018689f9e29659e/crates/backend/src/session/binding_anomaly_tests.rs#L39) 补充显式 Gap 和同 seq 不同绑定均为 Incomplete，仅保留可信前缀且不标为当前，另补未知记录时间的 live；查询前后会话行及 JSONL 字节不变。[软删除残留历史测试](https://github.com/ora-space/desktop/blob/914bde1875a42e2368bf9be78018689f9e29659e/crates/backend/src/session/binding_anomaly_tests.rs#L152) 验证残留 JSONL 仍返回 NotFound、会话未恢复且文件字节不变。[逐状态面板测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/chat/session-information.test.tsx) 直接断言 unavailable / incomplete / unsupported、retry / not_found / unavailable 各自文案，且不显示尚未创建或空 id 复制。[Windows 文件拒绝打开](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L617) 用 exclusive-share 句柄实际阻止生产读打开，明确 Unavailable 并保留 live，批内不存在会话保留独立结果；释放句柄后恢复 Readable 且文件字节不变。sharing denial 不等于 ACL 权限拒绝；ACL、Gap、冲突 seq、残留历史等从生产查询到原生呈现的逐项证据仍缺。 |

## An uncreated agent session stays uncreated during inspection

- 风险：排障读取可能意外建立插件会话，或提供空 id 复制。
- 前置状态：Ora 会话存在，当前无 binding，历史可读且没有绑定记录。
- 触发：从菜单打开面板，并在工作流无 session 的执行项打开入口。
- 必须成立：现存 Ora 会话显示尚未创建且无空值复制；没有 session 的执行项保留执行状态并说明尚无会话。
- 禁止结果：调用 session/new、session/load、prompt 或重试；将失败绑定或不可用历史混同尚未创建；伪造 Ora 或 Agent id。
- 决策依据：会话根决策 D1、D3、D5。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 断言两种未创建状态的呈现与复制能力，并观测全部插件入口没有调用。 | Covered | [binding_reads_distinguish_uncreated_from_truncated_history](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 验证无绑定且空历史为 Readable 空段，与截断 Incomplete 区分。合并后原生交互确认已有 Ora 会话在可读空历史/无 binding 时显示尚未创建，只有 Ora id 复制，无空插件 id 复制；相同会话在历史缺失时明确不可用。[索引行为测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/workflow-run/run-session-index.test.tsx) 补充无 session 的失败执行仍保留原状态、显示尚无会话且没有伪造 id 按钮；[面板测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/chat/session-information.test.tsx) 补充可读空段不提供空 id 复制。[真实插件协议观察](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_plugin_tests.rs#L60) 在普通监督已 Ready 的受测 generation 上，对无 binding 且可读空历史、历史不可用和不存在会话的查询深比较前后全量协议帧不变；无 binding 结果显式为空段，未创建 session/new 或触发 session/load、prompt 等实际插件通信。 |

## Workflow retries keep three execution entries and all binding segments

- 风险：以节点或当前绑定去重会丢掉失败尝试及其历史。
- 前置状态：运行 R 中节点 N1 失败后重试成功，N2 执行一次，共三次执行；某次执行含多段绑定，另让两次执行引用同一 Ora session。
- 触发：打开运行索引并展开、定位每次执行。
- 必须成立：显示三次执行的节点、尝试、状态和 Ora id；每项展示其全部绑定段并定位同一面板；读取可按 Ora id 去重，执行项仍独立。
- 禁止结果：仅展示成功尝试或当前段；以连接健康代替执行状态；同 session 合并执行项；编辑态伪造索引。
- 决策依据：会话根决策 D5。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 以持久 node-run 夹具核对三次定位、状态及绑定明细；断言去重查询不合并行，编辑态无执行索引。 | Partial | [执行投影测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/state/data/workflow-session-executions.test.ts) 验证两节点加失败尝试三项；[索引测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/workflow-run/run-session-index.test.tsx) 验证共享 session 查询去重仍三行、删除会话保留并可打开面板。合并后原生交互通过持久 node-run 消费验证共享 session 的两行、失败尝试 1、展开全部绑定及统一面板定位；重启后五行与三段绑定恢复。[编辑态测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/workflow-editor/workflow-editor.test.tsx) 补充实际 WorkflowEditor 加载后没有运行索引或会话信息按钮。尚缺真实 provider 执行生成记录的证据。 |

## Workflow rounds and runs retain their own session ownership

- 风险：仅按节点名或视觉位置关联会将不同轮次和 run 的会话混合。
- 前置状态：同名节点在两个循环轮次及两个 run 中各有执行与不同 session；包含会话丢失和历史不可用项。
- 触发：切换 run、轮次，点击节点及失败尝试入口。
- 必须成立：依持久作用域、轮次、尝试事实定位相应会话；沿用原有轮次计数语义；不可用项仍保留执行定位、已知 Ora id 和不可用说明。
- 禁止结果：跨 run 拼入其他会话；按节点名称匹配；重编轮次计数；删掉不可用行；追溯已丢弃临时会话并伪造 id。
- 决策依据：会话根决策 D3、D5。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 核对每个入口的 run/作用域/执行/session 归属及不可用项；切换 run 后旧结果不能填入新索引。 | Partial | [执行投影测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/state/data/workflow-session-executions.test.ts) 验证 scope、原始 iteration/loopRound、缺失 session 和不从重试次数推算 attempt；[索引测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/workflow-run/run-session-index.test.tsx) 验证不可用行保留。合并后原生交互确认两个循环 scope 的第 1/2 轮独立，切换 run 后由五行变三行、scope/session 引用随之变化，缺失/删除项保留；冷启动恢复一致。[索引行为测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/workflow-run/run-session-index.test.tsx) 使用受控响应分别验证仅切换 runId（执行集合不变）、同 run 更换执行集合（保留旧 session 可见行）后的旧请求已中止；迟到 NotFound 的 session id 在新索引仍有可见行，新请求待定时不呈现该迟到状态，新响应正常呈现；逆序回包仍把不同 provider id 对到原执行行，并逐行打开所属 Ora 会话面板。同 run 测试通过更换执行集合验证查询生命周期，不证明真实轮次选择器已接入该变化。跨 run/轮次的节点和失败尝试等各入口完整矩阵仍待补。 |

## Batch queries isolate per session failures and enforce access bounds

- 风险：单项异常可能清空整个索引，或因可见 run 而越权读取任意会话。
- 前置状态：批量包含重复的合法 S1、历史不可读 S2、无权读取 S3，且数量超过单批上限。
- 触发：加载运行索引的批量绑定查询。
- 必须成立：按 Ora id 去重、分批并限制并发；逐 session 对应成功或错误，S1 仍可展示；宿主校验每项访问范围与受控路径。
- 禁止结果：一个失败掩盖所有成功；把响应按返回顺序错配节点；允许客户端指定文件路径或越权 id；无限并发。
- 决策依据：会话根决策 D3、D5。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 验证去重、分批、并发上限、逐项错误及乱序回包映射；拒绝越权及任意路径输入。 | Partial | [批量查询测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/state/data/session-bindings.test.ts) 验证 65 项分为 32/32/1、去重、单批失败不抹去其他结果及关闭后不继续请求；[后端测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 验证去重及超过 32 项拒绝。[批量受控测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/state/data/session-bindings.test.ts) 补充 65 个 id 各重复一次仍分 32/32/1、每批释放前在途数为 1 且总峰值为 1，逆序 not_found / unavailable 响应原样透传（不验证行映射）；[索引行为测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/workflow-run/run-session-index.test.tsx) 直接核对逆序响应没有串执行行。[危险持久引用测试](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L159) 将穿越、反斜杠、绝对路径、file URI、空值及空格 id 写入真实仓库后查询，均为历史 Unavailable 并仅保留当前 live；其中 `..-x/../../outside` 在未经校验的分片路径落点预置身份匹配的合法 Meta（`escaped-provider`），确保越界读取会直接改变响应并使断言失败。外部合法历史、哨兵文件和会话行均不变；该夹具直接证明持久化危险引用被路径校验拒绝，不等于拒绝整个请求或验证调用者授权。宿主调用者范围/越权拒绝证据仍待补。 |

## Every diagnostic entry shares one panel and rejects stale responses

- 风险：重复诊断正文和迟到请求可能在新会话中展示旧绑定或子代理详情。
- 前置状态：会话 S1、S2；菜单、聊天诊断快捷入口、工作流执行入口和子代理详情定位均可用；S1 查询延迟返回。
- 触发：依次从各入口打开，切到 S2、关闭重开面板或切换 run 后再释放 S1 响应。
- 必须成立：入口定位同一默认收起的只读面板，按 Ora id、绑定、运行诊断、子详情定位入口呈现；子详情入口定位父时间线的同一份详情树及同一组件，保留其展开行为；保持所属上下文，失效查询生命周期被释放，迟到响应不更新新上下文。
- 禁止结果：保留第二份诊断正文或独立弹层；复制详情树或将树从父时间线迁入面板；子代理 id 混入父绑定；重组面板重新定义来源、子终态、活动或权限语义；新增 cancel/steer/close/审批。
- 决策依据：会话根决策 D3、D4。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 验证入口组合与迟到响应隔离；用运行诊断领域提供的观察结果验证面板只组合，不重建子状态，记录时间不冒充实时活动时间。 | Partial | [面板测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/chat/session-information.test.tsx) 验证按需查询、切会话中止和迟到结果隔离、单一诊断正文；[虚拟列表测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/chat/message-list.virtual.test.tsx) 验证定位未挂载子组、deferred resize 不覆盖定位及键盘展开。合并后原生交互确认普通对话快捷入口、列表右键、节点聊天、索引行进入统一面板；普通/节点长历史从第 45 回合附近定位第 1 回合原详情树，布局稳定后 Enter 展开且不跳回尾部，保留未知终态。[面板测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/chat/session-information.test.tsx) 补充同一 session 关闭重开时旧请求已中止，重开后释放旧响应不呈现旧 id，新响应可正常呈现；[索引行为测试](https://github.com/ora-space/desktop/blob/2fe0f91b9c6f1585017817fbc3c9796ed1f2100f/packages/app-shell/src/features/workflow-run/run-session-index.test.tsx) 同样验证索引关闭重开、切换 run 和执行集合后的迟到隔离。尚缺菜单、快捷入口、节点、失败尝试及子详情定位入口组合的完整受控矩阵。 |

## Diagnostic identifiers do not expand public session exposure

- 风险：面板字段可能泄漏到默认载荷、日志或消息流，并扩大敏感信息暴露。
- 前置状态：存在完整插件绑定及带配置、工具输出的历史；公开 Session、列表、消息气泡和默认日志消费者均启用。
- 触发：读取公开接口和消息流，再主动打开面板、复制 Agent id。
- 必须成立：公开 Session 结构及默认 id 暴露边界不变；新增 Agent id 仅在主动诊断展示与复制中出现；面板查询仅返回受控投影。
- 禁止结果：返回原始 JSONL、绝对路径、MCP 配置、密钥或任意工具输出；在公开 Session、列表、气泡或默认日志添加 provider session id；前端扫描 JSONL。
- 决策依据：会话根决策 D1、D3、D4。

| 验证义务 | 状态 | 代表性证据 |
| --- | --- | --- |
| 分别断言公开契约、默认呈现及日志无新增 provider id；查询输出无敏感原始数据，主动复制完整 id；子详情输出沿用原领域契约，不经绑定查询扩权。 | Partial | [后端测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/crates/backend/src/session/binding_tests.rs) 断言公共 getSession 不包含 provider id 并深比较绑定响应；[面板测试](https://github.com/ora-space/desktop/blob/d5073d09cc0e58cd3f866f2fe9a5bb8010cb5116/packages/app-shell/src/features/chat/session-information.test.tsx) 断言主动打开前不查询/展示 id、复制完整字符串。合并后原生交互通过主动展开和复制核对完整插件 id。[敏感历史投影](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L516) 先正向断言实际持久 JSONL 含工具 rawInput/rawOutput、MCP secret 等哨兵，再深比较受控 bindings 响应，并验证公开 get/list 无历史或当前 provider id、无原始敏感内容，查询前后文件及会话行不变。[同步 worker 日志观察](https://github.com/ora-space/desktop/blob/6cc77e37b3b146655cbb691f03bba532c249bfda/crates/backend/src/session/binding_boundary_tests.rs#L292) 在测试作用域 TRACE subscriber 下记录 event、初始及后续 span 字段，以 canary 证明 recorder 有效；Gap 及损坏输入路径不泄漏 provider id、原始内容或路径（含 Windows debug 转义）。尚缺跨线程外层 Tauri 请求日志及默认列表/气泡呈现的完整泄漏矩阵。 |
