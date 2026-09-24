---
status: implemented
---

# Space API 与 Space 级失效事件流：REST 为权威，SSE 只做失效通知

> 历史决策：REST 为权威、SSE 只做失效通知仍适用；下文的空间创建、归档及空间成员写 API 已由 [20260923-one-space-per-tenant](../tenancy/20260923-one-space-per-tenant.md) 取代。当前使用租户级成员管理和一租户一空间接口。

Cloud 公开 API 新增 `/api/v1/tenants/:tid/spaces` 路由族（Space CRUD、成员 upsert、Space 作用域
Project 集合）与 SSE 端点 `/spaces/:sid/events`。REST 保持唯一权威状态来源；SSE 只投递轻量失效
通知（`{type, spaceId, projectId?, version?}`），客户端收到后 invalidate 对应查询并 refetch REST。
事件在数据库事务提交成功后才广播，被拒绝的变更绝不发布；连接建立前按与 REST 读取完全相同的
路径校验 Space membership，非成员不建立连接（当前为 403）。已建立的连接在每次投递前重新校验凭证与成员身份，停用成员的连接随下一条通知关闭。单实例部署使用进程内内存 hub，无持久化、
无回放、无跨实例投递。

本决策已实现：`/spaces` 白名单路由、OpenAPI `Space*`/`SpaceEvent` schema、orval 生成客户端、
`internal/core/hub.go` 与 SSE handler、前端 `useSpaceEvents`。`status: implemented` 表示规则与
实现同步。既有 19 条公共路由、双凭证模型与错误契约风格不变；WebSocket、CRDT/Yjs、outbox
事件持久化不在本决策范围。

## D1：REST 路由与输入边界

新增路由全部进入 `router.Routes()` 白名单，携带字段白名单并沿用严格 JSON 解码（拒绝未知字段）：

| 路由 | 语义 |
| --- | --- |
| `GET /spaces` | 只返回调用者已加入且未归档的 Space（含角色） |
| `POST /spaces` | 原子创建 Space 与首个 owner；slug 归一化后校验 |
| `GET/PATCH/DELETE /spaces/:sid` | 读取需成员；PATCH 仅 name/description（admin/owner）；DELETE 归档（owner） |
| `GET/PUT /spaces/:sid/members/:uid` | 成员列表需成员；PUT upsert 需 admin/owner，授 owner 需 owner |
| `GET/POST /spaces/:sid/projects` | Space 作用域 Project 集合，创建走既有异步 operation 语义（202） |

错误码实际集合（对齐既有小写下划线风格）：`space_slug_conflict`(409)、`space_role_required`
(403)、`space_last_owner`(409)、`invalid_slug`(400)、`invalid_member`(400)；版本校验复用
`version_required`(428)/`version_conflict`(409)。非成员、已归档、不存在统一映射为 `404
not_found`，不区分「不存在」与「无权限」，避免泄露目标 Space 的存在性。计划阶段的
`space_access_denied` 码未采用：与既有「owner 范围外 404」策略保持一致。

## D2：OpenAPI 契约

`internal/contract/openapi.go` 新增 `Space`、`SpaceListItem`（含 `role`）、`SpaceMember`、
`SpaceMemberListItem`（含 `displayName`）、`SpaceEvent` schema；`Project` 与内部
`ControllerProject` 增加 `spaceId`；space 路由归入 `spaces` tag；409 错误描述补充
`space_last_owner` 与 `space_slug_conflict`。`task frontend:generate` 重新生成
`api/openapi.json` 与 `frontend/src/api`（含 `spaces/spaces.ts`），生成物禁止手改，契约漂移由
`task frontend:check` 把关。

## D3：SSE 事件流

`GET /api/v1/tenants/:tid/spaces/:sid/events` 是独立 handler（不在路由白名单内）：先做与 REST
相同的双凭证验证（gateway service JWT + `caller` 绑定的 user JWT），再以同一授权语义执行一次
Space 读取——成员校验通过才打开 `text/event-stream` 连接，当前非成员返回 403。连接存续期间，每次投递前重新执行凭证与成员校验；成员停用或凭证失效时关闭连接，不投递触发关闭的通知。

事件集合为 5 种失效通知：`space.updated`（PATCH/归档 Space）、`space.member_updated`（成员调整、内网目录添加、邀请兑换或申请批准）、`project.created`（含 legacy tenant 级创建）、`project.updated`（PATCH Project name）、
`project.archived`（删除操作创建、lifecycle 进入 deleting 时）。`POST /spaces` 本身不发布事件。
事件只携带失效信息，不携带业务状态；客户端按类型 invalidate 对应 query key 后向 REST 获取
权威状态。

## D4：提交后广播与单实例 hub

`Store.Public` 在事务成功提交后才 `Publish` 全部待发事件，事务失败或 `reject` 的变更不产生
任何事件；加入请求的幂等重放不重复广播。`SpaceHub` 是进程内 per-space 订阅表：订阅得到缓冲 8 的 channel，`Publish` 非阻塞，
慢订阅者丢弃过期通知（由下一次变更或显式刷新兜底）；无持久化、无回放。Cloud 事务经
`pg_advisory_xact_lock(67420911)` 全局串行化且当前单实例部署，内存广播与提交顺序天然一致。
多实例部署时以 broker（Redis/NATS）替换 hub 实现，保持同一 Subscribe/Publish 边界与提交后
发布语义；outbox 持久化仅在出现离线回放、审计或可靠投递需求时引入。

## D5：前端消费与凭证

前端 `useSpaceEvents` 订阅事件流并在边界校验 payload（`parseSSEFrames` 丢弃畸形帧），只触发
invalidate，绝不把事件内容当状态。真实账号凭证由 devgateway（`cmd/devgateway`，仅本地开发的
凭证桥，永不部署）签发双 JWT，浏览器按 tab 存于 sessionStorage，以支持多账号并行验收；生产
由真实 gateway 承担同一交换。mock 会话与真实会话由 `CurrentSpaceContext` 区分，Space 切换后
失效 Space 作用域缓存，防止旧 Space 数据闪现。

## 不变量

1. REST 是唯一权威状态来源；事件只触发 refetch，不携带业务状态。
2. 事件只在事务提交成功后发布，绝不发布未提交或已回滚状态。
3. 非成员不能建立事件流；成员失效后，既有连接不能继续接收事件，判定与 REST 读取同一授权路径。
4. 路由白名单 + 字段白名单 + 严格解码；服务端拥有的身份与范围字段不接受客户端输入。
5. 事件丢失最多造成一次延迟刷新，不破坏一致性。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| WebSocket | 后端无现成 WS 设施，project 级低频通知下过度设计 |
| CRDT / Yjs / OT | Project 状态变更频率低，状态同步方案复杂度远超需求 |
| 事件携带完整业务状态 | 客户端直接消费会与 REST 权威产生分叉，payload 膨胀 |
| 事件在事务内发送 | 未提交状态泄漏给订阅者，违反权威一致性 |
| outbox / 持久化事件表 | MVP 无离线回放与审计需求；多实例时再引入 |
| 仅在建立连接时校验成员 | 可省去每次通知的查询，但成员停用后旧连接继续获知空间变更 |

## 落地与验收

实现位于 `internal/api/router/router.go`（白名单 + SSE handler）、`internal/core/hub.go`、
`internal/contract/openapi.go`。核心行为由 `integration/space_events_test.go` 的
`TestSpaceEventsAuthorizationAndCommitOrder`、`TestRemovedMemberLosesOpenEventStream`、`TestNewMembershipNotifiesOnlineMembers` 直接验证（非成员订阅失败、停用后旧连接关闭、成员加入后通知在线成员、拒绝的变更零事件、
提交后事件到达且类型与变更一致），前端事件解析、停用后重连与凭证隔离由 `spaces.test.tsx`、
`cloud-session.test.ts` 覆盖。核心用例见
[Space API 与事件流](../../../test-cases/cloud/api-boundary/space-api-and-events.md)。
