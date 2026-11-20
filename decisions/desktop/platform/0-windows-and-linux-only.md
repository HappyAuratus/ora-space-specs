---
status: proposed
---

# Ora Desktop 只支持 Windows 和 Linux：v0.6.0 是最后一个 macOS 版本，删除 macOS 代码，取代 macOS 原生纳管

Ora Desktop 和 desktop 仓库构建、发布的全部本机组件只支持 Windows 和 Linux。v0.6.0（desktop tag，提交
`1f88ccce`，已发布）是最后一个 macOS 版本；从下一个版本起不构建、不测试、不发布 macOS，desktop 里的 macOS
代码分支、构建配置、darwin sidecar 和 macOS 专用测试一并删除，不保留尽力支持的中间档。可观察的变化：

- 发布产物只有 Windows（NSIS）和 Linux（AppImage、deb），`latest.json` 只有 `windows-x86_64` 与 `linux-x86_64`；
  已装 0.6.0 的 macOS 客户端检查更新会失败，owner 接受这一后果（D3）。
- 插件的 `*-apple-darwin` target 退役：已发布清单里的 darwin 资产被跳过，不让整份清单解析失败；安装包自述
  darwin `[artifact]` 时拒绝（D4）。
- [macOS 原生尽力纳管](../../node/process/containment/macos/0-native-best-effort-containment.md)整份被本文取代；
  进程纳管只剩 Linux 和 Windows 两个平台方向（D5）。

本文为 `proposed`，等待 owner 评审。决策来自 owner 2026-11-20 的结论（ora-space/circus#1365）。实现已经以
preflight 在 desktop 分支 `refactor/1367-drop-macos`（`374ec3dc..b7cdb9c0`，运行 ora-space/circus#1367）完成，
尚未开 PR、未合并；本文按该实现写成，评审改动决策时实现跟着改。核心用例在本 PR 一并登记，证据在 desktop 实现
合并前都是 `Missing`，由落地运行登记。

- **涉及**：desktop 的 CI 与发布流程（`.github/workflows/`、`Taskfile.yml`、`apps/desktop/src-tauri/tauri.conf.json`）、
  `scripts/`（`setup-binary`、`sidecar-checksums.json`、更新清单生成）、Tauri 宿主（`apps/desktop/src-tauri`）、
  插件清单与 registry（`crates/plugin-manifest`、`crates/plugin-registry`）、共享 crates 里的 macOS 注释与 errno、
  前端的 macOS 快捷键与窗口分支，以及中英文文档。
- **不涉及**：cloud、cluster、marketplace、sandbox 等其他仓库的 macOS 内容（另议）；Linux 走的 `cfg(unix)` 共用
  代码；数据库与 migration（没有 macOS 相关内容）；Node 协议、Controller 契约与插件清单的 `resolver` 版本。
- **取代**：[macOS 原生尽力纳管](../../node/process/containment/macos/0-native-best-effort-containment.md)整份取代；
  其余引用 macOS 的决策只改涉及 macOS 的部分，见“与既有决策的关系”。
- **不承担的迁移**：不迁移、不通知已安装的 macOS 客户端，不撤回或改写已发布的 macOS 版本，不改写 marketplace
  里已发布清单的 darwin 条目。

## 归属

specs 里没有关于“Desktop 支持哪些平台”的根决策：desktop 下只有 cloud-connection、core、plugin、protocol 四个
领域，都不拥有平台集合；`node/process/containment/<平台>` 只定义各平台的纳管机制，
[Node 直接执行 Git](../../node/git-execution/0-node-runs-git-directly-with-attempt-locks.md) D6 只给出 Node Git 在各平台
的机制表。支持平台是 desktop 产品范围的跨领域决策，因此新建叶子 `desktop/platform`，本文是它的根决策，核心用例
放在镜像目录 `test-cases/desktop/platform/`。其他叶子里与平台有关的规则由本文跨叶子修改，修改项列在下一节，
各原决策标题下加提示。

## 背景

- **没有 macOS 用户。** owner 确认当前没有 macOS 用户，继续发布 macOS 版本没有服务对象。
- **macOS 有持续成本。** PR CI 和打包各占一个 `macos-15` runner 并需要 brew 依赖；打包要维护 `dmg`/`app` 目标、
  `icon.icns` 和 `.app.tar.gz` 更新包；签名与公证从未配置，Gatekeeper 会拦截；宿主代码另有 macOS 分支（用
  `open` 打开链接和位置、WKWebsiteDataStore 标识）、`setup-binary` 的 darwin 默认 target 与 4 个 darwin sidecar
  校验值、前端的 ⌘/⌥/⇧ 符号表。
- **macOS 纳管从未实现。** macOS 原生纳管 ADR 获批后，新的 process host／guardian 一直没有 macOS adapter；
  host／guardian 已被 Node 直接执行 Git 取代并删除，ora-node 只有 Linux 版，Node Git D6 的 macOS 一列也标着未实现。
  这份 ADR 留下的“未来 macOS adapter ADR”责任在放弃 macOS 后不会再有人承接。
- **已发布清单带 darwin 资产。** marketplace 里 `ora-space.claude`、`ora-space.codex`、`ora-space.opencode` 三个
  官方 agent 的已发布清单都有 `aarch64-apple-darwin` target。

## 与既有决策的关系

| 决策 | 处理 | 处理后的规则 |
|---|---|---|
| [macOS 原生尽力纳管](../../node/process/containment/macos/0-native-best-effort-containment.md) | 整份取代，改为 `superseded` | 不再有 macOS 纳管方向，也不再有未来 macOS adapter ADR；它的两条核心用例只服务该决策，随之删除（D5） |
| [纳管保证契约](../../node/process/containment/guarantees/0-explicit-containment-and-cleanup-evidence.md) | D1–D3 与不变量继承，不变；平台列表修改 | 平台实现只有 Linux 和 Windows；正文中以原生 macOS 为例的理由作为历史记录保留，不构成平台承诺 |
| [Linux cgroup](../../node/process/containment/linux/0-cgroup-v2-containment.md)、[Linux helper](../../node/process/containment/linux/20260917-independent-privileged-helper.md)、[Windows Job](../../node/process/containment/windows/0-creation-time-job-containment.md) | 继承，不变 | — |
| [Node 直接执行 Git](../../node/git-execution/0-node-runs-git-directly-with-attempt-locks.md) D6、“保持有效”、未解决问题 | 修改 | D6 平台表的 macOS 一列作废，Node 平台只有 Linux（已实现）和 Windows（给出机制、未实现）；“保持有效”的 containment 决策不再含 macos；“macOS 和 Windows 的落地”只剩 Windows |
| [插件分发](../plugin/registry/0-registry-distribution.md)“发布资产与 Target” | 修改 | 白名单外的 triple 仍是打包错误；`*-apple-darwin` 改为退役 target，按 D4 处理；示例中的 darwin 条目作为历史示例保留 |
| [插件清单](../plugin/manifest/0-versioned-dual-form-manifest.md) D5、不变量 7、未解决问题 | 修改；不变量 8 继承，不变 | `HookTarget` 白名单去掉两个 darwin triple，另设退役集合；release 清单的退役项按 D4 跳过，installed 清单的退役 `[artifact].target` 拒绝；同一份文本在任何宿主上仍得到同一个结果 |
| [会话尝试的进程带环境标记](../../node/agent-runtime/20261120-attempt-mark-in-session-stop-evidence.md) 未解决问题 | 修改 | “macOS 与 Windows”一项只剩 Windows；“本决策不覆盖 macOS”继续成立 |
| [部署签名身份交接 Desktop 会话](../../cloud/identity-access/20261019-deployment-bound-desktop-sessions.md) D7 与风险段 | 修改 | “三平台正式包”改为 Windows 与 Linux 两个平台的正式包 |
| [进程本机协议](../../node/process/protocol/0-independent-local-streams-and-state-recovery.md)等已 `superseded` 的决策 | 不回改 | 正文是历史记录，提到的 macOS transport 不构成当前承诺 |

## 决策

### D1：支持平台只有 Windows 和 Linux，v0.6.0 是最后一个 macOS 版本

Ora Desktop 和 desktop 仓库构建、发布的全部本机组件（Desktop 应用及其后端与 Controller、ora-node、ora-reaper、
sidecar、插件宿主）只支持 Windows 和 Linux。macOS 不属于任何支持档：不构建、不跑 CI、不发布、不修 macOS 专属
缺陷，也不声明尽力可用。v0.6.0（`1f88ccce`）是最后一个带 macOS 产物的版本；v0.6.0 及更早的 macOS 版本保持已
发布状态，不撤回、不改写。

只设两档（支持、不支持）而不保留“尽力支持”，是因为尽力支持在这里没有可验证的含义：没有 CI 就没有人知道
macOS 还能不能编译、运行，文档里的“尽力”会被读成支持。重新支持 macOS 需要新的 ADR，届时按当时的代码重新实现
和验证，而不是复活本文删除的分支。

### D2：删除 macOS 代码，而不只是停止构建

desktop 删除所有只服务 macOS 的内容：`cfg(target_os = "macos")` 分支及其实现、macOS 专用测试、`dmg`/`app`
打包目标和 `icon.icns`、darwin sidecar 校验值、宿主 triple 的 darwin 映射、`.app.tar.gz` 更新包类型、只为 macOS
引入的依赖，以及前端的 macOS 约定（⌘/⌥/⇧ 符号，`mod` 统一显示为 Ctrl，窗口平台类型不再有 `macos`，Apple 专用
字体回退）。原来写成“非 Windows”的兜底分支改为显式的 `target_os = "linux"`。

保留的边界：

- `cfg(unix)` 共用代码照常保留，它服务 Linux；只删其中针对 macOS 的注释和 macOS 专有 errno（目录 fsync 不再接受
  errno 45）。
- 未识别平台的通用降级路径（如 `HostPlatform::Other`、`WebDataIsolation::Degraded`）保留，它们不是 macOS 专用。
- 为 macOS 编译不是受支持的配置，不承诺能编译通过。

只停构建、保留代码的做法看起来便于回退，但没有 CI 的分支会无声腐化，读代码的人和 agent 又会把它们当成支持的
证据；回退需要的历史在 Git 里，不靠留在 main 上的死代码。把兜底改成显式 `target_os = "linux"`，使以后加入第三个
平台时编译器指出每一处需要决定的分支，而不是静默套用 Linux 行为。

### D3：CI、发布与更新只覆盖 Windows 和 Linux

| 环节 | 规则 |
|---|---|
| PR CI | crates 矩阵只有 Linux 和 Windows runner，去掉 macOS 的依赖安装步骤 |
| 打包 | 只在 Linux（`ubuntu-22.04`）和 Windows（`windows-2022`）上构建；bundle 目标为 `nsis`、`appimage`、`deb`；没有 macOS 签名、公证或 entitlements 配置 |
| 更新清单 | `latest.json` 的平台恰好是 `windows-x86_64` 与 `linux-x86_64`；其中任一平台缺少已签名资产时生成失败、不覆盖已有清单（沿用现有规则） |
| 更新包 | 更新器只处理 NSIS 与 AppImage 两种包 |
| sidecar | `sidecar-checksums.json` 没有 darwin 条目；`setup-binary` 在 macOS 主机上不推导默认 target，报 `Unsupported platform: darwin-<arch>` |
| 文档 | 中英文文档写明只支持 Windows 和 Linux、v0.6.0 是最后一个 macOS 版本，并删除 macOS 专属说明 |

**接受的后果**：已安装 0.6.0 的 macOS 客户端读取新的 `latest.json` 时找不到 darwin 条目，检查更新会报错，而不是
显示“已是最新”（这是对 Tauri 更新器行为的判断，没有在 macOS 上实测）。owner 因当前没有 macOS 用户而接受这一后果：
不为 macOS 客户端保留更新条目，不在应用内通知，不提供迁移。issue 要求在 v0.6.0 发布说明里预告“这是最后一个 macOS
版本”，那是发布操作，记录在 circus issue，不属于本文的系统承诺。

`setup-binary` 在 macOS 上明确报错，而不是退回某个 Linux 默认值，是因为后者会下载一个能通过校验、却不能在本机
运行的 sidecar，把平台错误推迟成运行时故障。

### D4：`*-apple-darwin` 插件 target 退役

`x86_64-apple-darwin` 与 `aarch64-apple-darwin` 移出 `HookTarget` 白名单，归入单独的**退役 target** 集合。退役
target 不是未知拼写：解析时报告不同的原因（`Retired`），各入口按下表处理。

| 输入 | 行为 |
|---|---|
| release 清单的 `[[targets]]` 同时有 darwin 和受支持 target | darwin 项被丢弃，其余 target 照常校验、照常可选 |
| release 清单的 `[[targets]]` 只有 darwin 项 | 视为没有发布来源：条目照常索引和展示，本机不可安装（“no downloadable release”） |
| installed 清单的 `[artifact].target` 是 darwin | 拒绝，错误字段 `artifact.target`，原因为退役 target |
| 不在白名单也不在退役集合的 triple | 仍是清单错误，规则不变 |
| 宿主自身的 triple | 永远不会是 darwin；不在支持集合内的宿主照旧判为不兼容 |

退役项在任何校验之前丢弃：它的 `url`、`sha256` 不再校验，重复的 darwin 项也不报重复。丢弃发生在解析阶段，与
宿主无关，同一份清单在 Windows 和 Linux 上得到同一个结果，保持清单根决策不变量 8。

为什么这样分：

- **不能当作未知 triple 报错。** 三个官方 agent 的已发布清单都带 darwin target；报错会让这些清单在所有平台上整份
  解析失败，Windows 和 Linux 用户因为一个不再支持的平台装不了插件。
- **跳过而不校验。** 没有任何宿主能选中退役项，校验它们只会让一个已发布条目因为不可能被下载的字段而失效。代价是
  作者写错的 darwin 项不再被提示；新清单本就不应再声明 darwin，这个代价可以接受。
- **installed 形态拒绝。** 安装包自述 darwin `[artifact]` 说明里面是 macOS 二进制，在受支持的宿主上不能运行；
  安装包没有“其余 target”可以退回，跳过等于把一个错误的包当成通用包装上。
- **单列退役集合。** 与未知 triple 分开，使“已发布数据里的历史平台”和“打包错误”各自可解释，以后其他平台退役时
  也按同一规则处理。

### D5：macOS 原生纳管被取代，进程纳管只剩 Linux 和 Windows

[macOS 原生尽力纳管](../../node/process/containment/macos/0-native-best-effort-containment.md)整份被本文取代，
状态改为 `superseded`；它的核心用例
`test-cases/node/process/containment/macos/native-cleanup-and-identity.md` 只服务该决策，随之删除，镜像目录保留。
进程纳管的平台方向只剩 Linux 和 Windows；Node Git D6 的 macOS 一列作废；以后不再为 macOS 写纳管 ADR。纳管保证
契约的规则（请求策略与实际能力分开、结果只承诺已有证据、纳管不扩展到外部副作用）不变。

取代而不是标 `implemented` 或继续保留 `approved`：该决策从未实现，D1 已放弃 macOS，保留 `approved` 会让“未来
macOS adapter”继续显示为待办的长期方向。

## 不变量

1. desktop 的 CI 与发布只为 Windows 和 Linux 构建；发布产物与 `latest.json` 不含 macOS 条目。
2. 宿主不会解析出、也不会选择 `*-apple-darwin` target；`setup-binary` 不为 macOS 主机提供默认 target。
3. 带 darwin target 的 release 清单在每个宿主上解析为同一个结果：darwin 项被丢弃，其余 target 照常可选；只剩
   darwin 项时没有发布来源。
4. installed 清单的 darwin `[artifact].target` 一律被拒绝。
5. 退役 triple 与未知 triple 分开报告；未知 triple 仍是清单错误。
6. Linux 使用的 `cfg(unix)` 共用代码行为不因删除 macOS 而改变。
7. v0.6.0 及更早的 macOS 版本不撤回、不改写。

## 为什么不是这些替代方案

| 方案 | 优点 | 为什么没选 |
|---|---|---|
| 保留 macOS 为尽力支持：代码和可选 CI 留着，不承诺发布 | 有人愿意时可以自行构建 | 没有用户；不跑 CI 的分支没人知道是否可用，却会被读成支持（D1）；macOS 纳管 ADR 的未来责任永远悬着（D5）；持续占用 macOS runner |
| 只停 macOS 构建和发布，保留代码 | 改动小，恢复时不用重写 | 死代码无声腐化，`cfg(target_os = "macos")` 分支和 darwin sidecar 继续误导读者；恢复所需的历史在 Git 里（D2） |
| darwin target 与未知 triple 一样报错 | 规则只有一条 | 三个官方 agent 的已发布清单在所有平台上解析失败，违反不变量 3 |
| 先改写 marketplace 清单删掉 darwin，再按未知 triple 报错 | 数据干净，不需要退役集合 | 要改其他仓库、全部历史版本，第三方源不受控制，还要求发布顺序；退役跳过不依赖它，清单清理以后仍可另做 |
| `latest.json` 继续保留一个指向 0.6.0 的 darwin 条目 | 0.6.0 的 macOS 客户端显示“已是最新”而不是报错 | 每次发布都要继续维护 darwin 资产与签名，违反不变量 1；没有 macOS 用户，owner 接受报错 |
| 在应用内提示 macOS 用户不再更新 | 用户知道原因 | 需要在 0.6.0 之后再发一个 macOS 版本，与 D1 矛盾 |

## 本决策未解决的问题

- **其他仓库的 macOS 内容。** cloud、cluster、marketplace、sandbox 等仓库的 macOS 构建或说明不在本文范围，另议。
- **marketplace 清单的 darwin 条目清理。** 属于 marketplace 仓库；D4 不依赖它。
- **Linux 上“在文件管理器、终端或 VS Code 中打开”。** 这是既有缺口，不由删除 macOS 引入；Linux 成为两个支持平台
  之一后更显眼，另开任务。
- **发布的 CPU 架构。** 本文只决定操作系统；发布仍只有 x86_64 产物，是否增加 arm64 另议。

## 落地顺序

1. **specs（运行 ora-space/circus#1366）**：本文、核心用例、macOS 纳管决策改为 `superseded` 并删除其用例，其他
   决策加 `> 历史决策：` 提示。owner 批准后本文改为 `approved` 再合并。完成条件：引用 macOS 的 specs 文档都已
   按本文处理或注明为历史记录。
2. **desktop（运行 ora-space/circus#1367）**：按 D2–D4 删除 macOS 并退役 darwin target，已在
   `refactor/1367-drop-macos` 完成开发，本文 `approved` 后开 PR 合并。完成条件：Windows 和 Linux 的 CI 全绿，
   `ora-desktop` 的编译以 CI 的 tauri job 和 Windows 打包为准（本机未能编译）；`git grep -i -E 'macos|darwin|\.dmg|notariz'`
   只剩有意保留的内容并逐条列在 PR 描述里。
3. **落地运行**：desktop PR review 通过后，把本文改为 `implemented`，按最终实现同步正文，并登记核心用例证据。

分开的好处是决策评审不被 76 个文件的实现 diff 淹没，实现 review 时有已批准的规则可以对照。desktop PR 合并之前
行为不变；合并后的下一个版本是第一个只有 Windows 和 Linux 的版本。

## 核心测试用例

[支持平台的核心用例](../../../test-cases/desktop/platform/supported-platforms.md)登记本文的验证义务；desktop 实现
合并前证据都是 `Missing`。
