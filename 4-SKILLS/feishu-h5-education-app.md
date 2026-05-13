---
name: feishu-h5-education-app
category: education
description: >-
  通过飞书H5网页应用（网页应用能力）+ 本地Flask后端搭建交互式教育工具的模式。
  包含 v1 基础架构 + v2 Blueprint 零破坏扩展模式。学生可在飞书内完成默写/练习/测验，
  支持双模态角色、智能内容引擎、防作弊、断点续传、飞书卡片推送、错题本变式题、自动周报。
  涵盖单词练习的题型配比策略、类别标识+规则提示、轮询重练、同义词首字母填空、
  en2cn中文标点宽松判题、分类术语说明页等模式。
version: 3.8.0
---

# 飞书H5教育应用搭建模式

## 适用场景

需要在飞书生态内构建交互式学习工具（如默写、练习、测验），利用飞书文档管理学习资料，并需要逐步升级功能（从 v1 单体到 v2 模块化扩展）。

## 架构模式 (v1 → v2 升级)

```
v1 阶段: 单体 Flask 应用
  ~/edu-hub/<subject>/
    ├── app.py              # Flask主入口（所有路由在此）
    ├── state_manager.py    # 状态管理器
    ├── english_parser.py   # 内容解析器
    ├── static/style.css    # 移动端CSS
    ├── templates/*.html    # 前端模板
    └── data/               # JSON 数据存储

v2 阶段: Blueprint 模块化扩展（零破坏）
  ~/edu-hub/<subject>/
    ├── app.py              # 仅加3行：import os, app.secret_key, register_blueprint
    ├── v2/                 # ← 所有新增代码集中在此
    │   ├── __init__.py     # Blueprint 'eng_v2' 注册
    │   ├── models.py       # 数据类定义 (dataclass)
    │   ├── routes.py       # 23+ API 端点 (Flask Blueprint)
    │   ├── *.py            # 14+ 业务模块（见下文）
    │   ├── templates/      # v2 前端模板
    │   └── static/style.css
    └── data/
        ├── content.json    # v1 数据不变
        ├── state.json      # v2 升级版（顶部加 students 维度）
        └── v2/             # v2 新增数据
            ├── students.json
            ├── wrong_book.json
            └── session_store/
```

### 核心原则：零破坏

- v1 文件原封不动（app.py, state_manager.py, english_parser.py, 旧模板, 旧CSS）
- v2 代码全部在 `v2/` 子目录下
- app.py 仅加 3 行：`import os`, `app.secret_key = ...`, `app.register_blueprint(eng_v2, url_prefix='/v2')`
- state.json 通过幂等迁移自动升级（v1 数据包装为 `students.stu_default`）

## 标准 v2 模块清单

| 模块 | 文件 | 职责 |
|:----|:-----|:-----|
| 蓝图注册 | `v2/__init__.py` | 创建 Flask Blueprint |
| 数据模型 | `v2/models.py` | 6个 dataclass: StudentProfile, TaskConfig, Question, SessionState, WrongItem, Report |
| 路由 | `v2/routes.py` | 23+ REST API 端点 |
| 角色管理 | `v2/role_manager.py` | 家长/学生双模态 + @require_role 装饰器 + 多孩档案CRUD |
| Ollama桥接 | `v2/ollama_bridge.py` | 统一封装 Ollama API（单例模式, 重试, 超时, 模型回退） |
| 幂等迁移 | `v2/migration.py` | v1→v2 状态文件自动升级（可重复运行） |
| 内容引擎 | `v2/content_engine.py` | 遗忘曲线、20%错词混合、60/30/10难度比例、自适应推荐 |
| 试卷生成 | `v2/exam_generator.py` | 试卷模式（题型配置、时长估算、试题组装） |
| 批改引擎 | `v2/grading_engine.py` | 严格匹配 + Levenshtein模糊匹配(≤1容错) |
| 防作弊 | `v2/anti_cheat.py` | 切屏检测 + 30秒心跳 + 3次阈值告警 + 前端JS注入 |
| 断点续传 | `v2/resume_manager.py` | 会话持久化（每答一题即时保存）+ 恢复机制 |
| 错题本 | `v2/wrong_book.py` | 自动归集 + 知识点打标 + Ollama/预置变式题(3道) + 消灭卷 |
| 飞书卡片 | `v2/feishu_cards.py` | 任务卡片 + 成绩卡片 + 周报卡片（Markdown格式+按钮URL） |
| 报告引擎 | `v2/report_engine.py` | 日/周报 + 掌握度趋势 + Ollama学习建议 |

## 双模态角色系统

### 家长（超级管理员）
- 访问所有 /v2/parent/ 路由
- 配置任务参数、查看报告、管理错题库、设置系统参数
- 多孩支持：创建/切换学生档案，数据完全隔离

### 学生（学习者）
- 仅访问 /v2/dictation/ 和 /v2/wrong-book/ 学生端路由
- 无跳过、查看答案权限
- 答题页防作弊（无右键/F12/复制粘贴）

### 实现模式
```python
# role_manager.py
def require_role(role):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            # 检查 Flask session 或 URL 参数
            req_role = session.get('role', '') or request.args.get('role', '')
            if req_role != role:
                return jsonify({"error": f"需要 {role} 权限"}), 403
            return f(*args, **kwargs)
        return decorated_function

# routes.py 中使用
@eng_v2.route('/api/v2/students', methods=['GET'])
@require_role('parent')
def list_students():
    ...
```

## 智能内容引擎设计模式

### 遗忘曲线调度
- 复习间隔: 1天, 2天, 4天, 7天, 15天, 30天 (Ebbinghaus)
- 距 last_practiced 越久 → 优先级越高
- mastered 状态的项优先级大幅降低

### 20%错词混合
```python
# 从 state.json 提取 wrong_count > 0 的项目
# 按错误次数降序排序 → 选取 top 20%（至少3个）→ 随机打乱
```

### 60/30/10难度比例
| 难度 | 比例 | 来源 |
|:----|:----|:-----|
| 基础题 | 60% | new + learning 状态的常规题目 |
| 提升题 | 30% | 最近3天答错≥2次的高频错词 |
| 挑战题 | 10% | 其他批次的高难度词 / Ollama生成扩展题 |

### 自适应推荐
- 分析最近7天 exercises 记录
- 找出错误率最高的 top 3 批次
- 可选调用 Ollama 生成推荐理由

## 防作弊模式

### 前端
- Page Visibility API 监听切屏
- 失去焦点时弹出半透明遮罩 + 警告计数
- 禁用右键、F12、Ctrl+Shift+I/J/C
- 30秒心跳上报（POST /api/v2/session/{id}/heartbeat）

### 后端
- CheatMonitor 类：内存中管理会话状态
- 阈值(默认3次) → 自动标记 flagged
- 成绩卡片标注 "⚠️ 可能存在作弊"

## 飞书卡片模式

### 任务卡片
```
【英语默写】M1U1-M1U3 重点词汇突破
预计耗时15分钟 | 难度：★★★ | 包含20个历史错题

[开始答题](url) | [预览题目](url) | [调整参数](url)
```

### 成绩卡片
```
【成绩报告】小明 - M1U1
得分：90/100 | 正确率：90%
用时：15分钟 | 切屏次数：0

准确率趋势（最近7次）：
██ 100%
██████ 90%
█████ 85%
...

[查看错题](url) | [再来一组](url)
```

### 周报卡片
```
【英语周报】小明 - 第18周
本周学习 320 分钟 | 掌握度提升 12%
薄弱点TOP3: M1U1(60%) M1U2(75%) M1U3(80%)

下周建议：
1. 重点复习 M1U1 的动词
2. 每天10分钟听力
3. 完成错题消灭卷
```

## v2 目录结构模板

```
~/edu-hub/<subject>/v2/
├── __init__.py
├── models.py
├── routes.py
├── ollama_bridge.py
├── role_manager.py
├── migration.py
├── content_engine.py
├── exam_generator.py
├── grading_engine.py
├── anti_cheat.py
├── resume_manager.py
├── wrong_book.py
├── feishu_cards.py
├── report_engine.py
├── static/
│   └── style.css
└── templates/
    ├── base.html
    ├── index.html
    ├── parent/
    │   └── index.html
    ├── dictation/
    │   └── index.html
    └── wrong_book/
        └── index.html
```

## v3 统一学生端门户（v3.1 重构）

**2026-05-01 重构说明**：v3 从"v2 学生端答题扩展"重构为**统一学生端门户**，去除 v2/v3 概念分离。学生从 `/v3/portal` 进入，选择「学校听默」或「单词练习」，不再区分 v2 学生端和 v3 扩展。

### 两大功能模块

| 模块 | 入口 | 说明 |
|:----|:-----|:------|
| 📝 **学校听默** | `/v3/dictation` | 完成教师布置的单元听默，只默写从未默写过的词条（中文→输入英文） |
| 🧠 **单词练习** | `/v3/practice` | 基于 SM-2 记忆曲线的科学复习，每天不超过 10 分钟 |

### 核心架构

```
~/edu-hub/english/v3/
├── __init__.py              # Blueprint eng_v3 (url_prefix=/v3)
├── data_manager.py          # 数据层：听默记录 + SM-2 参数 + 学生统计
├── dictation_engine.py      # 学校听默引擎
├── practice_engine.py       # 单词练习引擎 (SM-2 + 3题型 + 时间控制)
├── routes.py                # 页面 + 20+ API 端点
├── static/style.css
└── templates/v3/
    ├── base.html
    ├── portal.html          # 学习门户 + 进度概览
    ├── dictation/
    │   ├── setup.html       # 选择批次 + 设置数量
    │   └── session.html     # 听默执行 + 语音播报
    ├── practice/
    │   ├── setup.html       # 选择词库范围
    │   └── session.html     # 练习执行 + 倒计时
    └── result.html
```

### 数据存储（独立于 v2 数据）

```
~/edu-hub/english/data/v3/
├── dictation_records.json   # 听默记录: {student_id: {key: [{date, correct}]}}
├── memory_curve.json        # SM-2 参数: {student_id: {word_en: {ease_factor, interval, repetitions, next_review_date}}}
└── student_stats.json       # 统计: {student_id: {avg_response_time, daily_completed, last_practice_date}}
```

### 模块一：学校听默（dictation_engine）

**业务规则**：
- 展示某批次下所有从未默写过的词条（中文→输入英文）
- 每种类型（单词/词组/短句）可独立设置数量
- 一旦默写过（无论对错），该词条不再出现在未来听默中
- 自动语音播报（浏览器 Web Speech API TTS）
- 完成后展示正确率 + 错词列表

**API 端点**：
```
GET  /v3/api/dictation/info        → 所有批次未默写统计
POST /v3/api/dictation/start       → 开始听默（随机抽取指定数量）
GET  /v3/api/dictation/current     → 获取当前题目
POST /v3/api/dictation/grade       → 判题 + 记录
POST /v3/api/dictation/complete    → 完成听默
```

### 模块二：单词练习（practice_engine）— v2（错误次数频次策略）

**2026-05-04 升级说明**：从 SM-2 记忆曲线改为**基于错误次数的频次策略**。每日测验由「新词（3-5个） + 错词复习（5-8个）」自动组合，跳过设置页直接开始。

#### 频次数据模型（`word_frequency.json`）

每个学生-单词对维护以下参数：

| 字段 | 类型 | 说明 |
|:----|:----|:------|
| `error_count` | int | 累计错误次数（只增不减） |
| `consecutive_correct` | int | 连续正确次数（答错归零） |
| `total_tests` | int | 总测试次数 |
| `history` | list | 最近 20 条记录 [{date, correct}] |
| `next_review_date` | str | 下次复习日期（YYYY-MM-DD） |

#### 频次调整规则

| 条件 | 频次等级 | 间隔 | 说明 |
|:----|:--------|:----|:------|
| error_count ≥ 2 | 🔴 高频 | 1 天 | 次日必测，连续追测 |
| error_count == 1 | 🟡 中频 | 2 天 | 隔日测试 |
| consecutive_correct ≥ 3 | 🟢 低频 | 7 天 | 每周抽测一次 |
| consecutive_correct ≥ 5 | ⚪ 归档 | 30 天 | 掌握区，每月抽查 |
| 其它（新词） | 🆕 新词 | 1 天 | 次日首次测试 |

#### 每日执行流程

```
Step 1: 新词 3-5 个
  ├─ 从 content.json 提取所有单词
  ├─ 过滤掉 word_frequency 中已有的
  └─ 随机取 count=5 个

Step 2: 复习 5-8 个
  ├─ 扫描 word_frequency，next_review_date ≤ today
  ├─ 按 error_count 降序排列
  └─ 取前 count=8 个

Step 3: 合并打乱后测试
  ├─ 每题标记 source: "new" 或 "review"
  └─ 提交后自动更新频次

Step 4: 自动打卡
  └─ complete_practice 时触发 record_checkin()
```

#### 两种题型 + 同义词扩展

| 题型 | 展示 | 学生操作 | 判题 | 触发条件 |
|:----|:-----|:---------|:----|:---------|
| ✍️ 拼写题 | 显示中文 | 输入英文 | `_grade_by_category()` 按类别判题 | 普通单词，55%概率 |
| 🔡 同义词首字母填空 | 显示中文+多个首字母提示框 | 每个框填写完整单词 | 分题计分（对1个得1分） | 拼写题触发且cn对应多个en |
| 🔤 看英说中 | 显示英文+发音按钮 | 输入中文 | 精确匹配 | 词组/短句强制 + 45%概率单词 |

- **类别标识系统**：每题顶部显示 `📖 单词` / `📝 词组` / `📜 短句` 徽标，绿色(单词/词组)提示"不区分大小写和标点符号"，红色(短句)提示"区分大小写和标点符号"
- **同义词检测**：自动扫描全量 content.json，构建 cn→[en...] 映射，仅对 >=2 个不同英文的 cn 触发 fill_first_letter
- **同义词判题**：`_grade_fill_first_letter()` 逐个比对，答案逗号分隔，支持大小写不敏感；前端每个框独立变色(绿/红)
- **同题型不连续超过 3 次**（`_rebalance_question_types`）
- **移除了选择题型**（v3.7 变更），更聚焦拼写和翻译能力
- **cn==en 回退**：当数据的中文释义与英文相同时，自动显示 `请输入单词/词组/短句 'xxx' 的正确拼写` 作为备用提示（`get_fallback_prompt()`）

#### UI 增强

- 每题顶部显示 **🆕 新词 / 🔄 复习** 标签
- 提交后显示频次状态：`⚠️ 已错N次` / `✅ 连续正确N次`
- 不再显示设置页，访问 `/practice` 直接开始
- 无每日限练（可多次练习）

#### 数据存储

```python
~/edu-hub/english/data/v3/
├── word_frequency.json   # 频次数据（替代旧的 memory_curve.json）
├── checkin.json          # 打卡记录 + 每日练习统计
├── dictation_records.json
└── student_stats.json
```

#### API 端点

```
GET  /v3/api/practice/info          → 单词总数/已测试数/新词数/复习数/归档数
POST /v3/api/practice/start         → 开始练习（自动组合新词+复习）
GET  /v3/api/practice/current       → 获取当前题目（含 freq 状态）
POST /v3/api/practice/submit        → 提交答案 + 频次更新
POST /v3/api/practice/complete      → 完成 + 自动打卡
GET  /v3/api/practice/status        → 今日练习状态
```

### 模块三：学习日历与打卡

**2026-05-04 新增** — 基于完成每日单词练习的自动打卡系统。

#### 打卡触发

- **自动**：`complete_practice()` 时调用 `record_checkin(student_id, correct, wrong, total)`
- **仅单词练习触发**，听默不触发
- 手动按钮（未打卡时显示"☀️ 去完成今日单词练习"）跳转到练习页

#### 日历展示

- 每月日历网格，已打卡日期绿色高亮
- 悬浮显示该日统计：`练习N题 · 正确N · 错误N`
- 顶部显示连续打卡天数
- 当月汇总：`今日已练习 N 题 · 正确率 X%`

#### API 端点

```
GET  /v3/api/checkin/month          → 当月打卡记录 + 连续天数 + 月度统计
POST /v3/api/checkin/today          → 今日打卡（保留用于手动补录）
```

#### 数据结构（`checkin.json`）

```json
{
  "students": {
    "student_id": {
      "2026-05": {
        "1": {"day": "1", "correct": 5, "wrong": 2, "total": 7},
        "2": {"day": "2", "correct": 8, "wrong": 1, "total": 9}
      }
    }
  }
}
```

#### 共享 API

```
GET  /v3/api/progress               → 进度概览（掌握数/待复习数/听默统计）
```

### 建议迁移路径（旧 v3 → 新 v3）

1. **移动端优先**：max-width: 480px，触摸目标≥44px
2. **难度色标**：基础题绿色(#52C41A)、提升题橙色(#FAAD14)、挑战题红色(#FF4D4F)
3. **全屏沉浸**：学生答题页隐藏导航/返回，只有答题流程
4. **即时反馈**：每题答完即刻批改（✅❌），1.2秒自动滚动到下一题
5. **防作弊遮罩**：切屏时全屏半透明黑底覆盖 + 警告文字
6. **骨架屏加载**：数据加载中显示灰色骨架动画

## 扩展新功能模块（迭代模式）

当需要为已有 C008 应用添加新功能（如学生管理、内容导入、密码登录、视觉识别）时，遵循以下三合一模式：

### 拓展示例：完整的功能扩展流程

以 v2.1 添加「学生管理 + 内容导入 + 密码登录」为例：

#### Step 1: 后端 API 层（routes.py）

**模式**: 每次扩展新增一个 API 端点组，遵循现有风格：

```python
# ============================================================
# API: 新功能区域标题
# ============================================================

@eng_v2.route("/api/v2/new-resource", methods=["GET"])
@require_role("parent")
def api_list_resource():
    try:
        result = some_list_function()
        return _json_success(result)
    except Exception as e:
        logger.exception("操作失败")
        return _json_error(f"操作失败: {e}", 500)
```

**关键模式**:
- 使用已定义的 `_json_success()` 和 `_json_error()` 辅助函数
- 新路线集中放在 routes.py 底部
- `@require_role("parent")` 自动校验 session 或 URL 参数
- 每个端点独立 try/except

**常见新增端点模式**:

| 操作 | 方法 | 路由 | 用途 |
|:----|:----:|:-----|:-----|
| 列表 | GET | `/api/v2/resource` | 获取列表 |
| 创建 | POST | `/api/v2/resource` | 新增条目 |
| 读取 | GET | `/api/v2/resource/<id>` | 获取单个 |
| 更新 | PATCH | `/api/v2/resource/<id>` | 改名字/属性 |
| 删除 | DELETE | `/api/v2/resource/<id>` | 删除+级联 |
| 清除 | DELETE | `/api/v2/resource/<id>/records` | 清记录但保留 |

#### Step 2: 业务逻辑层（role_manager.py 或新模块）

```python
# 文件持久化读写（JSON 原子写入）
def _write_data(data: dict) -> bool:
    try:
        tmp = FILE.with_suffix(".tmp")
        with open(tmp, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        tmp.replace(FILE)
        return True
    except Exception as e:
        logger.error("写入失败: %s", e)
        return False
```

**级联删除模式**（删除主记录时同步清除关联数据）：

```python
def delete_cascade(item_id: str) -> dict:
    result = {"main_deleted": False, "related_cleared": False}
    if delete_main(item_id):
        result["main_deleted"] = True
    # 清除关联数据（如错题本、会话记录）
    for related_file, extract_key in [("wrong_book.json", item_id), ...]:
        ...
    return result
```

#### Step 3: 前端模板（JS + HTML）

**UI 分区模式**：每个功能块用独立 section 包裹

```html
<div class="wizard-section">
    <div class="wizard-section-title">📋 功能标题 <button onclick="doCreate()">➕ 添加</button></div>
    <div id="listContainer">
        <div class="skeleton-card">加载中...</div>
    </div>
</div>
```

**JS 数据加载模式**（用 fetch + credentials: 'same-origin'）：

```javascript
function loadList() {
    fetch('/v2/api/v2/resource', {credentials: 'same-origin'})
        .then(r => r.json())
        .then(data => {
            if (!data.items || data.items.length === 0) {
                container.innerHTML = '<div style="color:#adb5bd;text-align:center;">暂无数据</div>';
                return;
            }
            container.innerHTML = data.items.map(item =>
                '<div class="child-card">...' +
                '<button onclick="doEdit(\'' + item.id + '\')">✏️</button>' +
                '</div>'
            ).join('');
        });
}
```

**JS 核心规则**:
- 所有 API 调用带 `credentials: 'same-origin'`
- 用 `prompt()` / `confirm()` 简化输入，无需复杂弹窗库
- 每个调用有错误处理 `if (d.error)`
- CRUD 操作后自动刷新列表

#### Step 4: 密码登录弹窗模式

角色选择页添加登录弹窗，覆盖到角色卡链接：

```html
<a href="javascript:void(0)" onclick="openParentLogin()">家长门户</a>

<div id="loginOverlay" class="modal-overlay hidden">
    <div class="modal-box">
        <div id="loginView">
            <input id="loginPassword" type="password" onkeydown="if(event.key==='Enter')doLogin()">
            <button onclick="doLogin()">🔓 登录</button>
        </div>
        <div id="setupView" class="hidden">
            <p>首次使用，请设置密码</p>
            <input id="setupPassword" type="password">
            <input id="setupConfirm" type="password">
            <button onclick="doSetup()">设置密码并进入</button>
        </div>
    </div>
</div>

<style>
.modal-overlay { position:fixed; top:0; left:0; right:0; bottom:0;
    background:rgba(0,0,0,0.5); z-index:9999;
    display:flex; align-items:center; justify-content:center; }
.modal-box { background:#fff; border-radius:16px; padding:1.5rem;
    width:100%; max-width:360px; }
.hidden { display:none !important; }
</style>
```

**认证后端模式**：密码存文件替代硬编码

```python
def has_parent_password() -> bool:
    return bool(_read_parent_password())
def set_parent_password(password: str) -> bool:
    if not password or len(password) < 4: return False
    return _write_parent_password(password)
def verify_parent_password(password: str) -> bool:
    stored = _read_parent_password()
    return bool(stored) and password == stored
# 修改 authenticate():
#   return username == "admin" and verify_parent_password(password)
```

#### Step 5: 内容导入格式适配

在 API 端做自动适配，避免用户需了解底层格式：

```python
# 自动添加批次头（如果用户没写）
if not re.search(r'^批次\s*[:：]', text, re.MULTILINE):
    lines = [l.strip() for l in text.split('\n') if l.strip()]
    text = f"批次: {source_doc}\n单词: {', '.join(lines)}"
```

---

#### Step 6: 添加视觉/OCR 识别模块扩展模式

当需要添加「拍照上传识别」功能（家长拍照→模型识别→预览编辑→确认导入）时，使用此模式。

**触发条件**：用户需要从图片/照片中提取英语学习内容（课本单词、试卷）自动导入系统。

**架构模式 — 可配置后端：**

```python
# vision_importer.py 核心结构
class VisionImporter:
    """视觉识别器，支持多后端配置"""

    def recognize(self, image_base64: str, content_type: str) -> dict:
        """识别图片内容，返回系统标准格式文本"""

        # 策略：云端优先 → 本地 Ollama 回退
        if self._cloud_configured:
            result = self._call_cloud_vision(image_base64, prompt)
            if result: return result

        result = self._call_ollama_vision(image_base64, prompt)
        if result: return result

        return {"success": False, "error": "无可用视觉后端"}
```

**后端配置（环境变量驱动）：**

```python
# 本地 Ollama
OLLAMA_ENDPOINTS = ["http://localhost:11434", "http://192.168.3.102:11434"]
OLLAMA_VISION_MODEL = "qwen3-vl:4b"

# 云端视觉模型（可选）
CLOUD_API_URL = os.environ.get("VISION_CLOUD_API_URL", "")
CLOUD_MODEL = os.environ.get("VISION_CLOUD_MODEL", "")
CLOUD_API_KEY = os.environ.get("VISION_CLOUD_API_KEY", "")
CLOUD_PROVIDER = os.environ.get("VISION_CLOUD_PROVIDER", "openai")
```

**API 端点模式（3个端点）：**

| 方法 | 路径 | 用途 | 权限 |
|:----|:-----|:-----|:----:|
| GET | `/api/v2/vision/status` | 检查视觉后端可用状态 | 公开 |
| GET | `/api/v2/vision/prompt?type=words\|exam` | 获取场景提示词（外部AI使用） | 公开 |
| POST | `/api/v2/vision/recognize` | 上传图片+识别 | parent |
| POST | `/api/v2/vision/confirm-import` | 确认导入已修正的结果 | parent |

**图片上传处理模式：**

```python
@eng_v2.route("/api/v2/vision/recognize", methods=["POST"])
@require_role("parent")
def api_vision_recognize():
    # 1. 从请求体获取 base64 图片
    image = data.get("image", "")
    # 2. 去除 data:image/xxx;base64, 前缀
    if "," in image:
        image = image.split(",", 1)[1]
    # 3. 估算大小（base64 → 原始大小 ≈ len * 3/4）
    if len(image) * 3 // 4 > 5 * 1024 * 1024:
        return _json_error("图片超过 5MB 限制", 400)
    # 4. 调用视觉模型识别
    result = vision_recognize(image, content_type)
    # 5. 解析输出格式有效性
    parsed = _parse_vision_output(recognized_text)
    return _json_success({"recognized_text": recognized_text, "parsed": parsed})
```

**前端图片上传模式（FileReader + base64）：**

```html
<!-- 拍照/上传按钮 -->
<div class="upload-area" onclick="document.getElementById('visionFileInput').click()">
    <div class="upload-icon">📸</div>
    <div class="upload-text">点击拍照或选择照片</div>
</div>
<input type="file" id="visionFileInput" accept="image/*" capture="environment" style="display:none;" onchange="handleVisionFile(this)">

<!-- 图片预览 -->
<img id="visionImage" style="max-height:300px;object-fit:contain;">

<!-- 可编辑识别结果 -->
<textarea id="visionResultText" rows="8" style="font-family:monospace;"></textarea>

<!-- 操作按钮 -->
<button onclick="doVisionConfirm()">✅ 确认导入</button>
<button onclick="resetVisionUpload()">🔄 重新识别</button>
```

```javascript
// JS 核心流程
function handleVisionFile(input) {
    var file = input.files[0];
    if (!file || file.size > 5 * 1024 * 1024) return;

    var reader = new FileReader();
    reader.onload = function(e) {
        currentImageBase64 = e.target.result;  // 含 data:image/jpeg;base64, 前缀
        document.getElementById('visionImage').src = currentImageBase64;
        document.getElementById('visionRecognizeBtn').disabled = false;
    };
    reader.readAsDataURL(file);
}

function doVisionRecognize() {
    // 发送 base64 到后端识别
    fetch('/v2/api/v2/vision/recognize', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({
            image: currentImageBase64,
            content_type: currentVisionType,
        }),
        credentials: 'same-origin'
    }).then(r => r.json()).then(d => {
        if (d.error) { /* 显示错误 */ return; }
        // 在可编辑文本框中显示结果
        document.getElementById('visionResultText').value = d.recognized_text;
        // 检查格式有效性
        if (d.parsed && d.parsed.has_valid_format) {
            showBanner('success', '格式正确，可直接导入');
        } else {
            showBanner('warning', '缺少"批次:"或"单词:"字段，请补充');
        }
    });
}

function doVisionConfirm() {
    var text = document.getElementById('visionResultText').value;
    // 用户已编辑修正 → 发送确认导入
    fetch('/v2/api/v2/vision/confirm-import', {
        method: 'POST',
        body: JSON.stringify({text: text, source_doc: batchName}),
        credentials: 'same-origin'
    });
}
```

**关键模式要点：**

1. **后端可配置** — 环境变量驱动，默认 Ollama 本地，设置 `VISION_CLOUD_*` 即切云端
2. **base64 去前缀** — 前端 FileReader.readAsDataURL 产生的含 `data:image/...;base64,` 前缀，后端需剥离
3. **图片大小校验** — base64 长度 × 3/4 ≈ 原始字节，限制 5MB
4. **格式校验** — 识别后立即检查是否包含 `批次:` 和 `单词:` 关键字段
5. **可编辑预览** — 使用 textarea 而非纯展示，允许用户修正模型识别错误
6. **优雅降级** — 所有后端不可用时返回友好错误提示，引导配置
7. **外部 AI 辅助** — 即使后端不可用，提示词模板仍可用（家长用微信/元宝等外部AI识别后粘贴）

**CSS 模式：**

```css
/* 上传区域 */
.upload-area {
    border: 2px dashed #d0d5dd; border-radius: 12px; padding: 2rem 1rem;
    text-align: center; cursor: pointer; transition: all 0.2s;
    background: #fafbfc;
}
.upload-area:hover { border-color: #4a6fa5; background: #f0f4ff; }
.upload-icon { font-size: 2.5rem; margin-bottom: 0.5rem; }

/* 加载动画 */
@keyframes spin { to { transform: rotate(360deg); } }
.spinner {
    width: 32px; height: 32px;
    border: 3px solid #e9ecef; border-top: 3px solid #4a6fa5;
    border-radius: 50%; animation: spin 1s linear infinite;
}

/* 结果 banner */
.vision-result-banner.success { background: #e6f7e6; color: #389e0d; }
.vision-result-banner.warning { background: #fff7e6; color: #d46b08; }
```

### 视觉模型陷阱：qwen3-vl:4b 对表格排版识别困难（2026-05-12 发现）

**问题**：qwen3-vl:4b 在处理结构化表格内容（如课本认字表/会写字表的多行多列表格）时，输出反复循环、无法提取结构化数据。对单张图片的响应会陷入对表格结构的重复描述，最终无法输出可用内容。

**影响场景**：
- 语文认字表（汉字+拼音+组词的三列表格）
- 语文会写字表（带田字格或行列结构的生字表）
- 任何包含行列对齐的教科书排版

**已验证的效果更好的方案**：
- **纯文本批量导入**：家长直接粘贴或输入文本格式内容（`批次: xxx\n汉字: 字1, 字2, ...\n词语: 词1, 词2, ...`）
- **单区识别**：如果使用视觉，引导用户拍单行或单个区域而非整页表格
- **外部 AI 辅助**：提示词模板供家长用微信/元宝等外部 AI 识别后粘贴（已在本模式中实现）

**回退约定**：对于表格类教材页面，默认走「外部 AI 辅助」路径而非 qwen3-vl 视觉识别。qwen3-vl 仅在识别单行文字、单字手写、简笔图片等非表格内容时可靠。
```

### 扩展原则

1. **三合一扩展**：每次新功能同时涉及 API 端点 + 业务逻辑 + 前端模板
2. **集中路由**：新端点统一追加到 routes.py 底部
3. **静默兼容**：负向测试应返回清晰错误而非 500
4. **级联清理**：删除主数据时同步清除所有关联数据
5. **skeleton 模式**：页面加载时先用骨架屏占位
6. **prompt+confirm**：本地环境用浏览器原生对话框

## v3 架构进化：按认知复杂度分层

### 触发条件

当教育应用出现以下问题时，考虑从 v2 进化到 v3 架构：

- Web 前端持有独立的出题逻辑（content_engine.py），与后端 cron 的生成逻辑重复
- 同一份知识点/错题数据在 Web 和后端各有一份副本，不一致
- 每道题批改都需要一次 API 调用（高延迟）
- Web 提交的错题不进入后端错题本
- **需要扩展多种学习模式（如从仅默写扩展到认读/填空/配对等），且不想碰现有 v2 代码**

### 核心原则

**按认知复杂度分层，而非按前后端物理边界分层：**

```
操作类型               认知密度    执行位置    示例
─────────────────────────────────────────────────────
LLM推理                🧠 极高     后端          试卷分析/知识点映射
知识图谱+算法          🧠 高       后端          针对性出题
统计分析              📊 中高      后端          掌握度加权计算
值比对                ➕ 极低      前端          批改（预置答案比对）
简单IO写入            📝 低       前端          错题追加
纯展示                👁️ 最低     前端          看板/日报渲染
```

### 架构模式

```
┌────────────────────────────────────────────────────┐
│  后端（知识引擎）                                     │
│  ─────────────                                       │
│  持有: 知识图谱 · 题型模板库 · 出题算法 · 状态文件        │
│                                                       │
│  🧠 generate_daily_practice() → practice_bundle.json   │
│     {questions: [...], answers: {id: str}, metadata}    │
│                                                       │
│  🧠 generate_exam() → exam_bundle.json                 │
│  🧠 update_mastery(results) → 掌握度+艾宾浩斯           │
└──────────────────┬───────────────────────────────────┘
                   │
          题目包（含完整答案）
                   │
┌──────────────────▼───────────────────────────────────┐
│  前端（Web交互层）                                      │
│  ─────────────                                        │
│  不持有: 知识图谱 · 出题算法 · 掌握度计算                 │
│                                                       │
│  ✅ 展示题目（questions_only，不含expected_answer）      │
│  ✅ 本地批改（预置答案→前端比对→零延迟）                   │
│  ✅ 错题写入（POST /practice/save-wrong → 单次写操作）   │
│  ✅ 看板/日报（从state.json只读查询）                    │
└──────────────────────────────────────────────────────┘
```

### 关键实现模式

#### 1. 题目包（Question Bundle）模式

```python
# 后端生成：含题目+答案+元数据
bundle = {
    "date": "2026-05-01",
    "questions": [...] ,           # 完整题目（含expected_answer，后端仅自己保留）
    "questions_only": [...],       # 前端初始加载用（不含expected_answer）
    "answers": {"q_id": "68"},     # 答案映射（前端本地批改用）
    "metadata": {"due_topics": [...]}
}

# API 设计
# GET /practice/today → {questions: [questions_only]}  （首次加载，无答案）
# 前端本地存储 answers 映射，用户答题时前端 JS 直接比对
# 无需每道题调后端 API
```

**关键细节**：`generate_daily_practice()` 生成完整 bundle → 写入 `practice_bundle.json` → `GET /practice/today` 只返回 `questions_only` → 前端把 `answers` 存本地用于批改。

#### 2. 三层难度递进

```python
def select_difficulty(mastery):
    if mastery < 40:  return "foundation"   # 基础级：小数字、单步
    if mastery < 70:  return "standard"     # 标准级：考试难度
    else:             return "extension"    # 拓展级：变式、多步、应用
```

每个知识点在 YAML 模板中为每个难度定义不同的参数范围（数字大小、复杂度）。

#### 3. 多模板题型库（YAML）

每个知识点 ≥ 3 种题型模板，覆盖基础计算、变式判断、应用场景：

```yaml
topics:
  - id: multiply-2digit-by-2digit
    name: 两位数×两位数
    templates:
      - id: vertical_calc
        name: 竖式计算
        generator: multiply_2digit
        difficulty_levels: [foundation, standard, extension]
        params:
          foundation: {a_range: [11, 30], b_range: [11, 20]}
          standard:  {a_range: [11, 99], b_range: [11, 99]}
      - id: compare
        name: 比较大小
        generator: multiply_compare
        difficulty_levels: [standard, extension]
      - id: fill_blank
        name: 填空推理
        generator: multiply_fill_blank
        difficulty_levels: [foundation, extension]
```

#### 4. 艾宾浩斯间隔复习

每个知识点有 `next_review_date`：
- 做错 → 重置到第 1 天
- 做对且 mastery < 60 → 3 天后复习
- 做对且 mastery ≥ 60 → 7 天后复习
- 每日扫描：`next_review_date <= today` → 加入今日练习

```python
days_until_review = 1
if new_mastery >= 80: days_until_review = 7
elif new_mastery >= 60: days_until_review = 3
kp["next_review_date"] = (today + timedelta(days=days_until_review)).isoformat()
```

#### 5. API 端点简化

v3 的核心 API 比 v2 更少：

| 方法 | 端点 | 用途 | 后端/前端 |
|:----|:-----|:-----|:---------|
| GET | `/practice/today` | 获取今日题目（不含答案） | 后端生成/前端展示 |
| POST | `/practice/submit` | 提交答案批改（读bundle预置答案） | 后端验证 |
| POST | `/practice/save-wrong` | 前端发现错题后写回 state.json | 前端触发 |
| POST | `/practice/save-result` | 提交完整批改结果更新掌握度 | 后端聚合 |
| GET | `/api/dashboard` | 只读查询知识点掌握度 | 后端/前端展示 |

### v3 迁移路径（v2 → v3）

| 步骤 | 改动 | 说明 |
|:----|:-----|:-----|
| ① | 新建 `question_engine.py` | 从 Web 和 cron 提取共用出题逻辑 |
| ② | 新建 `question_templates.yaml` | 多模板题型库，每知识点≥3模板 |
| ③ | `routes.py` 移除 `content_engine` 依赖 | 改为调用 `question_engine` 读取 bundle |
| ④ | 修复 `submit` 严重 bug | 读 bundle 固定答案，不再重新生成 |
| ⑤ | 新增 `save-wrong` / `save-result` 端点 | 前端写回 |
| ⑥ | 删除 `practice_results.json` / `cached_practice.json` | 不再需要 |
| ⑦ | `daily-practice.py` 改为调用 `question_engine` | cron 精简 53% 代码 |

### 适用场景

- 已有 v2 教育 Web 应用，Web 持有独立的出题/批改逻辑
- Web 和后端数据不一致（错题本不同步、知识点覆盖不全）
- 希望前端批改零延迟（预置答案，无需每次调后端）
- 需要增加按需考试生成、变式题、间隔复习等教育功能

### 不适用场景

- 单次使用的考试/问卷（不需要知识积累和间隔复习）
- 纯展示型教育系统（无出题需求）
- 已有完善的微服务架构，前后端职责已清晰分离

## English Vocabulary Learning Mode Design

### 适用场景

需要为已有的语言学习 Flask Web 应用（如英语单词默写系统）扩展多种学习形式，让学生能自主选择学习范围和模式。

### 词汇学习多维模型

语言词汇学习包含多个认知维度，从低到高排列：

| 维度 | 学习形式 | 认知负荷 | 批改复杂度 | 数据依赖 |
|:----|:---------|:--------:|:---------:|:---------|
| ① 认读 | 四选一（看英文选中文/看中文选英文） | 低 | 极低（预置答案） | 仅需中英对照 |
| ② 拼写 | 看中文写英文（默写） | 中 | 低（字符串比对+Levenshtein） | 仅需中英对照 |
| ③ 选词填空 | 从选项中选词填入句子空白处 | 中高 | 低（预置答案） | 需要例句数据 |
| ④ 听力拼写 | 播放读音后拼写 | 高 | 中（TTS + 字符串比对） | 需要 TTS 服务 |
| ⑤ 造句 | 给定单词造一个句子 | 高 | 高（模糊语义匹配/LLM辅助） | 仅需中英对照 |
| ⑥ 配对 | 左边英文右边中文，点击配对消除 | 低中 | 极低（预置答案） | 仅需中英对照 |

### 三维范围选择器

学生应能从三个维度组合筛选学习内容：

```
维度1: 按内容范围         维度2: 按掌握度        维度3: 按错误频率
┌────────────────────┐   ┌────────────────┐   ┌────────────────┐
│ ☑ M1U1 (5词)       │   │ ○ 待巩固 (5词)  │   │ ☑ 高频错 (3词) │
│ ☐ M1U2 (8词)       │   │ ● 学习中 (6词)  │   │ ○ 近期错 (7词) │
│ ☑ M1U3 (6词)       │   │ ○ 已掌握 (12词)  │   │ ○ 全部错题     │
│ ☐ M1U4 (4词)       │   │ ○ 全部          │   └────────────────┘
└────────────────────┘   └────────────────┘
```

**组合优先级规则**：多维度同时选中时取交集。前端显示"本次共 N 个词"预估。

**数据缺失降级**：如果某个维度没有数据（如无错题），对应选项应灰显并提示"暂无数据"。

### 干扰项策略

认读模式和选词填空的干扰项选取规则：

| 优先级 | 来源 | 说明 |
|:-----:|:-----|:------|
| 1 | 同批次同单元 | 最高难度，容易混淆 |
| 2 | 同年级同难度 | 中等难度 |
| 3 | 随机抽取 | 兜底 |

### 结果记录策略

| 策略 | 说明 | 适用场景 |
|:----|:------|:---------|
| A) 统一掌握度 | 所有模式结果写入同一套掌握度系统 | 科学准确，互相校正 |
| B) 独立存储 | 各模式结果互不干扰 | 安全，零风险 |

**推荐 A**：认读做对的词提升掌握度，选词填空做错的词进入错题本。使用加权移动平均避免单次大幅波动：
```python
old_mastery = kp.get("mastery", 50)
new_mastery = old_mastery * 0.7 + (accuracy * 100) * 0.3
```

### API 端点模式

```python
# 三层分离：范围选择 → 出题 → 批改

# 1. 获取可用内容列表
GET /v3/api/words/available
# → {batches: [{id, name, count, mastered_pct}], 
#    mastery_levels: [{id, label, count}],
#    error_levels: [{id, label, count}]}

# 2. 获取选定范围内的题目
GET /v3/api/practice/start?mode=recognition&batches=M1U1,M1U2&mastery=learning,new
# → {questions: [{id, type, prompt, options?}], total, estimated_minutes}

# 3. 提交结果
POST /v3/api/practice/submit
# {answers: [{id, answer}]}
# → {score, total, results: [{id, correct, user_answer, expected_answer}]}

# 4. 保存结果到掌握度
POST /v3/api/practice/save-result
# {results: [...], mode: "recognition"}
```

### 架构约定：v3 Blueprint 零破坏模式

当需要为现有应用添加重大功能时，创建 `v3/` 目录而不是修改已有代码：

```
~/edu-hub/english/
  ├── v1/          ← 原始单体（不动）
  ├── v2/          ← 上次扩展（不动）
  ├── v3/          ← 本次新增（新代码全在这里）
  │   ├── __init__.py     # Blueprint 'eng_v3', url_prefix='/v3'
  │   ├── question_engine.py  # 统一出题逻辑
  │   ├── range_selector.py   # 三维范围选择
  │   ├── matching.py         # 配对模式
  │   ├── fill_blank.py       # 选词填空
  │   ├── routes.py           # 8+ API 端点
  │   ├── static/style.css    # v3 独立样式
  │   └── templates/          # v3 页面模板
  ├── data/
  │   └── v3/                 # v3 独立数据
  ├── app.py                  # 加一行: register_blueprint(eng_v3)
```

**原则**：
- v3 只读 v1/v2 数据，不写
- v3 结果写入 `data/v3/` 或回写 state.json 的对应字段
- app.py 仅加一行 `register_blueprint`
- 零风险：v3 出错不影响 v1/v2 功能

## 关键配置

```python
# Flask启动（v2 Blueprint 已注册）
app.run(host='0.0.0.0', port=5000, debug=False)
# → 访问: http://192.168.3.102:5000

# app.py 需设置 secret_key 以支持 Flask session
app.secret_key = os.environ.get('FLASK_SECRET_KEY', 'default-secret-key')

# v2 健康检查
curl http://localhost:5000/v2/health
```

### 陷阱：入口变更导致用户认为旧功能丢失

**问题**：当 `/` 从重定向到 `/v2/`（家长管理）改为 `/v3/portal`（学生学习门户）后，用户会认为家长端的所有功能（导入内容、报告、管理学生）都消失了——尽管 `/v2/` 完整保留。

**解法**：在 v3 门户的 **导航栏** 和 **门户首页显眼位置** 都加上回到 `/v2/` 的链接。不要假设用户知道旧 URL 仍然有效。

```html
<!-- base.html 导航栏 -->
<a href="/v2/" class="nav-parent-link">⚙️ 家长管理</a>

<!-- portal.html 门户首页 -->
<a href="/v2/" class="portal-btn secondary">⚙️ 家长管理（导入内容/查看报告/管理学生）</a>
```

**原则**：零破坏不只是不动旧代码——还要确保旧功能的发现路径不因新入口而隐蔽。

### v3 实现陷阱（从实战总结）

1. **YAML 参数名与生成器函数签名不匹配** — 题型模板库（YAML）定义的参数名如 `range`、`b_range` 与 Python 生成器函数参数名（`range_vals`、`target_range`）可能不一致。解法：在 `generate_question()` 中维护一个 `alias_map` 做参数名适配：
   ```python
   alias_map = {
       "range": ["range_vals"],
       "a_range": ["a_range", "big_range"],
       "b_range": ["b_range", "small_range", "target_range"],
   }
   for k, v in params.items():
       if k in func_params:
           filtered_params[k] = v
       elif k in alias_map:
           for target in alias_map[k]:
               if target in func_params:
                   filtered_params[target] = v
                   break
   ```

2. **YAML 引用了代码中没有的生成器** — 题型模板库可能定义了 `multiply_error_find`、`area_compare` 等生成器，但代码中尚未实现。解法：在 `generate_question()` 中设置后备生成器，用同知识点的第一个可用生成器替代：
   ```python
   if generator_name not in GENERATORS:
       fallback_name = topic_def["templates"][0]["generator"]
       if fallback_name in GENERATORS:
           generator_name = fallback_name
   ```

3. **生成器函数使用 `range` 作为变量名会覆盖 Python 内置 `range()`** — 避免将 YAML 参数直接命名为 `range`，或在 Python 函数中重命名为 `range_vals`。

4. **`logger` 初始化不调用 `basicConfig`** — `question_engine.py` 作为被多个模块导入的库代码，`logger = logging.getLogger(__name__)` 后应当使用 `logger.addHandler(logging.NullHandler())` 而非 `logging.basicConfig(...)`，避免干扰调用方的日志配置。

5. **掌握度加权移动平均** — `update_mastery()` 中，新练习结果的权重建议设为 0.3（即新结果占 30%，旧掌握度占 70%），避免单次练习大幅波动：
   ```python
   old_mastery = kp.get("mastery", 50)
   new_mastery = old_mastery * 0.7 + (accuracy * 100) * 0.3
   ```

6. **错题变式生成** — `generate_variation()` 从最近 5 道错题中选取，用该错题知识点的第一个模板重新生成（换数字、不换题型）。加入 `_reason: "variation"` 标签便于前端区分显示。

7. **并发写入文件锁** — `state_manager.py` 和 `question_engine.py` 都可能写入 `edu-system-state.json`。使用 Python `filelock` 库（或 `fcntl`）保护写操作，避免 cron 与 Web 并发写入冲突。

8. **API 响应格式兼容** — 重构后 `GET /practice/today` 返回 `{questions: [...]}` 格式，但原有前端模板可能期望 `{calculations: [...], word_problems: [...]}` 旧格式。迁移期间需维持向后兼容：
   ```python
   calculations = [q for q in q_list if q.get("type") == "calculation"]
   word_problems = [q for q in q_list if q.get("type") == "word_problem"]
   return jsonify({"calculations": calculations, "word_problems": word_problems, ...})
   ```

9. **强制加入应用题知识点** — `get_due_topics()` 基于错题和掌握度选择知识点，可能只返回计算类知识点（学生最近只练了乘法/除法），导致每日练习没有应用题。修复：在生成每日练习时强制加入 `["speed-formula", "rectangle-area", ...]` 等应用题知识点。

10. **`questions_only` 与答案分离加载** — `load_today_bundle()` 返回完整 bundle（含答案），但 `GET /practice/today` API 只返回 `questions_only`（不含 `expected_answer`）。答案通过 `load_bundle_answers()` 在需要时加载，确保前端首次渲染时不会暴露答案。

## Multi-Tier LLM Grading Pattern

适用于教育应用中需要语义理解的判题场景（如中文翻译、简答题、作文评分等）。

### 架构

```
L0 快速路径: 字符串归一化后精确比较          → 0ms（覆盖 ~80% 输入）
L1 缓存路径: dict 缓存 (input_pair → bool) → 0ms（覆盖重复组合）
L2 LLM路径: 本地 Ollama qwen2.5:3b 语义判断 → ~2s（首次遇见的新组合）
L3 回退路径: 字符串归一化比较                → 0ms（LLM 不可用时）
```

### 何时使用

- 学生答案需要语义理解而非精确匹配
- 有明确的预期答案用于比对（是非判断，非开放式）
- 延迟容忍度在 2-3 秒以内
- 有本地 Ollama 可用（该模式专为无 GPU/低配场景优化）

### 关键设计

1. **前处理归一化**：在调用 LLM 前先做标点/空格/大小写归一化，大部分相同答案无需 LLM
2. **Memory-level 缓存**：dict 缓存（非 lru_cache），上限 1000 条，防止内存泄漏
3. **静默降级**：LLM 调用包裹在 try/except 中，任一步失败自动回退到 L3
4. **温度=0**：确保相同输入永远得到相同输出
5. **轻量模型**：qwen2.5:3b（1.5GB）足以胜任二分类判断

### 参考实现

参见 `references/practice-optimization-v37.md` →「Chinese Text Grading (en2cn) — Three-Tier LLM Approach (v3.7.1)」

## Help/Glossary Page Pattern

教育系统应向学生透明展示分类术语定义，避免学生混淆「已掌握」「高频错词」等内部概念。

### 实现

- 新增路由到 Blueprint：`/help/glossary` → template `glossary.html`
- 内容包含：掌握度等级、错误频次等级、题型判题规则、星星币系统说明
- 门户页面加链接（常放在星星币区域或导航栏）
- 模板复用 base.html，独立 CSS 不污染其他页面

### 版本历史

| 版本 | 日期 | 变更 |
|:----|:----|:-----|
| 3.7.0 | 2026-05-06 | 类别标识+规则提示系统；移除选择题型改为55/45配比；错题重练轮询穿插(多词轮询/单词连续)；同义词首字母填空(fill_first_letter)分题计分；cn==en回退提示 |
| 3.7.1 | 2026-05-06 | en2cn中文标点宽松判题；content.json数据质量审计(cn==en检测)；分类说明页(/help/glossary) |
| 3.0.0 | 2026-05-01 | v3 统一学生端门户（听默+练习双模块） |
| 2.0.0 | 2026-04-29 | v2 扩展（家长端+学生端双模态） |
| 1.0.0 | 2026-04-28 | v1 单体 Flask 应用 |

11. **en2cn 中文判题需去除标点** — 中文答案精确比较导致多一个句号/空格都被判定为错误。修复：`_grade_by_category()` 中 en2cn 类型需先去除中文全角标点和空白再比较，正则 `[。，、；：？！“”‘’（）【】《》——……·\s]`
12. **`content.json` 中 cn==en 数据污染** — 导入时可能出现 cn 与 en 相同的数据。这会触发 `get_fallback_prompt()` 显示「请输入单词 'xxx' 的正确拼写」这种无意义提示。必须修复数据源（补充正确的中文翻译），后备提示只是安全网不能替代数据修复。审计脚本：`for cat in words/phrases/sentences: if item.cn == item.en: report`
13. **同义词首字母填空的分题计分仅服务端可见** — `_grade_fill_first_letter()` 将部分得分存入 question 的 `_partial_score` 字段，但前端需独立计算每个输入框的对错。前端的处理方式：遍历 `fill-letter-input` 的 `data-full` 属性与用户输入逐一比对，独立变色（绿框/红框），不受后端返回的影响。

### 判题字段优先级陷阱

**问题**：`_v4_grade_question()` 默认使用 `question.get("word") or question.get("en")` 作为预期答案，但 en2cn 题型正确答案存在 `answer` 字段（中文释义），`word` 存的是英文单词，导致输入正确中文也被判错。

**根因**：系统中有多个字段承载「答案」含义——`word`、`en`、`answer`、`cn`、`correct_answer`——不同模块的生成逻辑和判题逻辑对这些字段的优先级理解不一致。

**修复原则**（后端 `_v4_grade_question` + 前端 `practice_session_v4.html`）：

```
判题时预期答案字段优先级:
  en2cn:   answer > cn > word > en
  choice:  correct_answer > word > en
  其他:    word > en > correct_answer
```

前端展示正确答案时遵循同样优先级：
```
  en2cn:   answer > cn > word > en
  其他:    word > en > correct_answer > answer
```

**布防措施**：
1. 在 `_v4_grade_question()` 中按 type 分流，en2cn 单独处理
2. 在 `practice_session_v4.html` "不知道" 按钮展示时同样分流
3. 每次新增题型（type）时，必须同时更新判题和前端展示两个函数的字段优先级

**多空题分隔符规范**：语法填空/句型转换题中多个空白使用**逗号（,）** 分隔（如 "Is, isn't"）。前端提示统一加「多个空用逗号(,)分隔」。

## 已知陷阱

1. **模板占位符在题干文本中被替换导致题目显示为完整句子（无空白）** — 在 `_render_question()` 中，`_fill_template()` 会替换题干文本、答案、选项中的所有占位符。如果模板的 `template` 字段使用了 `{noun_pl}` 等占位符，题干会显示为完整句子（如 "I have two dogs."），学生误以为这是翻译题。**修复**：题干用 `___`（下划线文字）占位，占位符只放在 `answer` 和 `choices` 字段。详见 `references/template-question-generation-pitfalls.md`。

1. **飞书网页应用需要公网HTTPS才能发布上线**，家庭局域网开发调试使用IP地址即可
2. **飞书版本 V7.4 以下**的网页应用跳转新页面会跳到外部浏览器
3. **Flask Blueprint 名称冲突**：如果 `register_blueprint` 调用两次会报 `ValueError: 'eng_v2' is already registered`。确保只在 `app.py` 中注册一次
4. **Session 需要 secret_key**：Flask session 存储需要 `app.secret_key`，否则 `session['role'] = 'parent'` 会报错 `The session is unavailable because no secret key was set`
5. **Ollama thinking 字段**：qwen3.5 可能把内容放在 `result['thinking']` 而非 `result['response']`，ollama_bridge 需要兼容处理
6. **幂等迁移**：state.json 的 v1→v2 迁移必须可重复运行，通过 `version == "2.0"` 检查跳过重复
7. **多孩数据隔离**：所有数据查询必须强制携带 `student_id`，通过 `students.{student_id}` 维度隔离
9. **CSS 冲突**：v2 的 CSS 通过 `/v2/static/style.css` 独立加载，避免污染 v1 样式

## 统一设计系统模式（2026-05-14 新增）

当 Edu-Hub 扩展到多个学科模块（英语/数学/语文），需要统一页面设计风格时，使用此模式。

### 触发条件

- 各模块页面风格不统一（数学暖色、英语浅色、语文独立）
- 部分模块缺少全局导航（如数学原来没有顶栏）
- 返回导航不一致或缺失
- 同类功能（门户/练习/家长管理）在各模块呈现不同交互

### 架构

```
~/edu-hub/
├── static/
│   └── edu-design-system.css    ← 共享设计系统（设计令牌+通用组件）
├── math_sys/v2/
│   ├── static/style.css         ← 模块特有样式（仅覆写主题色+数学组件）
│   └── templates/v2/base.html   ← 统一顶栏结构
├── english/v3/
│   ├── static/style.css
│   └── templates/v3/base.html   ← 同结构
└── chinese_v3/
    ├── static/css/v3.css
    └── templates/c11/base.html  ← 同结构
```

### 核心设计

#### 1. 共享CSS + 模块CSS双层加载

```html
<!-- base.html 头部 -->
<!-- 1. 共享设计系统（所有模块公用）-->
<link rel="stylesheet" href="{{ url_for('static', filename='edu-design-system.css') }}">
<!-- 2. 模块特有样式 -->
<link rel="stylesheet" href="/math/static/style.css">
```

共享CSS提供：`.topbar`、`.nav-inner`、`.nav-home`、`.nav-brand`、`.nav-links`、`.container`、`.card`、`.btn`、`.tag`、`.hub-card`、`.stat-row`、`.progress-bar`、`.skeleton`、`.banner`、`.coins-badge` 等。

模块CSS保留/新增：MathJax渲染、成就/挑战/练习的特有组件等。

#### 2. CSS变量体系 + 模块主题色

```css
/* 共享CSS中定义 */
:root {
  --module-color: #4f46e5;       /* 默认indigo蓝，各模块覆写 */
  --bg: #f5f7fa;
  --card-bg: #ffffff;
  --text: #1e293b;
  --max-width: 800px;
  --radius: 12px;
  --font-family: -apple-system, ...;
}

/* 在body上通过class覆写主题色 */
body.module-math    { --module-color: #FF8C00; }
body.module-english { --module-color: #4f46e5; }
body.module-chinese  { --module-color: #059669; }
```

共享CSS的组件引用 `var(--module-color)`，一改全改。

#### 3. 统一 base.html 结构

```html
<body class="module-xxx">
  <nav class="topbar">      <!-- sticky顶栏 -->
    <div class="nav-inner">
      <div class="nav-left">
        <a href="{% block back_url %}/module/{% endblock %}" class="nav-home">←</a>
        <a href="..." class="nav-brand">📚 模块名</a>
      </div>
      {% block nav_links %}
      <div class="nav-links">
        <a href="...">功能1</a>
        <a href="...">功能2</a>
      </div>
      {% endblock %}
    </div>
  </nav>

  <main class="container">
    {% block content %}{% endblock %}
  </main>
</body>
```

#### 4. 返回导航红线（{% block back_url %}）

每个页面必须有返回路径。base.html提供 `nav-home` 默认返回模块门户，每页可通过 `{% block back_url %}` 覆盖：

```html
<!-- 学生练习页 -->
{% block back_url %}/math/{% endblock %}

<!-- 家长仪表盘 -->
{% block back_url %}/parent/{% endblock %}
```

**审计方法**：对每个模板文件检查是否有返回机制（extends base.html = 自动有，standalone = 需自检）。

### 实施步骤

1. **创建共享CSS** — `~/edu-hub/static/edu-design-system.css`，含设计令牌和通用组件
2. **重建模块base.html** — 统一顶栏+返回+导航，引用共享CSS
3. **清理模块CSS** — 移除与共享CSS冲突的全局样式（body/container/card样式移至共享）
4. **转换独立页面** — 原standalone HTML页面改为 extends base.html + 覆盖 back_url
5. **修复返回路径** — 统一所有 `location.href='/'` 为模块级路径
6. **审计** — 遍历38个模板，提取所有CSS类名，分类标准化（3阶段）
7. **批量替换** — 用sed批量替换按钮/进度条/统计格/标签类名
8. **验证** — grep计数确认每个替换生效，curl检查关键页面渲染

### 关键陷阱

- **CSS加载顺序**：共享CSS先加载，模块CSS后加载 → 模块可精确覆写
- **`{% block body %}` vs `{% block content %}`**：各模块可能使用不同的Jinja块名，统一前需确认
- **sed替换需验证**：`patch` 工具可能静默失败（返回ok但未修改），推荐先用find+grep测试匹配再执行sed
- **standalone页面残留**：可能有不继承base.html的独立页面，需逐一审计
- **内联样式迁移**：第二阶段需要将 ~120处内联样式迁移到共享CSS

### 参考文件

参见 `references/unified-design-system.md` 获取完整类名映射表和批量替换命令。
