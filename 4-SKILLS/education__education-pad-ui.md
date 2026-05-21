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

### 色彩（教育类推荐暖白+紫，实际运行已确认使用 清透蓝白）

**正式确认（2026-05-21 学生端视觉重设计评估）：** 学生端使用 **清透蓝白** 色系而非紫色暖白：
- 背景 `#f0f6ff`，卡片 `#ffffff`
- 主色 `#4f7cff`（天空蓝），辅色 `#ff8a65`（暖橙）
- 文字 `#2d3436` / 辅助 `#636e72`
- 边框 `#e0e7ff`
- 成功 `#66bb6a` / 金币 `#f59e0b` / 错误 `#e53935`

**执行规则：** 所有学生端子模板的 `:root` CSS变量块必须与上述色值保持一致。`pad.css` 的 `--accent` 变量应设为 `#4f7cff`（而非 `#6c63ff`）。如果新增页面，直接引用 `pad.css` 的Token系统，不要在内联 `<style>` 中重复定义 `:root` 变量。

⚠️ **已知冲突**：设计Token层使用紫色 `#6c63ff`，但实际已实现的学生端页面（`home.html`）使用蓝色 `#4f7cff`。两套不同色系同时在代码库中存在，需要统一决策。

**设计Token层颜色（pad.css）：**
- 主背景：`#f5f0e8` 暖白
- 卡片背景：`#ffffff`
- 深色侧栏：`#1a1a2e` 深空蓝黑
- 侧栏文字：`rgba(255,255,255,0.85)` / 次要 `0.45`
- 主色（accent）：`#6c63ff` 紫色 → 按压 `#5a52e0`
- 成功：`#4caf50`
- 警告：`#f39c12`
- 危险：`#e57373`

**实际学生端页面颜色（home.html / chinese.html 的 `:root`）：**
- 主色（primary）：`#4f7cff` 天空蓝
- 辅色：`#ff8a65` 暖橙
- 强调色：`#66bb6a` 绿色
- 背景：`#f0f6ff` 浅蓝
- 文字主色：`#2d3436`
- 辅助文字：`#636e72`
- 边框：`#e0e7ff`

**冲突影响：**
| 元素 | 设计Token | 实际页面 | 差异 |
|------|-----------|---------|------|
| 主色 | #6c63ff (紫) | #4f7cff (蓝) | ❌ 完全不同 |
| 背景 | #f5f0e8 (暖白) | #f0f6ff (浅蓝) | ❌ 不同 |
| 按钮主色 | --accent | --primary | ❌ 两个CSS变量 |

**修复建议：** 统一为实际已在运行的学生端配色（蓝色系 `#4f7cff` + 浅蓝背景），将设计Token作为「下次大版本升级」目标。

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

### 儿童答题反馈动画（9岁用户专用）

答题反馈对儿童用户至关重要 — 纯文字不够，需要视觉动画产生成就感/安全感。

**正确反馈 — 星星/粒子从底部弹入：**
```css
@keyframes celebration-pop {
  0% { transform: scale(0) translateY(20px); opacity: 0; }
  60% { transform: scale(1.15) translateY(-4px); opacity: 1; }
  100% { transform: scale(1) translateY(0); opacity: 1; }
}
.animate-celebration {
  animation: celebration-pop 0.5s cubic-bezier(0.34, 1.56, 0.64, 1) both;
}
```

**获得金币 — 从题目位置向上飘到顶部：**
```css
@keyframes coin-fly {
  0% { opacity: 1; transform: translateY(0) scale(1); }
  100% { opacity: 0; transform: translateY(-80px) scale(0.5); }
}
.animate-coin-fly {
  position: fixed;
  z-index: 999;
  pointer-events: none;
  animation: coin-fly 0.8s ease-out both;
}
```

**错误反馈 — 轻微抖动（非惩罚性）：**
```css
@keyframes shake-gently {
  0%, 100% { transform: translateX(0); }
  20% { transform: translateX(-6px); }
  40% { transform: translateX(6px); }
  60% { transform: translateX(-4px); }
  80% { transform: translateX(4px); }
}
.animate-shake {
  animation: shake-gently 0.4s ease;
}
```

**连续打卡庆祝 — 火焰/星星 burst：**
```css
@keyframes streak-burst {
  0% { box-shadow: 0 0 0 0 rgba(255, 167, 38, 0.5); transform: scale(1); }
  50% { box-shadow: 0 0 0 20px rgba(255, 167, 38, 0); transform: scale(1.1); }
  100% { box-shadow: 0 0 0 0 rgba(255, 167, 38, 0); transform: scale(1); }
}
```

**获得称号 — 全屏弹窗 pop-in：**
```css
@keyframes title-reveal {
  0% { transform: scale(0) rotate(-10deg); opacity: 0; }
  50% { transform: scale(1.3) rotate(3deg); opacity: 1; }
  100% { transform: scale(1) rotate(0deg); opacity: 1; }
}
```

**设计原则：** 动画时长≤0.8s，ease-out 缓出，不干扰答题流程。9岁儿童注意力短，动画要快而强，不要慢而优雅。

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

**陷阱：base 的 url_for() 一次崩溃全站**

`pad_base.html` 的任何 `url_for()` 被所有子模板共享。一个不存在端点的 `url_for()` 会导致 **所有继承它的页面 BuildError → 500**。

```jinja
{# ❌ student.profile 不存在，所有页面崩溃 #}
<a href="{{ url_for('student.profile') }}">个人中心</a>

{# ✅ 不存在就用 # 占位 #}
<a href="#">个人中心</a>
```

**预防**：每次在 pad_base 中添加/修改导航项时，先确认端点是否存在：
```bash
grep -n "def $(echo 'student.profile' | cut -d. -f2)" routes/student.py
```

### 模板迁移模式：独立 HTML → pad_base 继承

当系统初始开发时，学生端模板可能以独立HTML编写（不含 base 继承）。后续统一为 Pad 侧栏布局时，需要进行**系统性模板迁移**。

#### 迁移步骤

每页精确转换：

| 原始独立HTML | → | pad_base 继承模板 |
|---|---|---|
| `<!DOCTYPE html>` | → | `{% extends 'pad_base.html' %}` |
| `<title>xxx</title>` | → | `{% block title %}xxx{% endblock %}` |
| `<style>...</style>` | → | `{% block head_extra %}<style>..</style>{% endblock %}` |
| 页面内容 | → | `{% block content %}...{% endblock %}` |
| `<script>...</script>` | → | `{% block scripts %}<script>..</script>{% endblock %}` |
| `</body></html>` | → | 删掉 |

**block 清单**（每个 block 可选，顺序规范）：

| block | 用途 | 必需 |
|-------|------|------|
| `title` | `<title>` 标签内容 | 推荐 |
| `page_id` | 侧栏导航高亮标识 | 推荐（空则为不亮） |
| `head_extra` | `<style>` + 额外 `<meta>` | 样式多时必用 |
| `content` | 主内容区 | ✅ |
| `scripts` | `<script>` JS 代码 | 有JS时必用 |

#### 迁移注意事项

1. **移除后退按钮** — 侧栏已提供导航，`<a class="back-btn">←</a>` 应移除
2. **body 的 flex 容器** — 独立页面的 `body{display:flex;flex-direction:column;padding:16px}` 需要迁移到一个内容容器的 class（如 `.practice-container`）内
3. **feedback-overlay** — 使用 `position:fixed` 的弹窗不受侧栏影响，直接留在 `{% block content %}` 内即可
4. **`url_for()` 检查** — 迁移后模板中的 `url_for('xxx')` 必须全部有对应路由
5. **`student` 变量由 context_processor 注入** — pad_base.html 不从 route 获取 student，检查 `routes/student.py` 的 context_processor 确保已注入

#### 验证方法

迁移后对每个页面检查三要素：

```python
for url in pages:
    resp = c.get(url)
    assert resp.status_code == 200
    assert b'pg-layout' in resp.data     # pad 布局骨架
    assert b'BuildError' not in resp.data # 无路由构建错误
    assert b'Traceback' not in resp.data  # 无 Python 异常
```

同时测试**原路由路径**正常（向前兼容），并检查 **nav 高亮**是否匹配 page_id。

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
- ❌ 子模板用固定 max-width 覆盖布局 → 不要在子模板的 `<style>` 中写 `.container { max-width: 480px }` 这类固定宽约束。即使是 body 级容器，也应使用相对值（pad 用 `780px`，移动端用 `100%`），否则侧栏布局在 iPad 上失效。子模板样式应该**补充/增强**父模板布局，而不是覆盖它。
- ❌ Pad 页面中重复功能入口 → 当首页已内联展示某功能（如月历网格），不要在快捷入口区再放一个同名图标。Pad 端信息密度更高，用户能同时看到多个区域，重复入口变成视觉冗余。
- ❌ 子页面 page_id 错误或不匹配 → 每个子模板必须用 `{% block page_id %}xxx{% endblock %}` 定义唯一的正确值，且必须精确匹配 `pad_base.html` 侧栏导航项的第一个 tuple 元素。如果 page_id 错用其他页面的值（例如 `chinese.html` 用了 `wrong_book`），会导致侧栏高亮错误指向。修复方法：grep 所有 student 模板的 `page_id` block 逐一核对。
- ❌ Pad 子页面同时有侧栏和「‹ 返回」按钮 → 侧栏本身提供了多级导航，子页面顶部再放「‹ 返回」按钮在 Pad 端是冗余的。对于 pad 视图（`@media min-width: 768px`），应设置 `.back-btn { display: none; }`。手机端（无侧栏）保留后退按钮作为唯一导航方式。
- ❌ 自定 padding 覆盖按钮高度，跌破最小触摸目标 → 当按钮使用自定的 `padding: 8px 16px` 而非 `--btn-h` 系列变量时，实际高度可降至 40px（小按钮需要 48px，默认 56px）。规则：所有交互按钮必须使用 `.pg-btn` / `.pg-btn.large` / `.pg-btn.small` 等组件类，禁止在按钮元素上覆写 padding。如果必须自定义，最小高度用 `min-height: 56px` 兜底。
- ❌ 双View 模板残留 → 当执行「移除双视图」重构后，phone_frame 和 pad_frame 的 HTML 结构可能残留（不可见但仍在 DOM 中）。验证方法：审查元素检查 `.phone-frame` `.pad-frame` `.frame-label` 是否存在。批量清理命令：`grep -rn 'phone-frame\|pad-frame\|frame-label' templates/student/` 找出所有残留。

## 学生端 vs 家长端设计哲学

当系统同时存在学生端（做练习）和家长端（管理内容）时，**二者不是同一套 UI 的简配/增强版关系**，而是从不同命题出发的平行设计。

| 维度 | 家长端（管理型） | 学生端（任务型） |
|:-----|:-----------------|:-----------------|
| 核心命题 | 管理内容、查看报告、配置系统 | 完成练习、获得反馈、积累成就 |
| 导航模式 | 侧栏 + 7 模块导航 | 手机底部 5Tab / Pad 侧栏 |
| 视觉风格 | 清透蓝白 · 信息密度高 | 暖色调 · 大图标 · 低信息密度 |
| 交互动力 | 点击浏览 | 任务驱动：今天做什么？ |
| 激励体系 | 无（管理者不需要） | 积分/称号/连击/庆祝动画 |
| 页面流 | 树形目录（进→退→进） | 线性闭环（课文→答题→庆祝→下一课） |

### 学生端 5Tab 底部导航规范

手机端优先底部导航（替代侧栏），让核心入口始终在拇指可达域：

| Tab | 图标 | 页面 | 说明 |
|:----|:-----|:-----|:------|
| 首页 | 🏠 | 小书房首页 | 一屏显示今日计划 + 四宫格入口 |
| 课程 | 📖 | 课文选择 | 按课展示三态进度（已做/待做/未学） |
| 学习 | 📊 | 报告/日历/历史 | 子 tab 切换 |
| 荣誉 | 🏆 | 荣誉墙/商城 | 称号 + 积分消费 |
| 我的 | 👤 | 个人中心 | 统计 + 称号 + 设置 + 退出 |

Pad 端保留侧栏，内容与底部 Tab 对齐。

### 学生端最小闭环页面流

```
课文选择 → 答题页 → 🎉 庆祝页 → 继续学习 / 📕 错题本 → 🏠 首页
```

每个环节的设计要点：

| 环节 | 核心设计 | 关键元素 |
|:-----|:---------|:---------|
| **课文选择** | 网格展示每课卡片，三态标记 | 已完成(绿色) / 有练习待做(橙色高亮) / 尚未学习(灰色) |
| **答题页** | 进度条 + 题型徽标 + 手写/选择/填空 | 清除干扰，聚焦当前题 |
| **庆祝页** | 得分动画 + 金币动画 + 题型分布 + AI评语 | 大号 emoji + 糖果色统计卡片 |
| **错题本** | 逐题展示，带消灭操作 | 红色错答案 + 绿色正确答案 + 重新练习按钮 |

### 学生端配色系统（暖色调 · 儿童友好）

区别于家长端的清透蓝白，学生端采用暖基调 + 高饱和点缀：

```
页面背景: #f0f6ff      卡片底色: #ffffff
主色调:   #4f7cff      辅色:     #ff8a65 (Badge)
成功色:   #66bb6a      金币色:   #f59e0b
强调色:   #6c5ce7      文字主色: #2d3436
错误色:   #e53935      辅助文字: #636e72
```

### 课文选择页设计模式

这是学生端新增的核心页面，按课展示学习进度。推荐**网格卡片 + 三态颜色编码**：

```html
<div class="lesson-card" style="border-left-color: #4f7cff">
  <div class="l-number">第 18 课</div>
  <div class="l-name">童年的水墨画</div>
  <span class="l-status ls-done">✅ 已完成</span>
</div>

<!-- 高亮：有练习待做 -->
<div class="lesson-card" style="border-left-color: #f59e0b; box-shadow: 0 0 0 2px #f59e0b33">
  <span class="l-status ls-now" style="background: #fff3e0; color: #f57c00">⏳ 有练习待做</span>
</div>

<!-- 置灰：尚未学习 -->
<div class="lesson-card" style="border-left-color: #888; opacity: 0.6">
  <span class="l-status ls-new">尚未学习</span>
</div>
```

需要后端聚合 SQL：`practice_schedule` JOIN `question_bank` GROUP BY `lesson_number` → 判定每课状态。

### 完成庆祝页设计模式

提交全部答案后的新结果页，替代旧版纯数字报告。核心组件：

1. **大号得分**（48-72px font-size）+ 星星等级（⭐×N）
2. **题型分布卡片** — 每题型一个小卡片（正确率 + 进度条 + 颜色编码）
3. **金币获得动画** — 暖色背景 + 大号数字（🪙 +26）
4. **AI 评语卡片** — `#f8f6ff` 底色紫色提示（💡 会认字选读音需要多多练习哦！）
5. **操作按钮** — 继续学习 / 查看错题 / 返回首页

```html
<div style="font-size: 72px; margin-bottom: 12px">🎉</div>
<div style="font-size: 48px; font-weight: 800; color: #4f7cff">85%</div>
<div style="font-size: 24px; margin-bottom: 12px">⭐⭐⭐</div>
```

### 学习日历设计模式

月视图日历，四色编码：

| 颜色 | 含义 | 色值 |
|:-----|:-----|:------|
| 🟢 绿色 | 已做练习 | `#e8f5e9` 背景 |
| 🔴 红色 | 有练习未做 | `#fde8e8` 背景 |
| 🔵 蓝色 | 今日 | `border: 2px solid #4f7cff` |
| ⬜ 灰色 | 未来日期 | `color: #ccc` |

点击日期 → 展开当天练习列表（练习名称 + 状态 + 完成数）。

## 参考文件

- `references/pad-prototype-design-spec.md` — 完整设计规范文档（含Token体系、页面映射表）
- `references/student-experience-design.md` — 学生端体验设计完整方法论（含14页原型、导航架构、色彩系统、页面流）
- `references/student-page-audit-20260521.md` — 学生端24个模板的逐页UX/前端审计（含配色冲突、page_id错误、双View残留清单、消毒命令）
- `references/student-redesign-prototype-v1.md` — 学生端视觉重设计方案v1.0索引（三角色评估结论+11页原型+4阶段实施计划），全量HTML原型在同一目录下的 student-redesign-v1.html
