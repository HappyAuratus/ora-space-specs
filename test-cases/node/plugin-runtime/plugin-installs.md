# Node 插件安装

依据[已批准的 Node plugin-runtime 根决策](../../../decisions/node/plugin-runtime/0-node-installs-plugins-into-workspace-home.md)。
证据预期来自 `apps/ora-node` 以本地 HTTP 服务提供 release 的测试；实现尚未开始，全部为 `Missing`。

## Plugins Appear Only After Digest And Package Verification

### 风险

下载被篡改或包不合法的插件被放进插件根，随后被 Agent 会话启动。

### 前置状态

Workspace 插件根可写；本地 HTTP 服务提供 release。

### 触发

安装：同版本已装、未装、摘要不符、包校验失败、无匹配 target、下载失败三次、插件正被会话使用。

### 必须成立

- 只有 SHA-256 与载荷一致且通过安装期校验后才原子出现在插件根；D2 表中每种情况得到对应的逐项结果。
- 同版本已装时不下载；安装执行不启动插件进程。
- 正被会话租用的插件返回 `plugin_in_use`，不替换、不移除。

### 禁止结果

校验失败的包出现在插件根；安装过程中插件根出现半成品；替换使用中的插件。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| D2 表逐项 | Node 测试：每种情况的逐项结果与插件根内容 | Missing |
| 使用中保护 | Node 测试：持有 `PluginCatalog` 租约时安装与移除均为 `plugin_in_use` | Missing |

### 决策依据

plugin-runtime 根决策 D2、D4，不变量 1、2、4；agent-runtime D7。

## Install Executions Rerun Safely After Restart

### 风险

Node 在安装中途重启，留下临时文件或永远没有终态。

### 前置状态

安装执行已受理、尚无终态。

### 触发

Node 重启。

### 必须成立

- 恢复时清理临时下载目录，按原输入重新运行，最终只发出一个终态事件。
- 重跑结果与一次执行相同。

### 禁止结果

两个终态事件；残留临时目录；因重启报告 `interrupted` 而不重跑。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 重启重跑 | Node 测试：在下载后、原子替换前注入停止，重启后得到同一结果与单个终态事件 | Missing |

### 决策依据

plugin-runtime 根决策 D3，不变量 3。
