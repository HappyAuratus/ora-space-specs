# Node 的快照与 bundle 交付

依据[已实现的 Node 交付 Revision 根决策](../../../decisions/node/revision/0-internal-snapshot-and-incremental-bundle-upload.md)。
证据来自 `ora-space/desktop`：`apps/ora-node/src/revision/tests/snapshot.rs` 用真实 Git，`upload.rs` 用本地 HTTP 替身，
`crates/node-db` 测试账本，`apps/ora-node/tests/standalone` 以真实 Node 进程运行。上传授权的协议形状另见
[Agent 运行的协议消息](../protocol/agent-run-messages.md)。

## Snapshot Saves Remaining Changes Without Hooks Or Touching The Checkout

### 风险

项目的提交钩子失败让成果无法保存；或快照改动了用户的工作区、暂存区或分支。

### 前置状态

会话已结束；checkout 上有 Agent 的普通提交，以及已跟踪文件的修改与删除、新文件、被忽略的文件和部分暂存的修改；
仓库配置了会失败的 pre-commit、commit-msg、post-commit 钩子。

### 触发

交付执行准备快照。

### 必须成立

- 有改动时在 HEAD 之后追加一个内部快照提交，收录修改、删除与新文件，不收录被忽略的文件；committer 为
  `Ora <revision@ora.invalid>`，author 为会话的用户身份，父提交为 HEAD。
- 无改动时最终提交就是 HEAD，结果为 `unchanged`，只冻结会话记录。
- 钩子都不运行；工作区、真实 index 与当前分支都不变；临时文件被清理。

### 禁止结果

运行任何提交钩子；工作区或 index 被改写；分支前移；收录忽略文件。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 无钩子快照且不动 checkout | `snapshot.rs::changes_become_a_hookless_snapshot_bundled_against_the_base`：三个钩子都 `exit 1` 且未运行，修改删除与新增被收录、忽略文件不收录，部分暂存的修改、工作区、index 字节与分支不变，committer、author 与父提交符合要求，临时文件已清理 | Covered |
| 无改动为 `unchanged` | `snapshot.rs::unchanged_checkout_freezes_only_the_history` | Covered |
| 子模块、LFS 与 clean 过滤器的边界 | 尚无测试 | Missing |

### 决策依据

Node revision D2，不变量 1、3、6。

## Delivery Bundles Only Against The Clone Base

### 风险

bundle 依赖更早的 bundle，任何一个过期都破坏恢复；或交付了另一个 checkout、另一个基础版本的成果。

### 前置状态

会话的 clone 执行记录了基础提交。

### 触发

交付执行生成 bundle。

### 必须成立

- bundle 为 `<revision_ref> ^<base_commit>`，上传前通过 `git bundle verify`，能在只有基础提交的新仓库中取出最终提交。
- checkout 不属于该会话、缺少基础提交或不是仓库时为 `checkout_unavailable`；会话记录不可读时为
  `history_unavailable`；两种失败都不留下冻结对象。

### 禁止结果

bundle 相对其他提交生成；未校验就上传；在错误的 checkout 上交付。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 相对基础提交且可恢复 | `snapshot.rs::changes_become_a_hookless_snapshot_bundled_against_the_base`：bundle 在只含基础提交的新 clone 中 `fetch` 出最终提交 | Covered |
| 输入不可用时失败且不冻结 | `snapshot.rs::unusable_inputs_fail_without_frozen_objects`：缺少基础提交或不是仓库为 `checkout_unavailable`，缺少会话记录为 `history_unavailable`，都没有冻结对象。缺口：checkout 不属于该会话、clone 提交与输入的基础提交不符没有 Node 服务层测试 | Partial |
| 冻结的声明只能是输入的键、ref 与基础提交 | `crates/node-db/src/tests/revision.rs::freezing_requires_the_inputs_keys_ref_and_base` | Covered |

### 决策依据

Node revision D3、D5，不变量 4。

## Delivery Waits For A Settled Session

### 风险

Agent 或其子进程仍在修改文件时做快照，成果不完整。

### 前置状态

交付引用的会话尚无终态，或已有终态但会话 actor 仍在运行。

### 触发

交付执行开始。

### 必须成立

- 会话不存在或尚无终态时为 `failed{session_not_settled}`。
- 会话已有终态但 actor 仍持有会话记录时等待，不失败。

### 禁止结果

在会话结束前开始快照；把仍在退出的会话当作失败。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 未结束的会话不交付 | Node 服务层没有直接测试；`crates/node-db` 与 Controller 映射只证明失败码可以记录和转交 | Missing |
| actor 未退出时等待 | 尚无测试 | Missing |

### 决策依据

Node revision D1，不变量 2。

## Frozen Delivery Bytes Survive Restarts And Grants Stay In Memory

### 风险

部分上传后重启重新生成快照，同一执行的对象内容与声明不一致；或签名 URL 被写入账本。

### 前置状态

交付已冻结对象字节与声明，至少一个对象已上传。

### 触发

Node 重启；授权过期或返回 403；上传持续失败；冻结文件被改动。

### 必须成立

- 冻结之前的执行重新准备，并丢弃上一次的残留；冻结之后重启复用同一份字节与声明。
- 冻结文件与声明不一致时失败，不上传。
- 请求原样携带签名头（包括 `x-amz-checksum-sha256` 与 `If-None-Match: *`），412 视为已创建；过期或 403 的授权
  被丢弃并以冻结的摘要重新请求；不是本执行的授权被忽略。
- 每个对象最多 3 次 PUT，仍失败则 `upload_failed`。
- 授权与签名 URL 不进入账本。

### 禁止结果

重启后重新快照或重新打包；修改过的字节被上传；授权被持久化。

### 验证义务与证据

| 义务 | 所需直接证据 | 状态 |
|---|---|---|
| 冻结前重做、冻结后复用 | `snapshot.rs::preparing_again_discards_an_abandoned_attempt`；`upload.rs::restart_after_partial_upload_reuses_frozen_bytes`；`crates/node-db/src/tests/revision.rs::delivery_freezes_completes_replays_and_acknowledges` | Covered |
| 改动的字节不上传 | `upload.rs::changed_frozen_bytes_fail_without_uploading` | Covered |
| 签名头、412 与换授权 | `upload.rs::grant_requests_carry_frozen_checksums_and_headers_travel_verbatim`、`::forbidden_grant_is_replaced`、`::foreign_grants_are_ignored` | Covered |
| 上传次数有界 | `upload.rs::three_failed_attempts_fail_the_upload` | Covered |
| 授权不落盘 | `apps/ora-node/tests/standalone/agent_sessions/delivery.rs::ended_session_is_delivered_with_requested_grants`：真实 Node 进程交付后账本中没有签名 URL；Node 日志不含授权尚无直接断言 | Partial |

### 决策依据

Node revision D4、D5，不变量 5、7。
