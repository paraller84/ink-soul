---
name: content-factory-workflow
description: "C014 个人品牌内容工厂 — 周一选题→周二生成→周四/五发布的周级内容运营流水线。涵盖模块架构、cron编排、话题源管理、Feishu通知格式及常见故障恢复。"
version: 1.9.0
triggers:
triggers:
  - user asks "run content factory" / "run C014" / "内容工厂"
  - cron job runs weekly_cron.py (mon/tue/thu)
  - content factory pipeline shows "待定" for topics (topics file missing)
  - user asks about content planning or weekly topic selection
  - generated article content shows "..." / mock placeholders / "在实际项目中"
  - cron ran but user didn't receive Feishu notification
  - article_generator.py produced unusable short content
  - complex task completed (5+ tool calls, non-trivial outcome) — auto-evaluate for topic potential
  - user says "这个可以写出去" or "这个值得分享" or "这个做话题不错"
  - daily conversation (daily-conversation-practice) produces a structured insight — auto-evaluate for topic potential
  - session_search finds patterns in recent work that could make 2+ articles
  - user asks about content progress and calendar shows old/irrelevant topic names  # calendar cache staleness after rewrite
  - user asks to publish / push / 发布 / 推送 content to platforms
  - user asks about writing / creating drafts in 公众号/知乎/小红书 editors
  - published articles are stuck as "drafts only" and need platform-specific fixes
  - user asks about hot/viral article styles on a specific platform
  - user says "先思考一下平台的热点文章结构" or similar
  - first time writing for a new platform (WeChat/Zhihu/Xiaohongshu)
---

# 内容工厂工作流 (Content Factory)

## 概述

内容工厂是一个自动化周级内容运营流水线，每周按三阶段运行：

| 阶段 | 星期 | 命令 | 产出 |
|:----:|:----|:-----|:-----|
| 🔍 选题 | 周一 08:00 | `python3 weekly_cron.py monday` | 1篇深度+2篇短内容的选题方案 → Feishu |
| ✍️ 生成 | 周二 | `python3 weekly_cron.py tuesday` | 文章初稿（公众号/知乎/小红书）× 深度话题 |
| 🛡️ 审计 | 生成后立即 | 自动触发 | 真实性校验 + AI-isms 审计（avoid-ai-writing detect） |
| 🚀 发布 | 周四/周五 | `python3 weekly_cron.py thursday` | 发布提醒 & 状态汇总 |

## 代码结构

```
~/.hermes/scripts/content-factory/
├── weekly_cron.py           # 主编排入口（按参数调度阶段）
├── content_pool.py          # 话题池
├── topic_calendar.py        # 主题日历
├── article_generator.py     # 文章生成器
├── style_learner.py         # 风格学习器
├── review_manager.py        # 审核管理器
├── wechat-api-publisher.py       # ✅ 微信公众号 API 发布器 (Python, 2026-05-16)
│                                  #    基于官方草稿箱API，无需浏览器，无需Cookie
│                                  #    用法: --publish / --draft / --test / --batch-publish
│                                  #    依赖: Python 3 (无额外依赖)
├── wechat-cover-uploader.js      # ✅ 封面图生成+上传 (Node.js Playwright)
│                                  #    用法: --batch 目录 输出目录
│                                  #    依赖: Node.js playwright (hermes-agent内置)
├── publisher-worker.js           # 🔶 多平台发布管线 (Node.js Playwright, 仅知乎/小红书)
│                                  #    公众号已改用API (wechat-api-publisher.py)
│                                  #    用法: --cookie-setup / --publish / --check
│                                  #    依赖: Node.js playwright (hermes-agent 内置)
├── publisher_base.py             # ❌ 弃用
├── publisher_wechat.py           # ❌ 弃用
├── publisher_zhihu.py            # ❌ 弃用
├── publisher_xiaohongshu.py      # ❌ 弃用
├── config.yaml                   # 主配置文件
```

## 每日任务 → 话题发现机制（周五集中批处理）

> 你的日常就是最好的选题库。每次完成一个复杂任务，都可能是下一篇内容的种子。

### 行为规则：周中静默积累 → 周五集中汇总

**重要**：不每周中即时提示用户。发现好话题后，默默写入话题池，周五 cron 统一汇总呈现。

```
周一至周四：发现话题 → 静默入库（不打扰用户）
周五 09:00：cron 自动汇总本周新增话题 → 推送到内容市场部群
周末：用户思考挑选 → 下周一选题时自然被筛选
```

### 话题发现评估流程

每次完成**有信息量的任务**后（5+工具调用、非平凡结果），自动评估选题潜力：

```
① 这个任务有没有「反常识」或「可复现」的部分？
   有? → 可做选题，静默入库
   无? → 检查②

② 这个任务的数据/指标能不能「量化表达」？
   能? → 可做选题（有数据支撑的文章更容易传播）
   不能? → 检查③

③ 这个任务反映了什么普适问题？
   → 是「只有我们遇到的」还是「很多人都有的痛点」？
   → 很多人有 → 可做选题（共鸣型）

④ 选题属于哪种类型？
   ├─ 🔧 工程实践（架构决策、技术选型、踩坑复盘）
   ├─ ⚡ 效率工具（自动化管道、效率提升流程）
   ├─ 👁 行业观察（趋势分析、对比研究、方法思考）
   └─ 📚 教育实践（亲子教育、学习系统、知识管理）
```

### 好话题的判断标准

| 维度 | 好话题 | 放弃 |
|:----|:------|:----|
| **真实性** | 亲身经历，每段都能溯源 | 纯理论推演，无真实案例支撑 |
| **数据化** | 有数字（21.8K→3.6K，-83%）| 「大幅提升」「显著改善」 |
| **普适性** | 痛点很多人都有（记忆膨胀、质量保障）| 只有我们才有的特殊情况 |
| **方法论** | 能提炼出可复用的思考框架（三层存储、根因升级机制）| 一次性操作记录 |
| **结构感** | 有对比（之前vs之后/方案A vs 方案B）| 平铺直叙的流水账 |

### 如何静默入库

找到好话题后，直接写入话题池，**不询问用户**：

```python
# 用 execute_code 执行静默入库
from hermes_tools import terminal

cmd = '''
cd ~/.hermes/scripts/content-factory
python3 -c "
from content_pool import add_manual_topic
add_manual_topic(
    title="话题标题",
    summary="精炼摘要，含关键数据",
    topic_type="deep"
)
"
'''
terminal(cmd, timeout=30)
```

**重要**：入库后不要额外告诉用户「我存了一个话题」。周五 cron 会统一汇总。

### 与 Weekly Cron 的衔接

话题池中的每个条目带 `created_at` 时间戳：
- **周一**：`content_pool.get_candidates()` 按新鲜度+相关性评分，选出本周选题
- **周五**：cron 检查本周新增话题（created_at >= 本周一），整理为「本周话题发现」汇总

核心变化：**从「即兴发现→即时提示」变成「周中静默积累→周五统一呈现→周末思考→周一确认」**。

### 内容真实性原则（2026-05-11 用户明确要求）

**文章内容不能写成"我们的实践"，应该是"我们对一件事情的思考总结"。**

- 如果话题是我们**真实做过**的事（如 Hermes Agent 架构、教育系统搭建）→ 可以用"实战"口吻，但不虚构具体场景
- 如果话题是我们**没做过**的事（如企业流程自动化、保险行业实践）→ 必须用"观察""思考""分析"的口吻，不能冒充亲身经历
- 判断标准：文章中的每一段是否都能追溯到我们实际做过的事？如果有一段是"编的"，整篇定位就该改为分析/思考文章

**编写时的自查清单**：
```
□ 是否有任何段落是虚构的具体案例？（如"车险理赔Agent"→ ✗ 不可接受）
□ 是否有 "X个月我们做了Y" 但实际没做过的表述？
□ 是否明确区分了"我们观察到的" vs "我们做到的"？
□ 整篇文章定位是"实战分享"还是"思考分析"？前者只用于真实做过的事情
□ 是否有任何工具/服务推荐不是用户实际使用的？（如推荐"通义听悟/Kimi/飞书AI"但用户实际用Ollama+DeepSeek/飞书Task/Get笔记 → ✗ 不可接受）
```

### AI-isms 审计：生成后的自动审校环节（2026-05-12 新增）

每篇文章生成后（无论是自动生成还是手动生成），必须在发布前经过 **双重审计**：

```mermaid
flowchart LR
    A[文章初稿] --> B{真实性校验}
    B -->|通过| C{AI-isms 审计}
    B -->|未通过| D[定位修正/重写]
    D --> A
    C -->|通过| E[✅ 可发布]
    C -->|有AI-isms| F[降AI味重写]
    F --> G[二次审计]
    G --> E
```

#### 第一关：真实性校验（沿用现有自查清单）

即上方「内容真实性原则」中的自查清单。确保每段内容都能追溯到真实经历。

#### 第二关：AI-isms 审计（新增）

使用 `avoid-ai-writing` 技能对文章进行 **检测模式（detect）** 扫描：

在生成文章时自动加载 `avoid-ai-writing` 技能，执行以下流程：

```yaml
# AI-isms 审计清单
# 检测模式输出分组检查：
P0 — 可信度杀手（必须修复）：
  □ 有无 cutoff disclaimers（"As of my last update"）
  □ 有无聊天机器人语气（"I hope this helps!" "Great question!"）
  □ 有无空洞归属（"Experts believe" 无来源）

P1 — 明显AI味（建议修复）：
  □ 三级词汇表中的违规词（delve, leverage, harness, robust 等）
  □ 模板句式（"Whether you're X or Y"）
  □ "Let's"开头过渡句
  □ 同义替换综合征（连续3次换词说同一件事）
  □ 公式化开头（"In the rapidly evolving world of..."）

P2 — 风格润色（时间允许时修复）：
  □ 段落长度均匀
  □ 空泛结尾（"The future looks bright"）
  □ 动词过度替换（serves as, features, boasts → 直接用 is/has）
  □ "Moreover"/"Furthermore"/"Additionally" 过渡
```

**审计输出**：在文章的可发布状态通告中，附上审计结果摘要。

#### 第三关：二次审计（降AI味重写后的复检）

如果第二关发现问题并进行了重写，必须对重写版本再次运行 `avoid-ai-writing detect`，确认新模式也已通过审计。

---

**三种典型虚构模式（2026-05-12 实战发现的系统性漏洞）**：

| 虚构类型 | 示例 | 识别方法 | 如何修正 |
|:---------|:-----|:---------|:---------|
| **场景虚构** | "在保险行业尝试用AI Agent处理理赔"（用户没做过） | 问自己：这个案例有真实来源吗？ | 改为行业观察定位，去掉"我们"表述 |
| **工具虚构** | 推荐"通义听悟/Kimi/飞书AI"（用户实际用Ollama+DeepSeek/飞书Task/Get笔记） | 核对用户的真实工具链 | 逐个替换为用户实际使用的工具 |
| **身份虚构** | "作为一名AI架构师，我经历了…"（实际不是以这个身份经历的） | 检查叙述人称与用户真实角色的匹配度 | 改为"观察到…"/"思考…"中的洞察 |

### 话题多样性要求

周一选题时，话题池应覆盖多个方向供用户选择。当前已验证的分类：

| 分类 | 深度/短篇 | 举例 |
|:----|:---------:|:-----|
| 🔧 工程实践 | 深度为主 | 多Agent架构、本地模型成本、RAG工程 |
| ⚡ 效率工具 | 短篇为主 | 飞书待办、内容工厂、Get笔记 |
| 👁 行业观察 | 深度为主 | Agent生产化、技术选型决策 |
| 📚 教育实践 | 短篇为主 | 200天每日练习、语文手写识别 |

话题池至少保持8-10个有效话题，确保每周有2个以上的可选方向。

## 话题源管理

话题从以下数据源提取候选：

### 0️⃣ 日常对话话题（新增源）
每日15分钟主题对话（见 `daily-conversation-practice` skill）是高质量话题的天然来源。用户在对话中表达的真实经历、行业观察和概念碰撞，往往具备「有数据、有观点、有反常识」的好话题要素。

**接入方式**：对话锚定阶段识别到可扩展的洞察后，由 `daily-conversation-practice` 的流程自动调用 `content_pool.add_manual_topic()` 入库。无需额外操作。

**自动评估标准**：对话中用户表达的观点如果同时满足：
- 基于真实经历（有具体案例）
- 包含明确的判断/框架（不仅是感受）
- 具有普适性痛点（不仅是个人困境）
→ 即可判定为话题池候选，在对话结束时主动向用户推荐入库。

### 1️⃣ 手动话题文件
- **路径**: `~/.hermes/data/content-factory-topics.json`
- **格式**（含分类字段）:
  ```json
  {
    "topics": [
      {
        "title": "多Agent协作的架构演进",
        "summary": "基于真实系统的多Agent协作架构设计",
        "type": "deep",
        "category": "工程实践",
        "created_at": 1715380558
      },
      {
        "title": "用飞书API搭待办管理",
        "summary": "基于飞书Task v2 API的管理实践",
        "type": "short",
        "category": "效率工具",
        "created_at": 1715380559
      }
    ]
  }
  ```
- **增删改**: 调用 `content_pool.add_manual_topic(title, summary, topic_type, category)`

### 2️⃣ Get笔记缓存
- **路径**: `~/.hermes/cache/content-factory-getnotes.json`
- 由 Get笔记同步管道定期填充
- 格式: `{"notes": [{"title": "...", "content": "...", "type": "...", "created_at": ...}, ...]}`

### 3️⃣ 评分机制
- `freshness_weight`: 0.3（越新越高）
- `relevance_weight`: 0.4（含品牌关键词越多越高）
- `trend_weight`: 0.3（默认0.5）
- 品牌关键词: AI, 保险, 数据, 管理, 架构, 自动化, 系统, 大模型, agent, 智能, 数字化

## 数据流

```
话题源 (manual + getnotes)
     │
     ▼
content_pool.get_candidates()
     │  按评分排序，取top N
     ▼
topic_calendar.select_weekly_topics()
     │  分类为deep/short，选取1 deep + 2 short
     │  写入缓存: ~/.hermes/cache/content-factory-calendar.json
     │  生成Feishu Markdown
     ▼
weekly_cron.py monday
     │  读取缓存，调用 generate_feishu_message()
     │  输出 → Feishu 通知
     ▼
  [用户确认选题] → tuesday → 生成 → thursday → 发布
```

## Feishu 通知格式

`weekly_cron.py` 中的 `generate_feishu_message()` 函数为每个阶段生成Feishu友好的Markdown：

### 周一选题
```markdown
**C014 内容工厂 | 2026-05-11 Monday 选题建议**

**本周深度话题 (1/1):**
  {title}
  > {summary}

**推荐短内容 (2):**
  1. {title}
  2. {title}

请回复确认选题：
  /select deep=1, short=1,2  (选择深度话题1，短内容1和2)
  /skip 跳过本周
  /custom <你的选题>
```

## 已知Bug与修复

### Bug 1: 字段名 `topic_title` vs `title` （已修复 2026-05-11）

**症状**: Feishu通知中所有话题显示"待定"，即使 `content-factory-calendar.json` 中有正确数据。

**根因**: `weekly_cron.py` 中 `generate_feishu_message()` 函数使用 `deep.get('topic_title', '待定')` 和 `s.get('topic_title', '待定')` 读取话题标题，但 `topic_calendar.py` 写入的数据使用 `title` 字段（不是 `topic_title`）。

**修复**: 将 `topic_title` 替换为 `title`：
```python
# 修复前
msg += f"  {deep.get('topic_title', '待定')}\n"
# 修复后
msg += f"  {deep.get('title', '待定')}\n"
```

同时也修复了短内容计数字符串 `"推荐短内容 (2/{len(shorts)}):"` → `"推荐短内容 ({len(shorts)}):"`（写死了"2/"前缀）。

### Bug 2: 手动话题文件缺失（已修复 2026-05-11）

**症状**: 所有话题源返回空列表，Feishu显示"待定"。

**根因**: `content-factory-topics.json` 文件不存在，导致 `load_manual_topics()` 返回空列表。而 `calendar cache` 中仍有历史数据可用。

**修复**: 从 `content-factory-calendar.json` 缓存恢复话题到 `content-factory-topics.json`。

**陷阱**: 如果两个话题源都为空，`topic_calendar.select_weekly_topics()` 会返回错误 dict（含 `"error": "No candidates available from content pool"`），而 `generate_feishu_message()` 不处理此情况，仍按正常流程格式化，导致全部显示"待定"。

## 常见故障排查

### 🔴 症状: Feishu 显示 "待定"

**排查步骤**:
1. 检查 `content-factory-topics.json` 是否存在、非空
2. 检查 `content-factory-getnotes.json` 是否存在（可选源）
3. 检查 `content-factory-calendar.json` 是否包含有效数据
4. 如果缓存有数据但源文件缺失 → 从缓存恢复手动话题文件
5. 确认字段名：`weekly_cron.py` 中用 `title`，不是 `topic_title`

### 🔴 症状: 脚本正常输出但用户没收到飞书通知

**排查步骤**:
1. 检查 cron job 的 `deliver` 设置：`cronjob list` → 查看目标job的 `deliver` 字段
2. 如果 `deliver: local` → 输出只保存在本地，从不发送。修：`cronjob update --job-id <id> --deliver origin`
3. 创建新 cron job 时默认 `deliver: local`，与 Feishu 交互的 cron 必须手动改为 `origin`

**预防措施（构建 cron 时直接设置）**:
- 创建 cron job 时，如果该 cron 需要向飞书发送消息，记得在创建时或创建后立即设置 `deliver: origin`
- 周一选题 cron（8:02 AM）必须 `deliver=origin`，否则用户不会收到选题建议
- 用 `cronjob list` 确认所有需要外发的 cron 的 `deliver` 字段值

### 🔴 症状: 初稿包含虚构内容（非真实案例/数据）

**排查步骤**:
1. 检查文章是否包含我们没有亲身参与的场景描述（如保险理赔Agent、企业级流程自动化等）
2. 检查是否有「某某场景下我们做了…」但实际不存在的表述
3. 检查是否有虚构的具体数据（如"某保险公司实现了X%的降本"）

**修复策略（分三步）**:
1. **定位文章类型**：如果话题是真实做过的事 → 反查真实数据重写；如果话题不是真实做过的事 → 改为「行业观察与思考」定位，去掉所有"我们"开头的实战表述
2. **重写流程**：
   ```
   ┌─ 确认真实系统/数据存在？
   │   │ 是 → 提取真实架构图/数据/指标 → 用真实细节替换所有虚构内容
   │   │ 否 → 删除所有"我们做了X"的表述 → 改为"观察到X趋势"/"思考X方向"
   │   └─ 检查每段都能追溯到真实经历？不能再有编的段落
   ```
3. **验证**：打开文章全文，逐段问"这一段我能证明是我做的吗？" — 任何一段答不上来，就改文章定位
4. **飞书文档版本规则**：如果文章已经上传到飞书，不要在原文档上修改。保留原文档（V1不动），新建V2文档写真实版本，通过 `feishu-doc-versioning` skill 管理
5. **清理日历缓存**：文章重写后，`content-factory-calendar.json` 仍引用原话题名。更新 `feishu_markdown` 字段使其与重写后的实际标题一致，避免后续查询时出现"标题A（缓存）→ 标题B（实际）"的混淆。

### 🔴 症状: 缓存日历中的话题名与实际发布的文章标题不一致

**排查步骤**:
1. 检查 `~/.hermes/cache/content-factory-calendar.json` 中 `deep_topic.title` 和 `short_topics[*].title`
2. 与 `~/.hermes/data/generated-articles/current_week_articles.json` 中的实际文章标题对比
3. 如果因真实性重写导致标题变更 → 更新 calendar.json 的 `feishu_markdown` 字段
4. 不影响 cron 功能（只影响状态查询），但建议保持一致性

**修复方法**:
```python
import json
CAL = "~/.hermes/cache/content-factory-calendar.json"
with open(CAL) as f:
    cal = json.load(f)
cal["deep_topic"]["title"] = "新标题"
cal["short_topics"][0]["title"] = "新短标题1"
cal["short_topics"][1]["title"] = "新短标题2"
cal["feishu_markdown"] = cal["feishu_markdown"].replace("旧标题", "新标题")
with open(CAL, 'w') as f:
    json.dump(cal, f, ensure_ascii=False, indent=2)
```

### 🔴 症状: 生成的初稿显示"…"占位符或"（内容生成中…）"

**排查步骤**:
1. 检查 `article_generator.py` 的 `_call_llm()` 是否走入了 `_mock_llm_response()` 分支
2. 根因：`article_generator.py` 默认网关 `http://localhost:11434/api/chat`（Ollama），但发送 model 名 `deepseek-chat`，两端不匹配
3. 检查环境变量：`${HERMES_GATEWAY}` 和 `${DEEPSEEK_API_BASE}` 是否为空
4. 若 DeepSeek API 未配置，`article_generator` 会**静默降级**为 mock 占位符，且 cron 报告 **"ok"**（不报错！不会自动检测！）
5. 修复：设置正确的 API gateway 环境变量，或手动生成真实文章作为替代方案

**验证方法**（判断当前文章是真实内容还是占位符）：
```python
# 检查是否包含 mock 关键字
mock_markers = ["在实际项目中…", "内容生成中…", "（内容生成中…）", "## 核心观点", "1. 明确业务目标"]
with open(article_path) as f:
    content = f.read()
has_mock = any(m in content for m in mock_markers)
# 如果 has_mock → 走手动替换
```

### 🔴 症状: 脚本报错 ModuleNotFoundError

**排查步骤**:
1. `topic_calendar.py` 中已有 `sys.path.insert(0, _SCRIPT_DIR)` 处理同级模块导入
2. 如果从其他目录直接调用，确保 `PYTHONPATH` 包含脚本目录

### 🔴 症状: 周二/周四阶段找不到缓存

**排查步骤**:
1. 确认周一阶段成功运行（产生 `content-factory-calendar.json`）
2. 缓存路径: `~/.hermes/cache/content-factory-calendar.json`
3. 如果缓存丢失，直接运行 `python3 weekly_cron.py monday` 重新生成

## 飞书云盘存档

用户确认文章初稿后（/approve），应将文章上传到专用的飞书云盘文件夹，方便用户随时查阅、修改和发布。

### 文件夹结构

```
Hermes生成文件 (Otppfr9EelPIawdezL2csUXCnoh)
  └── 内容工厂出品 (OaHdfQM9flT1vudSxmFcHVRMnEg)  ← 每周文章存放处
        ├── 公众号_标题1
        ├── 知乎_标题2
        └── 小红书_标题3
```

- **文件夹 token**: `OaHdfQM9flT1vudSxmFcHVRMnEg`
- **别名**: `content-factory-output`（已注册到 feishu-tokens.json）
- **已共享给用户**（full_access）

### 存档流程（用户确认文章后执行）

```
用户确认选题 (/select 或 /approve)
  → Create Feishu doc in the folder
  → Write content via feishu-md-writer.py
  → 告知用户文件位置
```

**脚本模板**（Python）：

```python
import subprocess, json, tempfile
HOME = os.path.expanduser("~")
FOLDER_TOKEN = "OaHdfQM9flT1vudSxmFcHVRMnEg"
MD_WRITER = os.path.join(HOME, ".hermes", "scripts", "feishu-md-writer.py")

# Step 1: Create document
token = get_tenant_token()  # 用 FEISHU_APP_ID / FEISHU_APP_SECRET
doc_resp = api_call("POST", "/open-apis/docx/v1/documents", {
    "folder_token": FOLDER_TOKEN, "title": title
})
doc_token = doc_resp["data"]["document"]["document_id"]

# Step 2: Write content
md_content = f"# {title}\n\n{content}"
with tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False) as f:
    f.write(md_content); tmp = f.name
subprocess.run(["python3", MD_WRITER, doc_token, tmp], timeout=120)
os.unlink(tmp)
```

**注意**: 
- `feishu-md-writer.py` 接受**位置参数**（doc_token, md_file），不是 `--doc-token` 标志
- 创建文档后，用户可自主编辑、分享或导出为 PDF/Word
- 文件自动存在于「Hermes生成文件 → 内容工厂出品」目录下，用户可从飞书云盘直接访问

## 平台前研究：热点文章结构调研

每次为用户构思平台化文章前，应先进行平台热点文体调研，而非仅凭经验直接动笔。

### 触发条件

以下任一情况出现时，执行平台研究：

- 用户要求将同一主题发布到多个平台
- 用户说"你先思考一下xx平台的流行文体结构"或类似表述
- 距离上次同赛道平台研究已超过 30 天（内容趋势会变化）
- 面向一个从未写过的新平台首次创作

### 研究方法（并行 delegate_task）

使用 Hermes 的 `delegate_task` 工具，同时启动 3 个独立搜索任务，每个针对一个平台：

```json
[
  {
    "goal": "搜索微信公众号「{赛道关键词}」赛道爆款文章结构特征",
    "context": "目标：理解公众号上什么标题结构最容易出爆款，开头用什么叙事抓住读者，正文段落切分方式，结尾钩子。侧重职场/效率人群。",
    "toolsets": ["web"]
  },
  {
    "goal": "搜索知乎「{赛道关键词}」话题高赞回答的叙事结构",
    "context": "目标：理解知乎长文的爆款结构——反常识断言开头的、结构严密的、有操作步骤的。分析标题/开头/正文/结尾模式。",
    "toolsets": ["web"]
  },
  {
    "goal": "搜索小红书「{赛道关键词}」爆款笔记特征",
    "context": "目标：分析封面设计、标题数字特征、正文篇幅、排版方式、emoji使用模式。侧重职场/自我管理话题。",
    "toolsets": ["web"]
  }
]
```

**三个任务并行执行**，结果汇总后交叉对比。

### 研究成果存档

每个主题的研究成果应当：

1. 提取各平台的核心结构模板（开头→正文→结尾）
2. 制作跨平台对比矩阵（字数/口吻/结构/图片/结尾）
3. 标注你最可能模仿的 2-3 个成功案例
4. 保存到 `content-factory-workflow` 的 `references/` 目录下，文件命名：`platform-research-{topic}-{YYYYMMDD}.md`

详细研究成果可参考 `references/platform-hot-article-research-guide.md`（通用版）或按话题定制的独立文件。

---

## 文章配图生成方案

内容工厂的文章在各平台发布需要配图（封面图、文中插图）。当前已验证三种方案，按推荐优先级排列：

### ✅ 方案 A（当前标准路径 · 2026-05-17 用户确认）：Agent 写提示词 → 用户手动即梦生成

**用户明确选择的标准路径**（而非继续尝试自动化出图方案）。Agent 负责精确设计提示词（含模型建议、比例、风格关键词、场景描述），用户登录即梦AI网页端手动生成并保存，发回给 Agent 嵌入文章。

#### 使用场景
- 所有需配图的文章发布前
- 三平台同时发布时，每个平台独立配图（风格不同）

#### 工作流
```
文章定稿
  ↓
Agent 分析文章核心视觉要素（数据对比/结构图/场景示意/隐喻）
  ↓
Agent 为每个平台设计 1-3 个配图提示词（封面+正文）
  ↓
Agent 整合为独立配图需求文档（HTML格式，含使用说明+比例+模型建议）
  ↓
上传配图需求文档至飞书云盘文章所在文件夹 → 发送链接给用户
  ↓
用户打开文档 → 复制提示词到 jimeng.jianying.com「文生图」→ 生成
  ↓
用户将满意的图片发送到对话框（无需说明是哪张，Agent自行识图匹配）
  ↓
Agent 将图片嵌入文章对应位置
```

#### 配图提示词结构模板

每条提示词应包含以下信息：

```
【使用说明】建议模型 / 比例 / 模型版本
【提示词正文】详细中文场景描述 + 视觉风格关键词
```

#### 三平台视觉风格规范

| 平台 | 视觉基调 | 主色 | 推荐比例 | 视觉隐喻偏好 |
|:----|:---------|:----|:---------|:------------|
| **微信公众号** | 暗色科技风·深度 | 青色/蓝色 | 16:9 横图 / 3:2 | 数据管道、发光线条、粒子流动、赛博朋克光影、IMAX质感 |
| **知乎** | 暖色思辨风·理性 | 琥珀色/金色 | 1:1 方图 | 几何分形、对称构图、工程师图纸、第一性原理图、抽象结构 |
| **小红书** | 粉紫实用风·干货 | 粉红/粉紫渐变 | 3:4 竖图 | 手写笔记、ins风排版、大数字对比、圆角卡片、柔和渐变背景 |

#### 配图类型与生成要点

| 配图类型 | 典型用途 | 提示词要点 |
|:---------|:---------|:-----------|
| **对比图** | 之前vs之后、旧vs新 | 左右二分：一侧杂乱/红色，一侧整洁/绿色，中间有过渡光束 |
| **管道图** | 数据流、分流模型 | 多段式水平结构（如三段管道+箭头），发光节点标注层次 |
| **场景图** | 通勤/办公/睡前 | 三张卡片式排列，各带标志性元素（地铁/台灯/月亮），色彩渐变 |
| **数字图** | 核心数据突出 | 大号数字 + 简要说明文字（如"20+ → 5-7"），数字发光强调 |
| **抽象图** | 方法论/哲学概念 | 几何抽象 + 分形元素，不具象但传递思维层次感 |

#### 配图需求文档格式规范

当为多平台系列文章设计配图时，将提示词整合为独立配图需求文档（而非散落在对话中）：

- **格式**：HTML 文档，暗色科技风主题（见 `html-info-doc-production` skill），卡片式布局
- **内容要求**：
  - 文档头部：版本号、日期、配图总数、平台分布统计
  - 每个平台独立区块：平台名 + 视觉风格关键词 + 色系标识
  - 每张配图独立卡片：封面/正文标注、比例、模型建议、提示词正文（蓝色/金/粉三色标识框）、使用提示
  - 文档尾部：执行清单（建议优先级排序）
- **存放**：上传到飞书云盘文章所在文件夹，分享给用户
- **原因**：一次给用户完整的图库概览，用户可按优先级逐张生成，无需反复沟通

#### 即梦AI操作要点
- 同一个提示词多刷几次（即梦每次生成质量波动大）
- 模型版本优先选 4.0/4.5，5.0 对中文理解更好但风格偏写实
- 生成后检查是否有破坏性文本渲染（即梦对中文文字渲染不稳定）
- 封面图优先，正文配图可以在封面确认后再逐步生成

#### 提示词示例库
见 `references/jimeng-ai-prompt-examples.md` — 包含本系列文章的完整提示词示例

---

### 方案 B（备选）：即梦AI CLI 出图

使用 **Jimeng AI (即梦/字节跳动) CLI 工具** `dreamina` 命令行出图。

**当前状态：Standard 账号无法使用 CLI 出图（需 Maestro VIP）**，详见下方章节。

### 方案 C（备选）：Hermes 内置 image_gen 插件

Hermes Agent 内置了 `openai` / `openai-codex` / `xai` 三个图片生成插件，但均需要对应的 API Key 环境变量（当前未配置）。

---

### 即梦AI CLI 工具详情

#### 工具状态

| 项目 | 状态 |
|:----|:----:|
| 安装 | `curl -fsSL https://jimeng.jianying.com/cli \| bash` → `~/.local/bin/dreamina` |
| 登录 | headless OAuth 模式（`dreamina login --headless` → 用户浏览器确认） |
| 出图 | `dreamina text2image --prompt="..." --ratio=16:9 --poll 30` |
| 账号要求 | **Maestro VIP**（Standard 账号无法使用 CLI 出图） |

#### 登录架构（反直觉但重要）

```mermaid
flowchart LR
    A[CLI dreamina login --headless] --> B[OAuth链接+设备码]
    B --> C[用户Windows浏览器打开确认]
    C --> D[Token持久化在本地]
    D --> E[后续调用复用Token]
    
    F[网页登录] -.->|window.open弹窗| G[无头浏览器无法捕获]
    G -.->|改用CLI方案| A
    
    注意：CLI的OAuth登录和浏览器登录是独立的session
```

**关键发现**：Jimeng AI 网页端的登录使用 `window.open()` 弹出新窗口，无头浏览器（Playwright/Puppeteer）无法捕获此弹窗。因此 **不要尝试通过浏览器自动化登录 Jimeng AI**。改用 CLI 的 headless 登录模式。

#### CLI 常用命令

```bash
# 检查登录状态和账号信息
dreamina user_credit
# → 输出: { total_credit, user_id, user_name, vip_level }

# 生成图片
dreamina text2image \
  --prompt "极简科技风, 扁平化插画, 数据管道概念, 蓝色色调" \
  --ratio 16:9 \
  --poll 30 \
  --model_version 5.0

# 支持的模型版本: 3.0, 3.1, 4.0, 4.1, 4.5, 4.6, 5.0
# 支持的比例: 21:9, 16:9, 3:2, 4:3, 1:1, 3:4, 2:3, 9:16
# 分辨率: 3.0/3.1 → 1k或2k; 4.0以上 → 2k或4k

# 列出任务（查询历史）
dreamina list_task

# 查询指定任务结果
dreamina query_result --task_id <id>
```

#### 账号升级路径

Standard 账号（网页端可用）→ 升级 Maestro VIP（CLI 出图可用）→ 升级入口在 jimeng.jianying.com 网页端登录后「會員中心」。

**检测命令**：`dreamina user_credit` 的输出 `vip_level` 字段显示当前等级。

详细 CLI 笔记和登录踩坑：见 `references/jimeng-ai-cli-notes.md`

---

## 三平台多载体文章生成技术

当同一话题需要同时面向公众号、知乎、小红书三个平台产出内容时，采用以下适配模式：

### 自动发布管线 (2026-05-17 新增)

使用 Node.js Playwright 脚本 `publisher-worker.js` 自动创建草稿/发布：

```bash
cd ~/.hermes/scripts/content-factory

# 首次运行：Cookie 设置（需在 Windows Chrome 导出各平台 cookie）
node publisher-worker.js --cookie-setup

# 检查 cookie 状态
node publisher-worker.js --check

# 发布篇01到三平台
node publisher-worker.js --publish all --article 01

# 单独发布某平台
node publisher-worker.js --publish zhihu --article 03
```

**发布能力矩阵（旧: Cookie方案）**：  
- ✅ 知乎：自动创建草稿 + 自动发布  
- ✅ 微信公众号：自动创建草稿 → 需在 mp.weixin.qq.com 手动发布  
- ✅ 小红书：自动创建草稿 → 需在 creator.xiaohongshu.com 手动发布  

**依赖**：Node.js Playwright（从 hermes-agent 的 node_modules 解析）  
**注意**：Cookie 有效期 3-30 天不等，过期需重新导出  

### 微信公众号 API 发布方案 (2026-05-16 新增，推荐替代Cookie方案)

**使用官方草稿箱 API，无需浏览器，无需 Cookie，无需手动操作。**

```bash
# 设置凭证（一次性）
python3 ~/.hermes/scripts/content-factory/wechat-api-publisher.py \
  --setup <appid> <appsecret>

# 创建并发布单篇文章
python3 ~/.hermes/scripts/content-factory/wechat-api-publisher.py \
  --publish /path/to/article_clean.html \
  --cover <永久素材media_id> \
  --title "文章标题"

# 批量发布4篇系列文章
python3 ~/.hermes/scripts/content-factory/wechat-api-publisher.py \
  --batch-publish
```

**依赖**：
- Python 3 标准库（urllib, json, re）— 零额外依赖
- 微信公众平台 appid + appsecret（从 mp.weixin.qq.com → 设置与开发 → 基本配置 获取）
- 封面图需先上传为永久素材（使用 `wechat-cover-uploader.js` 生成并上传，或手动上传）

**注意事项**：
- `thumb_media_id`（封面图）是 **必填参数**，不可省略
- 摘要限制 120 字符以内，超长会报错
- HTML 内容中 `<style>` 标签会被丢弃，必须使用内联 style
- 封面图不适用旧 publisher-worker.js 的 Cookie 方式管理

**详细 API 技术笔记**：见 `references/wechat-api-publishing.md`  

### 三平台文章结构对比

| 维度 | 公众号（6-8分钟阅读） | 知乎（深阅读约10分钟） | 小红书（3分钟浏览） |
|------|----------------------|----------------------|---------------------|
| **叙事风格** | 第一人称故事+感悟，有温度 | 理性分析+设计原理，有深度 | 短平快+痛点共鸣，有情绪 |
| **信息密度** | 中（观点+个人经历+金句） | 高（框架图+设计决策+技术论证） | 低（痛点→步骤→数据→收藏） |
| **标题策略** | 悬念+数字+"我是怎么" | 核心观点+设计思路 | 结果导向+"只需X步" |
| **开头模式** | 代入感场景 → "大多数人..." | 定义问题 → 第一性原理 → 挑战主流观点 | 对比之前vs之后 → "你是不是也..." |
| **正文结构** | 故事线：问题→尝试→失败→突破→感悟 | 论点线：问题→方案→原理→决策→数据 | 清单线：痛点→3-5个步骤→关键认知→数据 |
| **引语/金句** | 用 `.quote` 框独立呈现，1-2句 | 用 `.quote` 框呈现核心观点，贯穿论证 | 用 `.insight` 强调关键认知，醒目位置 |
| **数据呈现** | 嵌入式描述 + 大数字突出 | 表格/框架图对比 | 大数字卡片 + 简短标注 |
| **结尾** | 感悟+下篇预告 | 系统方法论总结+系列回顾 | 收藏引导+系列索引 |

### 生成工作流

```
话题确认
  ↓
① 并行平台研究（delegate_task × 3 → 产出跨平台对比矩阵）
  ↓
② 提炼核心框架（3-5句话概括共同的方法论骨架）
  ↓
③ 独立起草各版本（不互相推导，各自从核心框架出发）
```

**核心框架先行**：在写任何一个版本之前，先用 3-5 句话提炼出共同的方法论骨架。例如本次三层分流体系的核心框架：
> 信息入口（录音+邮件）→ 三层分流（我关注/他关注/场景切分）→ 自动归位
> 核心数据：待办从 20+ 到 5-7 项，委派完成率+65%

这个框架是所有版本的共同原点。不要从一个平台版本"推导"到另一个版本——那样会让第二个版本读起来像第一个版本的变体。

**平台差异判别树**（何时推导 vs 独立起草）：

```
平台风格是否属于以下类别？
├─ 叙事型（公众号：故事+感悟）→ 独立起草
├─ 分析型（知乎：原理+决策）→ 独立起草
└─ 清单型（小红书：痛点+步骤）→ 独立起草
如果三平台的风格属于不同类别 → 独立起草各版本
如果风格相同或接近（如都是同类的清单列表）→ 可以从一个版本推导
```

**要点**：
- **核心框架先行**——在动笔写任何版本前，先锁定 3-5 句核心方法论+3-5 个关键数据，这是所有版本的共同基因
- **独立起草意味着**：公众号版本走叙事线（我不需要先写好公众号才能写知乎），知乎版本直接走分析线，小红书版本直接走清单线
- **三者共享同一套核心数据和方法论框架**，但第一句话就已经是为特定平台量身定做的
- 2026-05-17 实践验证：任务体系三层分流话题，三版用独立起草→开头钩子完全不重样（公众号"假若你每天…"，知乎"大多数任务系统的根本缺陷"，小红书"你是不是也这样？"），但核心框架一致

### 实践参考：个人任务闭环体系系列（2026-05-16）

此系列4篇×3平台=12篇HTML的生产过程可作参考：
- 篇03（分类过滤三篇）被评为核心篇目，公众号版10489字节，知乎版9502字节，小红书版5812字节
- 篇01（总纲篇）的小红书版压缩到8169字节，大量使用emoji+步骤卡替代段落
- 所有文件使用内联CSS暗色科技风，每篇独立完整HTML

### 配图策略：每篇文章三套配图

多平台文章生成时，配图应独立设计而非复用同一张图：

| 平台 | 封面图风格关键词 | 比例 | 配图数量 |
|:----|:----------------|:----:|:--------:|
| 公众号 | 暗色科技·数据管道·发光 | 16:9 或 3:2 | 封面1张+正文2-3张 |
| 知乎 | 暖色思辨·几何抽象·分形 | 1:1 | 封面1张+正文2-3张 |
| 小红书 | 粉紫实用·大数字·手写感 | 3:4 | 封面1张+正文2-3张 |

> 提示词写法详见「文章配图生成方案 → 方案A」章节和 `references/jimeng-ai-prompt-examples.md`。

配图生成在文章初稿完成后、发布前执行，工作流：
```
文章定稿 → 配图提示词设计 → 用户手动即梦生成 → 图片发回 → 嵌入文章 → 发布
```

### 参考文件

- `references/multi-platform-writing-patterns.md` — 跨平台差异化写作模式
- `references/publishing-pipeline-workflow.md` — 发布管线实操指南（内容格式转换、Node.js Playwright、cookie管理、系列发布节奏）

```yaml
# ~/.hermes/scripts/content-factory/config.yaml
publishing:
  deep_articles_per_week: 1
  short_posts_per_week: 2
  publish_day: friday

content_sources:
  getnotes:
    enabled: true
    cache: "~/.hermes/cache/content-factory-getnotes.json"
  manual:
    enabled: true
    topics_file: "~/.hermes/data/content-factory-topics.json"

content_pool:
  max_candidates: 10
  scoring:
    freshness_weight: 0.3
    relevance_weight: 0.4
    trend_weight: 0.3

article_generation:
  model: "deepseek"

notifications:
  platform: "feishu"
  chat_id: "oc_575e28286dba895bd619f911399b7d01"
```

## 发布阶段实操工作流

> 内容工厂的「周四/周五发布」阶段不只是发提醒——需要真实将文章投递到三个平台。

### 当前发布管线状态

| 组件 | 状态 | 说明 |
|:----|:----:|:------|
| `publisher_base.py` | ❌ 弃用 | 缺 `_ensure_browser()` 方法，Python包不可安装 |
| `publisher_wechat/zhihu/xiaohongshu.py` | ❌ 弃用 | 依赖已废弃的基类 |
| `wechat-api-publisher.py` | **✅ 工作中** | 公众号 API 发布器，Python实现，零外部依赖 |
| `wechat-cover-uploader.js` | **✅ 工作中** | 封面图生成+上传到微信永久素材 |
| `publisher-worker.js` | **✅ 工作中** | 仅用于 知乎/小红书 Cookie 发布（公众号已改用API） |
| Puppeteer/Playwright | 无问题 | Node.js版自带，Hermes agent node_modules中 |
| headless模式 | ⚠️ 受限 | WSL无X Server → headless only，首次登录需用户桌面操作 |
| Python urllib | 无问题 | Python标准库，零配置 |

### 发布管线（`publisher-worker.js` 实操工作流）

完整发布脚本位于 `~/.hermes/scripts/content-factory/publisher-worker.js`。Node.js Playwright实现，支持**两阶段工作流**：

#### 阶段一：Cookie设置（用户操作一次，约6分钟）

WSL无X Server，headless模式下无法扫码登录。首次需用户在**Windows桌面Chrome**手动登录后导出cookie。

**用户操作流程**（每个平台重复）：
```
① 打开平台网址、扫码登录
② F12 → Application → Cookies → 全选(Ctrl+A) → 复制(Ctrl+C)
③ 在终端运行 --cookie-setup，粘贴cookie JSON
```

**支持的三个平台**：
- 微信: `mp.weixin.qq.com`
- 知乎: `zhihu.com`
- 小红书: `creator.xiaohongshu.com`

```bash
# 启动Cookie设置向导（交互式粘贴）
cd ~/.hermes/scripts/content-factory && \
node publisher-worker.js --cookie-setup

# 验证Cookie是否有效
node publisher-worker.js --check
```

Cookie持久化在 `~/.hermes/cache/{platform}_cookies.json`，有效期约7-30天。

#### 阶段二：自动创建草稿

Cookie就绪后，一键创建草稿：

```bash
node publisher-worker.js --publish all --article 01    # 全部平台，篇01
node publisher-worker.js --publish wechat --article 03 # 单独发布
```

知乎 → 自动发布；微信/小红书 → 自动存草稿，用户手动点发布。

#### 内容格式转换（从HTML到平台可读文本）

内置提取逻辑去掉CSS/JS/标签/实体，合并多余空行。各平台差异：公众号用iframe编辑器（frame.locator），知乎/小红书用contenteditable。

#### 已知约束

| 约束 | 缓解措施 |
|:----|:---------|
| WSL headless无法扫码 | 用户桌面登录后导出cookie（仅知乎/小红书）；公众号已改用API无需Cookie |
| 微信iframe编辑器 | ❌ 已弃用 — 改用 Python API 发布器直接调用草稿箱API |
| 小红书反爬严格 | 只创建草稿，手动发布 |
| Python playwright不可安装 | 改用Node.js Playwright；或对公众号改用Python API方案 |
| require声明顺序 | 将 `os`/`path`/`fs` 放在文件顶部 |
| Node.js 中文编码问题 | Python `json.dumps(ensure_ascii=False)` 稳定处理中文；API调用避开Node.js |

## 陷阱与注意事项

1. **不要在周一之前删除手动话题文件** — `weekly_cron.py monday` 依赖此文件，缺失会导致空选题
2. **话题分类是启发式的** — `classify_topic()` 基于关键词+评分，可能不准确。中高评分长内容会被判定为 `deep`
3. **周二阶段依赖周一缓存的日历文件** — 如果周一失败，周二直接跳到生成会报错
4. **Feishu消息是纯文本Markdown格式** — 不是富文本卡片，飞书会渲染Markdown基本语法
5. **脚本在cron中运行时没有交互式确认环节** — 所有输出自动发送到Feishu，用户需手动回复确认（/select或/skip）
11. **工作流中的每个阶段都必须有对应的 cron 调度** — 文档中写了"周二生成"阶段（`python3 weekly_cron.py tuesday`），但从未创建对应的 cron 任务。周一→周二→周五的链式依赖中，周二环节缺失导致周五发布空跑。2026-05-14 新增 cron `54c117b76344`。验证方式：`cronjob list | grep C014` 应看到 3 个 cron：
    - `109e836fd119` — 周一 08:00 选题
    - `54c117b76344` — 周二 08:00 生成（2026-05-14 新增）
    - `5772dbd93675` — 周五 09:00 发布提醒（已加缺稿检测）
7. **缓存文件写入时间戳** — 如果周一在凌晨运行，缓存的 `generated_at` 和 `publish_date` 会使用运行时的datetime，不影响功能但注意时间
8. **Cron deliver 默认为 `local`** — Hermes 创建 cron job 时 `deliver` 默认是 `local`。如果 cron 需要向用户发送飞书通知（选题建议、文章初稿等），必须手动改 `deliver: origin`。排查时先用 `cronjob list` 检查。

9. **文章生成器静默降级为 Mock 占位符** — `article_generator.py` 的 `_call_llm()` 在 LLM 网关不可达时无提示地回退到 `_mock_llm_response()`，产出类似「在实际项目中…」「## 核心观点」「1. 明确业务目标」等占位符文本。cron 状态显示 "ok"，不报错。必须主动检查文章内容是否有 mock 标记来发现此问题。

10. **文章生成器网关配置不匹配** — `article_generator.py` 硬编码默认网关 `http://localhost:11434/api/chat`（Ollama）但发送 model `deepseek-chat`。DeepSeek API 环境变量 `${HERMES_GATEWAY}` 和 `${DEEPSEEK_API_BASE}` 均未设置时，所有文章都是 mock 占位符。修复：设置正确的 API base/key，或手动生成真实文章替代。
