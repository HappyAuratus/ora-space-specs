---
status: proposed
---

# 用户可在 Cloud 配置 git 提交身份，Agent 运行以触发用户的身份经环境变量提交

每个用户可以在 Cloud 设置自己的 git 提交身份（姓名与邮箱）。用户触发 Agent 运行时，Cloud 把该身份
快照进运行输入，经 Controller 下发给 Node；Node 只通过 `GIT_AUTHOR_*`、`GIT_COMMITTER_*` 环境变量注入
Agent 及其子进程，不写入仓库或全局 git 配置。未设置时使用由显示名与平台 noreply 地址组成的默认身份。

本 ADR 处于 `proposed`，继承 [identity-access 根决策](0-gateway-mediated-external-identity.md) 与
[globalUserId 决策](20260923-huawei-global-id-alias.md)。注入方式由
[Node agent-runtime 根决策](../../node/agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md) D4 承担，
运行输入的快照时点由 [IssueRun 根决策](../issue-run/0-agent-run-in-disposable-isolated-workspace.md) 承担。
兼容性边界：`users` 表只增加可空列，已有用户不受影响。

## 继承与修改

| 前序约定 | 本文处理 |
|---|---|
| Gateway 验证外部身份，Cloud 持有授权；浏览器不能自选身份 | 继承，不变；git 身份是用户自述的提交署名，不是认证身份，不参与任何授权 |
| 外部 token 与 provider 字段不进入业务表 | 继承，不变；不从 GitHub 或 IDaaS 的邮箱自动填充 |
| 日志排除不必要的个人数据 | 继承；git 邮箱属于个人数据，不写日志 |
| 用户资料只有显示名等 Cloud 维护字段 | 修改：`users` 新增 `git_author_name`、`git_author_email`（D1） |

## 决策

### D1：git 身份是用户级资料，由用户本人设置

- `GET /api/v1/me/git-identity` 返回 `{name, email, isDefault}`；`PUT /api/v1/me/git-identity`（带 `version`）
  设置，`DELETE` 恢复默认。只有用户本人可以读写，租户管理员不能代设。
- 校验：`name` 1–200 个字符、不含换行与 `<`、`>`；`email` 不超过 254 字节、形如 `local@domain`、不含空白
  与 `<`、`>`。只做格式校验，不验证邮箱归属：它只是提交署名，Cloud 不以它授予任何权限。
- 默认身份：`name` 为用户显示名，`email` 为 `{userId}@{git.noreply_domain}`；`git.noreply_domain` 是 Cloud
  配置，proposed 默认 `users.noreply.ora.invalid`。

为什么是用户级而不是租户级：同一用户在不同 Space 中通常使用同一提交身份；租户级覆盖可以后续叠加，不改变
本条的默认来源。为什么不自动取登录 provider 的邮箱：根决策不持久化 provider 返回的额外字段，且企业账号的
邮箱可能不是用户希望出现在提交历史中的地址。

### D2：运行输入快照触发用户的身份

IssueRun 创建时，Cloud 把触发用户当时的 git 身份（或默认身份）写入运行输入 `gitIdentity{name, email}`。
运行期间用户修改设置不影响正在进行的运行；其他成员在 Thread 中追加消息也不改变身份。

### D3：只经环境变量注入，Author 与 Committer 相同

Node 为 Agent 进程及其全部子进程设置 `GIT_AUTHOR_NAME`、`GIT_AUTHOR_EMAIL`、`GIT_COMMITTER_NAME`、
`GIT_COMMITTER_EMAIL` 为该身份；不修改 clone 出的仓库配置、不写 `~/.gitconfig`。Revision 的内部快照
提交使用同一身份作为 author，committer 固定为平台身份（[Node revision 根决策](../../node/revision/0-internal-snapshot-and-incremental-bundle-upload.md) D2）。

为什么用环境变量：Git 中环境变量优先于任何配置文件，Agent 或仓库里的配置无法意外覆盖；身份随进程生命周期
存在，不在 Workspace 数据中留下需要清理的配置，也不会被下一次恢复出的 Workspace 继承。代价是 Agent 若
显式 `git -c user.email=...` 或自行设置环境变量仍可改写署名，第一版不阻止。

## 不变量

1. git 身份不参与认证与授权。
2. 用户的 git 身份只能由其本人设置。
3. Agent 运行使用创建时快照的触发用户身份。
4. Node 不修改仓库或全局 git 配置来设置身份。
5. git 邮箱不写入日志。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 写入仓库本地 `git config user.*` | 实现直观；但会出现在 Revision 以外的 Workspace 数据中，且 Agent 或脚本可能修改它 |
| 以 Agent 身份提交、用户放在 `Co-authored-by` | 更清楚地标注机器提交；但用户明确希望以用户身份提交，且提交后续可能被推到用户远端 |
| 设置为必填，未设置时拒绝运行 | 身份更准确；首次使用多一道阻塞，默认身份已能区分用户 |

## 本决策未解决的问题

- 租户级或 Space 级的身份覆盖与邮箱域名约束。
- 提交签名（GPG/SSH signing）。
- 是否需要在提交信息中标注由 Agent 生成。

## 落地

迁移增加两列；公开 API、OpenAPI 与前端设置页的“Git 身份”表单；运行输入快照与 Node 注入分别随 IssueRun
与 agent-runtime 决策落地。保持不变：登录、会话与授权。
