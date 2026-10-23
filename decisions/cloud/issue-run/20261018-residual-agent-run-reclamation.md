---
status: implemented
---

# 残留 Agent Run 资源以真实证据分层回收：证据齐沿 owned delete，未确认执行经部署运维受控回收，账本按目标闭环清除

> 历史决策：D2 的迟到成果救援由
> [迟到交付事实接管决策](20261022-late-delivery-fact-takeover.md) D2、D3 补充：同 Issue 已有后创建 Run 时，
> 所有清理阶段与 D8 受控接管都只收原声明与 receipt，不登记迟到 Revision，不把 receipt-only 当作已验证
> 交付或额外清理授权。创建先后按同 Issue 的原子 `run.enqueued` 活动 `seq` 判断，原 Run 与同 Issue 其他 Run 的创建顺序证据缺失或有歧义时同样只收回执；只有顺序证据唯一且没有后创建 Run 时，成果救援行为才不变。
> 无账本、无意图的根 D5/D8 事实接收不创建本 ADR 的回收责任。本文保持 `implemented`；新增规则的实现与证据状态以后续 ADR 为准。

D4/D5 业务收尾留下的残留账本驱动持续对账。自动对账层周期收集真实证据——迟到的执行终态与交付结果、执行已终结、沙盒已物理终止——证据齐的条目取得本文新增的系统清理授权后，复用 `delete_workspace` owned delete 的执行步骤清理，成果能救则先救；清理完成前账本保护不解除。证据收不齐的条目只能由部署运维按租户与资源范围发起受控回收：显式确认放弃未交付成果，复用管理强停的终止与围栏语义（终止与数据删除分离、逐目标确认 `terminated` 与 `lateEnsureFenced`、不因超时判成功），经当前 Controller lease 与既有 Substrate effects 执行，不新增物理路径、不伪造 `impactConfirmed`、不由业务 `done` 隐式强停。账本条目在全部目标资源取得闭环证据或显式处置记录后清除；Substrate 永久不可用时保留未完成意图，由运维外部确认兜底。存量 D8 删除残留按原身份接入同一路径，不改写历史。

本文处于 `implemented`：Cloud、Controller 与 Sandbox Server 的实现已合并；[owner 于 2026-10-21 的决定](https://github.com/ora-space/circus/issues/251#issuecomment-6315318982)采用下述更保守的首版规则，并接受本地 `cloudctl residual` 运维入口。联合演练按实际候选分别取证，工具 ora-space/cluster#36（ora-space/circus#388）已合并，合并提交 `7adcb6e`、合入 head `a11e025`；`implemented` 不表示所有联合验收义务均为 `Covered`。详见[落地结果](#落地结果)与[残留资源回收用例](../../../test-cases/cloud/issue-run/residual-resource-reclamation.md)。

本文承接 [有界启动与未派发撤回决策](20261012-bounded-agent-startup-and-undispatched-withdrawal.md)（下称“启动决策”）D4 预留的“Agent Run 残留资源对账与回收”ADR。联合职责为 Cloud 控制面（账本、对账编排、受控回收意图）、Controller/Fleet（许可、对账与派发证据）与 Substrate（终止与围栏执行）。涉及 cloud 的 `internal/core`（残留账本、operation、受控回收意图）与部署运维命令 `cloudctl residual`；desktop Controller 与 cluster Sandbox Server 不新增机制，只按既有决策提供证据与执行。兼容性边界：不改启动决策 D4/D5 的结算语义、不改根 D8 残留的状态与记录、不开放隐藏运行 Workspace 的公开操作入口；`agent_run_residuals` 表只增列。本文不承担对象存储中 Revision 的过期清理、告警与通知机制、以及生产 Substrate 适配器的具体实现。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| 启动决策 D4：残留责任表指定“后续回收 ADR”负责清理授权、任务持久化与重试、真实终止、late ensure fencing、成果救援与删除顺序、运维入口与失败升级 | 本文承担全部指定项；责任表中 Cloud/Controller/Node 的既有职责不变 |
| 启动决策 D4/D5：业务 `done/failed` 保留残留账本与资源保护，真实终态及交付/放弃证据齐全时可沿已有 owned delete 清理 | 继承执行步骤；首版自动清理仅接受按原身份核验的 Revision，不实现明确放弃交付路径；新增系统授权入口，清理完成前仍保留账本保护（D2） |
| 启动决策 D5：事实接管入口按原 effect/执行身份幂等补证据，不重开 admission、不再结算 | 继承并扩展接管时段至清理中和 `cleared` 后；不得仅因账本已清除而拒收同身份真实事实，不改 Run 终态（D2） |
| 根 D8：删除侧仅在存活沙盒的 Node 未知超过不可达窗口时放弃，残留如实保留，运维处置，无后台重试 | 继承条件、状态与不自动重试的边界；D6 仅新增按原身份发起受控回收的入口，不改写原结算来源 |
| 根不变量 8：`done` 不代表运行 Workspace 已删除 | 继承并加强：账本清除同样需要资源处置证据，业务终态不是清除条件（不变量 2） |
| [管理强停](../operation/20260927-safe-stop-and-administrative-force-stop.md)：强停是独立授权意图，`impactConfirmed=true`，终止与数据删除分离，逐目标确认 | 继承语义并扩展授权边界：隐藏运行 Workspace 对公开 API 为 404，租户管理员无入口；受控回收由部署运维发起，确认语义同强停（D3） |
| [运行时控制围栏](../controller-integration/20260927-fenced-runtime-control-delivery.md) D4：终止确认包括 `terminated` 与 `lateEnsureFenced`，紧急终止用独立短期 permit | 继承；受控回收的完成条件沿用双确认，permit 规则不变 |
| [Docker 沙盒生命周期](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) D6：`sandbox_terminate` 以容器不存在为证据、先写墓碑，目标从未 ensure 过也确认终止与围栏 | 继承为回收完成证据；对生产 Substrate 适配器提出同等语义要求（D4） |
| 启动决策 D4“需要强停时遵循强停与围栏原则，但对隐藏运行 Workspace 的授权扩展由回收 ADR 决定，不能伪造 `impactConfirmed`” | 本文即该扩展：授权归部署运维的受控回收命令，自动层与业务终态都无此授权 |

## 决策

依据[owner 决定（2026-10-21）](https://github.com/ora-space/circus/issues/251#issuecomment-6315318982)，D1 的异常旧行拒绝迁移、D2 的既有入口取证与已验证 Revision 资格、D4 的 Shutdown 控制关闭均以实现为准：不确定的情况保持阻挡并交运维，不推断完成或误删。D3 接受部署数据库凭据授权的本地命令；不新增运维 HTTP 认证体系。

### D1：残留账本增加清理状态机，清除需要资源处置证据

`agent_run_residuals` 每行增加清理状态 `reclaim_state`，取值 `reconciling → reclaiming → cleared`：

| 状态 | 含义 | 进入条件 |
|---|---|---|
| `reconciling` | 自动收集证据，尚未取得清理推进责任；业务结算写入账本时的初始状态 | D4/D5 登记账本（现状不变） |
| `reclaiming` | 一个持久清理责任在推进；记录自动 owned delete 或运维受控回收模式 | D2 的系统授权或 D3 的显式命令在同一竞争边界内取得责任 |
| `cleared` | 残留责任闭环 | 全部目标取得闭环证据或显式处置记录（D4/D5），写入 `cleared_at` 与处置方式 |

- `cleared` 的处置方式 `disposition` 取值：`released`（沙盒终止且数据已按 owned delete 或受控回收删除）、`terminated_data_retained`（终止与围栏已确认，数据按运维决定保留）、`externally_confirmed`（Substrate 之外的外部确认，D5）。
- 对账证据逐条记录在账本行上（证据类型、来源、时间），不改写 `reason` 与既有 `responsibility`；重复对账只追加证据，不重建 Run、Workspace、ensure 或执行身份。
- 存量行（D4/D5 已登记的）迁移时统一置 `reconciling`，不回溯判定证据。实现为迁移 `0057_agent_run_residual_reclamation.sql`，只给 `agent_run_residuals` 增列（清理状态、处置方式、单一责任、目标并集与追加证据）；`cleared_at` 此前只是预留，若旧行已有非空值，迁移明确失败并回滚，不猜测处置方式。
- 清理责任固定原租户、Project、Run、Workspace、sandbox/generation 与计划/执行身份，并关联唯一推进意图或 operation、模式及授权证据。推进者重启或 Controller lease 更替只能接管同一责任，不能取得第二份删除或终止权。

为什么给账本加状态而不是新建清理任务表：账本行已经是残留责任的唯一权威（启动决策 D4），清理是这份责任的收尾，分表会让“谁还欠什么”需要两表对齐；`cleared_at` 列在 D4/D5 落地时已预留。

#### 旧行已有 cleared_at 时的运维处置

以 Cloud main `33662780c358c50ae2b954ad402e666136b5284f` 的 `internal/core/migrations/0057_agent_run_residual_reclamation.sql` 为准：仅在表尚无 `reclaim_state` 且存在非空 `cleared_at` 时，迁移在增列前抛出以下数据库异常：

```text
residual reclamation migration cannot initialize legacy rows with non-null cleared_at; explicit compatibility handling is required (A2)
```

`cloudctl` 对未分类的数据库错误只输出 `internal_error`，不能依赖 CLI 回显上述原文；应在受控的数据库诊断中识别该异常。迁移器在同一事务执行本次所有待应用迁移，异常使本次事务回滚，0057 不记为已应用。旧时间戳本身不能证明现代处置方式、授权或逐目标物理证据；若直接按默认值设为 `reconciling` 又保留非空时间戳，也不满足新完成约束。

1. 暂停此次升级，保留旧行和迁移失败记录；不跳过 guard、不手工标记迁移已完成，不删除残留行或批量清空时间戳。在目标部署、正确 schema 的授权 PostgreSQL 会话中执行只读排查（仅使用旧表列）：

```sql
SELECT tenant_id, project_id, run_id, workspace_id,
       runtime_generation, reason, responsibility,
       created_at, session_ended_at, cleared_at
FROM agent_run_residuals
WHERE cleared_at IS NOT NULL
  AND NOT EXISTS (
    SELECT 1 FROM pg_attribute
    WHERE attrelid = 'agent_run_residuals'::regclass
      AND attname = 'reclaim_state'
      AND NOT attisdropped
  )
ORDER BY tenant_id, run_id;
```

2. 按原租户、Run、Workspace、generation 与责任目标逐行核对时间戳的写入来源、原 operation/effect 与交付记录、Controller/Substrate journal、沙盒和数据卷现状；确认是否仍有执行或写入者、是否曾真实处置、是否存在并发清理责任。保存原行、时间戳、证据、判断及审批记录，凭据与连接串不进入工单。无法查明则保持阻挡，不能把未知判为已清理。
3. 没有现成的旧行修复命令。只有证明某一行的 `cleared_at` 是误写、该责任仍应受残留保护后，才可由授权运维提交单独审阅的数据库修正：在暂停相关写入、事务内锁定并重读该行后，仅按原身份及原时间戳限定该行，将 `cleared_at` 恢复为 NULL，保留其他历史与原时间戳的变更记录。这是重新保留未完成责任，不是物理成功证明，不是默认批量修复。
4. 若时间戳确属真实历史闭环，不为通过迁移而擦除它。继续阻挡升级，另开 issue 准备并审阅兼容 SQL/迁移，在保留历史的同时明确完整责任、授权、处置方式与证据；当前版本没有通用映射或自动修复工具。
5. 经审阅的兼容处理完成后，恢复原部署迁移流程（现有入口为 `cloudctl -config <path> -command migrate`）。所需迁移全部成功后，才可用 `cloudctl -config <path> residual list --tenant-id <tenant>` 或 `cloudctl -config <path> residual get --tenant-id <tenant> --workspace-id <workspace>` 查看未完成责任，再按 D3/D5 决定是否显式终止、删数据或外部确认。迁移前这些命令依赖的新列和表尚不存在，不能用来修复旧行；也不能用外部确认替代本次兼容处理。

### D2：自动对账层只收集真实证据并沿 owned delete 清理，永不强停

本节的迟到成果救援受[迟到交付事实接管决策](20261022-late-delivery-fact-takeover.md) D2、D3 修改：同 Issue 已有按 `run.enqueued` 活动 `seq` 后创建的 Run 时，各账本阶段及 D8 受控路径只保存原声明与 receipt，不登记迟到 Revision；receipt-only 不是已验证交付或额外清理授权。原 Run 与同 Issue 其他 Run 的创建顺序证据缺失或有歧义时同样只收回执；只有顺序证据唯一且没有后创建 Run 时，本节成果救援行为才不变。该新增守卫的直接验证义务由后续用例单独登记，不由本文既有证据覆盖。

Cloud 以可配置周期（默认 1 分钟）扫描 `reconciling` 条目，只读取经既有事实入口持久接管的证据，不新增取证查询 RPC，也不要求 Controller 新增历史发现能力。必须覆盖原责任集合、已封闭新派发，且所有执行已终结或其全部可写实例已取得终止与围栏双确认，交付资格由原 `deliver_revision` 执行登记且按原身份核验完成的 Revision 提供，其他交付执行也须各自核验完成，才可申请系统清理授权。首版没有“明确放弃交付”作为自动清理资格的路径；失败、跳过、超时或没有结果都不算放弃。Controller 无法回报的历史事实使条目保持 `reconciling`，等待运维依 D3 接管。单个终态或单个沙盒确认不能代替完整证明。

| 证据 | 来源 | 允许的清理动作 |
|---|---|---|
| 迟到真实终态与交付结果 | 迟到事实按原执行身份接管（既有路径） | 补登记交付（成果救援），随后沿 owned delete；交付结果未确认时按启动决策保持 `deliveryState` 已结算值 |
| 执行封闭与终态 | 既有 `runtime_ack` 按原 control epoch/version、Node、sandbox/generation 接管关闭输入与未完成 execution IDs；既有执行事实入口接管原执行终态 | 仍须核对已验证的 Revision 与全部目标证据；缺任一证据则等待，不主动查询或推断放弃；齐全后以原身份声明 owned delete，quiesce 通过后释放 |
| 沙盒已物理终止与围栏 | 既有 effect 结果入口接管原目标 `sandbox_terminate` 的 `terminated` 与 `lateEnsureFenced` 双确认，包括从未 ensure 的计划 ID | 证明该目标的物理执行已封闭；仍须覆盖全部目标与交付资格，同一 `delete_workspace` 才能继续推进数据删除 |
| 以上都取不到 | — | 保持 `reconciling`，等待下一周期或运维受控回收 |

**系统授权与保护转换**：普通 `deleteRunWorkspace` 仍拒绝 `cleared_at IS NULL` 的残留；系统清理入口取得限定原责任的授权后复用 owned delete 的执行步骤，不能先填 `cleared_at` 再调用普通入口。创建侧事实接管覆盖清理前后，授权遵循以下规则：

1. 在与原资源计划、派发登记、账本事实写入和 D3 受控回收互斥的事务边界，重读原资源关联及完整证据，固定全部待处置目标，关闭后续新派发资格。只针对这次残留和原资源授予清理权，不授予新会话、ensure 或换代权；Project/Workspace 的既有冲突及范围保护仍须核验。
2. 取得单一持久责任并置 `reclaiming`，同时声明或关联唯一 owned delete operation；事务失败全部回滚。自动层与受控命令竞争时只有一方取得推进权，同一输入重放返回原责任，另一模式冲突不能并行计划副作用。自动层不能自行把模式升级为受控回收；需要强停时仍须 D3 的人工确认，已发动作先按原责任对账，不用覆盖意图制造第二个执行者。
3. `reclaiming` 期间 `cleared_at` 保持空值。只有绑定该责任的系统动作可越过针对本条目的残留阻挡；公开 API、普通删除及涉及残留的 Project stop/delete 继续受保护。执行仍依次核验 quiesce、逐目标终止与围栏、数据删除证据，不能因已有清理授权跳过任何物理证据。
4. 只有全部目标闭环后，才在核对最新责任及处置证据的事务中写 `cleared_at`、处置方式和完成关联；不能先清账以腾出删除入口。`terminated_data_retained` 只证明终止责任已收口，保留的数据仍归原 Workspace，不隐含后续删卷授权。

**事实接管贯穿清理前后**：创建侧事实入口与原执行接管必须扩展到 `reconciling`、`reclaiming` 和 `cleared`，仍验证原租户、资源、effect/execution 身份及当前服务资格；不能用 Run `done` 或账本已清除拒收。先持久接管再 ACK：相同事实幂等重放，补充事实只追加证据，不重开 admission、不再次结算、不重建已删除资源。事实与清理授权/完成竞争时，后提交者须看到先提交者的证据与责任；新发现的未闭环目标阻止清除。清除后若事实与处置结论冲突，保留原结论及冲突审计，阻断受影响资源的后续删除/复用并交运维核查，不静默丢弃事实或伪造新的完成证明。

- 自动层的边界：只做证据收集、成果救援与上述系统授权的 owned delete；它**不发起任何强停、不排针对未知执行的数据删除、不把证据不足解释为已放弃**。quiesce 拒绝说明当前不能安全清理，保留同一持久责任和保护，重新取证后按 owned delete 规则重试，不因 Run 已 `done` 丢失任务，也不自动升级。
- 成果救援只经既有事实接管与 Revision 核验链路；重复事实使用原执行身份，不复制 Agent 会话，不主动新发查询 RPC，不因救援重开输入或续期任何窗口（继承启动决策 D1/D3）。
- 既有事实未到达、Controller 不可达、租约失效或扫描读取失败均不记为证据；下个周期重读已有事实。历史事实始终无法取得时保持保护，由运维显式选择受控回收。

为什么证据收集放自动层而强停不放：证据只缩短未知（把“不知道”变成“知道已终结”），不产生新的不可逆动作；强停会放弃未确认成果并终止可能仍在执行的进程，其代价必须由人显式选择（D3）。

### D3：受控回收是部署运维的显式命令，确认语义同强停，终止与数据删除分离

- 发起者：部署运维，经部署主机上的本地命令 `cloudctl residual list/get/terminate/delete-data/confirm-external`，使用部署数据库凭据执行，操作主体记为 PostgreSQL `session_user`（`--operator-label` 只是人工自述）。没有 HTTP 路由，不暴露给租户用户；租户管理员对隐藏运行 Workspace 也没有入口——公开 API 恒 404。按租户与资源范围查询账本由 `list` / `get` 只读提供。
- 命令输入：目标（账本条目 / Run / Workspace 范围）、非空 `reason`、显式放弃确认（语义同强停的 `impactConfirmed=true`：确认终止未知执行并放弃未交付成果）。幂等键、同键冲突规则沿用强停意图的模式。
- 意图持久化：独立的残留回收意图（`registered → terminating → succeeded` 三态，响应丢失或不可达时保持未完成并继续对账，不因一过期限就成功——继承强停 D4 的偏离说明）；意图引用账本条目与目标资源身份，发起者、时间、reason 留审计。实现为迁移 `agent_run_controlled_reclamation.sql` 新增的意图、逐沙盒目标和按租户与幂等键唯一的命令回执三张表：每个资源一个 terminate 根意图、至多一个 delete-data 子意图，二者都用上述三态；同键不同输入先提交冲突审计再报错。
- 两阶段分离：**终止**（默认，对每个目标沙盒取得 `terminated` 与 `lateEnsureFenced`）与**数据删除**（单独的显式命令 `delete-data`，经既有 operation 协议 `DELETE_WORKSPACE` / `cleanup` 才排 `WORKSPACE_DATA_DELETE`）。只终止不删数据是合法终态：账本以 `terminated_data_retained` 闭环，Workspace 行与卷按运维决定保留；删除数据后以 `released` 闭环。
- 回收意图在 D2 的同一竞争边界取得单一责任后，账本为 `reclaiming`；已有自动清理责任时不并行取得终止/删除权。意图失败或不可达不回退状态，继续对账直到闭环。
- 受控回收不改变 Run 终态、不重开 admission、不重写 `deliveryState`；它只终止资源与闭环账本。目标涉及其他运行、主 Workspace 或历史 Revision 时按既有 Project stop/delete 的保护拒绝（继承启动决策 D5“涉及残留的 Project delete/stop 仍拒绝无证据操作”）。

授权依据是部署主机和数据库凭据的访问控制：只有获准使用该部署数据库业务权限的运维人员才能运行命令；残留命令本身不需要 DDL 权限，迁移另需 Schema 权限。`session_user` 是数据库认证主体，`--operator-label` 不能覆盖它，租户 admin/member 的 HTTP 身份不授予此权。Cloud 尚无独立运维身份；[owner 已接受](https://github.com/ora-space/circus/issues/251#issuecomment-6315318982)这一首版边界，HTTP 管理入口须先另定认证设计。

为什么新意图而不复用租户强停表：发起者与授权边界不同（部署运维 vs 租户管理员），合并会让租户侧出现可枚举隐藏 Workspace 的旁路；而确认、幂等、逐目标确认与三态语义完全复用强停已评审的规则，不发明新状态机。

### D4：物理终止经当前 Controller lease 与既有 effects，完成条件是逐目标双确认

- Cloud 为回收意图计划**既有**的 `sandbox_terminate` / `workspace_data_delete` effect；Controller 以当前 lease epoch 与许可执行，Cloud 不可用或许可失效时不发起新动作（继承围栏交付 D3）。不新增 Cloud 直连 Substrate 的路径。终止目标按根意图 ID 映射到既有强停 RPC 的形状（`ListForceStops` 列出计划、`GetEffectPermit` 逐目标取许可、`ConfirmForceStop` 回报双确认），不新增 RPC 或 proto，也不写伪造的 `runtime_force_stops` 行；数据删除走既有 operation 协议，不包装成强停。
- Controller 在发出破坏性 PUT 前重新核对当前 lease epoch 与许可到期时间，期间 lease 丢失、epoch 不符或许可过期则不发 PUT。单个目标的许可拒绝或 Substrate 失败不阻塞其他目标；lease 丢失或 Cloud 不可用时停止整轮派发。
- 终止前 Controller 只向可达 Node 尝试既有 Shutdown 控制关闭（有界、尽力而为，不把其确认作为终止前置），不为回收新增 EndSession 结束请求或授权路径。无论控制关闭是否送达，完成仍须逐目标取得 `terminated` 与 `lateEnsureFenced` 双确认。
- 目标集合 = 账本 `responsibility` 中记录的全部沙盒计划 ID 与预分配身份，包括**从未执行过的 ensure 计划**：对每个 ID 逐一 `sandbox_terminate`。[Sandbox Server](../../cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md) D6 已保证目标从未 ensure 也写墓碑并报告 `{terminated: true, lateEnsureFenced: true}`；生产 Substrate 适配器必须保持同等语义（先墓碑后破坏、幂等、对不存在的目标也确认），这是本 ADR 对 Substrate 的接口要求。
- 迟到许可与在途 ensure 的 fencing 完全依赖既有机制：墓碑之后同 ID 的 ensure 以 `sandbox_terminated` 拒绝——Sandbox Server 先校验许可有效期、作用域与本地权威，许可有效且作用域正确的受管 ensure 命中同 ID 墓碑时返回 `503 {"error":"sandbox_terminated"}`；Controller 识别后进入 Blocked，不重试、不换代；回收意图不删除原 effect 行、不假报 sandbox 缺失、不创建新 generation 挂载同一可写卷（继承启动决策 D5）。
- **完成条件**：意图下每个已计划的稳定 sandbox ID 都实际取得 `terminated` 与 `lateEnsureFenced` 双确认，数据删除阶段另需每个卷的删除确认。全部确认后意图 `succeeded`、账本按处置方式 `cleared`；任何目标未确认则保持 `terminating` 并继续对账，或由运维显式进入 D5 的外部处置分支。
- **未计划与未到达分开判断**：只有 Cloud 在与创建计划、派发登记互斥的事务内，证明目标从未产生 sandbox/ensure 计划、许可或在途责任，并持久封闭继续产生这些责任的入口，才可记录“无目标待终止”。已有计划不能因尚未受理或请求可能丢失而移出目标集合。Substrate 的 effect 查询 `404` 只说明查询时本地 journal 没有该 intent，既不写墓碑，也不排除持有效许可的 ensure 随后到达；它不是上述 Cloud 证明，更不是终止双确认的替代品。

反例：Cloud 已持久计划 ensure A，Controller 的请求仍在网络中；此时查询 A 得到 `404`。回收仍须按 A 的稳定 sandbox ID 执行 terminate，取得双确认；不能先清账或删卷再等待 A。墓碑持久化后 A 即使迟到，也必须被拒绝。

- 回收完成与迟到事实并存时：迟到真实终止/清理证据缩小残留范围，迟到终态与交付按原身份接管，但 Run 终态与业务结算不再变化（继承启动决策 D4）。

### D5：重试、运维升级与 Substrate 永久不可用的兜底

- 自动层重试：按周期重读既有事实；已授权的 owned delete 由 D2 新增的持久清理责任驱动，复用既有 quiesce/删除步骤和幂等规则，不依赖只扫描 `releasing` 的业务重试循环。失败保留责任、证据和保护，无新增超时成功规则。
- 受控层重试：意图保持未完成即持续对账（`terminating` 含结果未知），Controller/Substrate 恢复后继续；不以固定超时把未确认判为成功。
- 升级路径固定为三级，逐级显式：

| 层级 | 动作 | 触发 |
|---|---|---|
| 自动对账 | 证据收集、成果救援、owned delete | 账本登记后持续 |
| 受控回收 | 运维显式确认后终止（+可选数据删除） | 证据长期收不齐或运维判断需要 |
| 外部确认 | 运维凭 Substrate 之外的事实关闭条目 | Substrate 永久不可用（宿主退役、平台故障不可恢复） |

- 外部确认兜底：运维对无法经 Substrate 取证的目标提供外部事实（如宿主已销毁的处置记录），非空确认说明随意图留存；账本以 `externally_confirmed` 闭环。这是运维责任的显式转移与审计记录，不是系统证据——确认文本不写成 `terminated`，效果上不伪造围栏。外部确认要求先存在未完成的回收意图，不允许无意图直接销账。
- 升级不自动发生：自动层不发起受控回收、不申请外部确认；“证据长期收不齐”的可见信号是账本查询结果本身（`reconciling` 且无新证据），告警与通知机制不在本文范围。

### D6：存量 D8 删除残留按原身份接入，不回填不改写

- 存量特征（根 D8）：Run 已会话结算、`delete_workspace` 以 `node_unavailable` 终结失败、Workspace 行与运行绑定保留、无 `agent_run_residuals` 行。
- 接入规则：受控回收与外部确认以**资源身份**（Workspace、失败的 delete operation、沙盒计划）为目标，同样适用于存量 D8 残留，不要求先回填账本表——`reason` 的 CHECK 约束（`agent_start_failed` / `workspace_unavailable`）也不允许无据改写来源。回收意图单独记录“D8 删除残留”来源。
- 自动层不扫描或重启这些存量删除：根 D8 放弃后 Run 已为 `done`，既有重新声明循环仅扫描 `releasing`，因此 Node 恢复可达也不会触发既有重试。本 ADR 不新增存量 D8 的自动恢复扫描；存量只由 D3 新增的受控回收入口接管，证据仍未知时保留原资源，不把恢复连接等同于清理授权。
- 接管与处置：受控命令核验原租户、Run/Workspace、失败 operation 与全部计划目标，在 D2 同等的互斥边界取得单一持久回收责任；无账本行时由该意图承载保护与证据，不伪造 D4/D5 账本。终止仍逐目标双确认，数据删除仍需单独显式命令，不能将历史 `done` 改回 `releasing` 来借用普通扫描。
- 处置完成后的记录：由新的回收意图及其处置记录引用原失败 delete operation，保存真实新证据；不把原 operation 的 `failed/node_unavailable` 改为成功，不改 Run 的原 `status`、`done` 或业务结算来源。外部确认同样先有意图并保留 D5 的审计边界。

## 不变量

1. 业务终态（`done`/`failed`）与时间流逝都不触发物理终止；终止只来自证据齐的 owned delete 或显式确认的受控回收。
2. 账本 `cleared` 需要每个目标的闭环证据或显式处置记录；`cleared` 不是业务终态的重复，`reconciling` 不是可丢弃状态。
3. 受控回收的确认语义是运维显式放弃未确认成果；系统任何一侧都不伪造 `impactConfirmed`、`terminated` 或 `lateEnsureFenced`。
4. 回收不删除原 effect 行、不重开 admission、不改 Run 终态与 `deliveryState`；迟到事实只能补证据。
5. 终止与数据删除分离；自动清理须有证据齐全后的系统 owned delete 意图，受控终止后的删卷须有独立显式命令，终止成功本身不授予删卷权。
6. 未完成的回收意图不因超时转为成功；外部确认必须先有意图且留审计。
7. 存量 D8 残留按原稳定身份接入，不改写历史结算与状态。
8. 已计划的 sandbox 逐目标取得终止与围栏双确认；Substrate 查询 `404`、Node 不可达和许可过期都不能代替该证据。
9. 同一残留只有一个持久清理责任；完成前不提前清账，清理授权不会解除其他调用方的残留保护。
10. `cleared` 后同身份真实事实仍先持久接管再 ACK；重放不复活资源或业务，冲突不得静默丢弃。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 业务 `done` 后自动强停残留沙盒 | 收尾即时；但放弃未确认成果与终止未知执行是人的决定，Substrate 不可用时仍会永久悬挂（启动决策已否决该方向） |
| 残留登记后自动排普通 delete | 走既有路径看似省事；quiesce 对未知执行必然失败，只是把创建阻塞换成删除阻塞（启动决策 D5 已否决同构方案） |
| 先写 `cleared_at` 再复用普通删除入口 | 可绕过当前阻挡；但物理资源未清理就解除保护，并切断创建侧迟到事实接管，须改为限定原责任的系统授权（D2） |
| 复用租户管理员强停 API 清理残留 | 少一套意图；但授权边界错位——隐藏 Workspace 对租户侧 404，开放枚举与操作即制造旁路，且租户管理员不应为部署级故障买单（D3） |
| Cloud 直连 Substrate 执行终止 | 少一跳；绕过 Controller lease 与既有 effect 的幂等、围栏与许可，生产 Substrate 适配器要另开一条路径（D4） |
| 对账层拿不到证据超时后自动升级强停 | 无需人工；把“放弃成果”变成默认行为，违背显式确认（不变量 1、3） |
| 删除失败即自动外部确认销账 | 清账快；把未知写成已处置，伪造证据（不变量 3） |
| 用 Substrate 查询 `404` 排除已计划的 ensure | 少一次 terminate；但查询不写墓碑，尚未到达的 ensure 仍可创建实例，不能解除删卷保护（D4） |
| 存量 D8 残留回填进账本表统一状态机 | 单表查询方便；`reason` 约束与来源语义不允许，强行改写历史结算违背 D8 继承（D6） |

## 本决策未解决的问题

- 主动取证查询 RPC 与 Controller 历史发现能力：首版不提供，无法回报的历史事实保持 `reconciling` 并交运维；若要补齐，另开 issue 设计协议与验收。
- 自动清理的明确放弃交付路径：首版不提供，只有按原身份核验完成的 Revision 提供交付资格；若要引入放弃记录及授权规则，另开 issue。D3 运维显式放弃确认不构成自动层的交付证据。
- 回收专用 EndSession 结束请求：首版只有 Shutdown 控制关闭；若要新增结束请求及 Cloud 授权路径，另开 issue，不放宽双确认完成条件。
- 独立部署运维身份与 HTTP 管理入口：本地 `cloudctl residual` 已被接受；若要增加 HTTP 入口，须另开 issue 并先决定认证与角色模型。
- 告警与通知（何时提醒运维存在长期 `reconciling` 条目）：不在本文；可见信号是账本查询。
- 自动对账周期、外部确认的最小证据要求等运维细则的默认值调优。
- 生产 Substrate 适配器对“terminate 从未 ensure 的目标也双确认”的验收方式：联合故障演练只覆盖 Docker Sandbox Server，其他生产 Substrate 适配器仍需各自验收。
- 对象存储侧：迟到交付补登记后的 Revision 对象生命周期（含过期清理）沿用 Revision 既有规则，不由回收路径处理。

## 落地顺序

1. **账本状态机与自动对账层**（cloud）：`reclaim_state`/证据与清理责任迁移、周期扫描、新增系统清理授权、清理完成前后的事实接管与 owned delete 持久推进。完成条件：完整证据齐才授权；未清账即可执行获授权的删除且普通入口仍拒绝；授权/事实/完成竞争、清理重启、清除后重复事实和冲突事实均有直接验证，证据不足不排强停或数据删除。D2 的系统授权与清理前后事实接管是本文实现复杂度最高的部分，**竞争场景（授权与事实、自动与受控、完成与迟到事实）是这一阶段的验证重点**，不能只用顺序路径的用例代替。
2. **受控回收意图与本地运维命令**（cloud）：意图三态、两阶段命令、审计与按范围查询。完成条件：幂等/冲突、确认缺失拒绝、自动清理与运维命令竞争时只有单一责任、只终止不删数据的合法终态均有测试。
3. **Controller/Substrate 语义验证**（desktop、cluster）：既有 terminate 对从未 ensure 的计划 ID 双确认、迟到 ensure 被墓碑拒绝，作为回收完成条件成立。完成条件包括：已计划且持有效许可的 ensure 在途时，查询 `404` 不能清账或删卷；terminate 双确认后再送达该 ensure 必须被拒绝。Cloud 的“从未计划”证明与并发计划/派发只能有一方取得责任。
4. **联合故障演练**：Controller 失联恢复后意图继续、Substrate 不可达保持 `terminating`、外部确认留审计。存量 D8 需直接验证 Node 恢复不自动重新声明删除；受控入口按原身份取得责任后才能回收，无账本行也有保护；重复命令不产生第二份责任，原 failed operation 与 Run 终态始终不变。

保持不变：D4/D5 结算语义、根 D8 残留状态、公开 API 对运行 Workspace 的 404、`agent_run_residuals` 既有列义。从第 1 步起改变：账本条目有了清理状态、单一推进责任、系统授权和贯穿清理前后的事实接管；从第 2 步起改变：残留有了受控回收的显式入口。

## 落地结果

- ora-space/cloud#112（第 1 步）：账本增列迁移 `agent_run_residual_reclamation.sql`、追加证据与冲突审计、单一 owned delete 责任、独立周期扫描，以及清理前后经既有事实入口的接管。不新增 RPC；Controller 无法经既有入口回报的历史事实仍缺失，相应条目保持 `reconciling`。
- ora-space/cloud#116（第 2 步）：意图、目标与回执迁移 `agent_run_controlled_reclamation.sql`，`cloudctl residual` 五条命令，两阶段处置、外部确认与存量 D8 按身份接管；终止目标复用既有强停 RPC 形状。
- ora-space/desktop#710 与 ora-space/cluster#35（第 3 步）：Controller 在 PUT 前复核许可、终止前尝试 Shutdown 控制关闭、墓碑拒绝不重试、逐目标隔离失败；Sandbox Server 对命中墓碑的有效受管 ensure 返回 `503 sandbox_terminated`，并补齐 terminate 语义的直接测试。
- 演练证据按实际候选分三轮，不以可变 PR head 代替运行版本：

| 轮次 | 固定演练记录 | 候选与工具版本 | 场景及边界 |
|---|---|---|---|
| 第一轮 | [首轮结果](https://github.com/ora-space/circus/issues/388#issuecomment-6299342004)、[D8 身份接管补跑](https://github.com/ora-space/circus/issues/388#issuecomment-6300622765) | Cloud `59a2443`；desktop `7ea516b`；Sandbox Server 为 cluster#35 `590c444` 加 Node 内存选项（集成提交 `3d9f552`）。工具：S2/S3/S2b 为 `04e2bd8`，S1/原 S4 为 `da3903d`，D8 补跑为 `8640080`。 | S1 通过；S2 约 300 秒不可达后恢复通过，已有卷保留；S3 外部确认通过；S2b inconclusive；原 S4 因 Node 恢复失败未进入接管，D8 补跑仅身份接管通过，已有卷先保留再显式删除。 |
| 第二轮（post-#183） | [S1 重跑](https://github.com/ora-space/circus/issues/388#issuecomment-6308500014) | Cloud `118b759`；desktop `fbe37e24`；cluster main `3e07a4c`；工具 `8209d6b`。 | 只运行 S1，恢复后沿同一意图与两个目标取得双确认，已确认目标不重发，显式 `delete-data` 到 `released`。终止前 Run Workspace 没有数据卷，不能证明已有卷被保留再物理删除。 |
| 第三轮（#560） | [固定候选演练结果](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955) | Cloud `d73eca76`；desktop `e177e021`；cluster / Sandbox Server `ae1dd951`。工具 driver `b87d30b`，复用 proxy 镜像源码 `7e08dc6`。 | `p343-run01` 的 S2b inconclusive、S2 中断不计证据；`p343-run02` 复用同一批镜像，仅 S2/S3/D8，均通过。D5/D8 均使用真实默认 30m；D8 最后活动到事务 1807.9 秒，原身份接管、双确认 retained、显式 delete-data 到 released、零账本及历史不变。 |

前两轮都是单机 Docker 上的真实服务与 Node。S1 的第二个历史 sandbox 身份仅作为输入布置，后续计划、派发、终止与确认来自真实系统，不代表真实多代历史。第一轮 D8 补跑先确认交付已被 Cloud 接收，再经 `ReportNodeIdle` 屏障断线，避开未接收终态重连冲突（ora-space/circus#392）；约 9.5 秒即形成 D8，未遵守默认 30 分钟窗口（ora-space/circus#391，现已由 ora-space/cloud#118 修复，但前两轮均未验证修复后的窗口）。它不证明窗口条件或旧版本滚动升级。第二轮没有重跑 S2、S3、S4、S2b 或 D8 接管，第一轮结果只适用于第一轮候选，不能作为最终 PR head 的部署证据。

第三轮采用同一 pinned candidate 的真实默认窗口，[S2/S3/D8 均通过](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)；`last_seen_at` 是 Node 最后记录的活动，Run `updated_at` 是 D8 事务时间戳，1807.9 秒不是精确心跳发送到提交的耗时。它补齐重试与显式升级、D8 身份接管两项跨仓联合路径，不证明旧版本滚动升级或未测试的故障交错。

逐目标终止与围栏按 [#573 决定](https://github.com/ora-space/circus/issues/573#issuecomment-6365603852) 以“迟到 ensure 不能复活已回收 sandbox”组织三部分证据：固定候选的许可 / admission / epoch 结构性论证、有效旧 epoch 403 与当前 epoch 墓碑 503 的单仓测试、真实栈回收后许可 trace 与终态读回。[cluster#45](https://github.com/ora-space/cluster/pull/45) 补旧 epoch 直接测试（head `0cb2bcd390b0ef7a91242c79a8726ddf58b01bd9`，待合并）；#560 的 S2 / D8 支持最终未复活，但缺许可签发 trace 与明确双确认后窗口，因此仍为 `Partial`。完整位置及缺口见核心用例；本轮未启动补跑。

- 产品实现及演练工具均已合并：ora-space/cloud#112、ora-space/cloud#116、ora-space/cloud#117、ora-space/cloud#127、ora-space/desktop#710、ora-space/desktop#712、ora-space/cluster#35、ora-space/cluster#36（合并提交 `7adcb6e`、合入 head `a11e025`）；停止未确认证据及其运维展示已随关联改动合入。`a11e025` 包含 S2b 响应等待修复，但未重跑部署演练；上述历史演练工具 head 保持不变；第三轮补证与 #575 按新证据组成登记的剩余缺口见核心用例。

## 核心测试用例

本 ADR 的核心测试用例见[残留资源回收用例](../../../test-cases/cloud/issue-run/residual-resource-reclamation.md)。九项义务中 **8 项 `Covered`、1 项 `Partial`**：前六项的 Cloud 范围保持 `Covered`，第三轮 [#560 固定候选演练](https://github.com/ora-space/circus/issues/560#issuecomment-6363072955)补齐重试与显式升级、D8 身份接管的联合路径；逐目标终止与围栏按 #573 的三部分证据组成判断，仍缺第 3 部分的许可 trace / 明确观察窗口而保持 `Partial`；cluster#45 单仓补测待合并。ADR 状态仍为 `implemented`，未改变决策语义。旧候选的通过结果按上述版本保留。新增迟到登记守卫见[迟到交付用例](../../../test-cases/cloud/issue-run/late-delivery-fact-takeover.md)，该决策已为 `implemented`：Cloud #142 merge `b777496` 的直接测试覆盖各接收路径的后创建 Run 守卫；缺创建记录后的重放、双向并发交错仍为 `Partial`，workflow 后创建路径仍为 `Missing`。逐项证据见该用例，不由上述历史演练证据自动覆盖。
