---
name: feishu-chat-group-management
description: "飞书群聊管理 — 创建群聊、配置Bot、管理权限、多群平行对话架构"
version: 3.0.0
triggers:
  - user asks to create a Feishu group chat / 创建飞书群聊
  - user wants to set up parallel conversations / 设立平行对话通道
  - user asks to add bot to groups / 把Bot加到群
  - user wants multi-group architecture / 多群对话架构
  - user mentions FEISHU_GROUP_POLICY or im:chat permissions
  - user wants to rename groups / 重命名/改群名
  - user asks about group structure or department organization / 群组织架构/职能部门化
  - user asks about cross-group information sharing / 跨群信息同步
  - user mentions CEO-level orchestration or system-wide decisions / 系统整体决策
  - user asks about daily briefing or system pulse / 系统脉搏/每日简报
---

# 飞书群聊管理技能

## 概述

管理 Hermes Bot 在飞书 IM 中的群聊 — 创建群组、处理权限、配置群政策、建立多群平行对话架构。

## 核心信息

| 项目 | 值 |
|------|-----|
| **用户 open_id** | `ou_50b21c92548fbb2173b049e57dfbdbec` |
| **群政策** | `FEISHU_GROUP_POLICY=open`（无需 @ 即可对话） |
| **连接模式** | `FEISHU_CONNECTION_MODE=websocket` |
| **Token 注册表** | `~/.hermes/feishu-tokens.json` |
| **配置位置** | `~/.hermes/.env`（FEISHU_APP_ID / FEISHU_APP_SECRET） |

## 操作流程

### Step 1: 获取 Tenant Access Token

```python
import requests, os

def get_tenant_token():
    app_id = os.environ.get("FEISHU_APP_ID")
    app_secret = os.environ.get("FEISHU_APP_SECRET")
    # 如果环境变量没加载，从 .env 文件读取
    if not app_id:
        with open(os.path.expanduser("~/.hermes/.env"), "r") as f:
            for line in f:
                if line.startswith("FEISHU_APP_ID="):
                    app_id = line.split("=", 1)[1].strip()
                elif line.startswith("FEISHU_APP_SECRET="):
                    app_secret = line.split("=", 1)[1].strip()
    resp = requests.post(
        "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
        json={"app_id": app_id, "app_secret": app_secret},
        timeout=10
    )
    return resp.json().get("tenant_access_token")
```

### Step 2: 检查权限范围

Bot 需要有 `im:chat` 权限才能创建群聊。如果缺少权限，API 返回：
```json
{
  "code": 99991672,
  "msg": "Access denied. One of the following scopes is required: [im:chat, im:chat:create, im:chat:create_by_user]"
}
```

**解决方式一：用户手动开通（最快）**
提供以下链接让用户去飞书开放平台开通：
`https://open.feishu.cn/app/{APP_ID}/auth?q=im:chat,im:chat:create,im:chat:create_by_user&op_from=openapi&token_type=tenant`

**解决方式二：自动建群（需要提前开通）**
开通后无需重启 Gateway，新权限立即生效。

### Step 3: 创建群聊

**API**: `POST https://open.feishu.cn/open-apis/im/v1/chats`

```python
BASE = "https://open.feishu.cn/open-apis"

resp = requests.post(
    f"{BASE}/im/v1/chats",
    headers={"Authorization": f"Bearer {token}", "Content-Type": "application/json"},
    json={
        "name": "📊 群聊名称",
        "description": "群聊描述",
        "user_id_list": ["ou_50b21c92548fbb2173b049e57dfbdbec"],  # 初始成员
        "setting": {
            "membership_approval": "no_approval_required",
            "join_message_visibility": "only_owner",
            "leave_message_visibility": "only_owner",
        }
    },
    timeout=15
)
data = resp.json()
chat_id = data.get("data", {}).get("chat_id", "")
```

**注意**：
- `user_id_list` 使用 open_id 格式
- Bot 会自动加入群聊（由 API 创建者身份决定）
- 创建后无需额外操作，WebSocket 连接会自动接收该群的消息

### Step 4: 注册到 Token 注册表

将群信息写入 `~/.hermes/feishu-tokens.json`：

```python
import json
path = os.path.expanduser("~/.hermes/feishu-tokens.json")
with open(path, "r") as f:
    data = json.load(f)

data["chat_groups"] = {
    "_meta": {
        "description": "Feishu group chat IDs for Hermes parallel conversations",
        "created": "YYYY-MM-DD",
        "group_policy": "open (no @mention required)"
    },
    "groups": [
        {
            "name": "群名称",
            "chat_id": "oc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
            "purpose": "用途描述"
        }
    ]
}
with open(path, "w") as f:
    json.dump(data, f, indent=2, ensure_ascii=False)
```

### Step 5: 验证群状态

```python
resp = requests.get(
    f"{BASE}/im/v1/chats/{chat_id}",
    headers={"Authorization": f"Bearer {token}"},
    timeout=10
)
data = resp.json()
if data.get("code") == 0:
    print(f"✅ 群聊 {data['data']['name']} 状态正常，Bot 已加入")
```

## 参考文件

- `references/existing-groups.md` — 已建群聊清单（chat_id、用途、操作记录）
- `references/at-mention-gate-code-fix-20260512.md` — @提及门控逻辑修复记录（Gateway 代码修改详情）
- `references/department-role-design.md` — 部门负责人角色设计、决策层级、联动关系（v1.0 2026-05-15）
- `references/docx-api-quirks-session-20260516.md` — docx API 调试记录
- `references/chinese-inline-string-workaround.md` — 中文内联字符串的语法规避方案（Python inline 代码块的 SyntaxError 陷阱）

## 🏢 职能部门化架构（v3.0 — 2026-05-15）

### 核心理念

7个群不再是平行的「功能节点」，而是升级为**组织架构中的职能部门**。用户（夜雨=CEO）在此DM做全局决策，各群作为「部门」各司其职。

```
     ┌────── CEO (此DM) ──────┐
     │   🧠 系统脉搏 每日17:15 │
     └────────┬───────────────┘
              │
    ┌─────────┼─────────┬──────────┐
    │         │         │          │
 📊战略   📋会议   ✍️内容    💡战略
 材料部    情报部    市场部    研究部
    │         │         │          │
    └─────────┼─────────┼──────────┘
              │         │
           🎓教育    📬对外
           产品部    联络部
              │
           🗓️总裁
           办公室
```

### 群重命名：群名 → 职能部门名

**变更日期：** 2026-05-15
**API 注意事项：** 使用 `PUT /open-apis/im/v1/chats/{chat_id}`（非PATCH——PATCH会返回404）
**一次性全改7个**，同步更新 feishu-tokens.json 和参考文档。

### 部门负责人角色设计（升级版）

每个群不仅是一个「对话通道」，还设定了**部门负责人角色**——这个角色负责判断哪些信息是「常规执行」（无需CEO知晓）、哪些是「需要CEO知晓」（记录到部门日志）、哪些是「需要CEO决策」（主动推送至DM）。

| 群 | 部门负责人角色 | 汇报什么（只汇报CEO需要知道的） |
|:--|:--------------|:-------------------------------|
| 📊战略材料部 | 🎯首席策略顾问 | 新项目启动、材料结构变更、需要协调其他部门的数据/资源 |
| 📋会议情报部 | 📋会议调度官 | 关键会议结论、跨部门待办、新增重要联系人 |
| ✍️内容市场部 | ✍️主编 | 选题方向、发布计划、需要CEO审阅的稿件 |
| 💡战略研究部 | 🎯首席策略顾问+🧠认知管理 | 有价值的洞察、可落地的灵感、值得KB沉淀的知识 |
| 🎓教育产品部 | 🎓教育产品总监 | 模块改动范围、用户反馈的痛点、需要CEO拍板的决策 |
| 📬对外联络部 | ✉️联络主任 | 新增重要联系人、需要CEO介入的邮件、风格偏好更新 |
| 🗓️总裁办公室 | 🗓️总办主任 | 待办健康度、作息异常、提醒本周未处理的重要事项 |

**汇报层级过滤：**
```
群内发生事件
    ↓
部门负责人判断
    ├─ 🟢 常规执行 → 不汇报，自己处理
    ├─ 🟡 需要CEO知晓 → 记录到「部门日志」
    └─ 🔴 需要CEO决策/协调 → 主动推送至CEO DM
```

### CEO DM定时简报（系统脉搏）

**cron job_id:** da23b6d6c4ef
**时间:** 每个工作日 17:15
**内容:** 自动检索7部门当天动态，汇总输出精炼简报投递至CEO DM
**格式:**
```
━━━ 🧠 系统脉搏 5/15 ━━━
📊 战略材料部：[一句话]
📋 会议情报部：[一句话]
...（各群1行）
```

### 跨群信息桥

**问题：** 每个群是独立会话上下文，用户在A群分享的信息默认不可见。
**三层解决方案：**

| 层 | 机制 | 说明 |
|:---|:------|:-----|
| L1 🔍 | session_search | 全局决策前自动检索所有群近24h动态 |
| L2 📬 | 系统脉搏cron | 每日17:15自动汇总7部门动态至CEO DM |
| L3 💾 | memory持久化 | 跨群关键节点主动存档（偏好、架构决策、跨群关联） |

## @提及门控修复要点

`FEISHU_GROUP_POLICY=open` 仅控制"哪些用户可以触发"，不控制"是否触发响应"——Gateway 还有一个独立的 `_should_accept_group_message()` 方法执行 @mention 检查。修复细节见参考文件。

当用户问"群聊是否需要@机器人"时，**必须先查 Gateway 代码**，不要只凭环境变量值回答。

## 多群平行对话架构

### 关键配置

| 环境变量 | 说明 |
|----------|------|
| `FEISHU_GROUP_POLICY=open` | 不需要 @ 即可在群内对话 |
| `FEISHU_ALLOW_ALL_USERS=true` | 所有用户均可与 Bot 交互 |
| `FEISHU_CONNECTION_MODE=websocket` | WebSocket 实时接收事件 |

### 架构要点

- 每个群聊是 Hermes **独立会话上下文**，消息不会串
- 同一 Bot 可以加入多个群，Gateway 自动按 chat_id 路由
- 群聊与 DM 是等价的会话容器，共享同一个能力系统
- 任务类型识别：由用户在群内的第一条消息内容决定加载什么技能

### 命名规范

推荐使用 Emoji 前缀一目了然：
```
| 📊 战略材料部    → material-production
| 📋 会议情报部         → meeting-minutes
| ✍️ 内容市场部         → content-factory
| 💡 战略研究部     → daily-conversation
| 🎓 教育产品部     → edu-hub
| 📬 对外联络部      → email-assistance
| 🗓️ 总裁办公室      → daily-operations
```

## 群感知角色预设注册表（v2.0）

> 每个群已有明确任务主题，预设顾问团角色阵容，用户进入群无需再重申所需角色。

### 3+3 分群原则

| 档位 | 类型 | 群 |
|:----|:-----|:--|
| 🎯 **顾问团群**（需角色评估） | C007 模式C/A 自动启动 | 🎓教育产品部 / 📊战略材料部 / 💡战略研究部 |
| ⚡ **工作流群**（成熟管线） | 直接执行，不启动顾问团 | 📋会议情报部 / ✍️内容市场部 / 📬对外联络部 |
| 🗓️ **总裁办群**（综合服务） | 按需激活专业顾问 | 🗓️总裁办公室 |

### 完整映射矩阵

#### 顾问团群（需角色预设）

| 群 | chat_id | 模式 | 预设角色阵容 |
|:--|:--------|:-----|:------------|
| 🎓**教育产品部** | `oc_e9f5d48f82f74064f2526e271759235d` | C007 模式C Lightweight | 🎓教育专家→🎨UX专家→🏗️架构师，链式评估 |
| 📊**战略材料部** | `oc_8d3cb78bca0c29e354fe15d97279bca0` | C007 模式A 完整4轮 | 🎯首席策略→📋领域分析→🖋️内容设计，Socratic询问 |
| 💡**战略研究部** | `oc_64bcdf55431bedc8ba75d27c70cd4b0a` | C007 模式A 轻量(1-2轮) | 🎯首席策略+🧠认知管理，开放式讨论 |

**角色-群对应关系：**
| 角色 | 常驻群 | 职责 |
|:----|:-------|:-----|
| 🎓**资深教育专家** | 🎓教育产品部 | 课标对齐、认知规律、练习设计评估 |
| 🎨**UX/前端专家** | 🎓教育产品部 | 儿童适配、交互一致性、防误操作 |
| 🏗️**系统架构师** | 🎓教育产品部 | 数据模型、AI管线、改动量预估 |
| 🎯**首席策略顾问** | 📊战略材料部 / 💡战略研究部 | 目标澄清、叙事框架、方向引导 |
| 📋**领域分析顾问** | 📊战略材料部 | 数据支撑、事实约束、风险提示 |
| 🖋️**内容设计专家** | 📊战略材料部 | PPT结构、信息密度控制、视觉呈现 |
| 🧠**认知管理** | 💡战略研究部 | 灵感记录、知识关联、洞察沉淀 |

#### 工作流群（成熟管线，不启动顾问团）

| 群 | chat_id | 管线 |
|:--|:--------|:-----|
| 📋**会议情报部** | `oc_59e724c052b3c57d77946d81db82476e` | 会议纪要工作流 — 转写→说话人指认→待办→飞书同步 |
| ✍️**内容市场部** | `oc_9eef4ca9d845d397e908975a3104111d` | 内容工厂管线 — 选题→生成→双审→发布排期 |
| 📬**对外联络部** | `oc_9b099093361d28b19fc35dcd66cb2bdf` | 邮件起草工作流 — 场景→风格→草稿→差异学习 |

#### 总裁办群（定时推送+按需对话）

| 群 | chat_id | 职责 | 预设角色 |
|:--|:--------|:-----|:---------|
| 🗓️**总裁办公室** | `oc_50bc20a88803c7bfb72399145ce70e11` | 晨间简报(08:00)→健康节奏全天→晚间复盘(18:30) | 🗓️日程管家+🏥健康顾问+📋效率顾问+✉️邮件助理 |

**职责明细：**
- 🕗 **08:00 晨间简报**：C012飞书待办系统优先级 + 邮件待处理摘要
- 🏃 **全天健康节奏**：保留10条健康提醒cron
- 🕡 **18:30 晚间复盘**：通勤途中推送今日完成+明日调整
- **不做**：创建/修改待办（C012飞书待办系统负责）、起草邮件（📬对外联络部负责）、处理会议转写（📋会议情报部负责）

### 触发逻辑

```
消息进入 → 检测 chat_id
  ├─ oc_e9f5...235d (🎓教育产品部) → 自动加载：教育专家+UX专家+架构师，进入 C007 模式C
  ├─ oc_8d3c...ca0 (📊战略材料部) → 自动加载：首席策略+领域分析+内容设计，进入 C007 模式A
  ├─ oc_64bc...b0a (💡战略研究部) → 自动加载：首席策略+认知管理，进入轻量模式A
  ├─ oc_59e7...76e (📋会议情报部) → 直接执行：会议纪要工作流
  ├─ oc_9eef...11d (✍️内容市场部) → 直接执行：内容工厂管线
  ├─ oc_9b09...bdf (📬对外联络部) → 直接执行：邮件起草工作流
  └─ oc_50bc...e11 (🗓️总裁办公室) → 按需激活角色（日程/健康/效率/邮件助理），定时任务由cron驱动
```

### 使用指引

- 🎓**教育产品部**群进入后，用户只需描述「评估X功能」「优化Y流程」，系统自动链式调用3角色
- 📊**战略材料部**群进入后，用户描述汇报场景/受众/目的，系统自动启动Socratic 4轮询问
- 💡**战略研究部**群进入后，用户抛出话题即可，系统1-2轮讨论后输出笔记
- 📋**会议情报部**/✍️**内容市场部**/📬**对外联络部** 三群保持当前成熟管线不变
- 🗓️**总裁办公室**群以定时推送为主（cron驱动），用户进群对话时按需激活角色

## 跨群信息隔离与共享策略（⚠️ 关键设计约束）

### 问题本质

每个飞书群/DM是独立的会话上下文。用户在A群分享的信息（如通勤时间、个人偏好），在B群或DM会话中**默认不可见**。如果在B群被问到相关信息，需主动通过 `session_search` 跨群检索，或从持久化记忆查找。

### 行为铁律（用户的明确要求）

| 规则 | 说明 |
|:----|:------|
| 🚫 **不「外串」** | DM中讨论的内容、用户的交代，不主动带到其他群。除非用户明确说「把这个发到某某群」 |
| 🚫 **不「内串」** | 其他群的动态只作为DM任务的**背景参考**（了解相关情况），不把其他群的任务带到DM来处理 |
| 🚫 **不分心** | 其他群中同时发生的任务，不在当前对话中处理。当前对话只聚焦用户在此处的交代 |

**一句话原则：信息只在边界内流动。每个对话通道只处理它自己负责的任务。**

### 用户发现这个问题的场景

- 用户发现我在DM中混入了其他群的信息 → 纠正「不要串频道」
- 用户担心我会把DM信息泄露到其他群 → 强调「信息也不要串到别的群里」

### 应对策略（三明治方案）

| 层 | 机制 | 覆盖范围 |
|:---|:------|:---------|
| 🔍 **会话检索** | `session_search` 工具，指定关键词跨群搜索 | 所有群的完整历史，但需要主动触发 |
| 🧠 **持久化记忆** | `memory` 工具保存核心事实（偏好/环境/架构决策） | 跨所有会话，自动注入，但只存**我主动保存**的内容 |
| 💬 **实时询问** | 直接问用户「这个信息之前在其他群提过吗？」 | 覆盖一切，但烦人 |

### 最佳实践

1. **用户在**任何群 **说了重要事实** → 立即 `memory` 保存（偏好、行程、边界条件）
2. **被问到可能跨群的信息** → 先 `session_search` 检索，再决定是直接回答还是问用户
3. **DM中需要其他群的背景信息时** → 仅作参考使用，不把其他群的任务拉到DM处理
4. **用户指出信息缺失时** → 诚实承认架构限制，建议「是否需要我把这类信息长期记住？」

### 不要去做的

- ❌ 不要假设其他群的信息「你肯定说过，我记得」
- ❌ 不要在一次会话中重复保存相同事实到memory（会导致膨胀）
- ❌ 不要在用户指出缺失时假装知道（用户会立刻识破）
- ❌ 不要在当前对话中处理其他群的任务（即使看到了）
- ❌ 不要主动把DM信息转发到别的群（等用户指令）

## 常见陷阱

### 1. 权限开通后不生效
**症状**：开通权限后 API 仍返回 99991672
**解决**：重新获取 tenant_access_token（旧 token 不包含新权限）。不需要重启 Gateway。

### 2. Gateway 需重启才能响应新群消息

群聊创建后，在飞书开放平台侧 Bot 已经加入群，但 **Gateway 可能收不到该群的消息事件**，表现为 Bot 对群内消息无响应。

**原因**：Gateway 进程在启动时建立 WebSocket 连接并获取所有已加入群的列表。新加入的群在旧 WebSocket 连接上可能不会被自动推送消息事件（取决于飞书 WebSocket 协议实现）。

**解决**：重启 Gateway
```bash
systemctl --user restart hermes-gateway
# 或直接 kill + 自动重启
kill $(cat ~/.hermes/gateway.pid)  # 由 systemd 自动拉起
```

重启后验证：
```bash
curl -s http://localhost:8644/health  # 确认 Gateway 已重新上线
```

如果还收不到消息：
- 检查 `FEISHU_GROUP_POLICY=open` 是否配置正确
- 检查 `.env` 中是否有其他配置覆盖
- 在飞书群中手动检查 Bot 是否已加入（群设置→群成员）

### 3. 创建群的 API 响应不含 share_link
`POST /im/v1/chats` 的响应中 `share_link` 字段可能为空。用户可以在飞书客户端中手动分享群链接。

### 4. Cron投递目标：始终使用 chat_id，不要用群名称

**问题**：Cron job的`deliver`字段如果使用群名称（如 `"feishu:🏥 健康管理群 (group)"`），当群名变更时投递会失败。

**根因**：名称解析依赖 feishu-tokens.json 中的 `chat_groups.groups[].name` 字段。群名变更后旧cron仍然指向旧名称。

**修复**：2026-05-15 将10条健康提醒cron的deliver从名称格式改为chat_id格式。

**规范**：
```yaml
# ✅ 正确
deliver: "feishu:oc_50bc20a88803c7bfb72399145ce70e11"

# ❌ 错误（群名变更后会失效）
deliver: "feishu:🏥 健康管理群 (group)"
```

**迁移流程**：当升级/重命名一个群时，使用 `cronjob(action='list')` 检查所有投递到该群的cron，逐一用 `cronjob(action='update', deliver='feishu:{chat_id}')` 更新。

### 5. 重命名群聊：使用 PUT 而非 PATCH

**发现于：** 2026-05-15 职能部门化改名

飞书群重命名 API 是 `PUT /open-apis/im/v1/chats/{chat_id}`，不是 PATCH。
- `PATCH` 返回 `404 page not found` ❌
- `PUT` 返回 `{"code":0,"data":{},"msg":"success"}` ✅

```bash
curl -s -X PUT "https://open.feishu.cn/open-apis/im/v1/chats/${CHAT_ID}" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"📊 新群名"}'
```

**命名规范：** 使用 `Emoji + 四字部门名 + 部` 格式（总裁办公室例外，用「办」）

### 6. 发送消息时 content 字段必须是 JSON 字符串（双重编码陷阱）

**发现于：** 2026-05-15 连接测试

Feishu 发送消息 API (`POST /im/v1/messages`) 的 `content` 字段要求**经过 JSON 序列化的字符串**，而不是直接传递 JSON 对象。这是一个常见的陷阱，错误码为 `9499 Bad Request`。

```
# ❌ 错误 — content 是 JSON 对象
body = {
    "receive_id": "oc_xxx",
    "msg_type": "text",
    "content": {"text": "你好"}  # ← 这是 JSON 对象，API 不接受
}

# ✅ 正确 — content 是 JSON 字符串
body = {
    "receive_id": "oc_xxx",
    "msg_type": "text",
    "content": '{"text":"你好"}'  # ← 这是 JSON 字符串（stringified JSON）
}
```

**在 Python 中构造：**
```python
import json
body = {
    "receive_id": "oc_xxx",
    "msg_type": "text",
    "content": json.dumps({"text": "消息内容"})  # ← 正确：先 json.dumps 再放入 body
}
```

**在 curl 中构造（容易出错）：**
```bash
# ❌ 容易出错 — shell 转义地狱
curl -X POST ... -d '{"content":"{\"text\":\"你好\"}"}'

# ✅ 安全方式 — 用 Python 构造请求体
python3 -c "
import json, urllib.request
content = json.dumps({'text': '你好'})
body = json.dumps({'receive_id': 'oc_xxx', 'msg_type': 'text', 'content': content})
req = urllib.request.Request('...', data=body.encode(), headers={...})
urllib.request.urlopen(req)
"
```

**原理：** Feishu 的 API 约定 `content` 字段为 `string` 类型，其值本身是一个 JSON 字符串。当 `msg_type=text` 时，`content` 的值必须是 `{"text":"..."}` 的 JSON 序列化版本。

### 7. 飞书连通性诊断流程

当怀疑飞书连接有问题时，按以下层级逐级诊断：

**L1 — 检查 API 认证**
```bash
curl -s -X POST "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal" \
  -H "Content-Type: application/json" \
  -d "{\"app_id\":\"$(grep FEISHU_APP_ID ~/.hermes/.env | cut -d= -f2)\",\"app_secret\":\"$(grep FEISHU_APP_SECRET ~/.hermes/.env | cut -d= -f2)\"}"
# 预期: {"code":0,"expire":...}
# 错误: {"code":10014,"msg":"app secret invalid"}
```
❗ `.env` 文件中的密钥在 `grep` 显示时可能被截断（显示为 `dR2RKx...bi1E`），但实际文件内容完整。用 `cat -A` 或 `read_file` 验证。

**L2 — 检查 Drive/云盘 API**
```bash
# 获取 token 后测试文件夹读取
curl -s -X POST "https://open.feishu.cn/open-apis/drive/v1/metas/batch_query" \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"request_docs":[{"doc_token":"<已知文件夹token>","doc_type":"folder"}]}'
# 预期: {"code":0,"data":{"metas":[...]}}
```

**L2.5 — 检查群聊健康状况（新增）**

确认 Bot 在目标群中且群状态正常：
```bash
TOKEN=$(python3 -c "
import json, os, urllib.request
# 从 .env 读取凭证
for line in open(os.path.expanduser('~/.hermes/.env')):
    line = line.strip()
    if line.startswith('FEISHU_APP_ID='): app_id = line.split('=',1)[1]
    if line.startswith('FEISHU_APP_SECRET='): app_secret = line.split('=',1)[1]
body = json.dumps({'app_id': app_id, 'app_secret': app_secret}).encode()
req = urllib.request.Request('https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal', data=body, headers={'Content-Type': 'application/json'})
print(json.load(urllib.request.urlopen(req))['tenant_access_token'])
")

# 检查群聊状态
curl -s -H "Authorization: Bearer $TOKEN" \
  'https://open.feishu.cn/open-apis/im/v1/chats/oc_xxx' | python3 -m json.tool
```
预期返回包含 `chat_status: "normal"`, `bot_count: "1"`, `chat_mode: "group"`。如果 `chat_status` 不是 `normal` 或 `bot_count` 为 0，说明 Bot 不在群内或群异常。

**L3 — 检查消息发送 API**
```bash
# 使用 Python 构造（避免 JSON 双重编码陷阱）
python3 -c "
import json, urllib.request
token = 'YOUR_TOKEN'
content = json.dumps({'text': '连通测试'})
body = json.dumps({'receive_id': 'oc_xxx', 'msg_type': 'text', 'content': content})
req = urllib.request.Request(
    'https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=chat_id',
    data=body.encode(),
    headers={'Authorization': f'Bearer {token}', 'Content-Type': 'application/json'}
)
resp = json.loads(urllib.request.urlopen(req).read())
print(f'code={resp.get(\"code\")}, message_id={resp.get(\"data\",{}).get(\"message_id\",\"?\")}')
"
```

**L4 — 检查 Gateway WebSocket 连接**
```bash
# 查看网关日志确认飞书 WebSocket 连接状态
tail -20 ~/.hermes/logs/gateway.log
# 寻找: "Connecting to feishu..." → "[Feishu] Connected in websocket mode" → "✓ feishu connected"

# 检查 gateway 进程是否真实存活（即使 systemctl 显示 inactive）
ps aux | grep -i 'hermes_cli.*gateway' | grep -v grep
```
⚠️ **关键诊断陷阱**：在 watchdog 模式下，`systemctl --user status hermes-gateway.service` 可能显示 `inactive (dead)`，但网关实际上由看门狗脚本启动（PID 独立于 systemd）。**不要仅依赖 systemctl 判断网关是否存活**——用 `ps aux` 或 `pgrep` 二次确认。
**L5 — 消息投递验证**

在各群发送测试消息，确认 Bot 能接收并响应。检查：
- 群聊中的消息是否被 Bot 接收（WebSocket 事件）
- Bot 能否正确回复
- 确认 `FEISHU_GROUP_POLICY=open` 是否生效（无需 @ 即可响应）

⚠️ **关键先决检查**：在执行此步骤前，应先完成 L6（SEND vs RECEIVE分离诊断）——先确认 Gateway 是否能收到入站消息，否则即使发了测试消息，Bot 也不会响应（不是发信的问题）。

**L6 — SEND vs RECEIVE 分离诊断（关键陷阱）**

> ⚠️ **出信能力 ≠ 收信能力**。能成功向群发送消息（send_message 返回 success）**不代表 Bot 能接收该群的消息**。这是两个独立的飞书平台能力，由不同的配置控制。

**诊断步骤：**

1. **验证出信（send）是否正常** — 用网关的 `send_message` 工具发送测试消息到目标群
2. **验证收信（receive）是否正常** — 检查网关日志中是否出现过该群的入站消息：

```bash
# 搜索网关日志中某群聊的入站消息
grep -i 'Inbound.*message received.*chat_id=oc_xxx' ~/.hermes/logs/gateway.log

# 搜索所有非 DM 的群消息
grep 'Inbound.*message received.*chat_id=oc_' ~/.hermes/logs/gateway.log | grep -v 'oc_575e28286dba895bd619f911399b7d01'

# 查看最近是否收到过该群的消息
tail -500 ~/.hermes/logs/gateway.log | grep -B1 -A1 'oc_xxx'
```

如果 `send_message` 成功（API 返回 message_id）但网关日志中从未出现该群的入站消息 → **接收通路存在配置问题**。

**典型根因排查顺序：**

| # | 检查项 | 方法 | 修复 |
|---|--------|------|------|
| 1 | 飞书开放平台 → 事件订阅 | 访问 open.feishu.cn → 该应用 → 事件 | 确认 `im.message.receive_v1` 已订阅 |
| 2 | 事件范围含群聊 | 检查事件订阅的 `receive scope` 设置 | 配置为接收「群聊」和「私聊」消息 |
| 3 | Bot 是否在群内 | 飞书群设置 → 群成员 → 查看 Bot 头像 | 手动拉入或通过 API 加 Bot |
| 4 | `.env` 配置 | `grep -i 'FEISHU_GROUP_POLICY' ~/.hermes/.env` | 设为 `open` 以接受所有群消息 |
| 5 | Gateway 重启及日志分析 | 检查 WebSocket 连接 + 回顾入站消息历史 | `systemctl --user restart hermes-gateway` + 用 grep 检查日志中目标群的入站记录 |

> **实战案例（2026-05-21）**：向战略材料部和系统运维部发消息成功（send_message 返回 message_id），但 Gateway 日志 11,507 行中从未出现任何群聊入站消息。

**完整诊断过程：**

```
L1 ✓ API认证 — 获取token成功
L2 ✓ ⽬消息发送 — send_message 返回 message_id
L3 ✓ 群健康检查 — GET /im/v1/chats/{chat_id} 返回 chat_status=normal, bot_count=1
L4 ✓ Gateway日志 — 确认 WebSocket 已连接（"Connected in websocket mode"）
L5 ✗ 入站检查 — grep 全日志无任何群聊入站消息（仅 DM）
```

**根因分析**指向**飞书开放平台事件订阅未覆盖群聊消息**，而非网关端配置问题。

🔴 **警告**：仅检查 `.env` 的 `FEISHU_GROUP_POLICY=open` 不足以确认收信正常——必须在飞书开放平台控制台确认事件订阅配置。出信能力（send_message）与收信能力（receive）是两条完全独立的通路。

### 8. 群成员增加
如果之后需要添加更多成员，使用 `POST /open-apis/im/v1/chats/{chat_id}/members` 接口。

## 完整示例脚本

```python
"""FeishuChatGroupManager — 完整的群聊创建与注册流程"""
import json, os, time, requests

HOME = os.path.expanduser("~")
BASE = "https://open.feishu.cn/open-apis"

class FeishuChatGroupManager:
    def __init__(self):
        self.token = self._get_token()
        self.headers = {
            "Authorization": f"Bearer {self.token}",
            "Content-Type": "application/json"
        }

    def _get_token(self) -> str:
        app_id = app_secret = None
        with open(os.path.join(HOME, ".hermes", ".env"), "r") as f:
            for line in f:
                line = line.strip()
                if line.startswith("FEISHU_APP_ID="):
                    app_id = line.split("=", 1)[1]
                elif line.startswith("FEISHU_APP_SECRET="):
                    app_secret = line.split("=", 1)[1]
        resp = requests.post(
            f"{BASE}/auth/v3/tenant_access_token/internal",
            json={"app_id": app_id, "app_secret": app_secret}, timeout=10
        )
        return resp.json()["tenant_access_token"]

    def create_group(self, name: str, description: str, user_open_id: str) -> str:
        resp = requests.post(
            f"{BASE}/im/v1/chats",
            headers=self.headers,
            json={
                "name": name, "description": description,
                "user_id_list": [user_open_id],
                "setting": {
                    "membership_approval": "no_approval_required",
                    "join_message_visibility": "only_owner",
                    "leave_message_visibility": "only_owner",
                }
            }, timeout=15
        )
        data = resp.json()
        if data.get("code") == 0:
            return data["data"]["chat_id"]
        raise RuntimeError(f"创建失败: {data.get('msg')} [{data.get('code')}]")

    def register_to_tokens(self, groups: list[dict]):
        path = os.path.join(HOME, ".hermes", "feishu-tokens.json")
        with open(path, "r") as f:
            tokens = json.load(f)
        tokens["chat_groups"] = {
            "_meta": {"description": "Feishu group chat IDs", "group_policy": "open"},
            "groups": groups
        }
        with open(path, "w") as f:
            json.dump(tokens, f, indent=2, ensure_ascii=False)

    def verify_group(self, chat_id: str) -> dict:
        resp = requests.get(f"{BASE}/im/v1/chats/{chat_id}", headers=self.headers, timeout=10)
        return resp.json()
```

---

## 📄 飞书文档内容创建（Docx API）

> 本节整合自 `feishu-docx-content-creation` skill（已归档）。覆盖通过 Feishu Open API 创建结构化文档、添加内容块、管理文档内容的完整工作流。与前面群聊管理共用同一套 token 认证体系。

### 一、整体流程

```
1. 获取 tenant_access_token（复用群聊管理的认证流程）
2. （如需归档到已有文件夹）导航到目标文件夹获取 folder_token
3. 创建空白文档 → 获取 document_id
4. 通过 blocks API 批量添加内容（heading1–3, text）
5. 验证文档内容完整
6. 构造文档 URL 交付给用户
```

### 二、关键 API 端点

| 操作 | HTTP | 端点 |
|------|------|------|
| 创建文档 | POST | `/open-apis/docx/v1/documents` |
| 获取文档内容 | GET | `/open-apis/docx/v1/documents/{doc_id}/raw_content` |
| 批量添加块 | POST | `/open-apis/docx/v1/documents/{doc_id}/blocks/{block_id}/children` |
| 批量删除块 | DELETE | `/open-apis/docx/v1/documents/{doc_id}/blocks/{block_id}/children/batch_delete` |
| 查询文件夹子项 | GET | `/open-apis/drive/explorer/v2/folder/{token}/children` |

> ⚠️ **关键陷阱**：添加子的正确端点是 `/children`（POST），**不是** `/children/batch_create`（后者返回 404）。

### 三、支持的块类型

| block_type | key | 说明 |
|:----------:|:----|:-----|
| 1 | `page` | 页面根块（文档自动创建） |
| 2 | `text` | 普通段落 |
| 3 | `heading1` | 一级标题 |
| 4 | `heading2` | 二级标题 |
| 5 | `heading3` | 三级标题 |

**不支持的类型**：bullet (27)、divider (17) 等返回 1770001 (invalid param)。替代方案：bullet 用 text block 前加 `•`，divider 用 `─`。

### 四、创建文档步骤

#### Step 1: 创建空白文档

```python
r = requests.post("https://open.feishu.cn/open-apis/docx/v1/documents",
    headers={"Authorization": f"Bearer {TOKEN}"},
    json={"title": "文档标题", "folder_token": "目标文件夹token"})
doc_id = r.json()["data"]["document"]["document_id"]
```

- `folder_token` 可选，不传则创建在根目录
- 返回的 `document_id` 同时也是根 page block 的 `block_id`

#### Step 2: 批量添加内容块

```python
payload = {
    "children": [
        {"block_type": 4, "heading2": {
            "elements": [{"text_run": {"content": "标题文本", "text_element_style": {"bold": False}}}],
            "style": {}
        }},
        {"block_type": 2, "text": {
            "elements": [
                {"text_run": {"content": "加粗部分", "text_element_style": {"bold": True}}},
                {"text_run": {"content": "普通部分", "text_element_style": {"bold": False}}}
            ],
            "style": {}
        }},
    ],
    "index": 0  # 0=追加到末尾
}
r = requests.post(
    f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_id}/blocks/{doc_id}/children",
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"},
    json=payload
)
```

#### Step 3: 构造文档 URL

文档 URL 格式：`https://{tenant_prefix}.feishu.cn/docx/{doc_id}`

### 五、目录导航

```python
# 获取顶层文件列表
r = requests.get("https://open.feishu.cn/open-apis/drive/v1/files?page_size=50",
    headers={"Authorization": f"Bearer {TOKEN}"})
```

> ⚠️ `drive/v1/folders/{token}/children` 返回 404，需使用 `drive/explorer/v2/folder/{token}/children`。

### 六、常见陷阱

1. **app_secret 特殊字符**：含 `'` 等字符时，shell 中 curl 的 `-d '...'` 可能出错。用 Python requests 库或 `@file.json`。
2. **block payload 格式**：每个块必须包含 `block_type` 和对应的类型 key（type=2 → `text`，type=3 → `heading1` 等）。
3. **多块一次性提交**：children 数组支持一次创建多个块，已验证 24 个块单次 API 成功。
4. **Python True/False**：必须用 `True`/`False`（大写），不能用 `true`/`false`（JS 风格）。
5. **文档 token 漂移**：飞书文件夹 token 会随操作漂移，使用前应通过列表 API 重新获取。
6. **`upload_all` 对 HTML 文件不可靠**：上传 .html 返回 success 但不出现于文件列表。应创建 docx 文档。
7. **中文内联字符串语法陷阱**：Python 代码中的中文引号/特殊字符（`·`、`「」`）可能导致 SyntaxError。缓解方案：将 JSON/payload 写入独立文件再读取。
8. **`index` 参数**：`index: 0` 表示追加到末尾。也可指定位置插入。
