# minicloud 云端形态 clone 闭环验收

依据[已实现的后续决策](../../../decisions/minicloud/runtime/20260922-cloud-hosted-clone-loop.md)。启动器证据核对
desktop 主干 `aa635a12`（含 desktop#607），`deno test -A scripts/run-minicloud.test.ts` 已在该提交上本地重跑；
经 Gateway 提交的流程来自 desktop#610（`cf50128020391389805e2ceda281bb08027b7f86`，尚未合并）。Cloud 公开入口与
契约事务的证据见[Cloud 内部控制契约核心用例](../../cloud/controller-integration/cloud-owned-internal-grpc-contract.md)，
Controller 适配器证据见[持久协调双适配器核心用例](../../controller/persistence/coordination-store-adapters.md)。

## Cloud Mode Keeps Authorities And State Apart

- **风险**：云端形态复用本机状态目录或 SQLite，使 Node 的持久记录被两个权威认领；或启动器生成带 JSON
  接口、密钥的配置。
- **前置状态**：同一 checkout 先后以本机与云端形态初始化；用户修改过 Cloud 地址。
- **触发**：`--cloud` 初始化与重复初始化；以不一致的持久模式启动已有配置。
- **必须成立**：状态位于独立的 `~/.ora/cloud/<digest>/`；Controller 配置为 `persistence: cloud`、无 `api` 段、
  无凭据，启动参数只含 `--config` 与 `--single-node`；不生成 `client.json` 与 `vite/`；重复初始化保留用户修改的
  Cloud 地址；持久模式与启动模式不一致时拒绝启动。
- **禁止结果**：两种形态共用目录、云端形态启动 Vite 或 JSON 接口、生成或分发密钥、Cloud 不可达时退回本机形态。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 云端配置与目录 | `scripts/run-minicloud.test.ts` 的 "cloud initialization coordinates through Cloud without a JSON surface or frontend" 与 "cloud initialization preserves an edited Cloud endpoint" | Covered |
| 模式不一致拒绝 | 启动器校验已有 `controller.json` 的持久模式，目前没有直接测试 | Missing |
| Controller 不开监听、不建本地状态 | `apps/ora-controller/tests/composition.rs::cloud_persistence_serves_no_surface_and_creates_no_local_state` | Covered |

决策依据：D1，不变量 2、3。

## Cloud Hosted Clone Completes Once Across Controller Kill

- **风险**：经 Cloud 承载的闭环在 Controller 被强杀后重复 clone、丢失结果，或结果与磁盘不一致。
- **前置状态**：本机真实 PostgreSQL、Cloud server 与 Gateway，云端模式的 `ora-controller --single-node`，真实 Node、
  host／guardian 与 HTTPS Git。
- **触发**：经 Gateway 开发登录与租户提交 clone；以同一 `requestId` 重复提交；在登记派发后强杀 Controller 并重启。
- **必须成立**：查询最终为 `succeeded{path, commit}` 且磁盘 HEAD 与 `commit` 一致；重复提交返回原操作；
  强杀后 Node 独立完成，重启后原事件被接管，Cloud 只有一次执行，Node outbox 清空。
- **禁止结果**：第二次 clone、结果丢失、未知被展示为失败。
- **验证义务与证据**：

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 真实闭环与幂等 | desktop#610 描述的手工验证：经 Gateway 提交返回 202、重复提交得到同一 `operationId`、结果 `succeeded` 且 HEAD 一致；无自动化测试 | Partial |
| 强杀恢复 | desktop#602 描述的手工验证：`dispatch recorded` 后 `kill -9`，重启后接管重放事件、Cloud 只有一次执行；无自动化测试 | Partial |

决策依据：D1–D3，不变量 1。
