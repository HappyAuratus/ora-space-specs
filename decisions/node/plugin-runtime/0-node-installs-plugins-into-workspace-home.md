---
status: approved
---

# Node 以执行的方式把 Cloud 指定的插件安装到 Workspace 数据中的插件目录，复用 desktop 安装器

Node 收到 `InstallPlugins`/`RemovePlugins` 执行后，用 desktop 已有的 `ora-plugin-manager` 安装器按载荷下载、
校验 SHA-256、做安装期包校验，并原子放置到 Workspace 数据卷中的插件目录；逐个插件报告结果。安装只落盘，
不启动插件进程——进程由使用方（第一版只有 Agent 会话）按需启动。安装与移除是幂等的：同版本已装好直接
成功，不存在的插件移除直接成功。

本 ADR 处于 `approved`（已评审，待实现），是 `node/plugin-runtime` 的根决策（本目录范围：“Plugin 安装、启动、停止、运行和
本地状态”；第一版只承担安装与移除，运行由 [agent-runtime 根决策](../agent-runtime/0-shared-agent-runtime-crate-hosted-by-node.md)
按需启动 agent 插件）。消息见 [Node 协议后续决策](../protocol/20260928-streamed-thread-events-and-session-commands.md)；
载荷来源见 [Cloud plugin-marketplace 后续决策](../../cloud/plugin-marketplace/20260928-node-executes-plugin-installs.md)。
desktop 的插件清单、registry 与安装期校验决策（`desktop/plugin/*`）约束安装器本身，本文不修改它们。

## 决策

### D1：安装位置与生命周期

- 插件安装在 Node 数据目录（Workspace 数据卷挂载点）下的 `plugins/` 根，布局与 desktop `installed_root`
  相同；随 Workspace 数据跨沙盒保留，`start` 时的 plugin 步骤因此大多是“已安装”。
- Workspace 数据被 `workspace_data_delete` 删除时插件随之删除；Node 不另建全局插件缓存。

### D2：安装语义

| 情况 | 结果 |
|---|---|
| 同 ID 同版本已安装且清单有效 | `installed{version}`，不下载 |
| 未安装或版本不同 | 按 host target 选择 release → 下载到临时目录 → 校验 SHA-256 → 安装期包校验 → 原子替换 → `installed{version}` |
| 该插件正被运行中的会话使用 | `failed{plugin_in_use}`，不替换 |
| 没有匹配本 Node target 的 release | `failed{no_matching_target}` |
| 下载失败（重试 3 次，每次有期限） | `failed{download_failed}` |
| 摘要不符 / 包校验失败 | `failed{checksum_mismatch}` / `failed{invalid_package}` |

- 移除：删除该插件的安装目录；不存在即 `removed`；正在使用时 `failed{plugin_in_use}`。
- `pack` 与需要主机集成的类别（hook、webview、workbench）只落盘、不激活。
- 执行级失败（账本不可写、插件根不可用）使整个执行失败；逐项失败只写在结果中。

### D3：Node 重启后未完成的安装执行重新运行

与 clone 不同，安装执行在 Node 重启后可以从头重跑：安装以校验过的摘要为准、以原子替换落盘，重复执行的结果
与一次执行相同。恢复时 Node 清理临时下载目录，再按执行输入重新运行，最终仍只发出一个终态事件。

为什么可以重跑而 clone 不行：clone 的失败残留需要保留供排查，且目标目录由首次尝试预留（clone 契约 D6）；
安装的中间状态只在临时目录，最终状态由摘要唯一确定。

### D4：网络与信任

插件下载需要沙盒能访问 release URL；Node 不携带任何凭据下载。信任依据是 Cloud 从目录快照拼装的 SHA-256，
而不是下载来源。

## 不变量

1. 插件只在 SHA-256 与载荷一致、且通过安装期校验后才出现在插件根中。
2. 安装执行不启动插件进程。
3. 同一插件的安装结果只取决于载荷，重复执行结果相同。
4. 正在被会话使用的插件不会被替换或移除。

## 为什么不是这些替代方案

| 方案 | 取舍 |
|---|---|
| 插件打进 Node 镜像 | 无需下载；Space 的插件选择由用户决定，镜像无法覆盖，且每次插件更新都要重建镜像 |
| 全局插件缓存，跨 Workspace 共享 | 省下载；需要跨沙盒的共享卷与并发语义，与 Workspace 独立数据的边界冲突 |
| 安装后立即启动全部插件 | 与 Desktop 一致；运行 Workspace 只需要一个 agent 插件（agent-runtime D2） |

## 本决策未解决的问题

- 插件配置与密钥的下发、MCP/Skill 插件在会话中的激活。
- 插件根的磁盘配额。

## 落地

Node 接入 `ora-plugin-manager` 安装器与执行驱动；测试覆盖 D2 表中各情况（以本地 HTTP 服务提供 release）
与重启重跑。
