# Extension Pack 本地手工验收 SOP

本文档用于在 Extension Pack 提交至正式 `ora-space/marketplace` 之前，对 ORA Desktop
中的 Extension Pack 功能进行完整的本地手工验收。

自动化测试证明代码能重复验证；本 SOP 证明真实用户从 Desktop UI 操作也能完整走通。

## 1. 当前验收基线

验收时使用的三个仓库 SHA（以实际 `git log` 为准）：

```text
Desktop:     1e61cfe6 test(extension-pack): release qualification for marketplace update, content update, and download failures
Specs:       bf11f43 docs(plugin): release qualification evidence for python extension pack
Marketplace: ec36152 feat: add python-extension-pack with skill, MCP server, and agent-gated members
```

Marketplace 测试源：

```text
https://github.com/sunSUNQ/ora-marketplace
Branch: main
```

自动化测试基线（验收前最后一次全量运行）：

```text
workspace Rust:  70 suites / 1474 passed / 0 failed
e2e:             12 (desktop) + 1 (pack qualification) = 13 passed
app-shell:       166 test files passed
contracts TS:    9 passed
desktop TS:      42 passed
```

### 1.1 D5-Q1 状态裁决（2026-09-17）

```text
D0–D4:                              CLOSED
D5 Content:                         IMPLEMENTED
D5-Q1 Evidence Repair:              CLOSED
D5-A Release Qualification:         RELEASE QUALIFIED
                                    — with documented inherited baseline/environment exceptions
Manual Release Qualification:       READY
Marketplace upstream PR:            PAUSED（等待本地手工验收完成）
```

注脚所列两项例外均不归因于 Extension Pack：app-shell Prettier 存在 1 处既有违规
（`filter-discovered-plugins.ts`，已提交且与本轮无关）；`check:tooling` 因本机 Deno 版本与
仓库 pin 不一致无法原样完成。D5 修改范围零新增违规，最终 release evidence 应表述为
"D5 修改范围零新增违规，剩余两项为既有/环境例外"，而非"所有全局 gate 无条件全绿"。

冻结结论：ORA Extension Pack D5-Q1 已完成发布资格证据修复，4 个核心场景（member version
update、marketplace content update、download missing、SHA mismatch）均通过真实
Plugins/PluginApi 生产入口验证，ownership 与历史 ledger 不变量得到机械证明，D5-A 恢复
RELEASE QUALIFIED。

可复用的测试模式：test seam 只替换 artifact transport（URL→本地制品映射），生产 manifest
resolve、release selection、host compatibility、SHA 校验、staging、包校验、原子提交、
ownership、rollback 全部照走；后续同类 Marketplace 功能做 release qualification 时沿用。

正式开始测试前，确认三个仓库工作树无未知修改：

```powershell
cd D:\ora-sunq
git status
git log --oneline -5

cd D:\ora-sunq\specs
git status
git log --oneline -5

cd D:\ora-sunq\ora-marketplace
git status
git log --oneline -5
```

---

## 2. 编译并启动 ORA Desktop

### Step 2.1 进入仓库并确认版本

```powershell
cd D:\ora-sunq
git status
git log --oneline -5
```

### Step 2.2 安装前端依赖

```powershell
task install:frontend
```

### Step 2.3 可选：确认 Rust 编译

```powershell
cargo build --workspace
```

### Step 2.4 启动 Desktop

```powershell
task run:desktop
```

保持窗口运行。不要在测试过程中关闭。

---

## 3. 日志观察

建议另开一个 PowerShell 窗口：

```powershell
Get-Content .\.data\logs\ora.log.* -Wait
```

重点关注日志关键词：

```
marketplace sync / manifest resolve / pack preflight / download
sha256 / staging / install / agent-runtime / ownership / reconcile / uninstall
```

---

## 4. 确认独立测试数据

本地开发数据位于 `D:\ora-sunq\.data`。

不要混用真实用户生产数据。如需重置：

1. 退出 Desktop。
2. 备份 `.data`。
3. 删除 `.data`。
4. 重新 `task run:desktop`。

---

## 5. 配置 Marketplace Source

进入 Settings → Plugins → Manage Sources。

确认 Source：

```text
URL:     https://github.com/sunSUNQ/ora-marketplace
Branch:  main
Enabled: Yes
```

执行 Sync 并等待完成。

---

## 6. Marketplace 同步验证

### 6.1 Pack 卡片可见

检查 Marketplace 中：

- Python Extension Pack 卡片可见
- 存在 Pack 类型标识
- 成员列表可以正常展示

### 6.2 Hidden Member 不独立展示

检查 `marketplace_visible = false` 的成员（如 `ora-space.python-core`）：

- 不能作为普通独立插件出现在 Marketplace 列表中
- 但仍然是 Pack 的成员

如果 hidden member 独立出现 → **FAIL**

---

## 7. Fresh Install

确保所有 Pack 成员此前没有安装。

在 Marketplace 中点击 Python Extension Pack 的 Install 按钮。

### 7.1 观察安装过程

日志中应依次出现：

```
resolve pack → preflight → resolve members → download → SHA-256 → staging → validation → commit → ownership recording
```

Pack 自身不下载 artifact。真正下载的是 member。

### 7.2 检查结构化反馈

安装成功后应看到 Pack 的结构化反馈（不只是通用 "Success"）：

| 字段 | Fresh Install 预期 |
|---|---|
| Installed members | 所有适用成员 |
| Skipped | 空 |
| Failed | 空 |

---

## 8. 检查 Installed / Packs 页面

进入 Installed Plugins 页面。

检查：

- Python Extension Pack 出现在 Packs 区域
- Pack 关联成员信息可见
- 成员已真实安装（不在 Packs 区域中出现重复）
- Pack 本身不出现在普通 InstalledPlugin 列表中

---

## 9. 重启验证持久化

完全退出 Desktop。重新 `task run:desktop`。

进入 Installed / Packs 检查：

- Pack 仍然存在
- 成员关系正确
- 没有重复安装
- 没有 phantom failed 状态
- ownership 关系完整

---

## 10. PreExisting Member 验证

### 10.1 单独安装一个 Pack 成员

从 Marketplace 单独安装一个 Pack member（如 `ora-space.python-mcp`）。

### 10.2 安装 Pack

点击 Python Extension Pack 的 Install。

结构化结果应显示：

```text
python-mcp → skipped / reused
其他新成员 → installed
```

### 10.3 卸载 Pack

卸载 Pack。

预期：

```text
python-mcp（预装）→ 保留
其他 managed 成员 → 删除
Pack 关系 → 清理
```

如果预装成员被删除 → **FAIL**

---

## 11. Marketplace 更新验证

### 11.1 准备更新版本

在 fork 中将某个 member 从 1.0.0 更新为 1.1.0（artifact + SHA-256 + manifest）。

### 11.2 Desktop 重新 Sync

确认新版本被发现。

### 11.3 执行更新

点击对应成员的 Update。

日志应出现：

```
stop lifecycle → retire old → install new → reconcile
```

### 11.4 检查 Windows 旧目录退役

打开安装目录确认：

- 新版本目录存在
- 旧版本目录已删除
- 不残留 CWD 锁导致的旧目录

### 11.5 重启验证

重启 Desktop 后新版本仍然存在。

---

## 12. VersionChanged 安全卸载验证

### 12.1 独立更新成员

在 Pack 安装后，独立将某个 member 从 1.0.0 升级到 2.0.0。

### 12.2 卸载 Pack

卸载 Pack 时，该成员（VersionChanged）必须保留。

其他未变化的 ManagedByPack 成员正常删除。

---

## 13. Missing Member 卸载验证

人为删除某个 Pack-owned member 的安装目录。

执行 Pack 卸载。

预期：

- Missing 成员不执行文件删除
- 不阻塞整个卸载流程
- 其他成员正常处理
- Pack ledger 最终正常释放

---

## 14. 下载失败验证

将某个成员的 artifact URL 指向不存在的 HTTPS 地址。

Sync 后安装 Pack。

预期：

- 下载失败
- 安装失败
- UI 有明确错误提示
- 无 installed directory
- 无错误 ownership

---

## 15. SHA-256 校验失败验证

保留真实 URL，修改 listing 中的 sha256 为错误值。

Sync 后安装。

预期：

- 下载成功但校验失败
- 安装终止
- 无 installed directory
- 无 phantom ownership

---

## 16. Rollback 验证

准备一个临时测试 Pack：Member A（正常）、Member B（正常）、Member C（故意失败）。

安装 Pack。

预期：

```
A install → B install → C fail → rollback B → rollback A
```

最终：A、B、C 均不存在。本轮产生的 ownership 已清空。

如果 rollback 中某个成员删除失败，该成员保留且日志记录恢复证据。

---

## 17. 正常 Pack 卸载

恢复 Marketplace Source 到正常状态。

安装 Python Extension Pack 后执行 Uninstall。

检查：

- Managed + Expected 成员 → 删除
- PreExisting → 保留
- VersionChanged → 保留
- Missing → 无文件操作
- Pack ledger → 清空

重启确认不再出现。

---

## 18. Agent Contribution 验证

### 18.1 Skill

Pack 中包含 Skill 成员。创建新 Agent session，向 Agent 发送触发该 Skill 的任务。

验证 Skill 被发现并实际执行。

### 18.2 MCP

创建能调用 MCP 能力的任务。确认 MCP server 启动、tool 可调用、结果正常。

### 18.3 Agent-Gated Member

使用匹配 Agent 创建 session → 成员有效。

使用不匹配 Agent → 成员不进入使用路径。

---

## 19. 测试完成后

```powershell
git status
git rev-parse HEAD
git -C specs rev-parse HEAD
git -C ora-marketplace rev-parse HEAD
```

保存三个 SHA。

---

## 20. 验收记录模板

```text
测试日期:
OS:
Desktop SHA:
Specs SHA:
Marketplace SHA:
Marketplace URL:
Marketplace Branch:
```

| 场景 | 结果 | 证据/备注 |
|---|---|---|
| Desktop 编译 | PASS / FAIL | |
| Desktop 启动 | PASS / FAIL | |
| Marketplace Sync | PASS / FAIL | |
| Pack Marketplace 展示 | PASS / FAIL | |
| Pack 成员列表展示 | PASS / FAIL | |
| Hidden Member 不独立展示 | PASS / FAIL | |
| Fresh Pack Install | PASS / FAIL | |
| Structured PackInstalled Feedback | PASS / FAIL | |
| Installed / Packs 区域展示 | PASS / FAIL | |
| Restart Persistence | PASS / FAIL | |
| PreExisting Member Reuse | PASS / FAIL | |
| PreExisting Uninstall Protection | PASS / FAIL | |
| Marketplace Version Update | PASS / FAIL | |
| Windows Old Version Retirement | PASS / FAIL | |
| VersionChanged Protection | PASS / FAIL | |
| Missing Member Uninstall | PASS / FAIL | |
| Download Failure | PASS / FAIL | |
| SHA-256 Failure | PASS / FAIL | |
| Install Failure + Rollback | PASS / FAIL | |
| Rollback Failure Evidence | PASS / FAIL | |
| PreExisting Rollback Protection | PASS / FAIL | |
| Normal Pack Uninstall | PASS / FAIL | |
| Restart After Uninstall | PASS / FAIL | |
| Skill Contribution | PASS / FAIL | |
| MCP Contribution | PASS / FAIL | |
| Agent-Gated Member | PASS / FAIL | |

最终结论：`LOCAL MANUAL RELEASE QUALIFICATION: PASS / FAIL`

---

## 21. 失败归因

不要因单个场景失败直接判定机制失败。先区分：

| 类型 | 说明 |
|---|---|
| Desktop implementation defect | ORA 代码缺陷 |
| Marketplace content defect | orax.toml 内容或格式错误 |
| Artifact defect | .orax 包内容或格式错误 |
| Environment / filesystem issue | Windows 路径或权限问题 |
| Windows process lock | 杀毒/索引器或进程 CWD 占用 |
| Agent contribution / runtime issue | fake-agent 或实际 agent 进程问题 |
| Test setup issue | 测试环境或 fixture 配置问题 |

确认归因后再决定是否修改代码。

---

## 22. 通过标准

全部场景 PASS 后标记：

```text
LOCAL RELEASE QUALIFICATION = PASS
```

然后进入：

```
Marketplace Upstream PR → Online Qualification → Release Closure
```
