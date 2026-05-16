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
| practice_type | TEXT | daily/unit_test/challenge/big_practice |
| title | TEXT | 标题，如"每日练习·5月18日" |
| subject | TEXT | 学科，默认chinese |
| question_count | INTEGER | 总题数，默认15 |
| coverage | TEXT | 覆盖范围，如"第1-4课" |
| status | TEXT | active/archived |
| created_at | TEXT | |

**设计要点：** 这是一个预生成的、不可变的内容包。不是实时凑题。同一练习库可被多天的 schedule 引用。

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
| `get_schedule_detail(schedule_id)` | 练习内容详情 | schedule + library + questions 三表JOIN |
| `submit_schedule_answer(schedule_id, question_id, user_answer, is_correct, student_id)` | 记录答题+更新schedule状态 | INSERT practice_records + UPDATE schedule |
| `get_schedule_result(schedule_id, student_id)` | 结果报告 | practice_records 聚合查询 |

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

## 六、LLM驱动出题（上下文包模式）

v4.1 的7模块上下文包设计保持不变。主要变化：

- **新上下文源**：practice_records 中的 attempt_number 支持多次尝试统计
- **防重范围**：用 practice_library 中最近N天的题去重
- **单元测试覆盖规则**：单元测试 type=unit_test 必须覆盖该单元全部生词

## 七、standalone SQLite 脚本注意事项

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
| `textbook_chars` | 空表 | 未填充数据 |
| `textbook_words` | 空表 | 未填充数据 |

**管线直接从 `chinese_chars`/`chinese_words` 读取**。

## 八、测试验证模式

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

## 九、出题策略（运营模式）

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

## 十一、家长中心管理层（UI 架构）

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

## 十二、参考文件

| 文件 | 说明 |
|:-----|:------|
| `references/data-structure-design-v4.html` | 旧版 v4.1 完整 HTML 设计文档 |
| `references/llm-context-7-module-pattern.md` | LLM上下文包组装模式详解（跨项目复用） |
| `references/question-generation-strategy.md` | **出题策略方案v2：完整设计（比例/题型/窗口/动态分配/难度自适应/考点避重/管线）** |
| `references/agent-pipeline-implementation.md` | **出题管线实现参考：app_context()、实际表名、7模块组装、cron集成、已知问题** |
| `references/phase5-implementation-summary.md` | **Phase 1-5 实施总结 + 飞书文件上传/共享工作流 + 已知陷阱** |
| `references/feishu-file-upload-workflow.md` | **飞书文件上传→共享→发送链接完整工作流（含文件共享到群聊的特殊处理）**
| 项目内: `~/edu-hub/ai-family-tutor/docs/data-structure-design-v5.html` | 最新 v5.0 设计文档 |
| 项目内: `~/edu-hub/ai-family-tutor/services/library_service.py` | 核心只读库服务 |
| 项目内: `~/edu-hub/ai-family-tutor/scripts/001_add_v5_tables.py` | 新增13张表迁移脚本 |
| 项目内: `~/edu-hub/ai-family-tutor/scripts/seed_test_data.py` | 测试数据种子脚本 |
| 项目内: `~/edu-hub/ai-family-tutor/scripts/test_api_v2.py` | 端到端测试脚本 |
