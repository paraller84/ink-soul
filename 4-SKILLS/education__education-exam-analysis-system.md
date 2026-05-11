---
name: education-exam-analysis-system
category: education
description: "Build an educational exam analysis → knowledge point extraction → practice generation → targeted exercise closed-loop system. Includes knowledge graph construction, Feishu→Wiki integration, and three-phase weakness remediation."
---

# 小学教育知识点分析与智能出题系统

Build a complete closed-loop education system: exam input → structured parsing → knowledge point mapping → practice generation → result analysis → targeted three-phase remediation.

## When to Use

- User wants to analyze exam papers and extract knowledge points
- User needs to generate practice problems based on specific knowledge points
- User wants to track student weaknesses and create targeted exercises
- User has a Feishu education folder and wants automatic sync + analysis

## Architecture

```
Feishu Drive folder (数学目录)
    │ C002 sync pipeline detects new docs
    ▼
exam-engine.py analyze <text>   ← 试卷分析引擎
    │
    ├─ Step 1: Parse (分题型/分题/结构化)
    ├─ Step 2: Map to knowledge graph (30 topics across 9 units)
    ├─ Step 3: Coverage report (unit distribution, difficulty profile)
    │
    ▼
exam-engine.py generate <topic_ids>  ← 练习生成
    │
    ├─ Based on identified weak topics
    ├─ Difficulty progression: basic → medium → hard
    │
    ▼
exam-engine.py analyze-results <json>  ← 答题分析
    │
    ├─ Per-topic mastery calculation
    ├─ Weak point identification (mastery < 70%)
    │
    ▼
Three-phase targeted exercises:
    Phase 1: Concept rebuild (3 questions, mastery < 40%)
    Phase 2: Standard practice (5 questions, mastery < 70%)
    Phase 3: Challenge & solidify (3 questions, mastery < 85%)
```

## Step-by-Step

### 1. Create Project Structure

```bash
mkdir -p ~/edu-hub/{knowledge-graph,templates,output,scripts}
```

### 2. Build Knowledge Graph (YAML)

Create `~/edu-hub/knowledge-graph/gradeX-semesterY-subjectZ.yaml`:

```yaml
metadata:
  grade: 三年级
  semester: 下学期
  subject: 数学

units:
  - id: unit-01
    name: 两位数乘两位数
    topics:
      - id: multiply-2digit-by-2digit
        name: 两位数乘两位数（竖式计算）
        difficulty: basic
        tags: [竖式, 乘法, 计算]
        description: 掌握竖式计算方法
```

**Structure rules:**
- Each unit has 1-6 topics
- Each topic has: id, name, difficulty (basic/medium/hard/cross-cutting), tags, description
- Add `questions_ref` to link to known exam questions
- Add `is_cross_cutting: true` for meta-skills (审题, 验算)

### 3. Build Exam Engine

Create `~/edu-hub/scripts/exam-engine.py` with these functions:

| Function | Purpose | Key Logic |
|----------|---------|-----------|
| `load_knowledge_graph()` | Load YAML | `yaml.safe_load()` |
| `parse_exam(text)` | Parse raw exam text | Section header regex + keyword type detection |
| `identify_knowledge_points(content)` | Map question to topics | Keyword scoring (desc keywords match + tag match) |
| `analyze_exam(text)` | Full analysis | Parse → Map → Report |
| `generate_practice_plan(ids)` | Generate practice outline | Templates per topic |
| `generate_targeted_exercises(weaknesses)` | Three-phase plan | mastery < 40/70/85 thresholds |
| `analyze_results(results)` | Analyze answers | Per-topic correct/wrong/mastery |

### 4. Integrate with Feishu Sync

**IMPORTANT - learn from past mistakes:**

1. **.env file may be PROTECTED** — cannot write to it directly. Instead:
   ```python
   # Use state file for token fallback
   state_path = "~/.hermes/scripts/feishu-wiki-sync-state.json"
   with open(state_path) as f:
       state_data = json.load(f)
       _EDU_FOLDER_TOKEN = state_data.get("edu_folder_token", "")
   ```

2. **Update feishu-wiki-sync.py** — Add education folder to FOLDERS list:
   ```python
   if _EDU_FOLDER_TOKEN:
       FOLDERS.append({
           "name": "教育-小学三年级下学期数学",
           "token": _EDU_FOLDER_TOKEN,
           "wiki_section": "飞书导入",
           "tags": ["feishu-import", "feishu-sync", "education", "math-grade3"],
       })
   ```

3. **Sync scope** — `list_folder_docs(ftoken)` only returns DIRECT children (not recursive). Set the sync folder token to the specific sub-folder (e.g., the 数学 folder), not the root 教育 folder.

4. **SAFETY: rewrite entire file instead of patching** — The sync script has complex string literals (`"""`, `\`, `'`, `"`) that cause patch escaping issues. When adding a new folder:
   ```bash
   # Read original, modify, write entire file — safer than patch
   cp ~/.hermes/scripts/feishu-wiki-sync.py ~/.hermes/scripts/feishu-wiki-sync.py.bak
   # Then write the full updated content
   ```

### 5. Register in Capability Registry

Add C00X to `~/wiki/entities/capability-registry.md` with:
- State (active), domain, tags
- End-to-end flow diagram
- Asset table (scripts, knowledge graph, feishu folder)
- Current limitations
- Changelog

### 6. Update Wiki Index & Schema

- Add to `~/wiki/index.md` → Registry table + education section
- Add education tags to `~/wiki/SCHEMA.md` → Phase 5 tags section

## Knowledge Point Matching Strategy

Use keyword scoring rather than simple regex:

```python
desc_keywords = {
    "竖式": "竖式" in question_content or "×" in question_content,
    "递等式": "递等式" in question_content,
    "面积": "面积" in question_content,
    "速度": "速度" in question_content or "每分钟" in question_content,
}
# Score: desc keyword match = +2, tag match = +1
# Threshold: confidence >= 2 matches → link question to topic
```

**Known limitation**: This over-matches for simple operators like `×`. For better accuracy, use LLM-based semantic matching instead.

## Three-Phase Remediation Model

| Phase | Trigger | Count | Focus |
|-------|---------|-------|-------|
| Concept Rebuild | mastery < 40% | 3 questions | Foundational concepts, simplest form |
| Standard Practice | mastery < 70% | 5 questions | Error pattern targeting |
| Challenge | mastery < 85% | 3 questions | Variant forms + integrated application |

## Vision-Based OCR Parsing (raw-exam-parser)

For parsing **photographed math worksheets** (phone photos of printed/handwritten papers), replace Tesseract with a **vision model** (qwen3-vl:4b via Ollama). Tesseract produces garbled output for Chinese math worksheets with mixed symbols, while vision models extract content perfectly.

### Architecture

```
原始题/ (user uploads images)
    │ cron 8:00/20:00 or manual trigger
    ▼
raw-exam-parser.py
    ├── Download image from Feishu Drive
    ├── Resize to ≤1600px (4MB → ~230KB)
    ├── qwen3-vl:4b via /api/generate + images[]
    │   └── reads from `thinking` field (not `response`)
    ├── Parse output: separate questions + wrong items
    ├── Create Feishu doc in 解析试题/
    ├── If wrong answers found → create doc in 错题本/
    └── Delete original file
```

### Critical: qwen3-vl Thinking Renderer

The qwen3-vl:4b model uses a **thinking renderer** that puts content in a non-standard field:

```python
# Ollama API - use /api/generate (NOT OpenAI-compatible /v1/chat/completions)
payload = {
    "model": "qwen3-vl:4b",
    "prompt": prompt,
    "images": [base64_image],
    "stream": False,
    "options": {"temperature": 0.1, "num_predict": 4096}
}
req = urllib.request.Request("http://ollama-host:11434/api/generate", ...)
result = json.loads(resp.read())
# ⚠️ Content is in 'thinking' field, NOT 'response'!
content = result.get("response", "") or result.get("thinking", "")
```

**Why**: The OpenAI-compatible endpoint (`/v1/chat/completions`) returns empty `content` because the thinking renderer puts output in a separate field. Always use `/api/generate` with `images: [base64]` and read from `thinking`.

### Image Resizing

4MB phone photos produce ~5.6MB base64 payloads that the model can't process reliably:

```python
from PIL import Image
img = Image.open(io.BytesIO(original_bytes))
if max(w, h) > 1600:
    ratio = 1600 / max(w, h)
    img = img.resize((int(w*ratio), int(h*ratio)), Image.LANCZOS)
    buf = io.BytesIO()
    img.save(buf, format="JPEG", quality=85)
    img_bytes = buf.getvalue()  # ~230KB
```

### Wrong Answer Detection

Include in the vision model prompt to detect wrong answers automatically:

```python
prompt = """...【任务2】识别出所有做错/答错的题目（打×、有更正标记、答题处明显错误的）。
输出格式：
---
【题目内容】
...
【错题分析】
错题1：题目...
错误原因：...
正确答案：...
---"""
```

### Feishu Drive API Quirks

1. **`size=0` bug**: Files uploaded by users through shared folders show `size=0` in listing, but `download` returns full content. **Never skip files based on reported size.**

2. **0-byte stubs after DELETE**: DELETE leaves entries that can't be overwritten with the same filename. **Solution**: Create a new folder instead and redirect the parser.

3. **New folder creation**: Create with `POST /open-apis/drive/v1/files/create_folder` with `folder_token` in BODY. Share with user via `POST /open-apis/drive/v1/permissions/{token}/members?type=folder`.

### Title Sanitization

The model's thinking output often starts with first-person reasoning. Filter it:

```python
# Skip lines that are model self-reference
if re.search(r'我[需要要]|首先[，,]|任务[12]|仔细分析|让我', line):
    continue
# Then look for actual exam patterns
if re.search(r'\d{4}学[年]|练习|试卷|测试|考试|期末|期中|单元', line):
    return line.strip()
```

### File Path Sanitization

Always sanitize titles used in filenames to prevent `FileNotFoundError`:

```python
safe_title = re.sub(r'[\\/:*?"<>|]', '', title)[:80]
```

### Patch Tool Limitation

The `patch` tool corrupts Python triple-quoted strings (`"""` → `\"\"\"`). Always follow up with:

```bash
sed -i 's/\\"/"/g' script.py
```

### Performance

- ~35 seconds per image (vision model inference)
- 25 images ≈ 15 minutes total
- Use `background=true` with `notify_on_complete=true`

## Pitfalls

1. **patch damage on sync script**: Always rewrite the full file rather than patching when adding new folders to `feishu-wiki-sync.py`. The `"""` and escaped strings cause patch corruption.

2. **.env file protection**: The `.env` file may be write-protected by the system. Store folder tokens in the state JSON file as fallback.

3. **Feishu folder listing is NOT recursive**: `list_folder_docs()` only returns direct children. Set sync folder to the leaf (subject) folder, not the root education folder.

4. **Knowledge point overlap**: Simple `×` detection matches ALL multiplication questions. For better accuracy, combine with section context (e.g., "填空" + "×" + "末尾" → multiply-with-trailing-zeros).

5. **CLI execution**: The exam engine reads from files or stdin (`analyze <file>`), not from piped text directly. Use `analyze-text "content"` for inline analysis.

## Wiki Exam Import Workflow

For repeatedly adding raw exam/worksheet content to the wiki as structured pages:

### File Naming Convention
```
raw/education/exams/NN-descriptive-name.md
```
- Use sequential numbers (01, 02...) or emoji identifiers (🏁1, ①...)
- Descriptive English/Chinese name separated by hyphens

### Page Structure Template
```markdown
# 试卷标题

**标签:** `三年级` `关键知识点1` `关键知识点2`

---

*(full exam content, sections preserved)*

---

**涉及知识点：** 逗号分隔的知识点列表
```

### Index Table Update

Update `~/wiki/index.md` → `### 试卷库（原始题，无答案版）` section:

```markdown
| 编号 | [试卷名](raw/education/exams/NN-name.md) | 核心知识点(用·分隔) |
```

**Critical formatting rules:**
- Each row starts with `| 编号 |` — exactly ONE pipe prefix, no fewer, no more
- `patch` tool can introduce `||` (double pipe) or `|||` (triple pipe) artifacts — always verify the table alignment by reading the result
- Fix triple pipes with: `patch(old="||| ⑨", new="| ⑨")`

### Verification Commands
```bash
# Check table formatting is consistent
grep -n '||' ~/wiki/index.md | head -20  # Should only show header separator row with ||:---||

# Count total entries
grep -c 'raw/education/exams/' ~/wiki/index.md
```

## Targeted Practice Test Generation & Deployment

When creating a **focused/考前冲刺 practice test** (not daily practice, not generic review) based on a specific skill gap identified from exam analysis:

### 1. Design Coverage

A well-rounded practice test should cover **7 dimensions** of the target skill:

| # | Dimension | Example (多位数除两位数) | Purpose |
|:--:|:----------|:------------------------|:--------|
| 1 | **估算/试商热身** | `156÷13` 把13看成10试商 | Build number sense |
| 2 | **竖式计算** | `576÷24`, `429÷17` (含验算题) | Core procedural fluency |
| 3 | **商的位数判断** | 前两位≥除数→两位数, <除数→一位数 | Meta-cognitive check |
| 4 | **最大能填几** | `27×( )<200` | Reverse trial quotient |
| 5 | **列式计算** | "A里面有几个B""A除B" | Language→math translation |
| 6 | **应用题** | 装箱有余数、路程时间、倍数 | Real-world application |
| 7 | **纠错题** | 判断竖式正确+订正 | Common error pattern detection |

### 2. Difficulty Progression

Structure each skill section from easy → hard:
- Easy: numbers that divide evenly, small quotients
- Medium: remainder, larger numbers, mixed operations
- Hard: multi-step, error detection, word problems

### 3. Construction Checklist

Before creating the test file:
- Pull 3-5 relevant exam papers from `~/wiki/raw/education/exams/` to identify common question types
- Check the review book (`~/wiki/guides/education/grade3-math-review-book.md`) for key formulas and pitfalls
- Design 5-8 questions per dimension
- Include "易错点速查表" at the end
- Include "self-check checklist" for the student

### 4. Wiki Storage

Save to `~/wiki/raw/education/exams/NN-名称.md` (next sequential number):
```bash
cp ~/wiki/guides/education/practice.md ~/wiki/raw/education/exams/10-名称.md
rm ~/wiki/guides/education/practice.md
```

Update `~/wiki/index.md` 试卷库 table with next emoji/number:
```markdown
| 🎯 | [专项名称](raw/education/exams/10-名称.md) | 核心知识点用·分隔 |
```

**⚠️ Patch formatting hazard:** After updating, always verify no `|||` artifacts.

### 5. Deploy to Feishu Drive

```python
# Step 1: Create doc in the correct folder
FOLDER_TOKEN = "<folder_token_from_memory_or_state>"
body = {"title": "标题", "folder_token": FOLDER_TOKEN}
req = urllib.request.Request(
    "https://open.feishu.cn/open-apis/docx/v1/documents",
    data=json.dumps(body).encode(), headers=headers, method="POST")
resp = json.loads(urllib.request.urlopen(req).read())
doc_token = resp["data"]["document"]["document_id"]

# Step 2: Write content using feishu-md-writer.py
# cd ~/.hermes/scripts && python3 feishu-md-writer.py <doc_token> <md_file>

# Step 3: Share with user (full_access)
user_open_id = "<user_open_id_from_memory>"
share_body = {"member_type": "openid", "member_id": user_open_id, "perm": "full_access"}
req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx&need_notification=true",
    data=json.dumps(share_body).encode(), headers=headers, method="POST")
urllib.request.urlopen(req)

# Step 4: Provide the URL to the user
doc_url = f"https://rcncjr4hag8s.feishu.cn/docx/{doc_token}"
```

### 6. Verify

- [ ] Wiki page exists at correct path
- [ ] Index table updated and table formatting verified
- [ ] Feishu doc created in correct sub-folder
- [ ] User has full_access permission
- [ ] Provided the URL to the user in chat

## Verification

```bash
# Test exam analysis
python3 ~/edu-hub/scripts/exam-engine.py kp-list  # List all knowledge points

# Analyze an exam
python3 ~/edu-hub/scripts/exam-engine.py analyze /tmp/exam.txt

# Generate practice plan
python3 ~/edu-hub/scripts/exam-engine.py generate multiply-2digit-by-2digit speed-formula

# Test Feishu sync
python3 ~/.hermes/scripts/feishu-wiki-sync.py 2>&1 | head -20

# Check capability registry
grep -c "^## C" ~/wiki/entities/capability-registry.md
```
