# Sandbox Server 的 Docker 沙盒生命周期

依据[已实现的 Docker 沙盒生命周期决策](../../../decisions/cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md)。
本文件登记 Sandbox Server 以 Cloud 的 Substrate effects 接口创建、终止沙盒与删除 Workspace 数据的承诺；
标签路由见[Docker 标签解析用例](docker-label-resolution.md)，Cloud 侧步骤见
[运行时 Workspace 生命周期用例](../../cloud/operation/workspace-runtime-lifecycle.md)。证据来自 ora-space/cluster 仓库
（核对基线 `aef10f3`），路径相对该仓库根目录；Docker 行为由内存替身 Engine 模拟，真实 Docker 仅手工验收。
生命周期接口的认证、Cloud 许可与运行时围栏属于运行时控制决策，不在本文件登记。

## Sandbox Ensure Creates At Most One Container Per Effect And Per Workspace

### 风险

重试、响应丢失或 Sandbox Server 崩溃后重复创建容器；或者同一 Workspace 残留旧容器时又建一个，两个
Node 进程同时写同一个 Workspace 卷上的账本。

### 前置状态

Sandbox Server 以 Docker 模式运行，`-state-dir`、`-node-image`、`-controller-id` 已配置；Workspace `W`
的数据卷可能已存在（上一代沙盒留下）。

### 触发

Controller 以 effect ID `E` 发送 `PUT /effects/E`（`sandbox_ensure`，Workspace `W`），包括：首次、同请求
重发、不同请求重发、执行中进程被杀后重发、`W` 已有其他容器时发送。

### 必须成立

- 先写 journal `running` 再调用 Docker 写 API；同 ID 同请求返回同一结果，不同请求 `409`。
- 容器名 `ora-sandbox-E` 至多一个；同名冲突时 inspect 已有容器继续，不创建第二个。
- 成功结果 `sandboxInstanceId = E`、`externalId = E`、`nodeId = workspace-W`；容器带根决策三个路由标签，
  只接入沙盒网络，不发布端口，挂载卷 `ora-workspace-W`。
- `W` 已有其他 effect ID 的容器时 effect 失败为 `workspace_sandbox_live`，不创建容器。
- `GET` 不创建任何东西；journal 为 `running` 且没有进行中的 `PUT` 时按容器名查询 Docker 据实返回。
- 失败条目只带有界错误码（`node_image_missing`、`container_not_running`、`docker_failure` 等），响应为 `503`；
  同一请求再次发送会重新执行。

### 禁止结果

同一 effect 或同一 Workspace 出现两个容器；以超时或重启推断成功或失败；容器发布端口或接入其他网络；
不支持的 `kind` 写入 journal。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| journal 先于 Docker 写入、幂等与 `409` | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestJournalIsWrittenBeforeDocker` 在每次 Docker 调用前检查 journal 文件已存在；`TestEnsureIsIdempotentAndRejectsChangedRequest` 断言键序不同的同请求返回同一条目且无新 Docker 调用，不同请求 `409` | Covered |
| 崩溃恢复不重复创建 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestEnsureRecoversAfterCrashWithoutDuplicate`：容器已创建而 journal 仍为 `running`，重建服务后 `GET` 据实提升、重发 `PUT` 接管已停止容器，始终只有一个容器；`TestConcurrentEnsureOfOneEffectCreatesOneContainer` 断言并发同 ID `PUT` 只创建一次 | Covered |
| 同名冲突与 Workspace 唯一性 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestEnsureAdoptsNameConflict` 与 `TestEnsureRefusesSecondContainerForWorkspace` | Covered |
| 容器规格 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestEnsureCreatesLabelledContainer` 断言名称、五个标签、网络、镜像、卷挂载与完整 `ORA_NODE_CONFIG`，并以 Docker Resolver 解析到该容器；`sandbox-server/internal/docker/docker_test.go` 的 `TestCreateContainerSendsSpecWithoutPublishedPorts` 断言 Engine 请求体不含端口发布 | Covered |
| `unsupported_kind` | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestRejectedRequestsWriteNothing` 以 `worktree_ensure`、`storage_delete` 等请求断言 `400`、无 journal 文件且无 Docker 调用 | Covered |
| 有界失败码与失败后重试 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestEnsureFailureCodes` 断言镜像缺失、启动后退出、Docker 不可达分别为 `503` 与对应错误码，且错误码不含 Docker 原文；`TestFailedEnsureRetriesOnSameRequest` 断言失败后同一请求重发成功 | Covered |
| `GET` 不创建、`running` 据实返回 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestGetUnknownEffectIs404AndCreatesNothing`（`404`，无 journal、无容器）与 `TestReconcileLeavesUnprovenEnsureRunning`（无容器证明时保持 `running`，无写调用） | Covered |

### 决策依据

Docker 沙盒生命周期 D1–D6，不变量 1、2、3、5、7。

## Sandbox Terminate Reports Terminated Only After The Container Is Gone

### 风险

在旧 Node 仍可能写入 Workspace 卷时报告终止，Cloud 据此分配新 generation；或者迟到的 ensure 在终止
确认之后重新创建容器。

### 前置状态

Workspace `W` 有沙盒 `E` 的 ensure 记录（已成功、执行中或从未执行）。

### 触发

Controller 发送 `sandbox_terminate`（`sandboxInstanceId = E`）；可能与 `E` 的 ensure 并发，或 Docker 停止、
删除失败。

### 必须成立

- 同一 Workspace 的 effect 串行；terminate 在第一次破坏性调用前写终止墓碑（容器不存在时也写），之后针对
  `E` 的 ensure 以 `sandbox_terminated` 失败。
- 只有 Docker 确认 `ora-sandbox-E` 不存在后才返回 `terminated: true, lateEnsureFenced: true`；停止使用
  `-stop-timeout`。
- 容器 Workspace 标签与请求不一致时 effect 为 `failed`、`sandbox_scope_mismatch`，不写墓碑、不动容器；Docker
  报错或超时时 effect 为 `failed`。
- 卷 `ora-workspace-W` 保留。

### 禁止结果

容器仍存在时报告终止；终止确认后又出现 `ora-sandbox-E`；terminate 删除 Workspace 卷。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 以容器不存在为终止证据 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestTerminateReportsOnlyWhatDockerConfirms`（删除失败为 `failed`、重试成功）、`TestTerminateRemovesContainerAndBlocksLateEnsure`（成功且卷保留）、`TestReconcilePromotesTerminateOnlyWhenContainerGone`；尚缺停止超时的单独断言 | Partial |
| 墓碑阻止迟到 ensure | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestTerminateRemovesContainerAndBlocksLateEnsure` 与 `TestTerminateOfUnknownSandboxSucceedsAndTombstones` 顺序断言终止后的 ensure 以 `sandbox_terminated` 失败；尚缺并发交错的断言 | Partial |
| Workspace 不一致时拒绝 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestTerminateRejectsOtherWorkspace`：断言 `sandbox_scope_mismatch`、容器保留且没有写墓碑 | Covered |
| 真实 Docker 停止 | 真实 Docker 中 clone 进行时终止，断言 Node 记录中断失败且卷中数据保留 | Missing |

### 决策依据

Docker 沙盒生命周期 D5、D6，不变量 4、6；中断失败判定决策。

## Workspace Data Delete Requires No Remaining Sandbox Container

### 风险

Workspace 删除时仍有容器挂着卷，删除失败或删掉仍在使用的数据。

### 前置状态

Workspace `W` 的沙盒均已终止，或仍有残留容器。

### 触发

Controller 发送 `workspace_data_delete`（Workspace `W`）。

### 必须成立

- 存在带 `W` 标签的容器时失败为 `workspace_sandbox_live`，卷不变。
- 否则删除卷 `ora-workspace-W`，卷不存在时同样返回 `removed: true`。

### 禁止结果

有容器时删除卷；删除其他 Workspace 的卷。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 有容器时拒绝 | `sandbox-server/internal/lifecycle/lifecycle_test.go` 的 `TestDataDeleteWaitsForContainersAndIsIdempotent`：容器存在时为 `workspace_sandbox_live` 且卷保留 | Covered |
| 幂等删除 | 同一测试在卷存在与已删除两种情况下断言 `removed: true` | Covered |

### 决策依据

Docker 沙盒生命周期 D6，不变量 6；运行时 Workspace 生命周期 D3。
