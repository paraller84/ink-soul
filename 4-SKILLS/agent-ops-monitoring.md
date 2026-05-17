---
name: agent-ops-monitoring
version: 2.0.0
description: >-
  Build an AI Agent operations monitoring system with an embedded advice engine.
  Collects 6-dimension runtime metrics, generates auto-prioritized optimization
  recommendations (P0/P1/P2), produces weekly Feishu reports, and syncs system
  documentation from the capability registry to cloud docs.
author: Hermes Agent
tags:
  - devops
  - monitoring
  - self-optimization
  - observability
  - agent-ops
category: devops
trigger: |
  User needs to monitor AI Agent runtime health, track token costs,
  generate weekly operations reports, or auto-detect optimization opportunities.
---

# Agent 运营监控系统 — 构建指南

## 适用场景

需要持续监控 AI Agent 系统的运行状况并自动生成优化建议时使用。特别适用于：
- 多系统（C000-C00x）共存的 Agent 架构
- 需要跟踪 token 消耗、成本、会话趋势
- 希望通过数据驱动而非凭感觉进行系统优化
- 需要自动化周报和预警

## 5 步构建法

### Step 1: 定义采集维度

| 维度 | 数据源 | 关键指标 |
|:----|:-------|:---------|
| 📊 会话 | state.db (SQLite) | 总/今会话数、输入/输出 token、成本、大上下文预警 |
| 🏗️ 系统 | capability-registry.md | 各系统 active/needs-repair 数量 |
| 📜 脚本 | 文件系统 + py_compile | 文件数、总行数、语法错误详情 |
| 🔍 质量 | 应用日志 (如 c003-activity.log) | 24h ERROR/WARN 计数 |
| 💾 资源 | df / du | 磁盘/内存/向量库/Wiki 大小 |
| ⏰ Cron | cronjob list API | 运行状态/最近失败 |

### Step 2: 设计建议引擎规则

每条建议包含 4 个字段：`priority`, `area`, `finding`, `suggestion`, `metric`。详见下方「优化建议引擎」节。

### Step 3: 采集实现

见下方「数据采集：6 维度」节。

### Step 4: 历史趋势对比

详见「优化建议引擎 → 历史趋势分析」节。

### Step 5: 集成到 Cron + 通知

详见「Cron 设置」节。

## 核心架构

监控 AI Agent 系统的运行状态，提供数据驱动的优化建议，并通过飞书文档保持信息同步。

## 核心架构

```
┌─────────────────────────────┐
│  数据采集 (6维度)            │
│  session / system / script  │
│  quality / resource / cron   │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│  优化建议引擎                 │
│  规则分析 → P0/P1/P2 排序    │
│  + 历史趋势对比               │
└──────────┬──────────────────┘
           ▼
┌─────────────────────────────┐
│  输出层                      │
│  ├─ 飞书周报（每周一 6AM）   │
│  ├─ 系统文档同步（每月1日）   │
│  └─ 本地 JSON 报告           │
└─────────────────────────────┘
```

## 组件清单

| 组件 | 脚本 | 功能 |
|:----|:-----|:-----|
| 采集+建议 | `ops-monitor.py` | 4 种模式: `advise`(采集+建议), `all`(仅采集), `health`(健康检查), `history`(趋势) |
| 飞书周报 | `ops-report-to-feishu.py` | 运行 monitor → 解析输出 → 构建 markdown → 写入飞书 |
| 文档同步 | `ops-sync-feishu-docs.py` | 读取能力注册表 → 构建系统说明 → 写入飞书 |

## 数据采集：6 维度

### 1. 会话 (state.db)
```sql
-- 核心查询
SELECT COUNT(*) as total_sessions, SUM(input_tokens), SUM(output_tokens),
       ROUND(SUM(estimated_cost_usd), 4) as total_cost
FROM sessions;

-- 大上下文预警 (>100K input)
SELECT COUNT(*) FROM sessions WHERE input_tokens > 100000;

-- 云端 vs 本地
SELECT estimated_cost_usd > 0.0001 as is_cloud, COUNT(*), AVG(input_tokens)
FROM sessions GROUP BY is_cloud;
```

### 2. 系统 (capability-registry.md)
从表格中提取 `| **ID** | Cxxx |` 和 `| **状态** | \`active\` |`。注意 C005 可能使用 `| ID | C005 |`（无加粗标记），需用 `\*?\*?` 兼容两种格式。

### 3. 脚本 (文件系统 + py_compile)
遍历脚本目录，对每个 `.py` 文件执行语法检查：
```python
import py_compile
try:
    py_compile.compile(filepath, doraise=True)
except py_compile.PyCompileError:
    # 语法错误
```

### 4. 质量 (日志文件)
```bash
grep "\[ERROR\]" edu-hub/logs/c003-activity.log | grep "$(date +%Y-%m-%d)" | wc -l
```

### 5. 资源 (df / du)
```bash
df -h / | tail -1  # 磁盘
free -h | grep Mem  # 内存
du -sh ~/.hermes_tools/knowledge_base/vector_store/  # ChromaDB
```

### 6. Cron (cron list API)
通过 `cronjob --action list` 获取运行状态。

## 优化建议引擎

### 规则设计

每条建议包含 4 个字段：`priority`, `area`, `finding`, `suggestion`, `metric`

| 优先级 | 条件 | 示例建议 |
|:-----:|:-----|:---------|
| 🔴 P0 | 语法错误 > 0 | 对应脚本完全不可用，需立即修复 |
| 🔴 P0 | 系统 needs-repair > 0 | 查看注册表并安排修复 |
| 🔴 P0 | 磁盘 > 80% | 清理无用文件 |
| 🔴 P0 | 24h ERROR > 5 | 查看日志定位根因 |
| 🟡 P1 | 大上下文会话 > 3 | 对 >50 消息会话主动分叉 |
| 🟡 P1 | 记忆文件 > 5KB | 清理过时历史条目 |
| 🟡 P1 | 输出/输入比 < 15% | 上下文有大量无效输入 |
| 🟡 P1 | 日均会话 > 20 | 检查是否有不必要的重复操作 |
| 🟢 P2 | 云端成本 > $0.05 | 当前成本低，可关注趋势 |
| 🟢 P2 | 24h WARN > 10 | 警告较多但非致命 |

### 历史趋势分析

从 `ops-history.json`（30 天滚动）加载历史数据，对比近期趋势：
```python
recent = [h for h in history[-7:] if h.get("today_sessions", 0) > 0]
avg_sess = sum(h["today_sessions"] for h in recent) / len(recent)
```

## 飞书文档同步

能力注册表 → 飞书「我的系统」目录的同步，由 `ops-sync-feishu-docs.py` 负责。
详细流程（包括内容模板、字段提取、验证步骤）见 `capability-registry-management` skill 的「同步云文档」节。

注：系统文档内容按「基本逻辑 + 使用手册」结构生成，非本文档涵盖范围。

## Cron 任务通知治理

### 三阶通知策略

根据任务类型和用户需求，将 cron 任务分为 3 个通知层级：

| 层级 | 行为 | 适用场景 | 配置方式 |
|:----|:-----|:---------|:---------|
| 🔇 **静默** | 不推送任何通知，结果存本地 | 纯后台批处理（例行解析/同步/出题） | `deliver: local` |
| ✅ **决策触发** | 仅新内容/异常时推送 | 需要用户知道但无需决策 | Prompt 内置空跑检测逻辑 |
| 🔔 **决策必需** | 每次运行都推送 | 需要用户确认/决策/干预 | `deliver: feishu:oc_xxx` |

### 空跑静默模式

**问题**：扫描类任务（邮件检查、内容解析、文件同步）经常空跑——脚本正常执行但无新内容。每次都推送空跑结果等于噪声。

**方案**：
1. **Agent 任务**：在 prompt 末尾加「空跑判定」段——检查脚本 stdout 或状态文件，若无有效产出则回复 "无新内容" 并停止，不发送群消息。
2. **no_agent 脚本**：脚本逻辑判断后只输出结果；**空 stdout = 静默不发送**。Hermes cron 对 no_agent 任务只传送 stdout 内容，空输出跳过投递。

```yaml
# no_agent 空跑静默示例
jobs:
  - name: "邮件扫描"
    script: "email-classifier-scan.sh"
    no_agent: true       # 空 stdout = 不发送
    deliver: "feishu:oc_xxx"

# 脚本模式：有新邮件 → print 摘要，无新邮件 → print 空
if [ -n "$NEW_MAILS" ]; then
  echo "发现 $COUNT 封新邮件：$SUBJECTS"
fi
# 无输出时 cron 自动静默
```

### 异常即时告警模式

**问题**：任务失败时用户无法及时得知，直到下次手动检查。

**方案**：用 no_agent 脚本 + 状态文件（`anomaly-state.json`）+ 高频 cron 实现零成本的异常扫描。

```
┌─────────────────────────────────────┐
│ cron-anomaly-monitor.py (every 5m)  │
│                                     │
│  → 读取 ~/.hermes/cron/jobs.json     │
│  → 对比 anomaly-state.json          │
│  → 新异常 → print 告警文本（投递）     │
│  → 异常恢复 → print 恢复通知          │
│  → 无变化 → 空 stdout（静默）        │
└──────────────┬──────────────────────┘
               ▼
        🔧系统运维部群 (feishu:oc_xxx)
```

**成本**：每5分钟一次纯 Python 脚本执行，0 token 消耗。

### 双频调度模式

**问题**：某些任务在工作日和非工作日的执行频率应有差别。Get笔记在工作日更新频繁需每小时扫描一次，周末几乎无更新每8小时足够。单一 cron 表达式无法满足。

**方案**：将一个任务拆分为两个 cron job，分别监听工作日和非工作日时间线。

```
# 工作日（每1小时 at :00）
cronjob create \
  --schedule "0 * * * 1-5" \
  --script "getnotes-pipeline.py" \
  --name "管道名称(工作日)"

# 非工作日（每8小时 at :00）
cronjob create \
  --schedule "0 */8 * * 0,6" \
  --script "getnotes-pipeline.py" \
  --name "管道名称(非工作日)"
```

**注意**：周日凌晨00:00两个job可能同时触发，脚本应幂等，双重运行无副作用。

### 任务生命周期管理：长期空跑清理

**问题**：后台扫描/同步类任务（邮件扫描、内容同步、文件同步）长期无新内容产出时，持续运行等于空转消耗。用户会要求「长时间空跑的任务进行清除」。

**诊断方法**：检查 `~/.hermes/cron/output/<job_id>/` 目录下最近几条输出：
- **有实际产出**（新文件、新数据变更）→ 保留
- **持续0增量**（「New: 0」「无新文档」「共同步 0 个文件」）→ 标记为空跑候选

**清理决策矩阵**：

| 特征 | 处理方式 |
|:----|:--------|
| 连续N次空跑 + 较低频 | 直接清除 (`cronjob remove --job_id xxx`) |
| 连续N次空跑 + 高频 | 降频后观察；或直接清除 |
| 有监控/安全检查价值 | 保留但降频（如日→周） |
| 无任何使用预期 | 直接清除 |

**典型空跑候选**：
- 内容解析类：扫描上游文件夹，长期无新文档（英语内容解析、数学原始题解析）
- 文件同步类：目标目录无变化（hermes-kb-sync、飞书→Wiki同步）
- 知识库管线：文件数固定无变更（kb-wiki-sync增量检测）

核心逻辑：
- 新异常（`last_status=error` 且之前未报告）→ 立即推送
- 异常恢复（之前 error，现在 ok）→ 推送恢复通知
- 持续异常（已报告过且未恢复）→ 不重复推送
- 全部正常 → 静默

```python
# 状态文件 schema (~/.hermes/cron/anomaly-state.json)
{
  "known": {
    "<job_id>": {
      "name": "任务名称",
      "error": "错误信息摘要",
      "first_seen": "2026-05-17 08:00:00",
      "reported": true,
      "resolved": false
    }
  },
  "last_check": "2026-05-17 08:05:00"
}
```

**成本**：每5分钟一次纯 Python 脚本执行，0 token 消耗。

### 多渠道分发

Hermes cron 支持 fan-out delivery 语法 `deliver: "origin,feishu:oc_xxx,feishu:oc_yyy"`——将同一份内容同时投递到多个目标（如 DM + 部门群）。

示例——每日报告同时发 DM 和 系统运维部：
```yaml
jobs:
  - name: "🧠 系统脉搏"
    deliver: "origin,feishu:oc_3b372c56..."
```

### 通知矩阵检查清单

做 cron 通知配置时，逐任务检查：

- [ ] 这个任务用户需要决策吗？→ 投 DM 或相关工作群
- [ ] 这个任务影响用户行为判断吗？→ 投 DM
- [ ] 这个任务是纯后台处理吗？→ 静默
- [ ] 这个任务有空跑场景吗？→ 加空跑判定
- [ ] 这个任务失败需要告知吗？→ 加异常告警覆盖
- [ ] 同一个报告需要发给多人/多群吗？→ fan-out

## Cron 设置

### 基础模式（全 agent，优先用于周报/月报等需要AI分析的场景）

```bash
# 每周一 6AM 生成飞书周报
cronjob create \
  --schedule "0 6 * * 1" \
  --script "ops-report-to-feishu.py" \
  --name "运营监控-周报"

# 每月 1 日 5AM 同步系统文档
cronjob create \
  --schedule "0 5 1 * *" \
  --script "ops-sync-feishu-docs.py" \
  --name "系统文档-飞书同步"
```

### 无代理模式（no_agent，推荐用于高频健康检查）

Hermes v0.13.0+ 支持 `no_agent` cron 模式：脚本直接运行，空 stdout 表示正常（静默），有输出才通知。

适合场景：端口存活检查、API 响应检查、磁盘空间检查等纯脚本监控。

```yaml
# 在 cron.yaml 或 cronjob 中配置
jobs:
  - name: "教育系统-健康检查"
    schedule: "*/30 * * * *"
    script: "check-edu-hub-health.sh"
    no_agent: true      # 静默模式，仅异常时通知
```

脚本示例 (`check-edu-hub-health.sh`)：
```bash
#!/bin/bash
# 服务存活性检查 — no_agent 模式下，空 stdout = 静默正常
PORTS=(5002)
for port in "${PORTS[@]}"; do
    if ! curl -sf http://localhost:$port/ > /dev/null 2>&1; then
        echo "❌ edu-hub 端口 $port 无响应"
    fi
done
```

**收益对比**：

| 模式 | 每次运行 | 每日运行48次 | 适合场景 |
|:----|:--------:|:------------:|:--------|
| 全 agent | ~$0.005 + 5K token | ~$0.24 | 周报/分析报告 |
| no_agent | 0 token | $0 | 存活性/API/磁盘检查 |

## 实战效果

基于首次实施的经验数据：

| 指标 | 效果 |
|:----|:-----|
| 采集维度 | 6 个 |
| 建议分类 | 3 级（P0/P1/P2） |
| 历史窗口 | 30 天滚动 |
| 运行成本 | $0（纯本地 Python） |
| Cron 频率 | 每周一次 |
| 首次运行发现 | 7 次大上下文预警（P1） |

## 关键经验

1. **SQLite 数据可能为空** — `sqlite3 -json` 返回空字符串而非有效 JSON，需检查 `r.stdout.strip()` 是否为空
2. **注册表格式不统一** — 早期系统使用 `| **字段** |` 加粗格式，C005 使用 `| 字段 |` 纯文本格式，需用 `\*?\*?` 兼容
3. **段末分隔符不一致** — C000-C004 段后接 `<!-- =` 注释，C005 在文件末尾无分隔符，需用 `|\Z` 备选
4. **名称字段位置不同** — C000-C004 的 `**名称**` 在第二行，C005-C006 的 `名称` 可能在第三行。用行号不可靠，必须用字段名搜索
5. **Docstring 转义** — patch 工具写入文件时可能转义 `"""` 为 `\"\"\"`，修复方法：`sed -i 's/\\"/"/g'`
6. **飞书文件夹去重** — 创建同名文件夹不会报错，会生成重复文件夹。删除前需清空内容（飞书要求空文件夹才能删除）
7. **md-writer 需要完整 markdown** — 生成的内容必须包含 YAML frontmatter 和完整 markdown 结构，纯文本会写入失败
