---
name: feishu-chat-group-management
description: "飞书群聊管理 — 创建群聊、配置Bot、管理权限、多群平行对话架构"
version: 1.0.0
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
```

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
