# Session 核心测试用例

本目录镜像 session 决策领域；会话根决策的验证义务见
[会话身份、绑定历史与统一诊断](identity-bindings-and-diagnostics.md)。

| 用例文件 | 风险面 |
| --- | --- |
| [identity-bindings-and-diagnostics.md](identity-bindings-and-diagnostics.md) | 切换与重复绑定、并发读取、历史降级、未创建状态、重试与循环归属、批量隔离、统一入口及暴露边界 |

子领域 MCP 的验证义务保留在 [mcp](mcp/) 下。子领域 skill 镜像 [Skill 会话清单决策](../../../../../decisions/desktop/core/agent/session/skill/0-skill-catalog-from-storage-paths.md)；该决策仍为 `proposed`，核心用例在其 `approved` 后补充到 `skill/` 下。
