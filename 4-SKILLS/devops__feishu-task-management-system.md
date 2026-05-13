---
name: feishu-task-management-system
version: "2.3.0"
description: "C012 飞书待办事项管理系统 - 基于 Feishu Task v2 API 的待办全生命周期管理。v2.3.0 新增会议待办归类策略(父子任务结构)，6大分类维度自动聚合。"
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

## 🏷️ 会议待办归类策略（v2.3 新增）

### 概述

**痛点**：原来一场会议8-10条待办全部平铺，同类散落，看列表无法聚焦。

**解决**：按6大工作性质分类，同类合并成**父任务（专项）**，各项待办作为**子任务**。

### 6大分类维度

| 分类 | Emoji | 涵盖内容 |
|------|-------|---------|
| **数据核查** | 🔍 | 核实异常、核对数据、排查差异、匹配映射 |
| **文档输出** | 📄 | 写报告、整理清单、建台账、写邮件 |
| **协调沟通** | 🤝 | 联系某人、催促反馈、协调资源、跨部门对接 |
| **系统建设** | 🛠 | 权限申请、规则设计、模板修改、流程设计 |
| **汇报材料** | 📋 | 准备汇报、整理进度、PPT、季度会 |
| **人员考核** | 👥 | 人员扩增、考核修订、分工调整 |

### 聚合规则

**合并条件（满足任一条）：**
- ✅ **同类动作** → 同一会议中同为"核查/核对"性质的任务
- ✅ **同责任人** → 分配给同一人的多个任务
- ✅ **同工作流** → A完成后才能B（如"定位原因"+"制定修复方案"）

**不合并：**
- ❌ 跨会议（不同会议各自保留）
- ❌ 紧急独立事项（如"下午必须完成"的单独任务）
- ❌ 不同性质混编（核查+沟通不合并）

**拆分条件（子任务专项超过7项时）：** 将专项再分成2-3个子专项。

### 自动归类流程（处理新会议待办时强制执行）

每次从会议笔记/转录中提取待办事项时，按以下流程自动分类归组：

```
输入：会议名 + N条待办事项列表
  │
  ├─ 1. 内容分析 → 逐条判断属于哪个分类
  │    关键词匹配规则：
  │      🔍 数据核查 ← 含"核实/核对/排查/查/定位/匹配/统计/核查/清单/抽样/验证"
  │      📄 文档输出 ← 含"报告/台账/整理/梳理/撰写/整理"
  │      🤝 协调沟通 ← 含"联系/询问/催促/协调/对接/同步/对接/沟通/回复/对接/推动"
  │      🛠 系统建设 ← 含"权限/流程/规则/模板/系统/方案/设计/接入"
  │      📋 汇报材料 ← 含"汇报/材料/数据会/季度会/进度"
  │      👥 人员考核 ← 含"人员/考核/分工/招/抽调"
  │
  ├─ 2. 查重检查 → 检查注册表中是否已有该会议的同分类父任务
  │    匹配条件：same project(会议名) + same category emoji in summary
  │
  ├─ 3a. 已有父任务 → 直接创建为子任务（无members）
  └─ 3b. 无父任务 → 先创建父任务（有members），再创建子任务（无members）
```

**agent行为守则：** 处理任何新会议待办时，必须先执行自动归类流程，不得直接将待办创建为平铺任务。只有≤2条的小会议且类型完全不同时，可各自作为独立父任务处理。

### 三层结构

```
📋 [会议名称]
├─ 🔍 数据核查专项（含3项子任务）
│   ├─ 子任务1：核实异常数据
│   ├─ 子任务2：排查差异原因
│   └─ 子任务3：匹配映射
├─ 🤝 协调沟通专项（含2项）
│   └─ ...
└─ 📋 汇报材料（单项独立）
```

### 创建方式

父任务格式：`📋 [会议名] 分类 · 专项名称（含N项）`
子任务格式：直接用具体任务摘要，不加`📋 [会议名]`前缀

**父任务的description**中自动汇总子任务清单：
```
来自XX会议：背景描述

━━━ 子任务清单 ━━━
1. 林欣 核实手续费率异常数据
2. 张兆宇 确保异常清零
━━━━━━━━━━━━━━━━
```

### 脚本支持

`feishu-task-manager.py` 的 `create_task` 函数已支持 `parent_guid` 参数。子任务使用**独立API端点** `POST /tasks/{parent_guid}/subtasks`：

```python
# 创建父任务（有members → 用户可见）
parent = create_task("📋 [会议名] 🔍 数据核查专项（含3项）", "描述...")
# 创建子任务（无members → 不在我的任务列表出现，仅父任务内可见）
child = create_task("核实异常数据", "", parent_guid=parent["guid"])
```

**⚠️ 关键陷阱**：
1. `parent_guid` 不能放在 POST /tasks 的 body 中（会被**静默忽略**）。必须通过 `POST /tasks/{parent_guid}/subtasks` 端点创建。
2. **子任务不设members**——子任务不assign给任何人，只在父任务详情页内可见。这样用户"我的任务"列表只显示父任务（29条），不显示子任务（68条），达到视觉归并效果。

### 飞书App内效果

- ✅ 父任务可展开/折叠（仅显示父任务+子任务数，不展开显示所有子任务）
- ✅ 子任务独立跟踪完成进度
- ✅ 父任务上显示子任务完成进度（如 3/5）
- ✅ 完成所有子任务后手动完成父任务
- ✅ 子任务进度通过 `GET /tasks/{guid}/subtasks` 可查

---

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

### 2. 会议批量创建（父子结构模式 - 推荐）

**流程：先归类，再创建父子结构**

```python
# 1. 按6大分类维度归类会议待办
#   - 🔍 数据核查 → 同类合并为一个父任务
#   - 🤝 协调沟通 → 同类合并为一个父任务
#   - 等...
# 
# 2. 先创建父任务（专项），再创建子任务
# 3. 子任务通过 parent_guid 关联父任务

from feishu_task_module import create_task  # 或直接调用脚本

# 步骤A：创建父任务（专项）
parent = create_task("📋 [会议名] 🔍 数据核查专项（含3项）",
                     "来自XX会议：N项数据核查工作")

# 步骤B：创建子任务（归入父任务）
create_task("子任务1：具体内容", "", parent_guid=parent["guid"])
create_task("子任务2：具体内容", "", parent_guid=parent["guid"])
```

**示例（完整归类输出）：**

```
📋 [产险监管报送会议]
├─ 🤝 外部对接推进专项（含5项子任务）
│   ├─ 窦总：了解中银保信工具
│   ├─ 集团AI团队：讨论方向
│   └─ ...
├─ 🔍 数据修正与规则梳理专项（含4项）
│   ├─ 产险团队：6月完成存量修正
│   └─ ...
└─ ⚙️ 合规推动（单项独立）
```

### 2b. 传统会议批量（平铺模式 - 兼容保留，不推荐）

⚠️ **仅用于边缘情况**：单场会议≤2条待办且类型完全不同时，可各自作为独立父任务。

其余情况一律走 **2a. 自动归类流程**。
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
            app_id = line.split("=", 1)[1].strip().strip('"').strip("'")
        elif line.startswith("FEISHU_APP_SECRET="):
            app_secret = line.split("=", 1)[1].strip().strip('"').strip("'")

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
    "summary": "📋 测试任务",
    "description": "详细描述",
    "members": [
        {
            "type": "user",
            "id": "ou_50b21c92548fbb2173b049e57dfbdbec",
            "role": "assignee"
        }
    ]
}
data = json.dumps(body).encode()
req = urllib.request.Request("https://open.feishu.cn/open-apis/task/v2/tasks",
    data=data, headers=headers, method="POST")
result = json.loads(urllib.request.urlopen(req).read())
guid = result["data"]["task"]["guid"]
```

> ⚠️ **重要陷阱**：`members` 只能在**创建时**设置。`POST /tasks/{guid}/members` 返回 404，无法在创建后添加成员。

### A3. 设置截止日期 (PATCH)

```python
body = {
    "task": {
        "due": {"timestamp": str(int(time.time() + 3600*12)), "is_all_day": False},
        "guid": guid
    },
    "update_fields": ["due"]
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
| **子任务API误用（最危险）** | `POST /tasks` + body中放`parent_guid` -> 返回成功但静默忽略，子任务被创建为普通任务 | 必须用 `POST /tasks/{parent_guid}/subtasks` 端点，父GUID在URL路径中 |
| 子任务响应字段不同 | `POST /tasks/{parent}/subtasks` 返回 `data.subtask` 而非 `data.task` | 读取 `result.data.subtask.guid` |

### A5. 子任务创建 (Subtask API)

**端点**: `POST /open-apis/task/v2/tasks/{parent_guid}/subtasks`

**⚠️ ⚠️ ⚠️ 这是本节最重要的陷阱**

不要将 `parent_guid` 作为 body 参数传给 `POST /tasks` —— 该字段会被 Feishu API **静默忽略**，返回 `code: 0` 表面成功，但子任务被创建为普通任务，无任何错误提示。

正确做法：

```python
# 正确：父GUID在URL路径中
path = f"/task/v2/tasks/{parent_guid}/subtasks"
body = {
    "summary": "子任务标题",
    "description": "子任务描述",
    "members": [{"type": "user", "id": USER_OPEN_ID, "role": "assignee"}]
}
result = _http_request("POST", path, headers, body)
guid = result["data"]["subtask"]["guid"]  # 注意：是 subtask 不是 task
```

响应格式差异：
- 普通任务：`result.data.task.guid`
- 子任务：`result.data.subtask.guid`

### A6. 批量创建范例

```python
from datetime import datetime, timedelta
today = datetime.now()
USER_OPEN_ID = "ou_50b21c92548fbb2173b049e57dfbdbec"
tasks_data = [
    ("📋 任务1", today + timedelta(hours=12)),
    ("📋 任务2", today + timedelta(hours=10)),
]
for summary, due_time in tasks_data:
    body = {
        "summary": summary,
        "members": [{"type": "user", "id": USER_OPEN_ID, "role": "assignee"}]
    }
    req = urllib.request.Request("https://open.feishu.cn/open-apis/task/v2/tasks",
        data=json.dumps(body).encode(), headers=headers, method="POST")
    result = json.loads(urllib.request.urlopen(req).read())
    guid = result["data"]["task"]["guid"]
    patch_body = {"task": {"due": {"timestamp": str(int(due_time.timestamp())), "is_all_day": False}, "guid": guid}, "update_fields": ["due"]}
    urllib.request.urlopen(urllib.request.Request(
        f"https://open.feishu.cn/open-apis/task/v2/tasks/{guid}",
        data=json.dumps(patch_body).encode(), headers=headers, method="PATCH"))
```

> **参考文件**:
> - `references/feishu-task-api-verified.md` — API 端点验证记录、响应格式和错误信息
> - `references/feishu-subtask-api-trap-20260513.md` — 子任务API静默陷阱完整排查过程与修复记录

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
    """判断任务是否已存在（匹配标题和会议名）"""
    expected_prefix = f'📋 [{meeting_name}]'
    for t in existing_tasks:
        s = t.get('summary', '')
        if expected_prefix in s and any(kw in s for kw in [summary[:10], summary[:20]]):
            return True
    return False

new_todos = [...]
existing_count = 0
for todo in new_todos:
    if is_duplicate(todo, '会议名称', existing_tasks):
        existing_count += 1
        print(f'  ⏭️ 跳过（已存在）: {todo[:40]}...')
    else:
        print(f'  ➡️ 新建: {todo[:40]}...')

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

```
注册表查询结果：
- 会议三（保险数据报送项目讨论）-> 8条全部已存在，且存在2次重复（共16条）
- 会议二（数据安全工作会议）-> 10条全部已存在
- 会议一（IDS保单字段取值规则）-> 7条全部未同步，可以创建
```

**根因**：C013管道每15分钟轮询同步，同步后在待办段落标注了说话人编号。如果后续手动再用 `meeting` 命令导入同场会议的待办，就会产生完全相同的任务。

**预防**：在任何批量创建前，先做一次注册表搜索，三分钟内即可避免重复。

---

## 参考文件

| 文件 | 内容 |
|:-----|:------|
| `references/list-user-crash-bug-20260512.md` | list-user 因 due 字段类型不匹配崩溃的完整排查记录 |
| `references/feishu-task-api-verified.md` | Feishu Task v2 API 认证、端点验证和错误响应示例 |
| `references/feishu-subtask-api-trap-20260513.md` | 子任务API静默陷阱完整排查过程与修复记录 |
