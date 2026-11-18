# 项目仓库：绑定、拉取与当前快照、手动同步、私有仓库令牌

依据 [项目仓库根决策](../../../decisions/cloud/project-git/0-external-repository-pull-and-encrypted-token.md)（`implemented`）。
下文的 D 编号与不变量编号都指这篇根决策。

证据读取的是 cloud `affb2a885d81eca9aed98dca84fbfc486c1f7429`（PR ora-space/cloud#375 合并后）的 `v2-server/`
（路径相对该目录）。手动同步、随同步推进的当前快照和私有仓库令牌已由 I7（circus #1057）实现。

## A Project Binds One HTTPS Repository Without Embedded Credentials

- 风险：仓库地址带着凭据、用了别的协议或夹带 Git 选项，凭据因此进了数据库和日志，或者 Cloud 被引去执行意外的命令。
- 前置状态：用户正在建项目。
- 触发：提交 `https` 地址；提交带用户信息、查询串或片段的地址；提交 `http`、`ssh`、`file`、本地路径和以 `-` 开头的字符串；
  拉取失败。
- 必须成立：只接受不带用户信息、查询串和片段的 `https` 地址；其余在访问网络之前被拒绝；拉取失败时项目不创建。
- 禁止结果：带凭据的地址被保存；非 `https` 的地址被访问；拉取失败却留下项目。
- 验证义务与证据：**Covered** — `internal/projectrepos/projects_test.go` 的 `TestRepositoryURLPolicy`；
  `internal/server/projects_test.go` 的 `TestProjectRepositoryLifecycle`；
  `internal/projectrepos/sync_test.go` 的 `TestPrivateRepositoryTokenCreatesSyncsAndReportsRejection`。已覆盖：地址规则；
  建项目时拉取并记录提交；私有无令牌时建项目失败、不留下项目。地址和分支建后不可修改由没有修改接口保证，不是靠测试。
- 决策依据：D1。

## The Current Snapshot Moves Only After A Successful Pull

- 风险：同步失败或中途崩溃后，当前快照指向不存在或不完整的提交；已有的会话被同步改了基准。
- 前置状态：一个已建好的项目，有当前快照和以它为基准的会话目录；外部分支有了新提交。
- 触发：同步成功；同步因地址不可达、认证失败、分支被删除或超时而失败；外部分支被强推；同步途中 Cloud 重启。
- 必须成立：成功时写入新的快照 ref 并推进当前快照，新会话以它为基准；失败和中断时当前快照不变，原因可见；强推时
  跟随并保留旧快照；已有会话目录的基准不变；快照 ref 写入后不再改变。
- 禁止结果：当前快照指向没有快照 ref 的提交；旧快照被删除或改写；已有会话的基准随同步变化。
- 验证义务与证据：**Partial** — `internal/projectrepos/sync_test.go` 的 `TestSyncAdvancesCurrentSnapshotOnlyOnSuccess`、
  `TestSyncFollowsForcePushedBranchAndKeepsOldSnapshot`、`TestSyncFailureKeepsCurrentSnapshotAndShowsReason`；
  `internal/server/project_sync_test.go` 的 `TestProjectSyncAndRepositoryToken`。已覆盖：同步成功写入新快照 ref 并推进
  当前快照、旧快照保留；强推跟随并保留旧快照；失败（分支删除）时当前快照不变、原因可见；HTTP 层同步推进与失败状态。
  仍缺：同步中途 Cloud 重启的中断路径（`store.ClearStaleSyncs`）和"已有会话目录的基准不随同步变化"没有直接测试；
  "新会话以推进后的快照为基准"沿用建会话的既有机制，无针对同步推进的直接断言。
- 决策依据：D2、D3，不变量 2、3、5。

## Only One Sync Runs Per Project And Only On Request

- 风险：同一个项目两次同步并行写镜像；Cloud 在没人要求时访问外部仓库。
- 前置状态：一个已建好的项目。
- 触发：同步进行中再次点同步；建会话；长时间无人操作。
- 必须成立：进行中再点返回正在进行的那一次；建会话不触发同步；没有周期性的访问。
- 禁止结果：同一个项目两次同步同时进行；Cloud 自行访问外部仓库。
- 验证义务与证据：**Partial** — `internal/projectrepos/sync_test.go` 的 `TestSyncRunsOncePerProject`
  （两个并发同步并发探针只被探测一次）和 `TestSyncOutlivesTheRequestThatStartedIt`（发起同步的请求结束后共享的拉取仍完成）。
  已覆盖：同一项目同时最多一次同步、进行中再点共享同一次。仍缺：没有直接测试"建会话不触发同步"和"没有周期性访问"；
  这两者由实现只在建项目和显式同步时访问外部仓库保证（不变量 7），但不是测试证据。
- 决策依据：D3，不变量 4、7。

## The Repository Token Stays Inside Cloud

- 风险：私有仓库的令牌出现在日志、API 应答、命令行或下发给 Node 的配置里；数据库泄露后密文被挪到别的项目使用。
- 前置状态：项目配置了私有仓库令牌；部署配置了加密密钥。
- 触发：读取项目相关的 API；执行拉取并检查日志、子进程的命令行和写给 Node 的配置；把一个项目的密文复制到另一个项目；
  去掉加密密钥后启动 Cloud；普通成员尝试更换令牌；公开仓库不填令牌；外部平台拒绝令牌。
- 必须成立：API 只返回是否已配置和最后更换的时间；明文只在 Cloud 进程和它启动的 Git 子进程里；挪用的密文解密失败；
  密钥缺失时 Cloud 照常启动，需要令牌的拉取失败并写明原因；只有创建者和团队管理员能更换和删除；公开仓库匿名拉取；
  被拒绝时提示更换，不自动停用。
- 禁止结果：令牌明文进入日志、API 应答、命令行参数、地址、Git 配置或 Node 的配置。
- 验证义务与证据：**Partial** — `internal/projectrepos/sync_test.go` 的 `TestRepositoryTokenSealsToBindingAndNeverLeaks`
  和 `TestPrivateRepositoryTokenCreatesSyncsAndReportsRejection`；`internal/server/project_sync_test.go` 的
  `TestProjectSyncAndRepositoryToken`。已覆盖：密文落库且不含明文；挪到别的项目解不开；令牌只经 Git 子进程的环境使用、
  不进镜像配置和 API 应答；API 只返回是否已配置和最后更换时间；普通成员更换被拒（403）、创建者/管理员可更换删除；
  公开仓库匿名拉取；令牌被拒时同步失败、快照不变并提示更换令牌。仍缺：没有直接测试"去掉加密密钥后 Cloud 照常启动、
  需要令牌的拉取失败并写明原因"。
- 决策依据：D4，不变量 6。
