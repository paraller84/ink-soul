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
