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

## 更新规范

| 信号 | 动作 |
|:-----|:-----|
| 用户说「配色改一下」 | 新增变体或修改现有Token |
| 新场景出现（如汇报风→运营风） | 新增V0N记录 |
| 用户对已有风格表达偏好修正 | 更新对应Token值，标注变更日期 |
