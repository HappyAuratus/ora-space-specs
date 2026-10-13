# 用户 git 提交身份

决策依据：[用户 git 提交身份](../../../decisions/cloud/identity-access/20260928-user-git-commit-identity.md)。
Node 侧的环境变量注入证据见 [Node 会话执行](../../node/agent-runtime/session-execution.md)。

## Only The Owner Reads And Writes A Git Identity

- 风险：git 身份带有个人邮箱。若其他用户能读到或改写，邮箱会泄露，或 Agent 提交会以别人的名义署名；
  若并发写不受版本保护，两个标签页会静默覆盖彼此的设置。
- 前置状态：两个用户，其中一个已设置 git 身份，另一个没有。
- 触发：`GET`、`PUT`、`DELETE /api/v1/me/git-identity`，分别带正确版本、过期版本、`0`，以及重复的 `DELETE`。
- 必须成立：每个用户只看到自己的身份；未设置时返回默认身份且 `version = 0`；`PUT` 以 `0` 创建、以当前
  版本替换；`DELETE` 只在当前版本删除，身份已是默认时任何版本都按无操作成功；名称与邮箱按 D1 校验，
  数据库 CHECK 与 API 规则一致。
- 禁止结果：读到其他用户的邮箱；过期版本覆盖或删除已有身份；不合格式的名称或邮箱被保存。
- 决策依据：D1，不变量 2。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 只能读写自己的身份，另一用户只看到自己的默认身份 | `integration.TestGitIdentityAPI` | `Covered` |
| 版本：`0` 创建、过期版本 `409`、重复 `DELETE` 安全 | `integration.TestGitIdentityAPI` | `Covered` |
| 名称与邮箱的格式校验 | `core.TestGitIdentityValidation` | `Covered` |

## Session Start Fixes The Trigger User's Identity

- 风险：会话使用错误的身份（Thread 中追加消息的成员、Issue 作者），或没有身份就被派发，Node 会拒绝
  会话或以错误署名提交。
- 前置状态：一个 Agent 运行到达会话启动；触发用户未设置、已设置身份，或运行输入已携带身份。
- 触发：Cloud 构造会话启动规格。
- 必须成立：运行输入携带完整身份时原样使用；否则取触发用户已设置的身份；都没有时取默认身份
  （显示名与 `{userId}@users.noreply.ora.invalid`）。
- 禁止结果：会话启动规格没有身份；使用触发用户以外的人的身份。
- 决策依据：D2，不变量 3。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
|---|---|---|
| 默认、已设置、运行输入冻结三种来源的优先级 | `core.TestSessionStartUsesTheTriggerUsersStatedGitIdentity` | `Covered` |
| 会话启动后修改设置不影响该会话 | 无直接测试；会话启动规格写入后不再改写，由规格不可变保证 | `Missing` |
