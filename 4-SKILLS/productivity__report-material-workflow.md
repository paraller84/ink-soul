---
name: report-material-workflow
description: "从需求挖掘到交付复盘的7阶段完整工作流，适用于制作高质量PPT/文档/汇报材料。包含角色定义、提问脚本、模板和交付物清单。"
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [productivity, report, presentation, workflow]
    related_skills: [plan, writing-plans]
---

# 高质量汇报材料制作工作流

## 适用场景
当用户需要制作**正式的汇报材料**（向上汇报、客户提案、年度总结、项目评审、产品发布等）时，使用此工作流来确保输出高质量、有针对性的成果。

## 七阶段总览

```
需求挖掘  →  受众分析  →  内容架构  →  内容创作  →  视觉设计  →  审核优化  →  交付复盘
 需求分析     用户研究     内容架构     专业撰稿+     视觉设计师    质量评审官    项目经理
 顾问         专家         师           数据分析师
```

## 阶段一：需求挖掘与信息收集

**角色**：需求分析顾问 (Requirements Analyst)

### 1.1 结构化三轮回合

#### 第一轮：全景扫描（SPIN法则）
使用5个模块覆盖：情境层(Situation) → 受众层(Audience) → 目标层(Goal) → 内容层(Content) → 约束层(Constraints)

关键提问模板：
```
场景:
  Q: "请描述这份材料用在什么具体场合？"
  Q: "汇报时长多少？最终形式是什么？"
  
受众:
  Q: "核心受众是谁？谁有决策权？知识水平如何？"
  Q: "他们最关心什么？最可能质疑什么？"

目标:
  Q: "用一句话概括最终目标？"
  Q: "希望受众在听完后做什么？"
  Q: "怎么判断成功？"

内容:
  Q: "核心信息可以用一句话概括吗？"
  Q: "哪些信息必须包含？哪些可删减？"

约束:
  Q: "技术/安全/语言/预算方面有什么硬性限制？"
```

#### 第二轮：深度聚焦（5Why + 冰山模型）
对关键模糊点进行5Why递进追问。隐性需求探测：
- "你担心什么？"
- "如果做得不好，最大风险是什么？"
- "除了材料本身，还有什么重要考量？"

#### 第三轮：确认校准（反向复述）
结构化复述理解，请用户纠偏

### 1.2 源材料收集
使用源材料需求清单收集：历史材料、数据支撑、参考素材、辅助材料

### 1.3 需求确认书（阶段一核心交付物）
必须包含：基本信息、受众分析、核心目标、核心信息、内容要求、视觉风格、关键约束、源材料清单。**用户签字确认后方可进入下一阶段**

## 阶段二：受众分析与场景建模

**角色**：用户研究专家

### 2.1 受众画像卡
| 维度 | 内容 |
|------|------|
| 身份/角色 | |
| 知识水平 | 专家/了解/入门/陌生 |
| 核心关注 | |
| 潜在质疑 | |
| 决策权重 | 一票否决/建议权/参考 |
| 沟通偏好 | 数据驱动/故事驱动/逻辑驱动 |

### 2.2 说服策略蓝图
- 对CEO/高管层：讲战略、ROI、风险、竞争格局
- 对CTO/技术层：讲架构、可行性、技术指标、工程细节
- 对客户：讲价值、案例、信任、服务保障
- 对团队：讲愿景、路径、激励、协作方式

## 阶段三：内容架构

**角色**：内容架构师

### 3.1 金字塔结构设计
核心主张 → 论据（数据/案例/逻辑）

### 3.2 叙事结构选择
| 结构 | 适用场景 |
|------|----------|
| SCQA (Situation-Complication-Question-Answer) | 战略提案/问题分析 |
| 问题-解决方案 | 产品方案/项目申报 |
| 英雄之旅 | 品牌故事/变革管理 |
| 时间线 | 年度总结/项目复盘 |
| 对比叙事 | 竞品分析/方案选型 |
| What-Why-How | 技术方案/培训材料 |

### 3.3 叙事线脚本
每页PPT/每节3要素：关键信息（1句话）、说服逻辑、过渡

## 阶段四：内容创作

**角色**：专业撰稿人 + 数据分析师

### 4.1 写作原则
- 每页/每节只传递 **1个核心信息**
- 使用 **口语化短句**
- 采用 **"先说结论再说原因"** 的倒金字塔结构
- 每个观点都有 **数据或案例** 支撑

### 4.2 数据可视化决策
| 数据类型 | 推荐图表 |
|----------|----------|
| 趋势 | 折线图 |
| 对比 | 柱状图/条形图 |
| 占比 | 饼图/环形图 |
| 关系 | 散点图/气泡图 |
| 流程 | 流程图/泳道图 |
| 层级 | 树图/组织架构图 |

## 阶段五：视觉设计

**角色**：视觉设计师

### 5.1 8大设计决策维度
配色、字体、版式、图标、图片、图表、动效、模板

### 5.2 设计原则
对比、重复、对齐、邻近

## 阶段六：审核优化

**角色**：质量评审官

### 6.1 三审制
1. **内容审核**：事实准确性、逻辑连贯性、数据来源验证
2. **视觉审核**：品牌一致性、可读性、美观度
3. **受众视角审核（Red Team）**：模拟受众提出问题

### 6.2 审核清单
```
□ 核心信息是否每页都指向同一目标？
□ 是否有冗余或无关内容？
□ 数据是否准确、来源是否可追溯？
□ 逻辑链是否完整？
□ 每页是否只有一个核心信息？
□ 字体/颜色/图标是否一致？
□ 在投影演示场景下是否可读？
□ 是否预判并回应了主要质疑点？
□ 时间控制是否合理？
```

## 阶段七：交付复盘

**角色**：项目经理

### 7.1 交付物包
最终材料、讲稿/备注、讲者指引、备份文件、分发材料

### 7.2 复盘模板
- 本次做得好的
- 下次可以改进的
- 过程效率评分（1-10）
- 沉淀的模板/资产

## 场景扩展：企业内部竞聘/面试报告

For corporate interview presentations (竞聘报告), see the specialized reference:
[references/corporate-interview-presentation.md](references/corporate-interview-presentation.md)

**⚠️ Note**: This skill has significant overlap with `material-production-system` (devops category). For governance reports, regulatory materials, and tool-landing presentations, that skill is more comprehensive and actively maintained.

Key differences from general reports:
- Must start with **interviewer profiling** (new bosses need quick trust)
- Narrative chain: strategy → role understanding → advantages → proof → plan → summary
- Team-building measures must be **concrete actions** (轮岗/知识库/双向培训), not slogans
- Self-intro keywords page is essential when new leaders are on the panel

### 竞聘PPT的顾问团预检模式（v1.1新增）

竞聘PPT属于**高风险内容**——数据不一致会直接摧毁可信度。建议在PPT生成前，先走顾问团（C007）内部预审：

**流程**：用户提供内容稿 → 7位顾问独立评审 → 汇总P0/P1/P2问题 → 用户确认3个关键决策 → 生成实际PPT

**预检检查清单**（实战验证，2026-05-06）：
```
□ ROI/财务数据是否跨页一致？（最常见的问题——P1和P8数据易脱节）
□ 叙事链是否合理？（价值→愿景→定位→能力→案例，而非倒置）
□ 关键评委是否有专属页/内容？（一票否决权评委需占更多时长）
□ 财务模型是否合理？（过高ROI会被追问依据）
□ 数据口径是否统一标注？（保费规模/GMV/利润？时间跨度？）
□ 人事举措是否可执行？（轮岗周期、预算来源？）
□ 演讲节奏是否合理？（每位评委分配对应时长）
```

### 竞聘PPT标准化结构模板

基于平安竞聘实战提炼（14页，15分钟）：

| 序号 | 页面 | 建议时长 | 受众定位 |
|:----:|:-----|:--------:|:---------|
| 封面 | 价值宣言（三组数字定调） | 30s | 全员 |
| P1 | 价值锚定（三个确定性价值） | 1.5min | 全员 |
| P2 | 产品愿景（战略方向） | 1min | 全员 |
| P3 | 岗位定位（铁三角） | 1.5min | 全员 |
| P4 | 核心能力（雷达图/维度） | 1min | 技术+业务 |
| P5-P7 | 案例证明（三个业务场景各1min） | 3min | 各业务负责人 |
| **P8** | **团队打造**（2min ⭐） | **2min** | **人力/团队决策人** |
| **P9** | **资源优化**（2min ⭐） | **2min** | **财务/风控决策人** |
| P10-P11 | 技术+路线图 | 2min | CTO/全员 |
| P12 | 六个承诺（向每位评委承诺） | 1.5min | 全员 |
| 封底 | 三大数字收束 | 30s | 全员 |

### 竞聘PPT的15分钟节奏分配法则

- 有**一票否决权**的评委：每个评委2分钟专属内容
- 其余评委：1分钟或共享页
- 全员共鸣页（封面+P1+P2+P3+封底）：4.5分钟（30%总时长）
- 案例展示（P5-P7）：3分钟（20%总时长）——不能太长，否则变流水账
- 专属页（P8-P9）：4分钟（27%总时长）——核心说服时刻
- 收束（P12+封底）：2分钟（13%总时长）

### 竞聘PPT的视觉设计模板（平安商务风）

基于平安标准商务风格实战（可用pptxgenjs直接生成）：

**配色**：
- 主色深蓝: `1A3C6E`（封面/封底背景、标题）
- 强调红: `C41E24`（关键数字强调、风险标识）
- 金色: `D4A843`（财务数据强调、ROI高亮）
- 浅灰背景: `F5F6FA`（内容页）
- 深灰文字: `333333` / 副标题: `666666`
- 辅助底色: `D6E4F0`（服务卡片） / `FDECEA`（风险卡片） / `FAF3E0`（财务卡片）

**布局模式**（每个页面应选其一，减少重复）：
- 三栏卡片（价值锚定页：财务/技术/风险三栏）
- 三角模型（定位页：三个职责顶点）
- 双栏对比（团队页：平安基因 vs 互联网思维）
- 时间轴（发展历程/里程碑）
- 大数字仪表盘（业绩展示）
- 座席卡网格（六个承诺：2×3或3×2排列）
- 四象限矩阵（风险控制）

**页面设计参考资料**：本skill的references/目录下有完整的14页pptxgenjs生成脚本示例

## 场景扩展：跨多场专题会阶段性情况汇报

For a progress report that synthesizes items from multiple recent meetings led by the same leader — see the specialized reference:

[references/cross-meeting-progress-report.md](references/cross-meeting-progress-report.md) (new 2026-05-18)

Key differences from single-report workflow:
- **Multiple source meetings**: 2-4 meetings, each with multiple action items
- **Structure**: overview stats → per-meeting tables → deep-dive cards → timeline → items needing leader coordination → next steps
- **Leader profile pre-read required**: must align with the leader's management style (老苏: data-driven, conclusion-first, hates surprises)
- **State coloring**: ✅ completed / 🔄 in-progress / ⏳ needs coordination
- **Must not omit meetings**: users track completeness — cross-check with C013 Get notes + Feishu tasks + memory

## 场景扩展：EAST季度治理汇报（三段式+三交付）

For EAST quarterly governance reports — 监管动态→运行实况→工作部署 三段式结构 + 内容纲要/PPT提示词/逐页文稿 三交付物模式:

[references/east-quarterly-governance-report-v5.md](references/east-quarterly-governance-report-v5.md) (updated 2026-05-19: EAST2.0 宽/深/强框架 + 张燕事件排查汇报数据)

Key differences from general reports:
- **三段式是领导指定结构**，不是可选择的叙事选项——严格遵循不能改
- **三交付物并行产出**：内容纲要（设计图纸）→ PPT提示词（施工说明）→ 逐页文稿（剧本）
- **v6.0 14页架构已演化**：基于翟总8项具体要求，9页→14页演进，含三区监管景观、环图仪表盘、三卡片升级、责任矩阵、CTA收尾等新设计模式。详见 `material-production-system` skill 的 `references/east-q2-report-v6-pattern.md`
- **P0事项交叉强化**：必须同时出现在机制优化页和具体事项清单页
- **核心认知框**：P5页用深红边框大字展示核心结论
- **★决策页标记**：P7协作框架职责矩阵标记为核心决策页
- **EAST2.0三维度描述强制规则**：必须同时覆盖宽度（字段扩宽）、深度（表间关系+业务-财务-资金三流一致性）、强度（校验规则翻倍+数据量增2~3倍）三个方面
- **张燕事件内容来源强制规则**：必须引用《关于近期监管员EAST数据风险提示的排查汇报》文件中的确切数据（极端异常27单/其他异常213单/3类原因），不得使用泛化描述

## 场景扩展：微信/即时通讯简洁汇报

For short WeChat/TIM progress reports to leadership (not formal email documents) — connect to previous communication naturally while respecting the medium's brevity constraints.

### 适用场景

- 向领导微信汇报近期重点事项进展
- 上次已通过邮件/微信汇报过，本次更新进展
- 需要突出**1-2个关键变化点**而非全景覆盖

### 与正式汇报的关键差异

| 维度 | 正式邮件/文档汇报 | **微信简洁汇报** |
|------|----------------|----------------|
| 可滚动长度 | 不限（可加附件） | 3-4屏上限，不能再多 |
| 语气 | 正式书面语 | **半口语化**，开头可用问候语 |
| 结构 | 分节标题+表格+附录 | 自然段落，**无显式标题**（只有符号分段） |
| 已完成事项处理 | 全量详述 | **一句话带过**（已通过其他形式汇报过） |
| 事项覆盖 | 全部交代事项 | 选择性聚焦，1个重点深度展开 |
| 需协调项 | 独立的"需领导协调"区块 | **嵌入对应事项段落末尾**，不单独成节 |
| 外部变化 | 不提或仅在附录中 | 关键变化点需**自然嵌入**主体，展现信息同步 |

### 微信汇报五星结构法则

基于用户实战验证的5点要求（2026-05-19）：

```
① 连贯性挂钩 — 开头1句话关联上次汇报
   例："苏总，关于上次会议交代事项的最新进展跟您汇报一下"
   
② 已完成事项从简 — 已用其他形式汇报过的，1句话带过
   例："手续费率数据修复已报送，已通过邮件向您汇报过，不再赘述"
   
③ 推进中事项详述 — 展开描述进展，可链接邮件抄送内容
   例："审计田晓俊发来的建议书已抄送您，涉及的XX事项正在XX"
   
④ 确定本次重点 — 选1项作为深度展开的核心主题
   例："科技资源到位情况（重点）" → 用较长段落详述进展+压力
   
⑤ 外部变化+自身紧迫 — 体现最新动态，连接已发的回应邮件
   例："集团徐周上周发了East采购意见。我已回复明确了我方紧迫性..."
```

### 执行检查清单

```
□ 开头是否关联了上次沟通？（避免"突兀开启"）
□ 已完成事项是否从简处理？（避免"重复汇报"）
□ 推进中的事项是否逐项有进展描述？（不仅罗列事项名称）
□ 是否有1个重点话题明显展开？（识别出"这次要说的重点"）
□ 外部变化是否自然嵌入？（参考外部邮件/集团通知）
□ 需领导协调的事项是否嵌入段落而非独立成节？
□ 文档总长度是否控制在3-4屏以内？
□ 语气是否介于正式与口语之间？（不过分随意也不机械书面的）
```

### 常见陷阱

1. ❌ **已完成事项反而大写特写** — 用户已经用其他形式汇报过，不需要再详细叙述
2. ❌ **结构过于正式**（如冒号分段、独立"需协调事项"小节）— 微信要自然连贯
3. ❌ **不提外部变化** — 最新外部邮件/集团动态是展示"我在追踪"的关键信号
4. ❌ **重点不突出** — 所有事项平均用力等同于没有重点
5. ❌ **忘记关联上次回复** — 用户明确要求"连贯性"（与之前微信回复衔接）


## 场景扩展：向一把手汇报监管/事项类材料

For executive status reports to a top leader (党委书记/总经理级别) — such as regulatory inquiries, risk events, or special investigations — see the specialized reference:

[references/executive-status-report.md](references/executive-status-report.md) (updated 2026-05-11: v3 patterns — "完全压缩版" result-only structure + multi-point executive feedback workflow + EAST classification difference rule)

Key differences from general reports:
- **Not a request document**: inform, don't ask. Coordination needs stay out of the main body.
- **"举一反三" (proactive initiative) narrative**: separate "regulatory response" from "proactive discovery" to demonstrate initiative
- **Current progress three-segment structure**: 核心工作目标 → 当期已推进 → 后续优化 (not separate sections)
- **Causes only for the core issue**: exclude tangential causes even if technically accurate
- **Professional but accessible**: translate technical jargon into business language (e.g. "税金" → "上游来源系统数据处理")
- **Single authoritative data source**: all numbers from one document, with source attribution
- **Quantify everything**: completed items show numbers; targets show dates and standards

## 核心铁律

1. **需求挖掘 → 项目成功50%**
2. **内容先于设计**
3. **受众决定一切决策**

## 常见陷阱与应对

| 陷阱 | 应对 |
|------|------|
| 用户说"我也没想清楚" | 选项引导法 |
| 用户什么都想要 | 三选一法则 |
| 用户说"和上次一样" | 调取历史材料验证 |
| 用户不耐烦追问 | 说明节省改稿成本的理由 |
| 需求确认不签字 | 不要进入下一阶段 |
