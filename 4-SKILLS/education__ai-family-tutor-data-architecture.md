---
name: ai-family-tutor-data-architecture
category: education
description: AI家庭教师项目数据架构 v5.0 — 三层体系（L2题库→L3练习库→L4计划+执行）+ LLM驱动的上下文包出题模式 + Agent/App职责分离 + library_service只读模式
tags: [ai-family-tutor, data-model, sqlite, question-bank, llm-context, agent-app-split, practice-plan, three-layer, v5]
related_skills: [photo-import-prompt-design, design-document-production, flask-sqlite-migration]
---

# AI家庭教师数据架构 v5.0

## 项目定位

AI家庭教师 (`~/edu-hub/ai-family-tutor/`) 是一个独立的 Flask 项目，与 Edu-Hub (C003/C008/C011) 是两套系统。数据层使用 **SQLite**。端口 **5003**。

> **架构演进：v4.1（两层：题库→计划）→ v5.0（三层：题库→练习库→计划）**
> 核心改变：练习包预制化（practice_library），非实时凑题；schedule简化为只存library_id+date+status

## 一、三层数据体系

数据按职能分为四层，从原材料到成品逐步加工：

```
L1 内容源（原材料）       L2 分析层            L3 题库（缓存）       L4 计划+执行
┌─────────────────┐   ┌───────────────┐   ┌───────────────┐   ┌──────────────────┐
│ textbook_*      │   │ exam_analysis │   │ question_bank │   │ practice_library  │
│ exam_*          │──→│ (LLM分析产出)  │──→│ (预生成缓存)   │──→├──────────────────┤
│ training_*      │   └───────────────┘   │ 双轨难度      │   │ practice_library  │
│ knowledge_pts   │                       │ 新陈代谢      │   │ _questions        │
└─────────────────┘                       └───────────────┘   ├──────────────────┤
      ↑ Agent写                               ↑ Agent写       │ practice_schedule │
      └── OCR导入+AI分析                        └── 预生成       ├──────────────────┤
                                                                 │ practice_records  │
                                                                 └──────────────────┘
                                                                     ↑ App读写
```

**三层核心逻辑（L2→L3→L4）：**

```
question_bank（题库 — 所有题的池子）
      ↓ Agent 打包
practice_library（练习库 — 预制完整内容包）
  ├── "每日练习·5月18日"      → 15题   (practice_type=daily)
  ├── "第一单元测试"          → 全部生词 (practice_type=unit_test)
  └── "挑战：近义词辨析第17课" → 5题    (practice_type=challenge)
      ↓ Agent 排程
practice_schedule（计划 — 哪天做哪个库）
  5月18日 → library_id=3     status: assigned
  5月20日 → library_id=5     status: in_progress
```

**角色划分（核心设计决策）：**

| 层 | 谁写 | 谁读 | 说明 |
|:---|:-----|:-----|:------|
| L1 内容源 | Agent | Agent | 拍照OCR课本/试卷 → 结构化存入 |
| L2 分析层 | Agent(LLM) | Agent | Agent调DeepSeek分析试卷 → 描述性JSON |
| L3 题库层 | Agent(LLM) | Agent打包 | Agent每周预生成批题 → Agent打包到练习库 |
| L3 练习库 | Agent | App只读 | Agent预先生成完整内容包 → App读库展示 |
| L3 计划层 | Agent | App读写+记录 | Agent分配日期→App读口读+记录答题结果 |

## 二、数据库：完整31张表

### 表清单

```
  31-32张表：
  existing (18): families, students, student_titles, practice_sessions, practice_answers,
                 wrong_questions, questions, english_words, chinese_chars, chinese_words,
                 lesson_texts, page_lesson_map, knowledge_points, coin_transactions,
                 coin_approvals, daily_checkins, learning_plans, sqlite_sequence
  v5.0 new (13): textbook_units, textbook_lessons, textbook_chars, textbook_words,
                 textbook_lesson_texts, exam_papers, exam_questions, exam_analysis,
                 question_bank, practice_library, practice_library_questions,
                 practice_schedule, practice_records
  Phase 2-3 (2): student_settings, practice_records.answer_choices
```

### L1 内容源（6张表）

#### exam_papers — 试卷卷头
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | |
| semester | TEXT | 如'3下' |
| title | TEXT | 试卷名称 |
| exam_date | TEXT | 考试日期 |
| total_score | INTEGER | 满分，默认100 |
| score_earned | INTEGER | 实际得分 |
| question_count | INTEGER | 题目数 |
| raw_ocr_text | TEXT | OCR原始输出 |
| created_at | TEXT | |

#### exam_questions — 试卷原始题目
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| paper_id | INTEGER FK | |
| question_number | INTEGER | 题号 |
| question_type | TEXT | 题型 |
| question_text | TEXT | |
| student_answer | TEXT | 学生作答 |
| correct_answer | TEXT | 标准答案 |
| score | INTEGER | 分值，默认1 |
| is_correct | INTEGER | 0/1 |
### 学生设置（Phase 2 新增）

#### student_settings — 家长出题偏好
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK UNIQUE | 一对一关系 |
| current_lesson | INTEGER | 当前课号，默认1 |
| semester | TEXT | 学期，如'3下' |
| difficulty_pref | INTEGER | 家长偏好难度(1-5)，默认3 |
| created_at | TEXT | |
| updated_at | TEXT | |

**设置页面**：`/parent/settings/<student_id>` 供家长调整课号/难度/学期。
**影响范围**：Agent出题管线的固定窗口法（C-1, C, C+1）基于 `current_lesson`。

### L2 内容分析

#### exam_analysis — 试卷分析结果
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| paper_id | INTEGER FK | |
| student_id | INTEGER FK | |
| semantic_version | TEXT | 分析版本，默认v1.0 |
| analysis_data | TEXT(JSON) | 完整分析结果（题型分布/知识点覆盖/难度/出题规律/样本） |
| created_at | TEXT | |

**analysis_data JSON 结构示例：**
```json
{
  "question_type_breakdown": {"choice": {"count": 8, "score": 24, "percentage": 24}},
  "knowledge_point_coverage": {"会认字": {"type_ids": [1,5], "total_score": 12}},
  "difficulty_assessment": {"easy": "40%", "medium": "40%", "hard": "20%", "description": "..."},
  "question_patterns": "字词部分从词库选常见词语，阅读理解偏好叙事类课文",
  "representative_samples": [1, 3, 8]
}
```

#### textbook_units / textbook_lessons / textbook_chars / textbook_words

**⚠️ 关键陷阱：`textbook_chars` 和 `textbook_words` 使用 `lesson_id`（外键 → `textbook_lessons.id`），而非直接存储 `lesson_number`。**

```sql
-- ❌ 错误：直接查 lesson_number 会报 OperationalError: no such column
SELECT * FROM textbook_chars WHERE lesson_number = 18;

-- ✅ 正确：需 JOIN textbook_lessons 才能获取 lesson_number
SELECT tc.char, tc.pinyin, tl.lesson_number
FROM textbook_chars tc
JOIN textbook_lessons tl ON tc.lesson_id = tl.id
WHERE tl.lesson_number = 18 AND tc.char_type = 'write';
```

这是该项目的**最高频 SQL 陷阱**——设计文档中和实际数据库中的列名可能存在差异。任何查询 `textbook_chars` 或 `textbook_words` 的 SQL 都应先 `PRAGMA table_info` 验证。

#### textbook_chars 表结构（实际）

| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | 学生ID |
| lesson_id | INTEGER FK | → textbook_lessons.id（不是 lesson_number！） |
| semester | TEXT | 如 '3下' |
| char | TEXT | 汉字本身 |
| pinyin | TEXT | 拼音 |
| char_type | TEXT | 'recog' 会认 / 'write' 会写 |
| strokes | INTEGER | 笔画数 |
| radical | TEXT | 部首 |
| structure | TEXT | 结构 |

#### textbook_words 表结构（实际）

| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | |
| lesson_id | INTEGER FK | → textbook_lessons.id |
| semester | TEXT | |
| word | TEXT | 词语 |
| pinyin | TEXT | 拼音（⚠️ 可能被截断为最后一个字拼音，见下方陷阱） |

⚠️ **textbook_words pinyin 质量陷阱**：第18课（童年的水墨画）的 pinyin 字段被截断为最后一个字拼音（如"墨水"→"mò"而非"mò shuǐ"），共10词受影响。**缓解**：使用 `pypinyin` 从 word 字段重建全拼音。

结构同 v4.1 旧设计，见 `references/data-structure-design-v5.html` 完整定义。

### L3 题库层（核心新表）

#### question_bank — 统一题库（预生成缓存+统计追踪器）
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | |
| subject | TEXT | 学科：chinese/math/english |
| question_type | TEXT | 题型编码 |
| difficulty_assigned | INTEGER | LLM生成时标注的初始难度(1-5) |
| difficulty_effective | REAL | 基于答题数据校准的有效难度(1.0-5.0) |
| question_text | TEXT | 题目文字 |
| choices | TEXT(JSON) | 选项数组（选择题用） |
| answer | TEXT | 标准答案 |
| explanation | TEXT | 解析 |
| knowledge_point | TEXT | 知识点 |
| **lesson_number** | **INTEGER** | **对应课号（2026-05-17新增），回填源：chinese_words.lesson / textbook_words→textbook_lessons.lesson_number** |
| source | TEXT | 来源：ai_generated/exam/exercise |
| status | TEXT | pending/active/dormant/archived |
| total_attempts | INTEGER | 累计作答次数 |
| correct_attempts | INTEGER | 累计正确次数 |
| created_at | TEXT | |
| last_used_at | TEXT | 最后一次使用时间 |

**双轨难度校准：**
```
difficulty_effective = 1 + (1 - correct_rate) × 4
correct_rate = correct_attempts / total_attempts
条件: total_attempts ≥ 3 才启用有效难度，不足则沿用 assigned
```

**新陈代谢机制：**
```
pending → active（Agent审核或首次使用后）
active → dormant（correct_rate > 90% 或 < 30%，且 ≥3次答题记录）
dormant → archived（dormant 状态 ≥30天未触达）
```

### L3 练习库层（v5.0 新引入）

#### practice_library — 预制练习包
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | |
| practice_type | TEXT | daily/unit_test/lesson_test/challenge |
| title | TEXT | 标题，如"每日练习·5月18日" |
| subject | TEXT | 学科，默认chinese |
| question_count | INTEGER | 总题数，默认15 |
| coverage | TEXT | 覆盖范围，如"第1-4课" |
| status | TEXT | active/archived |
| created_at | TEXT | |

**设计要点：** 这是一个预生成的、不可变的内容包。不是实时凑题。同一练习库可被多天的 schedule 引用。

**practice_type 枚举值：**
| 类型 | 场景 | 规则 |
|------|------|------|
| `daily` | 每日练习 | 15题固定，基于 student_settings.current_lesson 的 [C-1,C,C+1] 三课窗口 |
| `unit_test` | 单元测验 | 覆盖整单元全部课的全部生词（30-62题不等） |
| `lesson_test` | **每课一练** | 每课一套独立练习，约18题（10拼音+6选音+2填空），按课逐日排期 |
| `challenge` | 挑战题型 | 薄弱知识点针对性练习，5-8题 |

#### practice_library_questions — 练习包-题目关联（纯映射）
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| library_id | INTEGER FK | |
| question_id | INTEGER FK | |
| sort_order | INTEGER | 排序 |
| UNIQUE | | (library_id, question_id) |

### L4 计划+执行层

#### practice_schedule — 练习日程（极简版）
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | |
| library_id | INTEGER FK | 引用预制的练习包 |
| schedule_date | TEXT | 计划日期，如'2026-05-18' |
| status | TEXT | assigned(未开始)/in_progress(进行中)/completed(已完成) |
| started_at | TEXT | 第一次打开时间 |
| completed_at | TEXT | 全部完成时间 |
| **agent_analysis** | **TEXT** | **Agent评估JSON（Phase 3新增）：完成时由LLM生成** |

**agent_analysis JSON 结构：**
```json
{
  "accuracy": 73,
  "total_questions": 15,
  "correct_count": 11,
  "evaluation": "整体表现良好，字词基础扎实...",
  "type_stats": {
    "看拼音写词语": {"total": 7, "correct": 6},
    "会认字选读音": {"total": 5, "correct": 3}
  },
  "wrong_highlights": ["承 (认)", "担 (心)"],
  "diff_suggestion": "建议保持当前难度",
  "next_difficulty": 3
}
```

**v4.1→v5.0 简化：** 移除了 practice_type、title、question_count、coverage、semester、unit_range、lesson_range、generated_by_agent 等字段。全部通过 library_id 外键从 practice_library 获取。

**状态流转：**
```
assigned（Agent预生成分配）
  ↓ 学生首次打开练习页
in_progress（做了一部分）
  ↓ 全部题目提交完成
completed（不再补做）
```

**补做判断（核心逻辑）：**
```sql
-- 今天及之前所有未完成的练习
SELECT ps.id, pl.title, pl.practice_type, pl.question_count,
       ps.status, ps.schedule_date
FROM practice_schedule ps
JOIN practice_library pl ON ps.library_id = pl.id
WHERE ps.student_id = ?
  AND ps.schedule_date <= ?
  AND ps.status != 'completed'
ORDER BY ps.schedule_date ASC
-- 补做标记：schedule_date < today → is_makeup=True
```

#### practice_records — 答题记录
| 字段 | 类型 | 说明 |
|:-----|:-----|:------|
| id | INTEGER PK | |
| student_id | INTEGER FK | |
| schedule_id | INTEGER FK | 关联练习计划 |
| question_id | INTEGER FK | 关联题库题目 |
| question_type | TEXT | 冗余题型（展示方便） |
| question_text | TEXT | 冗余题目文本 |
| user_answer | TEXT | 学生答案 |
| correct_answer | TEXT | 标准答案 |
| is_correct | INTEGER | 0/1 |
| attempt_number | INTEGER | 第几次作答，默认1 |
| response_seconds | INTEGER | 答题耗时(秒) |
| answered_at | TEXT | |

**设计要点：** 字段冗余（question_type、question_text、correct_answer）是为了避免结果页查询时频繁 JOIN question_bank。以写入时快照为准。

## 三、library_service：只读库服务模式

```
library_service.py 是 v5.0 的核心模式创新：App层完全不生成内容，只读取。
```

### 接口矩阵

| 函数 | 用途 | 查询模式 |
|:-----|:-----|:---------|
| `get_today_schedule(student_id)` | 首页"今日练习"卡片 | practice_schedule + practice_library JOIN |
| `get_schedule_detail(schedule_id)` | 练习内容详情 | schedule + library + questions 三表JOIN + **lesson_name 批量注入** |
| `submit_schedule_answer(schedule_id, question_id, user_answer, is_correct, student_id)` | 记录答题+更新schedule状态 | INSERT practice_records + UPDATE schedule |
| `get_schedule_result(schedule_id, student_id)` | 结果报告 | practice_records 聚合查询 |

**lesson_name 批量注入模式：** `get_schedule_detail()` 在获取题目列表后，收集所有非空 `lesson_number` → 单次查询 `textbook_lessons` 构建 {ln→title} dict → 遍历问题集为每个 question 注入 `lesson_name` 字段。模板渲染时即可显示 `第X课·课程名称`。

```python
# get_schedule_detail() 中的模式
lesson_nums = set(q.get('lesson_number') for q in questions if q.get('lesson_number'))
lesson_names = {}
if lesson_nums:
    placeholders = ','.join('?' * len(lesson_nums))
    ls_rows = db.execute(
        f"SELECT lesson_number, lesson_title FROM textbook_lessons WHERE lesson_number IN ({placeholders})",
        tuple(lesson_nums)
    ).fetchall()
    for ls in ls_rows:
        lesson_names[ls['lesson_number']] = ls['lesson_title']

for q in questions:
    ln = q.get('lesson_number')
    q['lesson_name'] = lesson_names.get(ln, '') if ln else ''
```

### 关键陷阱

**coverage 字段 JSON 安全解析：**
```python
cov = r.get('coverage')
r['coverage'] = json.loads(cov) if cov and (cov.startswith('{') or cov.startswith('[')) else cov
```
→ coverage 字段可能是普通文本（如"第一课-第四课"）也可能是JSON数组。必须用 startswith 判定，不能直接 json.loads。

## 四、4区域报告页（Phase 3 新增）

### 报告架构

```
┌──────────────────────────────────────┐
│ Region 1: 小朋友做题情况              │
│  正确/错误统计 → 错题列表 + 高亮标记  │
├──────────────────────────────────────┤
│ Region 2: Agent评估                  │
│  AI评语 + 各题型正确率分布 + 薄弱推荐 │
├──────────────────────────────────────┤
│ Region 3: 下周难度影响               │
│  趋势图 + 难度调整建议 + 当前难度显示 │
├──────────────────────────────────────┤
│ Region 4: 家长参数调整               │
│  确认/调高/调低按钮 → 更新settings    │
└──────────────────────────────────────┘
```

### 触发流程

```python
# 在 schedule_answer 路由中，当所有题目答完后触发
if response_data['done']:
    from services.report_v2_service import generate_agent_analysis
    generate_agent_analysis(schedule_id, student_id)
    # → 调用LLM生成评估 → 写入 practice_schedule.agent_analysis
    return redirect(url_for('student.schedule_report_v2', schedule_id=schedule_id))
```

### 相关文件

| 文件 | 说明 |
|:-----|:------|
| `services/report_v2_service.py` | 报告生成服务（get_schedule_report / get_recent_schedule_history / generate_agent_analysis） |
| `templates/student/report_v2.html` | 4区域报告模板 |
| `templates/student/history.html` | 历史练习记录列表 |

### 路由与模板结构（v5.0 完整版）

| 端点 | 用途 | 模板 |
|:-----|:-----|:------|
| `GET /student/practice/schedule/<id>` | 开始/续做（跳到第一个未做题） | redirect |
| `GET /student/practice/schedule/<id>/q/<index>` | 展示题目（支持手写格子+选择题） | schedule_practice.html |
| `POST /student/practice/schedule/<id>/answer` | 提交答案（form+JSON双模式） | redirect |
| `GET /student/practice/schedule/<id>/result` | 练习报告（旧版） | schedule_result.html |
| `GET /student/schedule/<id>/report` | **v2报告（4区域+Agent分析）** | report_v2.html |
| `GET /student/history` | **历史练习记录列表** | history.html |

**核心逻辑：** 进入练习时先查 practice_records 已答 question_id 列表，跳过分到第一个未做题（或全做完→结果页）。

**⚠️ 模板变量上下文陷阱：** `schedule_result.html` 接收 `report=report` 作为模板上下文，所有变量必须用 `report.` 前缀：
```jinja
{# ❌ 错误 — 变量名在嵌套层里 #}
{{ schedule.title }}
{% if wrong %}
{{ accuracy|int }}%

{# ✅ 正确 #}
{{ report.schedule.title }}
{% if report.wrong %}
{{ report.accuracy|int }}%
```
修改 Flask 上下文结构后，务必全面检查所有模板中的变量引用，包括 `{% if %}` 条件判断。

### submit_answer 双模式支持
```python
if request.is_json:
    data = request.get_json() or {}
else:
    data = request.form or {}
```
前端 Handwriting 格子用 form POST，后续手机端可用 JSON API。

## 五、Agent/App 责任矩阵（v5.0 版）

| 操作 | 谁做 | 涉及表 | 触发时机 |
|:-----|:-----|:-------|:---------|
| OCR识别课本/试卷 | Agent | textbook_*, exam_* | 用户发照片 |
| AI分析试卷→exam_analysis | Agent(LLM) | exam_analysis | OCR后自动触发 |
| LLM出题→入库 | Agent(LLM) | question_bank | 每周 cron |
| 打包练习库 | Agent | practice_library, practice_library_questions | 出题后 |
| 排程计划 | Agent | practice_schedule | 打包后 |
| 显示今日练习+补做 | App(只读) | practice_schedule + practice_library | 首页 |
| 展示题目 | App(只读) | practice_library_questions + question_bank | 练习页 |
| 记录答题结果 | App(写) | practice_records | 提交答案 |
| 更新schedule状态 | App(写) | practice_schedule | 开始答题/完成 |
| 显示练习统计 | App(读) | practice_records 聚合 | 结果页/统计页 |

## 六、数据完整性验证

### 6.1 pinyin_to_word 拼音完整性校验

**问题背景：** question_bank 中 `pinyin_to_word` 类型题目的 `question_text` 格式为 `看拼音写词语：<拼音>`。拼音分词数以空格分隔，必须 ≥ answer 的字数，否则学生只看单字拼音写多字词。

**校验规则：** `len(拼音.split()) ≥ len(answer)`。即拼音空格段数应 ≥ 答案字数。

**质量门禁方案（正式模块，推荐）：** `services/question_quality.py` 中的 `validate_pinyin_format()` 返回 `(valid, errors, warnings)` 三元组。

```bash
# 手动运行每日校验
bash /home/openclaw/.hermes/scripts/validate-ai-tutor-questions.sh

# 或直接调用 Python 模块
cd ~/edu-hub/ai-family-tutor && python -c "
from app import create_app
from services.question_quality import validate_question_bank
app = create_app()
with app.app_context():
    from db import get_db
    result = validate_question_bank(get_db())
    print(f'{result[\"passed\"]}/{result[\"total\"]} passed')
    if result['warnings']:
        print(f'⚠️ {len(result[\"warnings\"])} warnings (轻声等)')
    if result['errors']:
        print(f'❌ {len(result[\"errors\"])} errors')
"
```

详见 `services/question_quality.py`（含 check_data_contract 全题型校验、check_db_integrity）。

**修复方法：** 用 `pypinyin` 从 answer 字段重建完整拼音：

```python
from pypinyin import pinyin, Style

def word_to_pinyin(word):
    result = pinyin(word, style=Style.TONE)
    return ' '.join(p[0] for p in result)

# 检测并修复
rows = db.execute("""SELECT id, question_text, answer FROM question_bank 
    WHERE question_type='pinyin_to_word'""").fetchall()
for r in rows:
    after_colon = r['question_text'].split('：', 1)[1] if '：' in r['question_text'] else r['question_text']
    if len(after_colon.strip().split()) < len(r['answer'] or ''):
        correct = word_to_pinyin(r['answer'])
        db.execute("UPDATE question_bank SET question_text=? WHERE id=?", 
                   (f'看拼音写词语：{correct}', r['id']))
```

详见 `scripts/validate-pinyin-integrity.py`。

### 6.2 answer_char_count 字段约定

`schedul_practice.html` 中手写格子数量依赖 `question.answer_char_count`：

```html
<div class="hw-cells" data-count="{{ question.answer_char_count or (2 if question.question_type == 'pinyin_to_word' else 3) }}">
```

**陷阱：** 当 `get_schedule_detail()` 不计算此字段时，3字词（如"叠花篮"）默认只生成 2 个格子，学生少写 1 个字却只能提交 2 个字。

**修复：** 在 `get_schedule_detail()` 中补充：

```python
for q in questions:
    q['answer_char_count'] = len(q.get('answer', '') or '')
```

所有通过 `question_bank` → 模板展示的路径都必须补上此字段。

### 6.4 lesson_number 回填映射 & lesson_name 注入

**lesson_number（question_bank 字段，2026-05-17 新增）：** 每题对应的课号，用于在练习页显示「第X课」来源标签。回填为一次性迁移操作，但对新生成的题同样适用。

**lesson_name（运行时注入，非持久字段）：** 在 `get_schedule_detail()` 中通过批量查询 `textbook_lessons` 动态注入到每题的 `q.lesson_name`，模板渲染为 `第16课·宇宙的另一边`。

| 题型 | 映射路径 | 示例 |
|:-----|:---------|:-----|
| `pinyin_to_word` | `answer` → `chinese_words.word` → `.lesson` | answer="葫芦" → WHERE word='葫芦' → lesson=18 |
| `choose_pinyin` | 从 `question_text` 提取目标字 → `textbook_chars.char` → JOIN `textbook_lessons.lesson_number` | 题干"给'秘'选正确读音" → char='秘' → lesson=18 |
| `fill_blank` | 从 `answer` 取首字 → `textbook_chars.char` (同上) | answer="是" → char='是' → lesson=17 |
| 回退 | 无法映射时 lesson_number=NULL（不影响使用） | 外部来源词、未导入的字 |

**校验脚本模式：**
```python
sql = '''
UPDATE question_bank SET lesson_number = (
  SELECT cw.lesson FROM chinese_words cw WHERE cw.word = qb.answer LIMIT 1
) WHERE qb.question_type = 'pinyin_to_word' AND qb.lesson_number IS NULL
  AND qb.answer IN (SELECT word FROM chinese_words)
'''
rows_affected = db.execute(sql).rowcount
db.commit()
```

详见 `scripts/add_lesson_source.py`（本会话的完整实现）。

### 6.5 质量门禁体系总览（2026-05-19 新增）

| 门禁维度 | 模块 | 检测时机 | 阻断级别 |
|:---------|:-----|:---------|:---------|
| **D6 数据契约** | `services/question_quality.py` | 写入 question_bank、每日 cron 04:00 | 硬阻断（错误）/ 软提醒（轻声） |
| **拼音完整性** | `validate_pinyin_format()` | 同上 | 硬阻断（音节缺少）/ 软提醒（轻声） |
| **选择题选项** | `check_data_contract()` | 写入时 | 硬阻断（答案不在选项/选项重复） |
| **数据库完整性** | `check_db_integrity()` | 每日 cron | 硬阻断（PRAGMA integrity_check） |
| **pinyin_to_word 批量扫描** | `validate_question_bank()` | 每日 cron | 报告级（不阻断，异常告警） |

**文件清单：**
- `services/question_quality.py` — 核心验证模块
- `scripts/validate_question_data.py` — 独立运行脚本（入口）
- `~/.hermes/scripts/validate-ai-tutor-questions.sh` — cron 包装脚本
- cron: `5d086ebbf4cc` — 每日 04:00 静默运行（delever=local，无异常不通知）

**调用方式：**
```bash
# 验证整库
python3 scripts/validate_question_data.py

# 验证特定题型
python3 -c "from services.question_quality import validate_pinyin_format; print(validate_pinyin_format('看拼音写词语：lǐ zi', '李子'))"
# → (True, [], ['轻声 \'zi\' 无调号，合法但注意'])
```

`/student/practice/clear-today` 端点涉及 **4 表协调操作**：

| 操作 | 表 | 说明 |
|:-----|:---|:------|
| 🗑️ DELETE | `practice_records` | 按当天 schedule_ids 删除答题记录 |
| 🔄 UPDATE | `practice_schedule` | status → 'assigned', clear started_at/completed_at |
| 🗑️ DELETE | `practice_answers` | 旧系统按当天 session_ids 删除 |
| 🗑️ DELETE | `practice_sessions` | 旧系统当天创建的 sessions 删除 |

**关键逻辑：**
1. 查出今天该学生的 schedule_ids（`WHERE schedule_date=? AND student_id=?`）
2. DELETE practice_records + UPDATE practice_schedule 用同一组 ID
3. 旧系统 session 用 `WHERE date(started_at)=today` 定位
4. 前端加 `confirm()` 弹窗防误触

**适用范围：** 测试/家长管理场景。线上部署需加家长 PIN 校验保护。

---

## 七、LLM驱动出题（上下文包模式）

v4.1 的7模块上下文包设计保持不变。主要变化：

- **新上下文源**：practice_records 中的 attempt_number 支持多次尝试统计
- **防重范围**：用 practice_library 中最近N天的题去重
- **单元测试覆盖规则**：单元测试 type=unit_test 必须覆盖该单元全部生词

## 八、试卷闭环系统（exam closed-loop）

### 核心目的（4条用途，用户确认）

| 用途 | 数据路径 | 说明 |
|:-----|:---------|:------|
| **① 题库补充** | exam_questions → question_bank (source='exam') | 试卷中的看拼音写词语/选择题直接转为题库题，加 source 标签区分 |  
| **② 出题依据** | exam_analysis → LLM 上下文包 → 新题生成 | AI 分析试卷的题型分布/难度/知识点覆盖 → 作为 Agent 出题的上下文源 |
| **③ 专项练习** | parent 选择题型 → question_bank 筛同型题 → practice_library | 家长从试卷中选择特定题型 → 从 question_bank 筛选同类题目打包成专项练习 |
| **④ 错题同步** | exam_questions(is_correct=0) → wrong_questions | 试卷中的错题自动进入错题本，与日常练习错题合并展示 |

### 数据流图

```
试卷拍照 → exam_papers（卷头）
    │
    │ OCR+人工校正
    ▼
exam_questions（原始题目）
    │
    ├─①→ question_bank（source='exam'）→ 题库可检索
    ├─④→ wrong_questions（is_correct=0）→ 错题本
    │
    │ LLM 分析
    ▼
exam_analysis（题型分布/难度/薄弱分析）
    │
    ├─②→ Agent 出题上下文包 → 更精准的新题
    └─③→ 家长选题型/知识点 → question_bank 筛选 → practice_library（专项练习）
```

### 三层留存策略

| 层 | 存储位置 | 用途 | 谁用 |
|:---|:---------|:-----|:-----|
| L0 原始 OCR | exam_papers.raw_ocr_text | 人工校对回退 | 家长 |
| L1 结构化题目 | exam_questions | 逐题展示、错题提取 | App 展示 + Agent 分析 |
| L2 分析产出 | exam_analysis | AI 建议 | Agent 出题引擎 |

### 映射到 question_bank 的规则

**选择题（choice 型）：**
- question_type = 'choice' 
- question_text = 题干完整文字
- choices = JSON 数组 [A项, B项, C项, D项]
- answer = 正确选项字母（如 'C'）
- source = 'exam'

**看拼音写词语（pinyin_to_word 型）：**
- question_type = 'pinyin_to_word'
- question_text = '看拼音写词语：拼音'
- answer = 词语
- source = 'exam'

**课文填空（fill_blank 型）：**
- question_type = 'fill_blank'
- question_text = 含空格的题干
- answer = 答案
- source = 'exam'

**阅读理解（reading_comp 型）：**
- 不直接入 question_bank（涉及短文正文+多子题）
- 存储方式：sub_question 设计或将整个阅读题存入 practice_library 作为专项包

**修改病句/改句子（综合主观题）：**
- question_type = 'sentence_form'
- answer = 标准答案
- source = 'exam'

### 错题同步策略

```sql
-- exam_questions → wrong_questions 同步规则
-- 条件：is_correct = 0（答错）且 question_type ≠ 'reading_comp'
INSERT INTO wrong_questions (student_id, question_type, question_text, correct_answer, wrong_answer, source, exam_paper_id, created_at)
SELECT student_id, question_type, question_text, correct_answer, student_answer, 'exam', paper_id, datetime('now')
FROM exam_questions
WHERE paper_id = ? AND is_correct = 0;
```

### 家长端专项练习生成模式

1. 家长在试卷详情页看到题型分布（饼图/统计图）
2. 点击某个题型（如「看拼音写词语·错4题」）
3. 系统从 question_bank 筛选同题型题目（不限试卷来源）
4. 筛选规则：student_id=? AND question_type=? AND status='active'
5. 打包为 practice_library（practice_type='challenge', title='专项练习：看拼音写词语'）
6. 创建 practice_schedule 排入计划
7. 家长可设置难度过滤（difficulty_assigned ≤ ?）

### 参考架构

exam 相关表的数据流向已在 `references/data-structure-design-v5.html` 完整设计文档中定义。当前（2026-05-18）三表已建但无数据，需 OCR 导入管线就绪后填充。

---

## 九、standalone SQLite 脚本注意事项

### 陷阱：get_db() 需要 Flask 应用上下文

`db.py` 的 `get_db()` 使用 Flask `g` 对象，在独立运行的脚本中会抛出 `RuntimeError: Working outside of application context`。

**解决方案**：

```python
from app import create_app
app = create_app()
with app.app_context():
    db = get_db()
    # 所有数据库操作放在这里
```

### 陷阱：忘记 db.commit()

独立运行的 SQLite 脚本（migration、seed、pipeline、debug）中执行了 INSERT 但忘记调用 `db.commit()`，导致数据看似写入（代码无报错）但实际未持久化。

### 陷阱：DB Schema 设计文档 vs 实际差异

设计文档中的某些字段在实际 DB 中不存在。常见差异：

| 表 | 设计文档有 | 实际无 | 影响 |
|:---|:-----------|:-------|:-----|
| `question_bank` | `lesson_number`,  `coverage` | `lesson_number` ✅ **已添加 (2026-05-17)** | ★ 经回填85题后上线，覆盖第5、6单元全部题目 |
| `practice_schedule` | `created_at` | ❌ 不存在 | INSERT 时去掉 `created_at` |
| `practice_library` | `student_id` (设计文档有) | ✅ 实际有 | 保持使用即可 |

**验证方法：** 操作前先 `PRAGMA table_info(<表名>)` 确认实际列。

### 陷阱：复杂多步 DB 操作用 `.py` 脚本文件而非 heredoc

多步骤 DB 操作（插入多表、条件判断、循环）时，用 `write_file` → `terminal("python3 script.py")` 两步法替代 heredoc：

```bash
# ❌ 不推荐：heredoc 内嵌复杂逻辑，容易因引号/缩进/特殊字符出错
python3 << 'PYEOF'
... 复杂多表操作 ...
PYEOF

# ✅ 推荐：写入 .py 脚本文件后执行
write_file path=scripts/task_xxx.py content="""..."""
python3 scripts/task_xxx.py
```

heredoc 在以下情况会误报语法错误：`sorted()` 输出被误认为 `&` 后台标记、全角字符触发 tokenizer 边界问题。

```python
# ❌ 错误：没有 commit，数据不会持久化
db.execute("INSERT INTO practice_schedule ...")
print("创建今日计划 id=1")  # 脚本退出后数据消失

# ✅ 正确：必须提交
db.execute("INSERT INTO practice_schedule ...")
db.commit()
print("创建今日计划 id=1")  # 数据已落盘
```

**验证方法**：
```bash
sqlite3 instance/tutor.db "SELECT * FROM practice_schedule;"
```

### 避免背压包导入路径问题

独立运行脚本建议用完整文件路径导入 `sys.path.insert(0, os.path.dirname(os.path.dirname(__file__)))`，不要用相对导入。

### Cron 集成模式

Hermes cron 的 `no_agent` 模式要求脚本位于 `~/.hermes/scripts/`，并且不支持参数传递。需要创建 shell wrapper：

```bash
#!/bin/bash
# ~/.hermes/scripts/ai-tutor-weekly-pipeline.sh
cd /home/openclaw/edu-hub/ai-family-tutor || exit 1
export DASHSCOPE_API_KEY=$(grep DASHSCOPE_API_KEY ~/.bashrc | head -1 | cut -d= -f2- | tr -d '"')
export TUTOR_DB_PATH="/home/openclaw/edu-hub/ai-family-tutor/instance/tutor.db"
python scripts/agent_pipeline.py --student-id 1 --generate 7 2>&1
```

注册 cron：
```bash
hermes cron create \
  --name "AI家庭教师-周出题" \
  --schedule "0 20 * * 0" \
  --script "ai-tutor-weekly-pipeline.sh" \
  --no-agent \
  --deliver "feishu:oc_e9f5d48f82f74064f2526e271759235d"
```

### 实际使用的表名（设计文档 vs 实现）

| 设计文档名 | 实际表名 | 差异 |
|-----------|----------|------|
| `vocab_chars` | `chinese_chars` | 无 `lesson_number` 字段；`recognition_type` 替代 `char_type` |
| `vocab_words` | `chinese_words` | `lesson` 字段默认值=0 |
| `textbook_chars` | ✅ **已填充** (2026-05-17: 146 chars) | L1 内容源层生字表 |
| `textbook_words` | ✅ **已填充** (2026-05-17: 79 words) | L1 内容源层词语表 |
| `textbook_lesson_texts` | ✅ **已填充** (2026-05-17: 3篇课文正文) | L1 内容源层课文正文 |
| `page_lesson_map` | ✅ **已填充** (2026-05-17: 13条) | 页码→课时映射 |

**管线策略**：新 `textbook_*` 表为 L1 内容源层（标准化架构），旧 `chinese_*` 表保留为兼容层。当前出题管线从 `chinese_chars`/`chinese_words` 读取，未来可逐步迁移到 `textbook_*`。

⚠️ **textbook_words pinyin 质量陷阱（2026-05-17）**：`textbook_words` 表第18课（童年的水墨画）的 pinyin 字段被截断为最后一个字拼音（如"墨水"→"mò"而非"mò shuǐ"，"梳子"→"shū"而非"shū zi"），共 10 个词受影响。**原因**：OCR 导入阶段将`认识词语`表格的首字当整词标记。**缓解**：使用 `pypinyin` 从 word 字段重建全拼音：`' '.join(p[0] for p in pinyin(word, style=Style.TONE))`。

⚠️ **page_lesson_map UNIQUE 约束陷阱**：该表 UNIQUE(student_id, semester, lesson_number) 阻止同一 lesson_number 的多条记录。非课文页面（习作/语文园地/口语交际）需用**唯一负数 lesson_number**（如 -1, -2, -3...）作为替代键。详见 `references/textbook-import-workflow.md`。

## 十、测试验证模式

### 端到端测试

```python
import requests
BASE = 'http://localhost:5003'
s = requests.Session()

# 1. 登录
s.post(BASE + '/', data={'student_id': 1, 'pin': '0000', 'action': 'student'})
# 2. 首页->今日练习可见
r = s.get(BASE + '/student/home')
assert '今日练习' in r.text
# 3. 开始练习->跳到未做题
r = s.get(BASE + '/student/practice/schedule/1', allow_redirects=True)
# 4. 答题
s.post(BASE + '/student/practice/schedule/1/answer',
       data={'question_id': 1, 'answer': '7', 'index': 0})
# 5. 结果页
r = s.get(BASE + '/student/practice/schedule/1/result')
assert '正确率' in r.text
```

## 十一、出题策略（运营模式）

### 内容来源三源配比

| 来源 | 占比 | 说明 |
|------|------|------|
| 当期60% | ~9题 | 当前学习窗口（C-1, C, C+1）的字词 |
| 往期+跨期40% | ~6题 | 动态分配：往期缺多少跨期补多少 |

### 核心洞察

> 一年级的"春"只是认读，到三年级应该会写。年级跨度自然完成了"可读→可写"的转换。往期池的充实度 = 所有过往学期出现过的字词总数，不区分会认/会写标记。

### 每日题型结构

| 层级 | 题量 | 说明 |
|------|------|------|
| 核心固定（写字+认字） | 12题 | 看拼音写词语6-7题 + 会认字选读音5-6题 |
| 综合轮换 | 3题 | 每天一种：句/课/组/诗/阅 轮换 |

### 时间窗口

**固定窗口法（方案A）：** 家长设置当前课号C，窗口=[C-1, C, C+1]。窗口内的课→60%当期；窗口前→20%往期；跨期→独立于窗口外。

### 批次规格

- 启动：3天45题
- 常态：每周7天105题（1次LLM调用）
- 单元收尾：单元测试（全生词）

详见 `references/question-generation-strategy.md`（完整设计文档v2，含难度自适应+考点避重+7模块上下文包+UI/UE影响评估）。

## 十二、单元测试生成方案

### 10.1 两条路径

| 路径 | 工具 | 适用场景 | 成本 |
|:-----|:-----|:---------|:-----|
| **LLM生成** | `scripts/agent_pipeline.py` | 日常周出题（每日15题×7天） | 调用阿里云百炼（Token费用） |
| **手动构造** | 自定义 Python 脚本 | 单元测试（覆盖全生词）、快速补充 | 零 Token 成本 |

**关键区分：** `agent_pipeline.py` 只支持 **每日练习**（基于 student_settings.current_lesson 的[C-1, C, C+1]三课窗口出15题），不适用于**单元测试**（需覆盖整个单元的全体生字词）。

### 10.1.5 生成前预检：确认练习是否已存在

**关键工作流步骤（2026-05-17 确立）：** 用户要求"生成X单元练习"时，**必须**先做预检，确认目标练习是否已在 practice_library 中存在，而不是直接生成新的。

```sql
-- 检查指定单元的练习是否已存在
SELECT pl.* FROM practice_library pl
JOIN practice_library_questions plq ON pl.id = plq.library_id
JOIN question_bank qb ON plq.question_id = qb.id
WHERE pl.practice_type = 'unit_test'
  AND (pl.title LIKE '%第五单元%' OR pl.title LIKE '%第六单元%')  -- 替换为目标单元名
GROUP BY pl.id;
```

**处理策略：**
| 预检结果 | 操作 |
|:---------|:-----|
| 练习已存在且覆盖目标单元 | **不生成新练习** → 进入验证模式（10.6节）→ 报告用户已有结果 |
| 练习不存在或覆盖不全 | 按 10.2 节手动构造或 10.1 LLM生成 |
| 练习已存在但质量有问题 | 修复后通知用户已有练习已修正 |

**原理：** practice_library 是预制化内容包（v5.0 核心设计），应避免为同一单元生成重复练习。同一练习库可被多天 schedule 引用，重新生成会生成重复的 question_bank 条目。

### 10.6 已有练习的验证模式

当发现指定单元的练习已存在时，执行以下 4 项质量验证：

**验证1：拼音完整性（pinyin_to_word）**
```python
# 规则：看拼音写词语的拼音分词数 ≥ 答案字数
from pypinyin import pinyin, Style

for r in db.execute('''SELECT qb.id, qb.question_text, qb.answer
    FROM practice_library_questions plq
    JOIN question_bank qb ON plq.question_id = qb.id
    WHERE plq.library_id = ? AND qb.question_type = 'pinyin_to_word' ''', (lib_id,)):
    pinyin_part = r['question_text'].split('：', 1)[1].strip()
    expected = ' '.join(p[0] for p in pinyin(r['answer'], style=Style.TONE))
    if len(pinyin_part.split()) < len(r['answer'] or ''):
        # 不完整 → 修复
        db.execute("UPDATE question_bank SET question_text=? WHERE id=?", 
                   (f'看拼音写词语：{expected}', r['id']))
```

**验证2：选择题选项有效性（choose_pinyin）**
```python
import json
for r in db.execute('''SELECT qb.id, qb.choices, qb.answer
    FROM practice_library_questions plq
    JOIN question_bank qb ON plq.question_id = qb.id
    WHERE plq.library_id = ? AND qb.question_type = 'choose_pinyin' ''', (lib_id,)):
    choices = json.loads(r['choices'])
    assert r['answer'] in choices, f"答案不在选项中: {r['answer']}"
    assert len(choices) == len(set(choices)), f"选项重复: {choices}"
```

**验证3：fill_blank 答案来源**
```python
# 检查填空答案是否在已导入的课文正文中存在
for r in db.execute('''SELECT qb.id, qb.answer
    FROM practice_library_questions plq
    JOIN question_bank qb ON plq.question_id = qb.id
    WHERE plq.library_id = ? AND qb.question_type = 'fill_blank' ''', (lib_id,)):
    answer = r['answer']
    found = db.execute('''SELECT 1 FROM textbook_lesson_texts
        WHERE source_text LIKE ? LIMIT 1''', (f'%{answer}%',)).fetchone()
    if not found:
        print(f"⚠️  {answer} 不在已导入课文正文中（可能是外部来源，不影响使用）")
```

**验证4：日程完整性**
```python
# 检查是否已有 schedule 安排此练习
schedule = db.execute('''
    SELECT * FROM practice_schedule 
    WHERE library_id = ? ORDER BY schedule_date
''', (lib_id,)).fetchall()
if not schedule:
    print("⚠️  练习已存在但未排期 → 需创建 practice_schedule 条目")
```

**报告模板（向用户输出）：**
```
📋 第X单元·单元名（第N-M课）
  ├─ 练习库 ID=N（XX题）
  ├─ 题型分布：pinyin_to_word XX + choose_pinyin XX + fill_blank XX
  ├─ 拼音完整性：✅ 全部通过
  ├─ 选择题选项：✅ 全部有效
  └─ 日程：✅ 2026-05-17 已排期
```

### 10.2 手动构造模式（零LLM调用）

适用于快速生成单元测试、补充练习或验证阶段。

#### 10.2.5 每课一练（lesson_test） — 手动构造模式

**适用场景：** 词语表已导入（chinese_words / textbook_words），个别课程缺失练习题时。每次为**一课**生成独立练习包，而非整合为单元测验。

**⚠️ 工作流陷阱（用户纠正 2026-05-17）：** 用户明确指示"不是整合成一套，而是生成每课一练的课程练习"。生成缺口补全时，**默认每课独立建库，不要默认整合为单元测验**。汇报缺口时建议方案必须使用"每课一练"措辞，不可先提整合方案等纠正。

**构造规格（已验证）：**

| 题型 | 题量 | 来源 | 难度 |
|:-----|:----:|:-----|:----:|
| pinyin_to_word | 10题 | textbook_words + chinese_words | 2 |
| choose_pinyin | 6题 | textbook_lessons.write_chars | 2 |
| fill_blank | 2题 | textbook_lesson_texts | 3 |

**命名规范：**
- practice_library.title = `第{课号}课·{课文名}`
- practice_library.practice_type = `lesson_test`
- practice_library.coverage = `{"lesson": {课号}, "lesson_name": "第{课号}课·{课文名}"}`
- practice_schedule.schedule_date = 逐课连续排期（每天一课）

**构造脚本模板：**

```python
lessons_data = {
    22: {
        'title': '我们奇妙的世界',
        'words': [('珍珠', 'zhēn zhū'), ('呈现', 'chéng xiàn'), ...],
        'write_chars': ['珍','呈','幻','诱','润','辉','芒','剑','普','模','型', ...],
        'choose_chars': [
            ('呈', 'chéng', ['chéng','céng','chén','chěng']),
            # (char, correct_answer, [4 unique options])
        ],
        'text_fills': [
            ('第22课·我们奇妙的世界：这是一个________的世界。', '奇妙'),
            # (question_text_with_blanks, answer)
        ],
    },
}
# 每课一个 practice_library + 一个 practice_schedule
# practice_type = 'lesson_test'
# schedule_date = today + offset (0, 1, 2 days)
```

**验证要点：**
1. 每道 pinyin_to_word 的拼音分词数 ≥ 答案字数（用 pypinyin 重建验证）
2. choose_pinyin 的 4 选项必须唯一且正确答案在选项内
3. fill_blank 答案应在 textbook_lesson_texts 中存在原文匹配
4. 每课 schedule 日期不同（连续逐天）
5. **去重检查**：⚠️ `SELECT id FROM question_bank WHERE lesson_number=?`**会返回同一个词语的多个 qid**（多轮生成脚本对同一词语插入了多条记录）。必须用 `SELECT MIN(id) ... GROUP BY answer` 确保 uniqueness。
6. **新题型去重**：write_char 用 `GROUP BY answer`（同一字不同ID），sentence_form 用 `GROUP BY answer`（同一词不同ID）

### 6.5 新题型：写汉字 (write_char) 与 组词造句 (sentence_form)（2026-05-18 新增）

#### write_char — 写汉字

**数据来源：** `textbook_chars` WHERE `char_type='write'`（103个会写字，已映射 lesson_number）

**题型设计：**
- `question_text` 格式：`写汉字：pinyin`
- `question_type`：`write_char`
- `answer`：单字（如"秘"）
- `answer_char_count`：1（单格子）
- `pinyin_syllables`：由 `library_service.py` 从 `:` 后解析为单元素数组 `[pinyin]`

**模板渲染：**
```
写汉字 ← q-badge
zhēn     ← hw-pinyin-label（拼音标签居中）
┌─────────────┐
│   田字格     │  ← 单格手写 Canvas（260×260）
│   手写区     │
└─────────────┘
    1         ← hw-cell-label（序号）
```

**答案检查策略：** 精确匹配（`user_answer == correct_answer`）

**数据生成脚本：**
```python
chars = db.execute("""
    SELECT tc.char, tc.pinyin, tl.lesson_number
    FROM textbook_chars tc
    JOIN textbook_lessons tl ON tc.lesson_id = tl.id
    WHERE tc.char_type = 'write'
""").fetchall()
for c in chars:
    db.execute(
        "INSERT INTO question_bank (student_id, subject, question_type, question_text, answer, "
        "lesson_number, difficulty_assigned, status, knowledge_point) "
        "VALUES (1, 'chinese', 'write_char', ?, ?, ?, 2, 'active', '会写字·第X课')",
        (f"写汉字：{c['pinyin']}", c['char'], c['lesson_number'])
    )
```

#### sentence_form — 组词造句

**数据来源：** `chinese_words` WHERE `word` IS NOT NULL（119个词语，已映射 lesson）

**题型设计：**
- `question_text` 格式：`用「word」造句`
- `question_type`：`sentence_form`
- `answer`：词语本身（用于关键词包含检查）
- 无手写格子，使用 `<textarea>` 输入

**模板渲染：**
```
组词造句           ← q-badge
第X课·课文名       ← q-source
用「凉爽」写一个句子 ← 题干（keyword 从 question_text 正则提取）
┌─────────────────────────────────────┐
│  ［textarea 输入框］                   │
│  min-height:100px                     │
│  placeholder: "在这里写句子……"        │
│  oninput: updateSentenceSubmit()      │
└─────────────────────────────────────┘
至少写2个字      ← sentenceHint（字数提示）
[🤷 跳过]  [提交答案]
```

**答案检查策略：** 宽松包含匹配（`keyword in user_answer`），即学生答案中必须包含目标词语。

**JavaScript 提交：** `sentence_form` 在表单提交时走独立分支，不经过 `hwCells` 逻辑：
```javascript
// 在 form submit handler 中
} else if (document.getElementById('sentenceInput')) {
    answer = document.getElementById('answerInput').value;
    if (!answer || answer === '_unknown_') return;
}
```

**后端路由处理：**
```python
if q['question_type'] == 'sentence_form':
    keyword = correct_answer
    is_correct = keyword in user_answer
else:
    is_correct = user_answer == correct_answer
```

#### Template 题型扩展模式（四步法）

每次新增题型到 `schedule_practice.html` 时，必须按以下 4 步操作：

| 步骤 | 位置 | 修改内容 | 示例（write_char） |
|:----:|:-----|:---------|:------------------|
| 1️⃣ 添加 Badge 标签 | 模板 q-badge 区 | 增加 `elif question.question_type == 'xxx'` | `写汉字` |
| 2️⃣ 添加渲染分支 | 模板 if/elif 区 | 增加新的 HTML 渲染块 | 使用 hw-cells（与 pinyin_to_word 共用） |
| 3️⃣ 添加提交分支 | JS 表单 submit handler | 增加新类型的 answer 提取逻辑 | 共用 hwCells 提取（已存在，不新增） |
| 4️⃣ 添加答案检查 | 后端 route | 如需要特殊匹配逻辑 | 精确匹配（与默认相同，不新增） |

**常见陷阱：**
- **漏掉 `{% endif %}`**：添加 elif 分支后必须确保有一个闭合的 `{% endif %}` 在 `</form>` 之前
- **JavaScript 提交没有保护**：新题型必须有对应的 answer 提取逻辑，否则会误进入默认的 `hwCells` 路径（hwCells 不存在时抛异常）
- **`library_service.py` 的 pinyin_syllables 解析**：非 pinyin_to_word 题型如果 question_text 也含冒号，会误触发拼音解析。必须为每个新题型显式定义解析逻辑
- **answer_char_count 缺失**：write_char 需要 `answer_char_count=1`，否则默认值可能导致格子数错误

#### 完整题型清单（截至 2026-05-18）

| 题型 | question_type | 来源表 | 题量 | 答案检查 | 输入方式 |
|:-----|:--------------|:-------|:----:|:---------|:---------|
| 看拼音写词语 | `pinyin_to_word` | chinese_words + textbook_words | 170 | 精确匹配 | 多格手写 |
| 会认字选读音 | `choose_pinyin` | textbook_chars (char_type=recog) | 71 | 选项选择 | 点击选项 |
| 课文填空 | `fill_blank` | textbook_lesson_texts | 22 | 精确匹配 | 多格手写 |
| 写汉字 | `write_char` | textbook_chars (char_type=write) | 103 | 精确匹配 | 单格手写 |
| 组词造句 | `sentence_form` | chinese_words | 119 | 关键字包含 | textarea |

**lesson_test 库题型覆盖率：** 截至 2026-05-18，9个 lesson_test 库全部覆盖以上5种题型（第21课仅有 write_char + sentence_form，需补充 pinyin_to_word + choose_pinyin + fill_blank）。

**触发场景：** 用户说"根据词语表生成练习"、"检查是否有词语表记录，生成没课的练习题"时，必须先审计覆盖缺口，再汇报，获确认后执行。

**工作流原则：**
```
Step 0-A: 审计覆盖缺口 → 呈报发现（HTML暗色科技风信息图）→ 获确认后执行
Step 0-B: 若用户要求直接生成（无审计指令），跳过审计，从 10.2.1 起直接构造
```

**核心审计 SQL（对比 chinese_words.lesson 与 question_bank.lesson_number）：**
```sql
SELECT cw.lesson, COUNT(DISTINCT cw.id) as word_count, 
       COALESCE(qb.q_count, 0) as question_count,
       CASE WHEN COALESCE(qb.q_count, 0) = 0 THEN '❌ 缺题' ELSE '✅' END as has_questions
FROM chinese_words cw
LEFT JOIN (
    SELECT lesson_number, COUNT(*) as q_count 
    FROM question_bank GROUP BY lesson_number
) qb ON cw.lesson = qb.lesson_number
GROUP BY cw.lesson ORDER BY cw.lesson;
```

**汇报格式（HTML 信息图）：** 需包含：
1. 词语表总记录数 + 覆盖课号范围
2. 按课对比表（词数 vs 已有题数 vs 状态）
3. 缺失课的课文名、词数、建议题量
4. 清晰的确认执行提示

**数据源选择逻辑：**
| 数据源 | lesson 字段 | 说明 |
|--------|-------------|------
| `chinese_words` | `lesson`（直接可用，16-24） | 拍照导入，词量较大，部分无拼音 |
| `textbook_words` | 通过 `lesson_id` → JOIN `textbook_lessons.lesson_number` | 教材标准词，拼音较全但词量少 |

优先用 **`chinese_words`** 判断覆盖缺口（词量最大），生成时用 `textbook_words` + `pypinyin` 补全拼音。

**常见陷阱：**

**陷阱A：question_bank 中 lesson_number=NULL 的旧题不计入覆盖**
旧系统遗留题（lesson_number=NULL）即使出现在 practice_library 中也不计入"词语覆盖率"。必须查 `question_bank.lesson_number` 不为 NULL 且与 `chinese_words.lesson` 匹配的才算。

**陷阱B：Library #4（第七单元测验）全部 62 题 lesson_number=NULL**
虽覆盖22-24课内容，但用的是旧遗留题，需基于词语表新建练习库。

**陷阱C：已有 Library 但使用旧题时，仍需基于词语表新建练习库**

**陷阱D：SELECT DISTINCT answer vs SELECT id 数据膨胀**
```sql
-- ❌ 会返回重复答案的不同 qid
SELECT id FROM question_bank WHERE lesson_number=18 AND question_type='pinyin_to_word'
-- 可能返回: [75(葫芦), 91(水墨画), 76(染绿), 92(染绿), ...]
-- ↑ 染绿有2个不同id的qid，但 answer 相同

-- ✅ 需去重后取唯一答案对应的 qid
SELECT MIN(id) FROM question_bank
WHERE lesson_number=18 AND question_type='pinyin_to_word'
GROUP BY answer
```

**根本原因：** 同一课的多轮生成（手动构造+agent_pipeline+自定义脚本）可能对同一词语创建多条 question_bank 记录（content_hash 不同但 answer 相同）。使用 `SELECT id` 而非 `SELECT DISTINCT answer` + 聚合（取 MIN id）会导致 practice_library_questions 出现重复。

**修复方法（代码模式）：**
```python
# 获取每课唯一词语的题目ID
rows = db.execute("""
    SELECT MIN(id) as qid FROM question_bank
    WHERE lesson_number=? AND question_type='pinyin_to_word'
      AND subject='chinese' AND status='active'
    GROUP BY answer
""", (ln,)).fetchall()
qids = [r['qid'] for r in rows]
# 再用 qids 构建 practice_library_questions
```

**已存在的重复修复方案：** 若 practice_library_questions 已存在重复，用以下模式清理：
```python
for ans in duplicates:
    dup_qids = db.execute("""
        SELECT plq.id FROM practice_library_questions plq
        JOIN question_bank qb ON plq.question_id = qb.id
        WHERE plq.library_id = ? AND qb.answer = ?
        ORDER BY plq.id
    """, (lib_id, ans)).fetchall()
    for d in dup_qids[1:]:
        db.execute("DELETE FROM practice_library_questions WHERE id=?", (d['id'],))
db.execute("UPDATE practice_library SET question_count=? WHERE id=?",
           (new_count, lib_id))
db.commit()
```

**陷阱E：判断覆盖缺口要查 `chinese_words.lesson` 而非 `question_bank.lesson_number` 为空**
```sql
-- ❌ 只看 question_bank 不够，因为某些课可能有旧题但不标记课号
-- ✅ 正向核查：从 chinese_words 反查
SELECT cw.lesson, COUNT(DISTINCT cw.id) as word_count, 
       COALESCE(qb.q_count, 0) as question_count,
       CASE WHEN COALESCE(qb.q_count, 0) = 0 THEN '❌ 缺题' ELSE '✅' END as has_questions
FROM chinese_words cw
LEFT JOIN (
    SELECT lesson_number, COUNT(*) as q_count 
    FROM question_bank GROUP BY lesson_number
) qb ON cw.lesson = qb.lesson_number
GROUP BY cw.lesson ORDER BY cw.lesson;
```

| 单元 | 题量 | 题型分布 | 数据来源 |
|:-----|:----:|:---------|:---------|
| 第六单元（18-21课，4课） | 58题 | pinyin_to_word 47 + choose_pinyin 6 + fill_blank 5 | practice_library #3 |
| 第七单元（22-24课，3课） | 62题 | pinyin_to_word 54 + choose_pinyin 6 + fill_blank 2 | practice_library #4 |
| 第五单元·习作（16-17课，2课） | 34题 | pinyin_to_word 21 + choose_pinyin 10 + fill_blank 3 | practice_library #5 |

**构造步骤：**

```
Step 1: 获取该单元全部词语（从 textbook_words 或 chinese_words 按 lesson 过滤）
Step 2: 每词生成 1 道 pinyin_to_word（看拼音写词语）
Step 3: 选择单元中 6-10 个关键生字，每字 1 道 choose_pinyin（4选项，1正确）
Step 4: 添加 2-5 道 fill_blank（课内重点句子或诗词）
Step 5: 写入 question_bank（INSERT 循环）
Step 6: 创建 practice_library + 建立 practice_library_questions 映射
Step 7: 创建 practice_schedule 或更新现有 schedule
```

**choose_pinyin 选项验证规则：** 4个选项必须**唯一**且正确答案必须在其中。常见错误：忘了去重导致选项重复（如 ['dong', 'dong', 'dun', 'dong']）。

**fill_blank 题干设计：** 从课文重要句式或描写性语句中提取，空格使用 `________`（6个下划线），标点符号保留。

### 10.3 单元测试 vs 每日练习 vs 每课一练 的语义区别

| 维度 | 每日练习 | 单元测试 | **每课一练** |
|:-----|:---------|:---------|:-----------|
| 覆盖范围 | 3课窗口（C-1, C, C+1） | **整单元全部课** | **单课独立** |
| 题量 | 15题固定 | 全生词（30-62题） | ~18题（10拼音+6选音+2填空） |
| 题型 | 核心12+综合3 | 全部 pinyin_to_word + 少量综合 | pinyin_to_word + choose_pinyin + fill_blank |
| practice_type | `daily` | `unit_test` | **`lesson_test`** |
| 排期策略 | 每周 cron 批量7天 | 单元学完后手动 | **逐课连续排期（每天一课）** |
| 出题引擎 | `agent_pipeline.py`（LLM） | 手动构造或包装 LLM 单次调用 | **手动构造（零LLM）** |
| 触发时机 | 每周 cron | 整单元学完后手动触发 | **词语表导入后，按缺课查漏补缺** |

### 10.4 已知陷阱

**习作单元的特殊性：** 第五单元（大胆想象）是**习作单元**，只有2课（第16-17课），且：
- 词语量少（仅21个词，非习作单元通常40+词）
- 以阅读和写作为主，会认字/会写字要求少
- 单元测试题约25-35题（常规单元50-65题）

**单元不存在时的处理策略：** 当请求生成的单元在 `textbook_units` 中不存在时：
1. 先检查 `chinese_words` 中是否有该课号的词语数据 → 确认内容源存在
2. 补充 `textbook_units` + `textbook_lessons` 记录
3. 补充 `textbook_chars` + `textbook_words` 内容（可从 `chinese_chars`/`chinese_words` 迁移）
4. 再生成题目

### 10.5 参考文件

详见 `references/unit-test-generation.md`（本会话的完整操作实录与代码模板）。

## 十三、家长中心管理层（UI 架构）

### 整体结构

家长中心是数据架构（L1-L4）的**管理界面层**，负责操作和查看各层数据。当前设计为 **先选科目 → 进入科目管理** 的模式：

```
家长中心首页
├── 🏠 仪表盘（跨学科概览 — 进度、积分、今日待办）
├── 👦 学生管理（切换/添加学生）
├── 🎯 选择科目 ──→ 进入科目管理页
│   ├── 📚 语文
│   ├── 🔢 数学（规划中）
│   └── 🔤 英语（规划中）
├── ⭐ 积分管理（公共 — 跨学科积分池、兑换、审批）
└── ⚙️ 系统设置（公共 — 家长PIN、通知偏好等）
```

### 科目内管理模块（以语文为例）

每个科目下 7 个功能模块，直接映射到数据架构：

| 模块 | 定义 | 数据层对应 | 说明 |
|------|------|-----------|------|
| **📥 内容导入** | 课本拍照OCR、试卷上传、培训资料导入 | L1 内容源 | OCR识别 → 结构化入库 |
| **📖 词库管理** | 查看/编辑生字词、词语、拼音组词 | L2 题库(chars+words) | 旧"词汇管理"并入此 |
| **📝 题库管理** | 查看/编辑/校准 question_bank 题目 | L2 题库(question_bank) | 含手动添加、批量导入、校准 |
| **📦 练习库** | 预制练习包列表 → 查看详情 → 手动触发打包 | L3 练习库(practice_library) | **当前缺失**，需新增 |
| **📅 学习计划** | 练习投放排期、补做标记、进度看板 | L4 计划+执行(practice_schedule) | 替代旧"学习计划" |
| **📊 学习报告** | 正确率趋势、薄弱知识点、Agent分析摘要 | L4 执行(practice_records) | 替代旧"学习报告" |
| **🔧 出题设置** | 学科级参数：三源比例、窗口法课号、难度自适应、题型开关 | 学科配置(student_settings) | 替代旧"出题设置" |

### 当前冗余与清理

| 旧功能 | 问题 | 处理 |
|--------|------|------|
| 词汇管理（独立页面） | 与题库功能重叠，V3/V5混用 | **删除独立页面** → 并入词库管理 |
| 题库管理（question_bank） | 有题库但缺练习库管理入口 | **保留+扩展**，新增练习库模块 |
| 学习计划（plans） | 实为历史记录+重做，非计划管理 | **重命名**为学习报告，新建计划页 |
| 出题设置（settings） | Agent管线不消费此配置 | **重构**：拆为公共设置+学科设置 |
| 仪表盘（dashboard） | 跨学科混排 | **保留概要**，详细入口移至科目内 |

### 路由架构建议

```python
# 公共
/parent/                           → 选人/选科目入口
/parent/dashboard                  → 跨学科仪表盘
/parent/coins                      → 积分管理

# 科目级（语文示例）
/parent/chinese/                   → 语文科目管理首页
/parent/chinese/import             → 内容导入
/parent/chinese/vocab              → 词库管理
/parent/chinese/question-bank      → 题库管理
/parent/chinese/practice-library   → 练习库
/parent/chinese/schedule           → 学习计划
/parent/chinese/reports            → 学习报告
/parent/chinese/settings           → 出题设置

# 数学/英语后续按此模式扩展
/parent/math/...
/parent/english/...
```

### UI 布局规范

当所有科目都采用此模式时，模板设计中应注意：

1. **侧栏导航**（科目级）：左侧固定导航，展示7个模块 + 返回顶部入口
2. **面包屑**：家长中心 > 📚语文 > 题库管理
3. **科目色系**：语文章地绿、数学科技蓝、英语活力橙（继承 Edu-Hub 配色约定）
4. **返回路径**：每个科目页顶部应有"返回家长中心"链接

### 会话结果存储

每次家长中心优化/改版的设计文档、审计报告等产出物，使用 `feishu-doc-versioning` 规则，上传到飞书云盘 **项目→AI家庭教师** 目录（token: `H04qfCSORlT7FFdNQBdcDANwn8d`）。

详见 `references/parent-center-optimization-v1.html`（本会话产出的完整优化方案文档）。

## 十四、参考文件

| 文件 | 说明 |
|:-----|:------|
| `references/data-structure-design-v4.html` | 旧版 v4.1 完整 HTML 设计文档 |
| `references/llm-context-7-module-pattern.md` | LLM上下文包组装模式详解（跨项目复用） |
| `references/question-generation-strategy.md` | **出题策略方案v2：完整设计（比例/题型/窗口/动态分配/难度自适应/考点避重/管线）** |
| `references/unit-test-generation.md` | **单元测试手动生成操作实录+代码模板+陷阱清单** |
| `references/agent-pipeline-implementation.md` | **出题管线实现参考：app_context()、实际表名、7模块组装、cron集成、已知问题** |
| `references/phase5-implementation-summary.md` | **Phase 1-5 实施总结 + 飞书文件上传/共享工作流 + 已知陷阱** |
| `references/feishu-file-upload-workflow.md` | **飞书文件上传→共享→发送链接完整工作流（含文件共享到群聊的特殊处理）**
| 项目内: `~/edu-hub/ai-family-tutor/docs/data-structure-design-v5.html` | 最新 v5.0 设计文档 |
| 项目内: `~/edu-hub/ai-family-tutor/services/library_service.py` | 核心只读库服务 |
| 项目内: `~/edu-hub/ai-family-tutor/scripts/001_add_v5_tables.py` | 新增13张表迁移脚本 |
| 项目内: `~/edu-hub/ai-family-tutor/scripts/seed_test_data.py` | 测试数据种子脚本 |
| `references/old-system-residual-references.md` | **旧系统残留引用追踪（2026-05-18 新增）：50+处仍在查询 `practice_sessions`/`practice_answers` 的代码位置清单** |
| `references/per-lesson-generation-logs-20260517.md` | **每课一练生成操作日志：覆盖审计SQL + 已知陷阱（textbook_words拼音截断、duplicate去重、chinese_words重复记录）** |
| `references/learning-report-kg-sql-20260518.md` | **知识图谱+题型分布SQL模式（2026-05-18新增）—— textbook_chars JOIN textbook_lessons 陷阱 + 三态判定** |
| `references/learning-report-sql-pattern.md` | **学习报告SQL查询模式：每课掌握情况 + 高频错误字词统计（2026-05-17新增）** |**
