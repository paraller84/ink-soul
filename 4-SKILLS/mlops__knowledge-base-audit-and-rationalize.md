---
name: knowledge-base-audit-and-rationalize
description: 审计环境中的多套重叠知识库系统，评估每套系统的运行时状态，提供 RAG vs Wiki 的决策框架，执行安全清理和部署推荐方案
version: 1.0.0
author: Hermes Agent
license: MIT
tags:
  - knowledge-base
  - audit
  - cleanup
  - rag
  - wiki
  - karpathy
  - vector-search
trigger: |
  当用户提到环境中有多个知识库/向量检索系统、觉得混乱/重叠、不知道选哪个时。或者用户询问"我的知识库系统状态"、"帮我看看我有哪些知识库"、"清理不需要的系统"。
---

# 知识库系统审计与清理工作流

## 概述

在长期使用 AI 系统过程中，用户往往会积累多套实验性的知识库系统（RAG 管道、向量检索部署、个人 Wiki 等），它们功能重叠、状态混杂、占用大量磁盘空间。

本工作流提供一套 **系统化的审计、比较、决策、清理和部署** 流程，帮助用户从混乱走向清晰。

## 工作流全景

```
① 全面扫描          ② 运行时检查         ③ 架构对比分析
┌─────────┐         ┌─────────┐          ┌─────────┐
│ find/   │   →    │ API日志 │    →     │ RAG vs  │
│ ls -la  │         │ 数据库   │         │ Wiki    │
│ 所有KB   │         │ 向量计数 │          │ 决策框架 │
└─────────┘         └─────────┘          └─────────┘
                                              │
                                              ▼
④ 推荐方案          ⑤ 安全清理             ⑥ 部署推荐系统
┌─────────┐         ┌─────────┐          ┌─────────┐
│ 保留 /   │   →    │ 归档源码 │    →     │ 初始化  │
│ 清理 /   │         │ 移除venv │          │ SCHEMA  │
│ 新增     │         │ 释放空间 │          │ index   │
└─────────┘         └─────────┘          └─────────┘
```

## 第一步：全面扫描

扫描系统中的所有知识库相关目录和文件。使用三条搜索线：

### ① 查找目录和文件

```bash
# 查找 ~/.hermes_tools/ 下的知识库文件
find ~/.hermes_tools/ -name "*knowledge*" -o -name "*知识库*" -o -name "*vector*" -o -name "*rag*" -o -name "*retrieval*" 2>/dev/null

# 查找主目录下的部署项目
find ~/ -maxdepth 2 -name "*知识库*" -o -name "*knowledge*" -o -name "*wiki*" -o -name "*vector*" -o -name "*retrieval*" 2>/dev/null | grep -v __pycache__ | grep -v node_modules | grep -v .git

# 查找技能目录中相关的技能
ls ~/.hermes/skills/*/*knowledge* ~/.hermes/skills/*/*wiki* ~/.hermes/skills/*/*vector* ~/.hermes/skills/*/*retrieval* 2>/dev/null
```

### ② 识别每套系统的核心特征

对每个发现的系统，回答 5 个问题：

| 问题 | 检查方式 |
|------|----------|
| 这是什么系统？ | 看 README、项目名、脚本名 |
| 是以什么形式存储的？ | 向量嵌入 / Markdown 文件 / 其他 |
| 有数据吗？ | 检查数据库大小、文件数量、嵌入计数 |
| **是否曾被使用？** | 检查日志文件、cron 记录、API 启动日志 |
| **占多大空间？** | `du -sh <path>` 特别是 venv 目录 |
| **真实源头 vs 文档源头** | 检查脚本中的 SOURCE_DIR 或 KB_ROOT 实际指向 -> 是否与 README/技能描述一致（常见陷阱：RAG 系统把中间产物当源头）|
| **是否递归扫描子目录** | 检查脚本是否用 `os.walk` 或 `rglob` -> 如果只用 `os.listdir()` 或 `Path.iterdir()` 则漏掉了子目录 |

## 第二步：运行时状态检查

对每套系统，检查它是否真正"活着"：

### 检查向量数据库

```python
import sqlite3, os

db_path = os.path.expanduser('~/path/to/chroma.sqlite3')
if os.path.exists(db_path):
    conn = sqlite3.connect(db_path)
    c = conn.cursor()
    c.execute('SELECT COUNT(*) FROM embeddings')
    count = c.fetchone()[0]
    print(f'向量嵌入数: {count}')
    conn.close()
else:
    print('数据库不存在')
```

### 检查 API 是否曾启动成功

```bash
cat ~/path/to/logs/api.log 2>/dev/null | tail -30
# 看最后一行是否是启动成功/失败的证据
```

### 检查是否有活跃进程

```bash
pgrep -af "chroma|vector|faiss|api|wiki" 2>/dev/null | head -10
```

### 检查 cron 任务

```bash
# Hermes 内置 cron
hermes cron list 2>/dev/null || true

# 系统 crontab
crontab -l 2>/dev/null || true

# 自定义 cron 脚本
ls ~/.hermes_tools/crontab*.txt 2>/dev/null
```

## 第三步：RAG vs Wiki 架构对比决策框架

这是核心决策工具。将每个系统分类到以下两种架构之一，然后根据用户需求推荐：

### 对比维度

| 维度 | RAG / 向量检索系统 | Karpathy Wiki / Markdown 知识库 |
|------|-------------------|-------------------------------|
| **存储形式** | 向量嵌入（不可读） | Markdown 文件（可读可编辑） |
| **检索方式** | 语义搜索（模糊匹配） | 人读 + 超链接导航 |
| **知识更新** | 每次重新搜索（一次性） | 内容沉淀，一次编译永久受益 |
| **可读性** | ❌ 黑盒 | ✅ 可直接用 Obsidian/VS Code 阅读 |
| **适合** | **找文件**（模糊搜索） | **学知识**（深度理解） |
| **典型场景** | "帮我找到那份关于XX的PPT" | "解释一下LoRA的原理" |
| **查询效率** | O(log n) 向量搜索 | O(1) 直接翻看 |
| **知识复合效应** | 无（每次都独立搜索） | ✅ Wiki 越用越丰富 |

### 决策问题清单

向用户提出以下问题来帮助决策：

1. **你的内容是文档还是知识？**
   - 文档（PDF/PPT/Word）→ RAG ✅
   - 知识（概念/论文/技术笔记）→ Wiki ✅

2. **你希望怎么使用？**
   - "帮我找到XXX" → RAG
   - "帮我理解XXX" → Wiki
   - 两者都要 → **两个都保留，各司其职**

3. **你需要积累知识吗？**
   - 是 → Wiki（知识持续复合）
   - 否 → RAG（一次检索就够了）

### 推荐矩阵

```
用户场景                           推荐
─────────────────────────────────────────────────
个人文档管理（PPT/Word/PDF）       → RAG 系统（保留1套）
技术研究（论文/概念/模型）         → Karpathy Wiki
文档搜索 + 技术学习               → RAG + Wiki 双轨并行
```

## 第四步：生成清理方案

基于决策结果，按以下优先级处理每套系统：

### 保留（KEEP）
- 有数据的、正在运行的
- 与用户需求匹配的
- 占用空间合理的

### 清理（CLEAN UP）
- 有重叠的系统（功能相同或类似）
- 搭建了但从未成功运行的
- 用户不再需要的

### 归档（ARCHIVE）
对要清理的系统：**不要直接删除！** 先安全归档：

```bash
# 1. 创建归档目录
mkdir -p ~/archived-systems/

# 2. 归档源码（排除 venv 节省空间）
tar -czf ~/archived-systems/<project-name>-src.tar.gz \
  -C ~/<project-dir> \
  --exclude=venv \
  --exclude=__pycache__ \
  --exclude=*.pyc \
  .

# 3. 记录归档说明
echo "# <项目名> 归档说明
归档日期: $(date +%Y-%m-%d)
原路径: ~/<project-dir>
大小释放: <预计空间>
恢复命令: tar -xzf <project-name>-src.tar.gz -C ~/<project-dir>
" > ~/archived-systems/<project-name>-README.md

# 4. 删除原目录
rm -rf ~/<project-dir>
```

### 新增（DEPLOY NEW）
部署推荐的系统（Karpathy Wiki 或新的 RAG 系统）。

## 第五步：部署 Karpathy Wiki（当推荐时）

按照 `llm-wiki` 技能的流程初始化：

```bash
# 1. 创建目录结构
WIKI="$HOME/wiki"
mkdir -p "$WIKI"/{raw/{articles,papers,transcripts,assets},entities,concepts,comparisons,queries}

# 2. 询问用户领域
# 必须明确领域才能写 SCHEMA.md

# 3. 写 SCHEMA.md（包括标签体系、命名规范、阈值规则）
# 4. 写 index.md（内容目录）
# 5. 写 log.md（操作日志）
```

SCHEMA.md 模板见 `llm-wiki` 技能。

## 第六步：报告与确认

给用户一份清晰的最终报告：

```
📋 知识库系统审计报告
━━━━━━━━━━━━━━━━━━━━━━━

系统 A: 个人知识库 RAG
  ├ 状态: ✅ 有数据 (34K 向量)
  ├ 建议: 保留
  └ 用途: 文档模糊搜索

系统 B: 向量检索部署框架
  ├ 状态: ❌ 启动失败 / 未使用
  ├ 建议: 已归档到 ~/archived-systems/
  └ 释放空间: ~5.4GB

系统 C: Karpathy Wiki
  ├ 状态: ✅ 已初始化
  ├ 领域: [用户指定的领域]
  └ 下一步: 开始摄入第一篇资料

资源使用:
  ├ 总空间: ~/archived-systems/ = 36KB
  ├ 当前系统: [各系统空间]
  └ 总计释放: ~XGB
```

## 注意事项

1. **永远不要直接 rm -rf**，先归档再删除
2. **排除 venv 目录**（5GB+），只归档源码（通常 < 1MB）
3. **写归档说明文件**，方便以后恢复
4. **HuggingFace 网络问题**：如果 API 因 HF 下载失败，考虑设置 HF 镜像或使用离线模型
5. **向用户解释区别**：很多人不知道 RAG 和 Wiki 是两种不同的范式
6. **双轨策略**：用户往往两个都需要，不要强行二选一

## Pitfalls

- **不要只列目录名**：要检查运行时状态（日志/数据库/进程）才能判断是否"活着"
- **不要预设立场**：先审计再推荐，而非先推荐再审计
- **venv 是空间大头**：一个 venv 动辄 3-8GB，源码通常 < 1MB
- **知识库系统可能跨目录**：运行时脚本在 ~/.hermes_tools/，部署项目在 ~/，数据在 ~/.hermes/ 下
- **RAG 和 Wiki 不互斥**：最佳方案往往是两个都保留，各负责不同的需求场景
- **RAG 系统的输入源和中间产物容易混淆**：审计时留意脚本中的 SOURCE_DIR 配置——如果指向的是 organized/cleaned 目录而非原始目录，说明 flow 理解有偏差。正确模式是：WPSDocument(原始) → organize → 知识库文档(清理后) → vectorize → ChromaDB
