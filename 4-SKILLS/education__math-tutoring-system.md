---
name: math-tutoring-system
description: >-
  数学模块（C003）— Edu-Hub 下的数学学科模块。基于 session 模式的在线答题系统（非文档驱动），含每日练习(10题新分布)、大挑战(20题/30分钟)、挑战题(7类)、综合题(5类)、积分共享、成就体系。
  v7.0（2026-05-12）：从文档驱动全面转型为 session 模式 Web 应用。试卷分析→错题本→每日练习→大挑战。与英语 C008 架构对齐。
version: 6.1.0
author: Hermes Agent
metadata:
  hermes:
    tags:
      - education
      - math
      - tutoring
      - daily-practice
      - wrong-question
      - exam-analysis
    category: education
---

# 三年级数学辅导系统操作指南 v6.0

> ⚠️ **v6.0 核心变更（全面升级）**：
> - **OPT-1 wrong_based填充**：每日练习基于错题本生成，wrong_based字段不再为空
> - **OPT-3 ABC知识点轮换**：A组(每天练)+B组(隔天练)+C组(每周练)，智能分配
> - **OPT-4 知识图谱增强**：全部30个知识点增加prerequisites/related_topics/question_types
> - **OPT-5 学习日报**：每日7:30自动生成学习日报(cron)，含错题回顾/趋势/薄弱TOP3/复习提醒
> - **OPT-6 错题本结构升级**：增加知识点ID、错误类型字段
> - **OPT-7 艾宾浩斯复习**：记录第1/3/7/14/30天复习日期，到期自动提醒
> - **OPT-8 掌握度动态加权**：距今天越近的错题权重越高，连续3次正确自动恢复
> - **OPT-9 多孩子支持**：状态文件升级为children结构，load_state_for_child() API
> - **OPT-11 知识点自动推断**：infer_topic_from_question() 基于关键词+上下文推断
> - **🔬 自检机制**：每个操作执行后自动验证结果（self_check.py）

## ✅ v7.0 在线化转型（2026-05-12 实施完成）

> 从文档驱动全面转向在线交互。**先设计→用户确认→再实施** 是本模块的标准改造流程。
> 实施前必须做 **架构对齐检查**（对标英语模块 C008 v3/v4 的 Blueprint、session 模式、模块分离度），输出对比表供用户决策。
> 详见 `references/v7-question-strategy-design.md` 和 `references/architecture-alignment-english-v3.md`。

### 实施范围 & 状态

| # | 改造项 | 状态 | 说明 |
|:-:|:-------|:-----|:------|
| 1 | **在线答题（替代文档生成）** | ✅ **已完成** | session模式API（逐题提交），模板激活，cron 7:00取消 |
| 2 | **试卷批改 + 全量知识图谱更新** | 🔲 待实施 | 拍照上传→AI识别→批改→知识图谱更新 |
| 3 | **积分体系（统一账户）** | ✅ **已完成** | import `english/v3/coins_manager.py`，共用币池 |
| 4 | **大挑战模式** | ✅ **已完成** | 20题/30分钟倒计时/challenge_engine.py + session_manager.py |
| 5 | **每日练习增强** | ✅ **已完成** | 10题新分布：变式2+基础2+进阶1+应用1+挑战1+综合2+口算1 |
| 6 | **举一反三** | 🔲 待实施 | 分析上传试卷的出题模式→生成类似新题（需试卷批改后对接） |
| 7 | **架构对齐** | ✅ **已完成** | 见 `references/architecture-alignment-english-v3.md` |

### 学生交互路径（已确认）

```
角色选择 /math/
  ↓ 选择"学生"
学生首页 /math/student/ (NEW)
  ├── 🎯 今日练习 → /math/practice/ (10题，增强版)
  ├── 🏆 大挑战   → /math/challenge/ (NEW, 20题，30分钟倒计时)
  ├── 📊 我的成就 → /math/achievements/ (NEW, 等级+金币+连续打卡)
  └── 📖 错题复习 → /math/wrong-book/ (增强，可重做)
```

### 出题策略三层模型（已确认）

```
第一层 — 知识点选择（优先级排序）
  0. 错题知识点（当天错题所属知识点）
  1. 到期复习（艾宾浩斯间隔到）
  2. 薄弱知识点（掌握度<60%）
  3. ABC轮换（A每天/B隔天/C每周）
  4. 平衡补充（应用+计算均衡）

第二层 — 难度选择（按掌握度）
  <40% → foundation（基础级）
  40-70% → standard（标准级）
  >70% → extension（拓展级）

第三层 — 题型生成
  18个现有生成器 + 7个挑战题生成器 + 5个综合题生成器
```

### 每日练习10题分布（已确认）

```
错题变式 2题 → 基础计算 2题 → 进阶计算 1题 → 标准应用题 1题
→ 🧩 挑战题 1题 → 🌐 综合题 2题 → 口算 1题
```

### 大挑战规格（已确认）

| 维度 | 值 |
|:-----|:---|
| 题量 | 20题（14 standard + 6 extension） |
| 时限 | 30分钟倒计时，到点自动提交 |
| 覆盖 | 最近7天所有知识点 + 全部薄弱点(<60%) |
| 奖励 | 基础金币×2 + 正确率加成 |
| 触发 | 学生首页自主点击，每日最多1次 |

### 架构对齐要求（与英语 C008 v3 对比）

| 维度 | 对齐判定 | 现状→目标 |
|:-----|:---------|:----------|
| 数据流模式 | ❌ 需改为 session 模式 | bundle→ session（start→current→submit→complete） |
| 模块分离 | ❌ routes.py(35KB) 内联所有逻辑 | routes→路由 / practice_engine→出题 / challenge_engine→大挑战 / session_manager→会话 / coins_manager→复用英语 |
| 学生首页 | ❌ 无独立入口 | 新增 student_portal.html（参考英语 portal_v4.html） |
| API 风格 | ⚠️ 保持独立 | 不必完全 RESTful，但大挑战按英语 v4 的 `/api/v4/challenge/start` 模式 |

### 实施前必须用户确认的环节

1. **模板设计** → 先展示再实施（用户明确说"你先设计模板，在开始改造执行前先和我说一下"）
2. **架构对比表** → 先展示对齐方案（本session已完成比对，输出6维度对比表）
3. **执行矩阵** → 按 P0/P1/P2 分级，用户选执行范围

### 架构对齐检查清单（改造前必做）

改造数学模块前，先做以下对比：

```python
# 检查项（对照英语 v3）
checklist = [
    ("Blueprint注册方式", "register_blueprint + url_prefix"),
    ("模板目录结构", "templates/v{version}/ 子目录"),
    ("数据流模式", "session模式 vs bundle模式"),
    ("模块分离度", "routes.py中抽出的engine/*.py"),
    ("积分系统", "复用coins_manager.py"),
    ("API风格", "RESTful /api/v{version}/ prefix"),
]
```

输出对比表给用户确认后再开始编码。

## 技术陷阱 — v7.0 实施教训（2026-05-12）

### template-first 确认模式

用户要求在编码前先展示模板设计。这是本模块改造的固定流程：

```python
# 正确的改造节奏
Step 1: 设计模板/策略文档 → 用户确认
Step 2: 架构对齐检查（对照英语模块）→ 用户确认
Step 3: 编码实现 → 测试 → 交付
```

跳过 Step 1 直接编码会导致返工。

### 实践页面空白 — API 返回格式缺少 questions 字段（2026-05-12）

旧 API `/api/practice/today` 有两个返回路径：① `if bundle:` 从 `load_today_bundle()` 加载；② 兜底从 `generate_daily_practice()` 生成。**第一个路径的 `return jsonify(...)` 中缺少 `questions` 字段。**

```python
# ❌ 第一个路径（if bundle:）返回：
return jsonify({
    "calculations": calculations,
    "word_problems": word_problems,
    "date": ...,
    "title": ...,
    # ← 缺少 questions！
})

# ✅ 修复：两个路径都要加
return jsonify({
    "calculations": calculations,
    "word_problems": word_problems,
    "questions": q_list,
    "date": ...,
    "title": ...,
})
```

前端 JS `if (!data.questions || data.questions.length === 0)` → `showEmptyState()` → 渲染空白。

**排查方法**：页面标题正常、CSS加载、API 返回 200，但显示空白状态 → curl API 检查 `questions` 字段是否存在。

**修复检查清单**：当修改 `api_practice_today()` 等 API 的返回体时，搜索所有 `return jsonify` 实例，确认每个路径都包含前端期待的字段。

### challenge_bundle 已内含 session

`generate_challenge_bundle()` 内部调用 `create_session()`，不须在 API 路由中再次创建。API 路由只需获取 `bundle["session_id"]` 即可。

### session_manager 只能通过 sys.path 导入

`challenge_engine.py` 中 `from session_manager import create_session` 需先添加 `_MATH_V2_DIR` 到 `sys.path`。不能依赖相对导入（Flask Blueprint 上下文不可用）。

### order-of-operations / simplify-calculation 生成器参数不匹配

`gen_order_of_ops()` 期望 `a_range, b_range, c_range` 三个参数，但 YAML 模板传递 `ops, num_range` 等顶层参数。这是 v6.0 已有的问题，不影响挑战题和综合题生成（挑战引擎会跳过无效模板，用其他模板补足到20题）。

### 统一积分规则

| 行为 | 金币 | 说明 |
|:-----|:-----|:------|
| 每日练习答对1题 | +1 | 日上限20 |
| 完成练习 | +2 | 打卡奖励 |
| 全对 | +5 | 额外奖励 |
| 连续打卡3/7/14/30天 | +5/+15/+40/+100 | 里程碑 |
| 大挑战完成 | 基础×2 | 答对+1变+2，完成+2变+4 |
| 大挑战正确率加成 | +3~10 | 按正确率阶梯 |
| 等级提升 | +50/+100/+200 | 青铜→白银→黄金→钻石 |

### 举一反三机制（已确认需求）

> 分析上传试卷的出题模式 → 自动生成类似新题。

实现思路：
1. 家长拍照/扫描上传已完成试卷
2. 系统识别每道题的：题型（计算/应用/选择...）+ 知识点 + 数字范围 + 难度
3. 基于识别的参数组合，调用对应的 generator 模板，换数字生成相似题
4. 生成的相似题存入练习题库，可在后续练习中使用

## 系统组件

| 组件 | 路径 | 用途 |
|:----|:-----|:-----|
| 知识图谱 | `~/edu-hub/knowledge-graph/grade3-semester2-math.yaml` | 30个知识点，9大单元 |
| 试卷分析 | `~/edu-hub/scripts/exam-engine.py` | 试卷→知识点提取→报告 |
| **原始题解析(v6.0)** | **`~/.hermes/scripts/raw-exam-parser.py`**, cron 8&20点 | **纯文本→错题检测(3策略)+✅正确过滤+去重→统一错题本追加+同步state.wrong_questions** |
| Token注册表 | `~/.hermes/feishu-tokens.json` + `~/.hermes/scripts/feishu_tokens.py` | 12个文件夹token集中管理，每日3:00验证漂移 |
| **每日练习(v6.0)** | **`~/.hermes/scripts/daily-practice-cron.py`**, cron 7AM | **15分钟练习(计算+应用+纠错)+ABC知识点轮换+wrong_based填充** |
| 错题分析 | `~/edu-hub/scripts/wrong-analyzer.py` | 错题归因+掌握度加权计算+艾宾浩斯复习日期 |
| 复习宝典完善 | `~/edu-hub/scripts/review-updater.py` | 新知识点+易错点→补充宝典 |
| 飞书写工具 | `~/.hermes/scripts/feishu-md-writer.py` | Markdown→飞书文档 |
| 系统状态 | `~/.hermes/scripts/edu-system-state.json` | 错题本token/掌握度/练习日志 |
| **统一日志(v4.3)** | **`~/.hermes/scripts/edu_logger.py`** + **`~/edu-hub/logs/c003-activity.log`** | **所有操作的统一日志记录（时间+操作+详情）** |
| **修复脚本(v4.4 NEW)** | **`~/.hermes/scripts/fix-daily-practice.py`** | **检测缺失飞书练习文档并自动补创建（--date / --all）** |
| **符号链接(v4.4)** | `~/edu-hub/scripts/edu_logger.py` → `~/.hermes/scripts/edu_logger.py` | **镜像版本去漂移** |
| **🔬 自检模块(v6.0 NEW)** | **`~/.hermes/scripts/self_check.py`** | **每个操作执行后自动验证结果（飞书文档存在/重复条目/wrong_based填充/ABC分布）** |
| **📰 学习日报(v6.0 NEW)** | **`~/.hermes/scripts/daily-report-cron.py`**, cron 7:30AM | **每日学习日报自动生成（错题回顾+趋势+薄弱TOP3+艾宾浩斯复习提醒+飞书发布）** |
| **🌐 math_sys v2 Web Blueprint** | **`~/edu-hub/math_sys/v2/`** (34KB routes.py), 门户注册于 `/math/` | **Flask Web 应用: 学生在线做题(API提交)+家长仪表盘+错题本+学习日报+文本导入。⚠️ 模板已写好但在 `templates_unused/`，`templates/` 为空，未激活部署** |

## 统一错题本机制（v4.1核心变更）

不再每张试卷建一个错题文件，而是维护一个**「三年级数学错题本」**文档，持续追加。

### 追加流程

```
新试卷解析 → 检测到错题
    │
    ├── 从 edu-system-state.json 读取 wrong_book_token
    │     └── 不存在 → 创建新文档，保存 token
    │
    ├── 读取本地 wrong_book.md
    │     └── 统计已有条目数 (正则匹配 ## 错题 \d+)
    │
    ├── 构建新条目 (编号延续)
    │     ├── 试卷: xxx
    │     ├── 题目: xxx
    │     ├── 原始答案: xxx
    │     ├── 正确答案: xxx
    │     └── 记录时间: xxx
    │
    ├── 追加到本地文件末尾
    └── 同步到飞书文档 (write_md_to_feishu)
```

### 错题记录格式

```markdown
## 错题 1
**试卷**: 2025学年（下）三年级数学阶段练习（二）
**题目**: （原卷写600，应为计算错误，正确结果为28+64+72=164）
**原始答案**: 600
**正确答案**: 28+64+72=164
**记录时间**: 2026-04-27 21:18
```

### 原始答案提取（extract_original_answer）

| 输入格式 | 提取结果 | 匹配方式 |
|:--------|:--------|:---------|
| `原卷写600...` → | `600` | `re.search(r'原卷写(\d+)')` |
| `120×3=360 (×)` → | `360` | 去掉(×)后，取等号后第一个token |
| `原卷计算错误...` → | `未明确标注` | 无明确原始值时的兜底 |

## 完整处理流程

```
用户创建飞书文档（文本，含题目+答案+错题标记）
       │
       ▼ 放入「上传入口」目录 (Cj4sfKoXulTe...)
       │
       ▼ (cron 8:00 / 20:00)
   raw-exam-parser.py (v4.1)
       │
   1. 扫描上传入口 → 只取 type=docx（飞书文档）
   2. 读取文档内容 → read_doc_content(doc_token)
   3. 提取试卷标题
   4. 两段式内容净化（v4.2 新增）：
      ├── ① clean_math_notation() — 清理 LaTeX 符号
      │     $..$ → 内容; \\boldsymbol{X} → X; \\times → ×;
      │     \\begin{array}...\\end{array} → 移除; 孤立大括号 → 移除
      ├── ② strip_exam_answers() — 移除答案和注释
      │     移除: (原卷写...)/(验算...)/计算步骤行/答：行;
      │     括号答案: （8640）→（ ）; 孤立的 $ → 移除
   5. 检测错题（3策略级联，依次尝试）：\n      ├── 策略1: 错题章节（「错题」「错误分析」「错题本」等标题下内容）\n      ├── 策略2: 行内标注（「原卷写XXX…正确结果为YYY」模式）⭐新增\n      └── 策略3: 内联标记（行末(×)符号）
       │
   ├──→ 「解析试题」目录 (标题_试题)
   │
   ├──→ 「错题本」← 统一追加到「三年级数学错题本」
   │
   └──→ 原始文档移入「已处理备份」（代替删除）
```

## 错题检测策略（详细）

### 策略1：错题章节
检测文档中 `## 错题` / `## 错误分析` / `## 订正` 等章节标题下的内容。

支持两种子格式：
- **字段标记行**：`错误原因：xxx` + `正确答案：xxx`
- **编号题目行**：`1. 题目` / `(1) 题目` / `错题1. 题目`

### 策略2：行内标注（⭐新增）
专为教师批改试卷后的标注习惯设计。在括号内标注错误信息和正确结果：

```regex
# 格式A：有原始错误答案
（原卷写600，应为计算错误，正确结果为28+64+72=164）

# 格式B：仅有错误类型和正确步骤
（原卷计算错误，正确步骤：800÷25=32，32×4=128）
```

提取逻辑：
- `错误原因`：匹配 `应为XXX错误` 或 `原卷XXX错误` 或兜底 `计算错误`
- `正确答案`：匹配 `正确结果为` 或 `正确步骤` 后的内容

### 策略3：内联标记
检测行末的 `(×)` / `（×）` 符号。**关键陷阱**：乘法符号 ×（U+00D7）与错题标记是同一字符，只匹配行末模式不匹配乘式中间的×。

## 飞书目录结构

```
数学/  (token: Xeo8fW8MvlrYPTdk65WcXppPnJf)
  ├── 上传入口/      ← 🟢 用户放置飞书文档（纯文本）
  ├── 解析试题/      ← 🔵 系统解析后的试题文档
  ├── 错题本/        ← 🟠 **「三年级数学错题本」**（唯一文件，持续追加）
  ├── 已处理备份/    ← 📦 解析后原始文档移入
  └── 练习题/
      ├── 每日练习/  ← cron 7AM 自动生成
      └── 单元练习/  ← 手动命令生成

日志文件:
  ~/edu-hub/logs/c003-activity.log  ← 🪵 所有自动/手动操作的统一日志
```

## 日志系统（v4.3 新增）

### 日志位置
- **日志文件**: `~/edu-hub/logs/c003-activity.log`
- **日志模块**: `~/.hermes/scripts/edu_logger.py` / `~/edu-hub/scripts/edu_logger.py`

### 日志内容
所有 C003 相关操作均记录时间戳、操作类型和详细信息：

| 触发方式 | 记录哪些操作 | 示例 |
|:---------|:------------|:-----|
| ❄️ cron 7:00 | 每日练习生成成功/失败 | `[2026-04-27 22:28:36] [INFO] 今日练习飞书文档补创建 | date=2026-04-27, doc_token=JqpwddXBDoDPSlxDDQ8cWUJ3nac` |
| ❄️ cron 8&20:00 | 试卷解析开始/结束、错题数、备份 | `[2026-04-27 20:01:00] [INFO] 试卷解析开始 | doc=测试卷, token=B1x...` |
| ❄️ cron 8&20:00 | 错题本追加成功/失败 | `[2026-04-27 20:01:00] [INFO] 错题本追加成功 | 试卷=第二单元, 新增=3, 累计=12` |
| ⌨️ 手动命令 | 同上的手动触发 | 同上，分隔线标明手动触发 |

### 查看日志
```bash
# 查看最近30条日志
tail -30 ~/edu-hub/logs/c003-activity.log

# 查看今天的日志
grep "$(date +%Y-%m-%d)" ~/edu-hub/logs/c003-activity.log

# 查看错误日志
grep "\[ERROR\]" ~/edu-hub/logs/c003-activity.log

# 通过代码查看
python3 -c "from edu_system_common import get_recent_logs; print(get_recent_logs())"
```

## 修复工具：fix-daily-practice.py（v4.4 新增）

当每日练习飞书文档创建失败时，使用此工具自动补创建。

```bash
# 检查并修复今天
python3 ~/.hermes/scripts/fix-daily-practice.py

# 修复指定日期
python3 ~/.hermes/scripts/fix-daily-practice.py --date 2026-04-27

# 检查最近7天
python3 ~/.hermes/scripts/fix-daily-practice.py --all

# 显示帮助
python3 ~/.hermes/scripts/fix-daily-practice.py --help
```

检测逻辑：读取 `edu-system-state.json` → 检查 `daily_practice_log` 中的 `doc_token` → 为空则读取本地 wiki 文件 → 创建飞书文档 → 填充内容 → 更新状态。

## Token 管理（新增）

所有飞书文件夹 token 集中在 `~/.hermes/feishu-tokens.json` 管理。

| 机制 | 说明 |
|:----|:-----|
| **集中存储** | 12个文件夹的 name/alias/token/parent 统一登记 |
| **自动加载** | `feishu_tokens.py` 提供 `get_token("名称")` 供各脚本调用 |
| **每日验证** | cron 3:00 自动检查所有 token 是否有效、有无漂移 |
| **兜底机制** | 各配置文件保留硬编码值，JSON加载失败时回退 |

## 定时任务

| 时间 | 任务 | Cron ID |
|:---|:----|:-------|
| 🕐 3:00 | Token有效性验证（检查12个文件夹） | `385eb173f17a` |
| 🕐 7:00 | 每日练习生成（ABC轮换+wrong_based填充+自检） | `cdb6561260ec` |
| 🕐 7:30 | 学习日报生成（错题回顾+趋势+薄弱TOP3+复习提醒+自检） | `72d68e37c0cf` | 🟢 v6.0 新增 |
| 🕐 8:00, 20:00 | 原始题解析(统一错题本追加+同步state.wrong_questions+自检) | `f6771e5db4d4` |
| 🕐 9:00 | Feishu→Wiki同步 | `d5329fd6e21f` |
| 🕐 — | C003 Web 自动启动 | ❌ **未部署**（模板在 templates_unused/ 未激活，需先部署） |

## Web 模块架构（math_sys v2 Blueprint）

> ⚠️ **实际架构 vs 旧版文档**：之前版本描述了一个 port-5001 独立 Flask 应用，但实际代码以 Blueprint 形式集成在 `~/edu-hub/math_sys/v2/` 中，通过门户 app.py 注册到 `/math/` 路由前缀。

### 实际文件位置

| 内容 | 路径 | 说明 |
|:----|:-----|:-----|
| Blueprint 注册 | `~/edu-hub/math_sys/v2/__init__.py` | Flask Blueprint `math_v2` |
| **全部路由与API** | **`~/edu-hub/math_sys/v2/routes.py`** (34KB) | **16条路由/API，含在线答题、提交批改、错题本、日报** |
| 状态管理器 | `~/edu-hub/math_sys/v2/state_manager.py` | 掌握度、错题、练习日志管理 |
| 文本导入器 | `~/edu-hub/math_sys/v2/text_importer.py` | 外部AI识别文本导入 |
| 门户注册 | `~/edu-hub/app.py` | `register_blueprint(math_v2, url_prefix="/math")` |
| **模板目录（空）** | **`~/edu-hub/math_sys/v2/templates/`** | **蓝图引用此目录但为空 → 页面渲染失败** |
| **模板文件（未启用）** | **`~/edu-hub/math_sys/v2/templates_unused/`** | **全部HTML模板在此，需移入 `templates/` 激活** |
| 静态文件 | `~/edu-hub/math_sys/v2/static/style.css` (19KB) | 深色科技风CSS |
| 练习题库 | `~/edu-hub/math_sys/data/practice_bundle.json` | 预置练习题目 |

### 已实现的路由与API

| 路由 | 功能 | 状态 |
|:----|:-----|:-----|
| `GET /` | 角色选择（家长/学生） | 模板在templates_unused |
| `GET /parent/dashboard` | 家长仪表盘 | 同上 |
| `GET /practice/` | 学生在线做题（MathJax渲染） | 同上 |
| `GET /wrong-book/` | 错题本查看 | 同上 |
| `GET /report/` | 学习日报 | 同上 |
| `GET /import/` | 文本导入（3场景提示词） | 同上 |
| `GET /api/dashboard` | 仪表盘数据 | 代码就绪 |
| `GET /api/practice/today` | 今日练习题目 | 代码就绪 |
| `POST /api/practice/submit` | 答题提交+实时批改 | 代码就绪 |
| `GET /api/wrong-book` | 错题本数据 | 代码就绪 |
| `GET /api/daily-report` | 日报数据 | 代码就绪 |
| `POST /api/practice/save-wrong` | 保存错题 | 代码就绪 |
| `POST /api/practice/save-result` | 保存练习结果 | 代码就绪 |
| `GET /api/import/prompts` | 导入提示词 | 代码就绪 |
| `POST /api/import/submit` | 提交导入内容 | 代码就绪 |

### 激活步骤（待执行）

```bash
# 将模板从 templates_unused/ 移到 templates/
cp -r ~/edu-hub/math_sys/v2/templates_unused/* ~/edu-hub/math_sys/v2/templates/
# 重启门户服务
python3 ~/edu-hub/scripts/c003-web-manager.py restart
```

### v7.0 在线 API（新增，session模式）

| 路由 | 方法 | 功能 |
|:-----|:----|:-----|
| `/math/student/` | GET | 🎯 学生首页门户 |
| `/math/challenge/` | GET | 🏆 大挑战页面 |
| `/math/achievements/` | GET | 📊 成就页 |
| `/math/api/v2/practice/start` | POST | 开始每日练习（10题，含挑战+综合题） |
| `/math/api/v2/challenge/start` | POST | 开始大挑战（20题/30分钟限时） |
| `/math/api/v2/session/{id}/current` | GET | 获取当前题目（不含答案） |
| `/math/api/v2/session/{id}/submit` | POST | 提交一题答案，返回正确/错误+正确答案 |
| `/math/api/v2/session/{id}/complete` | POST | 完成会话+更新知识图谱掌握度+错题本入库 |
| `/math/api/v2/coins/balance` | GET | 获取金币余额（共享英语v3币池） |

### 实践页面UI陷阱（2026-05-12）

详见 `references/practice-ui-pitfalls.md`：

1. **word_problem 类型无输入框** — `renderQuestion()` 未处理 `word_problem`/`calculation` 类型，题目显示后无任何输入控件。修复：增加 handler 分支。
2. **无返回导航** — 做题页面头部无返回按钮，必须做完所有题目才能退出。修复：增加 ← 返回按钮。

所有修改需重启 Flask 应用才能生效（模板缓存）。

### 设计陷阱

- **templates/ vs templates_unused/**：Blueprint 的 `template_folder="templates"` 指向空目录。怀疑是开发过程中模板被移出但未同步激活。部署前需核对模板中的静态引用路径是否正确。
- **`~/edu-hub/math/` 目录（非 math_sys）**：这个目录仅有 `data/` 子目录，是另一个不完整的 Web 尝试，与当前架构无关。
- **state_manager 与文档管线的状态文件（`edu-system-state.json`）不同步**：Web 版有自己的状态管理，与 cron 文档管线的状态文件是两套数据源。整合时需决定统一数据源。

### 会话管理（v7.0 新增）

| 组件 | 路径 | 用途 |
|:----|:-----|:-----|
| Session管理器 | `~/edu-hub/math_sys/v2/session_manager.py` | Session生命周期管理（创建/读题/提交/完成），JSON文件持久化 |
| 大挑战引擎 | `~/edu-hub/math_sys/v2/challenge_engine.py` | 大挑战出题（20题/30分钟限时）、批改、金币计算 |

### 学生端模板（v7.0 激活）

| 模板 | 路径 | 说明 |
|:-----|:-----|:------|
| 学生首页 | `templates/v2/student_portal.html` | 新手引导式卡片布局：金币/等级+练习+挑战+成就+错题 |
| 大挑战页 | `templates/v2/challenge/session.html` | 倒计时进度条+逐题API+结果页+金币奖励动画+confetti |
| 成就页 | `templates/v2/achievements.html` | 等级进度条(7级)/金币统计/连续打卡/勋章墙(12枚)/本月柱状图 |

```bash
# 启动Web模块（通过门户）
cd ~/edu-hub && python3 app.py

# 检查Web是否运行
curl -s http://localhost:5000/math/health | python3 -m json.tool
```

### ×符号冲突
乘法符号 ×（U+00D7）与错题标记是同一字符。只匹配 `(×)` 或行末空格后的裸×。

### 错误原因误识别
`错误原因：xxx` 曾被正则捕获为新错题。已从新题检测分支中移除「错误」关键词。

### re.findall 捕获组陷阱
内联 `(×)` 仅3字符，被 `len>5` 过滤。必须用 `re.finditer()` 提取完整行。

### 原始答案提取误抓
对 `原卷计算错误...` 格式，正则 `原卷[写答]?\s*(\S+)` 会错误捕获 `计算错误...`。需要要求明确 `原卷写` 或 `原卷答` 前缀，且限制捕获长度 `\S{1,10}`。

### clean_math_notation 处理顺序
`$...$` 和 `\boldsymbol{...}` 同时出现时，**必须先处理 $ 再处理 \\**。如果先移除 `\boldsymbol{`，残留的 `}` 会破坏 `$...$` 的配对匹配。正确顺序：
1. 先 `re.sub(r'\$([^$]+)\$', ...)` 移除 $ 标记
2. 再 `re.sub(r'\\boldsymbol\{([^}]*)\}', ...)` 提取 LaTeX 命令内容
3. 最后替换 `\times` → `×`、`\div` → `÷`

### 两段式管线处理顺序（v4.2）
`build_exam_content()` 中 **必须先 `clean_math_notation` 再 `strip_exam_answers`**，不能反过来。

原因：`（$\boldsymbol{8640}$）` 这样的括号答案在 clean 之前被 `$...$` 包裹。如果先 `strip_exam_answers`，正则 `（\d+）` 匹配不到 `（$\boldsymbol{8640}$）`，因为括号内不是纯数字。必须在 clean 之后变成 `（8640）` 再匹配。

反直觉之处：`strip_exam_answers` 原本可通过 `$...$` 是否存在来判断一行是否为计算步骤。但 clean_math 先执行后，`$` 已被清除，因此**计算步骤行必须用关键词前缀直接判定**（如 `速度[:：]`、`时间[:：]`、`剩下的页数[:：]`），不能依赖 `$` 作为信号。

### strip_exam_answers 的行首锚点陷阱
正则中的 `^` 锚点是关键，区分答案注释与题目文本：
- ✅ `^[（(]?(原卷写|验算[：:])` — 只匹配以 `（原卷写` 或 `（验算：` 开头的行
- ❌ 无 `^` 锚点的 `(原卷写|验算[：:])` — 会误匹配 `二、竖式计算，打*的要验算（12☆）` 中的"验算"

类似问题：`^一共\s*\d+\s*[本个只米元页]` 匹配答案结论行 "一共143页"，但不匹配题干 "王老师共有1000本练习本..."（因为"共有"不等效于"一共"）。

### 镜像版本同步的 patch 边界风险
edu-hub 镜像版本（`edu-hub/scripts/`）与主版本（`.hermes/scripts/`）结构不同：
- 不同的 import（`edu_common` vs `edu_system_common`）
- 不同的行号和函数组织
- patch 中的 old_string 在镜像版本中的匹配边界可能意外包含相邻函数的 def 行

**最佳实践**：对镜像版本做大幅改动时，直接用 read_file 确认当前内容，避免跨函数边界的 patch old_string。

### 「订正」触发词误匹配 (2026-04-27)
`parse_wrong_questions()` 策略1的触发词 `订正` 会匹配到「判断并订正」这类章节标题，导致从该点到文档末尾全部被当作错题章节。**修复**：从触发词列表移除 `订正`，仅保留 `错题` / `错误分析` / `错题本` 等明确指示错题的词。

### ✅ 正确标记误判为错题 (2026-04-27)
练习完成情况汇总文档中，几乎所有题目都标 ✅ 正确。策略1被误触发后会将 ✅ 行也当作错题捕获。**修复**：判定错题前检查条目是否包含 `✅ 正确` / `批改：全对` / `打钩` 且无 ×/✗ 标记 → 跳过。

### 策略2 存储批注文字而非原题 (2026-04-27)
策略2命中 `（原卷写600，应为计算错误，正确结果为28+64+72=164）` 时，直接将批注文字存为 `question` 字段，丢失了学生实际做错的原题。**修复**：匹配到批注后向上追溯原题文本，或将批注提取为结构化字段。

### 章节结束边界缺失 (2026-04-27)
策略1检测错题章节后，使用 `^#{1,4}\s` 判断章节结束。但文档后续使用非标题分隔符（`图片 11：练习5.1`、`---`）时检测不到结束点，导致吞掉整个文档后半部分。**修复**：增加 `图片 \d+`、`---` 等作为可选章节结束标记。

### 状态文件 token 持久化
统一错题本的 token 存储在 `edu-system-state.json` 的 `wrong_book_token` 字段。设计要点：
- 读取时检查有效性（尝试 `read_doc_content` 调用）
- 无效时自动重建新文档并更新 token
- 本地镜像文件 `wrong_book.md` 与飞书文档同步

## 维护提醒

两个版本的 `parse_wrong_questions()` 必须同步修改：
- `~/.hermes/scripts/raw-exam-parser.py`（cron 运行版本）
- `~/edu-hub/scripts/raw-exam-parser.py`（edu-hub 镜像版本）

Token 修改只需操作 `~/.hermes/feishu-tokens.json` 一个文件。

> 适配器架构详解参见 `references/v4.5-adapter-architecture.md`（从 exam-grading-pipeline 合并）。
> Agent上下文调用C003脚本参见 `references/agent-execution-guide.md`（execute_code沙箱中的importlib加载+预检查清单+工具异常处理）。
> **实际架构调查（2026-05-12）**：`references/web-system-survey-2026-05-12.md` — 包含 math_sys v2 Blueprint 实际状态、templates_unused/ 激活问题、两套数据源分析、coins_manager 跨系统端口模式。

## 常用命令

```bash
python3 ~/.hermes/scripts/raw-exam-parser.py                    # 全覆盖解析
python3 ~/.hermes/scripts/raw-exam-parser.py --incremental       # 增量解析
python3 ~/.hermes/scripts/feishu_tokens.py                      # 验证Token有效性
python3 ~/.hermes/scripts/feishu_tokens.py --list               # 查看Token注册表
python3 ~/edu-hub/scripts/daily-practice.py                     # 手动每日练习
python3 ~/edu-hub/scripts/wrong-analyzer.py summary              # 查看错题摘要
python3 ~/.hermes/scripts/daily-report-cron.py --summary         # 查看今日日报摘要
python3 -c "from self_check import *; run_self_check()"          # 全面自检
python3 ~/edu-hub/scripts/c003-web-manager.py start             # 启动Web增强版
python3 ~/edu-hub/scripts/c003-web-manager.py stop              # 停止Web增强版
python3 ~/edu-hub/scripts/c003-web-manager.py status            # 查看Web状态
python3 ~/edu-hub/scripts/c003-web-manager.py logs              # 查看Web日志
```

## 自检机制 (v6.0 新增)

每个 C003 操作执行后自动验证结果。自检覆盖:

| 检查项 | 触发场景 | 验证内容 |
|:-------|:---------|:---------|
| `check_feishu_doc_exists` | 试卷解析/每日练习/学习日报 | 飞书文档存在且内容有效 |
| `check_state_format_version` | 每次状态加载 | 状态文件格式为v6.0 |
| `check_practice_wrong_based` | 每日练习生成后 | wrong_based字段不为空 |
| `check_topics_rotation` | 每日练习生成后 | 知识点覆盖ABC三组分布 |
| `check_no_duplicate_wrong_entries` | 错题本追加后 | 同一(试卷,题目)对无重复 |
| `check_state_updated` | 任意操作后 | 指定状态字段已正确更新 |

### 集成方式

各脚本在关键步骤后调用自检:
```python
from self_check import SelfCheck
SelfCheck.check_feishu_doc_exists(doc_token, "每日练习文档")
# 或批量自检:
results = SelfCheck.run_practice_checks(practice, child_data, doc_token)
```

## 技术陷阱 — v6.0 实施教训

### 文件名连字符导入问题

Python 脚本通过 `import` 语句无法导入文件名包含连字符的模块（如 `daily-practice-cron.py`）。必须使用 `importlib`:

```python
import importlib.util, sys

spec = importlib.util.spec_from_file_location(
    'module_alias', '/path/to/daily-practice-cron.py')
module = importlib.util.module_from_spec(spec)
sys.modules['module_alias'] = module  # ← 必须注册！否则模块内跨模块 import 会失败
spec.loader.exec_module(module)
# 使用: module.function_name()
```

**⚠️ 关键陷阱**: `sys.modules[alias] = module` 不能省略。如果被加载的模块内部有 `from edu_system_common import *` 等跨模块导入，且 `edu_system_common` 通过相对路径或 `sys.path` 查找，未注册会导致 `ModuleNotFoundError`。这个注册让 Python 的导入系统能找到已加载的模块。

**适用场景**: 测试验证、集成测试、跨模块函数调用。  
**替代方案**: 在 `__init__.py` 中加别名，或使用 `sys.path.insert` + 文件名替换。

### 状态文件格式迁移的兼容层设计

当状态文件格式需要升级（如 v4.5→v6.0），必须设计向后兼容层:

```python
def _migrate_to_v6(state: dict) -> dict:
    """检测旧格式 → 自动迁移到 v6.0"""
    if state.get("version") == "6.0":
        return state  # 幂等：已迁移的不重复处理
    
    # 将旧格式的顶层字段打包到 children.default
    child_data = {k: state.pop(k) for k in OLD_TOP_LEVEL_KEYS if k in state}
    state["version"] = "6.0"
    state["children"] = {"default": {"name": "默认学生", **child_data}}
    return state

def load_state() -> dict:
    """加载状态文件（含自动迁移）"""
    state = _read_state_file()
    state = _migrate_to_v6(state)  # 自动升级
    # 兼容层：将 children.default 的数据暴露到顶层
    for k, v in state.get("children", {}).get("default", {}).items():
        state.setdefault(k, v)  # 旧代码 state["knowledge_points"] 仍可用
    return state
```

**关键原则**: 
- 迁移函数必须是**幂等的**（重复执行不产生副作用）
- 旧格式访问路径保持可用（`state["knowledge_points"]` 仍能访问）
- 写入前清理兼容层字段，只写标准格式

### 大规模升级的架构方法

当改造范围超过 10 个文件时，使用以下模式:

1. **先做产品评估** — 站在最终用户角度列出所有优化点，按优先级排序（P0-P3）
2. **再写架构文档** — 设计数据流、状态结构、模块关系，统一共识
3. **并行委派工作流** — 按功能域切分（核心管道/知识图谱/报告），每个工作流在独立子代理中实现
4. **先测试再交付** — 所有模块导入验证 → 状态迁移验证 → 全流程集成测试
5. **三重交付** — Registry(版本号+资产+变更日志) + Skill(组件表+cron表+陷阱) + Memory(一句摘要)

### 自检防止"静默失败"

v6.0 之前系统存在"执行了但结果不对"的问题（重复文件、空字段、未同步）。自检机制的核心设计:

- **执行后立即验证**: 每个操作完成后紧跟验证，不等待下次调度
- **具体到字段级别**: 检查具体字段值而非笼统的"是否成功"
- **可见性**: 所有自检结果打印到 stdout，同时记录到日志
- **非阻断式**: 自检失败不影响流程，但标记问题供后续排查
