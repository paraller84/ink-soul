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

## 更新规范

| 信号 | 动作 |
|:-----|:-----|
| 用户说「配色改一下」 | 新增变体或修改现有Token |
| 新场景出现（如汇报风→运营风） | 新增V0N记录 |
| 用户对已有风格表达偏好修正 | 更新对应Token值，标注变更日期 |
