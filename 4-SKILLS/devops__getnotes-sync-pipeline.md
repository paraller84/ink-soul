---
name: getnotes-sync-pipeline
description: C013 — Get笔记认知同步管道 + 手动查询/检索。自动管道每15分钟轮询同步，手动模式支持MCP检索→本地文件回退的多层灾备策略。
version: 1.1.0
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
```python
# 1. 从 getnotes-cache.json 获取笔记元数据
# 2. 通过 note_id 在 wiki/raw/get-notes/ 查找对应文件
# 3. 直接读取本地文件解析内容

缓存文件: ~/.hermes/cache/getnotes-cache.json
同步文件: ~/wiki/raw/get-notes/          # 格式: YYYYMMDD-<title>-<note_id>.md
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

### 待办提取（通用方法）

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
- **无新笔记** → 静默退出（stdout 为空，cron 不发送消息）
- **有新笔记** → 输出通知消息（cron 自动送达飞书）
- **会议笔记含待办** → 输出具体待办列表，提示用户确认
- **阅读/灵感笔记** → 输出归档路径

### 确认待办
用户收到通知后，回复「确认」或「忽略」即可处理 PENDING 列表。

## 分类规则

| 类型 | 触发条件 |
|:----|:---------|
| 🎤 会议 | `note_type=meeting` 或标题含「会议/讨论/评审/周会/汇报/同步/复盘/规划」 |
| 📖 阅读 | `note_type=link` 或标题含「读书/笔记/摘要/学习/书评/文章」 |
| 💡 灵感 | 标题含「想法/灵感/反思/思考/感悟/随想/心得」 |
| 📝 一般 | 其他 |

## 已知限制
- 首次运行时需手动设置 `last_since_id` 跳过历史笔记
- Get笔记 API 5000次/天配额，96次/天轮询已远低于限制
- 会议笔记详情的 `content` 字段较大（100K+字符），超时已设为120秒

## 参考文档

| 文件 | 内容 |
|:-----|:------|
| `references/getnotes-api-reference.md` | API 认证、端点、速率限制、待办提取算法、增量轮询策略、已知陷阱 |
