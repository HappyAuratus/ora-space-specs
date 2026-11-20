# 支持平台的核心用例

依据[Ora Desktop 只支持 Windows 和 Linux](../../../decisions/desktop/platform/0-windows-and-linux-only.md)。下文的 D 编号和
不变量编号都指这篇 ADR。

ADR 为 `proposed`，实现在 desktop#804（分支 `refactor/1367-drop-macos`，运行 ora-space/circus#1367），尚未合并。引用未合并
代码的证据留到落地运行登记，下列义务目前都是 `Missing`。证据路径相对 desktop 仓库根目录。

| 不变量 | 用例 | 状态 |
|---|---|---|
| 1 | [A Release Publishes Only Windows And Linux Artifacts](#a-release-publishes-only-windows-and-linux-artifacts) | Missing |
| 3、5 | [A Listing With Darwin And Linux Targets Installs On Linux](#a-listing-with-darwin-and-linux-targets-installs-on-linux) | Missing |
| 3 | [A Darwin-Only Listing Has No Release](#a-darwin-only-listing-has-no-release) | Missing |
| 4、5 | [An Installed Darwin Artifact Is Refused](#an-installed-darwin-artifact-is-refused) | Missing |
| 2 | [A macOS Host Gets No Sidecar Target](#a-macos-host-gets-no-sidecar-target) | Missing |

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
| 更新清单只列 Windows 与 Linux，缺签名时失败且不覆盖旧清单 | 尚无已合入证据；待 desktop 实现合并后登记更新清单生成脚本的测试 | `Missing` |
| PR CI、打包矩阵与 bundle 目标只覆盖 Windows、Linux | 尚无直接证据；需要一个检查 `tauri.conf.json` bundle targets 与 workflow 矩阵只含 Windows、Linux 的静态测试，发布运行记录归档在 circus issue | `Missing` |

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

### 禁止结果

- 因 darwin 项整份清单解析失败、条目从索引消失或显示为不支持当前平台。
- darwin 项出现在已编译的 release 来源里，或被任何宿主选中。
- 未知 triple 被当作退役 target 静默跳过。

### 验证义务与证据

| 验证义务 | 直接证据 | 状态 |
|---|---|---|
| darwin 项被丢弃，其余 target 保持原样且可选 | 尚无已合入证据；待 desktop 实现合并后登记清单解析测试 | `Missing` |
| Linux 宿主经真实安装路径装上混合清单的 Linux 资产 | 尚无直接证据；需要从 registry 清单到安装完成的测试 | `Missing` |
| 退役与未知 triple 分开报告，未知仍报错 | 尚无已合入证据；待合并后登记 `HookTarget` 的退役与未知 triple 测试 | `Missing` |

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
| 只有 darwin 项的清单没有发布来源 | 尚无已合入证据；待 desktop 实现合并后登记清单解析测试 | `Missing` |
| 该条目在索引中可见、不可安装 | 尚无直接证据；需要索引构建与可安装性判定的测试 | `Missing` |

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
| installed 清单的 darwin `[artifact]` 以退役原因被拒绝 | 尚无已合入证据；待 desktop 实现合并后登记 installed 清单解析测试 | `Missing` |
| 安装路径遇到该包时失败且零残留 | 尚无直接证据；需要安装期包内清单校验的测试 | `Missing` |

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
| macOS 主机没有默认 target，报不支持的平台 | 尚无已合入证据；待 desktop 实现合并后登记 `setup-binary` 测试 | `Missing` |
| Windows、Linux 的下载与缓存校验保持原样 | 尚无已合入证据；待合并后登记 `setup-binary` 的各平台校验测试 | `Missing` |

### 决策依据

ADR D3，不变量 2。
