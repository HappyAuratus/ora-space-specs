---
status: implemented
---

# Confirm 只按所选发布版本自身的 schema 校验

`@` Workflow 的 Confirm 不再把活稿和所选版本放在一起比较：服务端只按用户选中的那份已发布版本的
`FormDescriptor` 校验提交值，`409 workflow_version_fields_changed` 不再存在。选择旧版本就是按那份快照
执行——它的字段与当前草稿是否一致都不影响准入，只有缺该版本必填字段（`400 required_field_missing`）
或提交它未声明的字段（`400 invalid_field_value`）才被拒绝。本决策修改
[Issue 协作根决策](0-typed-targets-and-explicit-execution-intent.md) D9 的校验依据；Confirm 的其余承诺
（唯一执行边界、单事务、版本冻结、幂等与比较交换）不变。

本 ADR 已 `implemented`：Cloud 侧实现见 [cloud#114](https://github.com/ora-space/cloud/pull/114)
（合并提交 `0db42305`），核心用例见
[Issue 协作交互核心用例](../../../test-cases/cloud/issue-collaboration/collaboration-interactions.md)
的「Confirm Creates Exactly One Run From Revalidated Values」。它涉及 Cloud 的
`POST /issues/{iid}/interactions/{ixid}/confirm`、按所选版本读取表单的
`GET /workflows/{wfid}/snapshots/{sid}/form` 以及前端表单；不改变 `issue_interactions` /
`issue_runs` / `workflow_runs` 的表结构，不需要迁移，也不回填历史交互或运行。

## 继承与修改

继承 [Issue 协作根决策：`@` 只选择类型化目标，执行只来自显式意图](0-typed-targets-and-explicit-execution-intent.md)。

| 约定 | 本决策 |
| --- | --- |
| Confirm 是 Workflow 唯一的执行边界；选择、填写、Assist 都不执行 | 继承，不变 |
| Confirm 重新解析目标，在同一事务里建运行、冻结版本、比较交换认领交互 | 继承，不变 |
| 活稿与所选版本对提交有影响的属性不一致时 `409 workflow_version_fields_changed` 并列出字段 | 修改：删除该门禁与错误码，不再比较两份描述 |
| 按所选版本的描述完整校验（`400 required_field_missing` / `invalid_field_value`） | 继承，不变；它成为唯一的字段校验 |
| 根决策不变量 5：Confirm 总是用服务端当前解析的描述重新校验 | 继承；「当前解析」指所选发布版本的描述 |
| 重复确认 `409 interaction_already_confirmed`；`Idempotency-Key` 语义 | 继承，不变 |

## D1：所选版本自身的 schema 是唯一校验依据

一次 Confirm 的字段准入只由两样东西决定：**所选发布版本的描述**与**提交值**。

- 所选版本决定哪些字段存在、各自的控件类型、必填性与选项集合。通过平台版本选择器提交
  `snapshotId` 时，该版本就是所选版本；没有提交时，服务端按活稿的版本规则从提交值解析，当前解析到
  最新已发布版本。
- 提交包含该版本声明的字段且满足约束即通过；缺少该版本必填的字段是 `400 required_field_missing`，
  提交该版本未声明的字段是 `400 invalid_field_value`，参数指向该版本的字段名。
- 活稿在此之后的编辑、再次发布都不改变任何一次 Confirm 的准入：只要所选版本不变，同一份提交的结果
  也不变。

**为什么不再比较活稿。** 被执行的是所选快照，活稿不是可执行文档；用一个不会运行的文档去否决一次
合法运行，把发布状态变成了 Confirm 的隐含前置条件。用户也无法修复这类拒绝：草稿里的字段通常要等
下一次发布才会进入可执行版本，原样重试不会改变结果，而错误指向的字段又不在用户按所选版本填写的
表单里。版本选择本身就是「按这一版执行」的承诺。

**客户端与校验同源。** 用户没有手动切换版本时，`@` 表单初次打开也按默认选中的发布版本（最新已发布
版本）读取并渲染，而不是渲染活稿；切换版本时同样按新版本读取。因此显示的字段与 Confirm 校验的
schema 始终来自同一个版本：活稿里未发布的字段不出现在默认视图，用户也不会被看不到的字段拒绝。
渲染仍不是准入判断，读取表单本身不创建运行。

这条同源承诺的前提是平台能指定版本，而 `snapshotId` 由**平台持有**（见
[snapshotId 是保留字段](20261021-snapshot-id-is-a-reserved-field.md)）：`@` 表单始终投影平台的版本选择器，
活稿里同名声明的作者变量不进入表单、也不按作者字段校验；Confirm 的版本游标一律按 `snapshotId` 读，不看
活稿是否还声明着它。**保留键由平台持有，作者声明不生效**——新的发布由门禁以 `reserved_field_name` 拒绝；
此前发布的、自己声明了该键的存量版本在选择器里不可选，在 Confirm 时返回
`409 workflow_version_reserved_field`。所以显示与执行同源不再需要任何例外。

用启动字段撤下 `snapshotId`（`launchFields` 里的 `enabled: false`）不属于作者声明该键：活稿没有声明它，
Confirm 的版本游标仍会读提交值。界面因此没有选择器，不提交该键时按最新已发布版本解析；如果解析到的版本
自己声明了该键，确认返回 `409 workflow_version_reserved_field` 而不执行。如果调用方仍然提交了该键（例如直接
调 API），它和平台选择器提交的值一样按版本游标读取。

**边界与反例。**

| 情形 | 结果 |
| --- | --- |
| 选中的旧版本没有活稿后来新增的必填字段，提交不含它 | 通过：该版本不要求这个字段 |
| 选中的旧版本声明了某个字段，活稿后来把它删了，提交不含它 | `400 required_field_missing`：该版本仍然要求它 |
| 提交只存在于活稿、该版本未声明的字段 | `400 invalid_field_value`：字段不属于这次执行 |
| 两个字段与活稿都不同，但都符合所选版本 | 通过，运行记录该版本的 `snapshot_id` |

## D2：错误契约收窄，不再有版本分歧专用错误

`409 workflow_version_fields_changed` 从协作契约中移除，字段拒绝只剩两种 `400`。该错误的唯一来源是
已删除的比较；保留一个不可达的错误码会让客户端继续维护无效分支，也让调用方误以为「重新发布后再确认」
是这类请求的修复路径。

## 不变量

1. 一次 Confirm 的准入只取决于所选发布版本与提交值，不取决于活稿当时的字段。
2. 被拒绝的 Confirm 不创建运行、不占用交互；字段拒绝只有 `400 required_field_missing` 与
   `400 invalid_field_value`。
3. 运行的 `snapshot_id` 与校验所用的版本是同一个。
4. 同一交互重复确认仍是 `409 interaction_already_confirmed`；幂等键语义不变。
5. 有平台版本选择器时，客户端渲染的表单与 Confirm 校验的版本是同一个，包括未手动切换时的默认视图。
   保留键 `snapshotId` 由平台持有，作者声明不生效，所以这条同源承诺没有例外，见 D1；用启动字段撤下
   选择器时本条不适用，见 D1。

## 为什么不是这些替代方案

| 替代方案 | 否决理由 |
| --- | --- |
| 保留活稿比较，只在字段有交集时放宽 | 仍让不执行的文档参与准入；规则更难解释，用户也无法从错误里得到可执行的修复动作 |
| 活稿与所选版本不一致时拒绝，并提示「请重新发布」 | 把发布变成确认的前置条件，扩大了执行边界，旧版本的合法执行被无关的草稿挡住 |
| 只按活稿校验 | 运行的图与校验依据分离，会放行与执行文档不符的输入，并让旧版本的执行不可预期 |
| 版本选择只当提示，总是执行最新版本 | 与「选中旧版本就按旧版本执行」的用户预期冲突，也丢掉了版本冻结的意义 |
| 默认视图渲染活稿，只在用户切换版本后才读版本表单 | 默认视图显示的字段与校验的 schema 不同源，活稿未发布的字段会让用户被这个版本没见过的字段拒绝，与本决策不变量 5 冲突；代价只是初次打开多一次版本表单读取 |

## 本决策未解决的问题

`fixtures=false` 部署上的真实 Node 联合验收不在本决策范围内，由 circus#71 承接，与 circus#77 的验收
范围协调。它不改变本决策的校验规则与客户端渲染承诺。

## 落地

- 服务端删除活稿比较与 `409`、只按所选版本校验，与客户端默认视图（含未手动切换时的默认版本）都按
  所选发布版本读取表单，已随 [cloud#114](https://github.com/ora-space/cloud/pull/114)（合并提交
  `0db42305`）落地；字段拒绝只剩带字段参数的两种 `400`。
- 核心用例与证据已同步为 `Covered`，指向 main 上的
  `TestWorkflowConfirmValidatesAgainstTheChosenVersion`、
  `TestWorkflowConfirmConcurrentInvocationCreatesExactlyOnePair` 与前端
  `opens on the default published version by reading that version, so rendering and validation match`。
- `fixtures=false` 部署上的真实 Node 联合验收由 circus#71 承接，验收记录写在该 issue；它不改变本决策的
  校验规则与客户端渲染承诺。
- D1 里「保留键由平台持有，作者声明不生效」的改写已随 [cloud#123](https://github.com/ora-space/cloud/pull/123)
  （合并提交 `fc12dbe4`）落地：发布门禁、版本选择器过滤与 Confirm 的 `409 workflow_version_reserved_field`；
  本决策原先「作者声明 `snapshotId` 时该键归作者」的例外不再存在。
- 版本冻结、单事务与幂等语义全程不变；本决策不改变表结构，也不需要迁移或回填历史交互与运行。
