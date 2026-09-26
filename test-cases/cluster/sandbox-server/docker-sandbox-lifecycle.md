# Sandbox Server 的 Docker 沙盒生命周期

依据[已批准的 Docker 沙盒生命周期决策](../../../decisions/cluster/sandbox-server/20260926-docker-sandbox-lifecycle-via-substrate-effects.md)。
本文件登记 Sandbox Server 以 Cloud 的 Substrate effects 接口创建、终止沙盒与删除 Workspace 数据的承诺；
标签路由见[Docker 标签解析用例](docker-label-resolution.md)，Cloud 侧步骤见
[运行时 Workspace 生命周期用例](../../cloud/operation/workspace-runtime-lifecycle.md)。实现尚未开始，证据均为 `Missing`。

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
- `GET` 不创建任何东西；journal 为 `running` 时按容器名查询 Docker 据实返回。

### 禁止结果

同一 effect 或同一 Workspace 出现两个容器；以超时或重启推断成功或失败；容器发布端口或接入其他网络；
不支持的 `kind` 写入 journal。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| journal 先于 Docker 写入、幂等与 `409` | 测试替身后端记录调用顺序，断言重发与不同请求的响应 | Missing |
| 崩溃恢复不重复创建 | 在 journal `running` 后丢弃进程状态重建服务，重发 `PUT`，断言只有一次创建且结果成功 | Missing |
| 同名冲突与 Workspace 唯一性 | 伪造 Engine 返回名称冲突、已有同 Workspace 容器，断言分别继续与失败 | Missing |
| 容器规格 | 伪造 Engine 捕获创建请求，断言名称、标签、网络、端口、卷与配置 | Missing |
| `unsupported_kind` | 发送 storage/worktree/plugin effect，断言 `400` 且无 journal 文件 | Missing |

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

- terminate 与 `E` 的 ensure 共用按 ID 锁，先写终止墓碑；之后针对 `E` 的 ensure 以 `sandbox_terminated` 失败。
- 只有 Docker 确认 `ora-sandbox-E` 不存在后才返回 `terminated: true`；停止使用 `-stop-timeout`。
- 容器 Workspace 标签与请求不一致、Docker 报错或超时时 effect 为 `failed`。
- 卷 `ora-workspace-W` 保留。

### 禁止结果

容器仍存在时报告终止；终止确认后又出现 `ora-sandbox-E`；terminate 删除 Workspace 卷。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 以容器不存在为终止证据 | 伪造 Engine 在删除失败、超时与成功时分别断言结果 | Missing |
| 墓碑阻止迟到 ensure | 同步 ensure 与 terminate 的开始顺序，断言所有允许的交错下终止后无容器 | Missing |
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
| 有容器时拒绝 | 伪造 Engine 返回同 Workspace 容器，断言失败且未调用卷删除 | Missing |
| 幂等删除 | 卷存在与不存在两种情况均返回 `removed: true` | Missing |

### 决策依据

Docker 沙盒生命周期 D6，不变量 6；运行时 Workspace 生命周期 D3。
