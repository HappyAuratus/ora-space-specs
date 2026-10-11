# M4 OpenCode：受控模型证据与检视修复边界

[English](opencode-m4.en.md) | 中文

以下保留 2026-10-10 的 `Cloud 5b6b5ec` / `Desktop f8536f9d` / `Cluster a8146c7` 历史成功记录；它不能证明随后检视指出的失败路径和项目配置隔离正确。固定官方 `official/ora-space.opencode@0.6.4` 的生产 CLI 曾通过平台代理读取 LICENSE、写文件并提交 Git。受控上游只替代模型服务，不代表真实供应商验收。

## 2026-10-11 检视复核

最新意见确认：系统 DNS 的保留地址答案被正确拒绝，但缺少可信解析路径；IPv6 基准网段遗漏；代理传输错误缺分类；网络 clone 共用短命令预算且失败重试无上限；项目 `.opencode` 会被 CLI 自动发现；ACP 错误被记为完成，页面可能继续等待或空闲。HOME 与公开 CA 在后续提交已修正，仍需新会话直接复验。

已按意见修正可信 HTTPS DNS、公网答案整体检查、安全错误分类、网络 Git 独立预算、持久三次 clone 重试，以及会话失败的关闭与持久安全条目。失败时关闭消息通道并释放授权，尝试交付已有 Revision；成功多轮行为保留。已提交的用户取消优先于竞态中的模型失败。禁止项目配置发现，不修改仓库已有 `.opencode` 内容。

逐意见的复现、验证、提交和原线程回复见 [检视处理记录](https://github.com/ora-space/cluster/blob/codex/m4-opencode-model-proxy/docs/m4-review-follow-up.md)。一次临时 hosts 映射后的 OpenAI 回复不能替代正常 DNS、Anthropic、错误页面、资源释放或新 Key 的完整验证。整体仍为 Partial，真实供应商义务仍为 Missing。

## 2026-10-11 检视修复的直接证据

独立 `ora-m4-review` 栈使用本轮 Cloud / Node 代码、固定官方 0.6.4 包和受控 HTTPS 模型服务。
仓库 fixture 已提交 `.opencode/package.json` 与哨兵文件；不靠删除用户文件测试隔离。
`browser-final.json` 与同 Run 的 `verified-browser.json` 均 passed：开发登录、私有连接配置、clone、
两轮读写、结束、Revision 恢复及容器/卷释放。恢复后相对基线只有 `ora-m4-result.txt`，原有
`.opencode` 内容不变，bundle 与 Node JSONL 大小/摘要匹配。

| 协议 | 本轮 Run | Revision | 最终 Git 提交 |
| --- | --- | --- | --- |
| `openai-completions` | `3890efc9-63c2-48ff-bb04-c7efec7efb19` | `67233399-9f15-4248-8c9a-99c2c4cd8f3d` | `3348203ef5c71d58d0153bff14bf0b07f4fb23da` |
| `anthropic-messages` | `92efb86b-df38-40ff-abe7-c13a33f30aeb` | `f59f0c28-ce88-48d5-8b68-d17f8be48227` | `6e9ac4d9b39b593662cd1cb8eb865408cf79771f` |

`failure-browser.json` / `verified-failure.json` 均 passed。受控供应商拒绝合成测试凭据，HTTP 401
经 Gateway 安全 502 和真实 CLI ACP 错误到达 Node；页面显示安全失败条目并禁止追加，Thread ended、
运行 failed，已有 unchanged Revision 登记。OpenAI Run `022547f5-d58f-407f-b870-a3fc4563c90e`、
Anthropic Run `09c01527-8ba4-4dea-b680-b830a60f6f14` 的授权归零，Node JSONL 摘要相同，运行容器和卷已释放。
未知 detail 的安全收敛、重放不重复、持久取消优先另由真实 PostgreSQL/gRPC 和 Node 插件测试证明。

真实 Controller/Node clone 故障操作 `e56b5445-d95f-4121-b369-17c641df652b` 恰好三次持久派发后
failed、无 retryAt、受理关闭，手动重试返回 409，主 Workspace 保留。真实 Node 延迟 clone 测试超过
本地五秒预算后仍成功；网络预算超时继续经过 guardian 清理屏障，Unknown 不自动重试。

独立 Echo 栈的 `echo-browser.json` 与 `echo-resume-final.json` passed，保留 M2 多轮/结束及 M3
三次运行的首次成果、恢复后 unchanged 和再写入路径。对象摘要、Git 作者及资源释放均复核。
Windows/WSL 的无凭据 HTTPS DNS 探测正常验证真实上游证书后得到 HTTP 401；它只证明公网连接，
不能计为真实模型生成。本轮未读取 Pi Key，Bluezone 新 Key 两协议验证仍 Missing。

首次冷启动在并行重负载下超过既有 30 秒 ready 预算，留下 AgentUnavailable；后续全新会话才通过。
没有通过增大 ready 预算、关闭证书验证或临时 hosts 映射获得通过。未开始会话空 JSONL 的完整交付
仍 Partial，完整进程后代纳管继续由 Desktop #665 跟踪。本轮检查与提交状态以检视处理记录为准。

## 2026-10-10 历史直接证据

证据位于 cluster 的忽略目录 `.acceptance/`；本记录仅列安全状态与资源标识，不复制密钥、Cookie 或令牌。页面证据和宿主复核必须是同一 Run，API 只作独立补充。

| 入口 / 协议 | Run | Revision | 最终 Git 提交 |
| --- | --- | --- | --- |
| Browser OpenAI / `openai-completions` | `ff53d668-4fde-4105-b903-61be7acf1f11` | `fcf6d81f-602e-4d87-9292-f39816555ad3` | `096fe77e4a3cb2262abecb3153bbd8b131c43a38` |
| Browser Anthropic / `anthropic-messages` | `e8617fbf-33e5-4f67-af41-d84f8e998478` | `cd3306ae-dc9e-4151-a604-1ed0601ec2b9` | `233e85021b43d411ad35e19b72833ff18950574d` |
| API / `openai-completions` | `72de9f9e-0289-4610-8ec4-1a85382c8973` | `e926d01a-f3d5-4515-8108-bc618811e7e7` | `bf1876c542e724ba8e84e5035311f07bf6ffcf7e` |
| API / `anthropic-messages` | `c428cd73-4eee-4fe3-ab92-25ad9c80f734` | `f7df02d0-b086-4455-a9a7-3b980c7951d9` | `8cae721e2b77e1918860396f08875e3493249744` |

页面证据为 `opencode-m4-receipt-final-browser-openai.json`、`opencode-m4-receipt-final-browser-anthropic.json`，同资源复核为其 `verified-*` 文件；API 为 `opencode-m4-api-receipt-final.json`，均 passed。模型 `m4/tool-model` 的斜线原样保留。两轮均有非空回复并进入 idle，结束后 Thread ended、公开运行 completed；核对 bundle/JSONL 大小和摘要、恢复文件、用户 Git 作者、无检测到的凭据材料，以及运行容器与卷实际释放。两个结束页截图已人工检查。

登录证据属于开发登录，不宣称外部身份提供方的生产登录验收。最终现场工作进程 UID/GID 1000、无附加组/权限能力、NoNewPrivs，不能读取 Node 私钥及私有目录；guardian 完整进程树纳管仍独立跟踪，不宣称多人生产执行隔离完成。

## 合并兼容与证据边界

同一会话可以同时保留个人模型绑定与前序 Revision，已发布恢复字段编号保持 6，模型引用使用 7。Controller 两种能力分别检查；新增通信往返和缺任一能力的组合回归通过。两个 `0033` 迁移保留完整文件名与原始 SQL，不改写已应用账本。以下为 2026-10-10 历史 CI：Cloud 472 项完整并发、前端 945 项及 Desktop 三系统/前端/桌面框架/端到端全绿，不作为本轮提交 CI；详见 [Cluster 记录](https://github.com/wanglongan587/cluster/blob/codex/m4-opencode-model-proxy/docs/opencode-m4-acceptance.md)。

| 验证义务 | 直接证据 | 状态 |
| --- | --- | --- |
| 两协议页面配置、clone、Issue、多轮与结束 | 两组最终浏览器记录，真实生产 CLI、受控上游 | Covered |
| 页面同一 Run 的成果恢复、Git 作者、容器/卷释放 | 两组 `verified-*` 最终记录 | Covered |
| API 两协议及含斜线模型标识正常链路 | `opencode-m4-api-receipt-final.json` | Covered |
| Bluezone 页面新 Key 的两协议真实模型 | 用户尚未配置，受控成功不可推定实际服务 | Missing |
| 多身份浏览器权限、并发、停用与撤销 | 下层规则测试已有；页面只覆盖发起者正常路径 | Partial |
| 全栈授权失效、进程崩溃和恢复 | 下层范围、续期、撤销及取消有测试；正常路径不证明故障演练 | Partial |
| 主密钥灾备恢复 | 加密/启动拒绝有测试，完整恢复未演练 | Partial |
| 未开始会话空 JSONL 完整交付和释放 | Node 收尾有测试，完整交付无直接证据 | Partial |

M4 总体保持 Partial，相关个人模型 ADR 保持 approved；用户经页面新 Key 验证两协议实际模型后才可记录完成。历史成功、中断恢复、启动失败及单次累计超时记录保留，不代替最终版本验证。
