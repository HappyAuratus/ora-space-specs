# Hook 插件生命周期命令

决策依据：[Hook 插件根决策](../../../../../decisions/desktop/plugin/category/hook/0-hook-plugin-pack.md)（`implemented`）。

本文件记录 Hook 插件从声明、授权到执行与失败的长期系统承诺。Hook 是第一个会在宿主侧产生真实外部
副作用的静态 kind，因此这里关注的不是"包能不能装上"，而是"包内程序在什么条件下、以什么身份、对
谁的文件做了什么"。

## Hook Declarations Must Resolve To A Contained Executable With Declared Lifecycle Commands

### 风险

如果 `hook` shape 仍能表达旧的别名或协议字段，或者可执行文件可以在校验之后被替换，宿主执行的就
不再是被验证过的那份程序。声明是宿主唯一能据以定位命令的来源，它必须收敛到"哪个包内文件、带什么
参数"，而不是留下需要宿主解释的语义字段。

### 前置状态

- installed package 的 `orax.toml` 声明 `kind = "hook"` 并携带 `[artifact].target`。
- `assets/config.json` 可能是新的 `hook` shape、旧 shape（含 `protocol` / `command` / `toolVersion`），
  或与 `transport` / Settings-only 混合的形态。
- `assets/` 下的可执行文件可能缺失、是目录、是符号链接，或以其他方式逃出包目录。

### 触发

`PluginManager::discover` 编译 `assets/config.json` 并验证 Hook contribution，随后宿主在授权后
定位并执行生命周期命令。

### 必须成立

- `hook` shape 只接受 `executable`、可选的 `supportedAgents` 与 `lifecycle`；`lifecycle.init` 必填，
  `lifecycle.deinit` 可选。
- `executable` 必须是当前 installed package `assets/` 内的普通文件：拒绝绝对路径、父目录穿越、
  仅指向 `assets/` 目录、符号链接/reparse point 逃逸，以及包外路径。
- `supportedAgents` 只做形状校验，不参与任何宿主判定，也不与宿主已知的 Agent 身份对齐。
- 旧的 `protocol` / `command` / `toolVersion` 被整包拒绝，错误指向被移除的字段。
- 三 shape 互斥（`transport`、`hook`、Settings-only）保持不变。

### 禁止结果

- 把旧声明按新语义解释，或忽略被移除字段后继续安装。
- 接受 PATH 解析的裸命令、绝对路径或包外路径作为 `executable`。
- 用 `supportedAgents` 拒绝安装、生成状态或筛选执行对象。
- 因最高 installed version 损坏而静默回退到较低版本。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 新 `hook` shape 编译，`init` 必填、`deinit` 与 `supportedAgents` 可省略 | `ora_plugin_config::hook::tests::compiles_the_rtk_hook_configuration`; `compiles_hook_configuration_without_optional_members`; `rejects_missing_init_phase` | `Covered` |
| `executable` 参数与生命周期参数的数量、长度与控制字符策略生效 | `ora_plugin_config::hook::tests::rejects_invalid_lifecycle_arguments`; `rejects_invalid_deinit_arguments_independently_of_init`; `rejects_oversized_and_excessive_lifecycle_arguments`; `rejects_executable_path_violations` | `Covered` |
| 旧字段被整包拒绝且错误指向被移除的字段 | `ora_plugin_config::hook::tests::rejects_descriptor_fields_removed_by_the_pack_decision`; `ora_plugin_manager::kind_tests::rejects_hook_package_declared_with_removed_descriptor_members` | `Covered` |
| `hook` shape 与 `transport` 不得同文件混用 | `ora_plugin_config::mcp::tests::rejects_a_mixed_transport_and_hook_declaration` | `Covered` |
| 包内可执行文件 containment 拒绝逃逸、目录与包外路径 | `ora_plugin_manager::kind_tests::rejects_hook_executable_containment_and_extension_violations`; `rejects_hook_executable_symlink_escape`; `rejects_hook_executable_that_is_a_directory`; `discovers_hook_package_with_executable` | `Covered` |
| `supportedAgents` 不影响任何宿主判定 | `ora_plugin_config::hook::tests::rejects_invalid_supported_agents` 只证明语法校验；"不参与宿主判定、不与本机 Agent 身份对齐"没有直接证据 | `Partial` |

### 决策依据

- Hook 插件根决策 D1、D3：Hook 携带包内可执行文件，适配知识归工具，`supportedAgents` 仅是展示信息。
- Hook 插件根决策不变量 2、9、14：执行来源必须是当前已安装包，包内文件校验沿用既有决策。

## Hook Lifecycle Commands Must Only Run On Explicitly Authorized Single-Plugin Operations

### 风险

Hook 的执行面是宿主以当前用户权限运行第三方程序。如果批量流程（pack 安装、pack 卸载、安装回滚）
会顺带执行成员的 `init`/`deinit`，或者缺少授权声明的请求也会执行，用户就在没有逐个确认的情况下
授权了多个作者的程序，并且无法把副作用对应回任何一次点击。

### 前置状态

- 一个已安装的 Hook 插件，或一个包含 Hook 成员的 pack。
- 请求可能带或不带执行授权声明。
- 插件可能在本次会话中从未执行过生命周期命令。

### 触发

用户发起单插件安装、本地导入、更新、显式初始化或卸载；或用户安装/卸载包含 Hook 成员的 pack。

### 必须成立

- 只有请求显式声明授权时才执行；缺省为拒绝，即使包携带完整 `lifecycle` 声明。
- 单插件安装与本地导入在安装成功后执行 `init`；更新每次都执行 `init`。
- 卸载在插件包目录被移除**之前**执行 `deinit`。
- 显式初始化只针对单个插件，并同样要求授权。
- pack 安装只落盘成员包，不执行任何成员的 `init`；pack 卸载与安装回滚不执行 `deinit`。

### 禁止结果

- 未带授权声明的请求执行任何生命周期命令。
- pack 安装、pack 卸载、回滚或后台流程触发 Hook 执行。
- `init` 失败后自动重试，或在后台定时重试。
- 在包目录已进入卸载暂存之后才执行 `deinit`，或改为在暂存目录中执行。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 未带授权的请求不执行生命周期命令 | `ora_backend::plugin::operations::install_tests::installing_a_hook_without_authorization_runs_nothing`; `initializing_a_hook_without_authorization_is_refused`; `uninstall_without_authorization_runs_nothing` | `Covered` |
| 带授权的安装与更新执行 `init` | 安装与本地导入：`an_authorized_install_runs_init_once`；**更新路径无直接证据** | `Partial` |
| 安装与更新前的确认框披露执行面 | `apps/desktop/web` 前端设置页 `discloses hook execution before installing a marketplace hook`; `discloses hook execution again before updating an installed hook` | `Covered` |
| pack 安装不执行成员 `init`，成员显示为未初始化 | `ora_backend::plugin::operations::install_tests::a_pack_install_does_not_run_a_hook_member` | `Covered` |
| pack 卸载与安装回滚不执行 `deinit` | 无直接证据（pack 卸载链路未断言 `deinit` 未被调用） | `Missing` |
| 卸载在包目录仍在原位时执行 `deinit`，且不因 `deinit` 失败而拒绝卸载 | `uninstall_runs_deinit_before_the_package_is_removed` 覆盖顺序与"可执行文件仍在原位"；**`deinit` 失败仍完成卸载无直接证据** | `Partial` |
| 显式初始化可重复调用且只作用于目标插件 | `a_failed_init_keeps_the_hook_installed_and_retryable` 覆盖重复调用；"只作用于目标插件"未单独断言 | `Partial` |

### 决策依据

- Hook 插件根决策 D2：触发点绑定在单个插件的用户操作上，pack 成员由用户单独授权。
- Hook 插件根决策 D4、D5：授权是请求上的显式声明，且与安装前的披露同源。
- Hook 插件根决策不变量 3、12：缺省拒绝，批量流程不执行。

## Hook Lifecycle Execution Must Use The Installed Package Executable In The Minimal Environment

### 风险

执行环境的每一项都是潜在的越权通道：工作目录决定相对路径落在哪里，环境变量决定工具看到什么，
参数决定凭据是否出现在进程命令行里。如果宿主在这里"顺手帮忙"——传入设置实值、Agent 发现结果或
额外的上下文，它就重新拥有了本决策明确交给工具的判断权。

### 前置状态

- 已授权的 Hook 插件已安装，`init` 或 `deinit` 待执行。
- 用户可能已填写 Hook 的 Settings。
- 包内可执行文件可能在安装校验之后被外部替换、删除或改为符号链接。

### 触发

宿主在授权后的安装、更新、显式初始化或卸载流程中执行生命周期命令。

### 必须成立

- 执行前重新定位当前 installed package 的根目录，并重新校验可执行文件仍是该包 `assets/` 内的
  普通文件；校验不通过时不执行。
- 参数严格等于声明中为该阶段写下的固定参数；不追加、不替换、不拼接，也不经过 shell。
- 工作目录是当前用户家目录；环境变量继承宿主进程环境。
- 标准输入关闭；标准输出与标准错误被捕获并限长。
- Hook 的 Settings 实值、凭据和 Agent 发现结果都不进入命令参数或环境。

### 禁止结果

- 用安装期缓存的可执行文件路径直接执行而不复校。
- 通过 shell 解释参数，或把设置值拼进命令行。
- 为 Hook 启动插件 runtime、注册 host method 或授予插件权限。
- 把宿主自身写入的任何路径当作 Hook 的配置写入位置。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| 执行前复校可执行文件仍在当前包内，替换/逃逸后拒绝执行 | `ora_backend::plugin::hook_lifecycle::tests::re_validates_containment_before_every_spawn` | `Covered` |
| 参数等于声明的固定参数且不经过 shell | `runs_the_declared_command_in_the_host_directory` 逐参数断言 `Vec<String>`，无 shell 展开 | `Covered` |
| 工作目录为用户家目录、环境继承宿主、标准输入关闭 | `runs_the_declared_command_in_the_host_directory` 只断言工作目录；**环境继承与标准输入关闭无直接证据** | `Partial` |
| Settings 实值与凭据不出现在命令行与环境 | 无直接证据 | `Missing` |
| Hook 仍不进入插件 runtime | `ora_plugin_lifecycle::registration::tests::hook_plugins_cannot_register` | `Covered` |
| 宿主自身不写 Agent 配置文件 | 无直接证据（须通过真实安装路径观察文件系统副作用） | `Missing` |

### 决策依据

- Hook 插件根决策 D1、D4：命令只由包内可执行文件与固定参数组成，宿主不解释工具语义。
- Hook 插件根决策 D5、D6：执行面在插件沙箱之外，副作用归属由工具承担。
- Hook 插件根决策不变量 2、5、13。

## Hook Lifecycle Failure Must Stay Visible Without Changing Install State Or Blocking Other Flows

### 风险

生命周期命令失败的最坏处置有两种：把失败藏进日志（用户以为 Hook 已生效），以及把失败升级成安装
失败（包已落盘却被回滚，用户无法重试，pack 的其他成员也受牵连）。执行结果一旦被当成持久状态，
还会在工具自己改过配置之后继续显示一个不再成立的事实。

### 前置状态

- 已授权的 Hook 插件正在执行 `init` 或 `deinit`。
- 命令可能以非零退出码失败、挂起不退出、输出大量内容，或工具未提供 `deinit`。

### 触发

宿主执行生命周期命令并收集退出码与输出，随后更新已安装列表中的执行结果。

### 必须成立

- 失败不改变插件的安装状态：插件保持已安装、包与声明仍然有效。
- 失败对用户可见，并带阶段、退出码与错误输出等可定位信息。
- 失败可由用户通过显式初始化重试；宿主不自动重试。
- 失败不阻断会话创建、prompt、插件管理、其他插件安装，或本次安装中其他 pack 成员的安装。
- 执行超时后终止进程树并按失败报告；输出超长时截断并标记。
- 工具未提供 `deinit` 时，卸载不执行命令，其写入的配置保持原样，卸载结果如实说明。
- 执行结果只存在于当前会话内存与结构化日志中，不作为跨重启状态。

### 禁止结果

- 失败后回滚插件安装、把包标记为损坏或从已安装列表中移除。
- 只写日志而不做用户可见反馈。
- 让失败的命令永远挂起安装或卸载流程。
- 未经工具授权而清理 Agent 配置，或因为无法清理而拒绝卸载。
- 把上次会话的执行结果当作当前事实展示。

### 验证义务与证据

| 验证义务 | 代表性证据测试 | 状态 |
| --- | --- | --- |
| `init` 失败后插件仍为已安装，且可由用户重试成功 | `ora_backend::plugin::operations::install_tests::a_failed_init_keeps_the_hook_installed_and_retryable` | `Covered` |
| 失败带阶段、退出码与输出，并呈现给用户 | `ora_backend::plugin::hook_lifecycle::tests::records_a_non_zero_exit_as_a_failure_with_the_code`; `reports_a_command_that_could_not_start`；前端 `keeps a failed hook initialization visible and retries it` | `Covered` |
| 失败不阻断其他插件与 pack 其他成员的安装 | 无直接证据 | `Missing` |
| 超时终止进程树并按失败报告 | `reports_a_command_that_exceeded_the_time_limit` | `Covered` |
| 输出超长被截断且标记截断 | `bounds_the_output_a_report_carries`；前端在失败明细中呈现截断标记 | `Covered` |
| 工具未提供 `deinit` 时保留其配置并如实说明 | `skips_an_undeclared_deinit`；前端 `states the hook teardown before uninstalling and authorizes it once confirmed` | `Covered` |
| 执行结果不持久化，重启后呈现为未知而非失败 | `keeps_only_the_latest_result_per_plugin` 只证明会话内存中按插件保留最新一次；**跨重启后呈现为未知无直接证据** | `Partial` |

### 决策依据

- Hook 插件根决策 D6、D8：失败语义、记录内容与记录边界。
- Hook 插件根决策不变量 7、8、10、11。
