# M4 OpenCode：最终受控模型验收

[English](opencode-m4.en.md) | 中文

2026-10-10，`Cloud 5b6b5ec` / `Desktop f8536f9d` / `Cluster a8146c7` 完成两协议 API、完整 Chromium 页面与同资源成果核验。固定官方 `official/ora-space.opencode@0.6.4` 的生产 CLI 通过平台代理读取 LICENSE、写文件并提交 Git。受控上游只替代模型服务；这不是 Echo 或模拟 Agent。

## 最终直接证据

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

同一会话可以同时保留个人模型绑定与前序 Revision，已发布恢复字段编号保持 6，模型引用使用 7。Controller 两种能力分别检查；新增通信往返和缺任一能力的组合回归通过。两个 `0033` 迁移保留完整文件名与原始 SQL，不改写已应用账本。Cloud 最新 472 项完整并发 CI、前端 945 项及 Desktop 三系统/前端/桌面框架/端到端 CI 全绿；详见 [Cluster 记录](https://github.com/wanglongan587/cluster/blob/codex/m4-opencode-model-proxy/docs/opencode-m4-acceptance.md)。

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
