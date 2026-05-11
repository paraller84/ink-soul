---
name: personal-knowledge-base-management
description: 个人知识库 RAG 系统 — WPSDocument（递归子目录）→ 整理 → 知识库文档 → 增量向量化 → ChromaDB 检索
version: 2.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [knowledge-base, document-management, automation, RAG, chromadb, ollama, WPS, file-organization]
    related_skills: [knowledge-base-audit-and-rationalize, capability-registry-management, llm-wiki]
    use_cases:
      - 从 WPSDocument（含子目录）自动化整理文档到知识库文档
      - 建立 ChromaDB 向量库并增量更新
      - 通过 Hermes Agent 进行语义检索
      - 排查和修复已有的个人知识库 RAG 系统
---

# 个人知识库 RAG 系统

## 概述

从 **WPSDocument**（含所有子目录）到 **语义检索** 的完整闭环。三阶段流水线：

```
WPSDocument (递归扫描) → 整理(去重+版本+分类) → 知识库文档 → 增量向量化 → ChromaDB → 语义检索
```

所有步骤全程 **增量处理** + **本地模型**。

## 核心架构

### 正确的三阶段 Flow（⚠️ 容易搞错）

| 阶段 | 目录 | 文件数（实测） | 作用 |
|------|------|--------------|------|
| **① 输入源** | `G:/WPSDocument/` | **3,579 文件** | 原始文件，含子目录 |
|  | 根目录/ | 1,664 | 根目录直接存放的文件 |
|  | WPS云盘/ | 1,898 | WPS 自动同步的子目录 |
|  | 我的邮件/ | 17 | 邮件导出 |
|  | | | |
| **② 中间产物** | `G:/知识库文档/` | **1,088 整理后** | 去重+版本管理+分类后 |
|  | _archive/ | 1,593 | 重复/旧版本的归档副本 |
|  | | | |
| **③ 检索层** | `~/.hermes_tools/knowledge_base/vector_store/` | **34K 向量 / 189MB** | ChromaDB 持久化索引 |

### 目录结构

```
G盘文件系统/
├── WPSDocument/                    # 🔴 唯一输入源（递归扫描所有子目录）
│   ├── (根目录文件, 1,664)
│   ├── WPS云盘/                    # ← 容易漏掉！WPS 自动生成
│   └── 我的邮件/                   # ← 容易漏掉！邮件导出
│
└── 知识库文档/                      # 🟢 整理产物（不要误当作源头）
    ├── PDF 文档/  Word 文档/  PPT 文档/
    ├── 会议纪要/  方案文档/  汇报材料/  其他/
    ├── _archive/                    # 重复/旧版本文件的归档
    └── metadata.json                # 文件元数据（哈希/版本/分类/溯源）
```

### 脚本位置（标准化后）

```
~/.hermes_tools/knowledge_base/
├── scripts/
│   ├── kb_organize.py       # ① 7 级递进去重引擎 WPSDocument → 知识库文档 [Phase 2]
│   ├── kb_vectorize.py      # ② 增量向量化引擎 知识库文档 → ChromaDB [已就绪]
│   ├── kb_retrieve.py       # ③ 语义检索模块 ChromaDB → 回答 [已就绪]
│   ├── kb_cron.py           # ④ Cron 任务管理 install/uninstall/status/run [已就绪]
│   └── kb_pipeline.py       # ⑤ 主流水线入口 EnvCheck→扫描→整理→向量化→报告 [已就绪]
├── vector_store/            # ChromaDB 数据（chroma.sqlite3 + 索引文件）
├── cache/                   # 增量扫描缓存（wpsdoc_cache.json / file_cache.json）
├── reports/                 # 每日执行报告
└── logs/                    # 运行日志
```

### CLI 用法

```bash
# 全流程增量
python3 ~/.hermes_tools/knowledge_base/scripts/kb_pipeline.py

# 全量重建
python3 ~/.hermes_tools/knowledge_base/scripts/kb_pipeline.py --full

# 二次整理（规则变更后清理旧文件）
python3 ~/.hermes_tools/knowledge_base/scripts/kb_pipeline.py --resync

# 仅文件整理（不向量化）
python3 ~/.hermes_tools/knowledge_base/scripts/kb_pipeline.py --organize-only

# 仅向量化
python3 ~/.hermes_tools/knowledge_base/scripts/kb_pipeline.py --vectorize-only

# 保存 JSON 报告
python3 ~/.hermes_tools/knowledge_base/scripts/kb_pipeline.py --report

# 语义搜索
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --query "关键词" --limit 5

# 按文件名搜索
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --file-search "关键词"

# 知识库统计
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --stats

# Cron 管理
python3 ~/.hermes_tools/knowledge_base/scripts/kb_cron.py install
python3 ~/.hermes_tools/knowledge_base/scripts/kb_cron.py status
python3 ~/.hermes_tools/knowledge_base/scripts/kb_cron.py run

# 一键健康检查
python3 ~/.hermes_tools/knowledge_base/scripts/kb_health.py
```

## ⚠️ 常见陷阱（从实践中来）

### 陷阱 1：源头搞错（最常见错误）

```
❌ 错误：把 G:/知识库文档/ 当作源头
✅ 正确：源头是 G:/WPSDocument/（含所有子目录）
        知识库文档是整理后的产物
```

**检查方法**：看 metadata.json 中的 `source` 字段是否指向 WPSDocument 而非 知识库文档。

### 陷阱 2：漏掉 WPSDocument 的子目录

```
❌ 错误：只扫描 WPSDocument 根目录（1,664 文件）
✅ 正确：递归扫描所有子目录（+ WPS云盘/ 1,898 + 我的邮件/ 17 = 3,579 文件）
```

**原因**：WPS 云盘自动同步会在 WPSDocument 下创建 `WPS云盘/` 子目录，其中包含大量同步文档。

### 陷阱 3：4 脚本碎片化（脚本膨胀预警）

```
❌ 错误：4 个脚本（2515 行），三份嵌入逻辑并存
✅ 正确：合并为 1 个流水线 + 1 个向量化引擎 + 1 个检索模块
```

**症状**：`knowledge_base.py` / `knowledge_base_v2.py` / `knowledge_base_daily.py` / `knowledge_base_cron.py` 功能重叠。
**修复原则**：见到 3 份嵌入逻辑 → 立即合并，统一分块参数。

### 陷阱 4：运行时安装依赖

```
❌ 错误：在 daily.py 中通过 subprocess 调用 pip install（60秒超时失败）
✅ 正确：取消运行时安装，首次一次性装好，运行时不检查
```

**原因**：chromadb 依赖巨大（500MB+），pip install 在 60 秒内无法完成。
**正确做法**：在 Hermes venv 中一次性装好所有依赖，脚本只检查 import 不尝试自动安装。

### 陷阱 5：余弦相似度的 Tokenization 精度

```python
# 当前实现（简单分词）：
def tokenize(t):
    words = re.findall(r'[a-z0-9]+', t.lower())     # 英文词
    chars = re.findall(r'[\u4e00-\u9fff]', t)        # 中文字（单字）
    return words + chars
```

**问题**：中文单字切分会丢失语义信息（"人工智能"→"人""工""智""能"四个字）。
**影响**：对纯中文文档的相似度判断可能不够精确，95% 阈值是经验值。
**改进方向**：如需更高精度，改用分词库（jieba）或 embedding 模型做语义相似度。

### 陷阱 7：PDF 解析会产生大量警告

pypdf 解析某些 PDF 时会输出大量无害警告：
```
Ignoring wrong pointing object 16 0 (offset 0)
Multiple definitions in dictionary at byte 0xd8f7 for key /Ascent
```

**处理方式**：这些是 PDF 内部结构问题而非代码问题。可以重定向 stderr 或使用 `warnings.filterwarnings('ignore')` 在生产模式中抑制。

### 陷阱 8：首次全量整理需要 5-10 分钟

**症状**：首次运行 `kb_organize.py` 可能超时（默认 120 秒不够）。
**正确做法**：首次运行建议用 `timeout=600` 或直接夜间调度执行。
**后续增量**：仅处理变更文件，通常 < 30 秒。

### 陷阱 5：在 RAG 系统里混入云端 API

```
❌ 错误：文档分类用 DeepSeek API，QA 也用云端模型
✅ 正确：全程本地模型
        - 分类：Ollama qwen2.5:3b（或关键词匹配）
        - 嵌入：Ollama qwen3-embedding:0.6b
        - QA：Ollama qwen2.5:3b / qwen3:8b
```

**原因**：个人知识库应该离线可用，且节省 API 成本。

## 模型策略（全本地）

| 环节 | 模型 | 说明 |
|------|------|------|
| 文件分类 | `qwen2.5:3b`（Ollama）或关键词匹配 | 本地，无需 API |
| 文档嵌入 | `qwen3-embedding:0.6b`（Ollama） | 已就绪，0.6B 轻量 |
| 语义检索 QA | `qwen2.5:3b` / `qwen3:8b`（Ollama） | 本地，无需 API |

## 7 级递进去重规则引擎（文件整理核心）

> **关键原则**：整理步骤是「筛选式复制」，WPSDocument 原始文件**永不删除/移动**。所有规则只决定「是否复制到知识库文档」。

```
全局递归扫描 WPSDocument (3,579 文件，含所有子目录)
    │
    ▼
┌─ Level 1: 临时文件过滤 ──────────────────────────────┐
│  文件名 ~$+结尾 / .tmp / .wbk / .bak / 自动恢复文件    │
│  → 不复制到知识库文档                                  │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─ Level 2: 同名同大小快速判重 ──────────────────────────┐
│  文件名+扩展名完全相同 + 字节数相同 → 只复制修改时间最新  │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─ Level 3: MD5 绝对重复 ──────────────────────────────┐
│  计算 MD5（比 SHA256 快 3-5 倍）→ 相同即为绝对重复      │
│  → 只复制组内修改时间最新的                              │
│  → 若组内有保护文件，优先复制保护文件中最新的             │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─ Level 4: 格式冗余 ──────────────────────────────────┐
│  同目录/近路径下 .docx≈.pdf 或 .pptx≈.pdf              │
│  → 优先复制可编辑源文件（.docx/.pptx），不复制对应 .pdf  │
│  → 仅存在 .pdf 无对应可编辑格式 → 复制 .pdf             │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─ Level 5: 高相似文本重复 ────────────────────────────┐
│  提取 .docx/.pptx/.pdf 文本 → 余弦相似度 ≥ 95%        │
│  + 文件大小差 ≤ 5% → 只复制修改时间最新的               │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─ Level 6: 老旧历史版本（主题聚类） ────────────────────┐
│  去版本号/日期后 → 文件名相似度 > 80% 或文本相似 > 80%  │
│  → 同主题簇内只复制修改时间最新的                       │
│  → 历史版本 > 2 年且非保护 → 不复制                    │
│  → 孤立文件（不属于任何簇）> 5 年且非保护 → 不复制      │
└───────────────────────────────────────────────────────┘
    │
    ▼
知识库文档（唯一文件集）+ metadata.json
```

### 各规则参数速查

| 规则 | 判断条件 | 复制策略 | 计算成本 |
|:----:|---------|---------|:--------:|
| **L7** 临时文件 | `~$*.tmp`, `.wbk`, `.bak`, 自动恢复 | **不复制** | 零（仅文件名匹配） |
| **L4** 同名同大小 | 文件名+扩展名 + 字节数 | 只复制 1 份（最新） | 零（仅 stat） |
| **L1** MD5 精确去重 | MD5 哈希 | 只复制 1 份（最新，保护优先） | 低（I/O 密集） |
| **L5** 格式冗余 | 同目录 docx/pptx ≈ pdf | 复制可编辑格式 | 低（仅路径匹配） |
| **L3** 高相似文本 | 余弦 ≥ 95% + 大小差 ≤ 5% | 只复制 1 份（最新） | 高（需文本提取+向量化） |
| **L6** 主题聚类 | 去版本号 > 80% 相似 or 文本 > 80% 相似 | 簇内只复制最新 | 最高（分组比对） |

### 实测性能（首次全量运行，3,579 文件 WPSDocument）

| 阶段 | 耗时 | 说明 |
|------|:----:|------|
| 扫描 | **3.2s** | os.walk 递归，2708 个可整理文件 |
| MD5 计算 | **~108s** | 2708 文件×0.04s，纯 I/O 瓶颈 |
| 文本提取 | **~270s** | 2708 文件×0.10s，实际更少（仅 docx/pptx/pdf） |
| 规则匹配 | **<1s** | 内存操作 |
| **首次全量** | **~277s (4.6min)** | 后续增量仅处理变更文件，<1min |

**实测分布**（来自 3,579 文件的 WPSDocument）：
- 扫描 2708 → R7→R4→R1→R5→R3→R6 逐层过滤
- **R4 同名同大小**: 1,257 跳过 ← WPS 云同步导致大量同名文件
- **R3 高相似文本**: 182 跳过 ← 同一文档不同格式/版本
- **R1 MD5 绝对重复**: 128 跳过 ← 完全相同文件
- **R5 格式冗余**: 34 跳过 ← docx/pptx 存在对应 pdf
- **R6 历史版本**: 0 ← 所有文件较新（WPS 同步活跃）
- **R7 临时文件**: 0
- **最终复制**: 1,107 文件

**关键发现**：R4（同名同大小）占据主导地位，因为 WPS 云盘同步会产生大量 `WPS云盘/` 子目录与根目录的同名文件。这是正常的——规则按设计执行。R6 未命中说明文件库较新（都在 2 年内），随着时间推移 R6 会逐渐生效。

### 保护规则（必须复制）

即使被去重规则判定为「不复制」，以下文件**必须复制**到知识库文档：

- config.json 白名单中标记为「受保护」的文件
- 修改时间距今 **2 年内**的文件（不触发规则 L6 的历史版本淘汰）
## 增量逻辑（三阶段贯通）

```
Step ① 增量扫描 WPSDocument
  比对 wpsdoc_cache.json → 识别新增/修改/删除
  缓存文件：cache/wpsdoc_cache.json

Step ② 增量整理
  仅处理 Step ① 识别出的变更文件
  → 7级递进去重 + 分类 → 知识库文档
  缓存文件：cache/organize_cache.json（含规则版本号）

Step ③ 增量向量化
  仅处理 Step ② 新增/修改的文件
  → 文本提取 → 分块 → qwen3-embedding → ChromaDB upsert
  文件哈希缓存 + 嵌入缓存
```

**状态文件**：
| 缓存 | 路径 | 用途 |
|------|------|------|
| WPS扫描缓存 | `cache/wpsdoc_cache.json` | WPSDocument 文件状态快照（路径+大小+修改时间） |
| 整理缓存 | `cache/organize_cache.json` | 每次整理的决策记录 + 规则版本号 |
| 嵌入缓存 | `cache/embedding_cache.json` | 已计算的文本嵌入（MD5 文本哈希键值） |

## 二次整理机制（resync — 规则变更后使用）

当整理规则被修改时（如新增/调整去重规则），已存在于知识库文档中的旧文件可能不再符合新规则。resync 机制解决这个问题：

### 原理

```
RULE_VERSION = "2.0"  ← 在 kb_organize.py 中定义
                       ← 每次修改规则时递增此值
```

**检测**：`_load_cache()` 检查缓存中的 `rule_version` 是否与当前 `RULE_VERSION` 一致。不一致时打印：

```
⚠ 规则已变更: 缓存版本 1.0 → 当前版本 2.0
  建议运行 --resync 进行二次整理
```

### 执行流程

```bash
python3 kb_pipeline.py --resync
```

```
① 重新扫描 WPSDocument（全量，忽略缓存）
② 应用当前 7 级规则 → 得出"应该存在"的文件列表
③ 对比知识库文档现有文件：
   ├ 匹配新规则的 → 保留
   └ 不匹配新规则的 → 移入 _archive/resync_YYYYMMDD/（不删除！）
④ 全量向量化 ChromaDB
```

### 何时使用

| 场景 | 操作 |
|------|------|
| 新增/修改了一条去重规则 | `RULE_VERSION` 递增 + `--resync` |
| 调整了规则参数（如相似度阈值 95%→90%） | `RULE_VERSION` 递增 + `--resync` |
| 新增了一种去重规则 | `RULE_VERSION` 递增 + `--resync` |
| 日常增量运行 | 无需 resync，`kb_pipeline.py` 即可 |

### ✨ 关键设计

- **不动原文件**：所有操作在知识库文档侧，WPSDocument 原文件永不触碰
- **归档不删除**：被清理的文件移入 `_archive/resync_日期/`，可手动恢复
- **规则版本追踪**：递增 `RULE_VERSION` 即触发 resync 提示，无需记忆

## 环境依赖管理

### 首次安装（一次性）

```bash
# 用 Hermes venv 的 pip 安装（不要用系统 pip3）
~/.hermes/hermes-agent/venv/bin/pip install \
  chromadb \
  pypdf \
  python-docx \
  python-pptx \
  requests
```

### 验证

```bash
python3 -c "import chromadb; import pypdf; import docx; import pptx; import requests; print('All OK')"
```

**不要** 在任何脚本中加入运行时 `pip install` 逻辑。

## 故障排查清单

遇到问题时按以下顺序排查：

```
① 环境检查
   python3 -c "import chromadb; import pypdf; import docx; import pptx; import requests; print('OK')"
   → FAIL: 回到「首次安装」步骤

② Ollama 检查
   curl http://localhost:11434/api/tags
   → 确认 qwen3-embedding:0.6b 和 qwen2.5:3b 存在

③ 源头确认
   ls -la /mnt/g/WPSDocument/
   ls -la /mnt/g/WPSDocument/WPS云盘/
   → 确认递归扫描能访问

④ 向量库检查
   ls -lh ~/.hermes_tools/knowledge_base/vector_store/chroma.sqlite3
   → 确认文件存在且非 0 字节

⑤ Cron 日志
   tail -50 ~/.hermes_tools/knowledge_base/logs/cron.log
   → 确认最近一次失败的原因
```

## 实战陷阱（从这次优化中发现）

### 陷阱 9：多进程解析可能卡死（ProcessPoolExecutor hang）

使用 `ProcessPoolExecutor`（`max_workers=4`）做并行文档解析时，在大型 PDF 文件或损坏的文件上可能**永久卡住**（实测在 ~97% 进度处 hang）。

```python
# ❌ 可能卡死的写法
with ProcessPoolExecutor(max_workers=4) as executor:
    futures = {executor.submit(parse_doc, fp): fp for fp in files}
    for future in as_completed(futures):
        result = future.result()  # ← 某个 worker 可能永远不返回
```

**原因**：某些 PDF 文件（特别是扫描件或加密文件）的 pypdf 解析会进入死循环或无限等待。

**解决方案**：

```python
# ✅ 可靠的写法：单进程解析 + 异常保护
all_chunks = {}
for filepath in files:
    try:
        if os.path.getsize(filepath) > 100 * 1024 * 1024:  # 100MB 跳过
            continue
        chunks = DocumentParser.parse(filepath)
        if chunks:
            all_chunks[filepath] = chunks
    except Exception as e:
        print(f"  [!] 解析失败 {Path(filepath).name}: {e}")
```

**性能权衡**：单进程慢但可靠（1125 文件约 4 分钟），多进程快但可能 hang。首次全量用单进程，后续增量用多进程（文件数少）。

### 陷阱 10：后台进程输出缓冲

使用 `terminal(background=true)` 运行长时间任务时，Python print 输出被缓冲：

```python
# ❌ background 模式下看不到实时进度
result = terminal('python3 script.py', background=True)
```

**解决方案**：

```bash
#!/bin/bash
# ✅ 重定向到日志文件
LOG="path/to/log.log"
exec > "$LOG" 2>&1
python3 script.py
```

同时在 Python 中主动 flush：
```python
print("进度信息", flush=True)
sys.stdout.flush()
```

### 陷阱 11：ChromaDB 安装需长超时

```bash
# ❌ 默认超时不够
pip install chromadb  # 失败

# ✅ 需要 --default-timeout=300
pip install chromadb --default-timeout=300
```

chromadb 依赖 onnxruntime、grpcio、opentelemetry、kubernetes 等，总计 >500MB。

### 陷阱 12：全量向量化直接调用底层更可靠

pipeline 层出问题时，直接调用底层引擎：

```python
from kb_vectorize import KnowledgeBase, DocumentParser
kb = KnowledgeBase(kb_root=KB_ROOT)
kb.initialize()
all_chunks = {}
for fpath in files:
    all_chunks[fpath] = DocumentParser.parse(fpath)
kb.vectorize_and_store(all_chunks, dict(file_hashes))
```

pipeline 层适合 cron 调度，但首次全量或修复时直接调底层模块。

---
## 附录 B: 知识库检索指南（从 knowledge-retrieval 合并）

> 本附录提供从 ChromaDB 知识库中做语义检索的操作方法，包括通用搜索策略和人生罗盘（C010）的专用检索规则。

### B1. 核心原则：ChromaDB 优先于直接读文件

当需要了解某个系统或文档内容时，**优先使用语义搜索**而非直接 `read_file`：

| 场景 | 优先操作 | 回退方案 |
|:----|:---------|:---------|
| 了解代码逻辑/架构 | `kb_retrieve --query` | `read_file`（仅当无搜索结果） |
| 查找特定关键词/配置 | `kb_retrieve --query` | grep/search_files |
| 查看系统状态/统计数据 | `kb_retrieve --stats` | `read_file`(状态文件) |
| 需要完整文件的精确内容 | `read_file` | （此场景无可替代） |

### B2. 搜索命令

```bash
# 语义搜索（推荐）
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --query "搜索内容" --limit 5 --json

# 按文件名搜索
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --file-search "关键词" --limit 10 --json

# 查看知识库统计
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --stats
```

### B3. 搜索结果解析

JSON 输出格式含 `results[]` 数组，每项包含：`rank`, `content`, `similarity`, `source_file`, `source_filename`, `source_dir`。
- `similarity` 越高越相关（0-1）
- `content` 是匹配的文档片段（截断到 500 字符）
- 多条结果可来自同一文档的不同片段

### B4. 人生罗盘（C010）检索策略

当用户查询关键人（老苏/翟总/妻子/儿子等）时，优先执行语义搜索：

```bash
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --query "画像 老苏" --limit 5 --json
python3 ~/.hermes_tools/knowledge_base/scripts/kb_retrieve.py --query "老苏 最近 互动" --limit 3 --json
```

**文档类型路由**：

| 用户问题包含 | 路由策略 | 检索目标 |
|:------------|:---------|:---------|
| 关键人名 | 画像优先 + 最近3条洞察 | `portraits/*` + `logs/*` |
| 自我成长/模式/趋势 | 聚合月报，时间倒序 | `reports/self/*` |
| 互动/最近/昨天 | 最新互动日志优先 | `logs/*/*-log.md` |
| 情感/情绪/状态 | 自我镜像 + 自我观察库 | `self-examination/*` + `self-observation-db.json` |
| 策略/汇报/怎么沟通 | 画像策略建议 + 最新报告 | `portraits/*` + `reports/workplace/*` |

**回退策略**：若 ChromaDB 未索引人生罗盘数据，直接读取文件：
```bash
ls -t ~/wiki/raw/systems/life-compass/portraits/workplace/
read_file ~/wiki/raw/systems/life-compass/portraits/workplace/老苏-v1.md
```

### B5. 搜索结果格式化输出模板

```markdown
🔍 找到 [N] 个相关结果：
1. 《文件名》— 位于：[目录]
   → [内容片段摘要...]
```

### B6. 故障排除

| 问题 | 原因 | 解决 |
|------|------|------|
| ChromaDB 无数据 | 未运行向量化 | 运行 `kb_pipeline.py --vectorize-only` |
| 查询嵌入失败 | Ollama 未运行 | `ollama serve` 启动 |
| 搜索结果为空 | 无匹配 | 换同义词或用 `--file-search` |

## 更新日志

- v2.2.0 (2026-04-26): 新增实战陷阱（多进程hang/后台缓冲/Chromadb安装/底层调用）: 新增 resync 机制 + 健康检查
  - 🆕 新增：二次整理机制（resync），规则变更后自动清理旧文件
  - 🆕 新增：RULE_VERSION 规则版本追踪和自动检测
  - 🆕 新增：kb_health.py 一键健康检查（18 项）
  - 🆕 新增：--resync CLI 参数（kb_organize.py + kb_pipeline.py）
  - 📝 新增：resync 原理、执行流程、使用场景说明
  - 📝 修正：缓存文件路径说明（wpsdoc_cache.json / organize_cache.json）
- v2.0.0 (2026-04-26): 大版本重构
  - 🔴 修正：源头从"知识库文档"纠正为"WPSDocument（递归子目录）"
  - ⚠️ 新增：5 大常见陷阱预警（脚本碎片化/源头混淆/子目录遗漏/运行时安装/云端依赖）
  - 🔧 修正：模型策略从 DeepSeek API 改为全本地
  - 📝 新增：增量策略说明和三阶段状态文件描述
  - 🧹 新增：故障排查清单（5 步定位法）
- v1.0.0 (2026-04-19): 初始版本
