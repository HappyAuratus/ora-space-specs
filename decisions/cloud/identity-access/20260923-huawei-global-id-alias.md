---
status: implemented
---

# IDaaS UUID 继续作为登录键，globalUserId 作为人员目录关联键

本决策继承 `0-gateway-mediated-external-identity.md` 的 Gateway 验证、Cloud 授权、双重短期凭据和外部 token 不持久化原则；修改「忽略 globalUserID」的约定。IDaaS `uuid` 与 `globalUserId` 均稳定且一一对应，工号可能变化。双标识关联和目录添加已落地，冲突按本决策拒绝自动合并。

## 继承与修改

| 前序约定 | 本次处理 | 当前规则 |
| --- | --- | --- |
| Gateway 验证外部身份，Cloud 持有授权 | 继承，不变 | 浏览器不能自选人员身份或租户权限 |
| `huawei-corp` 的登录 subject 是 IDaaS `uuid` | 继承，不变 | 工号、邮箱和姓名都不替代 `uuid` |
| 外部 token 不进入 Cloud 凭据或业务表 | 继承，不变 | 只传经验证的稳定身份键 |
| 忽略 userinfo 的 `globalUserId` | 修改 | 将其映射为同一用户的天舟关联身份，冲突拒绝合并 |

## 为什么不是这些替代方案

| 方案 | 取舍 |
| --- | --- |
| 用工号关联预添加成员 | 易于展示，但工号变化会断开或错误转移授权 |
| 用 `globalUserId` 直接替换已有登录 subject | 可统一目录键，却会重新解释已持久化的 `uuid` 身份 |
| 遇到两标识冲突时自动合并用户 | 减少人工处理，但可能把两个既有租户成员身份和资源归属合并为一人 |

## D1：可信双标识

Gateway 从同一次已验证的 IDaaS userinfo 读取 `uuid` 与 `globalUserId`，把后者按十进制字符串传入浏览器会话及签名的内部用户凭据。Cloud 将两者作为同一用户的不同来源身份；已存在且指向不同用户时拒绝自动合并，不依据姓名、工号或邮箱找回身份。GitHub 身份保持独立。

## D2：目录人员加入

Cloud 服务端以部署密钥调用天舟搜索，前端不直连跨源接口。管理员搜索结果只展示姓名、工号和部门；添加前再次向天舟核验结果与在职状态，只保存稳定身份键及显示名。可先创建以 globalUserId 标识的用户，首次 IDaaS 登录原子补上 uuid 绑定。目录失败不产生任何成员写入。

## 不变量与验收

1. 工号变更不改变用户、成员和资源归属。
2. 身份键冲突不能自动合并或转移权限。
3. 天舟请求头、IDaaS 原始 userinfo、手机号和邮箱不进入日志、公开响应或持久业务数据。

核心验证义务与证据位于 `test-cases/cloud/identity-access/huawei-person-binding.md`。第一版不做人事状态后台同步；已登录人员的后续离职准入依赖 IDaaS 再次登录校验与管理员停用成员。原有 `uuid` 身份不重写，未登录人员只按稳定目录键预添加。
