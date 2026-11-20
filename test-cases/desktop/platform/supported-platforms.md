# 支持平台的核心用例

依据[Ora Desktop 只支持 Windows 和 Linux](../../../decisions/desktop/platform/0-windows-and-linux-only.md)。下文的 D 编号和
不变量编号都指这篇 ADR。

ADR 为 `implemented`，实现由 [desktop#804](https://github.com/ora-space/desktop/pull/804) 合入，证据固定在合并提交
[`1456af72`](https://github.com/ora-space/desktop/commit/1456af721bad5577b628ac0d5d9ead14eb8b4809)（评审 head `51342699`，
运行 ora-space/circus#1367）。证据路径相对 desktop 仓库根目录，行号指测试函数或 `Deno.test` 的起始行。13 项义务中
7 项 `Covered`、1 项 `Partial`、5 项 `Missing`；5 个用例都是 `Partial`。

| 不变量 | 用例 | 状态 |
|---|---|---|
| 1 | [A Release Publishes Only Windows And Linux Artifacts](#a-release-publishes-only-windows-and-linux-artifacts) | Partial |
| 3、5 | [A Listing With Darwin And Linux Targets Installs On Linux](#a-listing-with-darwin-and-linux-targets-installs-on-linux) | Partial |
| 3 | [A Darwin-Only Listing Has No Release](#a-darwin-only-listing-has-no-release) | Partial |
| 4、5 | [An Installed Darwin Artifact Is Refused](#an-installed-darwin-artifact-is-refused) | Partial |
| 2 | [A macOS Host Gets No Sidecar Target](#a-macos-host-gets-no-sidecar-target) | Partial |

不变量 7 由发布流程只操作本次 tag 的 draft Release 保证，是发布操作而不是代码行为，不登记为代码证据。

不变量 6（Unix 共用代码行为不变）没有专属用例：它由 Linux 上既有的各领域测试继续承担，删除 macOS 不新增
需要单独证明的行为；desktop PR 的 Linux CI 全绿是它的回归门槛，不登记为某条义务的直接证据。

## A Release Publishes Only Windows And Linux Artifacts

### 风险

发布流程仍为 macOS 构建或在 `latest.json` 里留下 darwin 条目，就要继续维护无人验证的 macOS 产物和签名；反过来，
生成更新清单时漏掉 Windows 或 Linux，会让受支持平台的客户端收不到更新。

### 前置状态

一次带版本 tag 的发布，Windows 与 Linux 打包 job 各自产出已签名的更新资产；可制造其中一个平台缺少签名。

### 触发

`desktop-build` 按 tag 构建并汇总产物；生成 `latest.json`。

### 必须成立

- 打包只在 Windows 和 Linux runner 上运行，产物只有 NSIS、AppImage 与 deb。
- `latest.json` 的 `platforms` 恰好是 `windows-x86_64` 与 `linux-x86_64`。
- 任一平台缺少已签名资产时生成失败，已有的 `latest.json` 不被覆盖。

### 禁止结果

- 发布里出现 `dmg`、`.app.tar.gz` 或任何 macOS runner 的产物。
- `latest.json` 含 `darwin-*` 条目，或缺少 Windows、Linux 之一仍生成成功。
- 为了这次发布撤回或改写 v0.6.0 及更早的 macOS 资产。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 更新清单只列 Windows 与 Linux，缺签名时失败且不覆盖旧清单 | [`scripts/create-desktop-update-manifest.test.ts:19`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/scripts/create-desktop-update-manifest.test.ts#L19) `updater manifest contains complete signed targets and escaped download URLs`：只放 Windows、Linux 两份已签名资产，断言整份清单的 `platforms` 恰为 `windows-x86_64` 与 `linux-x86_64`；生成器对每个列出的平台都要求恰好一份已签名资产，多列一个平台（如 darwin）这条测试就会失败。[`scripts/create-desktop-update-manifest.test.ts:53`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/scripts/create-desktop-update-manifest.test.ts#L53) `missing signatures do not overwrite an existing manifest`：删掉 Linux 签名后生成失败，旧 `latest.json` 原样保留 | `Covered` |
| PR CI、打包矩阵与 bundle 目标只覆盖 Windows、Linux | 没有静态测试检查 `tauri.conf.json` 的 bundle targets 与 `desktop-build.yml`、`pr-tests.yml` 的矩阵只含 Windows、Linux；当前只靠这些文件的内容本身。合并前在 PR 分支上手动跑的 `desktop-build`（`workflow_dispatch`，`release=false`，在 Windows 上产出 NSIS 安装包）是运行记录，不算这项义务的证据 | `Missing` |

### 决策依据

ADR D1、D3，不变量 1。

## A Listing With Darwin And Linux Targets Installs On Linux

### 风险

把退役的 darwin target 当成未知 triple 报错，会让带 darwin 资产的已发布清单（三个官方 agent）在所有平台上整份解析
失败；把它当成可选 target，又可能让宿主选中一个不能运行的资产。

### 前置状态

一份 release 清单的 `[[targets]]` 同时含 `aarch64-apple-darwin`、`x86_64-apple-darwin` 与 Linux、Windows target；另有
一份含拼写未知的 triple。宿主为 Linux x86_64。

### 触发

marketplace 同步解析清单并建索引；在 Linux 宿主上安装该版本。

### 必须成立

- 清单解析成功，release 来源只含受支持的 target，顺序与原文一致。
- Linux 宿主选中 `x86_64-unknown-linux-gnu` 资产，下载、校验并安装成功。
- 同一份清单在 Windows 与 Linux 上解析出同一个结果。
- 拼写未知的 triple 仍使清单报错，错误原因与退役 target 不同。
- 被丢弃的 darwin 项只检查 TOML 结构（`target`、`url`、`sha256` 都是字符串）：`url`、`sha256` 的取值不校验，重复的
  darwin 项不报错。

### 禁止结果

- 因 darwin 项整份清单解析失败、条目从索引消失或显示为不支持当前平台。
- darwin 项出现在已编译的 release 来源里，或被任何宿主选中。
- 未知 triple 被当作退役 target 静默跳过。
- 因 darwin 项的取值非法或重复而拒绝整份清单；或因丢弃而放过 darwin 项缺字段、类型不符的结构错误。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| darwin 项被丢弃，其余 target 保持原样且可选 | [`crates/plugin-manifest/src/tests.rs:1419`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manifest/src/tests.rs#L1419) `skips_retired_macos_artifacts_in_published_listings`：两个 darwin 项与 Linux、Windows 项交错的 agent 清单解析成功，release 来源恰为 Linux、Windows 两项且保持原顺序。丢弃发生在解析中，没有平台分支，同一份清单在两个平台上结果相同 | `Covered` |
| 被丢弃项只检查结构：取值不校验、重复不报错，缺字段仍报错 | 没有直接测试：上一行测试里的 darwin 项取值合法、triple 各不相同；没有测试覆盖 darwin 项取值非法、重复或缺 `url`/`sha256` 的情形 | `Missing` |
| Linux 宿主经真实安装路径装上混合清单的 Linux 资产 | 没有直接测试：`crates/plugin-manager/src/install.rs` 的安装测试只用单一 target 的清单，`crates/backend/src/plugin.rs` 的 RTK E2E 用只有 Windows target 的清单，且未设置 `RTK_RELEASE_ORAX` 时跳过 | `Missing` |
| 退役与未知 triple 分开报告，未知仍报错 | [`crates/plugin-manifest/src/target.rs:176`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manifest/src/target.rs#L176) `reports_macos_triples_as_retired`：两个 darwin triple 都报 `Retired`；[`crates/plugin-manifest/src/target.rs:145`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manifest/src/target.rs#L145) `rejects_invalid_target_triples`：未知 triple 报 `Unsupported`；[`crates/plugin-manifest/src/tests.rs:1339`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manifest/src/tests.rs#L1339) `rejects_unsupported_target_triples`：release 清单的未知 triple 仍以 `targets[0].target` 的 `Unsupported` 拒绝 | `Covered` |

### 决策依据

ADR D4，不变量 3、5；[插件清单根决策](../../../decisions/desktop/plugin/manifest/0-versioned-dual-form-manifest.md)不变量 8。

## A Darwin-Only Listing Has No Release

### 风险

只剩 darwin 项的清单若被当成解析错误，条目会从 marketplace 消失；若被当成有发布来源，UI 会给出一个必然失败的
安装入口。

### 前置状态

一份 hook 或 agent 的 release 清单，没有顶层 `url`，`[[targets]]` 只有 darwin 项。marketplace 索引已由新版本重建（旧缓存
可能仍带 darwin 字符串，只影响不兼容提示文案）。

### 触发

marketplace 同步解析清单并建索引；用户在 Windows 或 Linux 上查看并尝试安装。

### 必须成立

- 清单解析成功，没有发布来源。
- 条目照常索引和展示，本机标为不可安装，理由与“清单没有可下载发布”相同。

### 禁止结果

- 条目解析失败或从索引消失。
- 宿主尝试下载 darwin 资产，或把该条目显示为可安装。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| 只有 darwin 项的清单没有发布来源 | [`crates/plugin-manifest/src/tests.rs:1419`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manifest/src/tests.rs#L1419) `skips_retired_macos_artifacts_in_published_listings`：只有 `aarch64-apple-darwin` 的 agent 清单解析成功，`release_source()` 为 `None` | `Covered` |
| 该条目在索引中可见、不可安装 | 没有直接测试：`PluginEntry::host_compatibility`（`crates/plugin-registry/src/entry.rs:175`）把没有发布来源的非 pack 条目判为“this listing has no downloadable release”，但没有测试用 darwin-only 清单建索引，也没有测试对非 pack 条目断言这一理由；`indexes_a_pack_listing_without_release_targets` 只覆盖 pack | `Missing` |

### 决策依据

ADR D4，不变量 3。

## An Installed Darwin Artifact Is Refused

### 风险

安装包自述 darwin `[artifact]` 说明里面是 macOS 二进制；若像 release 清单那样跳过，它会被当成没有 target 的通用包
装上，在受支持的宿主上运行失败。

### 前置状态

一个 hook 或 agent 安装包的 `orax.toml` 声明 `[artifact] target = "aarch64-apple-darwin"`。

### 触发

本地导入或包发现解析 installed 清单；安装时校验包内清单。

### 必须成立

- 解析失败，错误字段为 `artifact.target`，原因为退役 target，而不是未知 triple。
- 安装不留下任何安装状态或解包残留。

### 禁止结果

- 跳过 `[artifact]` 后把包当作通用包安装。
- 把退役原因报成未知 triple，或报成其他字段的错误。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| installed 清单的 darwin `[artifact]` 以退役原因被拒绝 | [`crates/plugin-manifest/src/tests.rs:1469`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manifest/src/tests.rs#L1469) `rejects_installed_macos_artifact`：`parse_installed` 以字段 `artifact.target`、原因 `InvalidHookTarget(Retired)` 拒绝 `aarch64-apple-darwin` | `Covered` |
| 安装路径遇到该包时失败且零残留 | 只有较低层的共用路径：安装期用同一个 `PluginManifest::parse_installed` 校验包内清单，[`crates/plugin-manager/src/install.rs:963`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/crates/plugin-manager/src/install.rs#L963) `rejects_a_package_manifest_missing_a_required_field` 证明包内清单解析失败时安装失败，并经 `assert_no_install_residue`（`install.rs:795`）确认零残留；没有测试安装一个自述 darwin `[artifact]` 的包 | `Partial` |

### 决策依据

ADR D4，不变量 4、5；[安装期包内清单校验](../../../decisions/desktop/plugin/registry/20260922-install-time-package-validation.md)的零残留规则。

## A macOS Host Gets No Sidecar Target

### 风险

`setup-binary` 在 macOS 主机上退回某个默认 target，会下载一个能通过校验、却不能在本机运行的 sidecar，把平台错误
推迟成运行时故障；保留 darwin 校验值则意味着仍在维护 macOS 产物。

### 前置状态

在 macOS（`darwin`，arm64 或 x64）主机上运行 `setup-binary`，没有设置 `TARGET_TRIPLE` 或 `RUST_TARGET`。

### 触发

为任一 sidecar 执行 `setup-binary`。

### 必须成立

- 以 `Unsupported platform: darwin-<arch>` 失败，不下载任何文件。
- `sidecar-checksums.json` 没有 darwin 条目。
- Windows 与 Linux 主机的默认 target 与校验流程不变。

### 禁止结果

- 在 macOS 上推导出 darwin 或其他平台的默认 target 并继续下载。
- 校验表里残留 darwin sidecar。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| macOS 主机没有默认 target，报不支持的平台 | [`scripts/setup-binary.test.ts:261`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/scripts/setup-binary.test.ts#L261) `unknown pins and conflicting versions fail without running commands`：不设 target 环境变量、`platform: "darwin"`、`arch: "arm64"` 时以 `Unsupported platform: darwin-arm64` 失败，测试末尾断言没有执行任何命令（不下载）。只取 arm64 样本；推导默认 target 的代码已没有 darwin 分支，x64 走同一条报错路径 | `Covered` |
| Windows、Linux 的下载与缓存校验保持原样 | [`scripts/setup-binary.test.ts:97`](https://github.com/ora-space/desktop/blob/1456af721bad5577b628ac0d5d9ead14eb8b4809/scripts/setup-binary.test.ts#L97) `${name} ${target} verifies downloads and reuses only verified cache`：`deno`、`rg` 在 `x86_64-unknown-linux-gnu` 与 `x86_64-pc-windows-msvc` 上下载、校验并只复用已校验的缓存 | `Covered` |
| `sidecar-checksums.json` 没有 darwin 条目 | 没有直接测试：`setup-binary` 测试的 fixture 写入自己的校验表，不读取仓库里的 `scripts/sidecar-checksums.json`；当前只靠该文件的内容本身 | `Missing` |

### 决策依据

ADR D3，不变量 2。
