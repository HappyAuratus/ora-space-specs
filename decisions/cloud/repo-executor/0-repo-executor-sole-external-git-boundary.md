---
status: proposed
---

# Repo Executor 是唯一访问外部仓库的程序：按 Operation 一次性授权执行同步与发布，结果只经 Git Server 交接

Ora 中只有一个程序访问外部 Git 仓库（GitHub、GitLab 等）：Repo Executor（下称“执行器”）。它是 Cloud 仓库中与 Git Server、
model-gateway 并列的独立程序，无状态，不连 Cloud 数据库，不保存仓库。Cloud 为每个同步或发布 Operation 签发一份一次性授权，
经 mTLS 主动调用执行器；执行器凭授权向 Credential Store 一次性兑换凭据，写入私有临时目录，只交给固定的 credential helper，
在临时裸仓库里完成一次 Git 操作后销毁一切。同步的产出只经 Git Server 的服务仓库监听写入快照 ref 和跟踪 ref；发布只从 Git Server
读取源提交，再以 expected old 推送到外部仓库，并以 `ls-remote` 前后核验。执行器只报告事实，Operation 的状态由 Cloud 决定。

本 ADR 处于 `proposed`，是新建领域 `cloud/repo-executor` 的根决策（同时建立镜像目录 `test-cases/cloud/repo-executor/`）。
它实现 [Git Server 代码中心契约](../git-server/20261016-project-repository-as-code-hub-and-ref-contract.md)（下称“契约”）中
“执行器”一方的全部义务。涉及 Cloud 仓库新增的 `cmd/repo-executor`、Cloud 的 Operation worker、Credential Store，以及 cluster
仓库的部署。业务流程分别由 [同步 ADR](../project/20261016-repository-binding-and-snapshot-sync.md) 与
[发布 ADR](../project/20261017-explicit-fast-forward-publish.md) 规定，凭据的保管与能力由
[identity-access 后续决策](../identity-access/20261016-git-credentials-in-independent-credential-store.md) 规定。
执行器和 Credential Store 的部署并入本文（D9），不单独成篇。

兼容性边界：

- 早先未合并的凭据执行器实现与其部署（下称“旧实现”）一概不依赖，最多作参考：本文需要的基础件按“复用边界”一节新写，或参考其设计
  摘出后作为新代码连同测试评审，其余按本文重写。
- 破坏性切换（契约范围边界）：不设过渡开关，不保留外部 clone 与 bundle 的旧路径，不做数据迁移。
- 本文不改变沙盒、Node 与 Git Server 的任何约定；执行器不接入沙盒网络。

## 现状与继承

| 前序约定或现状 | 本文处理 |
| --- | --- |
| 契约 D2：服务仓库监听只接受 `spiffe://ora.local/repo-executor` 服务证书；证书只证明身份，不带权限 | 继承，不变。执行器访问 Git Server 只用这张证书（D2） |
| 契约 D3：服务仓库监听上的请求必须且只带 `Ora-Git-Operation-Id`；授权按 Cloud 当前状态逐请求、`pre-receive` 再判定 | 继承，不变。执行器的每个 Git 请求都带本次 Operation 的 ID（D5、D6） |
| 契约 D4、D5 第 3 项：同步可写本 Binding 的快照 ref（create-only）与跟踪 ref（tracking-cas）；发布只能读源 ref，不能写 | 继承，不变。执行步骤只使用这些权限（D5、D6） |
| 契约 D6：执行器报告结果，Cloud 经 ref 查询核验后登记 | 继承，不变。执行器不登记任何东西（D1） |
| [仓库凭据连续性](../project/20260927-repository-credential-continuity.md) D4：执行端按引用选择凭据，不回退全局凭据；凭据临时提供，结束后回收，不进工作目录和日志 | 继承。执行端从 Controller/Node 改为本执行器，兑换与销毁的方式见 D3 |
| [默认分支决策](../operation/20260926-require-concrete-default-branch.md)：Cloud 不向用户给出的地址发起请求 | 继承，不变。访问外部地址的只有执行器，并受出口策略约束（D4） |
| 未合并的旧执行器实现：clone 执行器与 mirror 发布 | 不依赖，只作参考。不沿用其 clone、checkout 交付和直接写 Git Server 卷的 mirror 发布；基础件按“复用边界”新写或参考摘出 |

## 决策

### D1：程序边界

- 执行器是 Ora 中**唯一**向外部 Git 仓库发起网络请求的程序。Cloud、Node、Workspace、Git Server 都不访问外部仓库。
- **无状态**：不连接 Cloud 数据库，不保存仓库、凭据或任务队列。每个任务在私有临时目录里新建裸仓库，任务结束（成功、失败、
  超时、取消）后整体删除。进程启动时清扫上次遗留的临时目录。
- **不做授权决定**：能不能做由 Cloud 在签发授权时决定，能写哪些 ref 由 Git Server 按契约判定。执行器只核对“收到的任务与授权
  一致”，不一致就拒绝。
- **只报告事实**：返回 Git 操作观察到的 OID 和结果分类（D8），不改写 Operation 状态。Cloud 用 ref 查询（同步）或核验规则
  （发布）决定最终状态。

为什么无状态：执行器持有外部网络出口和短暂的明文凭据，是攻击面最大的组件。让它不持久化任何东西，失陷的影响就限于正在运行
的任务；重启也不需要恢复流程，未完成的任务由 Cloud 的租约和核验处理。

### D2：身份、调用与一次性授权

**调用方向**：Cloud 的 Operation worker 主动调用执行器，执行器不轮询 Cloud。

| 连接 | 客户端身份 | 服务端 |
| --- | --- | --- |
| Cloud → 执行器 | `spiffe://ora.local/cloud/repo-dispatcher` | 执行器的任务接口（mTLS，TLS 1.3） |
| 执行器 → Credential Store | `spiffe://ora.local/repo-executor` | Credential Store 的兑换接口 |
| 执行器 → Git Server | `spiffe://ora.local/repo-executor` | 服务仓库监听（契约 D2） |
| 执行器 → 外部仓库 | 兑换得到的凭据，或匿名 | 外部平台 |

执行器只接受 Cloud 的固定客户端身份。为什么由 Cloud 调用：Operation 的租约、代次和终态都在 Cloud；由 Cloud 发起，执行器就
不需要数据库连接和队列，也不需要另一种“执行器领取任务”的身份。

**一次性授权**采用旧实现中 `cloneauth` 的格式（参考其设计）：`base64url(payload).base64url(Ed25519 签名)`，签名覆盖原始字节；验证方持有
公钥集合，以便轮换签名密钥。签名私钥只在 Cloud。本文在原有用途之外新增三种用途，并让它们与原用途互斥：

| 用途 | 必填的绑定字段 | 执行器能做的事 |
| --- | --- | --- |
| `sync` | `operationId`、`tenantId`、`projectId`、`bindingId`、`repository`、`branch`、`expectedTrackingOid`，以及凭据 `credentialId` + `credentialVersion`（公开仓库为空） | D5 |
| `publish` | `operationId`、`tenantId`、`projectId`、`bindingId`、`repository`、`sourceRef`、`sourceCommit`、`targetRef`、`expectedOldOid`、`credentialId` + `credentialVersion` | D6 |
| `probe` | `operationId` 或 `verificationId`、`repository`、`targetRef?`、`service`（`upload-pack` 或 `receive-pack`）、`credentialId` + `credentialVersion` | D7：只读取 ref 广告，不传输对象、不写入 |

公共字段：`authorizationId`、`executorIdentity`（必须等于 `spiffe://ora.local/repo-executor`）、`issuedAt`、`expiresAt`。
`sync`、`publish`、`probe` 都禁止携带 Node 和运行时的围栏字段，`clone`、`mirror_prepare`、`push` 三个旧用途不再签发。

- **授权只约束“开始任务并兑换凭据”这一步**：有效期上限 `MaxLifetime = 15 分钟`，实际签发不超过 2 分钟；执行器收到任务时
  授权必须有效，Credential Store 兑换时也以有效期判定。兑换之后，任务能跑多久不由授权决定，而由 Operation 的租约决定：
  Cloud 的 worker 在任务期间持续续约，Git Server 对每个请求按租约判定（契约 D3）。同步大仓库可能远超 2 分钟，用授权的有效期
  约束任务时长会让它们必然失败。
- 一个 Operation 至多进入一次 `running`（契约不变量 14），授权只在进入 `running` 时签发一次，所以同一个 Operation 不会有两份授权。
- 执行器先在本地验证签名和有效期，再逐项比对任务与授权的绑定字段，任何一项不一致就拒绝，不兑换凭据。
- **单次使用**由 Credential Store 保证：兑换时以授权 ID 独占创建记录，第二次兑换被拒绝。公开仓库的 `sync` 不兑换凭据，
  其单次性由 Operation 只进入一次 `running`（同步 ADR D2）和 Git Server 的租约检查保证。

为什么新增 `probe` 而不是复用 `publish`：能力验证和 `uncertain` 的核验都只需要读取 ref 广告。给它们 `publish` 授权，执行器
在这两种场合就具有推送的能力；单列一个只读用途，授权本身就表达了“这次不会写”。

### D3：凭据兑换与使用

1. 执行器以 `spiffe://ora.local/repo-executor` 身份、带完整授权向 Credential Store 兑换。Credential Store 验证签名、用途、
   绑定的凭据与版本、单次性后返回明文（identity-access 后续决策 D2）。
2. 执行器把明文写入私有临时目录：目录 `0700`，文件 `0600`，以 `O_CREATE|O_EXCL` 创建并 fsync；同时写入作用域（host 与 path）。
3. Git 只经固定的 credential helper `git-credential-ora` 取得凭据：先以 `-c credential.helper=` 清空，再指向 helper 的绝对路径
   与材料目录，并设 `credential.useHttpPath=true`。helper 只在 Git 请求的 host 与 path 等于作用域时返回 `password`；`store`
   和 `erase` 什么也不做。
4. 任务结束的所有路径（成功、失败、超时、取消、panic）都删除材料目录并确认已删除；删除失败作为清理失败一并报告。

明文凭据**不进入**：命令参数、环境变量、远端 URL（URL 中带用户信息的任务直接拒绝）、Git 配置文件、日志、错误信息和返回给
Cloud 的任何字段。捕获的 stderr 在截断前先把明文替换为 `[redacted]`。

为什么用 helper 加文件，不用 `http.extraHeader` 或 URL：`-c http.extraHeader=Authorization: ...` 会让凭据出现在进程参数里，
`/proc/<pid>/cmdline` 对同一用户可读；URL 中的凭据会进入 Git 的错误输出和 reflog。helper 加 `0600` 文件让明文只存在于执行器
自己的临时目录和 Git 进程的内存中。

### D4：进程与资源边界

**进程**：每个 Git 子进程沿用 `processgroup.Isolate`：`Setpgid`，取消时向整个进程组发 `SIGKILL`，`WaitDelay` 5 秒；容器以
`init: true` 运行，由 init 回收孤儿进程。每个任务有总超时（同步默认 30 分钟，发布与 probe 默认 5 分钟），由 Cloud 在任务请求中给出，不超过部署的上限；它与授权的
有效期无关。Cloud 在任务结束前一直续约；worker 失联、租约过期后，执行器后续对 Git Server 的请求会被拒绝。

**输出**：stdout 与 stderr 各用有界缓冲 `capture.Buffer`（每流 64 KiB），写满后只计数，不阻塞子进程；stdout 被截断时，结果
不作为事实采用。报告给 Cloud 的错误摘录先脱敏、再截断到 2 KiB。

**环境**：从空环境构造：`GIT_CONFIG_NOSYSTEM=1`、`GIT_CONFIG_GLOBAL=/dev/null`、`HOME` 指向任务目录、固定 `PATH`、`LC_ALL=C`、
`GIT_TERMINAL_PROMPT=0`、`GIT_ASKPASS=`、`SSH_ASKPASS=`、`GIT_LFS_SKIP_SMUDGE=1`、`GIT_OPTIONAL_LOCKS=0`；stdin 为空。

**加固的 Git 配置**（参考旧实现中两份相同的列表，合并为一份）：

```text
protocol.allow=never  protocol.https.allow=always  protocol.{ssh,file,ext}.allow=never
core.hooksPath=/dev/null  core.askPass=  core.symlinks=false
http.followRedirects=false  http.proxy=  http.sslVerify=true  http.lowSpeedLimit=1000  http.lowSpeedTime=60
submodule.recurse=false  fetch.recurseSubmodules=false
filter.lfs.smudge=  filter.lfs.process=  filter.lfs.required=false
```

**大小**：Git 的 fetch 本身没有大小上限，执行器监控任务目录的占用，超过部署配置的上限（默认与 Git Server 的
`receive.maxInputSize` 相同）立即杀死进程组，以 `too_large` 报告。Git Server 的 `receive.maxInputSize` 与仓库配额是第二道检查。

**出口**：执行器只向外部 `https://` 地址发起请求，并在连接前解析目标地址：

- 拒绝回环、私有、链路本地、ULA 和元数据服务地址，拒绝内部服务的主机名；部署可以给出允许列表（验收用的本地 fixture）；
- 不跟随重定向（`http.followRedirects=false`），所以首个请求通过检查后不会被引到内部地址；
- 访问 Git Server 和 Credential Store 用固定的内部地址，不受用户输入影响。

为什么执行器自己做地址检查：同步的 URL 来自用户。执行器与 Git Server 服务仓库监听、Credential Store 在同一个内部网络，
不检查就等于给用户一个以执行器身份访问内部服务的 SSRF 入口。

### D5：sync 的执行步骤

输入是 `sync` 授权及同名任务字段。

1. 在任务目录 `git init --bare`。
2. **取已有的跟踪 ref 作为基础**（可选优化）：带 `Ora-Git-Operation-Id` 从服务仓库监听 fetch
   `refs/ora/upstreams/<bindingId>/heads/<branch>`。获授权的请求可读整个 Project 仓库（契约 D3），这一步让下一步只从外部下载增量。
3. 加固的 fetch：`fetch --quiet --no-tags --no-write-fetch-head --no-recurse-submodules -- <repository>
   +refs/heads/<branch>:refs/heads/ora-sync`。之后检查任务仓库里除第 2 步的 ref 外只有 `refs/heads/ora-sync`。
   外部仓库没有该分支时报告 `upstream_branch_missing`。
4. 解析 `refs/heads/ora-sync` 得到 `<commitOid>`，确认它是提交对象。以 `merge-base --is-ancestor <expectedTrackingOid> <commitOid>`
   计算 `forced`（`expectedTrackingOid` 为全零时为 `false`）。
5. 一次 `--atomic` 推送到服务仓库监听，带 `Ora-Git-Operation-Id`：

   ```text
   git push --atomic --porcelain --no-verify \
     --force-with-lease=refs/ora/upstreams/<b>/heads/<branch>:<expectedTrackingOid> \
     <git-server> <commitOid>:refs/ora/upstreams/<b>/snapshots/<commitOid> \
                  <commitOid>:refs/ora/upstreams/<b>/heads/<branch>
   ```

   全零的 `expectedTrackingOid` 表示跟踪 ref 必须不存在。Git Server 的 `pre-receive` 再按 create-only 和 tracking-cas 判定
   （契约 D4）。两个 ref 要么一起写入，要么都不写。

   **refspec 不带 `+`**。带 `+` 时 Git 客户端把这个 ref 当作强制更新，`--force-with-lease` 声明的旧值不再生效，连“必须不存在”
   也会被覆盖（实测）。不带 `+` 时，lease 正确地以 `stale info` 拒绝旧值不符的更新；旧值相符时，`--force-with-lease` 本身允许
   非快进，服务仓库监听又关闭了 `receive.denyNonFastForwards`（契约 D2），所以跟踪 ref 仍能跟随外部 force-push。
6. 报告 `{commitOid, snapshotRef, trackingRef, forced}`。

**重放**：同一提交再次同步时，两个 ref 已经是目标值，Git 不发送更新命令，推送是空操作，按成功报告。

**推送被拒**：执行器按 porcelain 报告区分两种拒绝，二者含义不同：

| 拒绝原因 | 含义 | 处理 |
| --- | --- | --- |
| `stale info`（lease 不符） | 跟踪 ref 不等于 `expectedTrackingOid` | 重新读取跟踪 ref：已经等于 `<commitOid>` 且快照 ref 存在，说明同一结果已经写入，按成功报告；否则报告 `superseded`，附观察到的值 |
| `pre-receive hook declined` | Git Server 按授权或规则拒绝：Operation 已结束、租约过期、写了表外的 ref、快照名与目标不符 | 报告 `failed(git_server_rejected)`，附 Git Server 返回的原因，不判为 `superseded` |

### D6：publish 的执行步骤

输入是 `publish` 授权及同名任务字段。执行器在 Git Server 上没有写权限；它可读整个 Project 仓库（契约 D3），但只取 Operation
记录的源 ref 与源提交（契约 D5 第 3 项）。

1. 在任务目录 `git init --bare`。
2. 带 `Ora-Git-Operation-Id` 从服务仓库监听 fetch `sourceRef`。源 ref 可能已经前进，但 Workspace 分支只能快进、交付 ref 不可变
   （契约 D4），所以 `sourceCommit` 一定是取回的顶端或其祖先。以 `merge-base --is-ancestor <sourceCommit> FETCH_HEAD` 确认；
   不成立报告 `source_unavailable`。
3. **只快进**：`expectedOldOid` 非全零时，确认它是 `sourceCommit` 的祖先（`merge-base --is-ancestor`）。判定只用第 2 步从
   Git Server 取回的对象：快进时 `expectedOldOid` 必在源提交的历史中；不在其中，说明不是快进，报告 `not_fast_forward`，
   不访问外部仓库。
4. **前置核验**：`ls-remote --refs -- <repository> <targetRef>`。结果不等于 `expectedOldOid`（不存在按全零比较）时报告
   `conflict`，附观察到的值，不推送。
5. **推送**：

   ```text
   git -c push.followTags=false push --porcelain --no-verify \
     --force-with-lease=<targetRef>:<expectedOldOid> -- <repository> <sourceCommit>:<targetRef>
   ```

   `--force-with-lease` 在这里的作用是比较并交换：外部平台在 ref 锁内比对旧值。快进已由第 3 步保证，所以它不会产生非快进写入。
6. **后置核验**：再次 `ls-remote` 读取 `targetRef`，与推送报告一起按 D8 分类。

为什么推送前也要 `ls-remote`：旧实现只在推送被拒后才读取。前置读取让最常见的冲突（外部已经前进）在不打开 receive-pack 的
情况下被发现，审计里也能记录冲突时外部的实际值；真正的原子性仍由第 5 步的 lease 保证，前置读取不替代它。

为什么不复用旧实现的 PushRequest：它要求新提交已经在外部仓库的对象中，只能把外部 ref 移到外部
已有的提交上，无法把 Workspace 的内容带出去。本文的源对象来自 Git Server，执行器把它们推送到外部，这是它与旧路径的根本区别。

### D7：probe 的执行步骤

只读取一次 ref 广告，不下载对象，不发送更新命令：

- `service = upload-pack`：`ls-remote --refs -- <repository> [<targetRef>]`，用于 `uncertain` 的核验（发布 ADR D5）。
- `service = receive-pack`：请求外部仓库 `git-receive-pack` 的 ref 广告（smart HTTP 的 `info/refs?service=git-receive-pack`），
  收到广告即断开，用于验证推送能力（identity-access 后续决策 D4）。

报告 `{refs | advertised, httpStatus 分类}`。各平台对“无写权限时请求 receive-pack 广告”的响应在落地时逐一验证（GitHub、GitLab
至少各一），验证结果记录在实现的测试里。

### D8：结果分类

执行器为每个任务返回一个分类，Cloud 据此决定 Operation 状态（同步 ADR D2、发布 ADR D3）：

| 分类 | 含义 | 是否可能已改变外部或 Git Server |
| --- | --- | --- |
| `applied` | 写入已确认：同步的原子推送成功或重放为空操作；发布的推送报告 `ok` | 是，已知 |
| `superseded` | 同步的 lease 以 `stale info` 失败，跟踪 ref 被别人推进 | 否 |
| `conflict` | 发布的前置核验或 lease 发现外部 ref 不等于 `expectedOldOid` | 否 |
| `rejected` | 外部平台明确拒绝了更新（保护分支、hook、无权限），后置核验确认目标仍是 `expectedOldOid` | 否 |
| `not_fast_forward`、`source_unavailable` | 本地判定不满足前提，没有访问外部仓库的写入接口 | 否 |
| `failed` | 确定没有写入的失败：授权不符、兑换失败、外部认证失败、fetch 失败、超时、超限、Git Server 的 `pre-receive` 拒绝 | 否 |
| `uncertain` | 推送请求可能已经发出，但没有得到可解析的结果：连接中断、超时、报告缺失或被截断，且后置核验也无法给出结论 | 可能 |

发布的推送报告与后置核验的组合：

| 推送报告 | 后置 `ls-remote` | 分类 |
| --- | --- | --- |
| `ok`（` ` 或 `+`）| 任意 | `applied`；记录观察值，不等于 `sourceCommit` 时标注“发布后外部又被改动” |
| 拒绝（`!`），原因是 lease / stale info | 任意 | `conflict` |
| 拒绝（`!`），其他原因 | 等于 `expectedOldOid` | `rejected`，附脱敏的平台原因 |
| 拒绝（`!`），其他原因 | 等于 `sourceCommit` | `applied`，标注“结果由核验确认” |
| 拒绝（`!`），其他原因 | 其他值 | `conflict` |
| 无报告、被截断、连接中断 | 等于 `sourceCommit` | `applied`，标注“结果由核验确认” |
| 无报告、被截断、连接中断 | 等于 `expectedOldOid` | `uncertain`：推送可能仍在外部排队，不能断定未生效 |
| 无报告、被截断、连接中断 | 其他值或读取失败 | `uncertain` |

**`uncertain` 不由执行器重试**。执行器不重推，也不在同一任务里反复轮询；之后的核验由 Cloud 以新的 `probe` 授权发起
（发布 ADR D5），核验只读。为什么不像旧实现那样重新派发同一推送：重新派发可能发生在批准很久之后，那时外部 ref 也许被别人
回退到 `expectedOldOid`，重推就会在用户不知情时再次发布；只读核验永远不会产生新的外部写入。

### D9：部署

| 组件 | 网络 | 出口 | 挂载 |
| --- | --- | --- | --- |
| 执行器 | `repo-internal`（内部服务网络）、`repo-egress` | 只有 `repo-egress` 可以出外网 | 只读：服务证书、Cloud 的授权验证公钥、`git-credential-ora`；可写：tmpfs 任务目录 |
| Credential Store | `repo-internal`；浏览器入口经边缘反向代理 | 无 | 可写：凭据卷；只读：KEK、服务证书、Cloud 的授权验证公钥 |
| Git Server 服务仓库监听 | `repo-internal` | 无 | 契约 D2 |
| Cloud | 默认网络与 `repo-internal` | 无 | 授权签名私钥只在 Cloud |

- 执行器**不接入沙盒网络**，沙盒也访问不到 `repo-internal`。这样持有外部凭据和外网出口的组件与工作负载之间没有直接路径
  （契约 D2）。
- 执行器和 Credential Store 都不发布宿主机端口；Credential Store 的浏览器入口只经边缘反向代理暴露（identity-access 后续决策 D3）。
- `repo-egress` 是唯一有外网出口的网络。部署可以在它上面加出口代理；执行器的地址检查（D4）是必须的，出口代理是附加的。
- 执行器以非 root 用户运行，任务目录是 tmpfs，容器重启即清空。
- Cloud worker 的并发上限不超过执行器声明的并发槽位；执行器以“容量已满”拒绝时，任务没有开始，按 `failed(executor_busy)` 报告。

## 复用边界

| 来自旧实现| 处理 | 边界 |
| --- | --- | --- |
| `internal/cloneauth` | 参考摘出，作为新代码评审 | 保留格式、签名、公钥集合与 `MaxLifetime`；新增 `sync`、`publish`、`probe` 三种用途及其字段校验；旧用途停止签发 |
| `internal/credentialmaterial`、`cmd/git-credential-ora` | 参考摘出，作为新代码评审 | 行为不变 |
| `internal/processgroup`、`internal/capture` | 参考摘出，作为新代码评审 | 行为不变 |
| `internal/credentialstore` | 参考其设计新写 | 见 identity-access 后续决策 D2：新增浏览器入口与兑换身份，去掉 Cloud 写入明文的能力 |
| `cloneexec/git.go` 与 `mirrorpublish/fetch.go` 的加固参数 | 合并为一份 | 两份列表相同，执行器只保留一份；新增目录占用上限 |
| `cloneexec/push.go`、`pushgit.go` 的 lease 推送与报告解析 | 参考后重写 | 源对象改为从 Git Server 取得；新增前置 `ls-remote`；`uncertain` 不重推 |
| `core/push_worker.go` 的租约与代次栅栏 | 参考后重写 | 授权只在进入 `running` 时签发一次；“重新领取并重推”改为只读核验 |
| mirror store、mirror 发布器、匿名 mirror 监听、clone 与 checkout 交付 | 不保留 | 与契约冲突：直接写 Git Server 卷绕过授权（契约“替代方案”） |

## 不变量

1. 只有执行器向外部 Git 仓库发起网络请求。
2. 执行器不连接 Cloud 数据库，不跨任务保留仓库、凭据或任务状态。
3. 每个任务都有一份与之逐字段一致的一次性授权；`probe` 授权不能用于推送。
4. 明文凭据不出现在命令参数、环境变量、URL、Git 配置文件、日志、错误信息和返回值中；任务结束后材料目录不存在。
5. 执行器只经服务仓库监听访问 Git Server，每个请求带本次 Operation 的 ID。
6. 同步的快照 ref 和跟踪 ref 在一次原子推送中写入。
7. 发布只推送 Git Server 上源 ref 可达的 `sourceCommit`，只做快进，并以 `expectedOldOid` 作为外部 ref 的比较值。
8. 执行器从不在 `uncertain` 之后重推。
9. 执行器拒绝访问内部地址，不跟随重定向。
10. 执行器不在沙盒网络中。

## 为什么不是这些替代方案

| 方案 | 否决理由 |
| --- | --- |
| 执行器轮询 Cloud 领取任务 | 执行器要么连数据库，要么需要另一套领取接口和身份；Cloud 已经持有租约，直接调用最简单。轮询的优点是 Cloud 不需要知道执行器地址，但单实例部署下这不是约束 |
| 执行器直接写 Git Server 的仓库卷（旧实现的 mirror 发布） | 绕过契约 D3 的授权和 `pre-receive`，违反契约不变量 1 |
| 凭据经环境变量或 `http.extraHeader` 交给 Git | 会进入进程参数或环境，同一用户可读；违反不变量 4 |
| 执行器长期缓存凭据或仓库 | 加快重复同步，但凭据撤销和成员离开就无法立即生效；失陷时泄露面更大 |
| `uncertain` 时重新派发同一推送（旧实现） | 可能在用户不知情时再次发布（D8）；只读核验足以判定 |
| 能力验证与核验复用 `publish` 授权 | 只读的动作获得了写能力，违反最小权限（D2） |
| 部署层出口代理代替执行器的地址检查 | 代理配置错误时就失去保护；执行器本身拒绝内部地址，代理只作附加限制 |
| 把执行器并入 Git Server | 少一个进程，但 Git Server 会同时持有外网出口和外部凭据，与“Git Server 不接触外部凭据”（契约需求 3）冲突 |

## 本决策未解决的问题

- **SSH 地址与 GitHub App 身份**：第一阶段只支持 `https://` 加 token。App 身份需要执行器向平台换取安装 token，留给后续决策。
- **多实例执行器**：第一阶段单实例；多实例需要 Cloud 选择实例和容量协调。
- **LFS 与 submodule**：执行器禁用二者，真正支持见同步 ADR 的未决项。
- **纵深安全加固**（契约“本决策未解决的问题”，第一阶段以跑通与正确为目标）：
  - 出口检查的 DNS rebinding：执行器在连接前解析并检查地址，但 Git 进程自己再解析一次，两次解析之间可能得到不同的地址。
    彻底解决需要把解析结果固定给 Git（例如经执行器内的出口代理），留给后续决策；
  - 大仓库的任务目录：第一阶段是 tmpfs，大仓库会占用大量内存，改用有配额的磁盘卷留给后续决策。

## 落地顺序

1. **基础件**：`cloneauth`、`credentialmaterial`、`git-credential-ora`、`processgroup`、`capture` 参考旧实现摘出或新写，连同测试
   作为新代码单独提交，不接入任何流程；不以旧实现合并为前提。完成条件：测试通过。这一步不依赖底座，可以最先做。
2. **执行器骨架**：`cmd/repo-executor`、mTLS 任务接口、授权校验、凭据兑换、进程与资源边界、出口检查。完成条件：对本地 fixture
   的 `probe` 可以运行；秘密扫描通过。
3. **sync**：依赖底座和服务仓库监听（契约落地顺序第 1、2 步）。完成条件：同步 ADR 的独立验收。
4. **publish 与核验**：完成条件：发布 ADR 的独立验收。
5. **部署**：cluster 的 Compose 加入执行器、Credential Store 和两个网络。

保持不变：沙盒、Node 与 Git Server 之间的一切。从第 3 步开始改变：外部仓库的内容由执行器带入 Git Server。

## 外部线的独立验收

不依赖内部线和 Node，不启动沙盒。

**起栈**：Cloud、Git Server（服务仓库监听与 ref 查询监听）、Credential Store、执行器，以及参照旧部署实现的两个本地 Git 托管
fixture：

- `private-git-fixture`：带 TLS、要 token 才能读；
- `push-git-fixture`：smart HTTP 可写，可切换 `normal`、`refuse`、`moved`、`slow-post`、`drop` 等模式。

fixture 的地址经执行器的允许列表放行。**替身**：底座的预置工具在 Git Server 的 `refs/heads/workspaces/...` 下写一个提交，作为
发布的源；同步的产出用底座的共享契约断言检查（契约落地顺序第 1 步）。

| 场景 | 必须成立 |
| --- | --- |
| 授权与任务不一致（任一绑定字段被改） | 执行器拒绝，不兑换凭据 |
| 同一授权使用两次 | 第二次兑换被 Credential Store 拒绝 |
| `probe` 授权提交发布任务 | 执行器拒绝 |
| 授权过期 | 执行器拒绝 |
| 同步：公开、私有、重放、外部 force-push | 见同步 ADR 的独立验收；执行器的报告与 Git Server 一致 |
| 发布：成功、`conflict`、`rejected`、`not_fast_forward` | 见发布 ADR 的独立验收 |
| `docker pause` 外部 fixture 后发布 | 执行器在超时后报告 `uncertain`；不重推；进程组被回收 |
| `drop` 模式发布 | 推送生效但响应丢失，后置核验确认，报告 `applied` |
| 身份边界 | 执行器用服务证书写 `refs/heads/workspaces/**` 和 `refs/ora/deliveries/**` 被拒；用发布 Operation 的 ID 推送任何 ref 都被拒 |
| lease 与 `+` | 预置工具先把跟踪 ref 写成另一个提交，再让同步以旧的 `expectedTrackingOid` 推送：以 `stale info` 失败、报告 `superseded`，跟踪 ref 不变；Operation 结束后用同一 ID 推送：以 `hook declined` 失败、报告 `failed(git_server_rejected)` |
| 长任务 | 同步一个需要数分钟的大仓库：兑换后授权过期不影响任务，任务期间租约持续续约，同步成功 |
| 出口 | 指向回环、私有地址或内部服务名的仓库地址被拒绝；外部地址 302 到内部地址时不跟随 |
| 超限 | 任务目录超过上限时进程组被杀，报告 `too_large`，没有新的 ref |
| 进程回收 | 任务超时后任务目录与材料目录都不存在，没有遗留的 Git 进程 |
| 秘密扫描 | 参照旧部署实现中的 `log-secret-scan-acceptance.py`：每轮生成 canary token，按字面、base64、base64url（含不补齐）、百分号编码和共享前缀等形式，扫描所有容器日志、Git Server 仓库（包括对象和 reflog）、Cloud 数据库导出、执行器任务目录；先用阳性对照证明扫描器能找到 |

**最终验收**：用 owner 提供的 GitHub 私有测试仓库，以真实 token 完成同步、`probe receive-pack`、发布和一次断网造成的 `uncertain`
核验。
