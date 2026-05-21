---
name: html-info-doc-production
category: design
description: 使用 HTML 信息图风格生成结构化文档/方案/报告。涵盖模板骨架、通用组件库、布局规范和工作流程。输出格式统一为暗色科技风（V01），保证所有生成文档风格一致。
tags: [html, design, info-graphic, document, report, template]
---

# HTML 信息图文档生产

## 定位

当需要向用户交付信息量大的结构化文档（方案、设计规格、分析报告、技术文档）时，使用本 skill 生成 HTML 格式的信息图风格文档。

**替代场景**：纯文字 Markdown 表格 → HTML 信息图卡片化展示

## 核心原则

1. **统一风格**：所有文档使用暗色科技风（V01），CSS 变量从 `visual-style-registry` 引用
2. **骨架先行**：先搭完整 HTML 结构（header → TOC → 章节 → footer），再填充内容
3. **信息密集优先**：卡片式布局 > 纯文字 > 表格 > 列表，按信息密度降序
4. **视觉层次清晰**：标题→章节标题→子标题→内容，字号递减，缩进递增
5. **禁止文字表格**：数据关系用 HTML 表格（带 card 样式），禁止 Markdown 纯文字表格

## 模板骨架

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{文档标题}}</title>
<style>
  /* === CSS变量（暗色科技风 V01） === */
  :root {
    --bg-lv0: #0a0a0f;
    --bg-lv1: #12121a;
    --bg-lv2: #1a1a28;
    --bg-lv3: #22223a;
    --border: #2a2a45;
    --text-pri: #e0e0f0;
    --text-sec: #9090b0;
    --accent-blue: #4a9eff;
    --accent-green: #4ae0a0;
    --accent-yellow: #f0c040;
    --accent-red: #ff4a6a;
    --accent-purple: #b080ff;
    --accent-cyan: #40e0f0;
  }
  /* === Reset & Base === */
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { font-family: -apple-system, 'PingFang SC', 'Noto Sans SC', sans-serif;
         background: var(--bg-lv0); color: var(--text-pri); line-height: 1.7; padding: 0; }
  .container { max-width: 1000px; margin: 0 auto; padding: 40px 24px 80px; }

  /* === 头部 === */
  .header { border-bottom: 1px solid var(--border); padding-bottom: 24px; margin-bottom: 40px; }
  .header h1 { font-size: 28px; font-weight: 700;
    background: linear-gradient(135deg, var(--accent-blue), var(--accent-purple));
    -webkit-background-clip: text; -webkit-text-fill-color: transparent; margin-bottom: 12px; }
  .header .meta { display: flex; gap: 20px; flex-wrap: wrap; font-size: 13px; color: var(--text-sec); }

  /* === TOC === */
  .toc { background: var(--bg-lv2); border: 1px solid var(--border); border-radius: 12px;
         padding: 20px 24px; margin-bottom: 28px; }
  .toc h3 { font-size: 14px; color: var(--text-sec); margin-bottom: 12px; }
  .toc ol { padding-left: 20px; }
  .toc ol li { margin: 4px 0; font-size: 14px; }
  .toc ol li a { color: var(--accent-blue); text-decoration: none; }

  /* === 章节卡片 === */
  .section { background: var(--bg-lv1); border: 1px solid var(--border); border-radius: 12px;
             padding: 28px; margin-bottom: 28px; }
  .section h2 { font-size: 20px; font-weight: 600; margin-bottom: 20px; padding-bottom: 12px;
    border-bottom: 1px solid var(--border); display: flex; align-items: center; gap: 10px; }
  .section h2 .badge { font-size: 11px; font-weight: 500; padding: 2px 8px; border-radius: 4px;
    background: var(--accent-blue); color: #000; }
  .section h3 { font-size: 16px; font-weight: 600; margin: 24px 0 12px; color: var(--accent-blue); }
  .section h4 { font-size: 14px; font-weight: 600; margin: 16px 0 8px; color: var(--accent-cyan); }
  .section p, .section li { font-size: 14px; line-height: 1.8; }

  /* === 代码/信息图块 === */
  .diagram { background: var(--bg-lv2); border: 1px solid var(--border); border-radius: 8px;
    padding: 20px; font-family: 'JetBrains Mono', 'Fira Code', monospace;
    font-size: 12.5px; line-height: 1.6; white-space: pre; overflow-x: auto; margin: 16px 0; }

  /* === 表格 === */
  table { width: 100%; border-collapse: collapse; margin: 16px 0; font-size: 14px; }
  th { background: var(--bg-lv3); color: var(--accent-blue); font-weight: 600;
    text-align: left; padding: 10px 14px; border: 1px solid var(--border); }
  td { padding: 10px 14px; border: 1px solid var(--border); vertical-align: top; }
  tr:hover td { background: var(--bg-lv2); }

  /* === 圆点 === */
  .dot { display: inline-block; width: 8px; height: 8px; border-radius: 50%; margin-right: 6px; }

  /* === 标签 === */
  .tag { display: inline-block; font-size: 11px; padding: 2px 8px; border-radius: 4px; font-weight: 500; }
  .tag-green { background: rgba(74,224,160,0.15); color: var(--accent-green); }
  .tag-yellow { background: rgba(240,192,64,0.15); color: var(--accent-yellow); }
  .tag-red { background: rgba(255,74,106,0.15); color: var(--accent-red); }
  .tag-blue { background: rgba(74,158,255,0.15); color: var(--accent-blue); }
  .tag-purple { background: rgba(176,128,255,0.15); color: var(--accent-purple); }

  /* === 卡片网格 === */
  .card-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 16px; margin: 16px 0; }
  .card { background: var(--bg-lv2); border: 1px solid var(--border); border-radius: 8px; padding: 18px; }
  .card h4 { font-size: 14px; font-weight: 600; margin-bottom: 8px; display: flex; align-items: center; gap: 8px; }
  .card p { font-size: 13px; color: var(--text-sec); line-height: 1.6; }

  /* === 标注块 === */
  .callout { padding: 14px 18px; border-radius: 8px; margin: 16px 0; font-size: 14px; }
  .callout-warn { background: rgba(240,192,64,0.1); border-left: 3px solid var(--accent-yellow); }
  .callout-info { background: rgba(74,158,255,0.1); border-left: 3px solid var(--accent-blue); }
  .callout-danger { background: rgba(255,74,106,0.1); border-left: 3px solid var(--accent-red); }
  .callout-success { background: rgba(74,224,160,0.1); border-left: 3px solid var(--accent-green); }

  /* === 页脚 === */
  .footer { text-align: center; color: var(--text-sec); font-size: 12px; margin-top: 40px;
    padding-top: 20px; border-top: 1px solid var(--border); }

  /* === 响应式 === */
  @media (max-width: 640px) {
    .container { padding: 20px 16px 60px; }
    .section { padding: 20px; }
    .card-grid { grid-template-columns: 1fr; }
  }
</style>
</head>
<body>
<div class="container">
  <!-- TITLE -->
  <div class="header">
    <h1>{{标题}}</h1>
    <div class="meta">
      <span>📄 版本: {{版本}}</span>
      <span>📅 生成: {{日期}}</span>
      <span>✍️ {{作者}}</span>
    </div>
  </div>
  <!-- TOC -->
  <div class="toc"> <h3>📋 目录</h3> <ol> {{TOC条目}} </ol> </div>
  <!-- 章节 -->
  {{章节内容}}
  <!-- FOOTER -->
  <div class="footer">{{页脚信息}}</div>
</div>
</body>
</html>
```

## 组件使用规范

| 组件 | 用途 | 何时使用 | 何时不用 |
|------|------|----------|----------|
| `.header` | 文档标题 + 元信息 | 每个文档顶部 | 短报告不用 meta |
| `.toc` | 目录索引 | 章节 ≥ 3 且内容量 ≥ 2000 字 | 短文档跳过 |
| `.section` | 主章节容器 | 每个独立章节 | 非章节内容（封面等） |
| `.diagram` | ASCII 流程图/代码块 | 流程说明、架构图、代码示例 | 连续文本段落 |
| `table` | 结构化数据对比 | 多行多列对比数据 | 单列列表（用 ul） |
| `.card-grid` | 并行信息卡片 | 3-6 条平行信息 | 需要顺序阅读的内容 |
| `.callout` | 高亮/提醒 | 关键结论、警示、用户原话 | 正文段落 |
| `.tag` | 状态/标签 | 标注进度、状态、优先级 | 正常文字强调 |
| `.dot` | 层级/状态圆点 | 配合 table 或 card 使用 | 独立不配文字时 |
| `.badge` | 章节标题旁标注 | 标注章节类型 | 非标题位置 |

## 工作流

### Step 1: 收集信息
- 确认文档主题、目标读者、核心信息点
- 收集所有需要呈现的结构化数据
- **可选**：如有PPTX源文件需要分析配色/风格，参见 `references/pptx-style-analysis.md`

### Step 2: 设计骨架
- 确定章节结构（h2 级别）
- 确定每个章节内使用哪些组件
- 规划信息密度：优先用卡片网格，其次表格，最后列表

### Step 3: 编写 HTML
- 从模板骨架复制到 `write_file`
- 填充 CSS 变量部分（从 `visual-style-registry` 的 V01 暗色科技风引用）
- 按章节顺序填充内容

### Step 4: 视觉检查
- 检查标题层级是否合理（h2 → h3 → h4 递减）
- 检查各组件是否使用正确（callout vs card vs table）
- 检查颜色标记是否一致（⚠️ 黄色警告 / ℹ️ 蓝色信息 / ✅ 绿色成功 / ❌ 红色错误）
- 检查响应式布局（@media query）

### Step 5: 交付

> 🔴 **路由规则**：所有生成的文档必须上传到飞书云盘 `Hermes生成文件` 对应的子目录。参照 `feishu-drive-directory-standard` 技能的6大板块目录结构确定目标位置。

- 将 HTML 保存到 `~/.hermes/output/` 或 `~/.hermes/cache/` 下
- 上传到飞书云盘对应目录（见目录路由规则）
- 分享给用户 + 发送链接到对话框
- **两步缺一不可**：上传 + 发链接
- **回复简洁**：HTML 已通过飞书链接交付时，回复仅需简短确认 + 链接，不得重复 HTML 中的内容

### Step 5b（可选）：PDF 输出

当用户需要 PDF 版本（打印、分发、存档）时，遵循以下工作流：

**⚠️ 关键陷阱**：`wkhtmltopdf`（v0.12.x）基于旧版 WebKit，**无法正确渲染暗色科技风主题**（SVG 不显示、黑色背景异常、布局错位）。**不能直接用暗色主题 HTML 转 PDF**。

**正确流程**：

1. **创建浅色打印版 HTML** — 从原始内容复制一份，替换为白底黑字主题：
   - 背景 `#ffffff` → 文字 `#1a1a2e`
   - 表格行交替色 `#f8f9fb` / `#ffffff`
   - SVG 填充改为浅木色系（`#e8dcc8` / `#f0e6d4`），描边调整为深色（`#8b7355`）
   - 移除渐变、阴影等 wkhtmltopdf 不支持的效果
   - 可添加 `.page-break { page-break-before:always; }` 控制分页
   - 参见 `references/print-theme-template.md` 获取完整主题模板

2. **用 wkhtmltopdf 转换**：
   ```bash
   wkhtmltopdf --enable-local-file-access \
     --page-size A4 \
     --margin-top 10mm --margin-bottom 10mm \
     --margin-left 8mm --margin-right 8mm \
     --no-stop-slow-scripts \
     /path/to/print-version.html \
     /path/to/output.pdf
   ```

3. **上传 PDF** — 与 HTML 文档一起上传到同一飞书文件夹，作为配套文件。

**为什么不直接用其他工具**：
- `weasyprint`：依赖系统库较多，pip 安装经常超时（60s+）
- 无头 Chromium：WSL 环境通常未安装
- `python-pdfkit`：底层仍调用 wkhtmltopdf，无额外优势

## 常见陷阱

1. **CSS 样式遗漏**：每次从模板骨架开始，不要手动拼 CSS。如果已存 skill 的模板有更新，以 skill 为准。
2. **ASCII 图过长**：diagram 块尽量控制在 20-30 行以内，太长会导致滚动困难。考虑拆分为多个小图。
3. **卡片网格阅读顺序**：card-grid 在移动端会变成 1 列，确保每个 card 自包含、不依赖左右顺序。
4. **表格内容溢出**：表格列数控制在 4 列以内，过多列时考虑转用 card-grid。
5. **颜色过度使用**：标签颜色用于强调，一页不超过 4 种颜色，否则视觉混乱。
6. **忘记响应式**：写完检查 `.card-grid` 和 `table` 在窄屏下的表现。
