# 仓库凭据分类、成员离开与受控重绑

依据[已批准的凭据连续性决策](../../../decisions/cloud/project/20260927-repository-credential-continuity.md)。新增义务按直接证据登记 Partial/Missing。
Cloud 保存引用和业务状态，部署凭据提供方保存秘密；Ora 权限不等同外部 Git 平台授权。

## Member Departure Freezes Personal Or Unknown References Without Deleting Local Work

- 风险：创建者离开后继续使用其个人凭据，或为收紧远程权限误删团队代码与任务。
- 前置状态：项目分别绑定团队、个人和未知引用；有本地克隆、已开始任务、未接受远程意图和未知派发结果。
- 触发：关联成员离开/停用，恢复成员关系，再请求 clone/fetch/pull/push、本地编辑、启动或清理。
- 必须成立：个人/未知引用不供新远程操作，团队引用不因旧 owner 离开误停；恢复成员不自动解冻。按原范围收尾，未知派发需核对；本地内容和不需远程的启动按权限/独占规则继续，清理审计引用保留。
- 禁止结果：自动转移个人凭据、冻结全部本地工作、删除项目、把外部凭据吊销导致的任务失败承诺为成功。
- 验证义务与证据：**Partial** — 需 PostgreSQL 停用/派发竞争，真实 Node/Git 与受控凭据提供方的三类引用测试；检查远程拒绝及本地保真。
- 决策依据：D1、D2，不变量 1、2。

- 本次直接证据：**Partial** — [Cloud integration/repository_credentials_test.go](https://github.com/wanglongan587/cloud/blob/codex/runtime-control/integration/repository_credentials_test.go) 的个人/未知冻结、团队保留、历史引用与重入不解冻。
- 仍缺：受控真实 Git 认证对端、已有任务及本地数据的联合停用矩阵尚缺。

## A Team Reference Becomes Active Only After Scoped Capability And Version Validation

- 风险：管理员声明“团队”即获得另一租户凭据，或迟到验证恢复过期绑定。
- 前置状态：原绑定可用或冻结，部署提供合法/越界候选；另有并发绑定更新、成员停用或引用撤销。
- 触发：管理员提交候选和版本/幂等键，普通成员提交，返回成功/失败/未知或迟到验证结果。
- 必须成立：仅管理员可选受控范围内引用；结果绑定引用版本、仓库及具体读/写能力，读取成功不等于可推送。仅候选和项目版本仍匹配时原子切换；失败保留原绑定及冻结状态，重放不重复验证责任。
- 禁止结果：Cloud 接收秘密或直接任意远端验证、失败覆盖原绑定、并发更新后迟到结果生效、改写 owner/创建者/任务 actor。
- 验证义务与证据：**Missing** — 需真实 HTTP/PostgreSQL 与受控 Git 对端的范围、能力、版本、重试及响应丢失测试。
- 决策依据：D3，不变量 3。

- 本次直接证据：**Missing** — 无受控团队验证重绑的直接证据。
- 仍缺：受控凭据解析器及管理员验证后原子重绑入口尚未实现，保持关闭；仍为 Missing。

## Git Execution Uses Its Planned Reference Without Static Credential Fallback

- 风险：Cloud 标冻结，但 Node 仍用全局 SSH 配置，或用户代码读取长期管理密钥后绕过引用治理。
- 前置状态：执行计划固定引用、版本、仓库和能力；全局环境中另有可用凭据，候选分别失效或越界。
- 触发：新远程动作、旧任务收尾、提供方拒绝、Git 平台吊销；工作负载尝试读取管理材料或转交资格。
- 必须成立：只解析计划范围内引用，缺失/失效/能力不符明确拒绝，无静态回退；临时材料按范围提供并在结束后回收，不进入工作目录、Cloud、前端或日志。旧任务不能向新任务转交资格；外部失败返回安全分类。
- 禁止结果：复用另一项目秘密、复制长期 token/密钥、提供方失败时自动读取个人 SSH、宣称 Ora 撤销外部个人账号权限。
- 验证义务与证据：**Partial** — 需真实 Git 认证对端及沙盒权限测试，特意提供可用回退凭据并断言仍拒绝；检查进程、日志和持久目录无秘密。
- 决策依据：D4，不变量 4；跨进程交付 D2。

- 本次直接证据：**Partial** — [Cloud integration/repository_credentials_test.go](https://github.com/wanglongan587/cloud/blob/codex/runtime-control/integration/repository_credentials_test.go) 证明缺解析器的引用凭据拒绝及不匿名回退；Rust Node 网络配置拒绝全局静态个人 Git 凭据。
- 仍缺：按项目引用的真实临时凭据解析/发放/秘密扫描尚缺，相关远程执行保持关闭。

## Credential Upgrade Preserves Unknown Ownership And Historical References

- 风险：迁移按 owner 猜测团队归属，或切换引用删除原执行和恢复关联。
- 前置状态：旧库有静态部署配置、历史引用、owner 外键、运行和失败验证记录；另准备空库。
- 触发：追加迁移并重入、使用不支持新能力的旧执行端、切换团队引用、外部引用过期。
- 必须成立：历史无法证明的类型为未知，保留关联和既有有效成员下的使用边界；离开时收紧。新库/升级均保留 owner 及原执行引用版本，旧执行端明确拒绝新能力；可用性及验证状态可审计且无秘密。
- 禁止结果：修改已发布 SQL、静态 SSH 默认升级为团队、切换删除历史引用、旧端静默忽略引用能力。
- 验证义务与证据：**Partial** — 需真实 PostgreSQL 新库/旧库升级和完整快照比较，联合旧新执行端兼容与真实凭据失效测试。
- 决策依据：D1、D5，不变量 5。

- 本次直接证据：**Partial** — [Cloud integration/runtime_upgrade_test.go](https://github.com/wanglongan587/cloud/blob/codex/runtime-control/integration/runtime_upgrade_test.go) 的 unknown、旧 owner/credential FK 与 operation 快照；[Cloud integration/repository_credentials_test.go](https://github.com/wanglongan587/cloud/blob/codex/runtime-control/integration/repository_credentials_test.go) 的失效触发器。
- 仍缺：完整执行历史快照、旧新组件组合与真实认证对端联合升级矩阵尚缺。

