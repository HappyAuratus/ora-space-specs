# 个人模型 Thread：输入与结束权限

决策依据：[发起者控制输入](../../../decisions/cloud/thread/20261010-personal-model-initiator-controls-input.md)。

## Only the model run initiator may append model requests

- 风险：成员和管理员通过 Thread 消耗他人的私有模型额度。
- 前置：使用个人模型绑定的运行，存在发起者、有效成员、当前管理员。
- 触发：读取 Thread、追加消息及结束。
- 必须成立：服务端仅受理发起者的追加；有效成员可读；发起者和管理员可结束；响应的 `canAppend` / `canEnd` 与实际受理规则一致。
- 禁止结果：界面禁用而服务端受理、管理员可读原始 Key、Echo 历史运行权限被无意收紧。
- 证据与义务：Core PostgreSQL 角色/账号/撤销矩阵，HTTP Thread 契约和前端成员只读测试。[受控上游浏览器证据](https://github.com/ora-space/cluster/pull/15#issuecomment-6107672919)覆盖发起者追加第二轮与结束；真实 OpenAI Completions / Anthropic Messages 的官方 OpenCode 两轮与结束正常路径也已验证，冻结模型及 `canAppend`/`canEnd` 与受理结果一致。状态 `Covered`（权限规则）；浏览器成员查看/拒绝追加和管理员结束的多身份路径在 [个人模型访问](../model-access/personal-model-access.md) 保持 `Partial`。
