---
status: implemented
---

# Project 必须给出具体默认分支，Cloud 不读取远端仓库解析 HEAD

> 后续已实现决策：[普通停止与管理员强停](20260927-safe-stop-and-administrative-force-stop.md)继承本文分支规则并扩展生命周期控制，后端状态为 `implemented`。强停 UI 与完整故障验收另由后续 ADR 承接；本文保留原状态。

创建 Project 时 `defaultBranch` 由可选（缺省为 `HEAD`）改为必填，且不能是 `HEAD`；缺省、空白或 `HEAD`
返回 `400 default_branch_required`，不写任何记录。isolated Workspace 的 `baseRef` 为 `HEAD` 时存为所属
Project 的默认分支。Cloud 不访问远端仓库，因此下发给 Controller 的 `requestedRef` 总是 desktop Node 接受的
具体分支名。

本 ADR 处于 `implemented`，由 ora-space/cloud#31 落地（`internal/core/public.go`、OpenAPI 契约与前端创建对话框），
验证证据见[运行时 Workspace 生命周期核心用例](../../../test-cases/cloud/operation/workspace-runtime-lifecycle.md#a-workspace-never-stores-head-as-its-requested-ref)。
方案由 Eric 在项目讨论中选定（2026-09-26，“要求显式分支”）。兼容性边界：公开 HTTP 契约（`defaultBranch` 变为必填，
属于有意的破坏性修改）与已持久化的 `projects.default_branch`（不迁移、不改写）。

## 继承与修改

| 前序约定（[根决策](0-workspace-runtime-follows-desktop-node.md)） | 本文处理 |
|---|---|
| Workspace 的 `requestedRef` 取自 Project 默认分支 | 继承，不变 |
| 默认分支为 `HEAD` 时 Cloud 原样下发，是否接受留给 Node 的 clone 决策（未解决的问题） | 修改：Cloud 不再接受 `HEAD` 作为默认分支（D1）；Node 协议拒绝 `HEAD` 的规则不变 |
| isolated Workspace 的 `baseRef` 原样成为 `requestedRef` | 修改：`HEAD` 解释为 Project 默认分支（D2），其余不变 |

## 决策

### D1：创建 Project 必须给出具体默认分支

- 规则：`defaultBranch` 缺省、去掉空白后为空或等于 `HEAD` 时返回 `400 default_branch_required`；其他值沿用原有
  ref 校验。检查在凭据引用校验之后、写入之前，失败不产生 Project、Workspace 或 operation。
- 为什么：desktop Node 协议只 clone 字面分支名，`HEAD` 会让 clone 步骤以 `external_failure` 永久阻塞，
  而 Project 已经建好、用户无从得知原因。在入口拒绝，调用方立刻知道要填什么。
- 为什么 Cloud 不自己解析：解析需要 Cloud 进程向用户给出的任意地址发起请求（等同 SSRF 入口，需要额外的地址
  过滤与超时策略），且 Cloud 只持有凭据引用、不持有密钥，私有仓库和 ssh 仓库本来就无法匿名读取。前端创建对话框
  默认填 `main`，显式填写的成本很低。

### D2：isolated Workspace 的 `baseRef: HEAD` 指 Project 默认分支

- 规则：`baseRef` 为 `HEAD` 时存为 Project 的 `default_branch`；该值本身仍是 `HEAD`（本决策之前创建的
  Project）时返回 `400 default_branch_required`。
- 为什么：“基于默认分支开一个 Workspace”是自然的表达，Cloud 已有具体答案，无需网络；旧 Project 没有答案，
  只能要求调用方写明分支。

## 不变量

1. 本决策之后创建的 Project 与 Workspace 不存储 `HEAD` 作为默认分支或 `requestedRef`。
2. 被拒绝的创建请求不写入任何记录。
3. Cloud 不为解析分支访问远端仓库。
4. 已存储的 `projects.default_branch` 不被迁移或改写。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
|---|---|
| 创建时匿名读取远端 HEAD（smart HTTP ref 广告） | 用户给出的 URL 让 Cloud 进程发起出站请求，需要自建地址过滤；私有与 ssh 仓库仍要显式分支，规则分两套（D1） |
| Controller 或 Node 在 clone 时解析 | Node 协议明确只接受字面分支，Controller 不持有仓库访问；改协议超出本次范围 |
| 迁移把旧 `HEAD` 改写为 `main` | 猜测会把不存在的分支写进权威数据，违反“不静默改写持久状态”（不变量 4） |

## 本决策未解决的问题

- 本决策之前以 `HEAD` 创建的 Project：其 Workspace 的 clone 仍按原规则阻塞，需要重新创建；阶段一没有生产数据。
- 创建后修改默认分支：Project 目前不允许修改 `defaultBranch`，是否开放留给后续决策。

## 落地

cloud#31 在一次变更中完成：`createProject`/`createWorkspace` 校验、OpenAPI 标记 `defaultBranch` 必填并重新生成
`api/openapi.json` 与前端客户端、前端创建对话框要求非 `HEAD` 分支、集成测试
`TestProjectsNameAConcreteDefaultBranch`。保持不变的行为：ref 校验、租户级 clone 请求对 `HEAD` 的拒绝。
