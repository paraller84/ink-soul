---
name: chinese-content-architecture
category: education
description: 语文模块内容体系设计 — 内容类型分类、数据模型、拍照导入管线、LLM提示词策略、出题引擎过滤逻辑。
tags: [chinese, c011, content-model, import-pipeline, ocr, data-model, eduhub]
---

# 语文模块内容体系设计

## 定位

C011语文模块（Edu-Hub）的内容层设计规范。覆盖从家长导入到学生练习的完整链路，确保提示词输出、数据存储、出题引擎三者对齐。

---

## 一、内容类型分类

### 完整分类

| 编号 | 内容类型 | 考核方式 | 导入方式 | 练习题型 |
|:----:|:---------|:---------|:---------|:---------|
| 1 | 会认字(recognize) | 看字写拼音 | 拍照/文本 | `char_to_pinyin` |
| 2 | 会写字(write) | 看拼音写字 | 拍照/文本 | `pinyin_to_char` + `meaning_to_char` |
| 3 | 试卷(exam) | 综合 | 拍照 | 自动拆解分流到各类型 |
| 4 | 背诵诗词(poem) | 默写/填空 | 拍照/文本 | `poem_fill` + `poem_full` |
| 5 | 阅读理解(reading) | 文章+问题 | 拍照/文本 | 选择题/判断题/简答题 |

### 拍照导入时的前端分类选择

拍照 Tab 必须提供五种类型的单选按钮，用户选择后：
- 后端根据 `content_type` 选择对应的 LLM 提示词和处理策略
- 提示词决定了 LLM 输出的结构和精度

---

## 二、数据模型 (content.json)

### characters 字段（核心改动）

```json
{
  "char": "存",
  "pinyin": "cún",
  "recognition_type": "recognize",     // "recognize" | "write"
  "word_examples": [
    {"word": "生存", "pinyin": "shēng cún"},
    {"word": "存款", "pinyin": "cún kuǎn"}
  ],
  "source": "photo_import"            // 追踪来源
}
```

**关键原则：数据字段必须与提示词输出字段一一对应。**

| 提示词输出 | 对应字段 | 说明 |
|:----------|:--------|:-----|
| 核心汉字 | `char` | ✓ |
| 汉字拼音 | `pinyin` | ✓ |
| 组词及拼音 | `word_examples[]` | ⚠️ 旧字段 `meaning` 只存文字，丢失拼音 |
| 会认/会写归属 | `recognition_type` | 由前端分类选择 + 提示词共同决定 |

**向后兼容：** 旧 `content.json` 的无 `recognition_type` 字段的字默认视为 `"write"`（全题型出题）。

### poems 字段

```json
{
  "title": "静夜思",
  "author": "李白",
  "dynasty": "唐",
  "text": "床前明月光，疑是地上霜。举头望明月，低头思故乡。"
}
```

### readings 字段（新增）

```json
{
  "title": "金色的草地",
  "passage": "我们住在乡下，窗前是一大片草地……",
  "questions": [
    {"id": 1, "text": "草地为什么是金色的？", "type": "choice|short|judge", "answer": "..."}
  ],
  "difficulty": "⭐⭐",
  "source": "photo_import"
}
```

---

## 三、拍照导入管线（OCR/vision）

### 架构

```
用户选择 content_type → 拍照/上传 → 后端接收
                                       |
                              ┌────────┴────────┐
                              │  策略分发         │
                              │  (ocr_engine.py)  │
                              └────────┬────────┘
                                       |
                    ┌──────────────────┼──────────────────┐
                    |                  |                  |
               write_char       recognize_char       exam
                    |                  |                  |
              Prompt #1           Prompt #2          Prompt #3
          (看拼音写字)         (看字写拼音)         (综合拆解)
                    |                  |                  |
               Markdown解析       Markdown解析       JSON结构提取
                    |                  |                  |
               upsert_batch      upsert_batch       分流入库
```

**注意**：拍照导入直接调用 `upsert_batch(characters=characters)` 入库，**不经过** `import_from_text()` 的文本解析层。

### 各类型的 LLM 提示词

见 `references/import-prompts.md`

### 后处理：Markdown 表格解析

会认字/会写字类型需要从 LLM 输出的 Markdown 表格解析为结构化数据。实际实现分为三层：

**第一层：提取表格（`_parse_markdown_table`）**

```python
def _parse_markdown_table(table_text: str) -> list[dict[str, str]]:
    lines = [l.strip() for l in table_text.split("\n") if l.strip()]
    if not lines:
        return []
    table_start = None
    for i, line in enumerate(lines):
        if line.startswith("|") and "---" in line:
            table_start = i
            break
    if table_start is None or table_start < 1:
        return []
    header_line = lines[table_start - 1]
    headers = [h.strip() for h in header_line.strip("|").split("|")]
    results = []
    for line in lines[table_start + 1:]:
        if not line.startswith("|"):
            break
        cells = [c.strip() for c in line.strip("|").split("|")]
        row = {headers[i]: cells[i] for i in range(min(len(headers), len(cells)))}
        results.append(row)
    return results
```

**第二层：按类型解析**

```python
def _parse_write_char_result(table_rows):
    # Prompt #1: 拼音 | 对应汉字
    chars = []
    for row in table_rows:
        pinyin = row.get("拼音", "").strip()
        char = row.get("对应汉字", "").strip()
        if not pinyin or not char:
            continue
        chars.append({
            "char": char, "pinyin": pinyin,
            "recognition_type": "write",
            "word_examples": [], "source": "photo_import",
        })
    return chars

def _parse_recognize_char_result(table_rows):
    # Prompt #2: 核心汉字 | 汉字拼音 | 组词及拼音
    import re
    chars = []
    for row in table_rows:
        char = row.get("核心汉字", "").strip()
        pinyin = row.get("汉字拼音", "").strip()
        if not char or not pinyin:
            continue
        word_examples = []
        raw = row.get("组词及拼音", "")
        if raw:
            for w, wp in re.findall(r"([\u4e00-\u9fff]+)\(([^)]*)\)", raw):
                if w.strip():
                    word_examples.append({"word": w.strip(), "pinyin": wp.strip()})
        chars.append({
            "char": char, "pinyin": pinyin,
            "recognition_type": "recognize",
            "word_examples": word_examples, "source": "photo_import",
        })
    return chars
```

**第三层：入口函数（`recognize_document_page`）**

按 `content_type` 分发：选择提示词 → 压缩图片(1024px, 85%) → 调用 qwen3.5-256k（num_predict=2048, 120s timeout） → 解析 Markdown 表格 → 类型特定后处理 → 返回 characters list。

⚠️ 2026-05-14 踩坑（三层问题）：
  - L1（表象）前端"网络错误"：手机照片 3-10MB → base64 7-14MB POST 体 → fetch 超时。修复：客户端 canvas 压缩 1200px@75%。
  - L2（功能）导入失败"无法解析"：OCR 引擎原硬编码 qwen3-vl:4b/qwen3-vl:8b 为字符串字面量。升级 Ollama 后仍在调用旧模型名。
  - L3（静默）qwen3.5-256k 的 API 输出在 `thinking` 字段而非 `response` 字段，`_call_ollama()`/`_call_ollama_document()` 只读 `response` 导致空结果。
  修复详见约束注册表 C021+C022。

---

## 四、文本导入格式

`import_from_text()` 支持以下输入格式，适用于手动粘贴和家长编写的结构化文本：

```
课时: Y3U1
标题: 第一课
会认字: 晨-chén-早晨(zǎo chén),绒-róng-绒毛(róng máo)
会写字: 付-fù,出-chū
生字: 旧-jiù (backward compat)
词语: 早晨-zǎo chén
古诗: 静夜思-唐-李白-床前明月光
```

**关键规则：**
- `会认字:` 和 `会写字:` 使用 **追加模式**（`characters += _parse_character_list(...)`），同一课时内可以共存
- `生字:` 使用 **覆盖模式**（`characters = _parse_character_list(...)`），兼容旧导入行为
- 顺序：先出现的类型先处理，后出现的追加，除非碰到 `生字:` 则覆盖

---

## 五、出题引擎过滤逻辑

`practice/character.py` 的 `generate_questions()` 中增加：

```python
def generate_questions(characters, student_id, batch_id, count=12):
    # ... 按频次筛选 ...

    for c in selected:
        recognition_type = c.get("recognition_type", "")

        if recognition_type == "recognize":
            # 会认字 → 只看字写拼音
            qtype = "char_to_pinyin"
            question = make_char_to_pinyin_q(c)

        elif recognition_type == "write":
            # 会写字 → 看拼音写字 + 组词填空
            r = random.random()
            if r < 0.6:
                qtype = "pinyin_to_char"
            else:
                qtype = "meaning_to_char"
            question = make_write_q(c, qtype)

        else:
            # 旧数据（无类型）→ 三种题型随机（兼容）
            question = make_legacy_q(c)

        questions.append(question)
```

**频次模型不受影响** — 所有字符共用同一套 error_count → freq_level 逻辑，区别只在题型范围。

---

## 六、常见陷阱

### 6.1 数据模型与提示词不对齐

`meaning: "生存"` 会丢失"存款"及其拼音。建新字段 `word_examples` 时必须保证每对(word, pinyin)完整。**关键原则：数据字段必须与提示词输出字段一一对应。**

### 6.2 旧数据兼容

无 `recognition_type` 的字不能报错也不能只出一种题型，应默认全题型出题。实现方式：

```python
def normalize_character(char: dict) -> dict:
    result = dict(char)
    if "recognition_type" not in result or not result["recognition_type"]:
        result["recognition_type"] = "write"  # 旧数据视为会写字
    if "word_examples" not in result:
        result["word_examples"] = []
    if not result["word_examples"] and char.get("meaning"):
        result["word_examples"] = [{"word": char["meaning"], "pinyin": ""}]
    if "source" not in result:
        result["source"] = "migrated"
    return result
```

所有读取数据的接口（`get_batch_content()`）必须在返回前调用 `normalize_character()`。注意：这只会标准化已加载的数据，不会持久化到 `content.json`。

### 6.3 会认字/会写字共轭

同一个字可能同时出现在会认字表和会写字表。以最后导入的类型为准，不合并。

### 6.4 试卷导入后分流

从试卷中提取出的生字要追加到已有课时中，如该字已存在则更新 `recognition_type`。

### 6.5 文本导入中会认字/会写字共存陷

当同一课时既有会认字又有会写字时，`import_from_text()` 的解析必须使用 `+=` 追加，而非 `=` 覆盖：

```python
# ✅ 追加模式 — 会认字和会写字可以共存
m = re.match(r"^会认字\s*[::]\s*(.+)$", line)
if m:
    characters += _parse_character_list(m.group(1), recognition_type="recognize")

m = re.match(r"^会写字\s*[::]\s*(.+)$", line)
if m:
    characters += _parse_character_list(m.group(1), recognition_type="write")
```

### 6.6 拍照导入直接走 upsert_batch

拍照导入不经过 `import_from_text()`，而是：
```python
ocr_result = recognize_document_page(content, content_type=content_type)
characters = ocr_result["characters"]
upsert_batch(batch_id=batch_id, characters=characters, ...)
```

### 6.7 Python 3.11 全角字符编译问题（关键坑）

**症状：** Python 3.11 on Ubuntu 24.04 在编译包含全角标点 `（）`、`，`、`：` 的 `.py` 源文件时，会抛出 `SyntaxError: invalid character '（' (U+FF08)`，即使这些字符位于字符串/注释内部。

**根因：** Python 3.11 tokenizer 对 Unicode 类别判断有边界情况，全角标点在特定条件下（尤其当 `"` 与全角字符相邻时）被误判为非字符串内容。

**修复方法**（当 CI/部署环境无法升级 Python 时）：

方案 A - 构建时自动替换：
```bash
find . -name '*.py' -exec sed -i \
  -e 's/\xef\xbc\x88/(/g' -e 's/\xef\xbc\x89/)/g' \
  -e 's/\xef\xbc\x8c/,/g' -e 's/\xef\xbc\x9a/:/g' \
  {} +
```

方案 B - 源码中手动替换：
```python
# 在 .py 文件中避免使用全角标点
# （ → (   ） → )   ， → ,   ： → :   ； → ;   、 → ,
```

**对于 `"""` 结构的影响：** 替换全角字符后，必须检查模块文档字符串的闭合 `"""` 是否被意外附加到行末。正确结构应为：
```python
"""模块文档...

内容存储在: xxx/content.json
"""
import json  # 确保 """ 在独立行
```

**检测脚本：** 见 `scripts/check-fullwidth-in-py.sh`

**经验教训（2026-05-13）：** 该问题的首次排查花了大量时间，因为：
1. 全角标点在其他 Python 版本和平台上均无问题，仅限于 Ubuntu 的 Python 3.11 特定构建
2. 隔离测试时 `compile('''测试（中文）''')` 正常通过，但融入大文件时触发
3. 需要通过字节级别的 `"""` 状态机调试，而不是依赖源码阅读
4. 删除 `.pyc` 缓存后问题才会暴露（缓存掩盖了问题存在）

---

## 七、参考文件

- `references/import-prompts.md` — 五种内容类型的 LLM 提示词（完整可复制版本）
- `references/data-model-migration.md` — 从旧 `meaning` 字段迁移到 `word_examples` 的脚本和步骤
