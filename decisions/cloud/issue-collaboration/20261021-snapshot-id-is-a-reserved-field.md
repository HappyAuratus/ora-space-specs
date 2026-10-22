---
status: implemented
---

# `snapshotId` 是保留字段：发布门禁拒绝作者声明，存量版本不可选

`@` 表单的版本选择器只在 `snapshotId` 这个键属于平台时成立：它由平台注入，值是按所选版本执行。本决策按
circus#405 的 owner 结论（方案 D）落地，并采用 [circus#417](https://github.com/ora-space/circus/issues/417)
复审选定的 A′（`snapshotId` 不适用「作者声明优先」，一律归平台）；它消除作者占用这个键的可能，而不是在
占用已经发生后补救：

- **发布门禁**：发布（含 workflow 资产导入的冻结路径）时，图里任何 `snapshotId` 变量声明都被拒绝，问题点名
  节点或字段（`reserved_field_name`），不产生快照；活稿保存不拦。
- **键属于平台**：只要有一个可选的已发布版本，`@` 表单就把 `snapshotId` 投影成平台的版本选择器；仍然声明着
  该键的活稿不把这个变量渲染进表单，Confirm 也始终把提交里的这个键当版本游标读，不看活稿是否声明同名变量。
- **存量版本**：已经发布、自己声明了该键的版本不进入平台的版本选择器，选择器同时给出「占用了保留字段，
  请重新发布」的提示；Confirm 解析到这类版本时按可定位的原因拒绝，不论提交里有没有版本游标。
- **`launchFields` 撤下 `snapshotId` 不是作者声明**：行为不变，界面没有选择器，不提交该键时按最新已发布
  版本解析（该版本自己声明了保留键时按 D3 拒绝），提交了仍按版本游标读取。

本决策修改 [Confirm 只按所选发布版本自身的 schema 校验](20261020-confirm-validates-against-the-chosen-version.md)
的 D1 限定与不变量 5：「作者在图里声明 `snapshotId` 时不承诺显示与执行同源」不再是一条需要读者记住的例外：
保留键由平台持有，作者声明不生效；发布门禁让新版本不能声明它，存量占用版本按 D2 不可选、按 D3 拒绝。存量
版本的不可选与拒绝对该决策承诺的同源性没有影响。

本决策已 `implemented`：它涉及 Cloud 的发布（`POST /workflows/{wfid}/publish`）与 workflow 资产导入两个冻结
入口、`GET /collaboration/forms/{formRef}?issueId=` 与 `GET /workflows/{wfid}/snapshots/{sid}/form` 的版本
选择器投影，以及 `POST /issues/{iid}/interactions/{ixid}/confirm` 的版本游标解析，实现见
[cloud#123](https://github.com/ora-space/cloud/pull/123)（合并提交
[`fc12dbe4`](https://github.com/ora-space/cloud/commit/fc12dbe4722b291a806ea07a1217655cdd7789b5)）。核心用例与
证据已同步为 `Covered`，见
[发布与启动准入核心用例](../../../test-cases/cloud/workflow/publish-and-start-admission.md) 与
[Issue 协作交互核心用例](../../../test-cases/cloud/issue-collaboration/collaboration-interactions.md)。本决策不改变
任何表结构，不迁移、不改写已发布快照，也不改写历史 interaction 的 `input`：已经冻结的文档是不可变数据，处理
放在读取侧。

## 继承与修改

继承 [Issue 协作根决策](0-typed-targets-and-explicit-execution-intent.md)、[工作流执行根决策](../workflow/0-cloud-side-scheduler-owns-workflow-execution.md)
与 [Confirm 只按所选发布版本自身的 schema 校验](20261020-confirm-validates-against-the-chosen-version.md)。

| 约定 | 本决策 |
| --- | --- |
| 协作根决策 D8 规则 1：Start 节点自己声明平台 key 时，该 key 的控件、标签与必填性归作者 | 修改（仅对 `snapshotId`）：这条规则不适用于保留键。只要有一个可选的已发布版本，`@` 表单就投影平台的版本选择器，活稿里的同名声明不渲染、不按作者字段校验；Confirm 的版本游标一律读 `snapshotId`。作者声明本身由发布门禁拒绝，存量占用版本按 D2 不可选、按 D3 拒绝 |
| 协作根决策 D8 规则 2：没有可选项时不注入版本选择器 | 修改：全部已发布版本都不可选时也不注入，与「没有已发布版本」同一种合法形态 |
| 协作根决策 D8 规则 3：版本选择器列出全部已发布版本 | 修改：只列出**未占用保留键**的已发布版本，其余不可选并提示；选项上限只计可选版本 |
| 协作根决策 D8 规则 5：`launchFields` 按 key 覆盖，对作者自己声明的 key 无效 | 继承，不变；用 `launchFields` 撤下 `snapshotId` 不是作者声明，不落入门禁 |
| 协作根决策 D8 规则 6：`snapshotId` 等是平台保留 key | 继承并加强：`snapshotId` 不再只是「作者声明优先」的 key，而是作者不能声明的 key |
| 协作根决策 D8 规则 7：旧名 `version` 的读取、折叠与历史数据兼容 | 继承；`version` 不是保留字段，理由见 D1。折叠对 `snapshotId` 的 guard 失效：那条 guard 让「活稿声明了该键」成为不折叠的理由，A′ 下这个前提不再存在，游标一律读 `snapshotId`；作者自己声明 `version` 时仍跳过折叠，这一半不变 |
| 协作根决策 D9：Confirm 是唯一执行边界，重新解析目标、校验、建运行、认领交互在同一事务 | 继承，不变；本决策在创建运行前增加一种可定位的拒绝 |
| 工作流执行根决策 D2：发布门禁拒绝「能从文档本身判定」的配置错误并点名，保存草稿不拦 | 扩展：保留键声明是新增的一类文档级拒绝；D2 列出的图结构、节点类型、节点配置与 canonical PluginId 不是穷举 |
| Confirm 决策 D1：所选版本自身的 schema 是唯一校验依据；客户端与校验同源 | 继承，不变；同源承诺不再需要「作者声明 `snapshotId`」的例外，见 D2、D3 |
| Confirm 决策不变量 5：有平台版本选择器时，表单与校验版本同源 | 修改：保留键由平台持有，作者声明不生效，所以同源不再需要「作者没有声明 `snapshotId`」这个前提 |

## D1：发布门禁拒绝作者声明的 `snapshotId`

发布一条图，以及 workflow 资产导入冻结一份文档时，只要图声明了会进入 `@` 表单的 `snapshotId`，这次
发布/导入就被拒绝，且不产生任何快照。声明的位置与问题参数：

| 声明位置 | 判定 | 问题参数 |
| --- | --- | --- |
| Start 节点的 `inputVariables[]` 里 `name` 为 `snapshotId` | 拒绝 | `code: reserved_field_name`、`nodeId`、`field: snapshotId` |
| `globalVariables[]` 里 `name` 为 `snapshotId` | 拒绝 | `code: reserved_field_name`、`name: snapshotId` |

发布沿用现有的 `400 workflow_graph_invalid` 信封，问题列表里带这条 `reserved_field_name`；资产导入沿用现有的
`document_not_executable_graph` 失败码，问题进入失败详情。两个入口共用同一份图校验，所以同一条声明在两个入口
得到同一个问题码与同一组参数。活稿的保存、更新、恢复不受影响：与其他发布门禁一致，保存允许未完成的编排，
只有冻结成可执行文档时才判定。

**判定顺序与可观察的错误码变化。** 保留键判定先于全局变量的限定名判定：名字恰为 `snapshotId` 的全局会得到
`reserved_field_name`（`name: snapshotId`），不再得到 `global_name_invalid`。两条规则都能拒绝它，但只有前者的
问题码说出了真正的原因——名字是保留字段，而不是限定名写错；门禁落地前它只有一个错误码，落地后订阅
`global_name_invalid` 的调用方会看到这个具体值的码变了。这是有意的契约变化，核心用例按「保留字段优先」登记。
名字只是看起来像（例如前后带空格）的全局不触发保留键判定：`@` 表单按变量自己声明的名字取键，带空格的变体
不会与平台的键重名，仍按限定名规则拒绝。

**`snapshotId` 不适用「作者声明优先」。** 协作根决策 D8 规则 1 让 Start 节点自己声明的平台 key 归作者：
控件、标签与必填性都按作者写的来。保留键是例外：只要有一个可选的已发布版本，`@` 表单就把 `snapshotId` 投影成平台的版本选择器，
活稿里同名的作者变量不渲染、也不作为作者字段校验；Confirm 的版本游标一律读提交里的 `snapshotId`，不看活稿
是否声明它。于是同一个提交值只有一种含义——用户选的版本——界面显示的版本与实际执行的版本同源，不再依赖
「作者没有占用这个键」这个前提。

作者声明到不了执行：下一次发布会被 D1 拒绝，此前发布的、自己声明了该键的版本按 D2 不可选、按 D3 拒绝。
如果反过来让活稿的声明接管这个键，作者会得到一个显示着自己字段、执行却由平台游标决定的表单；如果在 `@` 表单
或 Confirm 上拒绝仍然声明该键的活稿，用户会被一个自己看不到原因的作者配置挡住。两者都只是把边界挪到用户面前，
本决策的选择是让边界不存在。

**为什么仍然需要发布门禁。** 只把键判给平台、不在发布时拒绝，也能消除同一个值的两种含义，但作者的变量会从
表单里无声消失：必填的变量永远填不上，非必填的永远拿不到值，而作者在编辑器和草稿里看到的仍是它。发布门禁
把这件事变成一次带节点与字段名的拒绝，作者能直接改名；它同时保证新的已发布版本不再产生 D2、D3 要处理的
存量形态。

**`launchFields` 不落入门禁。** 图信封里的 `launchFields: [{key: "snapshotId", enabled: false}]` 是作者要求平台
**撤下**这个字段，不是把名字声明成自己的变量：`snapshotId` 不会进入表单，也不会与平台游标争夺同一个值。这种
配置保持现状（协作根决策 D8 规则 5）：界面没有选择器，不提交该键时按最新已发布版本解析——如果解析到的版本
自己声明了该键，Confirm 按 D3 拒绝——提交了仍按版本游标读取。

**`version` 不是保留字段。** 旧名 `version` 只用于读取改名前的提交与已发布快照里的声明（协作根决策 D8 规则
7）；平台不再渲染、也不再写入这个名字。作者自己声明 `version` 时，平台照常注入键为 `snapshotId` 的选择器，
版本游标从 `snapshotId` 读取，`version` 的值归作者；折叠旧 key 的规则也跳过作者自己的声明。两条通道不共享
同一个值，不存在 D1 要消除的歧义，所以不需要把 `version` 一并保留，也不会让存量图因为一个无关的名字被拒。

**为什么不能靠 `globalVariables` 的限定名规则顺带解决。** 全局变量名必须带 `.`，所以一个恰好叫 `snapshotId`
的全局本来就会被 `global_name_invalid` 拒绝；但这条规则的目的是限定名语法，不是保护平台 key。门禁按「这个
名字是不是保留字段」独立判定，问题码也直说保留字段，不依赖另一条规则恰好覆盖它；将来出现新的声明形态时，
判定仍然成立。

## D2：存量版本不进入版本选择器

已发布快照不可变。在本决策之前发布的版本里，可能有自己声明了 `snapshotId` 的图；它们不能改写，也不能删除
（快照是运行执行证据的载体），处理放在读取侧。选择器是否注入、选项里有什么，只由已发布版本自己的图、`launchFields` 声明与有没有可选的已发布版本
决定，不因活稿是否声明同名变量而变：

- **选项**：平台的版本选择器只列出自身冻结图没有声明 `snapshotId` 的已发布版本。选项上限按可选版本计算：
  被排除的版本不占 100 个名额，选择器仍按最新的可选版本填满上限。
- **默认值**：仍是最新的**可选**版本；没有可选版本时不注入选择器，与「没有已发布版本」的合法形态一致
  （协作根决策 D8 规则 2）。
- **提示**：只要有一个已发布版本因为这个原因被排除，选择器的 `description` 就写明有版本占用了保留字段
  `snapshotId`、已不可选、需要重新发布。提示随选择器字段下发，前端按现有字段描述渲染；作者重新发布一个不含该
  声明的版本后，新版本可选，存量版本仍不可选。
- **投影范围**：活稿 `@` 表单与带 `issueId` 的 `GET /workflows/{wfid}/snapshots/{sid}/form` 共用同一个选择器
  投影与保留键处理，规则一致；不带 `issueId` 时读到的是工作流自己的 schema，平台字段本来就不注入，作者声明按
  原样投影，本决策不改这部分。

**为什么不给选项加 `disabled`。** 选项形状是 cloud `internal/core/form_descriptor.go` 里的
`FormOption {value, label}`，没有禁用语义，前端只能把选项渲染成可选；新增禁用字段会改动所有实现方共用的描述契约，
代价远大于收益。从选项集合里移除是描述契约内唯一
能表达「不可选」的形态，解释由选择器的 `description` 给出。若所有已发布版本都不可选，界面没有选择器，原因由
D3 的 Confirm 拒绝给出，而不是把不可选版本伪装成可选后静默执行。

**为什么不改写或删除存量快照。** 快照是运行执行过的冻结文档，运行记录引用它、历史 interaction 也按它校验。
改写会让已经发生的执行与它声称执行的文档不一致；删除会让引用了它的运行失去执行身份。存量版本的唯一修复
路径是：作者从活稿去掉声明后重新发布，得到一个新的、可选的版本。

## D3：Confirm 不执行声明了保留键的版本

Confirm 解析出要执行的版本后（不论提交里有没有版本游标、不论活稿是否还声明着 `snapshotId`），如果它解析到
的版本自己的冻结图声明了该键，Confirm 返回可定位的 `409 workflow_version_reserved_field`，参数带
`snapshotId`（被解析到的版本 id）与 `field: snapshotId`（冲突的键名），不创建运行、不占用 interaction，也不
改选另一个版本。

拒绝成立，是因为这个键在任何活稿里都归平台：D1 之后作者无法发布声明它的版本，D2 的投影范围也不让活稿里残
留的声明进入 `@` 表单或参与游标解析。于是声明了该键的已发布版本只剩一种可能的含义，而这正是 D1 要消除的
歧义：

- 平台把提交值当版本游标，是 `snapshotId` 在这个键上唯一的含义（D1、协作根决策 D9 的游标规则）。执行一个
  自己声明了该键的版本，会让同一个值既是平台选择的版本、又是该版本自己声明的变量，这不是「按用户选中的版本
  执行」的合法含义。
- 「静默改跑最新版本」更不允许，那正是 circus#405 要消除的结果：用户选择的版本被无声替换。
- 没有提交版本游标时同样适用：此时版本按既有规则解析到最新已发布版本，若它声明了保留键，Confirm 拒绝而不是
  静默执行它（所有已发布版本都被排除、界面没有选择器时，用户能得到的定位原因就是这条拒绝）。

**不改变的其他入口。** 直接运行入口（`POST /workflows/{wfid}/runs`）的 `snapshotId` 是请求体的版本选择，
运行输入在独立的 `input` 对象里，不与表单 key 共享含义，本决策不改它。Assist 不执行、也不创建运行；它按提交里的版本游标选择要投影的版本，
读的是同一份表单投影，保留键的处理随之生效。

## 不变量

1. 本决策之后发布或导入冻结的每一份快照，图里都没有会进入 `@` 表单的 `snapshotId` 声明。
2. 发布与资产导入对同一条保留键声明给出同一个问题码（`reserved_field_name`）与同一个定位，且都不产生快照；
   名字恰为 `snapshotId` 的全局变量返回 `reserved_field_name`，不再返回 `global_name_invalid`。
3. 保存草稿从不受保留键门禁限制。
4. 已发布版本如果声明了 `snapshotId`，平台版本选择器不会把它当成可选版本；存量快照不被改写。
5. Confirm 不执行自己声明了 `snapshotId` 的已发布版本，不论提交里有没有版本游标、活稿是否还声明着该键；
   被拒绝时不产生运行，也不改选版本。
6. `launchFields` 撤下 `snapshotId` 的配置行为与决策前相同：没有选择器，不提交该键时按最新已发布版本解析，
   解析到的版本自己声明了保留键时按 D3 拒绝；提交了仍按版本游标读取。

## 为什么不是这些替代方案

| 替代方案 | 否决理由 |
| --- | --- |
| 只在编辑器或表单里提示作者不要用这个变量名 | 提示不改变行为：用户仍可能按旧版本的字段确认、执行的却是另一份文档，与 circus#405 的 A、B 选项同类 |
| 让活稿里仍然声明该键的作者变量接管这个键（协作根决策 D8 规则 1 原样适用） | circus#405 的 A 选项：用户看到作者自己的字段，执行的版本却由平台游标决定，同一个提交值有两种含义；D2 的不可选与 D3 的拒绝正是为了不出现这种形态 |
| 在 `@` 表单或 Confirm 上拒绝仍然声明该键的活稿 | circus#405 的 B 选项：把作者配置的错误变成用户看不到原因的阻塞，而键的归属不需要用户理解就能确定；发布门禁已让作者在下次发布时得到点名拒绝 |
| 给版本选项加 `disabled`，保留在列表里 | 描述契约没有禁用语义，前端只能画成可选；扩展契约影响所有实现方，收益只有提示更贴近选项 |
| 改写或删除声明了该键的存量快照 | 快照是执行证据，运行与历史 interaction 引用它；改写会让记录与事实不符，删除会让运行失去身份 |
| 把 `version` 也列为保留字段 | 平台不再渲染或写入 `version`，作者声明它不会与游标共享值；一并拒绝只会让无关的存量图被卡住 |
| 上传/导入时只在 CI 或离线工具里检查 | 资产导入会在服务端自动冻结快照，检查必须在那条路径上；离线工具覆盖不到 API 发布的图 |
| 让 Confirm 遇到这类游标时静默执行最新版本 | 用户选择的版本被无声替换，正是 circus#405 明确不允许的结果 |

## 落地

- 发布与资产导入的保留键判定、`@` 表单与带 `issueId` 的版本表单按 D2 注入选择器、版本选择器的过滤与提示、
  Confirm 的 `409 workflow_version_reserved_field`，已随
  [cloud#123](https://github.com/ora-space/cloud/pull/123)（合并提交
  [`fc12dbe4`](https://github.com/ora-space/cloud/commit/fc12dbe4722b291a806ea07a1217655cdd7789b5)）落地，并通过
  `task check`、`task frontend:check` 与 `task test:race`。两处判定顺序是有意的：保留键判定先于全局变量的
  限定名判定；Confirm 的保留键拒绝先于执行器槽位门禁——解析到的版本根本不能执行，让发起人先去配置槽位只会
  指错地方。
- [Confirm 只按所选发布版本自身的 schema 校验](20261020-confirm-validates-against-the-chosen-version.md) 的 D1
  限定与不变量 5 已按本决策同步为「保留键由平台持有，作者声明不生效」，不存在此前仍按原例外运行的窗口。
- 核心用例与证据已同步为 `Covered`，指向 main 上的
  `TestWorkflowReservedFieldIsRefusedOnPublishAndImport`、
  `TestWorkflowReservedFieldWithholdsStockVersionsAndRefusesThem`、
  `TestWorkflowReservedFieldStaysThePlatformsWhenTheDraftDeclaresIt`、
  `TestWorkflowReservedFieldSelectorCoversEveryDeclarationShape`、
  `TestWorkflowReservedFieldRefusalOutranksTheExecutorSlotGate`、
  `TestWorkflowPublishRefusesReservedFieldDeclarations`、
  `TestAuthorDeclaredSnapshotIDDoesNotTakeTheKeyBack`、
  `TestWorkflowVersionCursorIsThePlatformsForTheReservedKey`、
  `TestLaunchKeysCollideLikeTheOlderOnes` 与 `TestVersionFieldExplainsWithheldVersions`。

不改变的行为：活稿保存、直接运行入口、`launchFields` 撤下字段、已有快照的内容与历史 interaction 的 `input`。
Assist 不执行、也不创建运行；它按提交里的版本游标选择要投影的版本，读的是同一份表单投影，保留键的处理随 D1 变化。
从本决策起改变的行为：新的发布与资产导入拒绝保留键声明（名字恰为 `snapshotId` 的全局从
`global_name_invalid` 变为 `reserved_field_name`），版本选择器不再列出存量占用版本，Confirm 对解析到它们的
确认（不论提交里有没有版本游标）返回新的 `409`。
