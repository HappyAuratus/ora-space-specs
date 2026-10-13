# Agent 控制面实现说明

本文说明 Cloud 的 Agent 执行控制面如何连接工作声明、Node 执行证据、Thread 业务投影和 Revision 交付。
源码依据是 [Cloud 控制面](https://github.com/ora-space/cloud/tree/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core)；
实现变化时同步维护本文及源码引用。术语见[领域词汇表](../../../domain/glossary.md#agent-issue-run)。

行为规则以[执行与 Thread 接管决策](../../../decisions/cloud/controller-integration/20260928-agent-run-executions-thread-and-upload-grants.md)
和[Revision 决策](../../../decisions/cloud/revision/0-cloud-owned-object-store-and-verified-revisions.md)为准。
本文描述源码接缝，不改变 ADR 状态；启动撤回与未知执行收尾另见
[启动结算提案](../../../decisions/cloud/issue-run/20261012-bounded-agent-startup-and-undispatched-withdrawal.md)。

## 职责与源码入口

Cloud 持久化执行计划、执行身份与接管结果。Controller 领取计划、登记执行、转发命令和 Node 事件；
Node 产生受理、会话记录和终态事实，并执行 Git 与对象上传。业务生命周期通过同事务钩子解释已接管事实。

| 接缝 | 源码入口 | 职责 |
|---|---|---|
| 工作声明与领取 | [execution_work.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/execution_work.go) | 固定工作输入与目标，声明会话或交付工作；领取是读取，派发登记才绑定执行 |
| 派发登记与结果接管 | [node_execution.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/node_execution.go) | 校验当前目标、输入、许可与执行身份；持久化 Node 结果和事件收据 |
| 业务转换 | [agent_run_hooks.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/agent_run_hooks.go)、[business_hooks.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/business_hooks.go) | 在调用方事务内更新 IssueRun、Thread，并声明后续工作 |
| 成果验证与授权 | [revision.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/revision.go)、[revision_grants.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/revision_grants.go) | 校验冻结的对象声明、签发临时请求、访问对象存储并登记验证结果 |
| 运行 Workspace | [run_workspace.go](https://github.com/ora-space/cloud/blob/b82834ec87ccb326859cba60c5ee2aa7befd0dc0/internal/core/run_workspace.go) | 通过已有 operation 声明运行专属 Workspace 的创建与删除 |

控制面拥有 `execution_work`、`node_executions`、`node_event_receipts` 等执行证据，业务端拥有
IssueRun 的 phase/Thread 状态和 `thread_entries`。业务端通过声明与钩子接缝协作，避免分别重建派发器与状态机。

## 工作声明到派发登记

业务端在事务中声明 `agent_session` 或 `deliver_revision`，同时固定输入及 Workspace/sandbox/Node 目标。
交付对象键按 tenant/run/work 身份固定；同一 work 重试使用原键，新 work 有自己的键。

Controller 领取未登记工作后，先登记再向 Node 投递。登记重新检查当前 Workspace 准入、运行 maintenance
绑定及 Node/sandbox 身份；带模型绑定的会话还检查当前 Node 的模型代理能力。`node_executions` 写入与
`execution_work.execution_id` 绑定在同一事务完成。同一执行、Node 和输入的重放返回原登记，矛盾输入拒绝。

领取、登记与 Node 受理是不同事实。缺少回复不能证明外部执行不存在；未登记撤回和已登记未知执行的
业务处理由相应生命周期决策定义，本文不从超时推导新的终态。

## Thread、终态与同事务钩子

Node 事件先按执行身份与连续序号接管，重复事件必须与原收据一致。会话终态使用同一事件序列，不能
越过尚未接管的 Thread 事件；Controller 在 Cloud 提交之后才确认事件。

控制面在接管事务中调用五个命名接缝：`threadEventsTakenOver`、`sessionEnded`、`runWorkspaceSettled`、
`runWorkspaceDeleted` 和 `deliverySettled`。`BindBusinessHooks` 在服务启动时连接业务适配器；适配器使用
传入的 `*sql.Tx` 及其所属 transaction，不另开事务，不在钩子内做外部 I/O。

Thread 条目、业务转换和后续工作声明与对应执行事实一起提交或回滚。钩子错误回滚接管，Node 事件保持
可重放；业务状态由证据推进，而不是由 Controller 内存中的下一步决定。

## Revision 的外部验证与原子写回

交付输入先固定对象键、基础提交和 Revision ref。上传授权或恢复下载授权由 Cloud 校验仍有效的执行、
Workspace 与运行绑定后临时签发，URL 和签名不写入持久执行输入。

收到成功交付声明后，Cloud 先校验服务身份、租约、执行身份、对象形状及其与冻结输入的一致性，再在
事务外检查对象存在、大小与 SHA-256。网络故障返回可重试错误，保留未接管状态；明确对象不匹配则记录
Cloud 自身的验证失败，保留原 Node 声明。

回到接管事务后，Cloud 再检查输入与当前运行/Workspace/Node/维护绑定。原 Node 结果、收据、验证结果、
有效 Revision 和 `deliverySettled` 业务转换在同一事务提交。已接管结果重放不重复进行外部验证或结算。
`saved/unchanged` 在有效 Revision 登记后才交给业务端；未配置对象存储时，沿已有接缝结算为明确的 skipped。

## 恢复与 Workspace 释放

恢复依赖持久工作、登记、输入和收据，Controller 按稳定身份查询与重投递。重复与响应未知不会改变输入，
迟到结果仍需通过当前身份和状态边界；临时授权刷新也不扩大已接受的对象范围。

运行 Workspace 的建删复用已有 operation 与 runtime maintenance 接缝。交付结算后由业务端声明 owned
delete，删除成功再经 `runWorkspaceDeleted` 推进运行收尾；Project 忙时保留声明与后续重试责任。主 Workspace
与其他运行的存储由各自身份和生命周期持有。

## 验证入口

系统承诺与证据状态见[控制面核心用例](../../../test-cases/cloud/controller-integration/agent-run-executions-and-thread.md)
及相关领域用例。阶段验收经过与历史结论使用固定提交或 circus 归档引用；本文不维护进度、运行 ID、
CI 次数或现场报告，也不据低层验证升级跨仓库的覆盖状态。
