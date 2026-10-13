# 运行时控制 UI 与联合故障验收

依据[后续 ADR](../../../decisions/cloud/execution-admission/20261013-runtime-control-ui-and-fault-acceptance.md)。
三处 UI 为 approved 待交付义务；后端局部证据保留在各领域原用例，不重复标记为完整 UI/故障证明。

## Operator Controls Must Recheck Authority After Disconnection

- **风险：** acquiring 或旧 session 被界面视为有效，恢复时盲目重放写入。
- **前置状态：** 同用户两页面、占用/无权限、续期将到期、网络断开或成员失效。
- **触发：** 获取、续期、释放、断开及恢复。
- **必须成立：** held 确认后才开放交互，断开停发，恢复重查；释放后等待责任结清，冲突与权限提示区分。
- **禁止结果：** 第二页面共享旧资格、离线自动重放、到期立即显示空闲。
- **验证义务与证据：** Missing：真实浏览器两页面、响应延迟/丢失与服务端状态联合验证。
- **决策依据：** D1。

## Force Stop Confirmation Must Preserve Scope and Pending Responsibility

- **风险：** 误强停、接受即显示终止、响应丢失制造第二意图。
- **前置状态：** 当前管理员/普通成员/跨租户用户，两个运行时及一个在途任务。
- **触发：** 普通停止、确认强停、响应未知、关闭页面与重查。
- **必须成立：** 展示具体目标和未保存风险，原因/版本/稳定幂等输入完整；服务确认终止前保持 pending。
- **禁止结果：** stop 自动强停、重新请求换键、扩大目标、自动启动/夺锁/删卷。
- **验证义务与证据：** Missing：浏览器确认与真实服务意图/目标对象联合验收。
- **决策依据：** D2。

## Runtime Views Must Separate Shared Summaries From Restricted Content

- **风险：** 我的运行时或管理员过滤让其它成员获得受限详情。
- **前置状态：** 创建者、普通他人、管理员、unknown creator 及已降级用户。
- **触发：** 列表、详情、直接链接、重试、角色变化及事件刷新。
- **必须成立：** 安全概况共享，服务权限决定内容/操作，失效后清除受限视图；不出现未批准共享授权。
- **禁止结果：** 共享卡片带命令/路径/输出，按钮状态充当权限或 unknown creator 对成员开放。
- **验证义务与证据：** Missing：真实角色矩阵浏览器路径；HTTP 局部证据不替代此义务。
- **决策依据：** D3。

## Unknown Execution and Termination Must Block Handoff Across Real Fault Boundaries

- **风险：** 接受丢失/重启/旧队列/迟到 ensure 后出现第二写者，升级丢失执行或创建者证据。
- **前置状态：** ADR D4 各行所述多实例、未知接受、在途执行、软删除历史或新旧组件组合。
- **触发：** 精确故障注入与恢复后交接/强停/升级/装卸。
- **必须成立：** 每行均核对稳定责任、真实文件/进程/容器/卷及允许结果；未确认继续阻止交接。
- **禁止结果：** 注入 Node 确认被当作实际终止、模拟器替代真实装卸、修改版本恢复旧实例或抹去缺口。
- **验证义务与证据：** Partial：现有直接局部证据见以下稳定导航，ADR D4 要求的完整矩阵仍未覆盖。

| 风险 | 保留的核心用例 |
| --- | --- |
| 原子独占、到期/撤回 | [独占控制](exclusive-runtime-control.md) |
| 接受丢失、旧消息、身份、升级/回滚 | [跨进程交付](../controller-integration/fenced-runtime-control-delivery.md) |
| 终止、迟到 ensure、数据保留 | [安全停止/强停](../operation/safe-stop-and-administrative-force-stop.md) |
| 历史创建者及运行时隔离 | [使用权限](../workspace/runtime-workspace-use-permissions.md) |
| 最新插件期望与真实装卸 | [插件维护](../plugin-marketplace/admin-selection-and-idle-maintenance.md) |

- **决策依据：** D4、D5；每项完整证据达到 Covered 后才能完成本后续 ADR。
