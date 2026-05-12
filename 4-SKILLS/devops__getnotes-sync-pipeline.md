---
name: getnotes-sync-pipeline
description: C013 — Get笔记认知同步管道 + 手动查询/检索。自动管道每15分钟轮询同步，手动模式支持MCP检索→本地文件回退的多层灾备策略。
version: 1.3.0
author: Hermes Agent
metadata:
  hermes:
    tags:
      - devops
      - c013
      - getnotes
      - pipeline
    category: devops
---

# C013 — Get笔记认知同步管道 v1.0.0

## 概览

将 Get笔记 作为「认知捕获入口」，通过自动化管道将笔记内容分流到多个子系统。

## 架构

```
Get笔记 OpenAPI (openapi.biji.com)
    │ 每15分钟 no_agent=True          │ 按需手动查询 (MCP/server)
    ▼                                  ▼
getnotes-pipeline.py (主调度)     MCP getnote 工具 (26 tools)
    │                                  │
    ├─ ... 增量处理 ...                 ├─ recall: 语义搜索
    └─ 写入本地缓存/文件               ├─ list_notes / get_note
                                       └─ (备选) 本地文件回退
                                            │
                                            ▼
                                      wiki/raw/get-notes/ (已同步文件)
```

## 查询与检索（手动模式）

当需要从 Get笔记 中检索信息时（如提取待办、搜索历史会议），使用以下分层策略：

### L1: MCP Server（优先）
```bash
# 语义搜索
# 调用 getnote recall(query="关键词")

# 列出笔记
# 调用 getnote list_notes()

# 获取详情
# 调用 getnote get_note(id=<note_id>)
```

### L2: 本地缓存回退（MCP 不可用时）

当 MCP `get_note(id)` 返回"笔记不存在"错误（即使 `recall` 能找到该笔记），使用本地缓存文件替代。

**可靠读取方法**：
```bash
# 1. 查找本地缓存文件（按时间）
cd ~/wiki/raw/get-notes/ && ls -la 20260511*

# 2. 用 cat 读取（read_file 工具对中文文件名不兼容）
cat "~/wiki/raw/get-notes/20260511-数据管理部AI项目与east报送工作周例会纪要-1909645560491873880.md"

# 3. Python 提取待办
python3 -c "
import re
with open('path/to/file.md') as f:
    content = f.read()
# 提取 📋 待办事项 段落
m = re.search(r'### 📋 待办事项(.+?)(?=### |\Z)', content, re.DOTALL)
if m:
    for line in m.group(1).strip().split('\n'):
        line = line.strip()
        if line.startswith('- '):
            print(line[2:])
"
```

> **⚠ 已知问题**：`read_file`（Hermes内置工具）无法定位含中文的文件路径，即使 `terminal ls` 正确显示。必须使用 `terminal cat` 或 `terminal python3` 读取。这是工具层限制。

**缓存文件路径映射**：
```python
缓存路径: ~/wiki/raw/get-notes/
文件命名: YYYYMMDD-<title>-<note_id>.md
         # 例: 20260511-数据管理部AI项目与east报送工作周例会纪要-1909645560491873880.md
```

### L3: 直接从缓存提取元数据
```python
# 按日期范围过滤
cat ~/.hermes/cache/getnotes-cache.json | python3 -c "
import json,sys
d=json.load(sys.stdin)
for nid,n in d['notes'].items():
    if '2026-05-07' <= n['updated_at'][:10] <= '2026-05-10':
        print(f\"{nid}: {n['title']} ({n['updated_at']})\")
"""
```

### 待办提取后处理：自动同步到 C012

C013 管道在同步会议笔记时，会自动提取 **📋 待办事项** 段落中的条目并写入 C012 的本地注册表（`~/.hermes/data/task-registry.json`），无需手动 `feishu-task-manager meeting`。

**⚠️ 可见性限制**：C013自动同步的任务写入的是**本地注册表**，不调用 `create_task` API，因此这些任务**不会出现在飞书Task App中**（无 `members` 字段）。用户只能通过 `list-user` CLI 或每日回顾推送查看。

**这意味着**：
- 当你想从会议笔记创建待办到 C012 时，**先检查 C012 是否已自动同步**（运行 `list-user` 或读注册表）
- 手动调用 `feishu-task-manager meeting` 前，确认不会重复创建
- C013 同步的待办带 `[数据管理部周会]` 之类的项目标签，与手动会议创建格式一致

无论数据来源（MCP 响应 / 本地文件 / 管道输出），会议笔记中待办事项格式一致：

```
### 📋 待办事项
- 说话人1：具体事项描述
- 张明星：具体事项描述
```

**提取流程**：
1. 搜索 `📋 待办事项` 标题
2. 提取下方所有 `- 行`（遇到下一个 `###` 停止）
3. 只需读取 AI 总结段落的待办部分（Get笔记已自动提取）
4. 若需更完整信息，可结合章节概要的拟定标题判断**隐含待办**

**隐含待办识别**：某些会议中明确布置的任务（如「5月10日前完成」「要求林欣负责」）未出现在 📋 待办事项 区块，但散布在会议总结或章节概要中。需要全文扫描时间指示词（「要求」「负责」「下周」「X月X日前」）。

```
# 扫描待办关键词的正则
待办关键词: 要求|负责|安排|跟进|确认|推进|完成|落实|提交|汇报|整理|修改|协调|配合|通知|对接
时间节点: X月X日|下周|明天|今天|X/XX|下周一|月底
```

## 文件

| 文件 | 用途 |
|:-----|:------|
| `~/.hermes/scripts/getnotes-pipeline.py` | 主调度脚本 |
| `~/.hermes/cache/getnotes-cache.json` | 增量检测缓存 |
| `~/.hermes/cache/getnotes-pending.json` | 待办待确认列表 |
| `~/.hermes/data/c013-config.json` | 可配置参数（关键词等） |
| `~/.hermes/.env.getnote` | API凭证 |

## 使用

### 手动运行
```bash
cd ~/.hermes && python3 scripts/getnotes-pipeline.py
```

### 运行时行为
- **无新笔记（正常）** → 静默退出（stdout 为空，cron 不发送消息）
- **无新笔记（cursor已落后）** → 自动降级全量扫描，拉 `since_id=0` 第一页过滤已知笔记。若发现新笔记则正常处理；若真的无新笔记则更新 `last_since_id` 后静默退出
- **有新笔记** → 输出通知消息（cron 自动送达飞书）
- **会议笔记含待办** → 输出具体待办列表，提示用户确认
- **阅读/灵感笔记** → 输出归档路径
- **stderr 输出**（cron不可见，仅日志）：`🔄 全量扫描发现 N 条新笔记（cursor已落后）`

### 确认待办
用户收到通知后，回复「确认」或「忽略」即可处理 PENDING 列表。

## 分类规则

| 类型 | 触发条件 |
|:----|:---------|
| 🎤 会议 | `note_type=meeting` 或标题含「会议/讨论/评审/周会/汇报/同步/复盘/规划」 |
| 📖 阅读 | `note_type=link` 或标题含「读书/笔记/摘要/学习/书评/文章」 |
| 💡 灵感 | 标题含「想法/灵感/反思/思考/感悟/随想/心得」 |
| 📝 一般 | 其他 |

## Speaker Mapping 发言人识别模块 v1.1

会议笔记中的发言人被标记为"说话人1、说话人2…"，无法直接用于下游分析。

### ⚠️ 正确流程（用户明确约定的工作流）

```
会议笔记同步后
    │
    ▼
我从完整转写原文中提取每个说话人"最具识别性的一段话"
    │  （含人名/项目/业务术语的具体段落，2-4段/人）
    ▼
呈现给你指认:
  【说话人1】▶ "...科创中心的预算..."
  【说话人2】▶ "...财务部的时间..."
  ...
    │
    ▼
你给我映射关系: "说话人1=老苏, 说话人2=翟总, 说话人3=夜雨"
    │
    ▼
我继续后续分析（待办提取/风格分析/画像更新）
    映射自动保存 → 下次同会议复用
```

### ❌ 错误的做法（不要再犯）
- 🚫 不要自行推断映射关系（你会认错人）
- 🚫 不要跳过指认直接进入待办提取
- 🚫 不要问用户"你告诉我映射关系"——你的工作是提取段落给用户指认

### 核心脚本

| 文件 | 用途 |
|:-----|:------|
| `~/.hermes/scripts/getnotes-speaker-analyzer.py` | 发言识别与分析 |
| `~/.hermes/data/speaker-mappings.json` | 持久化映射库 |

### 命令

```bash
# 1. 列出会议发言人（当新会议同步时）
python3 getnotes-speaker-analyzer.py speakers --file "<会议笔记路径>"

# 2. 你告诉我映射 → 分析（映射自动保存）
python3 getnotes-speaker-analyzer.py analyze \
  --file "<会议笔记路径>" \
  --mapping "说话人2=翟总,说话人3=夜雨"

# 3. 查看已知人物库
python3 getnotes-speaker-analyzer.py list-known
```

### 分析产出示例（翟总）

```
发言: 19段/2415字  均长: 127.1  问句: 42.1%
句式: 长篇论述型
焦点: 进展(4次), 结果(3次), 运营(3次)
习惯: 常用'就是' '其实' '对吧' '是吧'
```

映射保存后，下回同笔记ID的会议自动复用映射。

## 已知限制
- 首次运行时需手动设置 `last_since_id` 跳过历史笔记
- Get笔记 API 5000次/天配额，96次/天轮询已远低于限制
- 会议笔记详情的 `content` 字段较大（100K+字符），超时已设为120秒
- **Cursor分页落后**（2026-05-11发现并修复）：管道使用 `since_id` 游标做增量检测，但游标指向分页链末端后，新创建的笔记（位于游标之前）会被静默遗漏。修复：cursor无结果时自动降级全量扫描（`since_id=0`）并过滤已知笔记。详见 `references/cursor-pagination-bug-20260511.md`
- **OpenAPI Detail vs MCP GetNote ID差异**：MCP list返回的 `id` 有时与OpenAPI detail的 `id` 尾数不同（如 `1909645560491873800` vs `1909645560491873880`），导致MCP get_note返回"笔记不存在"。管道使用OpenAPI detail稳定可靠，MCP get_note仅作备选手动查询用

## 参考文档

| 文件 | 内容 |
|:-----|:------|
| `references/getnotes-api-reference.md` | API 认证、端点、速率限制、待办提取算法、增量轮询策略、已知陷阱 |
| `references/cursor-pagination-bug-20260511.md` | Cursor分页Bug的完整排查路径、根因分析、修复记录（2026-05-11发现） |
