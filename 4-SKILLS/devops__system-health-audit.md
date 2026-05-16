---
name: system-health-audit
description: >-
  Comprehensive system health audit checklist and methodology. Covers 7 audit dimensions:
  infrastructure resources, service availability, error log analysis, cron job integrity,
  dependency health, subsystem-specific checks, and structured report generation with
  P0/P1/P2 priority classification.
category: devops
---

# 系统健康审计 — 方法论与检查清单

## 概述

当用户要求「系统整体情况分析」「健康检查」「运行状态审计」时，遵循此方法论。无需每次加载全部维度，根据上下文选择相关维度即可。

## 7 维审计框架

```
┌─────────────────────────────────────────────┐
│ 1. 基础设施资源   2. 服务可用性              │
│    RAM / CPU / GPU    进程 / API / 端口       │
│    磁盘 / 负载                                │
├─────────────────────────────────────────────┤
│ 3. 错误日志         4. Cron 任务完整性         │
│    errors.log         所有计划任务运行状态      │
│    agent.log         上次/下次执行时间          │
│    子系统日志                                 │
├─────────────────────────────────────────────┤
│ 5. 依赖健康         6. 子系统专项              │
│    Ollama 模型状态    教育系统端点              │
│    ChromaDB 状态      飞书同步管道              │
│    外部API连通性      质量门禁状态              │
├─────────────────────────────────────────────┤
│ 7. 结构报告输出                                │
│    表格化 → 优先级标记 → 建议列表               │
└─────────────────────────────────────────────┘
```

## 维度详解

### 1. 基础设施资源

```bash
# 系统资源
free -h                           # 内存 + swap
df -h / /mnt/c/ 2>/dev/null       # WSL 磁盘 + Windows C: 盘
uptime                             # 运行时间 + 负载
nvidia-smi --query-gpu=memory.used,memory.total,temperature.gpu,utilization.gpu --format=csv,noheader 2>/dev/null

# 大目录（排查空间瓶颈）
du -sh ~/.cache/                  # 可能占用较大空间
du -sh ~/.hermes/                 # Agent 目录
du -sh ~/edu-hub/                 # 教育系统
```

**⚠️ WSL 特有检查**：
- WSL 磁盘 (/) 通常剩余很多，但 Windows C: 盘 (/mnt/c/) 可能接近 80%+
- C: 盘满时会导致 WSL I/O 性能下降
- .cache 目录可能累积至 6GB+，包含 pip/uv/camoufox/playwright 缓存，可按需清理

**🔧 C: 盘深度排查（从 WSL 调 Windows PowerShell）**：
```bash
# 从 WSL 内部快速查询 Windows C: 盘大目录
powershell.exe -Command @'
$folders = @(
    'C:\Users\yeyu_\Downloads',
    'C:\Users\yeyu_\Desktop',
    'C:\Users\yeyu_\AppData\Local\Temp',
    'C:\Windows\Temp',
    'C:\Windows\System32\DriverStore',
    'C:\Program Files',
    'C:\Program Files (x86)',
    'C:\ProgramData'
)
foreach ($f in $folders) {
    $size = (Get-ChildItem $f -Recurse -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
    $s = if ($size) { '{0:F1} GB' -f ($size/1GB) } else { '0 GB' }
    Write-Output (\"{0,-60} {1}\" -f $f, $s)
}
'@

# 查 WSL 发行版本身大小（位于 Packages 目录）
powershell.exe -Command "Get-ChildItem 'C:\Users\yeyu_\AppData\Local\Packages' -Filter '*WSL*' -Directory | ForEach-Object { \$size=(Get-ChildItem \$_.FullName -Recurse -EA SilentlyContinue | Measure-Object Length -Sum).Sum; Write-Output (\"WSL: {0} ({1:F1} GB)\" -f \$_.Name, (\$size/1GB)) }"
```
> 注意：`du -sh /mnt/c/...` 从 WSL 遍历 Windows 文件系统极慢，**始终用 `powershell.exe -Command`** 替代。

> **WSL C: 盘固定成本**：WSL Ubuntu 发行版自身占用约 **24-25 GB**（存储在 `AppData\Local\Packages\CanonicalGroupLimited.Ubuntu_*`），这是系统运行的必要开销，不可清理。C: 盘 84% 时 WSL 占最大头但不可省。

**🧹 .cache 清理标准流程**：
```bash
# 1. 先查明构成
du -sh ~/.cache/* | sort -rh

# 2. 安全清理项（幂等，不影响系统功能）
pip cache purge         # 清理 pip 包下载缓存（实测回收 ~2.9GB, ~900 files）
uv cache clean          # 清理 uv 包管理器缓存（实测回收 ~1.3GB, ~37000 files）

# 3. 谨慎选项（需重新下载）
# rm -rf ~/.cache/camoufox      # 浏览器自动化配置（1.3GB）
# rm -rf ~/.cache/ms-playwright # Playwright 浏览器二进制（631MB）
```

**清除后验证回收效果**：
```bash
df -h /                   # WSL 磁盘用量变化
du -sh ~/.cache/          # 清理后总大小（典型值 6.2GB → 2.0GB）
```

### 2. 服务可用性

```bash
# 关键进程
ps aux | grep -E "app.py|edu-hub|flask" | grep -v grep
ps aux | grep ollama | grep -v grep
ps aux | grep "hermes.*gateway" | grep -v grep

# API 端点探活（以 edu-hub 为例）
for path in / /math/ /english/ /chinese/; do
  curl -s -o /dev/null -w "$path: %{http_code}\n" http://localhost:5002$path
done
```

### 3. 错误日志分析

```bash
# Hermes 日志
tail -50 ~/.hermes/logs/errors.log        # 最新错误
grep -i "error\|exception\|fail\|timeout" ~/.hermes/logs/agent.log | tail -20
tail -20 ~/.hermes/logs/gateway.log       # 网关活动

# 教育系统日志
tail -50 ~/edu-hub/logs/c003-activity.log 2>/dev/null

# 注意：区分当前问题与历史存档问题
# agent.log 中的旧 WARNING（如 4 月的 context summary 404）若之后未重现，标记为历史
```

**🔍 确认历史错误是否已修复**：
```bash
# 查找过去 30 天内某个错误是否重现
grep -c 'your-error-pattern' ~/.hermes/logs/errors.log       # 总出现次数
grep 'your-error-pattern' ~/.hermes/logs/errors.log | tail -1 # 最后一次出现时间
grep -n 'May 0[5-9]' ~/.hermes/logs/errors.log              # 限定日期范围

# 结合 cron 状态确认：若 cron 任务的 last_status="ok" 且最近日志无对应错误，可判定已修复
```

### 4. Cron 任务完整性

使用 `cronjob list` 获取：
- 每个任务的 `last_status`（ok / failed / null）
- `next_run_at` vs `last_run_at` — 推断是否按预期执行
- 重点关注：每日 7AM 练习生成、8/20 点解析、2h 英语同步、3AM 飞书同步

### 5. 依赖健康

```bash
# Ollama 模型
curl -s http://localhost:11434/api/tags | python3 -c "import sys,json;d=json.load(sys.stdin);[print(f'  {m[\"name\"]:30s} {m[\"size\"]/1e9:.1f}GB') for m in d['models']]"
curl -s http://localhost:11434/api/ps    # 显存驻留状态

# ⚠️ 验证 Ollama 是否真正使用 GPU
# Ollama 可能在 CPU 上运行却不报错——仅检查 API 响应不够
# 详见 references/ollama-gpu-diagnostics.md

# ChromaDB
du -sh ~/.hermes_tools/knowledge_base/   # 向量库大小

# 飞书 API 连通性（通过 Token 验证脚本间接确认）
```

#### 5a. 依赖拓扑分析（关键补充 — 防隐形空跑）

❗ **常见错误**：只检查单个 cron 的 `last_status`，忽略了跨任务的基础设施依赖。两个看似独立的 cron 可能共享同一文件路径/挂载点，一个挂了全家瘫痪。

**依赖拓扑绘制方法**：

```bash
# Step 1: 列出所有 cron 及其脚本路径
cronjob list

# Step 2: 提取所有脚本读取的文件路径/Ollama模型/外部服务
# 关注点：多个脚本是否共享同一条路径？
grep -rn "/mnt/g/\|WPSDocument\|知识库文档\|EMAIL_INBOX\|WPSDOC_ROOT\|KB_ROOT" ~/.hermes_tools/knowledge_base/scripts/*.py ~/.hermes/scripts/*.py 2>/dev/null | grep -v ".pyc"

# Step 3: 标记共享路径的脚本为「依赖集群」
# 例：6个脚本共享 /mnt/g/WPSDocument/ → WPS云盘是单点故障枢纽
```

**关键检查项**：

| 维度 | 检查方法 | 典型风险 |
|------|----------|----------|
| 共享文件路径 | `grep -rn` 所有脚本中的硬编码路径 | WPS云盘 /mnt/g/ 挂了→6脚本全空跑 |
| 共享Ollama模型 | 检查多脚本引用的模型名 | 模型升级→静默失效 |
| 共享API Token | 多脚本读同一 token 文件 | token 过期→连锁失败 |
| 跨平台依赖 | Windows端脚本+WSL端脚本是否成对 | Foxmail导出不运行→分类器永远空跑 |
| 脚本间数据依赖 | A 的输出 = B 的输入 | 周二生成缺 cron→周五发布空跑 |

**排查技巧 — 识别「空跑链」**：

对于有依赖关系的任务批次，检查链上每个环节的**实际产出**而非仅执行状态：

```
任务A (✅ ok) → 输出文件X → 任务B (✅ ok) → ...
```

```bash
# 检查 A 是否产生了供 B 使用的文件
ls -lt ~/.hermes/cache/content-factory-calendar.json  # 选题缓存存在？
ls -lt ~/.hermes/data/content-factory-articles/       # 生成的文章存在？

# 检查共享路径是否可写可读
ls -la /mnt/g/WPSDocument/WPS云盘/邮件/自动导出/       # 邮件自动导出目录
test -w /mnt/g/WPSDocument/                          # WPS云盘根目录可写
mount | grep " /mnt/g "                              # DrvFs 挂载状态
```

**⚠️ WSL 特有 — DrvFs 挂载依赖**：
- `/mnt/g/` 等挂载点在 WSL 重启后自动恢复（改: 仅当 WSL 正常关闭后再启动时；硬重启或 `wsl --shutdown` 后可能需重新 mount）
- Foxmail 导出脚本运行在 Windows 端，通过 WPS云盘本地同步目录 → WSL 通过 DrvFs 读取
- **关键诊断**：Windows 任务计划程序中的 Foxmail 导出脚本是否正常运行，比 WSL 侧 cron 状态更重要

**参考**：`references/dependency-topology-20260514.md` — 本环境的完整依赖拓扑图

### 6. 子系统专项检查

| 子系统 | 检查项 | 命令/路径 |
|--------|--------|----------|
| 教育系统统合 | 4 端点可用性 | curl http://localhost:5002/* |
| AI家庭教师 | 入口+三科练习+拍照 | 详见 references/ai-family-tutor-health-check.md |
| 数学 C003 | 日志最近运行 | tail ~/edu-hub/logs/c003-activity.log |
| 英语 C008 | 数据大小 | du -sh ~/edu-hub/english/data/ |
| 质量门禁 C015 | 上次扫描结果 | tail ~/edu-hub/quality_gate/quality_gate.log |
| 系统知识同步 C009 | 文档覆盖 | ls ~/wiki/raw/systems/ |
| 飞书同步 | Token 有效性 | 检查 ~/.hermes/feishu-tokens.json |

### 7. 报告输出格式

报告应包含：
1. **总览表** — 状态标记 🟢🟡🔴 + 简要说明
2. **问题列表** — 按 P0/P1/P2 优先级排序
3. **建议优先级** — 带预估工时的执行建议

示例标记规范：
```
| 项目 | 状态 | 说明 |
|------|------|------|
| 系统资源 | 🟢 健康 | RAM 1.7/15GB, 负载 0.07 |
| C: 盘 | 🟡 84% | 167GB/200GB — 需关注 |
| 质量门禁 | 🟡 未复查 | 上次5月6日, C008 3/7通过 |
```

## 已知检查清单（WSL 环境特化）

- [ ] WSL 磁盘使用率 < 80%?
- [ ] Windows C: 盘使用率 < 80%? (常见瓶颈!)
- [ ] RAM 可用 > 2GB?
- [ ] GPU 显存利用率 < 85%?
- [ ] Edu-Hub 4 端点全部 200?
- [ ] Ollama 响应正常?
- [ ] ChromaDB 文件存在?
- [ ] 所有 cron 任务 last_status = ok?
- [ ] 近 24h errors.log 无新错误?
- [ ] agent.log 无反复出现的 Exception?
- [ ] 质量门禁 7 天内运行过?
- [ ] 系统知识文档覆盖所有 active 子系统?
- [ ] ~/.cache 大小 < 5GB?
- [ ] Agent.log / errors.log 大小 < 5MB?

## 典型输出结构

```markdown
## 一、系统健康度总览

| 项目 | 状态 | 说明 |
| --- | --- | --- |
| 系统资源 | 🟢 | ... |

## 二、各子系统运行详情

### 教育系统 (Edu-Hub)
...

## 三、发现的问题与优化建议

### 🔴 P1 — 需关注
1. **问题描述** — 数据 + 建议措施

### 🟡 P2 — 常规优化
...

### 🟢 P3 — 清爽建议
...

## 四、建议优先级
| 优先级 | 事项 | 预计工时 |
|--------|------|---------|
| P1 | ... | 15min |
```

## 关键经验

1. **C: 盘可能是瓶颈** — WSL 磁盘 (/dev/sdd) 通常够用，Windows C: 盘 (/mnt/c/) 才是常见限制
2. **.cache 目录定期清理** — huggingface/pip/uv 缓存可累积至 6GB+
3. **区分历史日志与当前问题** — agent.log 可能包含数周前的旧警告，检查时间戳
4. **ChromaDB 位置** — 位于 `~/.hermes_tools/knowledge_base/`（非 `~/chroma_db/`）
5. **Dual-registration 检查** — 如果 Flask 服务同时注册了独立 app 和统一 app，确认哪个进程实际运行
6. **质量门禁滞后** — 修复后需主动 rerun quality_gate，不会自动更新
7. **知识文档覆盖 gap** — C009 增量同步可能未覆盖所有系统，手动补齐
8. **no_agent cron 不支持 script 字段带参数** — script 字段值被整体当文件路径解析，含空格时报 "Script not found"。参数必须由 shell wrapper 脚本封装（如 `health-reminder-0600.sh` → `exec python3 health-reminder.py 0600`）。创建 no_agent cron 时 script 路径必须无参数。
9. **依赖拓扑分析要检查「批次上下游产出」而非仅 cron 状态** — 状态✅不代表有产出。五步法：列出依赖链 → 检查每个环节的中间文件是否存在 → 检查共享路径挂载 → 检查跨平台(Windows)脚本 → 检查脚本间数据流是否断裂（如周二生成缺 cron 导致周五发布空跑）
