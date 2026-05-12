---
name: feishu-task-management-system
version: "2.2.0"
description: C012 飞书待办事项管理系统 — 基于 Feishu Task v2 API 的待办全生命周期管理。两组任务分组，支持创建/列表/完成/删除/会议批量/每日回顾。包含 Task v2 API 底层参考（认证、创建、PATCH、陷阱速查）。
author: Hermes Agent
trigger: 用户说"记一个待办"、"创建任务"、"待办事项"、"会议待办"、"今日回顾"、或要求"创建飞书任务"、"放入飞书任务"、"在飞书任务中记录"等关键词
---

# C012 飞书待办事项管理系统

## 概述

基于飞书 Task v2 API + 本地注册表双轨存储的待办管理系统。由于 Feishu API 的 list 接口无法返回应用创建的任务，本地注册表（`~/.hermes/data/task-registry.json`）作为任务索引，Feishu 任务作为真实存储。

## ⚠️ 关键UX：任务必须在创建时指定负责人

### 问题

通过 Feishu Task v2 API 创建任务时，如果**不指定 `members` 字段**，任务归属**应用**而非用户，在飞书「任务」App中不可见。

用户反馈：「我在飞书的任务中没有看到新增的任务」。

### 解决方案

创建任务时，必须在 POST body 中同时指定用户为 `assignee`：

```json
{
  "summary": "📋 任务标题",
  "members": [
    {
      "type": "user",
      "id": "ou_50b21c92548fbb2173b049e57dfbdbec",
      "role": "assignee"
    }
  ]
}
```

任务的 `members` API（`POST /tasks/{guid}/members`）确实返回 404，**无法在创建后添加成员**。成员必须在**创建时一次性指定**。

### 修复记录

2026-05-12 发现此问题后，`feishu-task-manager.py` 的 `create_task` 函数已修改为始终包含 `members` 字段。**之前创建的所有任务（不含members）在飞书App中仍不可见**，需要删除重建。

### 每日回顾依然有效

cron 每日 7:30 的回顾推送和 `list-user` 命令行仍正常显示所有任务（含无members的旧任务），不受此影响。

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
# 👤 你的任务
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
# 基本用法：纯标题列表
r = subprocess.run(["python3", script, "meeting", meeting_name] + todo_list, capture_output=True, text=True, timeout=30)

# 带描述（推荐）："标题::描述" 格式，描述包含会议中提及的关键建议/背景
r = subprocess.run(["python3", script, "meeting", "会议名",
    "任务1::会议中讨论的关键建议和背景",
    "任务2::另一个任务的讨论要点"], capture_output=True, text=True, timeout=30)
```

**用户偏好（2026-05-12）**：每次创建会议待办时，应尽量带上 `::` 描述的格式，描述内容包含：
- 该任务在会议中讨论的核心背景/问题
- 关键人的建议或决策
- 后续需要关注的风险点
- 截止日期或里程碑信息

例：
```
"倩如：给信保提供理赔表取数规则，并抽取示例::信保理赔部门反馈多张表数据与业务认知不符，要求在取数规则基础上抽取少量示例给业务核对。"
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

**必须在创建时指定负责人**，否则任务不会出现在用户飞书App中：

```python
body = {
    "summary": "🏠 今晚前：设立21:15洗漱闹钟",
    "description": "详细描述",
    "members": [
        {
            "type": "user",
            "id": "ou_50b21c92548fbb2173b049e57dfbdbec",  # 用户的open_id
            "role": "assignee"  # 指定为负责人
        }
    ]
}
data = json.dumps(body).encode()
req = urllib.request.Request("https://open.feishu.cn/open-apis/task/v2/tasks",
    data=data, headers=headers, method="POST")
result = json.loads(urllib.request.urlopen(req).read())
guid = result["data"]["task"]["guid"]  # 注意：是 guid，不是 id
```

> ⚠️ **重要陷阱**：`members` 只能在**创建时**设置。`POST /tasks/{guid}/members` 返回 404，无法在创建后添加成员。

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
| 成员不可追加 | `POST /tasks/{guid}/members` 返回 404 | 必须在**创建时**通过 `body.members` 指定，无法后加 |
| 清单归属 | 清单属于 APP 而非用户 | 无法将任务归属到清单 |

### A5. 批量创建范例（教育计划场景）

```python
from datetime import datetime, timedelta
today = datetime.now()
USER_OPEN_ID = "ou_50b21c92548fbb2173b049e57dfbdbec"
tasks_data = [
    ("🏠 今晚前：设立21:15洗漱闹钟", today + timedelta(hours=12)),
    ("📋 今天白天：家庭会议签《契约》", today + timedelta(hours=10)),
]
for summary, due_time in tasks_data:
    # Create with members (required for visibility in Feishu app)
    body = {
        "summary": summary,
        "members": [{"type": "user", "id": USER_OPEN_ID, "role": "assignee"}]
    }
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

> **参考文件**: `references/feishu-task-api-verified.md` — API 端点验证记录、响应格式和错误信息。

## 注意事项

1. **GUID 是唯一标识**：任务创建后返回 guid (UUID格式)，后续所有操作需要 guid
2. **本地注册表必需**：Feishu API 的 list 接口不返回 app 创建的任务，必须依靠本地注册表
3. **删除后不可恢复**：delete 操作同时删除 Feishu 任务和本地注册记录
4. **分类通过 summary emoji 前缀**：`💼 工作 · 标题`，`📋 [项目名] · 标题`
5. **会议待办格式**：summary = `📋 [会议名称] · 待办标题`
6. **两组区分**：你的任务不加 `[My Hermes Agent]` 项目标签；我的任务必加 `--hermes` 参数

## 已知问题

### 旧任务在飞书App中不可见

**2026-05-12 之前创建的任务**（未含 `members` 字段）在飞书Task App中仍不可见。如需迁移，需要：
1. 从注册表中找到旧任务列表
2. 删除飞书中的旧任务（`delete` 命令）
3. 重新创建（脚本已更新，会自动加members）

### list-user / list-hermes 因 due 字段类型不匹配崩溃（已修复 ✅）

**根因**：注册表中某些任务的 `due` 字段存储为字符串（`"1747238400"`）而非 dict（`{"timestamp": "1747238400"}`），导致 `.get("timestamp")` 报 `AttributeError`。

**修复**：2026-05-12 将所有 `due.get("timestamp")` 调用改为兼容字符串和dict两种格式的健壮写法。详情见 `references/list-user-crash-bug-20260512.md`。

### C013自动同步的任务可能已存在

Get笔记会议笔记的「📋 待办事项」段落会被C013管道自动提取并导入C012。手动调用 `meeting` 命令前应先确认是否已被自动同步，避免重复创建。

## ⚠️ 批量创建任务前：去重验证协议（2026-05-12 用户明确要求）

**每次从会议待办批量创建飞书任务前，必须先执行去重检查**，否则可能产生重复任务（C013已自动同步 + 之前手动创建 = 每条2-4份）。

### 标准去重流程

```python
import json, os

registry_path = os.path.expanduser('~/.hermes/data/task-registry.json')
with open(registry_path, 'r', encoding='utf-8') as f:
    registry = json.load(f)

existing_tasks = registry.get('tasks', [])

def is_duplicate(summary, meeting_name, existing_tasks):
    \"\"\"判断任务是否已存在（匹配标题和会议名）\"\"\"
    # 注册表中的会议任务格式：📋 [会议名称] · 待办标题
    expected_prefix = f'📋 [{meeting_name}]'
    for t in existing_tasks:
        s = t.get('summary', '')
        # 同一会议中，标题含目标会议名且待办内容匹配
        if expected_prefix in s and any(kw in s for kw in [summary[:10], summary[:20]]):
            return True
    return False

# 使用示例：
new_todos = [...]  # 从待办中提取
existing_count = 0
for todo in new_todos:
    if is_duplicate(todo, '会议名称', existing_tasks):
        existing_count += 1
        print(f'  ⏭️ 跳过（已存在）: {todo[:40]}...')
    else:
        print(f'  ➡️ 新建: {todo[:40]}...')
        # 创建任务...

if existing_count > 0:
    print(f'⚠️ 跳过 {existing_count}/{len(new_todos)} 条重复任务')
```

### 三种结果场景

| 场景 | 注册表状态 | 行动 |
|------|-----------|------|
| **全新增** | 注册表中没有匹配条目 | 全部创建 |
| **全部已存在** | 每条待办都能在注册表中找到匹配 | 全部跳过，告知用户已同步完成 |
| **部分重复** | 部分待办已存在、部分未同步 | 仅创建未同步的条，报告跳过明细 |

### 检查要点

1. **会议名称匹配**：注册表中任务 summary 格式为 `📋 [保险数据报送项目工作讨论会议] 下午整理历史问题保单清单`，用 `📋 [会议名]` 前缀判断同会议
2. **模糊匹配**：待办标题可能存在微小差异（如增减标点/空格），用 `待办[:10]` 或 `待办[:20]` 做前缀匹配，不需要精确全文匹配
3. **C013同步标记**：C013导入的任务在注册表中 `source` 字段标记为 `"c013"`，手动创建不会有该标记
4. **勿忽略"说话人"变体**：同一待办在不同批次中可能说话人编号不同，建议用**任务核心动词+宾语**来匹配（如"核对车险非车险保单状态"），而非完整标题

### 实际案例（2026-05-12）

处理三场5月12日会议时发现：

```python
# 注册表查询结果：
# 会议三（保险数据报送项目讨论）→ 8条全部已存在，且存在2次重复（共16条）
# 会议二（数据安全工作会议）→ 10条全部已存在
# 会议一（IDS保单字段取值规则）→ 7条全部未同步，可以创建
```

**根因**：C013管道每15分钟轮询同步，同步后在待办段落标注了说话人编号。如果后续手动再用 `meeting` 命令导入同场会议的待办，就会产生完全相同的任务。

**预防**：在任何批量创建前，先做一次注册表搜索，三分钟内即可避免重复。

---

## 参考文件

| 文件 | 内容 |
|:-----|:------|
| `references/list-user-crash-bug-20260512.md` | list-user 因 due 字段类型不匹配崩溃的完整排查记录 |
| `references/feishu-task-api-verified.md` | Feishu Task v2 API 认证、端点验证和错误响应示例 |
