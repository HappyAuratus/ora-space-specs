# workflow 插件资产导入：逐文档导入、跨版本幂等与出网边界

依据[workflow 插件资产决策](../../../decisions/cloud/plugin-marketplace/20261016-workflow-plugin-assets.md)
（`approved`）与它的[workflow 接缝决策](../../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)。

资产导入尚未实现：落地阶段 1 是 ora-space/circus#73，Cloud 仓库里还没有导入步骤、来源记录表与
逐文档结果。因此本文件今天没有 `Covered` 行；能引用的直接证据只有 Cloud 既有的目录定位符校验与
出网客户端原语——它们证明被复用的规则本身成立，不证明导入步骤已经接上它们（那部分记 `Missing`）。

## A Document Path Is Imported At Most Once Across Package Versions

- 风险：包升级或重装覆盖用户编辑过的副本；反过来，一次失败（例如名称冲突）被当成永久结论，用户
  修好冲突后同一路径再也导不进来；或用户删掉的副本在重装时被自动复活。
- 前置状态：来源记录里同一 `(source_namespace, identifier, document_path)` 可能已有 `imported` 或
  `failed` 结果；用户副本可能已被编辑、改名、归档或删除；包可能已升级到新版本。
- 触发：`v1` 安装成功后升级到 `v2`；重复安装同一版本；对上次 `failed` 的路径再次触发；删除或归档
  副本后重装插件；同一文档在新版本里内容变化。
- 必须成立：跳过判定只看 `(canonical Plugin ID, document_path)` 下是否已有 `imported`，与
  `package_version` / `package_sha256` 无关；`failed` 不占用判定键，下次触发重新尝试并刷新结果与
  时间；升级只导入从未成功导入过的路径；新版本删掉的路径不删副本；删除或归档不改写来源记录，重装
  不复活；结果按判定键独立持久化、可单独读取。
- 禁止结果：按包版本或 sha256 决定跳过、同一路径成功导入两次、失败一次后永久放弃、重装时把用户
  删除或归档的副本再导入一份、升级覆盖已编辑的副本。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 判定键跨包版本，只有 `imported` 占用 | 真实 HTTP + PostgreSQL：`v1` 导入后升级 `v2`，同一路径不再产生新副本与快照 | Missing |
| `failed` 不占用判定键，下次触发重试并刷新记录 | 真实 PostgreSQL：名称冲突修复后再次触发，同一路径从 `failed` 转为 `imported`，记录时间被刷新 | Missing |
| 删除/归档副本后重装不复活 | 真实 PostgreSQL：软删除或归档副本后重装，`skip_key` 仍被占用，副本数不变 | Missing |
| 结果独立于安装结果持久化 | 真实 PostgreSQL：无 live workspace 的租户也有按 `skip_key` 可读的逐文档结果 | Missing |

- 决策依据：D1（来源记录与 `skip_key`）、D4（判定键跨版本且只认成功、升级只增不改、不复活用户
  处置过的副本），不变量 1、5；[circus#68](https://github.com/ora-space/circus/issues/68) 结论 6。
- 说明：本用例只管「跳过与重试」的判定，不重复「导入产物是已发布快照」那部分义务，后者记在
  [资产导入与已发布快照用例](../workflow/asset-import-and-published-snapshots.md)。
- 仍缺：无直接证据。导入落库与触发路径都不存在，四条义务全部 `Missing`。

## One Bad Document Fails Alone With A Bounded Reason

- 风险：一份坏 JSON 或非法图让整包文档都不导入，用户丢掉其余可用工作流；或失败原因无界，用户
  无法分辨是 JSON、缺名字、图不可执行还是名称冲突。
- 前置状态：包内有多份 `assets/workflows/*.json`，其中至少一份非法；租户里可能已有同名 workflow；
  导入步骤在数据库事务外解析、事务内落库。
- 触发：导入含 `document_invalid_json`、`document_without_name`、`document_not_executable_graph`
  或 `name_conflict` 的文档；同包其他文档合法。
- 必须成立：每份文档一个结果，坏文档只让自己失败，同包其他文档照常导入并各得一个已发布快照；
  失败码限于四个有界值；`document_not_executable_graph` 带门禁的问题列表（节点 id 与 kind、槽位、
  字段名），与编辑器发布同源；名称冲突不改名、不覆盖；下载、校验或解压失败是包级结果，不产生任何
  文档结果，下一次触发重试。
- 禁止结果：整包因一份文档失败、失败码之外的自由文本成为唯一原因、自动改名后导入、名称冲突时
  覆盖既有 workflow、包级失败仍留下部分文档结果。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 逐文档失败不牵连同包其他文档 | 真实 HTTP + PostgreSQL：混入坏文档的包导入后，其余文档各有 workflow 与快照，坏文档只有一条 `failed` | Missing |
| 失败码有界且可定位 | 真实 HTTP：四类失败分别返回四个错误码；图不可执行时问题列表点名节点/槽位/字段 | Missing |
| 门禁拒绝与编辑器发布同源 | 同一份不可执行图在导入与发布两条路径得到同一组问题 | Partial |
| 名称冲突不改名、不覆盖 | 真实 PostgreSQL：同名文档失败为 `name_conflict`，既有 workflow 的名称与图不变 | Missing |

- 决策依据：D4（逐文档结果、有界失败码、名称冲突不改名不覆盖）；[workflow 接缝决策](../../../decisions/cloud/workflow/20261016-asset-import-and-published-snapshots.md)
  D1（导入走发布门禁）。
- 门禁原语的直接证据：**Partial** — [`internal/core/workflow_graph.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph.go)
  的 `validateWorkflowGraph` 已经按首期子集拒绝并携带问题列表，[`integration/workflow_admission_test.go`](https://github.com/ora-space/cloud/blob/df507f0/integration/workflow_admission_test.go)
  的 `TestWorkflowPublishValidatesTheGraphAndSavingDoesNot` 与
  `TestWorkflowPublishRefusesUnusedAgentConfiguration` 在真实 HTTP 上断言了节点与字段被点名，单元矩阵是
  [`internal/core/workflow_graph_test.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/core/workflow_graph_test.go)
  的 `TestValidateWorkflowGraph`。这只证明「门禁会拒绝并点名」，不证明导入步骤调用了它、也不证明
  导入把问题列表放进 `document_not_executable_graph`；后者 `Missing`。
- 仍缺：导入路径整体不存在，除门禁原语外三条义务全部 `Missing`。

## The Asset Fetch Stays Inside The Declared Outbound Boundary

- 风险：第三方仓库在目录快照里给出的 `url` 成为 SSRF 入口，打到云内私有地址；跟随重定向绕过刚做过
  的地址判定；或无界下载把远端字节变成内存与带宽风险。
- 前置状态：目录快照已同步，`universal.url` 通过 `validateLocator`，形态为无凭据 HTTPS URL（可带
  签名 query、无 fragment）或 S3 object key；导入步骤在数据库事务外取包。
- 触发：DNS 应答落在私有、环回、链路本地、未指定或文档/保留网段；服务端返回 3xx；响应体超过
  16 MiB；连接、响应头或整包超时；S3 object key 形态取包。
- 必须成立：只接受 `https`（S3 object key 走 Cloud 对象存储的私有端点，不出公网、不加凭据）；host
  不设白名单，但连接前解析全部 DNS 应答，任一答案落在上述网段即拒绝；不使用环境代理；不跟随重定向，
  任何 3xx 按下载失败处理；连接 10 秒、响应头 30 秒、整包 60 秒；边读边计数，超过 16 MiB 立即中断；
  sha256 校验先于任何解析；失败是包级结果，下一次触发重试。
- 禁止结果：连接私有或环回地址、经重定向换目标后再下载、读完再判断大小、把下载失败降级成逐文档
  失败、S3 形态走公网或要求新凭据。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 定位符只允许两种形态，`https` 之外一律拒绝 | 目录同步拒绝 `http`、`s3://`、带 fragment 或带凭据的 URL；S3 object key 被识别 | Partial |
| 连接前判定全部 DNS 应答，拒绝私有/环回/链路本地/保留网段 | 真实 HTTP：指向这些地址的包下载失败，且没有发出连接 | Partial |
| 不跟随重定向 | 真实 HTTP：3xx 被记为包级失败，目标地址没有被请求 | Partial |
| 超时与流式字节上界 | 真实 HTTP：慢响应在 10/30/60 秒边界被取消；超 16 MiB 的响应在读完前中断 | Missing |
| 失败是包级且下一次触发重试 | 真实 PostgreSQL：下载失败不产生文档结果，重试后成功 | Missing |

- 决策依据：D3（触发点与取包、出站请求的约束）、继承表「根决策 D3：目录快照由 Cloud git 同步，
  读取永远不出网 → 修改」、不变量 3、4、8。
- 被复用原语的直接证据：**Partial** — [`internal/pluginmarket/manifest.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/pluginmarket/manifest.go)
  的 `validateLocator` 已固定定位符形态；[`internal/modelgateway/transport.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/modelgateway/transport.go)
  的 `publicAddress` / `secureDialer` 已实现「解析全部应答、任一落在禁止网段即拒绝、不用环境代理」；
  [`internal/objectstore/verify.go`](https://github.com/ora-space/cloud/blob/df507f0/internal/objectstore/verify.go)
  的客户端用 `ErrUseLastResponse` 拒绝重定向。这些证明规则在原语层面成立，不证明导入步骤接上了它们，
  也不证明使用了本文的 10/30/60 秒取值，因此按 `Partial` 记。
- 仍缺：超时取值的直接证据与包级失败语义都随导入步骤才有对象，记 `Missing`；重定向与 SSRF 目前只有
  原语证据，没有「导入请求因此失败」的端到端用例。

## Import Results Are Persisted Independently Of The Install Outcome

- 风险：导入结果只作为插件安装结果的一个字段，没有 live workspace 的空间拿不到模板；或 `installed`
  被当成「有模板」，用户看不出导入失败。
- 前置状态：管理员把 `kind = "workflow"` 插件安装到某个 space；该 space 可能没有 live workspace；
  Node 安装可能成功、失败或在途。
- 触发：没有 live workspace 的 space 安装 workflow 插件；Node 安装失败但 Cloud 导入成功；导入失败
  但 Node 安装成功；卸载插件后读取副本。
- 必须成立：导入由 space 期望状态触发，不依赖 Node 安装结果，也不 fan-out、不建 operation；导入
  结果独立持久化并按 `skip_key` 单独读取；四种证据（安装、接管、发布可用、执行可用）可分别判断，
  不能合成一个布尔值；卸载插件不删除副本、不改写快照。
- 禁止结果：把导入挂在安装 fan-out 上、用 `installed` 宣称有模板、导入失败改写插件安装状态、卸载
  时回收副本。
- 验证义务与证据：

| 义务 | 所需直接证据 | cloud 侧状态 |
| --- | --- | --- |
| 无 live workspace 的 space 也会导入 | 真实 HTTP + PostgreSQL：零实例的 space 安装后出现 workflow 与快照 | Missing |
| 导入结果独立持久化并可单独读取 | 真实 PostgreSQL：导入结果按 `skip_key` 读取，不依赖插件执行结果行 | Missing |
| 安装结局与导入结局互不推导 | 真实 HTTP：Node 安装失败时导入结果仍为 `imported`，反之亦然 | Missing |
| 卸载不回收副本 | 真实 PostgreSQL：卸载后 workflow 与快照仍在，发布可用性不变 | Missing |

- 决策依据：D3（触发不依赖 Node 安装结果）、D5（四种证据分离）、不变量 1、2、7。
- 仍缺：导入步骤与来源记录都不存在，四条义务全部 `Missing`；既有插件聚合测试只覆盖安装状态，
  不能当作本用例的证据。
