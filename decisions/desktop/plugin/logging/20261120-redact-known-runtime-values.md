---
status: approved
---

# 插件日志：Host 按已知运行时值尽力脱敏

本决策在已实现的 [插件日志根决策](0-host-owned-plugin-log.md) 之上，给宿主拥有的插件日志增加
「对已知运行时值尽力脱敏」：Host 在把 stderr 解码成记录、通过级别过滤之后，到渲染成 JSONL
之前，对记录中**插件可控的字符串字段**做替换，把一组 Host 已知的敏感明文（插件 MCP Settings
的有效值、宿主 MCP 的 URL 与请求头、模型 token、desktop Skill secret、desktop 工作流完成
token）替换为固定占位符 `[REDACTED]`。desktop 与 Node 共用同一套采集与匹配实现。这是尽力而为
的脱敏：只匹配明文子串，不解析 JSON 转义、百分号编码或 base64；跨 64 KiB 分片的超长值会部分
泄露；它不把「已脱敏」宣称为「日志绝不泄露」。

本 ADR 目前是 `approved`。实现落在 `ora-plugin-runtime`（匹配）、`ora-plugin-lifecycle`
（登记表订阅）、共享 `ora-agent-runtime`（交付点采集）与各宿主的 spawn 环境提供者，不改
Plugin Protocol 的对外字段，也不改变日志的归属、级别、队列、容量与收尾语义。

## 继承与修改

继承 [插件日志根决策](0-host-owned-plugin-log.md) 的全部约定：日志归属独立 logs tree、单
writer、JSONL 身份与接收时间由 Host 填写、SDK 五个 console 方法共用 stderr、版本化 envelope
与 raw fallback、按 Plugin ID 的持久阈值、队列与 sink 故障隔离、收尾截止时间。本决策只修改
两处边界：

| 前序约定 | 处理 |
| --- | --- |
| 根决策开头「不承诺任意第三方输出自动脱敏」 | 修改：对 Host 已知的运行时值，改为尽力脱敏 |
| 根决策「日志可能含插件主动写出的敏感内容」，读取、导出、远端 sink、rotation/累计磁盘额度、更强进程隔离需各自后续决策（根决策「后果与边界」末段） | 本决策只承接其中的「已知运行时值脱敏」；读取、导出、远端 sink、rotation/累计磁盘额度、更强进程隔离仍各自后续决策 |

## 决策

### D1：脱敏发生在「解码之后、渲染之前」，只作用于插件可控字段

stderr 字节经 `decode → filter(level) → render JSONL → 有界队列 → 写文件`。脱敏作用在
`decode_frame` 产出记录、级别过滤通过之后、`to_json_line` 渲染之前，作用于记录中所有插件
可控的字符串：`message`、`target`、`method`、`error`（递归），以及 `context`（递归，含键名；
Host 保留键除外）。

- Host 写入的字段不参与匹配：`timestamp`、`level`、`plugin_id`、`host_session_id`、
  `generation`，以及 `context` 里的 `encoding`、`fragment`、`format_failure`。这些字段在结构上
  不可能被改写，与根决策 D3「Host 字段可信、不受插件影响」一致。
- raw 记录（含无效 UTF-8 的转义文本）与 v1 envelope 走同一处理：JSON 转义由序列化器自然处理，
  不发生在明文子串匹配之后。
- `context` 键名被替换后若两个键冲突，保留先出现者，其余同名键丢弃。
- 队列字节核算按脱敏后的渲染长度计算，不按脱敏前的行长预留。

### D2：脱敏值按 Plugin ID 登记进 Host 的实时注册表，交付前先登记

每个 Plugin ID 有一份 Host 持有的脱敏注册表，和持久日志级别一样以订阅形式进入
`PluginLogSetup`（`redactions: watch::Receiver<…>`），在 plugin-lifecycle 的 `complete_launch`
中注入。插件无法读取或改写它。

- **交付前登记**：Host 把任何敏感值交给插件进程之前，必须先把它登记进该插件的注册表并发布：
  spawn 环境（legacy token、desktop Skill 环境）、`agent/start` 的 `AgentModelContext`、ACP
  会话 setup / load / 刷新携带的 `mcpServers`（插件 MCP 与宿主 MCP）。发布完成之后才发送。
- **登记点**：采集放在 desktop 与 Node 共用的 `ora-agent-runtime` 交付点（MCP 映射
  `mcp_resolver`、`host_mcp_servers()`、`start_agent` 的 `model_context`），外加各宿主的 spawn
  环境提供者，使两端共用一套采集与匹配实现。
- **只增不减**：同一进程代内，值一经登记就保留到该代结束；会话结束、刷新替换都不撤销旧值，
  因为 Agent 仍可能回显旧值。新进程代从仍在交付的值重新登记。Node 的模型 grant 在 renew 时不
  换 token（spawn 环境沿用同一临时 token），所以会话中不存在模型 token 轮换需撤销的问题。
- **有界**：注册表最多 256 条、总字节最多 64 KiB。超限后新值不再登记，并记一条
  只含计数、不含值的 Host 告警。超限不阻塞交付，也不改变插件协议。
- 更新只影响之后入队的行，不回改已落盘记录。

### D3：匹配语义与代价

只登记 `String` 类型的值，Number / Boolean 永不登记。最小长度定为 **8 个 UTF-8 字节**：低于
此长度的值误伤率高，且作为秘密本身已不可靠；短于阈值不脱敏，并列进 D5 的已知局限。

- **匹配语义**：在原字符串上一次找出所有登记值的全部出现位置（允许重叠，如 `abcXYZ` 与
  `XYZdef` 同时出现在 `abcXYZdef` 中），合并相交或相邻的区间，每个合并后的区间替换为一个
  `[REDACTED]`。结果与登记顺序无关，任何登记值的任何字节都不会残留在被匹配的区间之外。
- **代价**：匹配器（如 Aho-Corasick 自动机）在注册表变化时构建一次；每条记录的匹配为
  O(记录长度 + 匹配数)，与登记值个数无关；注册表为空时不做任何工作。
- **失败语义**：匹配器构建失败或注册表超限时，按已构建的最后一版继续匹配，并记一条 Host
  告警，不丢弃、不阻塞记录。
- 占位符统一为 `[REDACTED]`，与 `SessionMcpBinding` 的 `Debug` 占位一致，避免在日志里引入
  新的可识别标记。

### D4：覆盖的运行时值集合

| 值 | 来源与登记时点 |
| --- | --- |
| 插件 MCP Settings 的 `String` 有效值 | 绑定进 stdio `env`/`args` 与 http `url`/`headers` 的 setting 值（`resolve_mcp_transport` 产物），在对应 ACP setup 发出前登记 |
| 宿主 MCP 的 `url` 与每个 `headers` 值 | Node 的 `HostMcpServer`（`NodeSessionMcp` / `HostServers` 输入），以及 desktop Backend 交付的宿主 HTTP MCP Server（含完成工具 URL），在对应 ACP setup 发出前登记 |
| 模型 token | `AgentModelContext.token`（adapter 路由）与 legacy 环境的 `ORA_MODEL_ACCESS_TOKEN` |
| desktop Skill Settings 中 `secret` 为真的 `String` 值 | `SkillEnvironment` 启动快照（注入 Agent 进程环境与 host-managed 子进程环境） |
| desktop 工作流完成 token | Backend 为交互式工作流签发并随 prompt 交付的 `attemptToken`，在发送该 attempt 的 prompt 前登记 |

**不登记 `proxy_base_url`**：它是网关主机加固定路径（`/runtime/openai/v1`、`/runtime/anthropic/v1`），
同一部署的所有会话都相同，且 [模型 adapter 决策](../../../node/agent-runtime/20261017-plugin-owned-model-adapters.md)
已写明「公开 CA 和固定代理地址不是秘密」。登记它只会抹掉 Agent 的正常网络诊断；模型 token 才
是应脱敏的秘密。

desktop 路径没有 Host 注入的模型 token，但有 Skill secret 环境与会话级插件 MCP Settings；Node
路径三者皆有。

### D5：尽力而为的边界与已知局限

脱敏是尽力而为，只覆盖 Host 拥有的 Plugin log（`<home>/plugins/logs/<namespace>/<name>/plugin.log`
及其 D10 恢复文件）；Ora runtime log（含 debug 构建的 TRACE）、会话 JSONL、ACP 消息不在本
决策范围。明确以下不覆盖的情形：

- JSON 转义（`\u0041`）、百分号编码（`%41`）、base64 等非明文形式匹配不到。
- 值跨 64 KiB 分片边界时，两侧分片各自保留该值的一部分明文（部分泄露，不是整体绕过）。
- 宿主 MCP 只打印 token（去掉 `Bearer ` 前缀的凭据部分）或经 URL 解析器规范化后的 URL（加
  尾斜杠、host 小写）会漏，因为 D4 只登记完整 `url` / header 值，不登记派生形式。
- 短于 8 字节的秘密不脱敏。
- Agent 写到自己能访问的其他文件（非 Host 拥有的 Plugin log）的不归本决策管；插件不经过
  Host 管道、直接自行写文件的不归本决策管。

## 不变量

1. 脱敏只改写 `decode_frame` 之后、`to_json_line` 之前记录中插件可控的字段，不改写帧、解码
   计数与过滤结果。
2. 脱敏值由 Host 登记，插件不能读取或覆盖；注册表为空时管道行为与未实现脱敏时完全一致。
3. 脱敏不改写 Host 写入的字段、记录条数与入队顺序；队列字节核算按脱敏后的渲染长度计算；
   脱敏后单条记录长度增长有上界（由最小长度与占位符长度决定）。
4. 匹配器构建失败或注册表超限时，按已构建的最后一版继续匹配并记一条 Host 告警，不丢弃、
   不阻塞记录，不改变记录数量。
5. 脱敏是尽力而为的明文子串替换，不承诺覆盖 JSON 转义、百分号编码、base64 或跨分片值。
6. 对任一已登记值 S（满足登记规则），Host 把 S 交付给插件之后，该插件进程代写到 stderr 的
   任何包含 S 的完整帧（raw 行或 v1 envelope 的任何插件可控字段），落盘到该 Plugin ID 日志树
   的记录中都不含 S；跨 64 KiB 分片的出现除外（D5）。
7. 在第 6 条中，S 覆盖 Node 路径的插件 MCP Setting（含刷新后新值）、宿主 MCP 请求头、模型
   token（adapter 与 legacy 环境），以及 desktop 路径的插件 MCP Setting、Skill secret。
8. 脱敏后的每条记录仍是合法 JSON，Host 字段与未脱敏时逐字节相同。

## 为什么不能直接改写已有日志

本决策不改写已落盘的 `plugin.log`、其 D10 恢复文件与既有明文历史：改写会破坏根决策 D2 的
单 writer 与 D10 的残缺尾部保留，且历史文件没有轮转、要等卸载才删除。清除历史明文的安全
路径是卸载并删除数据（根决策 D6/D8 的删除数据卸载），或等后续 history/retention 决策；导出
仍是原样复制文件、不再次脱敏，这些边界与「上线前历史文件仍含明文」一起，随 ADR 落地同步
记录到受影响文档。

## 为什么不是这些替代方案

| 方案 | 更好的维度及最终取舍 |
| --- | --- |
| 要求插件或 SDK 自己脱敏后再写 stderr | 值离插件最近，但第三方依赖和原生组件直接写 stderr，Host 无法强制；仍会有未脱敏内容。 |
| 把含敏感值的整行丢弃 | 不会泄露，但会丢掉同一行里的正常诊断，且需要可靠的「含敏感值」判定，误判代价高。 |
| 用正则或语义解析脱敏（URL、JSON、header 结构） | 覆盖更多编码形式，但值的形态多样、跨语言，维护成本高；明文子串替换已覆盖主要泄露路径，剩余边界在 D5 明确排除。 |
| 加密整个 plugin.log | 从文件层面杜绝泄露，但破坏现有可读性与诊断工具链，且仍需密钥管理；与「Host 拥有、可导出诊断」的定位冲突。 |
| 只靠文件权限与信任域，不做脱敏 | 现状。但诊断导出会把 plugin.log 带出信任域，且同工作区用户/进程可读，泄露面仍然存在。 |

## 本决策未解决的问题

- 远端 sink、日志导出、rotation/累计磁盘额度的脱敏与授权，留各自后续决策。
- 对编码形式（JSON 转义、百分号、base64）的覆盖，如需要更强的保证再另开决策。
- 跨 64 KiB 分片边界的完整匹配，需要改动分帧语义，暂不承诺。
- desktop 复用同一运行中插件进程、各会话值不同且不经启动链交付时的合并语义（P2-2），留到
  实现时按 D2 的「先登记后交付 + 只增不减」原则细化，可另开 PR。

## 落地顺序

1. **ADR 评审**：本 ADR 已评审到 `approved`，核心用例在同一 PR 补齐（证据先 `Missing`）。
2. **实现**：`ora-plugin-lifecycle` 登记表订阅 + `ora-agent-runtime` 交付点采集 + 宿主 spawn
   环境提供者 + `ora-plugin-runtime` 管道匹配（三段各自可验收）。
3. **落地同步**：实现验证后把 ADR 改为 `implemented`，并按最终实现同步正文与核心用例证据；
   同时把以下既有文档里「按已知运行时值的尽力脱敏，circus#1294 承接」的引用改指向本 ADR，并
   在 Skill ADR 的已知边界里注明「插件日志已有尽力脱敏」：
   - `decisions/node/agent-runtime/20261114-explicit-session-mcp-in-agent-session-spec.md`
   - `decisions/node/agent-runtime/20261118-host-mcp-servers-in-agent-session-spec.md`
   - `test-cases/node/agent-runtime/session-mcp.md`
   - `test-cases/node/agent-runtime/host-mcp-servers.md`
   - `decisions/desktop/plugin/category/skill/20261016-secret-settings-and-startup-environment.md`