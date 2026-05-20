---
name: large-context-optimization
description: 大上下文任务优化 — 通过数据库逆向分析识别工具调用反模式（串行读取、调试循环、重复逻辑、碎片命令），应用聚合策略降低50%+ token消耗
tags:
  - large-context
  - token-optimization
  - tool-call-analysis
  - delegate-task
  - fork-on-heavy
  - session-analysis
---

# 大上下文任务优化

当用户报告 token 消耗过高、或会话消息数/工具调用数异常大时使用此技能。通过数据库逆向分析找出工具调用反模式，并应用聚合策略。

## 触发条件

- "token 消耗太高"
- "这个会话太大了"
- "工具调用太多"
- "大上下文任务有什么优化方案"
- 检测到会话消息数 > 80 或工具调用 > 30

## 分析流程

### 1. 定位大上下文会话

```bash
# 找到 token 消耗最大的会话
sqlite3 ~/.hermes/state.db "
SELECT id, datetime(started_at, 'unixepoch', 'localtime') as time,
       message_count, tool_call_count,
       input_tokens, output_tokens, cache_read_tokens,
       estimated_cost_usd
FROM sessions
WHERE date(datetime(started_at, 'unixepoch', 'localtime')) = date('now', 'localtime')
ORDER BY input_tokens DESC
LIMIT 5;
"
```

### 2. 提取消息角色分布

```bash
sqlite3 ~/.hermes/state.db "
SELECT role, COUNT(*) as count,
  ROUND(AVG(LENGTH(content)), 0) as avg_len,
  SUM(CASE WHEN role = 'assistant' AND tool_calls IS NOT NULL THEN 1 ELSE 0 END) as with_tool_calls
FROM messages
WHERE session_id = '<SESSION_ID>'
GROUP BY role;
"
```

预期分布：
- **user**: 少量（5-20）— 用户发起的指令
- **assistant**: 大量（50-250）— 大部分带 tool_calls
- **tool**: 大量（50-250）— 工具返回结果，通常 avg_len 很高（文件内容）

### 3. 提取工具调用类型

```python
from hermes_tools import terminal, read_file

# 提取所有 assistant 消息中的 tool_calls
result = terminal(f"""
sqlite3 ~/.hermes/state.db "
SELECT m.tool_calls FROM messages m
WHERE m.session_id = '<SESSION_ID>'
  AND m.role = 'assistant'
  AND m.tool_calls IS NOT NULL
ORDER BY m.timestamp ASC;
"
""")

# 解析并统计工具名
import json
from collections import Counter

tool_names = []
for line in result["output"].strip().split("\n"):
    try:
        calls = json.loads(line)
        for c in calls:
            name = c.get("function", {}).get("name", "unknown")
            tool_names.append(name)
    except:
        pass

counts = Counter(tool_names)
total = sum(counts.values())
for name, count in counts.most_common():
    pct = count / total * 100
    bars = "█" * int(pct / 3)
    print(f"{name:20s} {bars} {count:3d} 次 ({pct:.0f}%)")
```

## 五大反模式识别

### 🔴 反模式 1：串行读取链

**特征**：连续 5+ 次 `read_file`，每次读不同文件或同一文件的不同偏移

```
read_file(A, offset=1, limit=50)
→ 等结果 → 消化 → read_file(B, offset=1, limit=50)
→ 等结果 → 消化 → read_file(C, offset=1, limit=50)
```

**诊断**：相邻 timestamp 的 read_file 调用，中间无其他工具调用。

**聚合方案**：用单个 `execute_code` 批量读取：

```python
# 一次调用替代 N 次串行调用
from hermes_tools import read_file
files = {
    "file_a": read_file("path/to/a.md", limit=50).get("content", ""),
    "file_b": read_file("path/to/b.md", limit=80).get("content", ""),
    "file_c": read_file("path/to/c.py", limit=50).get("content", ""),
}
print(json.dumps(files, ensure_ascii=False))
```
→ N 次调用 → 1 次，节省 (N-1) 轮上下文切换

### 🔴 反模式 2：调试循环

**特征**：`read_file → patch → terminal(测试) → 出错 → read_file → patch → terminal(测试) → ...` 往复 3+ 轮

**诊断**：时间轴上形成 `[patch, terminal]` 重复模式 3 次以上。

**聚合方案**：用 `delegate_task` 委托子代理独立调试：

```
何时触发：
  ✓ 第二次 patch 同一文件
  ✓ 第三次 terminal 测试同一 pipeline/脚本
  ✓ 工具调用计数 > 20 且 < 50% 为信息获取型

# 创建独立子代理，在隔离上下文完成所有试错
delegate_task(
    goal="调试并修复 <脚本名> 的特定功能",
    context=f"""当前代码路径: <path>
要修复的问题: <具体描述>
测试命令: <可执行的完整命令>
当前代码内容: <代码片段>
已发现的错误: <error info>""",
    toolsets=["terminal", "file"]
)
```
→ 15 轮试错 → 1 次委托，主会话几乎零消耗。子代理只返回修复结果。

### 🔴 反模式 3：重复的 execute_code 逻辑

**特征**：多个 `execute_code` 调用包含大量重复代码（如每次都要读 .env、获取 token、构建请求）

**诊断**：不同 execute_code 调用之间有 60%+ 的代码行重复。

**聚合方案**：将重复逻辑提取为独立脚本，在 execute_code 中只做 import 或调用：

```python
# 替代 3 个 execute_code 调用共 150 行重复代码
terminal("python3 ~/.hermes/scripts/feishu-api.py create-doc --title 'xxx' --folder xxx")
terminal("python3 ~/.hermes/scripts/feishu-api.py download --file-token xxx")
```
→ 减少 60-80% 的代码重复，降低上下文膨胀

### 🔴 反模式 4：微小 terminal 命令碎片化

**特征**：频繁的 1-2 行 terminal 命令（ls, cp, rm, head, ln）

**诊断**：连续 3+ 个 terminal 调用，每个命令长度 < 100 字符。

**聚合方案**：合并为 `"""..."""` 多行脚本：

```python
# 替代 4 次调用
terminal("""
cp fileA fileB && rm fileA
ls ~/edu-hub/scripts/
ln -sf edu_common.py edu_system_common.py
head -11 script.py
""")
```
→ 4 次 → 1 次，节省 3 轮切换

### 🔴 反模式 5：上下文固定开销膨胀（内存注入）

**特征**：每轮消息都要加载全部记忆系统（Memory），其中大量条目是**已过时的历史日志**而非持续有效的事实。

**诊断**：统计记忆文件 `~/.hermes/memories/MEMORY.md` 中：
- **应保留**：架构规则、用户偏好、系统当前状态、工具使用技巧
- **应删除**：单次操作完成记录（「文件夹已创建」「Token 已保存」）、已过时的版本历史（v4.1→v4.5 只需保留 v4.5）、设置过程描述

**聚合方案**：两阶段清洗法

阶段 1 — 识别可删除条目（占记忆的 60-70%）：
```markdown
应删除的条目类型:
  ① "X已完成/已创建" → 一次性操作，通过 session_search 回溯
  ② "v4.1升级" "v4.2升级" "v4.3升级" → 只保留最新版本
  ③ "部署过程中遇到问题X" → 问题已解决，不再需要
  ④ "今日创建了..." → 过了今天就过期
  ⑤ placeholder 条目 → 残留物

应保留的条目类型:
  ① 系统架构规则（「只复制不删除」「全本地模型」）
  ② 当前运行状态（C003 v4.5, ChromaDB 22K向量）
  ③ 工具使用技巧（feishu-md-writer block_type 映射）
  ④ 环境配置（WSL GPU、Ollama 地址）
  ⑤ API 关键陷阱（token 漂移、pipe表格问题）
```

阶段 2 — 直接重写 MEMORY.md 为压缩版本（19K→3K，节省 85%）。

### 🔴 反模式 6：知识检索路径错位

**特征**：需要了解系统时直接 `read_file` 加载整个文件（~27K chars）到上下文，而不是用 ChromaDB 搜索返回最相关的 3-5 个片段（~2K chars）。

**聚合方案**：ChromaDB 优先决策矩阵

```
场景                       优先操作                  回退
了解代码逻辑/架构           kb_retrieve --query       read_file(无结果)
查找关键词/配置             kb_retrieve --query       search_files
查看系统状态                kb_retrieve --stats       read_file
修改代码                   read_file                 无可替代
```

→ 节省 93% 的文件读取输入。

### 🔴 反模式 7：会话连续性税（Conversation Continuity Tax）

**特征**：单次飞书对话的消息数持续增长（50→100→300条），每次用户回复/继续时，整个历史上下文被重新加载。这是**成本最高但最容易被忽视**的税。

**诊断**：

```sql
-- 飞书会话按消息数分组，看成本分布
SELECT 
  CASE 
    WHEN m.msg_count <= 20 THEN '1-20条'
    WHEN m.msg_count <= 50 THEN '21-50条'
    WHEN m.msg_count <= 100 THEN '51-100条'
    WHEN m.msg_count <= 200 THEN '101-200条'
    ELSE '201+条'
  END as msg_range,
  COUNT(*) as sessions,
  ROUND(AVG(s.input_tokens), 0) as avg_input,
  ROUND(AVG(s.input_tokens) / NULLIF(AVG(m.msg_count), 0), 0) as avg_input_per_msg,
  ROUND(SUM(s.input_tokens+s.output_tokens)/1000, 0) as total_k
FROM sessions s
JOIN (SELECT session_id, COUNT(*) as msg_count 
      FROM messages GROUP BY session_id) m ON m.session_id = s.id
WHERE s.source='feishu' AND s.started_at > strftime('%s','now','-7 days')
GROUP BY msg_range ORDER BY MIN(m.msg_count);
```

**典型结果：** 21+条消息的会话贡献了 90%+ 的成本，但数量只占 30%。一条 400 条消息的会话，最后 10 条消息的每次成本是最初 10 条的 20 倍。

**聚合方案：主动会话拆分**

```python
# 模式 A：话题切换时主动 /new
# 用户说"继续上次的XX" → 不要在当前会话继续
# 而是用 session_search 找到历史结论，在新会话中重启

# 模式 B：大 HTML 输出 → 飞书上传发链接（替代在对话中输出全文）
# 例：生成报告后，upload_all 到飞书，只发链接
# 效果：不把几万 chars 的 HTML 代码塞进上下文下一轮
```

**成本收益：**
| 行为 | 当前成本（50轮） | 优化后（每10轮 /new） | 节省 |
|:----|:---------------:|:--------------------:|:----:|
| 50轮对话 | 15-20K 输入/轮，末轮极高 | 8-10K 输入/轮，稳定 | **35-50%** |
| 大HTML回显 | 输出 30-50K → 下一轮输入 +30-50K | 飞书链接 0.2K | **~100%** |

## 五大策略优先级

| 策略 | 代码改动 | 预期节省 | 适用场景 |
|:----|:-------:|:-------:|:--------|
| 1️⃣ 串行读取聚合 | 零 | 15-20% | 任何大上下文任务 |
| 2️⃣ 调试循环委托 | 零 | 35-45% | bug 修复、新功能开发 |
| 3️⃣ 重复逻辑脚本化 | 低（提取脚本） | 8-12% | 重复性 API 操作 |
| 4️⃣ 微小命令合并 | 零 | 3-5% | 文件操作密集型任务 |
| 5️⃣ 记忆瘦身 | 中等（重写 MEMORY.md） | **60-80% 固定开销** | 所有会话，持续性最大 |
| 6️⃣ 知识检索路径优化 | 零（行为习惯） | **93%/次 文件读取** | 需要了解系统时 |
| **7️⃣ 会话连续性税减免** | **零（行为习惯）** | **35-50% 长对话成本** | **飞书对话 ≥50 条消息时** |

## 自动分叉阈值（Fork-on-Heavy）

### 触发条件

当检测到以下信号时，主动推荐使用子代理：

#### 硬阈值
- 消息数 > 50          → 建议使用 delegate_task 分解任务
- 工具调用 > 30        → 自动检查反模式
- 工具调用计数 > 20 且信息获取类 < 50%
- 同一文件被 patch 2 次以上
- 同一 pipeline 的 terminal 测试执行 3 次以上
- 连续 5+ 次 read_file 无 write_file 介入

#### 模式检测（更早触发）
- **正在调试**：检测到 `read_file(某脚本)` → `patch(同脚本)` → `terminal(跑测试)` 的三元组
- **批量读取**：连续 3+ 次 read_file 针对不同文件，无中间输出
- **重复样板**：2+ execute_code 包含相同的 boilerplate（如 Feishu token 获取）
- **📞 会话连续性税**：飞书会话消息数 ≥ 80 且输入/输出比 > 3:1 → 建议用户 /new 新会话继续

### 会话健康度指标

```
工具调用密度 = 工具调用数 / 消息总数
  健康: < 0.3
  警告: 0.3-0.5
  危险: > 0.5  → 应触发分叉

调试循环深度 = 同一文件的 patch+terminal 对数
  健康: < 2
  警告: 2-4
  危险: > 4  → 应触发分叉

读取效率 = read_file 次数 / 独立文件数
  健康: 1.0 (只读了一次)
  危险: > 1.3 (重复读取)
```

### 不要分叉的场景
- 需要用户交互确认的任务（子代理不能调用 clarify）
- 必须保持严格因果关系的步骤
- 极短任务（总调用 < 10）

### 分叉失败时的回退
如果 delegate_task 返回空/错误，自动回退到串行模式，输出告警。

### 成本收益估算

| 场景 | 优化前调用 | 优化后调用 | 节省 |
|:---|:---:|:---:|:---:|
| 调试循环（15轮） | 45 调用 / 120K 输出 | 1 + 子代理内部 | ~80% |
| 批量读取（6文件） | 6 调用 / 12K 输出 | 1 调用 / 3K 输出 | ~75% |
| 重复逻辑（3次） | 3×50 行代码 | 3 行命令 | ~60% |
| 碎片命令（4次） | 4 调用 / 4K 输出 | 1 调用 / 1K 输出 | ~75% |

## 预期效果（综合应用全部 6 种优化）

基于 4 月 27 日全系统优化会话的实测数据：

| 指标 | 优化前 | 优化后 | 节省 |
|:---|---:|---:|:---:|
| 记忆系统大小 | 19K | 2.8K | **-85%** |
| 每轮固定上下文开销 | ~25K | ~8K | **-68%** |
| 100 轮会话记忆开销 | 1.9M | 280K | **-1.6M chars** |
| C005 子代理通信/次 | ~12.5K | ~4.5K | **-64%** |
| 文件读取输入/次 | ~27K | ~2K | **-93%** |
| 工具调用数（大任务） | 112 | ~45 | **-60%** |
| 消息数（大任务） | 248 | ~100 | **-60%** |
| 全系统货币成本/月 | $0.136 | ~$0.08 | **-41%** |

## 验证方法

优化前后对比：

```bash
# 优化前基准
sqlite3 ~/.hermes/state.db "
SELECT SUM(tool_call_count), SUM(message_count),
       SUM(input_tokens), SUM(output_tokens)
FROM sessions WHERE ...;
"

# 优化后执行同类任务后再次查询对比
```

## 注意事项

- 串行读取聚合的**前提**：读取的文件之间无依赖关系
- 调试循环委托的**限制**：子代理不能使用 `clarify`，需在 context 中给足信息
- 不要对已完成的会话应用优化——这是**预防性策略**，用于**下一个**大上下文任务
- 本技能与 `hermes-token-analysis` 互补：该技能负责成本统计，本技能负责行为优化

## 参考文档

- [DeepSeek-V4 1M 上下文配置指南](references/deepseek-v4-1m-config.md) — Hermes 配置、缓存定价策略、路由层级设计、前代模型对比
