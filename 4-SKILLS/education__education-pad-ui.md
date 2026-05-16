---
name: education-pad-ui
description: 儿童教育App的Pad端界面设计规范和适配方法论。覆盖布局系统（侧栏+主区/全宽网格/全宽单列）、组件尺寸（儿童触摸友好56px+）、设计Token系统、从手机到Pad的适配策略，以及原型先行的工作流。
tags:
  - pad-ui
  - children-ux
  - design-system
  - touch-targets
  - responsive-layout
trigger: 当用户要求做Pad端适配、界面大屏优化、触屏儿童UI设计、布局重构时加载此技能。
---

# 儿童教育Pad端UI设计规范

## 核心原则

1. **Pad优先，非手机拉伸** — 不要将手机界面简单放大，而是重新设计布局以利用Pad宽度
2. **拒绝留白** — 利用Pad宽度做左右分栏或大宫格布局，不把内容挤在中间
3. **大尺寸、易操作** — 所有交互元素面向儿童手指设计，最小触摸高度56px
4. **全局统一** — 所有页面遵循同一套设计Token系统

## 布局系统

### 三种布局模式

| 模式 | 适用场景 | 说明 |
|---|---|---|
| **A: 侧栏+主区** | 首页、做题页、结果页、错题本、个人中心 | 左侧240px固定深色导航栏，右侧弹性主内容区，充分利用水平空间 |
| **B: 全宽网格** | 选学科、选题模式、荣誉墙、商城 | 3-4列大宫格，每个格≥240px，内容饱满 |
| **C: 全宽单列** | 做题页(长文本)、拍照纠正、知识图谱 | 居中宽列(800-960px)，左右留白但内容完整 |

### 容器层级

```
.app-container (max-width: 1200px, 居中, 24px内边距)
  └─ .pg-layout (display:flex, gap:24px)
       ├─ .sidebar (240px, 深色#1a1a2e, sticky)
       └─ .main-area (flex:1)
            ├─ .pg-top (顶栏, 白色卡片)
            └─ 内容卡片/网格...
```

## 组件尺寸系统（Pad专用）

面向儿童的触摸目标是核心竞争力：

| 组件 | 最小高度 | 内边距 | 字号 | 标准（WCAG） |
|---|---|---|---|---|
| `.pg-btn` (默认) | 56px | 16px 36px | 1.1rem | ≥44px ✅ |
| `.pg-btn.large` | 64px | 20px 48px | 1.2rem | ✅ |
| `.pg-btn.small` | 48px | 12px 24px | 0.95rem | ✅ |
| `.choice-list li` (选项) | 56px | 18px 24px | 1.05rem | ✅ |
| `.choice-marker` (选项标记) | 28px圆 | - | 1rem | ✅ |
| `.pg-input` (输入框) | 56px | 16px 20px | 1.05rem | ✅ |
| `.pg-tab-item` (选项卡) | 48px | 14px 24px | 1.0rem | ✅ |
| `.hw-cell` (手写格子) | 120×130px | - | - | ✅ |
| `.q-nav-dot` (题号导航点) | 36px圆 | - | 0.85rem | ✅ |
| `.subj-card` (学科卡片) | min-h 240px | 32px | 1.5rem标题 | ✅ |

## 间距系统

| Token | px | 使用场景 |
|---|---|---|
| `--gap-xs` | 8 | 极紧凑内联间距 |
| `--gap-sm` | 12 | 图标+文字之间 |
| `--gap-md` | 16 | 弹性布局默认gap |
| `--gap-lg` | 24 | 卡片间距、分栏间距 |
| `--gap-xl` | 32 | 段落间距 |
| `--pad-sm` | 16 | 卡片内边距(小) |
| `--pad-md` | 24 | 标准卡片内边距 |
| `--pad-lg` | 32 | 大卡片内边距 |

## 字号系统（Pad）

| 层级 | 值 | 字重 | 使用 |
|---|---|---|---|
| fs-xl | 1.5rem (24px) | 800 | 大标题 |
| fs-lg | 1.3rem (21px) | 700 | 卡片标题 |
| fs-md | 1.1rem (18px) | 700 | 按钮文字、题号 |
| fs-body | 1.05rem (17px) | 600 | 正文、选项文字 |
| fs-sm | 0.9rem (14px) | 500 | 辅助文字 |
| fs-xs | 0.8rem (13px) | 400 | 标签、徽章 |

## 视觉系统

### 色彩（教育类推荐暖白+紫）

- 主背景：`#f5f0e8` 暖白
- 卡片背景：`#ffffff`
- 深色侧栏：`#1a1a2e` 深空蓝黑
- 侧栏文字：`rgba(255,255,255,0.85)` / 次要 `0.45`
- 主色：`#6c63ff` → 按压 `#5a52e0`
- 成功：`#4caf50`
- 警告：`#f39c12`
- 危险：`#e57373`

### 圆角

| 层级 | 值 | 使用 |
|---|---|---|
| radius-sm | 8px | 小图标 |
| radius-md | 12px | 卡片、输入框 |
| radius-lg | 16px | 标准卡片 |
| radius-xl | 20px | 大卡片、侧栏 |
| radius-full | 9999px | 胶囊按钮 |

### 动效

- 过渡：`0.15s ease`
- 按钮点击反馈：`scale(0.96)`
- 页面淡入：`fadeIn 0.3s`

## 页面-布局映射（参考）

| 页面 | 布局模式 | 说明 |
|---|---|---|
| 学生首页 | A: 侧栏+主区 | 左侧学生信息+导航，右侧4列统计+3列科目+最近练习 |
| 做题页 | A: 侧栏+主区 | 顶部进度条，中部题目+答题区(选择/手写) |
| 结果页 | A: 侧栏+主区 | 分数圆环+4列子统计+答题详情列表 |
| 选学科 | B: 全宽网格 | 3大科目大卡片+今日概况 |
| 选模式 | B: 全宽网格 | 3种练习模式大卡片+学习建议 |
| 个人中心 | A: 侧栏+主区 | 头像+等级信息+多列统计 |
| 错题本 | A: 侧栏+主区 | 左侧筛选+右侧错题列表 |
| 荣誉墙 | B: 全宽网格 | 4列荣誉徽章大卡片 |
| 金币商城 | B: 全宽网格 | 3列商品+侧栏余额 |
| 知识图谱 | C: 全宽单列 | 图表最大化宽度 |

### 适配策略（从手机到Pad）

1. **移除双视图框架** — 不要同时维护phone(375px) + pad(600px)两套视图。统一为单一Pad优先视图
2. **max-width 从600→1200px** — 充分利用Pad宽度
3. **嵌入侧栏导航** — 替代手机端底部导航，利用水平空间
4. **单列→多列** — 统计卡片1列变4列，科目列表变3列宫格
5. **放大所有交互元素** — 按钮/选项/输入框高度提升到56px+
6. **原型先行** — 先出HTML原型让用户确认布局方向，再实施改造

### Jinja 实现技巧：侧栏导航高亮

当使用父模板 + 子模板继承模式实现侧栏导航时，需解决 **"哪个导航项高亮"** 的问题。标准做法：

**方案：`self.page_id()` 模式**

父模板 (`pad_base.html`) 定义 block 占位，然后用 `self.page_id()` 读取子模板的填充值：

```jinja
{# pad_base.html — 父模板 #}
<body>
  {% block page_id %}{% endblock %}               {# ← 子模板填充页标识 #}
  {% set _page_id = self.page_id() %}             {# ← 读取子模板的 page_id 值 #}
  
  <nav class="sidebar-nav">
    {% set _nav_items = [
      ('home', '🏠', '首页', url_for('student.home')),
      ('wrong_book', '📕', '错题本', url_for('student.wrong_book')),
    ] %}
    {% for _id, _emoji, _label, _url in _nav_items %}
    <a href="{{ _url }}" class="sidebar-nav-item{% if _page_id == _id %} active{% endif %}">
      <span class="emoji">{{ _emoji }}</span> {{ _label }}
    </a>
    {% endfor %}
  </nav>
</body>
```

子模板只需定义 `page_id`：

```jinja
{% extends "pad_base.html" %}
{% block page_id %}home{% endblock %}
{% block content %}
  ...
{% endblock %}
```

**关键原理**：`self` 在 Jinja 中指向当前模板对象，`self.page_id()` 调用子模板的 block 并返回其渲染内容。这避免了在子模板中手动设置变量（因为 `{% set %}` 不能在 `{% extends %}` 之前使用）。

**不要使用** context_processor 注入 page_id 的方式 — 会导致高亮逻辑分散在路由层，每个端点需要重复传递参数。block 机制是模板层原生方案，零耦合。

### 批量化模板改造流程

当需要将多页面（15+）从手机/Pad双视图改造为Pad单视图时，手动改造效率低且易出错。推荐批量化流程：

1. **清单枚举** — 列出所有需改造模板，按结构模式分组（带 frame-container 的 vs 不带 frame-container 的 vs 不参与改造的）
2. **脚手架先行** — 先创建 `pad.css` 和 `pad_base.html`，确保布局骨架就绪后再改造具体页面
3. **批量化脚本** — Python 脚本做一次性多文件处理：
   ```python
   # 核心变换逻辑
   for fname in TEMPLATES:
       content = read(fname)
       content = content.replace('extends "base.html"', 'extends "pad_base.html"')
       content = insert_page_id_block(content)
       content = strip_phone_section(content)   # 删除 <!-- PHONE VIEW --> 块
       content = strip_wrapper_divs(content)     # 移除 pad-frame/pad-inner 包裹层
       content = strip_frame_container(content)  # 移除 frame-container
       write(fname, content)
   ```
4. **自愈检查** — 检查每个模板的 `{% endblock %}` 前是否有孤立 `</div>`（原结构残留），自动移除
5. **模板验证** — 检查所有模板的 `{% extends %}` 正确性（`pad_base` vs standalone vs fragment）
6. **全链路端点测试** — 14+ 学生端点全 200 + 布局检查（sidebar/main-area/no-phone CSS class）
7. **浏览器视觉确认** — 登录后截图关键页面确认布局

## 原型制作方法

1. 使用 **纯HTML+CSS** 制作可交互原型（不可用框架避免依赖）
2. 先做设计系统展示页（所有组件尺寸+色卡+布局示例）
3. 再按页面优先级逐一制作页面原型
4. 原型必须包含 **交互状态**（悬停/选中/正确/错误等）
5. 用 `file://` 或 `python3 -m http.server` 本地预览

## 常见陷阱

- ❌ 直接把手机版等比例放大 → 必须重新设计布局
- ❌ 保留手机版代码不删 → 移除双视图，只维护一套
- ❌ 按钮只改宽不改高 → 儿童触摸需要大高度(56px+)
- ❌ 用滚动条解决宽度问题 → 用分栏/网格利用宽度
- ❌ 先改代码再做原型 → 先出原型确认方向再实施
- ❌ 侧栏内容过多 → 保持核心5项导航，底部放状态信息

## 参考文件

- `references/pad-prototype-design-spec.md` — 完整设计规范文档（含Token体系、页面映射表）
