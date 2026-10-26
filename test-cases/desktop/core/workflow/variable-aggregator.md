# 变量聚合的声明、优先级与失败边界

Aggregator 的候选顺序是业务优先级；键存在即已赋值。以下证据固定到 desktop
`d91344097ac9a2d0f079585cb1d8fbffa8dd78f7`，对应
[V1 contract 与直接/parse 测试][contract]、[引擎场景测试][engine]。
引擎场景使用内存 repository；不把它等同于 SQLite 保存重载或桌面端到端证据。

## Invalid Candidate Declarations Are Rejected Before Execution

### 风险

非法根、类型不一致或非前驱来源会使取值依赖调度时序，或把无法赋值的输出传播到下游。

### 前置状态与触发

冻结图包含 Aggregator；执行前解析图及候选声明。

### 必须成立

- 候选列表有序、非空、无重复，只允许根选择器，根必须已声明。
- 节点来源必须是严格传递前驱，不能是自身、下游或无关节点；互斥分支的两个前驱均可选。
- 所有候选声明类型相等，输出以该类型声明，writer 为 Aggregator 自身。
- globals/system 不按节点前驱检查，仍执行根声明和类型校验。

### 禁止结果

接受非法候选后等运行时碰运气取值，或解析时重排作者声明的优先级。

### 验证义务与证据

| 验证义务 | 代表性直接证据（[源码][contract]） | 状态 |
| --- | --- | --- |
| 缺配置/空列表、嵌套路径、重复候选被拒绝 | `rejects_an_empty_selector_list`、`rejects_nested_and_duplicate_selectors` | Covered |
| 类型相等、输出类型与 owner 声明；候选顺序保留 | `rejects_type_mismatched_selectors`、`accepts_sibling_branch_sources_and_declares_the_common_type`、`keeps_the_declared_selector_order_through_parse` | Covered |
| 拒绝非前驱、自引用、未声明的闲置节点根 | `rejects_downstream_selectors`、`rejects_unrelated_selectors`、`rejects_self_referencing_selectors`、`rejects_selectors_referencing_unused_spare_nodes` | Covered |
| globals/system 均免节点前驱校验 | `accepts_global_variables_as_candidates` 直接覆盖 global；此测试族未直接覆盖 system 候选 | Partial |

### 决策依据

[根决策 D6][decision] 的候选声明、类型与前驱边界。

## Declaration Order Selects The First Assigned Candidate

### 风险

沿用早期“多值冲突失败”会拒绝合法优先级图；按完成先后选值会让并行执行结果不确定。

### 前置状态与触发

多个候选已经声明，部分或全部已赋值；Aggregator 满足 DAG 扇入就绪条件后执行。

### 必须成立

- 跳过未赋值候选，按声明顺序透传第一个已赋值值到 `{aggregatorId}.output`。
- 同时存在多个值也成功；反转候选顺序会改变结果。
- 并行扇入等待活跃前驱完成；优先级由声明顺序决定，与完成先后无关。
- 互斥分支的未执行来源保持未赋值，已执行分支的值可经 Aggregator 传至 Output 并完成 run。

### 禁止结果

多值即失败、选择先完成的低优先级分支、等待不活跃分支赋值，或将多个候选合并成新值。

### 验证义务与证据

| 验证义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 同时赋值时按顺序选择，未赋值候选被跳过 | [直接测试][contract]：`first_assigned_selector_wins`、`skips_unassigned_candidates` | Covered |
| 活跃并行前驱完成前不启动；先完成 A 仍选择声明在前的 B | [引擎场景][engine]：`parallel_fan_in_uses_the_declared_priority` | Covered |
| 互斥分支两种选择均写入聚合输出并完成 run | [引擎场景][engine]：`active_branch_a_output_flows_through_the_aggregator`、`active_branch_b_output_flows_through_the_aggregator` | Covered |

### 决策依据

[根决策 D6][decision] 的 first-assigned 与 DAG 就绪规则，明确取代早期多值冲突失败。

## Falsy Values Remain Assigned And Pass Through Unchanged

### 风险

用真值判断代替键存在，会丢失用户明确产出的空值并错误回退到低优先级候选。

### 前置状态与触发

候选键已存在，其值为 `null`、`false`、`0`、`""`、`[]` 或 `{}`；Aggregator 执行取值。

### 必须成立

上述值全部算已赋值，原样透传；不转换成未赋值或默认值。

### 禁止结果

跳过已存在的空值，或改变值的类型/内容。

### 验证义务与证据

| 验证义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 六种空值在选择逻辑中均命中并保持原值 | [直接测试][contract]：`falsy_values_are_assigned` | Covered |
| 空值经引擎写入聚合输出并使 run 成功 | [引擎场景][engine]：`falsy_boolean_passes_through_the_aggregator` 直接覆盖 `false`，其余五种值尚无本场景的直接证据 | Partial |

### 决策依据

[根决策 D6][decision] 的键存在判定。

## No Assigned Candidate Fails The Node And Run

### 风险

无匹配被当作空成功会掩盖图定义或上游产物错误，让下游消费虚构结果。

### 前置状态与触发

所有合法候选均已声明但未赋值；Aggregator 就绪并扫描候选。

### 必须成立

选择失败，诊断列出扫描的候选；引擎以 `AggregatorNoMatch` 使节点和 run 失败。
未声明根是声明错误，不能降级为“未赋值，继续扫描”。

### 禁止结果

无匹配输出空对象并成功，或静默忽略未声明根。

### 验证义务与证据

| 验证义务 | 代表性直接证据 | 状态 |
| --- | --- | --- |
| 无匹配失败并列出候选；未声明根拒绝 | [直接测试][contract]：`no_assigned_selector_fails_naming_every_candidate`、`undeclared_candidate_fails_closed` | Covered |
| 节点与 run 均失败，错误种类与候选诊断保留 | [引擎场景][engine]：`no_assigned_candidate_fails_with_the_aggregator_kind` | Covered |

### 决策依据

[根决策 D6][decision] 的无匹配失败与声明边界。

[decision]: ../../../../decisions/desktop/core/workflow/0-node-runtime-orchestration.md#d6变量输出和并发由各自-owner-保证
[contract]: https://github.com/ora-space/desktop/blob/d91344097ac9a2d0f079585cb1d8fbffa8dd78f7/crates/application/src/workflow_run/engine/aggregator.rs
[engine]: https://github.com/ora-space/desktop/blob/d91344097ac9a2d0f079585cb1d8fbffa8dd78f7/crates/application/src/workflow_run/engine/aggregator_tests.rs
