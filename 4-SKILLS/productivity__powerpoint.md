---
name: powerpoint
description: "Create, read, edit .pptx decks, slides, notes, templates."
license: Proprietary. LICENSE.txt has complete terms
---

# Powerpoint Skill

## When to use

Use this skill any time a .pptx file is involved in any way — as input, output, or both. This includes: creating slide decks, pitch decks, or presentations; reading, parsing, or extracting text from any .pptx file (even if the extracted content will be used elsewhere, like in an email or summary); editing, modifying, or updating existing presentations; combining or splitting slide files; working with templates, layouts, speaker notes, or comments. Trigger whenever the user mentions "deck," "slides," "presentation," or references a .pptx filename, regardless of what they plan to do with the content afterward. If a .pptx file needs to be opened, created, or touched, use this skill.

## Quick Reference

| Task | Guide |
|------|-------|
| Read/analyze content | `python -m markitdown presentation.pptx` |
| Edit or create from template | Read [editing.md](editing.md) |
| Create from scratch | Read [pptxgenjs.md](pptxgenjs.md) |

---

## Reading Content

```bash
# Text extraction
python -m markitdown presentation.pptx

# Visual overview
python scripts/thumbnail.py presentation.pptx

# Raw XML
python scripts/office/unpack.py presentation.pptx unpacked/
```

---

## Editing Workflow

**Read [editing.md](editing.md) for full details.**

1. Analyze template with `thumbnail.py`
2. Unpack → manipulate slides → edit content → clean → pack

---

## Creating from Scratch

**Read [pptxgenjs.md](pptxgenjs.md) for full details.**

Use when no template or reference presentation is available.

---

## Design Ideas

**Don't create boring slides.** Plain bullets on a white background won't impress anyone. Consider ideas from this list for each slide.

### Before Starting

- **Pick a bold, content-informed color palette**: The palette should feel designed for THIS topic. If swapping your colors into a completely different presentation would still "work," you haven't made specific enough choices.
- **Dominance over equality**: One color should dominate (60-70% visual weight), with 1-2 supporting tones and one sharp accent. Never give all colors equal weight.
- **Dark/light contrast**: Dark backgrounds for title + conclusion slides, light for content ("sandwich" structure). Or commit to dark throughout for a premium feel.
- **Commit to a visual motif**: Pick ONE distinctive element and repeat it — rounded image frames, icons in colored circles, thick single-side borders. Carry it across every slide.

### Color Palettes

Choose colors that match your topic — don't default to generic blue. Use these palettes as inspiration:

| Theme | Primary | Secondary | Accent |
|-------|---------|-----------|--------|
| **Midnight Executive** | `1E2761` (navy) | `CADCFC` (ice blue) | `FFFFFF` (white) |
| **Forest & Moss** | `2C5F2D` (forest) | `97BC62` (moss) | `F5F5F5` (cream) |
| **Coral Energy** | `F96167` (coral) | `F9E795` (gold) | `2F3C7E` (navy) |
| **Warm Terracotta** | `B85042` (terracotta) | `E7E8D1` (sand) | `A7BEAE` (sage) |
| **Ocean Gradient** | `065A82` (deep blue) | `1C7293` (teal) | `21295C` (midnight) |
| **Charcoal Minimal** | `36454F` (charcoal) | `F2F2F2` (off-white) | `212121` (black) |
| **Teal Trust** | `028090` (teal) | `00A896` (seafoam) | `02C39A` (mint) |
| **Berry & Cream** | `6D2E46` (berry) | `A26769` (dusty rose) | `ECE2D0` (cream) |
| **Sage Calm** | `84B59F` (sage) | `69A297` (eucalyptus) | `50808E` (slate) |
| **Cherry Bold** | `990011` (cherry) | `FCF6F5` (off-white) | `2F3C7E` (navy) |
| **Modern Minimalist**⚡ | `1A3C6E` (deep blue) | `F2F4F7` (light gray) | `E04F16` (orange-red) |

> ⚡ **Modern Minimalist** is the user's preferred business style for executive presentations. See `references/modern-minimalist-business-style.md` for the complete spec including fonts, layouts, page footers, templates (S-T-A-R case study, 3-phase strategy, personal commitment page), and icon conventions.

### For Each Slide

**Every slide needs a visual element** — image, chart, icon, or shape. Text-only slides are forgettable.

**Layout options:**
- Two-column (text left, illustration on right)
- Icon + text rows (icon in colored circle, bold header, description below)
- 2x2 or 2x3 grid (image on one side, grid of content blocks on other)
- Half-bleed image (full left or right side) with content overlay

**Data display:**
- Large stat callouts (big numbers 60-72pt with small labels below)
- Comparison columns (before/after, pros/cons, side-by-side options)
- Timeline or process flow (numbered steps, arrows)

**Visual polish:**
- Icons in small colored circles next to section headers
- Italic accent text for key stats or taglines

### Typography

**Choose an interesting font pairing** — don't default to Arial. Pick a header font with personality and pair it with a clean body font.

| Header Font | Body Font |
|-------------|-----------|
| Georgia | Calibri |
| Arial Black | Arial |
| Calibri | Calibri Light |
| Cambria | Calibri |
| Trebuchet MS | Calibri |
| Impact | Arial |
| Palatino | Garamond |
| Consolas | Calibri |

| Element | Size |
|---------|------|
| Slide title | 36-44pt bold |
| Section header | 20-24pt bold |
| Body text | 14-16pt |
| Captions | 10-12pt muted |

### Spacing

- 0.5" minimum margins
- 0.3-0.5" between content blocks
- Leave breathing room—don't fill every inch

### Avoid (Common Mistakes)

- **Don't repeat the same layout** — vary columns, cards, and callouts across slides
- **Don't center body text** — left-align paragraphs and lists; center only titles
- **Don't skimp on size contrast** — titles need 36pt+ to stand out from 14-16pt body
- **Don't default to blue** — pick colors that reflect the specific topic
- **Don't mix spacing randomly** — choose 0.3" or 0.5" gaps and use consistently
- **Don't style one slide and leave the rest plain** — commit fully or keep it simple throughout
- **Don't create text-only slides** — add images, icons, charts, or visual elements; avoid plain title + bullets
- **Don't forget text box padding** — when aligning lines or shapes with text edges, set `margin: 0` on the text box or offset the shape to account for padding
- **Don't use low-contrast elements** — icons AND text need strong contrast against the background; avoid light text on light backgrounds or dark text on dark backgrounds
- **NEVER use accent lines under titles** — these are a hallmark of AI-generated slides; use whitespace or background color instead

---

## QA (Required)

**Assume there are problems. Your job is to find them.**

Your first render is almost never correct. Approach QA as a bug hunt, not a confirmation step. If you found zero issues on first inspection, you weren't looking hard enough.

### Content QA

```bash
python -m markitdown output.pptx
```

Check for missing content, typos, wrong order.

**When using templates, check for leftover placeholder text:**

```bash
python -m markitdown output.pptx | grep -iE "xxxx|lorem|ipsum|this.*(page|slide).*layout"
```

If grep returns results, fix them before declaring success.

### Visual QA

**⚠️ USE SUBAGENTS** — even for 2-3 slides. You've been staring at the code and will see what you expect, not what's there. Subagents have fresh eyes.

Convert slides to images (see [Converting to Images](#converting-to-images)), then use this prompt:

```
Visually inspect these slides. Assume there are issues — find them.

Look for:
- Overlapping elements (text through shapes, lines through words, stacked elements)
- Text overflow or cut off at edges/box boundaries
- Decorative lines positioned for single-line text but title wrapped to two lines
- Source citations or footers colliding with content above
- Elements too close (< 0.3" gaps) or cards/sections nearly touching
- Uneven gaps (large empty area in one place, cramped in another)
- Insufficient margin from slide edges (< 0.5")
- Columns or similar elements not aligned consistently
- Low-contrast text (e.g., light gray text on cream-colored background)
- Low-contrast icons (e.g., dark icons on dark backgrounds without a contrasting circle)
- Text boxes too narrow causing excessive wrapping
- Leftover placeholder content

For each slide, list issues or areas of concern, even if minor.

Read and analyze these images:
1. /path/to/slide-01.jpg (Expected: [brief description])
2. /path/to/slide-02.jpg (Expected: [brief description])

Report ALL issues found, including minor ones.
```

### Verification Loop

1. Generate slides → Convert to images → Inspect
2. **List issues found** (if none found, look again more critically)
3. Fix issues
4. **Re-verify affected slides** — one fix often creates another problem
5. Repeat until a full pass reveals no new issues

**Do not declare success until you've completed at least one fix-and-verify cycle.**

---

### Large Decks: Delegate to a Subagent

For presentations with 10+ slides (500+ lines of JS), **delegate script generation to a subagent** rather than writing inline in `execute_code`. Rationale:

- A 14-slide PPT script is typically 1000-2000 lines — this pollutes your context and makes debugging harder
- Subagent handles its own `write_file → node run → fix → rerun` loop without bloating your turn count
- Pass the complete slide spec (content, layout, colors, fonts) in the `context` parameter
- Load the PowerPoint skill in the subagent's `toolsets` so it has access to `pptxgenjs.md` reference
- After delegation, verify with `python3 -c "from pptx import Presentation; prs = Presentation('path.pptx'); print(len(prs.slides))"` and `python3 -m markitdown path.pptx` for content

**Workaround for markitdown empty output**: Use the Python API directly:
```python
python3 -c "
from markitdown import MarkItDown
md = MarkItDown()
result = md.convert('/path/to/file.pptx')
print(result.text_content[:3000])
"

Convert presentations to individual slide images for visual inspection:

```bash
# Preferred — using soffice.py wrapper (handles clean temp files)
python scripts/office/soffice.py --headless --convert-to pdf output.pptx

# Fallback — direct libreoffice (more widely available)
libreoffice --headless --convert-to pdf output.pptx

# Convert PDF to images
pdftoppm -jpeg -r 150 output.pdf slide
```

This creates `slide-01.jpg`, `slide-02.jpg`, etc.

To re-render specific slides after fixes:

```bash
pdftoppm -jpeg -r 150 -f N -l N output.pdf slide-fixed
```

---

## 竞聘材料 ⚠️ 内容陷阱与规范

**这是本技能最重要的规则** — 在接手任何竞聘/述职/晋升PPT任务前，必须通读并执行。

### 致命陷阱：把部门成绩当个人标签

```
❌ 错误做法：部门/KPI数据
    "续保应收112.2亿，达成率99.6%"
    "策略转化率提升40-50%"
    评委反应：这是你们团队的成绩，不是你的个人标签

✅ 正确做法：个人贡献数据
    "主导5家险企核心系统建设"
    "管理48人产品团队"
    "获金融科技发展奖三等奖"
    评委反应：这个人有真本事
```

**黄金法则**：如果评委用这个数字反问"这是你个人的还是你们部门的？"你答不上来 → 这个数字不应该出现。

### 竞聘材料三问（输出前必须自检）

1. **这个数字代表谁？** → 必须是候选人个人主导/负责/参与的事情
2. **这个数字能验证吗？** → 有奖项、有署名、有360评分佐证更好
3. **少一个数字会影响说服力吗？** → 如果会，保留；如果不会，删掉

### 竞聘材料的正确结构

```
封面 → 我是谁（1页）
个人核心能力（1页） → 3个标签，每句话都是"我"
个人关键业绩（1页） → 2-3个个人数字，其余概括描述
岗位理解+规划（1页） → 我对这个角色的认知
个人承诺（1页） → 对应每位评委
团队/文化（1页） → 如果你是管理者
封底 → 一句话收尾
```

**总页数建议**: 7页以内。竞聘不是汇报工作，是卖自己。越少越好。

### 竞聘数据的来源规范

从用户获取信息时，必须明确区分：

| 数据类型 | 可用？ | 说明 |
|:---------|:------|:------|
| 个人带团队规模 | ✅ 可用 | "管理48人产品团队" |
| 个人主导的项目数 | ✅ 可用 | "主导5家险企核心系统建设" |
| 个人获奖/排名 | ✅ 可用 | "360排名前11"、"获金融科技奖" |
| 个人工作年限 | ✅ 可用 | "18年+保险行业经验"、"4年平安经验" |
| 部门KPI/营收 | ❌ 不可用 | 属于部门成绩 |
| 业务转化率 | ❌ 不可用 | 属于业务团队成绩 |
| 成本节约金额 | ❌ 不可用 | 除非是个人主导的具体项目 |
| 接手前vs接手后 | ❌ 不可用 | 用户明确禁用对比式表述 |

### 正确的内容模板（来自用户样本）

```
标题: 岗位匹配度与业绩贡献

个人业绩与经验优势	| 个人能力与技能优势	| 个人特质与态度优势
----------------------|----------------------|----------------------
• 主导了X个核心系统建设	| • 具备XX专业能力	| • 长期从事一线业务
• X年+行业经验		| • 主导XX架构升级	| • 快速学习能力
• X年+团队管理经验	| • 跨部门协同能力	| • 危机公关能力
• 荣获XX奖项		| • 资源整合能力	| • 企业文化认同
```

每个条目以个人主语开头：主导了·负责了·荣获了·具备·擅长

---

## Templates in this Directory

| Template | Description |
|----------|-------------|
