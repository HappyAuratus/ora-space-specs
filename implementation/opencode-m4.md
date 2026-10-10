# M4 OpenCode：受控上游的浏览器与 API 验收记录

[English](opencode-m4.en.md) | 中文

2026-10-10，独立 M4 Compose 栈完成 `openai-completions` 与 `anthropic-messages` 的真实浏览器和 API 正常链路。
固定官方 `official/ora-space.opencode@0.6.4` 的生产 CLI 通过平台代理访问受控模型服务，读写仓库并交付 Revision。
所有下表证据为 `mode=fixture`、`status=passed`，命令退出码均为 0；这证明实际产品与执行组件已接通。
用户尚未在页面填写新的 Bluezone Key，M4 整体验收仍为 `Partial`，相关 ADR 保持 `approved`。

决策与核心义务见[个人模型访问](../test-cases/cloud/model-access/personal-model-access.md)、
[Thread 输入权限](../test-cases/cloud/thread/personal-model-input.md)、
[Node 模型会话](../test-cases/node/agent-runtime/model-proxy.md)和
[模型访问身份及部署](../test-cases/cluster/sandbox-server/model-client-identity.md)。

## 直接证据

文件保存在 cluster 的 `.acceptance/`，本记录只收录安全状态、协议及资源标识，不复制密钥、响应正文或截图内容。
浏览器证据与后续复核必须指向同一个 Run；API 验收是独立补充，不能替代页面操作。

| 入口与协议 | 浏览器记录及精确资源复核 | Run | Revision |
|---|---|---|---|
| 浏览器 `openai-completions` | `opencode-m4-browser.json`、`opencode-m4-browser-verification.json` | `e464a25d-f31f-4770-ada9-b00f32b0ad34` | `69dc202d-1bb2-472f-9672-04f4d0deb6aa` |
| 浏览器 `anthropic-messages` | `opencode-m4-browser-anthropic.json`、`verified-opencode-m4-browser-anthropic.json` | `c696ad9b-3159-4cdc-a3c1-f99d8731ffa3` | `3204dcbb-1c64-40d6-9672-975374bf2b9c` |
| API `openai-completions` | `opencode-m4-final.json` 的对应 scenario | `81c988f3-9ea6-48cb-883b-5e2b0c160bcb` | `365d9a75-de7e-4630-a28f-c49034a0ecfb` |
| API `anthropic-messages` | `opencode-m4-final.json` 的对应 scenario | `c4f74e77-b648-4eeb-b53f-a685f6b9e65b` | `61da1046-390d-4a80-a3b4-4d7e27ce4300` |

两种协议的记录均保留模型 ID `m4/tool-model`，其中斜线原样经过个人默认值、运行绑定和真实 CLI 请求。
浏览器分别留下 10 张与 9 张检查点截图；截图数量是追溯元数据，不替代断言或对象核验。

## 已验证的正常链路

浏览器执行开发登录、创建协作空间、填写模型连接与默认模型、保存 Git 作者身份、选择官方插件、
创建项目并 clone GitHub 的 `main` 分支、创建关联 Issue、`@OpenCode`、追加一轮消息及结束会话。
模型设置保存后密码输入清空；两轮均有非空回复并进入 idle，结束后页面显示已保存的有改动 Revision，公开运行 completed。
这里的登录证据属于开发登录，不额外声明外部身份提供方的生产登录验收。

后续复核读取上述浏览器创建的精确 Run，检查 bundle 与 JSONL 对象摘要、恢复结果文件、
Git 作者匹配、产物未包含检测到的凭据材料，以及运行 `done`、运行 Workspace 容器与卷均释放。
浏览器的两个运行 Workspace 分别为 `e83c5481-a6d0-493e-bc5f-49339b989acc` 和
`e84ce9d5-0d2b-4c92-85e8-fcdc53084a21`；复核不是另建一轮运行取得成功结果。

API 最终验收对两种协议分别完成生产 CLI 的仓库工具调用、多轮回复、结束、Revision、Git 身份及清理核验。
受控上游替代的是模型服务；执行端使用固定官方 CLI，不能把此证据归为 Echo 或模拟 Agent。
验收可使用正常 CA 校验的 HTTPS 缓存提供与官方 SHA-256 完全相同的原始包，不修改包内容或摘要。

## 证据状态与剩余验收

| 验证义务 | 当前直接证据 | 状态 |
|---|---|---|
| 两种协议的页面配置、clone、Issue、多轮对话与结束 | 两组真实浏览器记录，固定生产 CLI、受控上游 | Covered |
| 浏览器产生的 Revision、Git 作者、成果恢复与容器/卷释放 | 同一 Run 的两组精确资源复核 | Covered |
| API 两协议与带斜线模型 ID 的正常链路 | `opencode-m4-final.json`，模型 ID 为 `m4/tool-model` | Covered |
| 使用页面新填 Bluezone Key 的两种协议真实服务链路 | 尚未执行，不能由 fixture 的成功推定 | Missing |
| 多身份浏览器权限、并发、停用与撤销 | Core/HTTP/前端规则有测试；此次浏览器记录仅覆盖发起者正常链路 | Partial |
| 全栈授权失效、进程崩溃及恢复 | 下层范围、撤销、续期与取消有测试；此次正常路径不证明故障恢复 | Partial |
| 主密钥卷灾备恢复 | 加密身份及启动拒绝测试已有证据；完整恢复演练未执行 | Partial |
| 首次授权已 ending 时的空 JSONL Revision 上传、登记及释放 | Node 结束协调与空记录封存有测试；完整交付链路尚无直接证据 | Partial |

只有新的 Bluezone Key 经页面配置并完成两种协议的真实服务验收后，才能记录 M4 完成。
agent 进程降权与 guardian 纳管仍单独跟踪；本记录不宣称多人生产执行隔离已完成。
