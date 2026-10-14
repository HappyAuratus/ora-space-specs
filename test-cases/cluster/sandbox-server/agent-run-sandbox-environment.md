# 沙盒部署环境：环境变量注入

依据[已实现的沙盒部署环境决策](../../../decisions/cluster/sandbox-server/20260928-agent-run-sandbox-environment.md)。
本文件登记 Sandbox Server 向新建沙盒注入部署环境变量的承诺；模型访问证书见
[模型客户端身份用例](model-client-identity.md)，对象存储直传见 Cloud Revision 的用例。证据来自 ora-space/cluster
仓库（核对基线 `aef10f3`），路径相对该仓库根目录。

## Deployment Node Environment Reaches Only The Sandbox Container

### 风险

部署文件中的敏感值（代理凭据、Agent CLI 配置）写进 journal、日志或 effect 结果，被 Controller 与 Cloud 读到；
或者部署文件覆盖 Sandbox Server 自己设置的 `ORA_` 变量，替换 Node 配置或身份材料；或者格式错误被静默接受，
启动后才以难以定位的方式失败。

### 前置状态

Sandbox Server 以 `-lifecycle-addr` 与 `-node-env-file <path>` 启动。

### 触发

Sandbox Server 启动时读取文件；之后 Controller 发送 `sandbox_ensure`。

### 必须成立

- 空行与 `#` 注释行忽略；值是第一个 `=` 之后的全部内容，按字面使用。
- 变量名不匹配 `^[A-Z][A-Z0-9_]*$`、以 `ORA_` 开头、重复、只写变量名或文件缺失时，启动失败；错误只含行号与
  变量名，不含值。
- 变量按文件顺序追加在 `ORA_NODE_CONFIG` 与管理身份变量之后，进入容器环境。
- 变量值不出现在 journal、effect 结果与日志中。

### 禁止结果

部署文件设置 `ORA_` 前缀的变量；错误信息回显变量值；值进入 journal、effect 结果或日志；`-node-env-file` 在没有
`-lifecycle-addr` 时生效。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 文件格式 | `sandbox-server/internal/lifecycle/nodeenv_test.go` 的 `TestParseNodeEnv` 断言注释、空行、`=` 后内容按字面保留（含引号与 `\r` 处理） | Covered |
| 非法变量拒绝且不回显值 | 同文件的 `TestParseNodeEnvRejectsWithoutEchoingValues` 以小写名、数字开头、前导空格、含横线、`ORA_` 前缀、重复与只写变量名等输入断言报错且错误不含值；`TestLoadNodeEnvMissingFile` 断言文件缺失报错 | Covered |
| 只进入容器环境 | 同文件的 `TestEnsureInjectsNodeEnvOnlyIntoContainer` 断言容器环境为 Node 配置后接部署变量，且值不出现在 journal 文件、effect 结果与日志中；`sandbox-server/internal/lifecycle/model_environment_test.go` 的 `TestSecuredEnsurePreservesNodeEnvAndIsolatesModelIdentity` 断言启用管理身份时部署变量仍保留 | Covered |
| 参数组合 | `cmd/sandbox-server` 拒绝没有 `-lifecycle-addr` 的 `-node-env-file`，无测试 | Missing |

### 决策依据

沙盒部署环境 D1，不变量 1、2。
