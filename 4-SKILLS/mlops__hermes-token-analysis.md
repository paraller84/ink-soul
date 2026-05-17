---
name: hermes-token-analysis
description: 分析Hermes Agent的token消耗情况，包括数据库查询、成本计算、报告生成、追踪体系搭建和异常告警
tags:
  - hermes
  - token
  - cost
  - analysis
  - billing
  - usage
  - dashboard
---

# Hermes Token消耗分析与追踪体系

Three sub-skills, invoked by task type:

1. **ANALYZE** — query spending, run ad-hoc cost checks
2. **SETUP** — build the tracking infrastructure from scratch
3. **MAINTAIN** — handle anomalies, pricing changes, script issues

---

## 1. ANALYZE — 查询分析

当用户询问当前/历史Token消耗时使用。

### 快速查询

```sql
-- 最近n条会话（含cost）
SELECT id, datetime(started_at,'unixepoch') as time, source, title, billing_provider, model, input_tokens, output_tokens, ROUND(estimated_cost_usd,6) as cost FROM sessions ORDER BY started_at DESC LIMIT 20;

-- 按日汇总（最近14天）
SELECT date(datetime(started_at,'unixepoch')) as day, COUNT(*) as sessions, SUM(input_tokens+output_tokens) as total_tokens, ROUND(SUM(COALESCE(estimated_cost_usd,0)),4) as cost FROM sessions WHERE started_at > strftime('%s','now','-14 days') GROUP BY day ORDER BY day DESC;

-- 按模型统计
SELECT billing_provider, model, COUNT(*) as cnt, SUM(input_tokens+output_tokens) as total_tokens, ROUND(SUM(COALESCE(estimated_cost_usd,0)),4) as cost FROM sessions GROUP BY billing_provider, model ORDER BY cost DESC;

-- 按来源统计（飞书/终端CLI/cron）
SELECT source, COUNT(*) as sessions, SUM(input_tokens+output_tokens) as tokens, ROUND(SUM(COALESCE(estimated_cost_usd,0)),4) as cost FROM sessions WHERE source IS NOT NULL GROUP BY source ORDER BY cost DESC;
```

### 任务分类与多维分析

通过 `title` 字段对会话自动归类，支持 `re` 关键词匹配。分类规则见 `token-daily-report.py` 中的 `CATEGORIES` 字典。输出按来源（飞书/终端/cron）和任务类别（英语模块/语文模块/日常对话等）分组。

### 成本计算

```python
cost = (input_tokens / 1_000_000 * input_price) + (output_tokens / 1_000_000 * output_price)
```

定价参考见 `references/deepseek-pricing-verified.md`。

### 异常检测规则（内置于日报告）

| 规则 | 阈值 | 触发动作 |
|:-----|:-----|:---------|
| 日Token环比突增 | >30% | 标记 + 系统脉搏告警 |
| 单会话Token超限 | >500K | 记录上下文详情 |
| 凌晨活跃 | 02:00-06:00 >100K | 检查cron是否失控 |
| 成本单日超预算 | >$5/day | 推送告警到运维群 |

---

## 2. SETUP — 基础设施搭建

当需要建立/修复 Token 追踪系统时使用。

### 2.0 整体架构（四层）

```
L4 决策层: 模型路由优化 → Cron频率调优 → 预算控制
L3 展示层: Feishu仪表盘 → 系统脉搏集成 → 运营监控周报
L2 分析层: 日分析脚本(23:55) → 周趋势报告(周日20:00) → 异常检测
L1 数据层: 成本补丁 → 任务标签 → Cron Token追踪
```

### 2.1 定价修复（最多执行的步骤）

**根因**：`sessions` 表的模型名与 `agent/usage_pricing.py` 的定价表不匹配。例如会话存 `deepseek-v4-flash` 但定价表用 `deepseek-chat`。

**修复流程**：
1. 在 `agent/usage_pricing.py` 的 `_OFFICIAL_DOCS_PRICING` 中添加别名条目（见 `references/deepseek-pricing-alias-fix.md`）
2. 运行回溯脚本：`cd ~/.hermes/scripts && python3 token-cost-backfill.py`

### 2.2 部署自动报告

```bash
# 步骤1: 创建 no_agent cron wrapper 脚本
# token-daily-report.sh / token-weekly-report.sh
# 内容: cd /home/openclaw/.hermes/scripts && exec python3 <script>.py

# 步骤2: 注册 cron（no_agent=true，deliver=feishu:oc_xxx）
# 日报告: 55 23 * * *
# 周报告: 0 20 * * 0
# 投递目标: 🔧系统运维部群 chat_id

# 步骤3: 端到端验证
cd ~/.hermes/scripts && python3 token-daily-report.py
```

### 2.3 创建 Feishu 仪表盘

```python
# 1. 创建文档 → 获取 doc_token
# 2. 写入 Markdown → feishu-md-writer.py（注意：必须从 ~/.hermes/scripts 目录运行）
# 3. 共享给用户 open_id
# ⚠️ 陷阱：feishu-md-writer.py 被 subprocess.run 调用时可能空写入
#   从终端直接运行 python3 feishu-md-writer.py <doc_token> <md_path> 才可靠
```

### 2.4 集成现有 Cron

- **系统脉搏**（17:15）：prompt 中加入 Token 前置查询行
- **运营监控周报**（周一06:00）：prompt 中加入 Token 分析章节

### 2.5 创建项目文件夹

```
Hermes生成文件夹 → 项目 → Token管理体系/
  文档: ⚡Token消耗全景仪表盘 vN
  设计: token-tracking-system-design-vN.html
```

### 2.6 部门职责分配

当用户说"由XX部负责执行"时：
1. 确认该部门对应的 Feishu 群 chat_id
2. 所有 cron 投递指向该群
3. 脚本自动运行，部门只在异常时介入
4. 日常维护：定价更新、异常响应、周报检查

### 2.7 模型分流（L1简单对话 → 本地模型）

当用户确认仅启用 L1 路由时：
1. 确认硬件评估结果（`references/hardware-evaluation-results.md`）
2. 选择 qwen2.5:3b（1.9GB, 132 tok/s）作为简单对话模型
3. 调用方式：`curl -s http://localhost:11434/api/generate -d '{"model":"qwen2.5:3b","prompt":"...","stream":false}'`
4. 将路由规则写入 MEMORY.md 的 `=== L1模型路由规则 ===` 段
5. 规则清晰定义触发条件（问候/确认/打卡/简单询问/3句话以内）和排除条件（编码/分析/多步/工具调用）

⚠️ 8GB VRAM 不能同时加载两个模型。L2/3 级路由需要用户额外确认。

---

## 3. MAINTAIN — 运维与故障处理

### 成本数据丢失

**症状**：新会话的 `estimated_cost_usd` 为 0。

**排查路径**：
1. 检查 `agent/usage_pricing.py` 中是否有该模型的定价条目
2. 模型名精确匹配（`_lookup_official_docs_pricing` 做字符串相等）
3. 如果有新模型上线但没有定价 → 添加别名（参考 `references/deepseek-pricing-alias-fix.md`）

### 日报告不投递

**检查事项**：
1. cron job 是否启用（`cronjob list` → 看 `enabled` 状态）
2. no_agent 脚本是否可执行（`bash token-daily-report.sh` 测试）
3. wrapper 脚本路径是否正确（no_agent cron 不支持参数，需要 .sh 包装）
4. 投递 chat_id 是否正确（群名变更后需要更新）

### 凌晨活跃误报（时区偏移）

**症状**：日报告标记 `[凌晨活跃] 02:00-06:00 消耗 1.XM`，但实际分析发现该时段全是白天会话。

**根因**：数据库存 UTC 时间戳，但脚本定义凌晨窗口时直接取 `today_start.replace(hour=2)`（UTC 02:00），而标签写「凌晨 02:00」（CST）。UTC 02:00 = CST 10:00，是正常工作时间。

**排查路径**：
1. 用 `+8 hours` 转换查 CST 真实时段：`datetime(started_at,'unixepoch','+8 hours')`
2. 对比脚本的 `t1/t2` 定义与标签文字是否一致
3. 看该时段会话的 `source` 和 `title` → 如果是正常对话标题（"高质量题库准备策略 #17"等）而非 cron 任务 → 确认是误报

**修复**：见 `references/timezone-bug-detection.md`

**一次性根治**：所有报告脚本中与「时/日/周」相关的窗口计算，都应在注释中标注时区并做 CST 偏移。核心原则：**数据库时间戳是 UTC，业务逻辑是 CST，两者之间必须显式转换。**

### 仪表盘内容为空

**根因**：`feishu-md-writer.py` 从 Python `subprocess.run` 调用时可能因工作目录错误导致空写入。

**修复**：直接从终端运行 `cd ~/.hermes/scripts && python3 feishu-md-writer.py <doc_token> <md_path>`

### 费用基线（部署参考）

截至 2026-05-15，全部 718 会话累积成本 $30.84：
- DeepSeek V4 Flash: 493会话, 29.9M tokens, $21.26
- DeepSeek Reasoner: 211会话, 14.1M tokens, $9.58
- 本地 Ollama: 3会话, 0.15M tokens, $0.00

详细基线数据见 `references/hermes-token-system-baseline-20260515.md`。

---

## 4. OPTIMIZE — 上下文税诊断与优化

当用户要求分析Token优化（超出模型路由策略的范围）时使用。核心思路：**将「上下文税」作为首要分析对象**，而非仅关注模型单价。

### 4.1 工作流：前置验证

**在提出任何优化建议之前，必须先验证该优化是否已被实施过：**

```
Step 1: 用 memory_tencentdb_conversation_search 搜索相关关键词
        → "token优化" "压缩" "上下文" "系统提示词" "SOUL" "MCP" "瘦身"
Step 2: 检查对应技能中是否已有相关方案
        → skill_view('hermes-token-analysis') 看 OPTIMIZE 章节
        → skill_view('agent-memory-management') 看是否已做记忆瘦身
Step 3: 对比今天的提案和历史提案的异同
        → 如果完全重叠 → 告知用户已存在，问是否要执行
        → 如果部分重叠 → 标注差异并更新方案
        → 如果全新 → 正常提案
```

**陷阱**：用户会在不同时间反复提出同类优化需求。不做前置验证就重复提案，会让用户觉得你健忘或不认真。

### 4.2 诊断1：I/O比分析（核心指标）

这是判断「上下文税是否严重」的最直接指标。**3:1以下属于正常，20:1以上属于严重上下文税。**

```sql
-- 按日I/O比（近7天）
SELECT date(datetime(started_at,'unixepoch')) as day,
       ROUND(AVG(1.0*input_tokens/NULLIF(output_tokens,0)),2) as io_ratio,
       ROUND(SUM(input_tokens)/1000,0) as input_k,
       ROUND(SUM(output_tokens)/1000,0) as output_k
FROM sessions WHERE started_at > strftime('%s','now','-7 days')
GROUP BY day ORDER BY day DESC;

-- 按会话长度分组（飞书）
SELECT 
  CASE 
    WHEN message_count <= 2 THEN '1-2条'
    WHEN message_count <= 5 THEN '3-5条'
    WHEN message_count <= 10 THEN '6-10条'
    WHEN message_count <= 20 THEN '11-20条'
    ELSE '21+条'
  END as msgs_range,
  COUNT(*) as cnt,
  ROUND(AVG(input_tokens+output_tokens),0) as avg_tokens,
  ROUND(SUM(COALESCE(estimated_cost_usd,0)),4) as cost
FROM sessions 
WHERE started_at > strftime('%s','now','-7 days') AND source='feishu'
GROUP BY msgs_range ORDER BY MIN(message_count);
```

### 4.3 诊断2：系统提示词大小

```sql
-- 查看系统提示词长度分布（近7天最长5个）
SELECT id, datetime(started_at,'unixepoch') as t, 
       LENGTH(system_prompt) as sys_len_char,
       input_tokens, output_tokens,
       ROUND(COALESCE(estimated_cost_usd,0),4) as cost
FROM sessions 
WHERE started_at > strftime('%s','now','-7 days') AND system_prompt IS NOT NULL
ORDER BY sys_len_char DESC LIMIT 5;

-- 读取具体的 SOUL.md 文件内容
cat ~/.hermes/SOUL.md | wc -c
```

**解读**：52K字符 ≈ 13K tokens。其中 SOUL.md（~7K字符的文学描述）是纯装饰性内容，对日常对话无价值。

### 4.4 诊断3：压缩机制触发率

```sql
-- 会话结束原因分布
SELECT COALESCE(end_reason,'NULL') as reason, COUNT(*) as cnt, 
       ROUND(AVG(input_tokens+output_tokens),0) as avg_t
FROM sessions WHERE started_at > strftime('%s','now','-7 days')
GROUP BY reason ORDER BY cnt DESC;
```

**解读**：
- `compression` 条数少 → 压缩阈值太高，绝大多数会话从未触发压缩
- `session_reset` 条数多 → 用户在用 `/new` 主动拆分，这是好习惯
- `NULL` 条数多 → 会话被丢弃浪费，应分析原因

### 4.5 实施1：SOUL.md 精简

**效果**：每次会话省1,600+ tokens（52K字符 → 300字符精简版）。月省约$2-3。

```markdown
# 推荐的精简版 SOUL.md
# Hermes Agent Persona

你是夜雨（Yè Yǔ）的 AI 助手，专注高效完成任务。
风格：结构化、表格化、直接。
偏好：只复制不删除，骨架先行，一次性全量。

重要参考信息见 MEMORY 和 USER PROFILE 文件。
```

**位置**：`~/.hermes/SOUL.md`

**生效方式**：该文件每次对话都会被读取，修改后立即生效（无需重启）。

**⚠️ 注意**：保留HTML注释块（`<!-- ... -->`）和第一行 `# Hermes Agent Persona`，Hermes 依赖这些标记解析 persona 文件。

### 4.6 实施2：压缩阈值调优

**效果**：将触发点从250K降到100K，使大部分长会话都能触发压缩。月省约$3-5。

**位置**：`~/.hermes/config.yaml` 的 `compression` 段

```yaml
compression:
  enabled: true
  threshold: 0.10         # 原0.25，改为0.10（100K触发）
  target_ratio: 0.15      # 原0.20，改为0.15（更激进压缩）
  protect_last_n: 15      # 原20，保持最近15轮
  hygiene_hard_message_limit: 400  # 保持不动
```

**生效方式**：需要新会话（`/new`），因压缩相关配置在会话启动时快照。

### 4.7 实施3：MCP工具Schema裁剪

**效果**：每次会话省1-2K tokens。53个工具的JSON schema描述是系统提示词膨胀的主要来源之一。

**原理**：Hermes 的 `mcp_config.py:682-744` 支持 `tools.include` / `tools.exclude` 字段，可在 config.yaml 中为每个 MCP 服务器指定白名单/黑名单。

**操作步骤**：

```bash
# 1. 列出所有已注册MCP工具，确认高频使用的
hermes mcp list

# 2. 编辑 config.yaml，在 MCP 服务器配置中添加 tools.include
# 位置：~/.hermes/config.yaml
# 格式：
#   mcp_servers:
#     token-savior:
#       command: ...
#       # ...其他配置...
#       tools:
#         include:
#           - skill_view
#           - skills_list
#           - skill_manage
#           # ...仅挑选高频工具

# 3. 建议保留的核心工具（2026-05-16已验证）
# skill类: skill_view, skills_list, skill_manage
# 记忆类: memory, memory_tencentdb_memory_search, memory_tencentdb_conversation_search
# 编码类: execute_code, read, glob, grep
# 文件类: file_write, file_edit, edit
# 网络类: web_search, web_fetch
```

**⚠️ 注意**：`hermes mcp configure` CLI 可能不存在——确认操作方式的方法是直接编辑 `~/.hermes/config.yaml`。

**生效方式**：需要新会话（`/new`），MCP 配置在会话启动时快照

### 4.8 实施4：长会话主动拆分（行为习惯）

**效果**：月省$2-4。21+条飞书会话占98%的成本。

| 习惯 | 说明 |
|:-----|:------|
| 话题切换用 `/new` | 避免历史消息累积 |
| 看到会话>20条时主动 `/new` | 15-20条是最佳平衡点 |
| Git操作前 `/new` | 防止git输出膨胀上下文 |

### 4.9 实施5：Cron任务 no_agent 化（可选）

**效果**：月省$0.5-1。当前cron任务虽单次成本低但频次高（223次/周）。

当前已实施的：日报告、周报告使用 no_agent cron。检查其他cron是否也能迁移。

---

## 附录：优化方案总结表

| # | 方案 | 月省估算 | 难度 | 生效方式 |
|:-:|:-----|:--------:|:----:|:---------|
| 1 | SOUL.md精简 | $2-3 | ⭐ | 立即生效 |
| 2 | 压缩阈值调优 | $3-5 | ⭐ | 新会话生效 |
| 3 | MCP工具裁剪 | $1-2 | ⭐⭐ | 新会话生效 |
| 4 | 长会话 /new 习惯 | $2-4 | ⭐ | 行为习惯 |
| 5 | Cron no_agent化 | $0.5-1 | ⭐⭐ | cron编辑 |

---

## 定价参考

### DeepSeek

| 模型 | 输入 $/M | 输出 $/M |
|:-----|:--------:|:--------:|
| V4 Flash (= deepseek-chat) | $0.14 | $0.28 |
| Reasoner (R1) | $0.55 | $2.19 |

### 本地 Ollama（零成本）

qwen3.5-256k:latest, qwen3:8b, qwen2.5:3b, qwen3-vl:8b — 均在本地 GPU 免费运行。

---

## References

| 文件 | 内容 |
|:-----|:------|
| `references/deepseek-pricing-verified.md` | 完整定价表 + 计算公式 |
| `references/deepseek-pricing-alias-fix.md` | 模型名别名修复原理与排查路径 |
| `references/hermes-token-system-baseline-20260515.md` | 部署基线快照（数据库/消耗/趋势） |
| `references/task-category-classifier.md` | 会话任务分类体系（CATEGORIES字典 + classify_task函数） |
| `references/daily-conversation-optimization.md` | 日常对话优化分析（上下文税问题 + 三档优化方案） |
| `references/context-tax-optimization-example-20260516.md` | 上下文税优化示例（含完整诊断流程与数据） |
| `references/timezone-bug-detection.md` | UTC/CST 时区偏移导致「凌晨活跃」误报的诊断与修复 |
| `references/hardware-evaluation-results.md` | RTX 4070 硬件评估与本地模型性能基准 |
