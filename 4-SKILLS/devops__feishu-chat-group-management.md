---
name: feishu-chat-group-management
description: "飞书群聊管理 — 创建群聊、配置Bot、管理权限、多群平行对话架构"
version: 2.0.0
triggers:
  - user asks to create a Feishu group chat / 创建飞书群聊
  - user wants to set up parallel conversations / 设立平行对话通道
  - user asks to add bot to groups / 把Bot加到群
  - user wants multi-group architecture / 多群对话架构
  - user mentions FEISHU_GROUP_POLICY or im:chat permissions
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
📊 汇报材料生成群    → material-production
📋 会议处理群         → meeting-minutes
✍️ 内容生成群         → content-factory
💡 每日思想碰撞群     → daily-conversation
🎓 教育系统优化群     → edu-hub
📬 邮件消息协助群      → email-assistance
🏥 健康管理群          → health-management
```

## 群感知角色预设注册表（v2.0）

> 每个群已有明确任务主题，预设顾问团角色阵容，用户进入群无需再重申所需角色。

### 3+3 分群原则

| 档位 | 类型 | 群 |
|:----|:-----|:--|
| 🎯 **顾问团群**（需角色评估） | C007 模式C/A 自动启动 | 🎓教育优化 / 📊汇报材料 / 💡思想碰撞 |
| ⚡ **工作流群**（成熟管线） | 直接执行，不启动顾问团 | 📋会议处理 / ✍️内容生成 / 📬邮件协作 |
| 🏥 **专业服务群**（专业角色服务） | 按需激活专业顾问 | 🏥健康管理 |

### 完整映射矩阵

#### 顾问团群（需角色预设）

| 群 | chat_id | 模式 | 预设角色阵容 |
|:--|:--------|:-----|:------------|
| 🎓**教育优化** | `oc_e9f5d48f82f74064f2526e271759235d` | C007 模式C Lightweight | 🎓教育专家→🎨UX专家→🏗️架构师，链式评估 |
| 📊**汇报材料** | `oc_8d3cb78bca0c29e354fe15d97279bca0` | C007 模式A 完整4轮 | 🎯首席策略→📋领域分析→🖋️内容设计，Socratic询问 |
| 💡**思想碰撞** | `oc_64bcdf55431bedc8ba75d27c70cd4b0a` | C007 模式A 轻量(1-2轮) | 🎯首席策略+🧠认知管理，开放式讨论 |

**角色-群对应关系：**
| 角色 | 常驻群 | 职责 |
|:----|:-------|:-----|
| 🎓**资深教育专家** | 🎓教育优化 | 课标对齐、认知规律、练习设计评估 |
| 🎨**UX/前端专家** | 🎓教育优化 | 儿童适配、交互一致性、防误操作 |
| 🏗️**系统架构师** | 🎓教育优化 | 数据模型、AI管线、改动量预估 |
| 🎯**首席策略顾问** | 📊汇报材料 / 💡思想碰撞 | 目标澄清、叙事框架、方向引导 |
| 📋**领域分析顾问** | 📊汇报材料 | 数据支撑、事实约束、风险提示 |
| 🖋️**内容设计专家** | 📊汇报材料 | PPT结构、信息密度控制、视觉呈现 |
| 🧠**认知管理** | 💡思想碰撞 | 灵感记录、知识关联、洞察沉淀 |

#### 工作流群（成熟管线，不启动顾问团）

| 群 | chat_id | 管线 |
|:--|:--------|:-----|
| 📋**会议处理** | `oc_59e724c052b3c57d77946d81db82476e` | 会议纪要工作流 — 转写→说话人指认→待办→飞书同步 |
| ✍️**内容生成** | `oc_9eef4ca9d845d397e908975a3104111d` | 内容工厂管线 — 选题→生成→双审→发布排期 |
| 📬**邮件协作** | `oc_9b099093361d28b19fc35dcd66cb2bdf` | 邮件起草工作流 — 场景→风格→草稿→差异学习 |

#### 专业服务群（专业角色按需激活）

| 群 | chat_id | 服务类型 |
|:--|:--------|:---------|
| 🏥**健康管理** | `oc_50bc20a88803c7bfb72399145ce70e11` | 🏥个人健康专家 — 体检分析/改善计划/饮食运动追踪 |

### 触发逻辑

```
消息进入 → 检测 chat_id
  ├─ oc_e9f5...235d (🎓教育优化) → 自动加载：教育专家+UX专家+架构师，进入 C007 模式C
  ├─ oc_8d3c...ca0 (📊汇报材料) → 自动加载：首席策略+领域分析+内容设计，进入 C007 模式A
  ├─ oc_64bc...b0a (💡思想碰撞) → 自动加载：首席策略+认知管理，进入轻量模式A
  ├─ oc_59e7...76e (📋会议处理) → 直接执行：会议纪要工作流
  ├─ oc_9eef...11d (✍️内容生成) → 直接执行：内容工厂管线
  ├─ oc_9b09...bdf (📬邮件协作) → 直接执行：邮件起草工作流
  └─ oc_50bc...e11 (🏥健康管理) → 激活🏥个人健康专家，进入健康分析模式
```

### 使用指引

- 🎓群进入后，用户只需描述「评估X功能」「优化Y流程」，系统自动链式调用3角色
- 📊群进入后，用户描述汇报场景/受众/目的，系统自动启动Socratic 4轮询问
- 💡群进入后，用户抛出话题即可，系统1-2轮讨论后输出笔记
- 📋/✍️/📬 三群保持当前成熟管线不变

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

### 4. 群成员增加
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
