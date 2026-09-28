# minicloud 云端形态 clone 闭环验收

依据[云端形态决策](../../../decisions/minicloud/runtime/20260922-cloud-hosted-clone-loop.md)与
[minicloud 退役决策](../../../decisions/minicloud/runtime/20260928-retire-minicloud.md)。启动器及其云端形态已删除，
原用例“Cloud Mode Keeps Authorities And State Apart”的被测对象（启动器生成的配置与目录）不再存在；Controller
云端模式不建本机状态的证据保留在 [Controller API 边界核心用例](../../controller/api-boundary/controller-hosted-client-api.md#generated-cloud-contracts-preserve-the-boundary)。
经 Cloud 的闭环由 cluster 仓库的 Compose 栈承担，下列用例改为约束该链路。Cloud 公开入口与契约事务的证据见
[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md)，
Controller 适配器证据见[持久协调双适配器核心用例](../../controller/persistence/coordination-store-adapters.md)。

## Cloud Hosted Clone Completes Once Across Controller Kill

- **风险**：经 Cloud 承载的闭环在 Controller 被强杀后重复 clone、丢失结果，或结果与磁盘不一致。
- **前置状态**：cluster Compose 栈：真实 PostgreSQL、Cloud server 与 Gateway，云端模式的 `ora-controller`，
  Sandbox Server 创建的沙盒中的 Node 与真实 Git。
- **触发**：经 Gateway 登录与租户提交 clone；以同一 `requestId` 重复提交；在登记派发后强杀 Controller 并重启。
- **必须成立**：查询最终为 `succeeded{path, commit}` 且沙盒内 HEAD 与 `commit` 一致；重复提交返回原操作；
  强杀后 Node 独立完成，重启后原事件被接管，Cloud 只有一次执行，Node outbox 清空。
- **禁止结果**：第二次 clone、结果丢失、未知被展示为失败。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 真实闭环与幂等 | minicloud 云端形态曾有手工验证（desktop#610：202、同一 `operationId`、`succeeded` 且 HEAD 一致），但该形态已删除；cluster 链路尚无自动化测试 | Missing |
| 强杀恢复 | minicloud 云端形态曾有手工验证（desktop#602：`dispatch recorded` 后 `kill -9`、Cloud 只有一次执行），但该形态已删除；cluster 链路尚无自动化测试 | Missing |

决策依据：云端形态决策 D1–D3，不变量 1；退役决策 D2。
