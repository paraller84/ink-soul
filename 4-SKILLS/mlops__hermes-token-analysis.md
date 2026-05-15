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
| `references/hardware-evaluation-results.md` | RTX 4070 硬件评估与本地模型性能基准 |
