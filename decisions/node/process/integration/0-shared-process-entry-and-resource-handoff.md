---
status: approved
---

# Node 的 GitRunner 统一受管 Git，恢复先关闭旧尝试再对账

当前 Linux 独立 Node 的 Git 命令经 ManagedGitRunner → process-client → host → guardian 执行。
gitlancer 仅构造命令与解析结果，不持有 Node／process 概念。本文收紧至 Node Worktree 切片；
插件、现有 Backend 写入切换、Controller IPC 和跨 Node 资源协调由未来 ADR 负责。

## 决策

### D1：唯一执行入口，不把本地句柄当作远程所有权

production Node::open 显式注入数据目录、host 目录、Git 程序、环境及停止参数。
所有 Git 调用走 host；仅明确注入依赖的测试／嵌入入口可使用直接 runner。
取消请求或 Drop 不隐式关闭远程 Scope；正常停止显式请求收尾并检查结果。

### D2：变更尝试先持久关联，查询也受管

业务 Execution 先接受，变更 Run 的原 host／Scope／Run 关联先写入 Node DB，再派发。
每次变更独立 Scope；通信失败不换身份重跑。只读 preflight 尚无业务 Execution，
共用有限数量 Run 的只读 Scope，host 仍持久保存其 Run 意图；不虚构业务关联。

### D3：恢复交接不只看 Node 数据库锁

恢复先按记录关闭原 Scope 并确认观测到 Closed，随后才观察和修复资源。
仅发送 kill、owner 退出、拿到 DB 锁或单 Run 退出均不替代该交接。
legacy 在途执行没有进程关联时保持阻塞，不凭升级证明旧 Git 已停止。

当前 Linux BestEffort 在承诺的清理完成及业务对账后允许继续，但不能排除未跟踪逃逸后代。
guardian 丢失时缺少收尾证据，保留 Unknown 和冲突资源预留，不阻塞不冲突工作。
正常停止给当前 Git 有限宽限，超时请求收尾；Node 强杀由 guardian 独立检测 owner 身份退出。

### D4：业务归属仍由 Node 判断

Node 保留原 Execution、冻结目标和 base_commit；分支／目录残留按 Worktree 规则处理。
进程层不解释仓库成功或资源冲突。本地清理不证明 hook 委托的外部服务已经停止。
当前保护已登记身份、分支和重叠路径；同仓库不同 worktree 共享元数据的完整冲突政策仍需确认，
不能把路径不同直接提升为所有 Git 操作互不冲突的承诺。

## 不变量

1. 变更尝试关联先于派发，原身份重传不产生新业务尝试。
2. 收尾证据和业务对账分别负责，未知不释放冲突资源。
3. Git 语义归 gitlancer／Node，运行层不承担业务解释。

## 取舍与未来 ADR

仅替换插件 reaper 无法解决 Git 恢复交错；把全部业务塞入 process 会破坏领域归属。
插件持续 I/O 与会话接管、公共 Session／Scope／Run 高层句柄、只读角色、Backend 切换、
Controller 投递以及跨 Node 协调由后续对应领域 ADR 定义，不以本切片宣称完成。

## 验收与状态

真实 Node 强杀重启、guardian 丢失、不冲突任务继续、正常宽限与超时，以及持久关联故障测试
已通过。见[核心用例](../../../../test-cases/node/process/integration/business-entry-and-resource-handoff.md)。
共享仓库冲突边界待 Eric 确认，本文保持 approved，不靠收窄措辞消除这一安全问题。
