# Controller API 边界验收

依据[已批准 ADR](../../../decisions/controller/api-boundary/0-controller-hosted-client-api.md)。
入口迁移证据基于 [desktop#595](https://github.com/ora-space/desktop/pull/595) 的
`e22bfdd063032d22532c2a007320072b374aaed9`；该 PR 尚未合并。已核对测试源码与绿色 CI，
本次未重跑 Linux 进程测试。proto／gRPC 属于后续阶段，仍缺直接证据。

## Transport Validation Preserves Private State

- **风险**：非法参数留下半初始化状态，或 socket 恢复覆盖用户文件。
- **前置状态**：部署配置、未初始化数据目录，以及普通文件、符号链接或活跃 socket。
- **触发**：默认 TCP、显式非回环、Unix socket 或不兼容参数组合启动。
- **必须成立**：默认 `127.0.0.1:4820`；显式非回环告警；非法组合在打开数据库前拒绝；
  socket 为数据目录直接子路径且为绝对路径，遵循私有创建与恢复规则。
- **禁止结果**：隐式监听所有接口、参数失败后创建数据库、覆盖普通文件、符号链接或活跃 socket。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 监听与拒绝时机 | `apps/ora-controller/src/cli.rs::flags_map_to_explicit_transports` 验证默认值与参数拒绝；缺真实入口拒绝无副作用及非回环日志断言 | Partial |
| Unix 路径保护 | `apps/ora-controller/tests/api.rs::http_acceptance_is_idempotent_and_survives_service_restart` 覆盖越界拒绝与合法 Unix HTTP；缺其余文件保护入口断言 | Partial |

决策依据：D2，不变量 3。

## Api Results Follow Durable Controller Records

- **风险**：HTTP 适配另造事实，或回复早于持久接受。
- **前置状态**：真实 Controller、Node，可注入存储失败和中断回复的调用方。
- **触发**：提交、重传、冲突、查询缺失／无终态记录，并中断调用方连接。
- **必须成立**：持久提交后才接受；相同请求返回原执行；冲突不覆盖；缺失与无终态区分；
  存储失败为暂时不可用；目标来自配置 `api.node_id`；调用方退出不取消已接受执行。
- **禁止结果**：调用方指定 Node、重复执行、存储失败伪装 clone 失败、断连取消执行。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 接受、查询与目标边界 | `apps/ora-controller/tests/api.rs::http_acceptance_is_idempotent_and_survives_service_restart` 覆盖幂等、冲突、缺失／待完成和配置目标；`apps/ora-node/tests/standalone/repository_minicloud.rs` 真实写锁覆盖 503 且无接受记录 | Covered |
| 断连与恢复 | `apps/ora-node/tests/standalone/repository_minicloud.rs::minicloud_http_clone_survives_controller_kill_and_replays_original_intent` 截断 202 回复并重启重传，保留原执行及唯一 Run | Covered |

决策依据：D1、D3，不变量 1、2、6。

## Generated Cloud Contracts Preserve The Boundary

- **风险**：两侧契约漂移、破坏 `v1` 或在 Controller 复制租户授权。
- **前置状态**：Controller 拥有 proto；Cloud 固定 tag；双方提交生成代码。
- **触发**：变更契约、重新生成和编译，并以真实 Controller 执行 Cloud 集成测试。
- **必须成立**：生成无漂移；破坏性修改新增 `v2` 并保留 `v1`；错误枚举映射唯一；
  proto 不含租户、用户、成员或角色字段；不改变 Controller 拨入 Cloud 领取操作的方向。
- **禁止结果**：同一 DTO 的手写并行副本、解析错误字符串、以编译通过替代行为验证、Controller 做租户筛选。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 生成与版本门禁 | 待补两侧漂移检查、buf lint／breaking 与固定 tag 编译 | Missing |
| 协议与错误行为 | 待补 Cloud 使用真实 Unix Controller 的调用和结构化错误测试，以及 TCP 上 JSON／gRPC 共用监听器验证 | Missing |
| 授权与调用方向边界 | 待补契约字段评审及 Cloud 授权／领取集成证据 | Missing |

决策依据：D4、D5，不变量 4、5、7。
