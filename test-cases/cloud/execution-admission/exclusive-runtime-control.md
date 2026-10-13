# 独占操作、失权与安全交接

已实现服务承诺仍由原 ADR 拥有；下文的 UI 与完整联合故障矩阵改由[后续验收 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)承接。保留本文件与英文标题作为稳定引用，Partial/Missing 不因后端 implemented 而自动提高。

依据[已实现后端的独占准入决策](../../../decisions/cloud/execution-admission/20260927-exclusive-runtime-control.md)。新增代码与部分直接证据已登记，完整验收仍有缺口。
租约是有期限的操作资格，fencing 表示拒绝旧资格的新操作；两者都不是进程已结束的证明。

## One Runtime Has At Most One Effective Operator Session

- 风险：多实例、两个管理员或同一用户两个页面分别持有有效操作权。
- 前置状态：空闲运行时，有权限的两个用户及同用户不同会话；获取使用不同幂等键并同步竞争。
- 触发：同时获取，获取中再次获取，重复提交原请求，同键更换输入。
- 必须成立：恰一个会话原子预留，其他请求 409；执行端确认绑定后才有效。默认有效期 60 秒、每 20 秒续期，以数据库时间裁决；只有同一有效绑定可续期。
- 禁止结果：两个持有者、获取中放行写入、客户端身份建立权限、历史幂等成功结果复活失效资格。
- 验证义务与证据：

| 义务 | 所需直接证据 | 状态 |
| --- | --- | --- |
| PostgreSQL 原子获取与幂等 | 多连接及多服务实例同步竞争，完整检查允许结果、持久行与重复请求 | Partial |
| 执行确认前不放行及界面冲突 | 延迟绑定确认的真实对端测试，管理员和同用户第二页面的界面测试 | Partial |

- 决策依据：D1、D2、D5，不变量 1、3。

- 本次直接证据：**Partial** — [Cloud integration/runtime_control_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_control_test.go) 的同步获取、幂等/第二页面/确认前拒绝；[cluster 真实验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/runtime-control-acceptance.py) 的真实 Node 绑定。
- 仍缺：多 Cloud 服务实例、UI 和响应丢失完整矩阵尚缺。

## Conflicting Write Activities Are Exclusive Even Within One Session

- 风险：单用户同时启动 Agent、终端、Git 或文件写入，仍竞争同一目录。
- 前置状态：有效持有者已开始写活动；另有获准只读观察者和插件维护待办。
- 触发：同会话启动第二个冲突活动、管理员编辑、插件卸载；shell 返回但留下后台进程。
- 必须成立：同运行时至多一个冲突写活动，第二个 409 且无副作用；只读按内容权限可并行。后台进程仍属于执行责任，不因 shell 返回报告空闲；维护不抢占人类会话。
- 禁止结果：按入口名称分别加锁、Agent 运行时编辑、只跟踪父 shell、系统维护绕过占用。
- 验证义务与证据：**Partial** — 需真实文件/Git/进程测试以及活动登记竞争，检查拒绝方无磁盘或进程副作用；尚未交付的产品入口保持关闭。
- 决策依据：D1、D3，不变量 2、4。

- 本次直接证据：**Partial** — [desktop crates/node-db/src/tests/repository.rs](https://github.com/ora-space/desktop/blob/cf166325d393e1b81a6b3f2fde7e6026d13cb128/crates/node-db/src/tests/repository.rs) 的 runtime_mutex_spans_identities_and_started_work_survives_closure；Cloud ticket 唯一约束和插件维护等待。
- 仍缺：真实文件/终端/Agent/插件入口尚未具备执行配套，保持关闭；不同入口实际进程竞争仍缺。

## Expiry Or Disconnection Does Not Permit Handoff Before Old Work Ends

- 风险：数据库到期后立即交接，旧连接、排队命令或进程继续写入。
- 前置状态：旧会话有已接受执行或派发结果未知，租约将到期；管理员准备接管。
- 触发：断线、释放、到期与续期竞争；重放旧请求；恢复浏览器或 Node 后继续输入。
- 必须成立：到期不能迟到续期恢复；进入收尾/待对账，关闭旧新输入并保留既有执行责任。旧入口关闭且全部执行结束才回空闲，新获取严格增加控制代次。前端恢复先读权威状态，不盲目重放离线编辑。
- 禁止结果：未知结果当未执行、仅心跳超时就交接、自动夺取有效会话、旧请求复活原代次。
- 验证义务与证据：**Partial** — 需真实 PostgreSQL 与 Node，控制响应丢失和断线时点，确认交接前后旧消息无新写入；必须与跨进程核心用例联合验证。
- 决策依据：D2、D4，不变量 3–5。

- 本次直接证据：**Partial** — [Cloud integration/runtime_control_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_control_test.go)、[Cloud integration/runtime_connection_recovery_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/runtime_connection_recovery_test.go) 与 [desktop crates/node-db/src/tests/repository.rs](https://github.com/ora-space/desktop/blob/cf166325d393e1b81a6b3f2fde7e6026d13cb128/crates/node-db/src/tests/repository.rs) 的关闭/恢复实际入口拒绝。
- 仍缺：Cloud–Node 接受结果丢失、在途执行到期及交接的联合故障矩阵尚缺。

## Membership Revocation Denies New Access While Preserving Accepted Execution Responsibility

- 风险：停用后 HTTP、事件、续期或派发仍获准；或为撤权删除正在收尾的任务。
- 前置状态：成员持有操作权、打开事件流，既有任务已接受，另有尚未接受和接受结果未知的请求。
- 触发：成员停用提交与新请求、续期、派发和事件投递竞争；创建者离开并尝试用旧凭证恢复。
- 必须成立：提交后的 Cloud 新请求和续期被拒，事件下次投递前重新校验；未接受动作不新增执行，未知动作按稳定身份对账。已开始任务按原范围收尾，不再接收新交互输入；引用及审计保留，最后管理员保护保持。
- 禁止结果：无限离线续权、因发起人失权丢弃结果、成员停用后继续投递受限内容或新建 ticket。
- 验证义务与证据：**Partial** — 需 HTTP、SSE、真实执行派发及 PostgreSQL 竞争的直接联合证据。既有成员停用 HTTP/SSE 测试只证明前序义务，不证明操作权撤回和远端执行约束。
- 决策依据：D4、D5，不变量 5、6。

- 本次直接证据：**Partial** — [Cloud integration/space_events_test.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/integration/space_events_test.go)、runtime control 测试及 [cluster 真实验收脚本](https://github.com/ora-space/cluster/blob/aef10f3b6de16bf0bed6797a37bf0b1b3d9bc7c5/scripts/runtime-control-acceptance.py) 的成员停用步骤。
- 仍缺：真实脚本已通过 HTTP/SSE、续期、新运行时准入及 Node 控制撤回联合验证；在途任务停用、未知接受及并发联合矩阵仍缺。
