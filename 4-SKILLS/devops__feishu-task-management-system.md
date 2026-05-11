---
name: feishu-task-management-system
version: "2.0.0"
description: C012 飞书待办事项管理系统 — 基于 Feishu Task v2 API 的待办全生命周期管理。两组任务分组，支持创建/列表/完成/删除/会议批量/每日回顾。包含 Task v2 API 底层参考（认证、创建、PATCH、陷阱速查）。
author: Hermes Agent
trigger: 用户说"记一个待办"、"创建任务"、"待办事项"、"会议待办"、"今日回顾"、或要求"创建飞书任务"、"放入飞书任务"、"在飞书任务中记录"等关键词
---

# C012 飞书待办事项管理系统

## 概述

基于飞书 Task v2 API + 本地注册表双轨存储的待办管理系统。由于 Feishu API 的 list 接口无法返回应用创建的任务，本地注册表（`~/.hermes/data/task-registry.json`）作为任务索引，Feishu 任务作为真实存储。

## 两组任务分组

| 分组 | 标记 | 说明 | 创建方式 |
|------|------|------|----------|
| 👤 **你的任务** | 无项目标签或自定义 [项目名] | 你随口记的待办、会议提炼的待办 | 不加 `--hermes` |
| 🤖 **我的任务** | `[My Hermes Agent]` 项目标签 | 你分配给我的事务、我执行中的派生事项 | 加 `--hermes` |

## 核心文件

| 文件 | 说明 |
|------|------|
| `~/.hermes/scripts/feishu-task-manager.py` | 核心脚本 |
| `~/.hermes/scripts/daily-task-review.py` | 每日回顾脚本 |
| `~/.hermes/data/task-registry.json` | 本地注册表 |

## 分类体系

| Emoji | 分类 | 使用场景 |
|-------|------|----------|
| 💼 | 工作 | 数据管理、部门协作、汇报材料 |
| 📋 [项目名] | 会议/项目 | 方括号标项目名，如 `📋 [部门周会] · 确认上线时间` |
| 🏠 | 生活/家庭 | 孩子教育、家庭事务 |
| 📚 | 成长 | 阅读、自媒体、技能提升 |
| 🎯 | 里程碑 | 专项节点、系统建设 |
| ⚡ | 紧急 | 突发事项 |

## 交互流程

### 0. 两组任务创建
```python
# 👤 你的任务（加 --hermes 或使用 --project）
r = subprocess.run(["python3", script, "create", "标题", "--cat", "工作"], ...)
# 🤖 我的任务（自动设 project=My Hermes Agent）
r = subprocess.run(["python3", script, "create", "标题", "--cat", "工作", "--hermes"], ...)
```

### 1. 随手记待办（用户口头告知）
```python
import subprocess
script = os.path.expanduser("~/.hermes/scripts/feishu-task-manager.py")
# 你的任务
r = subprocess.run(["python3", script, "create", title, "--cat", category, "--desc", desc], capture_output=True, text=True, timeout=15)
# 我的任务（加 --hermes）
r = subprocess.run(["python3", script, "create", title, "--cat", category, "--hermes"], capture_output=True, text=True, timeout=15)
```

### 2. 会议批量创建
```python
r = subprocess.run(["python3", script, "meeting", meeting_name] + todo_list, capture_output=True, text=True, timeout=30)
```

### 3. 查看待办
```python
# 全部
r = subprocess.run(["python3", script, "list"], ...)
# 仅我的待办
r = subprocess.run(["python3", script, "list-hermes"], ...)
# 仅你的待办
r = subprocess.run(["python3", script, "list-user"], ...)
```

### 4. 标记完成
```python
r = subprocess.run(["python3", script, "complete", guid], capture_output=True, text=True, timeout=15)
```

### 5. 删除任务
```python
r = subprocess.run(["python3", script, "delete", guid, "--force"], capture_output=True, text=True, timeout=15)
```

### 6. 查看任务详情
```python
r = subprocess.run(["python3", script, "get", guid], capture_output=True, text=True, timeout=15)
```

### 7. JSON 模式（供自动化调用）
```python
import subprocess, json
r = subprocess.run(["python3", script, "--json", "create", f"title={title}", f"category={cat}"], capture_output=True, text=True, timeout=15)
result = json.loads(r.stdout)
```

## 每日回顾流程

由 cron 每天早上 7:30 自动执行：
1. 读取本地注册表
2. 分析待办任务（逾期/今日到期/长期未推进/分类分布）
3. 生成结构化报告（含建议）
4. 通过 send_message 推送到飞书

手动触发：
```bash
python3 ~/.hermes/scripts/daily-task-review.py        # 完整版
python3 ~/.hermes/scripts/daily-task-review.py --brief # 简洁版
```

---
## 附录 A: Task v2 API 底层参考

> 以下内容来自 `feishu-task-management`（已合并），提供直接调用 API 端点时的完整参考。

### A1. 获取 tenant_access_token

```python
import os, json, urllib.request

ENV_FILE = os.path.expanduser("~/.hermes/.env")
app_id = app_secret = ""
with open(ENV_FILE) as f:
    for line in f:
        line = line.strip()
        if line.startswith("FEISHU_APP_ID="):
            app_id = line.split("=", 1)[1].strip().strip("\"").strip("'")
        elif line.startswith("FEISHU_APP_SECRET="):
            app_secret = line.split("=", 1)[1].strip().strip("\"").strip("'")

auth_data = json.dumps({"app_id": app_id, "app_secret": app_secret}).encode()
req = urllib.request.Request(
    "https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal",
    data=auth_data, headers={"Content-Type": "application/json"})
token = json.loads(urllib.request.urlopen(req).read())["tenant_access_token"]
```

### A2. 创建任务 (POST)

**端点**: `POST /open-apis/task/v2/tasks`
```python
body = {"summary": "🏠 P0 · 今晚前：设立21:15洗漱闹钟", "description": "详细描述"}
data = json.dumps(body).encode()
req = urllib.request.Request("https://open.feishu.cn/open-apis/task/v2/tasks",
    data=data, headers=headers, method="POST")
result = json.loads(urllib.request.urlopen(req).read())
guid = result["data"]["task"]["guid"]  # 注意：是 guid，不是 id
```

### A3. 设置截止日期 (PATCH)

```python
body = {
    "task": {
        "due": {"timestamp": str(int(time.time() + 3600*12)), "is_all_day": False},
        "guid": guid  # 必须包含 guid
    },
    "update_fields": ["due"]  # 必须声明要更新的字段
}
```

### A4. API 陷阱速查

| 陷阱 | 现象 | 解决 |
|------|------|------|
| list 返回空 | `GET /task/v2/tasks` 始终返回 `[]` | 必须用本地注册表 |
| `guid` vs `id` | 创建响应中是 `data.task.guid` | 用 guid 做后续操作 |
| PATCH 结构 | 只传 `{"due":...}` 报 validation error | 必须 `{"task":{...}, "update_fields":[...]}` |
| `update_fields` 必填 | 不声明则字段不更新 | 始终加上此字段 |
| `completed_at` 格式 | 必须 13 位毫秒级 | 设 `"0"` 重新打开，当前毫秒完成 |
| 成员/协作者 API | `POST .../members` 返回 404 | 不支持 |
| 清单归属 | 清单属于 APP 而非用户 | 无法将任务归属到清单 |

### A5. 批量创建范例（教育计划场景）

```python
from datetime import datetime, timedelta
today = datetime.now()
tasks_data = [
    ("🏠 今晚前：设立21:15洗漱闹钟", today + timedelta(hours=12)),
    ("📋 今天白天：家庭会议签《契约》", today + timedelta(hours=10)),
]
for summary, due_time in tasks_data:
    # Create
    body = {"summary": summary}
    req = urllib.request.Request("https://open.feishu.cn/open-apis/task/v2/tasks",
        data=json.dumps(body).encode(), headers=headers, method="POST")
    result = json.loads(urllib.request.urlopen(req).read())
    guid = result["data"]["task"]["guid"]
    # Set due
    patch = {"task": {"due": {"timestamp": str(int(due_time.timestamp())), "is_all_day": False}, "guid": guid}, "update_fields": ["due"]}
    urllib.request.urlopen(urllib.request.Request(
        f"https://open.feishu.cn/open-apis/task/v2/tasks/{guid}",
        data=json.dumps(patch).encode(), headers=headers, method="PATCH"))
```

> **参考文件**: `references/feishu-task-api-verified.md` — 包含完整的 API 端点验证记录、响应格式和错误信息。

## 注意事项

1. **GUID 是唯一标识**：任务创建后返回 guid (UUID格式)，后续所有操作需要 guid
2. **本地注册表必需**：Feishu API 的 list 接口不返回 app 创建的任务，必须依靠本地注册表
3. **删除后不可恢复**：delete 操作同时删除 Feishu 任务和本地注册记录
4. **分类通过 summary emoji 前缀**：`💼 工作 · 标题`，`📋 [项目名] · 标题`
5. **会议待办格式**：summary = `📋 [会议名称] · 待办标题`
6. **两组区分**：你的任务不加 `[My Hermes Agent]` 项目标签；我的任务必加 `--hermes` 参数
