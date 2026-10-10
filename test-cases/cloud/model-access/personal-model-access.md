# 个人模型访问：归属、冻结、撤销和密钥隔离

决策依据：[个人连接与运行代理](../../../decisions/cloud/model-access/0-personal-model-connections-and-run-scoped-proxy.md)。
状态对应直接证据范围；生产 CLI 与 Bluezone 的真实联调不能由下层替身测试代替。
2026-10-10 两协议的受控上游浏览器/API 正常链路已通过，安全元数据与剩余义务见
[M4 验收记录](../../../implementation/opencode-m4.md)。整体 M4 仍未完成，相关 ADR 保持 `approved`。

## Personal model configuration must remain owner scoped

- 风险：另一个成员或管理员读取、修改个人连接或 Key。
- 前置：两个有效用户，各有个人连接。
- 触发：公开 CRUD、默认选择、凭据写入及跨用户重试。
- 必须成立：归属来自已验证身份；响应仅含公开字段；资源版本和幂等性由 PostgreSQL 保障。
- 禁止结果：请求体选定用户、读取 Key/密文、重复写入或覆盖并发修改。
- 证据：`cloud/internal/core/model_connections_db_test.go` 真实 PostgreSQL；HTTP 边界及严格结构测试；两协议浏览器记录直接覆盖发起者表单配置、默认模型和保存后密码输入清空。状态 `Partial`，多身份及完整配置矩阵的浏览器路径仍待直接验收。

## Model bindings must freeze inputs and never revive after revocation

- 风险：排队期间编辑配置改变已创建运行，或重新启用连接复活旧授权。
- 前置：OpenCode 运行已绑定个人连接；之后替换 Key、停用连接或清除 Key。
- 触发：编辑配置或密钥后再次取得/使用旧授权，或重新启用连接。
- 必须成立：编辑仅影响新运行；撤销绑定永久失效；缺默认配置的评论和运行原子回滚。
- 禁止结果：旧运行读取新 Key、缺配置运行无限等待、重新启用后旧绑定再次获权。
- 证据：`cloud/internal/core/model_connections_db_test.go` 冻结、原子回滚、清除/停用后重新配置矩阵。状态 `Covered`（数据库义务）。

## Model grants must follow confirmed runtime scope and current authorization

- 风险：陈旧或错误证书目标申请、续期或使用另一个运行的授权。
- 前置：已登记会话执行、当前运行绑定及确认过的 Node。
- 触发：申请、续期和使用临时授权。
- 必须成立：证书用途/租户/Workspace/代次和数据库目标匹配；当前账号、成员、能力、会话和维护权均有效。
- 禁止结果：调用者选定 Node 身份、未确认绑定获得授权、失效账号/成员续期、仅持 WSS 证书调用。
- 证据：Cloud 的真实 PostgreSQL 授权矩阵；Desktop 的真实 mTLS grant/renew/revoke 测试；cluster 证书用途测试。状态 `Partial`，全栈故障恢复尚缺直接证据。

## Forwarding must preserve protocol streams and cancel revoked requests

- 风险：代理泄露认证头、改用另一模型、重试生成或撤销后继续调用。
- 前置：一条有效运行授权，上游可持续发送 SSE。
- 触发：调用冻结模型、上游错误、撤销绑定或超时。
- 必须成立：两种协议和工具数据持续转发；加入冻结的认证与模型；撤销取消上游；HTTP 失败不回显诊断正文。
- 禁止结果：跨模型请求、客户端指定目标、跟随重定向、访问私网或把真实 Key 返回客户端。
- 证据：`cloud/internal/modelgateway/service_test.go`、`transport_test.go` 使用真实 TLS HTTP 边界验证转发、撤销、错误过滤与地址策略；`events_test.go`、`framing_test.go`、`completion_test.go` 直接覆盖 HTTP 200 内错误、SSE 格式及阻塞写入/正常结束竞态。状态 `Partial`，实际 OpenCode 受控上游两协议已通过，Bluezone 仍需直接验收。

## Personal model usage must remain controlled by the run initiator

- 风险：协作成员通过追加消息消耗另一个用户的个人模型额度。
- 前置：有模型绑定的 Thread；发起者、其他有效成员和当前管理员。
- 触发：读取、追加消息和结束运行。
- 必须成立：仅发起者追加消息；其他成员可读取；发起者及管理员可结束；终态均不可追加。
- 禁止结果：界面禁用但服务端仍受理、管理员默认获得 Key 权限、Echo 历史权限被一并改变。
- 证据：Cloud PostgreSQL 权限矩阵与前端 Thread 测试。两协议浏览器正常路径补充发起者的追加和结束，状态 `Covered`（规则与界面义务）；多身份浏览器路径仍为 `Partial`。

## Real OpenCode must modify the checkout and deliver a secret-free Revision

- 风险：只有 Echo 或模拟器跑通，实际 CLI、工具调用、成果保存及释放未接通。
- 前置：独立真实 Compose 栈，安装固定版本官方 OpenCode，用户通过页面配置连接和模型。
- 触发：登录、创建项目并 clone、@OpenCode、追加轮次、结束会话。
- 必须成立：真实 CLI 读取/修改 checkout，JSONL 与 bundle 保存并校验，Git 作者归发起者，运行 Workspace 释放。
- 禁止结果：以替身代替 CLI、Key/临时令牌进入 Git/日志/截图、未登记成果就宣称完成。
- 证据：最终 `Cloud 5b6b5ec`、`Desktop f8536f9d`、`Cluster a8146c7` 的独立 M4 栈以 `opencode-m4-receipt-final-browser-openai.json` / `opencode-m4-receipt-final-browser-anthropic.json` 直接覆盖开发登录、页面配置、clone、Issue、真实 CLI 两轮会话及结束；对应的两个 `verified-*` 文件对同一浏览器 Run 核验 Revision、Git 作者和运行 Workspace 容器/卷释放。`opencode-m4-api-receipt-final.json` 补充两协议 API 链路，模型 ID 为 `m4/tool-model`。受控上游正常浏览器及交付义务已有直接证据；整体状态仍为 `Partial`，用户页面新 Key 的 Bluezone 实际服务验收尚未执行，不能标记 M4 完成。历史成功和失败记录不代替本次最终源码验证。
