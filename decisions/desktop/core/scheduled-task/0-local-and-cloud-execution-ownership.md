---
status: approved
---

# 定时任务以持久定义产生幂等触发，执行归属在创建时确定且不支持切换

用户可以在 Desktop 左侧进入“定时任务”，创建周期性的代码质量检查、工作汇报、待办整理等本地任务；Cloud 首期
通过 Web 管理无头 Prompt 单会话任务。`local` 与 `cloud` 分别由各自执行端创建和管理，Desktop 当前不提供可激活的
`cloud` 创建入口，也不合并展示 Cloud 任务。执行归属在创建时确定，之后不支持在两端间切换；需要换执行位置时，复制成新任务并停用
旧任务。`local` 只在 Desktop 进程运行时触发；`cloud` 由服务端拥有未来触发，因此 Desktop 关闭后仍可运行。两种模式
共享任务定义、occurrence 和运行状态的语义，各自持久化，不把浏览器计时器、内存状态或示例模板当作执行事实。

本 ADR 的当前承诺范围为 Desktop local + Cloud Web Prompt（首期），保持 `approved`：不能用本地 MVP 或独立 Cloud 首期的完成替代
本文全部义务的实现与验证。本地 manual occurrence、运行资格与截止闭环及部分跨边界验证仍需补齐。
[Cloud 首期根决策](../../../cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md) 已为 `implemented`，
其 D8 明确首期仅 Web 管理：真实执行与取消链就绪是 Desktop 解锁的必要条件，不能据此推导 Desktop 已开放 Cloud。
Desktop Cloud 管理（创建、展示、历史）与 Cloud 定时触发已发布 Workflow 都属于未来扩展，须另立 ADR 并获批，不属于本文已承诺但未交付的范围。当前范围的实现与直接验证齐全后，本文即可转为 `implemented`，不等待这两项扩展。

## 决策

### D1：Scheduled Task 是持久定义，模板只负责预填

`ScheduledTask` 至少持久化：稳定 ID、名称、启用状态、schedule、执行目标引用及其冻结版本、执行归属、定义 revision、
创建/更新时间和下一次触发投影。执行归属在创建时写入，此后不可变更。代码质量检查、每日工作汇报和待办整理是可编辑的
模板；保存后都是普通 `ScheduledTask`，不会在运行时分叉出三套特殊逻辑。

本地任务及其运行历史只保存在这台设备上：不上传 Cloud、不跨设备同步，也不要求登录即可创建和运行；本地路径与提示词
等内容不离开本机。Cloud 任务及其运行历史的存储位置、可见范围与账号归属随 Cloud 决策定义，本地实现不得预埋上传、
同步或登录依赖。

schedule 首版只提供结构化的受限周期：每 N 小时、每天某个时刻、每周某几天某个时刻；每 N 小时的最短间隔为 1 小时。
不得存储或解释 cron 字符串。local 任务不提供时区选择，schedule 按系统当前时区求值；时区作为持久化字段留给 Cloud
任务随 Cloud 决策定义。任务列表中的 `nextRunAt` 是可重算投影，不是触发事实。

### D2：每个到期点产生稳定的 occurrence，接纳运行是幂等操作

一次计划触发由 `ScheduledOccurrence{taskId, scheduledAt}` 唯一标识，其中 `scheduledAt` 是 UTC 时间点。同一个
occurrence 最多接纳一个 Scheduled Run；进程重启、事件重复投递、系统时间回拨或网络重试必须复用该身份，不能另建
一次运行。执行归属在创建时即固定，不参与身份计算；local 与 cloud 各自在自己的持久库中保证不重复，幂等是各端内部
规则而不是跨端协议。运行的实际副作用仍遵循目标执行器自己的重试与幂等约束；本决策只保证调度层不会把同一到期点重复
接纳为多个运行。

修改 schedule 或执行目标会增加 task revision。名称等纯展示修改不改变 revision。已经接纳的运行冻结原 revision 和
执行目标，不随任务后续编辑漂移；运行记录中的 `revision` 只标识这次运行使用的定义版本，不参与幂等。

### D3：执行目标引用既有真实用例，不由调度器解释 Prompt 或 Workflow

Scheduled Task 保存带版本的 `ExecutionTargetRef`，到期后把 occurrence 交给相应的 application use case。目标可以
在后续决策中扩展为 Agent Task、冻结 Workflow revision 等，但每种目标必须先定义：输入如何冻结、由谁取得凭据、运行
与 session 如何关联、取消及结果如何投影。调度器不得直接调用模型、拼装未版本化 Prompt，或把内存 Workflow runtime
的模拟成功写成真实运行。本地首版的执行目标是随 task revision 冻结的提示词，由新增的 application use case 承载；每次
Scheduled Run 都新建会话，不复用已有会话。

Cloud 首期的执行目标同样只接受随 task revision 冻结的提示词，不绑定 Workflow snapshot。到期时 Cloud 调度器复用既有
Agent `IssueRun` / `Thread` application use case，在目标 Project / Workspace 下拉起一次会话，并沿用既有结果记录与收尾。
每个 Cloud Scheduled Task 绑定一个 issue；每次触发在该 issue 下新建一个 agent IssueRun，保持既有
`issue_runs.issue_id` 与 Thread 路由契约不变。首期不涉及 `executorSlot`；无头单会话的 API 和权限校验在独立 Cloud ADR
定义，调度器不另建 Agent 运行入口。Cloud 触发已发布 Workflow 是后续扩展，届时再定义目标校验与凭据边界。

### D4：local 由 Desktop 进程内 scheduler 拥有，错过与迟到的到期点一律记 missed

`local` 任务由 Desktop 后端的持久 repository 与原生 scheduler 组合驱动。WebView 只读取、编辑和订阅状态，不运行
计时器作为事实源。首版不注册操作系统服务、不设置开机自启，也不在 Desktop 关闭时唤醒机器。

实际触发时间晚于 `scheduledAt` 超过 10 分钟（固定阈值，不提供配置）的到期点一律记为 `missed`，不执行也不补跑。
这同时覆盖 Desktop 未运行、笔记本合盖睡眠后唤醒补触发和系统繁忙导致的迟到；10 分钟内的迟到触发照常执行。
`missed` 是任务级观察，在历史中可见并写明原因；不为离线区间逐条制造运行。用户可显式“立即运行”，它产生独立的
manual occurrence，不冒充原计划时间。未来若增加 catch-up，必须另行决定上限、合并、过期和并发策略。

每次本地定时运行在独立 worktree 中执行目标项目，不直接改动用户正在编辑的目录；worktree 创建复用 Desktop 已有的
初始化端口。运行结束后，无改动的 worktree 直接删除；有改动的 worktree 保留，运行历史显示其路径并提供“打开”和“丢弃”。
非使用中的保留 worktree 上限为 10，超过后按最旧优先清理；正在使用的 worktree 不自动删除，也不计入该上限。
资源归属与崩溃恢复由[原子归属提交决策](20261020-worktree-ownership-atomic-commit.md)细化；
仅声明产物造成的改动按[结果与产物决策](20261021-run-result-annotation-and-artifacts.md)分类。保存为 active 的 local 任务必须通过目标目录是 Git 仓库的
校验；不满足时不得降级为直接在原目录执行。

### D5：cloud 交由服务端拥有的语义冻结，细节另行决策

`cloud` 任务的未来 occurrence 只由 Cloud 持久调度器产生，Desktop 关闭不影响它；这是需要提前冻结的共用语义。创建后不
支持迁移为 `local`；需要更换执行位置时，复制成新任务并停用旧任务。
Cloud 首期的未来 occurrence 由 Cloud 持久调度器产生，并以本 ADR 的 `(taskId + scheduledAt)` 规则在 Cloud 侧幂等
接纳。首期只调用 D3 所述的无头 Prompt 单会话链；资格复查和同意的校验时点以独立 Cloud 根决策 D5 为准：
保存与接纳检查目标和模型，撤销同意原子停用任务，接纳不再单独校验同意。不接入
ora-space/circus#60 / ora-space/circus#67 的 Server Workflow 执行链，也不解析 `executorSlot`。Cloud 的任务登记、存储、取消、状态读取、授权、通知、发起者
归属与费用，以及无头会话 API 细节由独立 `decisions/cloud/` ADR 定义，本决策不预置协议。

Cloud 定时运行的默认超时为 60 分钟；超时取消依赖 Cloud 侧的取消写入者。该写入者就绪前，`cloud` 不得成为可选执行归属。

Cloud 触发已发布 Workflow 必须先另立扩展 ADR 并获批；通用 Workflow server 执行链落地不自动开放定时 Workflow。扩展获批后仍须遵循以下共用语义：每次触发按
任务所有者的个人执行器槽位映射解析执行器；不得复用其他用户的映射。所需槽位缺失时，该次触发明确失败并通知任务所有者。

Cloud 首期已交付 Web 管理与真实无头单会话链路。按独立 Cloud 根决策 D8，Desktop 的 `cloud` 创建、展示与历史仍待
另立扩展 ADR 并获批；当前可以展示模式说明，但必须标为不可用，不能保存为执行归属、不能用本地 timer 代跑，也不能展示模拟成功。

两端只对齐展示用的字段名与状态名，不共享 ID、存储或同步协议；Desktop 任务列表是否合并展示 Cloud 任务、运行历史
能否跨端可见、Cloud 任务的账号与归属，均随 Cloud 决策一并定案。

### D6：时区求值、夏令时和并发规则

schedule 按系统当前时区求值，occurrence 落盘后只以 UTC `scheduledAt` 标识。夏令时跳过的本地时间不补跑；夏令时
回拨造成同一当地时间重复时，只接纳第一个对应的 UTC 时间点，第二个不产生 occurrence。系统时区变化只影响尚未求值的
未来触发，不改写已落盘的 occurrence；local 任务不接受用户单独指定时区。

同一任务的默认并发策略为 `skip_if_running`：到期时若前一 Scheduled Run 仍未终止，记录 `skipped_busy`，不排队
补跑。所有本地定时任务共享一个全局并发上限（首版固定 4，不可配置）；达到上限时该到期点同样跳过并记录可区分的原因，不排队。
后续若提供 queue 或 parallel，必须显式进入任务定义并由目标执行器声明支持，不能只在 UI 增加选择项。

### D7：运行记录统一表达计划结果，删除任务不删除审计事实

Scheduled Run 至少关联 task、revision、occurrence、目标运行/session ID，并使用
`accepted / running / succeeded / failed / cancelled / skipped_busy / skipped_capacity` 状态。`missed` 是任务级观察，
不是伪造的运行，其原因只能是 `app_not_running`（Desktop 未运行期间到期）或 `late_trigger`（进程内触发晚于 10 分钟）。
失败至少区分 `target_unavailable`、`model_unavailable`、`worktree_failed`、`approval_policy_invalid`、`timeout` 与
`execution_error`。UI 的 last run、失败提示和历史列表来自这些持久事实，不从当前时间反推。

每次定时运行开始时发出桌面通知；失败、超时和 `missed` 也分别发出一条桌面通知并进入运行历史。
通知的导航边界按各端能力实现：应用内的 toast 提供点击导航——开始通知进入对应会话，失败、超时和
`missed` 通知定位对应运行历史；系统级通知在窗口不可见时发送，点击只负责把窗口带回前台。这是因为
桌面系统通知插件不提供点击回调（点击动作仅移动端可用），窗口不可见时发出的通知必须在窗口重新可见
后仍以应用内 toast 的形式保留可导航入口，不得因发通知时窗口不可见而丢失导航路径。通知是交互辅助，
不是运行事实源；点击行为的进一步增强（点击系统通知直接导航）属后续决策。首版不自动重试失败或超时
的 occurrence。

禁用任务只阻止未来接纳，不取消已经运行的工作。删除任务采用软删除或等价墓碑，保留既有运行关联。每个任务的
历史保留最近 100 条运行记录，超出时从最旧开始删除；仍带保留 worktree 链接的运行例外，继续保留（数量受
D4 的 worktree 保留上限约束），链接清除后（用户丢弃、或清理路径确认任务已删除）该运行恢复可裁剪。删除
任务后的墓碑和既有记录语义不变。

### D8：local 与 Cloud 分别在保存和每次运行时检查资格

local 不记录账号身份或 Cloud 授权引用。保存 local 任务时检查目标项目目录存在、是 Git 仓库且模型配置可用；每次接纳
时再次检查，失败则该 occurrence 以明确失败结束，不换用其他模型。Cloud 首期保存及每次接纳的资格规则由
[Cloud 根决策 D5](../../../cloud/scheduled-task/0-cloud-owned-scheduled-prompt-runs.md#d5保存与每次接纳都校验资格失败明确到原因不回退不重试)
细化：目标和模型在保存与接纳时复查，同意在保存/启用时校验，撤销即停用；不回退 local，不涉及 Workflow 执行器槽位。Cloud 的创建者、
Workspace、模型凭据、费用与无头会话权限细节由独立 Cloud ADR 定义；定时 Workflow 的资格与执行契约须由另立且获批的扩展 ADR 定义。

local 首版唯一的无人值守工具批准策略是 `full_auto_in_worktree`：只在独立 worktree 中运行，运行时工具调用全自动批准。
创建任务时必须以清晰风险说明取得用户显式同意，并将策略名与同意时间随任务持久化；说明至少覆盖 worktree 不是沙箱、
agent 仍可执行任意命令并读写 worktree 外文件、可使用本机凭据（例如 `git push`），且会消耗模型额度。仅当目标项目或
策略变更时才必须重新同意；只改提示词或计划时间不要求重新同意。每次接纳均校验该策略仍受 runtime 支持且 worktree 可
创建；不满足则以 `approval_policy_invalid` 或 `worktree_failed` 失败。每次运行固定 60 分钟，不可配置；超时后取消并记录
`timeout`。用户撤销同意后，任务自动停用，已有运行事实不改写。以后新增细粒度策略时，既有 `full_auto_in_worktree` 任务
不自动扩大或缩小权限。

## 不变量

1. 任务的执行归属在创建时确定，之后不可切换；每个任务的未来触发始终只由一个执行端产生。
2. 同一 `taskId + scheduledAt` 最多接纳一个 Scheduled Run。
3. WebView timer、模板和 `nextRunAt` 投影都不是触发事实源。
4. local 不承诺 Desktop 关闭或睡眠期间运行；实际触发晚于 `scheduledAt` 超过 10 分钟的到期点记 `missed`，不执行
   也不补跑。
5. cloud 首期只能通过真实调度和无头 Prompt 单会话链成为可选模式；不得以本地 timer 或 Workflow 模拟成功代替。
6. 已接纳运行冻结原 task revision 与执行目标，不因后续编辑漂移。
7. 调度器只调用已定义的 application use case，不自行解释或模拟执行目标。
8. 任务禁用或删除不改写既有运行事实。
9. local 真实执行只能使用已显式同意、仍有效的 `full_auto_in_worktree` 策略；目标项目或策略变更必须重新同意，撤销同意
   后任务自动停用；没有有效策略时不接纳运行。
10. 本地定时运行不直接改动用户正在编辑的目录，默认在独立 worktree 中执行。
11. 本地任务及其运行历史只保存在本机：不上传 Cloud、不跨设备同步，也不要求登录。
12. 保存为 active 的 local 任务必须以 Git 仓库为目标；不允许回退为直接修改原目录。

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| React / WebView `setInterval` 直接触发 | UI 开着时实现快，但刷新、休眠和多窗口会漏触发或重复触发，也没有持久幂等事实 |
| local 与 cloud 同时监听，谁先抢到谁执行 | 能自动容灾，但需要跨端强一致租约；离线 local 无法证明 Cloud 未执行，容易重复产生有费用的 Agent 运行 |
| 允许任务在 local 与 cloud 间迁移切换 | 看似灵活，但两端执行环境、凭据与执行内容完全不同，切换实质是重新绑定执行目标，并引入双 owner 与幂等键漂移风险；复制成新任务、停用旧任务更简单可预期 |
| 为迟到触发提供可配置的补跑窗口 | 表面体贴，但阈值、合并和并发策略都会成为新的决策面；首版选择“10 分钟内照跑、之外记 missed”的最简单可预期行为 |
| Desktop 重启后补跑全部离线 occurrence | 看似完整，但长时间离线会形成无界队列，日报等时效任务也会产生失真结果 |
| Cloud 不可用时自动回退 local | 提高表面成功率，却改变用户选择、授权边界与费用归属，并可能在 Desktop 重开后重复执行 |
| 为三个首发模板分别实现 scheduler | 初期代码直观，但模板行为会分叉，用户自定义任务也无法复用同一运行与审计模型 |

## 本决策未解决的问题

- 三种受限周期已由 domain/contracts 和 Desktop 表单承载；进一步扩展计划规则需后续决策。
- 本地 Workflow revision 目标属于后续扩展。
- 更细的无人值守工具批准策略取值，以及与现有 Session approval 行为的迁移关系；当前 worktree 上限见 D4。
- Cloud 首期的 API、存储、调度、通知和账号/费用归属已由独立 Cloud 根决策细化；跨任务配额、外发通知、
  Desktop 管理等未解决范围以该决策为准，不能继续把已经交付的 Cloud 首期整体列为未实现。
- Desktop Cloud 管理与 Cloud 定时触发已发布 Workflow 不属于当前承诺。未来有需求时分别另立 ADR 并获批，明确管理入口、目标冻结、Start 参数校验及执行器槽位映射等边界；通用 Workflow 执行链交付不是批准这些扩展的依据。
- 本地与 Cloud 任务互不同步；Cloud 首期只管理租户内 Cloud 任务，不读取本地历史，Desktop 不合并展示 Cloud 任务。
  Desktop 后续管理入口与跨执行位置复制任务携带的数据范围仍需后续决策；本地只存本机、不上传、不要求登录的边界不变。
- 操作系统后台服务、开机自启及显式 catch-up。

## 核心测试用例

本 ADR 的核心用例维护在 `test-cases/desktop/core/scheduled-task/`。至少覆盖：
受限周期的三种结构化规则可完整校验，且每 N 小时拒绝小于 1 小时的间隔；同一 occurrence 重放只接纳一次；Desktop
关闭期间 local 不执行且重启记录 missed；实际触发晚于 `scheduledAt`
超过 10 分钟记 `missed` 且不执行、10 分钟内迟到照常执行；manual run 不冒充计划触发；运行中到期产生
`skipped_busy`；达到固定 4 项的全局并发上限时产生 `skipped_capacity`；每次 Scheduled Run 新建会话而不复用已有会话；
本地运行在独立 worktree 中进行且不改动用户当前工作区，
非 Git 目标不能保存为 active；任务编辑冻结旧运行；Desktop Cloud 扩展获批前不能激活；授权失效明确失败且不自动回退；禁用/删除保留历史；本地任务及其运行
历史不上传、不跨设备同步且不要求登录；失败、超时与 `missed` 分别通知；失败与超时均不自动重试；每个任务的历史保留最近
100 条运行记录并从最旧删除，仍带保留 worktree 链接的运行例外（受 D4 上限约束，链接清除后恢复可裁剪）。
还需覆盖：没有 `full_auto_in_worktree` 显式同意时不接纳运行；仅改提示词或计划时间不要求重新同意、目标项目或策略变更
要求重新同意；策略撤销后任务自动停用且既有运行事实不被改写；固定 60 分钟超时取消运行并记录 `timeout`；无改动 worktree
在运行后删除、有改动 worktree 保留且可从历史打开或丢弃；Cloud 首期 Prompt 复用绑定 issue 下的 IssueRun / Thread 并校验
Project、模型和无人值守授权，不解析 Workflow 或执行器槽位；Cloud 取消写入者未就绪时不可激活。定时 Workflow 扩展 ADR 获批后，才按 D5 的共用语义补充槽位缺失时明确失败并通知任务所有者的核心用例；该扩展不构成当前落地验收义务。

## 落地顺序

1. **specs**：本根决策获批，并据批准内容补核心测试用例。
2. **desktop / local definition**：新增 domain、repository 与应用用例，契约生成后接入左侧入口、列表和编辑器；
   Cloud 选项保持可见但不可激活。
3. **desktop / local scheduler**：用后端 scheduler 重建 active local jobs，落实 occurrence 幂等、迟到/离线
   missed、独立 worktree 执行、全局并发上限、`skip_if_running` 和真实 ExecutionTarget adapter；没有真实
   adapter 时不得把模拟运行作为 MVP 完成。
4. **cloud / Web Prompt**：独立 Cloud 根决策已定义并交付无头 Prompt 单会话的 API、权限、持久化与 Web 管理。
   该交付不自动开放 Desktop Cloud 管理或定时 Workflow；扩展 ADR 获批前 Desktop cloud 继续不可保存，不由本地 timer 代跑。
5. **当前范围验收与状态落地**：补齐 local manual occurrence、每次资格复查、完整截止与取消闭环及跨边界直接验证。
   Desktop local + Cloud Web Prompt 的当前承诺满足生命周期条件后，同步最终实现和核心用例，将根 ADR 转为 implemented。
   Desktop Cloud 管理与定时 Workflow 不属于本轮落地步骤；未来须另立 ADR 并获批，本篇状态不等待两项扩展。
