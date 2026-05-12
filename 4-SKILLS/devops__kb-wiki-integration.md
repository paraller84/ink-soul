---
name: kb-wiki-integration
title: 知识库 RAG → Wiki 增量源集成
description: 将本地知识库 RAG 系统的文档目录构建为与飞书对等的 Wiki 增量摄入源。包含门户页、增量同步管道、状态追踪、索引集成。
category: devops
trigger: 用户希望将知识库文档目录与 Wiki 打通，作为 Wiki 的一个增量摄入源（与飞书管道平级）
---

# 知识库 RAG → Wiki 增量源集成

## 概述

将知识库文档目录构建为 Wiki 的第二个增量摄入源（与飞书管道对等）：
1. **增量管道** — 文件签名（size:mtime）检测 → 状态追踪 → 只处理变更
2. **门户页** — 文档概览、主题聚类、类别统计、**门户访问方式**
3. **文档摘要页** — 每文档一个 Wiki 摘要页（`raw/knowledge-base/`）
4. **SCHEMA.md 源注册** — 在 Phase 5（信息同步闭环）中注册为第二个源

## 架构（与飞书对等）

```
飞书源                             知识库源
  │                                   │
  ▼                                   ▼
feishu-wiki-sync.py                  kb-wiki-sync.py
  │                                   │
  ├─ API 扫描 Drive                   ├─ 文件系统扫描
  ├─ edited_time 增量检测              ├─ size:mtime 签名检测
  ├─ 读取内容 → 结构化 JSON            ├─ 生成 Wiki 摘要页
  └─ 状态: feishu-state.json           └─ 状态: kb-state.json
  │                                   │
  ├─→ raw/feishu/                     ├─→ raw/knowledge-base/
  └─→ index.md「飞书导入」             └─→ index.md「知识库文档」
```

## 增量策略

核心设计：**文件签名（size:mtime）** → **状态文件对比** → **只处理变更**

```python
# 文件签名生成
sig = f"{stat.st_size}:{stat.st_mtime}"

# 增量检测逻辑
if rel not in state['synced_files'] → 新增
elif state['synced_files'][rel]['sig'] != current_sig → 变更
elif rel in state but not in current → 删除
else → 跳过
```

状态文件 (`~/.hermes/scripts/kb-wiki-sync-state.json`) 结构：

```json
{
  "version": 1,
  "last_sync": "2026-04-26T12:45:38",
  "source": "knowledge-base",
  "synced_files": {
    "PDF 文档/xxx.pdf": {
      "sig": "275264:1775442723.0",
      "wiki_path": "/home/.../raw/knowledge-base/xxx.md",
      "category": "PDF 文档",
      "synced_at": "2026-04-26T12:45:31"
    }
    // ... 每个文档一条
  }
}
```

## 文件

- **同步脚本**: `~/.hermes/scripts/kb-wiki-sync.py`
- **门户页**: `~/wiki/guides/kb-portal.md`
- **状态文件**: `~/.hermes/scripts/kb-wiki-sync-state.json`
- **文档摘要页**: `~/wiki/raw/knowledge-base/`（1,000+ 页）

## 脚本模式

### 用法

```bash
python3 ~/.hermes/scripts/kb-wiki-sync.py                # 增量（推荐日常）
python3 ~/.hermes/scripts/kb-wiki-sync.py --catalog-only  # 仅更新门户统计+索引（秒级）
python3 ~/.hermes/scripts/kb-wiki-sync.py --full          # 全量重建所有摘要页
python3 ~/.hermes/scripts/kb-wiki-sync.py --report        # 输出 JSON 报告
```

### 三种模式的执行路径

| 模式 | 流程 | 场景 |
|:----:|:----|:-----|
| `--catalog-only` | 扫描 → 主题分析 → (跳过增量检测) → 更新门户 → (跳过摘要页) → 更新索引 | 每次会话开始时快速了解现状 |
| 默认（增量） | 扫描 → 主题分析 → **增量检测(state对比)** → 更新门户 → **增量生成/更新/删除摘要页** → 更新索引 | 定期维护（每周/每日） |
| `--full` | 扫描 → 主题分析 → (标记全部为变更) → 更新门户 → **全量重建摘要页** → 更新索引 | 首次接入/脚本大版本变更 |

### 幂等性

所有模式都是幂等的：二次运行无副作用。增量模式下第二次运行总是输出 "0 新增/0 变更/0 删除"。

## SCHEMA.md 标签注册

### Phase 1（标签族）
```yaml
- **KB Wiki Tags**: kb-portal, kb-doc, kb-summary, kb-sync, kb-index
- **KB Categories**: kb-pdf, kb-ppt, kb-word, kb-meeting, kb-plan, kb-report
```

### Phase 5（源注册）
在飞书源下方添加知识库源描述：

```markdown
> **已接入的知识库文档源（Phase 1）：**
> 1. **个人知识库文档** → 标签: `kb-portal`, `kb-doc`, `kb-summary`
> 2. 知识库文档来自 `/mnt/g/知识库文档/`（7 类别，1,342 文件，3GB）
> 3. 文档摘要页写入 `raw/knowledge-base/`，同步脚本 `~/.hermes/scripts/kb-wiki-sync.py`
> 4. 增量策略：文件签名（size:mtime）检测变更，只处理新增/变更/删除
> 5. 门户页在 `guides/kb-portal.md`，`index.md` 的「知识库文档」分区建立索引
```

## 门户页设计

门户页 (`guides/kb-portal.md`) 必须包含以下区块（已实战验证的结构）：

### 1. 门户访问方式（🚪 区块）
这是用户明确要求添加的。提供三种途径的访问指引：

```markdown
## 🚪 门户访问方式

### 从 Wiki 内访问
| 路径 | 说明 |
|:----|:-----|
| `~/wiki/guides/kb-portal.md` | 📄 **此门户页** |
| `~/wiki/index.md` →「知识库文档」 | 📑 **索引页** |
| `~/wiki/raw/knowledge-base/` | 📝 **文档摘要页** |
| `~/wiki/entities/capability-registry.md#c001` | 🔧 **C001 注册表** |

### 从终端直接检索
```bash
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --query "..."
python3 ~/.hermes/scripts/kb-wiki-sync.py --catalog-only
```

### 从对话中
直接对 Hermes 说：**"查一下知识库里有没有关于 [主题] 的资料"**
```

### 2. 文档概览表（📊）
```markdown
| 类别 | 文件数 | 大小 |
|:----:|:-----:|:----:|
| 📄 PDF 文档 | 363 | 978 MB |
| ... | ... | ... |
| **合计** | **1,342** | **3 GB** |
```

### 3. 主题聚类表（🏷）
通过文件名关键词匹配的统计：
```markdown
| 主题 | 文件数 | 关键词 |
|:----:|:-----:|:--------|
| 🔷 保险金融 | 441+ | 保险、太保、金融、IFRS |
```

### 4. 索引与同步区（🧹）
显示最近同步状态和变更统计：
```markdown
| 新增 | 变更 | 删除 | 总文档 | Wiki 摘要页 |
|:----:|:----:|:----:|:-----:|:----------:|
| 0 | 0 | 0 | 1,342 | 1,166 |
```

### 5. 关联链接（🔗）
- C001 注册表（系统详情）
- 解决方案台账（架构参考）
- 体系化能力注册表（系统总览）

## C001 注册表更新模式

在 `capability-registry.md` 的 C001 条目中做 4 件事：

1. **资产表** — 新增 `kb-wiki-sync.py` 脚本行（放在 kb_retrieve.py 之前）
2. **Wiki 集成区块** — 在健康检测之后新增，说明门户页、索引、脚本、检索桥
3. **变更日志** — 新增 wiki-integrate 操作记录（描述要具体）
4. **标签** — 追加 `kb-sync`, `kb-portal`

ASCII 图数字更新陷阱：capability-registry.md 中的流程示意图数字（如"1,342 可整理文件"）必须与扫描结果一致。替换时注意保留 ASCII 对齐空格。

## index.md 集成

在 `index.md` 中新增「知识库文档」分区（放在「飞书导入」之前）：

```markdown
## 知识库文档

<!-- 个人知识库 RAG → 文档门户与摘要页 -->
<!-- 更新于 YYYY-MM-DD HH:MM -->
<!-- 共 N 文件，X GB -->

| 类别 | 文件数 | 页面 |
|:----:|:-----:|:----|
| PDF 文档 | 363 | [知识库门户](guides/kb-portal.md#PDF-文档) |
| ... | ... | ... |

> 门户页: [知识库文档门户](guides/kb-portal.md) | 检索: `kb_retrieve.py --query "..."`

## 飞书导入
...
```

同时更新顶部 `| 总页面数: N` 计数。

## 文档摘要页生成

每文档的 Wiki 摘要页使用 `_safe_stem()` 对文件名做安全清洗（去特殊字符、截断 120 字符）。摘要页包含：
- YAML frontmatter: title, created/updated (取自文件 mtime), type, tags, phase
- 文件元信息：路径、大小、类别、格式
- 摘要占位（待 LLM 提取）
- 关联链接

**特别情况**：不同原始文件可能映射到同一 sanitized 文件名（如 `1_L2-Day1-电子练习册.pdf` 和 `1_L2Day1-电子练习册.pdf` 都可能变成 `1_L2Day1-电子练习册.md`）。后写入的那个会覆盖前者。这在首次全量时不可控，但增量模式下只会覆写被再次检测到变更的文件。

## 部署与调度（Cron 设置）

kb-wiki-sync 和 feishu-wiki-sync 脚本一旦完成，**必须显式创建 cron 任务**，否则它们永远不运行。
「记忆中说有 cron」不等于「实际上有 cron」——两个脚本都曾被遗漏调度长达 16 天。

### kb-wiki-sync（每日 23:00）

```bash
hermes cron --create \
  --name "知识库-Wiki摘要同步 kb-wiki-sync" \
  --schedule "0 23 * * *" \
  --script "kb-wiki-sync.py" \
  --no-agent
```

### feishu-wiki-sync（每日 09:00）

```bash
hermes cron --create \
  --name "飞书-Wiki同步 feishu-wiki-sync" \
  --schedule "0 9 * * *" \
  --script "feishu-wiki-sync.py" \
  --no-agent
```

### 验证调度生效

```bash
hermes cron --list
# → 确认两个任务 state=scheduled, next_run_at 正确
# → 检查 last_status 在首次执行后不为 null

# 手动触发测试
python3 ~/.hermes/scripts/kb-wiki-sync.py --catalog-only
python3 ~/.hermes/scripts/feishu-wiki-sync.py
```

> ⚠️ 这两个脚本与 kb_pipeline 的调度方式不同：pipeline 走系统 crontab，它们走 Hermes cron。
> 系统 crontab 用 `crontab -l` 查看，Hermes cron 用 `hermes cron --list` 查看，**互不包含**。

## 已知陷阱

1. **ASCII 图对齐脆弱** — capability-registry.md 中的流程图是 ASCII art，用 `patch` 替换时多余的字符会破坏对齐。替换后必须 visual check。
2. **中文文件名 → Wiki 文件名冲突** — 不同源文件（不同路径或不同内容）可能因 sanitize 后产生同名 .md 文件，后写的覆盖先写的。这是已知限制，不影响 RAG 检索（内容存在向量库中）。
3. **index.md 页面计数** — 每次新增/删除页面后手动更新 `| 总页面数: N`。否则数字会过时。
4. **portal 页的 regex 替换脆弱** — 尝试用正则替换 portal 页的表格区块时，如果表格内容格式有细微变化（多余空行、额外列），替换可能失败或生成重复/混乱的表格。**推荐直接重写整个门户页**而非局部替换。
5. **主题分析只是文件名级** — 关键词匹配基于文件名而非文件内容。统计数字显示为"441+"而非精确值。精确统计需要读取文件内容或使用向量聚类。
6. **log.md 必须追加** — 每次 Wiki 变更都要记录到 log.md，否则 audit 时无法追溯变更历史。
7. **主题数据在 portal 页写死后可能过时** — 门户页的统计、主题数据如果不定期运行 `kb-wiki-sync.py` 更新，会与实际知识库状态脱节。
8. **首次全量生成耗时** — 1,342 文件的首次全量处理约 1.8 秒（纯元数据生成，不涉及 LLM）。如果后续添加 LLM 摘要提取，耗时将大幅增加。
