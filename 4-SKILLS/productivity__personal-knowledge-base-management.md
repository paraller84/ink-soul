---
name: personal-knowledge-base-management
description: 个人知识库 RAG 系统 — WPSDocument（递归子目录）→ 整理 → 知识库文档 → 增量向量化 → ChromaDB 检索
version: 2.4.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [knowledge-base, document-management, automation, RAG, chromadb, ollama, WPS, file-organization]
    related_skills: [knowledge-base-audit-and-rationalize, capability-registry-management, llm-wiki, email-to-knowledge-pipeline]
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
| **① 输入源** | `G:/WPSDocument/` | **5,508 文件** | 原始文件，含子目录 |
|  | 根目录/ | 1,664 | 根目录直接存放的文件 |
|  | WPS云盘/ | 3,500+ | WPS 自动同步的子目录（含邮件） |
|  | 我的邮件/ | 17 | WPS 导出邮件 |
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
│   │   └── 邮件/                   # ← 2026-05 新增！用户手动同步的邮件归档
│   │       └── EAST邮件/           #    290+ .eml 文件（持续增长）
│   └── 我的邮件/                   # ← 容易漏掉！WPS 导出邮件
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
│   ├── kb_pipeline.py       # ⑤ 主流水线入口 EnvCheck→扫描→整理→向量化→报告 [已就绪]
│   ├── trust_labeler.py     # ⑥ 信任层级判定模块（L1/L2/L3规则引擎） [v1.0.0 / 2026-05-19]
│   └── batch_label_existing.py # ⑦ 存量文件批量标注信任层级 [v1.0.0 / 2026-05-19]
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
| 缓存 | 实际路径 | 文档中写的路径 | 用途 |
|------|---------|---------------|------|
| WPS扫描缓存 | `~/.hermes_tools/knowledge_base/wpsdoc_cache.json` | ~~cache/wpsdoc_cache.json~~ | WPSDocument 文件状态快照（路径+大小+修改时间），**代码写的是 parent 目录而非 cache/ 子目录** |
| 整理缓存 | `~/.hermes_tools/knowledge_base/cache/organize_cache.json` | — | 每次整理的决策记录 + 规则版本号 |
| 嵌入缓存 | `~/.hermes_tools/knowledge_base/cache/embedding_cache.json` | — | 已计算的文本嵌入（MD5 文本哈希键值） |

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

> 📎 参考文件：
> - `references/ollama-host-fix-and-cron-setup.md` — OLLAMA_HOST 修复、crontab 路径修正、缺失 cron 创建、强制全量重建
> - `references/eml-file-ingestion.md` — .eml 邮件文件→知识库的三步法：复制→文本提取→附件提取→向量化
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

### 陷阱 13：OLLAMA_HOST 硬编码为 Docker 桥接 IP（⚠️ 高频故障）

**所有 4 个脚本**（kb_pipeline.py / kb_health.py / kb_retrieve.py / kb_vectorize.py）硬编码了
`OLLAMA_HOST = os.getenv("OLLAMA_HOST", "http://172.19.144.1:11434")`。

**问题**：`172.19.144.1` 是 Docker 桥接 IP，WSL 重启或 Docker 重启后可能变化。
如果 Ollama 监听 `localhost:11434` 但脚本连的是 Docker IP，**健康检查通过但全流程失败**。

**修复命令**（一次性）：
```bash
sed -i 's|http://172.19.144.1:11434|http://localhost:11434|g' \
  /home/openclaw/.hermes_tools/knowledge_base/scripts/kb_pipeline.py \
  /home/openclaw/.hermes_tools/knowledge_base/scripts/kb_health.py \
  /home/openclaw/.hermes_tools/knowledge_base/scripts/kb_retrieve.py \
  /home/openclaw/.hermes_tools/knowledge_base/scripts/kb_vectorize.py
```

**预防**：设置环境变量 `OLLAMA_HOST=http://localhost:11434` 覆盖默认值。

### 陷阱 14：`--full` / `--organize-only` / `--vectorize-only` 不触发增量扫描

⚠️  pipeline 只有**默认模式（无参数）**会执行 `scan_wpsdocument_incremental()`。
`--full`、`--organize-only`、`--vectorize-only`、`--resync` 这些**带 flag 的模式都跳过扫描步骤**，
直接进入组织器/向量化器。

**后果**：如果 `wpsdoc_cache.json` 不存在或被清理，运行 `--full` 并不会重新扫描——
它会加载空缓存、走组织器、报告「0 变更」，然后退出。

**正确做法**：要强制重新扫描所有 5,508 文件，必须先删除缓存，再执行默认模式：

```bash
# ❌ 无效
python3 kb_pipeline.py --full

# ✅ 正确
rm -f ~/.hermes_tools/knowledge_base/wpsdoc_cache.json
rm -f ~/.hermes_tools/knowledge_base/cache/organize_cache.json
python3 kb_pipeline.py         # ← 无参数，触发完整增量扫描
```

**原理**：`scan_wpsdocument_incremental()` 对比 `wpsdoc_cache.json` 的签名（size:mtime）。
缓存为空 → 所有 5,508 文件都标记为「新增」→ `has_changes=True` → 依次触发组织器和向量化器。

### 陷阱 15：Cron 最小 PATH 下 Python 找不到 Hermes venv

系统 crontab 使用最小 PATH（`/usr/bin:/bin`），其中 `python3` 是系统 Python 3.12.3
（不包含 chromadb/pypdf/docx）。Hermes venv python 在
`/home/openclaw/.hermes/hermes-agent/venv/bin/python3`（Python 3.11.15，含依赖）。

**症状**：cron.log 报「缺失依赖: chromadb, docx, pptx」但交互式 shell 下 pip list 全在。

**修复**：crontab 中用绝对路径替换裸 `python3`：
```bash
# crontab 条目示例（注意不要用 ~，cron 不展开）
0 23 * * * cd /home/openclaw && /home/openclaw/.hermes/hermes-agent/venv/bin/python3 \
  /home/openclaw/.hermes_tools/knowledge_base/scripts/kb_pipeline.py --report \
  >> /home/openclaw/.hermes_tools/knowledge_base/logs/cron.log 2>&1
```

### 编者注：邮件处理已独立为 email-to-knowledge-pipeline

`.eml` 文件的处理已从本技能中剥离，由独立的 `email-to-knowledge-pipeline` 技能管理。该技能包含：

- **五级分类系统**：needs_reply / needs_attention / valuable / routine / discard
- **Foxmail 自动导出**：公司电脑 → WPS 云盘 → Hermes
- **LLM 分类引擎 + 规则回退**：全本地运行
- **邮件时间线提取**：从会议通知/纪要中提取关键事件

本技能负责分类后的 KB 向量化环节（流水线 Step 3）。新的 `.eml` 文件进入 WPSDocument 后：
1. 先由 email-to-knowledge-pipeline 分类
2. 有价值的保留在原位 → KB 管线通过增量扫描自动拾取
3. 无价值的移入 `_routine/` 或 `_discard/`，不进 KB

### 陷阱 16：`.eml` 邮件文件进入知识库但无专用分类

`WPSDocument/WPS云盘/邮件/` 和 `WPSDocument/我的邮件/` 中的 `.eml` 文件会被流水线递归扫描。
但 kb_organize.py 的 7 级去重规则按扩展名分类，`.eml` 文件不属于
PDF/Word/PPT/会议纪要/方案文档/汇报材料/其他 七类中的任何一类，全部落入「其他」类别。

**影响**：不会丢数据（`_archive` 只去重不丢弃），但知识库文档目录中
「其他」类别会膨胀。

**解决方案**：
1. **短期**：在 organize 规则中新增 `.eml` → 邮件归档 的映射
2. **长期**：使用 `email-to-knowledge-pipeline` skill 在邮件进入KB前完成五级分类（valuable→进KB，routine→仅存档，discard→不保留），分类后的邮件再走标准KB管线

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

## 知识入库策略 v1.0（2026-05-19 新增）

知识管线的上游质量由「知识入库策略」文档定义。该文档位于飞书云盘能力体系文件夹，定义了：

### 准入控制

| 门禁 | 规则 | 信任层级 |
|:----|:-----|:---------|
| ✅ **白名单** | 正式工作文档／纪要／你修正的稿／历史回复／系统设计文档 | L1 |
| ⚠️ **观察** | C013 会议原始待办／阅读笔记 | L2 |
| ❌ **禁入** | 原始转录（未处理）／临时草稿／外部第三方未验证文件／纯技术输出 | 不入库 |
| 🔒 **L3 暂存** | 网络爬取／未分类邮件／AI未审阅输出（超过30天无人审阅→自动移除） | L3 |

### 三层信任体系

- **L1·已验证区**（可信度 0.8-1.0）：你确认/实践验证的内容 → Agent 可直接引用
- **L2·待验证区**（可信度 0.4-0.7）：文档/被动接收/AI分析 → Agent 引用时标注"待你确认"
- **L3·暂存区**（可信度 <0.4）：外部未验证输入 → Agent 不主动引用

### 生命周期

活跃（60天内被引用≥1次）→ 观察（60天未引用，降权但可检索）→ 归档（180天未引用，移出ChromaDB）

### 与 kb_pipeline 的集成关系

**Phase 1+2 已全部完成（2026-05-19）：**

| 阶段 | 内容 | 状态 |
|:----|:-----|:----:|
| Phase 1 | 策略定义（本文档上方的准入/L1-L3/生命周期） | ✅ 已完成 |
| Phase 2 | 代码实现 | ✅ 已完成 |

**Phase 2 实现详情：**

在 `kb_organize.py` 中新增 `_write_metadata()` 方法，整理文件时同步将信任层级写入 `/知识库文档/metadata.json`；在 `kb_vectorize.py` 中读取 metadata.json，L3 文件自动跳过向量化，所有入库文件在 ChromaDB metadata 中携带 `trust_level` 字段。

**管线新增数据流：**

```
kb_organize (整理) → _execute_copy() → _write_metadata()
                                              ↓
                                       metadata.json ← batch_label_existing.py (存量标注)
                                              ↓
kb_vectorize (向量化) → 扫描文件 → 查 trust_level
                                   L3 → 跳过（不入 ChromaDB）
                                   L1/L2 → 向量化 + trust_level 写入 metadata
```

**Agent 使用 ChromaDB 检索时**，`source_filename` 和 `trust_level` 均在 metadata 中，Agent 可按以下规则引用：

| trust_level | 引用方式 |
|:-----------|:---------|
| L1 | 直接引用，无需标注置信度 |
| L2 | 引用时标注"待你确认" |
| L3 | 不主动引用（已在向量化阶段跳过，ChromaDB 中不存在 L3 数据） |

📎 **策略文档**：[知识入库策略_v1.html](https://nofile.feishu.cn/file/HbXNbCfAloZ9E0xDm6ScBoqKndB)（飞书云盘·能力体系文件夹）

---

## 更新日志

- v2.4.0 (2026-05-19): Phase 2 信任标注+禁入检查代码实现已完成
  - 新增 trust_labeler.py（L1/L2/L3 规则引擎，路径+文件名模式匹配）
  - 新增 batch_label_existing.py（存量 2,541 文件批量标注脚本）
  - kb_organize.py 新增 _write_metadata()，整理时输出信任层级到 metadata.json
  - kb_vectorize.py 加载 metadata.json，L3 文件跳过向量化，ChromaDB metadata 含 trust_level
  - kb_pipeline.py 报告新增信任层级统计行
  - 存量 2,541 文件已标注：L1=53 / L2=2481 / L3=7
- v2.3.1 (2026-05-19): 新增知识入库策略章节（准入/L1-L3/生命周期），关联飞书文档
- v2.3.0 (2026-05-12): 新增 OLLAMA_HOST / Cron / --full 陷阱 + .eml 分类缺口
  - 🚨 新增陷阱 13: OLLAMA_HOST 硬编码 Docker IP（4 脚本痛点）
  - 🚨 新增陷阱 14: `--full`/`--organize-only`/`--vectorize-only` 跳过增量扫描
  - 🚨 新增陷阱 15: Cron 最小 PATH 下 Python 找不到 Hermes venv
  - 🚨 新增陷阱 16: `.eml` 邮件文件进入知识库但无专用分类
  - 📝 修正: `wpsdoc_cache.json` 路径（代码写 parent 目录而非 cache/）
  - 📝 修正: 文件计数 3,579→5,508；新增 `WPS云盘/邮件/` 子目录说明
  - 📎 新增: `references/ollama-host-fix-and-cron-setup.md` 参考文件
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
