# Extension Pack：安装编排核心用例

对应决策：[插件集合包决策](../../../../decisions/desktop/plugin/extension-pack/0-extension-pack.md)
的 D4（agent 过滤）、D5（preflight + 顺序安装）、D6（preflight 零下载；成员失败停止且不
回滚）、D7（pack 不进入已安装模型）与不变量 3、5、6、7、10。

前置状态：marketplace source 已同步，pack 清单与成员清单均已在源检出中。安装编排仅替代
一次传输环节（本地制品替代 HTTPS 下载，沿用 RTK E2E 的既有先例），摘要校验与全部落盘
路径保持生产实现。

## Pack installs install applicable members in declaration order

- **风险**：成员安装若脱离既有单插件链路（自建下载、自建落盘），SHA-256、staging 与原子
  提交的保证将不复存在；若顺序不按声明序，聚合进度与失败定位都会失真。
- **触发**：对包含一个隐藏成员与一个可见成员的 pack 执行安装编排。
- **必须成立**：
  - 每个入选成员按声明顺序走完整单插件链路（下载 → 摘要校验 → staging → 包校验 → 原子
    提交 → finalization）；
  - 隐藏成员（`marketplace_visible = false`）照常被安装，安装后是真实 Installed Plugin；
  - 每个 finalization 与单插件安装完全一致（Skills、快照、MCP 状态即时生效）。
- **禁止结果**：任何绕过 `Installer` 校验的落盘路径；成员安装顺序与声明顺序不一致。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 两个全新成员按声明顺序安装且逐成员 outcome 为 `Installed` | `crates/backend/src/plugin/operations/install_tests.rs` `pack_install_installs_every_applicable_member_in_declaration_order` | Covered |
  | 隐藏成员被 pack 安装为真实 Installed Plugin | 同上（断言 `installed/official/ora-space.python-core/1.0.0` 存在） | Covered |
- **决策依据**：D5 第 2 步、不变量 10。

## Already installed members are skipped without touching their versions

- **风险**：已安装成员若被重装或降级，用户既有的插件版本会被破坏；若被记为失败，pack 的
  "补齐缺失成员"语义就变成不可重入。
- **触发**：预先安装其中一个成员，再执行 pack 安装编排。
- **必须成立**：
  - preflight 将其列入 already-installed；安装循环跳过它（不改动既有版本目录）；
  - 其余成员照常安装；
  - 聚合结果中它出现在 `skipped`，不算失败。
- **禁止结果**：已安装成员的版本被改动、重装或报错。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | preflight 识别 + 安装循环跳过 + `skipped` 如实上报 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_install_skips_an_already_installed_member` | Covered |
- **决策依据**：D5（已安装成员跳过并继续）、D7（重装 pack 等价于补齐缺失成员）、不变量 5。

## Preflight failures leave the installed tree untouched

- **风险**：静态问题若在安装中途才暴露，会产生部分下载与半完成状态，破坏"第一个成员开始
  安装前发现所有静态问题"的承诺。
- **触发**：分别以重复成员、自引用成员、不存在的成员、嵌套 pack、无适用成员（agent 引用
  无一命中）执行 preflight。
- **必须成立**：
  - 每种场景返回封闭的 public error 变体（duplicate / self-reference / member-not-found /
    nested / no-applicable-members）；
  - preflight 失败不发生任何下载，`installed/` 不发生变化。
- **禁止结果**：任一静态问题在成员开始安装后才出现；preflight 失败留下任何落盘痕迹。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 五类静态问题分别映射到五个 public error 变体 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_preflight_failures_leave_the_installed_tree_untouched` | Covered |
  | preflight 失败后 `installed/` 不存在 | 同上（断言安装树未创建） | Covered |
- **决策依据**：D6 第 1 段、不变量 3。

## A member failure stops the run without rollback and is reported per member

- **风险**：成员失败若伪装成整包失败，调用方无法知道哪些成员已就位；若自动回滚，刚装好的
  可用插件会被破坏。
- **触发**：第二个成员的制品与其清单摘要不符（下载校验失败），第一个成员已安装成功。
- **必须成立**：
  - 编排返回 `Ok(PackInstalled)`：`members` 携带失败前已装成员（含其单插件 outcome），
    `failed` 携带首个失败成员与错误码，剩余成员不再尝试；
  - 已落盘成员保留，不回滚。
- **禁止结果**：整体布尔失败掩盖部分成功；已安装成员被自动卸载。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 首个失败成员停止运行、已装成员保留并在结果中可辨 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_install_stops_at_a_failing_member_without_rolling_back` | Covered |
- **决策依据**：D6 第 2 段、不变量 6、7。

## Agent-gated members join the install only when a referenced agent is installed

- **风险**：agent 过滤若误将"引用未命中"当错误，或把未命中成员也安装，都会违背"按 agent
  适配安装"的需求。
- **触发**：成员声明 `agents = ["ora-space.codex"]`：分别在没有已安装 agent 与已安装
  `ora-space.codex` 的情况下执行安装编排。
- **必须成立**：
  - 无匹配 agent 时，成员不入选；若无任何成员入选，preflight 以
    `PackNoApplicableMembers` 失败；
  - 已安装匹配 agent 后，成员入选并正常安装；引用值不进入任何持久化状态。
- **禁止结果**：未命中引用报错；未命中成员被安装。
- **验证义务与证据**：
  | 义务 | 证据 | 状态 |
  | --- | --- | --- |
  | 无匹配 → `PackNoApplicableMembers`；有匹配 → 成员入选并安装 | `crates/backend/src/plugin/operations/install_tests.rs` `pack_preflight_failures_leave_the_installed_tree_untouched`（无匹配分支）、`pack_install_applies_an_agent_gated_member_whose_agent_is_installed`（匹配分支） | Covered |
- **决策依据**：D4、不变量 8。
