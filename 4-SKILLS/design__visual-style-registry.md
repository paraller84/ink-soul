---
name: visual-style-registry
category: design
description: 统一视觉风格注册表 — 注册所有跨系统/跨场景的视觉风格规范（CSS变量+设计Token），供 design-document-production、education-pad-ui、汇报材料系统等skill引用。
tags: [design, css, visual, theme, token, style-guide]
---

# 视觉风格注册表

## 定位

当需要为系统/功能设计视觉方案时，先查本注册表，看已有风格是否可直接引用，避免重复声明CSS变量。

当用户表达对配色的偏好/修正时，更新本注册表。

## 注册规则

- 每条风格记录包含：名称、适用场景、CSS Token体系、十六进制色板、典型使用示例
- 风格之间允许重叠（如「护眼教育风」是「清透蓝白风」的衍生变体）
- 风格注册后，在对应skill中通过 `#视觉风格引用：XXXX` 标记引用，不要在各skill中重复写完整CSS变量

---

## 风格索引

| 编号 | 名称 | 适用场景 | 注册位置 |
|:----|:-----|:---------|:---------|
| V01 | 暗色科技风 | 系统设计文档、管理后台原型 | 本文件 §V01 |
| V02 | 护眼教育风 | Edu-Hub教育系统、AI家庭教师 | 本文件 §V02 |
| V03 | 清透极简蓝白风 | 汇报材料PPT、EAST季度会材料 | 本文件 §V03 |
| V04 | 公司汇报标准风 | 所有公司内部汇报PPT标准风格 | 本文件 §V04 |
| V05 | 儿童教育暗色风 | 小学男生管理界面（深色三选方案） | 本文件 §V05 |
| V06 | **清爽蓝天** ✅ 当前使用 | AI家庭教师家长管理端（用户选定） | 本文件 §V06 |

---

## V01: 暗色科技风

**触发场景**：系统设计说明书、管理后台原型、技术文档UI

```css
:root {
  --bg-primary: #0a0e1a;
  --bg-secondary: #111827;
  --bg-card: #1a1f2e;
  --text-primary: #e2e8f0;
  --text-secondary: #94a3b8;
  --accent: #3b82f6;
  --accent-hover: #2563eb;
  --success: #22c55e;
  --warning: #f59e0b;
  --error: #ef4444;
  --border-color: #2a3148;
  --font-mono: 'Fira Code', 'JetBrains Mono', monospace;
  --font-sans: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.3);
  --radius: 8px;
}

/* 页面原型内设备屏幕使用暖白底色模拟真实设备 */
.device-screen {
  background: #fdf8f0;
  color: #333;
}
```

**使用示例**：`design-document-production` skill 中的 `:root` 段直接引用此规范。

---

## V02: 护眼教育风

**触发场景**：Edu-Hub教育系统、AI家庭教师App、儿童学习Pad界面

```css
:root {
  /* 主背景 — 暖白护眼 */
  --bg-primary: #fdf8f0;
  --bg-secondary: #f5f0e8;
  --bg-card: #ffffff;
  
  /* 文字 — 柔和 */
  --text-primary: #2c2c2c;
  --text-secondary: #666666;
  --text-muted: #999999;
  
  /* 强调色 — 低饱和度彩 */
  --accent-blue: #4a90d9;
  --accent-green: #5cb85c;
  --accent-orange: #f0ad4e;
  --accent-red: #d9534f;
  --accent-purple: #8e6cc7;
  
  /* 儿童友好 */
  --btn-primary: #4a90d9;
  --btn-success: #5cb85c;
  --btn-warning: #f0ad4e;
  --btn-danger: #d9534f;
  --btn-radius: 16px;
  
  /* 边框 */
  --border-color: #e0d8cc;
  --border-radius: 12px;
  
  /* 字体 */
  --font-size-base: 18px;    /* 儿童阅读最低16px */
  --font-size-title: 28px;
  --font-size-heading: 22px;
  --font-family: -apple-system, 'PingFang SC', 'Noto Sans SC', sans-serif;
  
  /* 间距 */
  --spacing-xs: 4px;
  --spacing-sm: 8px;
  --spacing-md: 16px;
  --spacing-lg: 24px;
  --spacing-xl: 32px;
}
```

**使用示例**：Edu-Hub的 `edu-design-system.css` 直接引用此规范。

**设计文件路径**：`~/edu-hub/static/css/edu-design-system.css`

---

## V03: 清透极简蓝白风

**触发场景**：内部汇报PPT、季度会材料、工作简报

```css
:root {
  /* 背景 */
  --bg-page: #ffffff;
  --bg-card: #f8fafc;
  --bg-accent-light: #eff6ff;
  
  /* 文字 */
  --text-primary: #1e293b;
  --text-secondary: #64748b;
  --text-muted: #94a3b8;
  --text-on-accent: #ffffff;
  
  /* 主色调 */
  --accent: #2563eb;
  --accent-light: #3b82f6;
  --accent-lighter: #bfdbfe;
  
  /* 辅助色 */
  --success: #16a34a;
  --warning: #d97706;
  --danger: #dc2626;
  --info: #0284c7;
  
  /* 装饰 */
  --border-color: #e2e8f0;
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.05);
  --shadow-md: 0 4px 6px rgba(0,0,0,0.05);
  
  /* 字号（PPT专用） */
  --title-size: 24pt;
  --heading-size: 18pt;
  --body-size: 12pt;
  --note-size: 10pt;
}
```

**使用示例**：EAST二季度汇报PPT v4「清透极简蓝白风」

---

## V04: 公司汇报标准风

**触发场景**：所有公司内部汇报PPT、EAST治理材料、工作回顾、专项方案

**设计依据**：基于《EAST数据治理专项工作方案v2》和《EAST监管报送问题回顾》两份材料的风格分析——蓝色系占全部用色70%以上，统一为深蓝+亮蓝双轨制。

**完整风格文档**：已上传至飞书云盘 系统文档→能力体系 文件夹 (`ZO1QbyE9ZoBKAIxVbbzcaQscnQg`)

**色板**：

```
品牌主色:     #154B8C (深蓝·封面/大标题)
品牌深色:     #1E40AF (章节标题/强调文字)
品牌标准色:   #2563EB (二级标题/图表主色)
品牌亮色:     #3B82F6 (辅助图表/选中态)
品牌淡色:     #60A5FA (底色装饰)

语义色:
  警戒红:     #C00000 (高风险/禁止/强制提醒)
  异常红:     #DC2626 (数据异常/负面指标)
  预警橙:     #EA580C (预警/关注/待改进)
  强调金:     #C4962C (成就/特殊标记/领导指示)
  成功绿:     #16A34A (完成/达标/增长)

中性色(七阶):
  正文:       #1E293B (默认正文)
  辅助文字:   #475569 (次要说明)
  注释:       #94A3B8 (注释/页脚)
  边框:       #CBD5E1 (分割线/表格边框)
  浅线:       #E2E8F0 (浅分割线)
  底纹:       #F1F5F9 (表头/卡片底)
  白底:       #FFFFFF (页面底色)
```

**使用规则**：
- 主色占比 ≥ 70%（蓝系色占绝对主导）
- 单页语义色 ≤ 2种（红/橙/金最多同时出现2种）
- 红色只用于「有问题」场景，不可作装饰色
- 绿色只用于「已完成/增长」场景
- 中性色占页面 60-70%
- 1页不超过4种彩色

**字体层级**：
| 层级 | 字号 | 粗细 | 色值 |
|:-----|:----:|:----:|:----:|
| 封面大标题 | 36-44pt | Bold | #154B8C |
| 页面标题 | 24-28pt | Semi-Bold | #1E40AF |
| 二级标题 | 18-20pt | Semi-Bold | #1E293B |
| 正文 | 14-16pt | Regular | #1E293B |
| 注释 | 10-12pt | Regular | #64748B |
| 表格内文 | 9-11pt | Regular | #1E293B/#475569 |

**字体**：微软雅黑（中文）+ Arial（英文/数字）

**数据色板(10色)**：2563EB→1E40AF→EA580C→16A34A→C4962C→8B5CF6→DC2626→3B82F6→EC4899→14B8A6

**布局模板**：5种标准布局（封面页/目录页/标准内容页/数据表格页/四格对比页）

**飞书文档**：https://nofile.feishu.cn/file/ZO1QbyE9ZoBKAIxVbbzcaQscnQg

---

## V05: 儿童教育暗色风（深色版·三种子风格）

**触发场景**：AI家庭教师家长管理端、教育App深色模式、小学男生使用的管理系统

**设计背景**：用户要求在现有暗色布局基础上调整配色，目标受众为**小学男生**，需兼顾视觉吸引力与抗疲劳需求。

**三种配色范式（子风格）：**

| 编号 | 名称 | 定位 | 核心色彩 | 抗疲劳设计 |
|:-----|:------|:------|:---------|:-----------|
| V05-A | **宇宙深蓝** 🌌 | 科技探索·太空游戏感 | 深空蓝底 `#070d1a` + 亮蓝强调 `#38bdf8` | 卡瓦蓝低对比背景，亮蓝仅用于强调色 |
| V05-B | **翠绿森林** 🌿 | 自然柔和·顶级护眼 | 墨绿底 `#070f0a` + 翠绿强调 `#4ade80` | 绿光波长居中不易疲劳，背景最低对比 |
| V05-C | **暖橙活力** 🔥 | 温暖阳光·积极亲切 | 暖褐底 `#0a0705` + 金橙强调 `#fbbf24` | 暖色系低蓝光刺激，温馨感降低用眼紧张 |

### V05-A: 宇宙深蓝

```css
:root {
  --bg-primary: #070d1a;
  --bg-card: #0f1a2e;
  --bg-card-hover: #14213d;
  --bg-hero: linear-gradient(135deg,#0f2942,#1a1a3e);
  --text-primary: #e2e8f0;
  --text-secondary: #64748b;
  --accent-primary: #38bdf8;   /* 亮蓝 — 太空主题 */
  --accent-success: #34d399;   /* 翠绿 */
  --accent-warning: #fbbf24;   /* 琥珀 */
  --accent-purple: #a78bfa;    /* 淡紫 */
  --accent-pink: #f472b6;      /* 淡粉 */
  --accent-orange: #fb923c;    /* 暖橙 */
  --border-color: #1a2d4a;
}
```
**适合场景**：三年级以上男孩、偏喜欢科技/游戏/太空题材的儿童

### V05-B: 翠绿森林

```css
:root {
  --bg-primary: #070f0a;
  --bg-card: #0f1f14;
  --bg-card-hover: #152a1c;
  --bg-hero: linear-gradient(135deg,#0d2818,#1a2e1a);
  --text-primary: #dcfce7;     /* 偏绿白 */
  --text-secondary: #6b7280;
  --accent-primary: #4ade80;   /* 翠绿 */
  --accent-cyan: #2dd4bf;      /* 青绿 */
  --accent-warning: #facc15;   /* 金黄 */
  --accent-purple: #e879f9;    /* 淡紫 */
  --accent-pink: #fb7185;      /* 淡粉 */
  --accent-orange: #fb923c;    /* 暖橙 */
  --border-color: #1a3a24;
}
```
**适合场景**：长时间使用（>30min）、低龄段儿童、视觉敏感易疲劳的用户

### V05-C: 暖橙活力

```css
:root {
  --bg-primary: #0a0705;
  --bg-card: #1f140e;
  --bg-card-hover: #2a1c14;
  --bg-hero: linear-gradient(135deg,#2d1b0e,#3a1f0e);
  --text-primary: #fef3c7;     /* 偏暖白 */
  --text-secondary: #6b7280;
  --accent-primary: #fbbf24;   /* 金橙 */
  --accent-blue: #38bdf8;      /* 亮蓝 */
  --accent-green: #34d399;     /* 翠绿 */
  --accent-purple: #a78bfa;    /* 淡紫 */
  --accent-pink: #fb7185;      /* 淡粉 */
  --accent-orange: #f97316;    /* 暖橙 */
  --border-color: #3a2818;
}
```
**适合场景**：需要营造温暖积极氛围的界面、偏活泼好动的儿童

### 使用说明

- 三种子风格共享相同布局结构（同 V01 暗色科技风），仅替换色值
- 选择依据：用户年龄段 + 使用时长 + 视觉偏好
- V05-B 护眼最佳（绿色光谱中段，睫状肌调节最小）
- V05-A 吸引力最强（太空主题对男童天然吸引力）
- V05-C 情绪最积极（暖色提升多巴胺分泌）

## V06: 清爽蓝天（家长管理端·当前主题）

**触发场景**：AI家庭教师家长管理端、小学男生使用的家长中心界面

**设计背景**：用户要求非深色布局，适合小学男生审美且防止视觉疲劳。选择「清爽蓝天」方案——蓝天白云般干净明快，低刺激护眼。

### CSS Token (CSS变量)

```css
:root {
  /* 页面底色 — 柔和的天空蓝 */
  --bg-page: #f0f7ff;
  /* 卡片底色 — 纯白 */
  --bg-card: #ffffff;
  --bg-card-hover: #f8fbff;
  /* 头部渐变 — 浅蓝到天蓝 */
  --bg-hero: linear-gradient(135deg, #dbeafe, #e0f2fe);
  /* 文字 */
  --text-primary: #1e293b;
  --text-secondary: #64748b;
  --text-muted: #94a3b8;
  --text-hero: #1e40af;
  /* 主色调 — 天空蓝 */
  --accent: #0284c7;
  --accent-hover: #0369a1;
  --accent-light: #dbeafe;
  /* 边框 */
  --border-color: #dce8f5;
  --border-hover: #93c5fd;
  /* 辅助色 */
  --success: #16a34a;
  --warning: #d97706;
  --error: #dc2626;
  --info: #0284c7;
  --purple: #7c3aed;
  --pink: #ec4899;
  --orange: #f97316;
}
```

### 实现方式

- `static/css/parent-theme.css` — 以 `.parent-theme` 为作用域的所有UI组件覆盖
- `parent-theme.css` 使用 `!important` 安全覆盖确保深色内联样式被覆盖
- body 标签添加 `class="parent-theme"` 启用主题
- 学生端不受影响（无 parent-theme class）

### 使用示例

AI家庭教师家长端12个页面全部继承此风格，包括：语文主页、古诗/课文、出题设置、学习报告、字词详情、题库、练习卷、内容管理、练习计划、积分管理、拍照批改、家长仪表盘。

### 选择记录

用户对比了 V05 三套深色方案（深蓝/翠绿/暖橙）后拒绝了所有深色，选择了浅色方案 V06。这对未来配色决策有参考价值——**小学男生的家长管理界面，浅色明快风格优于深色科技风**。深色风格虽视觉冲击力强，但长时间使用易视觉疲劳，且不符合儿童学习的心理学氛围。

### 实现架构（可复用）

```html
<!-- base.html: 添加 body_class block -->
<body class="{% block body_class %}{% endblock %}">

<!-- 每个页面: 启用主题 -->
{% block body_class %}parent-theme{% endblock %}
{% block head_extra %}
  <link rel="stylesheet" href="{{ url_for('static', filename='css/parent-theme.css') }}">
{% endblock %}

<!-- parent-theme.css: 以 .parent-theme 为作用域 -->
.parent-theme { background: #f0f7ff; color: #1e293b; }
.parent-theme .pg-card { background: #ffffff; border-color: #dce8f5; }
```
这种通过 body_class + 作用域 CSS 的架构，可以安全地对 **页面子集** 应用不同主题，而不影响主 CSS 体系。无需修改任何全局 CSS，所有覆盖都在独立文件中。`!important` 用于确保覆盖内联 `<style>` 块中的深色值。

---

## 更新规范

| 信号 | 动作 |
|:-----|:-----|
| 用户说「配色改一下」 | 新增变体或修改现有Token |
| 用户对儿童配色表达偏好 | 在V05子风格中新增或调整 |
| 新场景出现（如汇报风→运营风） | 新增V0N记录 |
| 用户对已有风格表达偏好修正 | 更新对应Token值，标注变更日期 |
