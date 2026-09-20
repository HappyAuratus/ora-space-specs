# Space API 与事件流

决策依据：[Space API 与 Space 级失效事件流](../../../decisions/cloud/api-boundary/0-space-api-and-event-stream.md)。
以下用例通过真实 PostgreSQL 集成测试（`integration/space_events_test.go`、
`integration/project_space_test.go`）与前端 Vitest 验证。

## Non-members cannot open an event stream

- 风险：猜测 spaceId 即可订阅他人 Space 的事件，泄露变更情报。
- 前置状态：carol 是 tenant 成员但非该 Space 成员。
- 触发：carol 携带合法双凭证 `GET /spaces/:sid/events`。
- 必须成立：404 且连接不建立（与 REST 读取同一授权路径）。
- 禁止结果：200、或空流连接保持。
- 验证义务与证据：**Covered** — `integration TestSpaceEventsAuthorizationAndCommitOrder`
  断言非成员订阅 404。

## Events are published only after commit and never for rejected mutations

- 风险：事件在事务提交前广播，订阅者看到未提交状态；或失败变更也发布，客户端 refetch
  后与通知矛盾。
- 前置状态：bob（成员）已订阅事件流。
- 触发：owner 用错误 version PATCH Space（409）；随后 bob 创建 Project（202）。
- 必须成立：被拒绝的 PATCH 不产生任何事件；提交成功的事件随后到达。
- 禁止结果：订阅者先收到与失败变更对应的事件、或成功变更无事件。
- 验证义务与证据：**Covered** — `integration TestSpaceEventsAuthorizationAndCommitOrder`
  以「若失败 PATCH 错发，第一条事件将是 space.updated 而非 project.created」的顺序断言
  拒绝变更零事件、提交后事件到达。

## Events carry invalidation metadata matching the committed mutation

- 风险：事件类型或负载与实际变更不一致，客户端失效错误的查询。
- 前置状态：成员订阅后依次发生 project 创建与 space 更新。
- 触发：断言收到的两条事件类型与负载，随后以 REST 重读权威状态。
- 必须成立：事件依次为 `project.created`（含 projectId）与 `space.updated`；REST 列表与事件
  指认的资源一致。
- 禁止结果：类型错序、事件携带业务状态或错误资源标识。
- 验证义务与证据：**Covered** — `integration TestSpaceEventsAuthorizationAndCommitOrder`
  断言两条事件类型，并在事件后重读列表确认权威状态一致。

## Project resources expose spaceId in the public contract

- 风险：契约缺 spaceId，前端无法建立 Space 作用域的查询键与失效映射。
- 前置状态：Space 内存在 Project。
- 触发：读取 Space 项目列表与单资源。
- 必须成立：响应含 `spaceId` 且与 Space 一致。
- 禁止结果：spaceId 缺失或错误。
- 验证义务与证据：**Covered** — `integration TestProjectSpaceScopingAndRoleGates` 断言
  列表项 spaceId；OpenAPI `Project`/`ControllerProject` schema 含 spaceId 且契约漂移由
  `task frontend:check` 把关。

## Malformed event frames are dropped at the boundary and only well-shaped events invalidate queries

- 风险：畸形网络数据直接进入 query cache，或以类型断言信任未校验的 payload。
- 前置状态：浏览器会话持有 SSE 字节流。
- 触发：解析完整帧、半帧尾、畸形 JSON、非 data 行。
- 必须成立：完整帧产出类型化事件；半帧保留为尾部；畸形与非 data 行被丢弃。
- 禁止结果：解析抛错、畸形 payload 进入 invalidate 流程。
- 验证义务与证据：**Covered** — 前端 `spaces.test.tsx` 的 `parseSSEFrames` 用例覆盖帧切分、
  尾部保留与畸形丢弃。

## Cloud credentials live per tab and never include keys

- 风险：凭证泄漏到共享存储或密钥落入前端代码，多账号验收互相串号。
- 前置状态：浏览器 tab 通过 devgateway 登录。
- 触发：存储/读取/清除会话；写入损坏或缺少任一 token 的条目。
- 必须成立：凭证按 tab 存于 sessionStorage；损坏或缺失 token 视为无会话；登录失败不落任何
  凭证；密钥只存在于 devgateway 进程。
- 禁止结果：localStorage 跨 tab 共享、损坏条目被信任、失败登录残留凭证。
- 验证义务与证据：**Covered** — 前端 `cloud-session.test.ts` 覆盖 tab 生命周期、显式清除、
  损坏条目拒绝与 devgateway 失败路径。
