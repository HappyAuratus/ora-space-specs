# Desktop 工作流按执行位置调试

决策依据为[按选中的执行继续](../../../../decisions/desktop/core/workflow/20261124-debug-from-selected-execution.md)。每节标出适用条款。跨包验证从同一运行页选择、预览、确认，再重开详情。用持久事实判断哪些结果有效，用记录输入和调用 ID 的可控 Agent 核对真实派发次数。表中 `Missing` 表示该义务尚无已登记的直接实现证据。

## Editing the third item must preserve the ordered prefix

**D1、D3、D5。** 冻结 [a,b,c,d,e]，Agent 原样返回项值。前两项已结算，旧 Agent 正在处理 c。选中这次尝试，把提示词改为给项值加 `new-`。若按定义节点或数组值定位，可能重做前缀或读到另一项。

| 验证义务 | 证据 |
| --- | --- |
| 预览定位 it1/index=2/f3；预览前后调用记录、账本一致。确认后旧、新尝试各收到 c，五项累计调用 [1,1,2,1,1]。 | Missing |
| 释放旧尝试的迟到回报，重开详情。有效 output 为 [a,b,new-c,new-d,new-e]；entries 按原序共五条、失败数 0，旧尝试仍可回看。 | Missing |
| 全部五项成功后再回选第 3 项，累计调用 [1,1,2,2,2]。第 3–5 项的旧账与旧汇总退出当前投影。用重复项值复测，索引仍能区分两项。 | Missing |

## Rerouting a condition must withdraw the old branch from the join

**D2、D3、D5。** Start 提供 flag=true，旧 IF A 命中，ELSE 通往 B，汇合 J 按 A、B 顺序读取首个已赋值的候选。旧 J 已读 a，独立 S 已完成。将 c1 的比较值改为 false。残留的 A 会优先遮住 B。

| 验证义务 | 证据 |
| --- | --- |
| 预览保留 Start、S，列出旧分支、A、J 的失效范围。确认后当前 A 未赋值，J 只收到 b，旧 a 仍可查历史。Start/S/A/B 累计各一次，Condition/J 各两次。 | Missing |
| 改为旧 J 的消费者仍在执行时确认。先停稳旧消费者，新尝试仅接收 b；独立 S 若仍在途，只提交原尝试一次。 | Missing |

## Resuming a while member must use the selected round

**D1、D4。** 轮内 Agent 依次收到 S0、S1、S2，挂起第 3 轮的 S2 尝试，改其提示词使新尝试返回 DONE 并命中退出条件。错用其他轮的输入会改变反馈和退出时机。

| 验证义务 | 证据 |
| --- | --- |
| 新尝试定位到 round=3，亦收到 S2；前两轮与本轮有效前置保留。三轮累计调用 [1,1,2]，外层只读 DONE，第 4 轮零派发。 | Missing |
| 若旧执行已完成到第 4 轮，从第 3 轮改为提前退出后，旧第 4 轮、旧导出和旧下游仍可审计，均退出当前计算。 | Missing |

## Changing a container must rebuild it from its entry

**D1、D3、D4。** 先在 it1 的第 3 项改成员提示词，得到 [a,b,new-c,new-d,new-e]，再选 it1 改源为 [e,c]。复用旧后缀会将旧位置错误映射到新输入。另以 Loop 验证从入口重新取初值。

| 验证义务 | 证据 |
| --- | --- |
| 第二次确认前，预览列出要重建的旧账与导出，显示新容器成员的有效提示词为原样返回 item。确认后两个新调用分别收到 e、c，输出 [e,c]。历史完整保留 it1 的成员覆盖、五项账本及旧导出。 | Missing |
| 只改收集选择器或错误策略也从第 1 项重建；容器外已提交的上游和独立工作均只执行一次，下游只消费新汇总。 | Missing |
| 选择 Loop 把上限 5 改为 2，退出条件持续为假。用初值开启新第 1 轮并按原上限策略终止，无第 3 轮；只改反馈或退出条件时，同样从入口建立新轮池和导出。 | Missing |

## A ready unstarted node must enter normal scheduling

**D1、D2、D5。** 运行 P 中 Start 已提交 text="hello"，在调度器首次认领 X 前打开其表单。它尚无 nodeRunId。使用 D1 的 pendingTarget 及相同输入，覆盖 UI、预览接收端和调度器。

| 验证义务 | 证据 |
| --- | --- |
| 表单提交 rootP/rootP/X；预览不创建执行行。确认后由普通就绪门首次派发，恰好一个真实尝试绑定新修订并收到 hello。 | Missing |
| 分别缺少 Start 的输入、缺少可唯一识别成员的作用域。前者报告缺少的前置值，后者指出缺少的位置字段。两者均无可确认凭据、无派发。 | Missing |
| 预览后抢先派发 X，再确认返回 stale_preview，原尝试继续，额外修订和调用均为零。未启动的 Pending 运行使用原编辑/启动入口。 | Missing |

## A top level edit must be private to its run and preserve history

**D1、D2、D6。** 两个 run 使用同一发布快照。在其中一个选择已成功的根级 Agent X 改模型，再选新尝试改提示词。原地改写配置会让历史或另一 run 显示错误的模型。

| 验证义务 | 证据 |
| --- | --- |
| X 的三次尝试分别显示原配置、新模型、新模型加新提示词；父修订依次为 0、1。上游和独立分支只执行一次，下游读取最新 X。 | Missing |
| 按请求键重查能找回各操作；失效尝试可查实际配置及替代关系。另一 run 和发布快照保有原配置。取消编辑时修订、调用数均不增加。 | Missing |
| 分别从 Running（含 AwaitingInput）、Failed、Cancelled、Succeeded 选择证据齐备的目标，均可预览确认；已终结 run 在同一 ID 下继续。 | Missing |

## Confirmation must fence retries and survive restart

**D5、D6。** 挂起旧 Agent，保留延迟重试、独立在途兄弟和它写入的文件。分别制造影响范围内事实变化、兄弟单独推进、停稳超时和确认过程中重启。

| 验证义务 | 证据 |
| --- | --- |
| 目标输入、未赋值状态或受影响尝试集合变化时，旧预览返回 stale_preview，零停止、零修订。仅独立兄弟推进且读集相同，原预览可确认。 | Missing |
| 同键同凭据双击只受理一次，同键异凭据报冲突；并发确认最多一个生效。客户端断连或漏收通知后读回同一操作与阶段。 | Missing |
| 无法证明在途外部写入已隔离时，记录 stop_failed、具体尝试和实际停止结果，受影响范围零新修订、零新派发。兄弟仍可提交。解除阻塞后新凭据与新键才能再试。 | Missing |
| 在停稳中及生效后尚未派发时分别重启。持久操作最多生效一次；显式释放旧回报、重试或人工确认，记录其到达，新变量和下游调用数均不受影响。 | Missing |
| 确认前能看到文件保留和外部动作可能重复的提示；重开后仍可读兄弟已写入的文件。 | Missing |

## Missing provenance and invalid patches must block precise resume

**D1、D2、D5、D6。** 存量 run 仅保留最后一项的池值，缺第 3 项的输入来源。另在历史齐全时预览越界变量、无效模型、无效数组源或修改连线的补丁。

| 验证义务 | 证据 |
| --- | --- |
| 分别返回具体缺失事实或字段路径，无可确认凭据、无修订、无派发。缺少关联的历史行返回未知修订和未知配置，不能以当前快照倒填。 | Missing |
| 若另有容器入口证据，须选入口、重新预览全部影响并确认后才重算。入口也缺证据时给出原因。 | Missing |
| 普通失败恢复继续按原容器粒度及其原有工作区选项执行，运行内调试保留现场文件。分别从两个入口观察状态与结果。 | Missing |
