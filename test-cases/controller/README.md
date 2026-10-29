# Controller 核心测试用例

本目录与 `specs/decisions/controller/` 镜像。Controller 决策进入 `approved` 后，在对应子目录
登记需要长期维护的租户边界、持久化、队列、并发、恢复、协议和跨进程用例。

| 领域 | 核心用例 | 验证义务与证据状态 |
| --- | --- | --- |
| node-management | [不可用 Node 延后与 clone 分轮等待](node-management/20261028-defer-unavailable-node-operations.md) | 2 秒宽限、Cloud 持久退避与有限原因、5 秒轮次/1 秒再领、30 分钟墙钟截止、晚到已知结果自动恢复、Unknown 阻塞、围栏与 #698/多失联/持续连接联合路径；ADR 为 approved，新增义务全部 Missing |

