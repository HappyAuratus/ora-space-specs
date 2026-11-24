---
status: proposed
---

# Desktop 项目是本机目录，Git 是按目录实时探测的可选能力，而不是项目前提

Desktop 的项目是一个本机目录：添加项目只要求目录存在，会话、Agent 就地读写和文件浏览不依赖 Git。Git 是该目录
可能具备的一项能力，由后端在每次列出 Workspace 时对目录实时探测，得到 `repository` / `notRepository` /
`unknown` 三态之一，不从存储的项目数据推断、也不缓存。只有 Git 明确回答“不在任何仓库内”且目录及其祖先都没有
`.git` 条目时才判为 `notRepository`；其余无法确定的情况一律是 `unknown`，客户端按原有行为处理。依赖 Git 的入口
据此降级：变更面板改为说明、worktree 任务入口隐藏、Git 操作在后端返回带类型的 `workspace_not_git_repository`、
定时任务表单拦截后端必然拒绝的保存。

本 ADR 为 `proposed`，等待 owner 评审。所述行为已由 ora-space/desktop#812 实现并合并；该 PR 先于本决策合并是
owner 在 ora-space/circus#1400 下的安排。涉及的模块是 Desktop 的 workspace/project（契约 `WorkspaceDirectory`、
`WorkspaceGit`，后端 Workspace 列表与 Git 审阅操作，gitlancer 的工作树探测）以及使用它的变更面板、worktree
任务入口和定时任务表单。兼容性边界：已有 Git 项目的行为不变；`unknown`、目录缺失、远程 Workspace 和列表失败
都回落到原有行为。本 ADR 只覆盖 Desktop 本机目录，不决定 Cloud 是否支持目录项目（见 D8），也不承担任何数据
迁移：探测结果从不落盘，没有需要迁移的历史状态。

## 决策

### D1：项目是本机目录，Git 不是项目的前提

Desktop 项目以一个本机目录作为主 Workspace 的位置。添加项目只校验目录存在，不要求它是 Git 仓库，也不替用户执行
`git init`。在普通目录（没有 `.git`）上，以下能力与 Git 项目相同：开启会话、Agent 在目录中直接读写文件、文件
面板浏览与引用。依赖 Git 的能力（变更审阅、worktree 任务、暂存/提交/推送、定时任务）按 D6 降级，而不是阻止
项目存在。

这样选的原因：

- 用户把文档、资料、脚本目录交给 Agent 处理是正当场景；要求先建仓库会把“能不能用 Ora”绑在一个与任务无关的
  前提上。
- 自动 `git init` 会在用户目录里留下用户没要求的 `.git`，改变其他工具（编辑器、同步盘、外层仓库）对该目录的
  判断，且无法安全地自动撤销。
- 项目存储不记录“这是不是 Git 项目”作为事实：用户可以随时在 Ora 之外 `git init` 或删除 `.git`，任何存储的标记
  都会过期（见 D2）。

### D2：Git 能力在读取时按目录探测，三态表达，不存储

`listWorkspaces` 对每个本机目录存在的 Workspace 返回 `directory: { kind: "present", git }`，`git` 即
[Workspace Git 能力](../../../../domain/glossary.md#workspace-git-capabilityworkspace-git-能力)，取值：

| `git` | 含义 | 客户端行为 |
| --- | --- | --- |
| `repository` | 从该目录运行的 Git 命令作用于一个工作树（含外层 checkout 与 linked worktree） | 原有行为 |
| `notRepository` | Git 明确回答目录不在任何仓库内，且目录与所有祖先都没有 `.git` 条目 | 按 D6 降级 |
| `unknown` | Git 无法给出确定回答 | 原有行为，让操作自身的错误照常暴露 |

规则：

- 探测对象是每个列出的 Workspace（项目主 Workspace 与 worktree 任务的隔离 Workspace）在列出时刻解析出的目录；
  目录不存在时为 `{ kind: "missing", path }`，非本机 checkout 为 `{ kind: "remote" }`，二者都不携带 Git 能力。
- 每次列出都重新探测，结果不缓存、不写库；同一次列出内各 Workspace 以有限并发探测。
- 单次探测有固定时限（5 秒），超时记为 `unknown`。时限约束 Git 进程及其输出，目录 `stat` 不在其内。
- 探测在 Workspace 目录内启动 Git 进程，因此与其他 checkout 读取者一样持有 worktree 使用租约；但它只“尝试”取得
  共享租约：清理正持有或等待该 Workspace 时不等待，直接记为 `unknown`。列表不能被一次删除阻塞，正被清理的
  checkout 也没有值得报告的能力。

为什么在读取时探测而不存储：

- 能力的真实来源是磁盘上的目录与 Git 本身。项目表里的 `repository_kind` 之类数据只描述“添加时是什么”，与当前
  磁盘状态没有同步机制；据此隐藏或展示入口会在用户 `git init` 或删 `.git` 后给出错误判断。
- 每个 Workspace 一次轻量 `rev-parse` 的代价可控，且与 D5 的操作守卫共用同一判定，客户端看到的能力与操作实际
  的行为一致。
- 隔离 Workspace 也探测，而不是按“它是 Ora 建的 worktree”跳过：跳过就是从存储数据推断，正是本决策排除的做法。

### D3：只有明确证据才判为 `notRepository`，其余无法确定的情况都是 `unknown`

探测执行一次 `git rev-parse --is-inside-work-tree`，它与其他 Git 命令一样向上发现仓库。分类如下：

| Git 的结果 | 附加检查 | 分类 |
| --- | --- | --- |
| 退出 0，输出 `true` | — | `repository` |
| 退出 0，输出 `false`（裸仓库、元数据目录） | — | `unknown` |
| 非 0 退出，stderr 含 `not a git repository (or any` | 规范化后的目录及其所有祖先都没有任何 `.git` 条目 | `notRepository` |
| 同上 | 任一层存在 `.git` 条目（含损坏的目录、悬空的符号链接） | `unknown` |
| 同上 | 规范化路径或检查 `.git` 时出现 IO 错误（如无权限） | `unknown` |
| 其他非 0 退出（所有权不受信任、配置损坏、`not a git repository: <path>` 形式的断链 `.git` 文件） | — | `unknown` |
| Git 未安装、超时、输出无法解析、清理进行中 | — | `unknown` |

要点与原因：

- **为什么还要查 `.git` 条目**：Git 跳过损坏或不可读的 `.git` 目录（缺 `HEAD`、无对象、无权限）时，打印的正是
  普通目录那条消息。只看消息会把损坏的仓库说成普通目录，界面随之提示用户“这不是 Git 仓库”，诱导对一个有历史
  的仓库重新 `git init`。查 `.git` 用 `symlink_metadata`，悬空链接也算存在。
- **为什么沿规范化路径查祖先**：Git 沿真实路径向上发现。通过符号链接添加的项目若按字面路径检查，会漏掉目标
  上方损坏的 `.git`。
- **嵌套在外层 checkout 内的普通子目录判为 `repository`**：从该目录运行的 diff、status、commit 实际作用于外层
  checkout，能力报告必须与操作行为一致；把它说成普通目录会隐藏实际可用、且确实会修改外层仓库的操作。
- **消息分类不受用户语言影响**：探测命令单独设置 `LC_ALL=C`，使 Git 的诊断保持英文。只对按消息文本分类的
  命令这样做，其他 Git 命令及其钩子保留用户的语言环境。
- **探测判定刻意比“能否尝试操作”的判定严格**：worktree 任务创建前的仓库校验把任何非 0 退出都当作“不是仓库”，
  因为它只决定是否尝试；探测的结果会展示给用户，必须只接受 Git 的明确回答。两者不得合并。

### D4：`unknown` 与任何缺少答案的状态都保持原有行为

客户端把后端报告折算成一个可用性：

| 可用性 | 来源 | 行为 |
| --- | --- | --- |
| `checking` | 共享 Workspace 列表尚未加载 | 依赖 Git 的读取暂不发出，入口保持加载态 |
| `notRepository` | 列表中目标 Workspace 为 `present` 且 `git: notRepository` | 按 D6 降级 |
| `available` | 其余一切：`repository`、`unknown`、`missing`、`remote`、目标不在列表中、列表请求失败 | 原有行为 |

决定可用性的 Workspace：给定 Workspace 时取该 Workspace；按项目判断（例如“能否新建 worktree”）时取该项目的
主 Workspace，因为 worktree 从它分出。

只有确定的 `notRepository` 会改变界面。理由是不对称的代价：把可用的 Git 功能藏起来，用户无从自救；而在少数
无法确定的情况下保留原有入口，最坏只是看到操作自身的错误，与本决策之前一致。

### D5：Git 审阅操作在后端统一守卫，普通目录返回 `workspace_not_git_repository`

`getWorkspaceDiff`、`getWorkspaceStatus`、`stageWorkspaceChanges`、`unstageWorkspaceChanges`、
`commitWorkspaceChanges`、`pushWorkspaceBranch` 在解析出 Workspace 的实际 checkout 目录后、检查基准提交之前、
运行操作自身的 Git 命令之前，用 D3 的同一探测判定目录：

- `notRepository`：拒绝，返回公开错误 `workspace_not_git_repository`（分类为冲突：请求合法，但与目录当前状态
  不符），不运行任何操作 Git 命令。
- `repository` / `unknown`：照常执行；`unknown` 时真实失败及其诊断与本决策之前一致。

守卫位于这六个操作共用的 checkout 解析处，是“Git 审阅需要工作树”的唯一执行点。放在基准检查之前，是为了让
没有基准的范围（例如主 Workspace 请求 `Branch`）在普通目录上也先说明“不是 Git 仓库”，而不是报告与根因无关的
`workspace_diff_baseline_unavailable`。错误码与能力字段都是 Desktop 内部契约（`PublicError` 与 Workspace 契约），
不构成跨系统协议。

后端守卫不依赖前端隐藏：列表结果可能已过期（用户刚删掉 `.git`），也可能有旧状态打开的视图；无论界面状态如何，
操作都得到同一个可解释的错误，而不是内部错误。

### D6：各依赖 Git 的入口在 `notRepository` 上的降级规则

| 入口 | `notRepository` 时 | 依据 |
| --- | --- | --- |
| 变更面板（Changes） | 显示“该目录不是 Git 仓库”的说明与重新检查按钮，保留面板工具栏以便切到文件面板；不发出 diff 与 status 查询 | 没有可审阅的 diff：Agent 直接写入目录 |
| 会话中指向 diff 的链接 | 打开文件面板中的对应文件；列表仍在 `checking` 时等待，不提前跳走 | 普通目录没有 diff 可定位 |
| 新建 worktree 任务（侧栏新建菜单、输入框分支菜单） | 入口隐藏 | worktree 需要从项目目录用 Git 分出 |
| worktree 任务对话框（被旧状态打开时） | 显示说明，不列分支、不可提交；分支列表等探测结果出来后才请求 | 避免对普通目录发出 Git 分支查询 |
| 后端创建 worktree 任务 | 沿用既有的 `worktree_requires_git_repository` 拒绝 | 创建前的仓库校验先于本决策存在；见 D3 末条 |
| Git 审阅操作 | 后端返回 `workspace_not_git_repository` | D5 |
| 定时任务 | 后端拒绝需要解析或使用项目分支的保存（新建、换到该项目）以及启用；不涉及这些的保存（例如已冻结基准分支的停用任务改名）照常允许。表单恰好拦截后端会拒绝的保存，并在项目选择旁说明原因；列表上的启用开关对普通目录项目改为打开表单；停用始终可用 | 见下 |

定时任务的规则继承[定时任务执行归属根决策](../scheduled-task/0-local-and-cloud-execution-ownership.md)：其 D4、
D8 与不变量 12 已要求保存为 active 的 local 任务以 Git 仓库为目标、每次接纳再次检查、不得降级为直接在原目录
执行。本决策不修改这些规则，只规定普通目录下的表现：后端以既有的定时任务资格校验拒绝（公开错误为 `invalid_request`），
而不是因分支解析失败报告内部错误；前端用 D4 的可用性提前说明，不让用户提交必然失败的保存。前端拦截以后端规则为准，二者不一致时
修正前端，不放宽后端。

不受影响的入口：会话、Agent 读写、文件面板等不依赖 Git 的功能在普通目录上照常工作，不读取 Git 能力。

### D7：能力变化通过重新列出被发现，Git 拒绝会触发重新探测

后端不监听目录变化，也不推送能力变化；客户端看到新能力的时机就是 Workspace 列表被重新获取的时机：

- 共享列表的常规刷新（窗口重新聚焦且缓存已过期、Workspace 事件）；
- 变更面板说明页的“重新检查”按钮，立即重新获取列表；
- 任一 Git 审阅请求被拒为 `workspace_not_git_repository` 时，立即重新获取列表，使已打开的变更面板从“重试”切到
  说明页，而不是反复提供必然失败的重试。

这保证 `git init` 或删除 `.git` 后无需重启即可生效；代价是两次列表之间界面可能短暂沿用旧判断，而 D5 的后端
守卫保证这段时间里的操作仍得到正确的错误。

### D8：本决策只覆盖 Desktop 本机目录，不预判 Cloud 是否支持目录项目

Cloud v2 的项目当前绑定一个外部 HTTPS 仓库分支，会话目录、回合交付与删除回收都以 Git 为前提
（见 [外部仓库拉取决策](../../../cloud/project-git/0-external-repository-pull-and-encrypted-token.md)）。Cloud 是否
以及如何支持非 Git 的目录项目由 ora-space/circus#1444 交 owner 决定，候选为：A 真正的目录项目、B 导入后在受管
副本上建内部 Git 种子、C Cloud 暂不支持。

本决策的边界：

- 只约束 Desktop 本机目录的 Workspace；非本机 checkout 报告为 `remote`，不探测、不降级，保持原有行为。
- `WorkspaceGit` 三态与 `workspace_not_git_repository` 是 Desktop 内部契约，不作为 Cloud 或 Node 协议的一部分，
  也不意味着 Cloud 要采用“读取时探测”的模型。选 B 时 Cloud 项目在执行侧总是 Git 仓库，选 A 时其能力来源也
  可能是项目类型而非探测，这些由 #1444 及其后续 ADR 决定。
- Cloud 侧决策落地后若需要 Desktop 呈现 Cloud 目录项目，应另立后续 ADR，说明它与本决策三态的关系。

## 不变量

1. 添加 Desktop 项目不要求目录是 Git 仓库，Ora 从不自动在用户目录中执行 `git init`。
2. Workspace 的 Git 能力只来自列出时刻对磁盘目录的探测，从不读取或写入存储的项目数据。
3. 只有 Git 明确回答“不在任何仓库内”且规范化目录及其所有祖先都没有 `.git` 条目时，能力才是 `notRepository`。
4. 损坏、不可读或不受信任的仓库、IO 错误、超时、Git 缺失和清理进行中都报告为 `unknown`，从不报告为 `notRepository`。
5. 嵌套在外层 checkout 内的目录报告为 `repository`，Git 审阅操作作用于外层 checkout。
6. 只有确定的 `notRepository` 改变客户端行为；`unknown`、`missing`、`remote`、列表加载中或失败都保持原有行为
   （加载中只推迟 Git 读取，不降级）。
7. Git 审阅的六个操作在 `notRepository` 目录上返回 `workspace_not_git_repository`，且不运行操作自身的 Git 命令、
   不先报告基准缺失。
8. 列表探测从不等待 worktree 清理；正在清理的 checkout 报告为 `unknown`。
9. 在 `notRepository` 项目上不提供新建 worktree 任务的入口，变更面板不发出 diff 与 status 查询。
10. 定时任务表单拦截的保存与后端拒绝的保存一致；active 的 local 定时任务必须以 Git 仓库为目标的规则不变。
11. 探测结果的分类不受用户语言环境影响，其他 Git 命令的语言环境不被改动。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 从存储的项目数据（如 `repository_kind`、默认分支）推断能力 | 零进程开销，但只反映添加时的状态；用户在 Ora 外 `git init` 或删 `.git` 后判断即失效，违反不变量 2。隔离 Workspace 按“Ora 建的 worktree”跳过探测同理 |
| 添加普通目录时自动 `git init` | 后续所有功能都能沿用 Git 链路，实现最省；但在用户目录留下未要求的 `.git`，改变外层仓库、编辑器与同步工具的行为，且无法安全撤销，违反不变量 1 |
| 嵌套在外层 checkout 内的目录当作普通目录 | 更贴合“我添加的是这个子目录”的直觉；但 diff、commit 实际作用于外层仓库，隐藏入口不能阻止 Agent 或用户在那里使用 Git，能力报告会与操作行为相悖，违反不变量 5 |
| 任何 Git 失败都当作普通目录 | 分类最简单，界面也更“干净”；但损坏或不受信任的仓库会被告知“不是 Git 仓库”，诱导重新 `git init`，并掩盖真正需要修复的错误，违反不变量 4 |
| 为所有 Git 命令全局设置 `LC_ALL=C` | 一处设置即可保证所有消息可分类；但 `LC_ALL` 同时覆盖 `LC_CTYPE`，会影响提交、推送触发的用户钩子与非 ASCII 路径处理。只对按文本分类的命令设置，违反面最小 |
| 用 `git worktree list` 或完整的仓库发现做探测 | 能拿到更多信息（分支、worktree 关系）；但每次列表的开销更大，且列表只需要“能否在此运行审阅操作”这一位信息 |
| 缓存探测结果、监听目录变化 | 列表更快，变化可即时推送；但跨平台目录监听（含网络盘）可靠性差，缓存还需失效策略。当前以“列表时探测 + 拒绝即重探”满足需求，并由后端守卫兜底 |
| 只在前端隐藏入口，后端不守卫 | 改动更小；但过期的列表或旧视图仍会触发操作并得到内部错误，违反不变量 7 |

## 本决策未解决的问题

- **Cloud 目录项目**：是否支持及采用 A、B、C 哪条路，由 ora-space/circus#1444 交 owner 决定；选 A 或 B 时由
  承接的 Cloud ADR 定义能力来源与降级，必要时另立 Desktop 后续 ADR 说明呈现方式。本决策不预置任何 Cloud 协议。
- **普通目录上的变更审阅替代**：普通目录目前没有“Agent 改了什么”的审阅视图。若需要基于文件快照的审阅，涉及
  快照存储与回收，属于后续决策。
- **定时任务 Git 资格的错误类型**：后端对普通目录项目的定时任务保存以通用的 `invalid_request` 拒绝（前端依据
  能力提前说明，不依赖错误码）。是否为此单列带类型的错误码，留待定时任务模块后续决策。

## 核心测试用例

本 ADR 获批后，在 `test-cases/desktop/core/workspace/` 补充核心用例，至少覆盖：探测三态的分类（明确消息且无
`.git` 才是 `notRepository`；损坏、不可读、符号链接上方损坏、超时、清理中都是 `unknown`；嵌套目录是
`repository`）；能力每次列出重新探测、`git init` 后翻转；六个 Git 审阅操作的统一拒绝及先于基准检查；`unknown`
与列表失败保持原有行为；变更面板、diff 链接、worktree 入口与对话框的降级；拒绝后重新探测；定时任务表单拦截与
后端拒绝一致；探测不受语言环境影响。

## 落地顺序

1. **desktop**：ora-space/desktop#812 已实现并合并 D1–D7（契约、后端探测与守卫、gitlancer 分类、前端各入口
   降级与文档）。已有 Git 项目的行为保持不变；从该版本起普通目录项目按本决策降级。
2. **specs / 评审**：owner 评审本 ADR，按反馈纠偏后标为 `approved`，并补充上节核心用例与其证据。
3. **specs / 落地状态**：实现已先于评审合并，核对通过后可在同一变更中直接标为 `implemented`，并按生命周期
   精简正文（删除仅为评审服务的推测与实现细节）。若评审改变了已实现的行为，先在 desktop 修正实现，再转为
   `implemented`。
4. **后续**：ora-space/circus#1444 有结论后，按其选项决定是否另立 ADR；本决策的 Desktop 本机范围不随之改变。
