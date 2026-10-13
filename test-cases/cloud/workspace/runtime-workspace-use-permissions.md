# 运行时使用权限、创建者及历史兼容

已实现服务承诺仍由原 ADR 拥有；下文的 UI 与完整联合故障矩阵改由[后续验收 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)承接。保留本文件与英文标题作为稳定引用，Partial/Missing 不因后端 implemented 而自动提高。

依据[已实现后端的运行时使用权限决策](../../../decisions/cloud/workspace/20260927-runtime-workspace-use-permissions.md)。
新增代码与部分直接证据已登记；缺口逐项保留。既有租户成员与跨成员创建测试继续保留，不能代替新增内容权限和独占控制的验收。

## Runtime Content Access Requires Its Creator Or A Current Tenant Admin

- 风险：共享项目或继承的 owner 被误当作全部运行时的内容使用权，普通成员越权查看代码或输出。
- 前置状态：alice 与 bob 为同租户普通成员，各自创建运行时；另有当前管理员、跨租户用户和已停用成员。
- 触发：各身份读取项目及运行时概况、代码、输出、operation 详情，进入操作模式或经旧链接重试。
- 必须成立：有效成员可读共享项目与安全概况；内容只允许创建者和当前管理员；普通操作还须独占及活动准入。权限按当前 PostgreSQL 状态计算，main 不因名称公开。
- 禁止结果：凭 owner、历史 actor、资源 ID、前端按钮或共享项目成员身份获得他人内容；跨租户泄漏；建立成员授权表、授予/撤销接口或授权界面。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| 概况与内容分层、详情和重试入口一致 | 真实 HTTP 与 PostgreSQL 权限矩阵，逐项断言拒绝响应无受限内容及无写副作用 | Partial |
| 前端有效操作与第一阶段无授权功能 | 创建者、管理员及他人运行时界面测试，绕过按钮直接调用仍拒绝 | Missing |

- 决策依据：D1–D3，不变量 1–3。

- 本次直接证据：**Partial** — [Cloud integration/runtime_permissions_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_permissions_test.go) 的 TestRuntimeContentPermissionsAndReplay；[cluster 真实验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/runtime-control-acceptance.py) 的共享概况、main 内容与跨租户拒绝。
- 仍缺：完整运行时操作界面尚未接入，文件/输出产品入口仍关闭。

## Runtime Creator Is Recorded Separately From Durable Ownership

- 风险：其他成员创建运行时导致外键失败，或持久 owner 被重新解释为创建者。
- 前置状态：alice 创建项目，bob 为同租户普通成员；项目已有 main 运行时。
- 触发：bob 创建独立运行时并完成初始化；读取创建记录、任务发起者和权限投影。
- 必须成立：运行时 owner 沿用项目，创建者为 bob；main 创建者来自项目创建请求；operation actor、任务发起者与当前操作者分别记录。管理员进入不转移上述身份。
- 禁止结果：改写项目 owner、把最近操作者当创建者、拒绝合法跨成员创建。
- 验证义务与证据：**Partial** — 需在既有 `TestDifferentMemberCanCreateRuntimeInSharedProject` 基础上直接断言新增创建者、身份保真及实际权限，原测试不能证明新增规则。
- 决策依据：D1、D4，不变量 1、4。

- 本次直接证据：**Partial** — [Cloud integration/runtime_permissions_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_permissions_test.go)、TestDifferentMemberCanCreateRuntimeInSharedProject 与 [cluster 真实验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/runtime-control-acceptance.py) 的 owner/creator 对比。
- 仍缺：已有生命周期 actor 与创建者得到证明；完整任务/Agent 产品入口的联合验收仍缺。

## Historical Creators Are Backfilled Only From Unique Creation Evidence

- 风险：升级猜测归属、把未知历史运行时开放给普通成员，或删除恢复所需引用。
- 前置状态：升级前数据库含唯一有效创建 actor、缺失 actor、冲突证据、main、软删除资源及未完成执行；另准备空库。
- 触发：追加迁移建新库、升级旧库并重复执行；尝试访问回填后的运行时。
- 必须成立：只从匹配目标且唯一的创建记录回填；不能证明的创建者明确未知且仅管理员使用。原 owner、时间、版本、operation/execution/effect、清理和审计引用保留；存量执行不被解释为空闲。
- 禁止结果：改写历史 SQL、用 owner 或任意管理员填补缺失、未知资源对成员开放、迁移期间留下旧准入旁路。
- 验证义务与证据：**Partial** — 需真实 PostgreSQL 新库/升级/重入与完整历史快照比对，结合旧执行对账后授权切换；0017 租户迁移证据不证明新迁移。
- 决策依据：D4，不变量 4–6。

- 本次直接证据：**Partial** — [Cloud integration/runtime_upgrade_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_upgrade_test.go) 的唯一/冲突/缺失证据、operation 完整快照、重复迁移及不可变创建者。
- 仍缺：软删除、在途执行与所有历史表联合快照矩阵尚缺。

## Separate Runtimes Keep Their Code Tasks And Persistent Data Isolated

- 风险：独立运行时共用可写目录、任务范围或维护对象，导致成员工作互相覆盖。
- 前置状态：同项目两个创建者各有独立克隆和持久目录，均取得本运行时的有效操作权。
- 触发：并行编辑、Git 操作或任务；停止、强停及删除其中一个运行时。
- 必须成立：不同运行时普通执行可并行，代码、任务与数据分别归属；单个运行时处置不影响另一个。项目生命周期仍保持既有串行约束。
- 禁止结果：共享同一可写目录、跨运行时任务终止或数据清理、以不同目录为由免除单运行时独占。
- 验证义务与证据：**Partial** — 需真实 Cloud–Controller–Node 和部署存储链路，检查两套磁盘内容、执行及进程；模拟器独立 clone 不构成完整证据。
- 决策依据：D1–D3，不变量 4；同批独占 D3 与强停 D5。

- 本次直接证据：**Partial** — [cluster 真实验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/runtime-control-acceptance.py) 创建同项目两个真实 Node/Git/命名卷，并检查只在成员 clone 目录写入标记。
- 仍缺：任意 shell、Agent 及完整停止/删除故障矩阵仍缺，未开放入口保持关闭。
