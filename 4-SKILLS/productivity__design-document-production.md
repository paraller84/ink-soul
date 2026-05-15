---
name: design-document-production
category: productivity
description: 系统化HTML设计文档生产工作流 — 页面清单分析→原型生成→版本合并→飞书上传。适用于UI/UX系统、教育平台、管理后台等任何需要多页面原型+设计规格说明的场景。
tags: [design, prototype, html, feishu, ui-ux, design-doc]
---

# HTML 设计文档生产工作流

## 定位

当需要为系统/功能创建**完整设计说明书**（含页面原型、数据模型、API设计、规则说明等）时使用。区别于一次性HTML原型（用 `sketch`/`claude-design`），本工作流面向**多页面、多版本、需迭代**的大规格设计文档。

## 触发条件

用户要求设计/重设计一个系统的完整方案，且明确或暗示：
- 需要看到所有页面原型
- 需要手机+Pad/PC双端展示
- 需要输出为HTML（飞书可预览）
- 需要多版本迭代

## 核心产出物

一个自包含的HTML文件（暗色科技风主题），包含：

| 章节 | 内容 |
|:----|:------|
| 项目定位 | 一句话定位、教材依据、技术栈 |
| 完整页面清单 | 所有页面的编号/名称/角色/分类/原型状态 |
| 全部页面原型 | 每个页面的手机端+Pad端双端设计 |
| 数据模型 | 核心JSON结构设计 |
| API设计 | 关键API端点 |
| 规则说明 | 积分/称号/出题等策略的完整规则 |
| 分期计划 | 实施路线图 |

## 工作流

### Phase 0: 调研现有系统

在开始原型设计前，必须先调研：
1. 现有系统的路由和页面 — 用代码阅读或终端命令提取
2. 现有系统的数据模型 — 查看 database schema 或 content.json
3. 现有系统的OCR/视觉识别能力 — 若有拍照导入需求
4. 相关的用户偏好和约束（从 memory 读取）

```
# 提取路由示例
grep -rn "@app.route\|@.*\.route\|\.add_url_rule" app.py --include='*.py'
```

### Phase 1: 页面清单分析

第一步永远是列出**完整页面清单**，而非直接画原型。

```
ALL_PAGES = {
  # 学生端
  "1": {"name": "系统入口", "role": "公共", "category": "导航"},
  "2": {"name": "学科选择", "role": "学生", "category": "导航"},
  # 家长端
  ...
}
```

按角色（学生/家长/公共）和分类（导航/学习/激励/管理/安全/个人/工具/参考）归类。

对每个页面标记状态：
- **已有** — v1.0中已有完整原型
- **新增** — 本版本新增页面
- **增强** — 已有但本版本大幅重做

生成一个表格作为设计文档的第一个实质性章节（Section 2）。

### Phase 2: 原型生成策略

⚠️ **关键原则 — 避免单次生成过大：**

35+页面的大规模原型会超过模型上下文限制。使用子代理并行生成策略：

```
策略:
1. 主代理: 读取现有HTML，确定需要哪些新页面（Phase 1产出）
2. 子代理1: 生成页面A-F（6个新页面）
3. 子代理2: 生成页面G-H（2个新页面）
4. 子代理3: 生成增强版页面（替换旧版本）
5. 主代理: 用Python合并所有片段
```

**子代理委托模式：**
```python
delegate_task(
    context="...",  # 包含所有设计规范和上下文
    goal="生成8个缺失页面原型的HTML",
    toolsets=["file"]
)
```

每个子代理输出独立的HTML片段文件，供后续合并。

### Phase 3: HTML 合并策略

使用Python字符串操作（非手动编辑）将所有片段合并为完整HTML：

```python
# 策略: 按section边界拆分v1.0，替换/插入片段
# 1. 找到section边界
sec2_start = v1.find('<div class="section"><h2>2. 全部页面原型')
sec2_end = v1.find('<div class="section"><h2>3. 答题反馈')

# 2. 拆分为part_a（header+section1）、pages（section2）、part_c（剩余）
part_a = v1[:sec2_start]
part_c = v1[sec2_end:]

# 3. 替换受损页面（旧page11-12 → 新）
old_p11 = v1.find('<h3>11. 积分规则</h3>')
# ... 精确找到页面卡片边界 ...

# 4. 组装
v21 = part_a
v21 += inventory_table    # 新页面清单
v21 += '<div class="section"><h2>3. 全部页面原型</h2>'
v21 += page_styles
v21 += pages_1_10
v21 += enhanced_11_12
v21 += pages_13_27
v21 += new_pages_A_H
v21 += '</div>'
v21 += part_c
```

**关键边界定位技巧：**
- 用 `rfind('<div class="page-card">', 0, target)` 找card起始
- 用 `find('<div class="page-card">', target)` 找card结束
- 用 `find('</div>', target, limit)` 逐级定位闭合标签
- 写一个验证循环：`for check in checks: assert check in result`

### Phase 4: 版本迭代

```python
# 版本号更新
v1 = v1.replace('v2.0 完整最终版', 'v2.1 完整版 · 35页面')
v1 = v1.replace('27页', '35页 · 8页新增')
v1 = v1.replace('<span>手机+Pad双端 · 27页全原型</span>', 
                '<span>v2.1 35页面</span><span>手机+Pad双端 · 全题型</span>')
```

### Phase 5: 飞书上传

```python
# 使用 drive/v1/files/upload_all API
folder_token = get_token("Hermes生成文件")
headers = {'Authorization': f'Bearer {access_token}'}

with open(file_path, 'rb') as f:
    files = {'file': (file_name, f.read(), 'text/html')}
data = {'file_name': file_name, 'parent_type': 'explorer', 
        'parent_node': folder_token, 'size': str(file_size)}

resp = requests.post(
    'https://open.feishu.cn/open-apis/drive/v1/files/upload_all',
    headers=headers, data=data, files=files
)
```

✅ **完成后必须共享文件给用户：**
```python
requests.post(
    f'https://open.feishu.cn/open-apis/drive/v1/permissions/{file_token}/members?type=file',
    headers=headers,
    json={'member_type': 'openid', 'member_id': user_openid, 'perm': 'full_access'}
)
```

## CSS 主题规范

使用暗色科技风主题：

```css
:root {
  --bg-primary: #0a0e1a;
  --bg-secondary: #111827;
  --bg-card: #1a1f2e;
  --text-primary: #e2e8f0;
  --text-secondary: #94a3b8;
  --accent: #3b82f6;
  --border-color: #2a3148;
}
```

页面原型使用暖白底色（`#fdf8f0`）模拟真实设备屏幕。

## 页面原型框架

每个页面同时展示 **手机端 (375px)** 和 **Pad端 (600px)**：

```html
<div class="page-card">
  <h3>N. 页面名称</h3>
  <div class="device-row">
    <div class="device-col">
      <div class="phone-frame">
        <div class="screen"><div class="inner">...内容...</div></div>
      </div>
    </div>
    <div class="device-col">
      <div class="pad-frame">
        <div class="screen"><div class="inner">...内容...</div></div>
      </div>
    </div>
  </div>
</div>
```

## Phase 6: 基于用户反馈的迭代更新（vN → vN+1）

当设计文档已有完整版本（v1.x+），用户给出结构化反馈后需迭代到下一版，走以下流程：

### Step 1: 结构化反馈收集

用户可能给出**编号清单**式反馈（如36条）。逐条标注动作：

```
📋 用户反馈处理矩阵

| # | 反馈 | 动作 | 原因 |
|:-:|:-----|:-----|:------|
| 1 | 不要TTS | ❌ 忽略 | 3年级已能独立阅读 |
| 2 | 前置知识校验 | ✅ 立即执行 | 教育专家P0建议 |
| 3 | OCR用qwen3-vl:8b | ✅ 架构变更 | 简化GPU管理 |
| ... | ... | ... | ... |
| 12 | 迁移策略后续 | ⏸️ 二期 | 当前无用户 |

三种动作：
- ✅ **立即执行** — 修改方案中的文字/设计
- ⏸️ **二期** — 标记在方案中但不纳入当前实施
- ❌ **忽略** — 从方案中删除相关内容
```

对用户说 **"不太明白"** 的项，先解释再执行，不要跳过。

### Step 2: 规划变更范围

按变更类型归类：

| 类型 | 示例 | 操作方式 |
|:-----|:------|:---------|
| **文字/章节描述** | 出题流程、SM-2参数、UX规范 | Python字符串替换或子代理文本补丁 |
| **数据模型** | 新增实体、修改字段 | 更新数据模型表格 |
| **页面说明** | 页面描述、page-desc | 修改对应文本 |
| **架构决策** | OCR引擎、AI模型、系统关系 | 替换/删除相关章节描述 |
| **页面原型自身** | phone+pad端UI HTML结构 | ❌ **不修改**（只改文字描述） |

### Step 3: 子代理委托的精确指令

这是最容易出错的步骤。⚠️ **子代理收到"生成vN+1"指令时，容易生成一个缩略版本（65KB）而非完整版本（445KB）**。必须显式禁止：

```python
# ✅ 正确的子代理委托指令
delegate_task(
    context=f"""
    原始完整HTML路径: v2.1.html（445KB, 35页面原型, phone+pad双端）
    
    任务: 读取全文后，应用以下文本变更生成v2.2
    
    ⚠️ 关键约束（违反将导致返工）:
    1. 必须读取原始v2.1全文（分多次read_file，每2000行）
    2. 只修改文字描述（章节标题、段落、表格、page-desc）
    3. ❌ 不要修改页面原型的phone-frame/pad-frame HTML结构
    4. ❌ 不要从头重写——要在v2.1基础上应用文本变更
    5. 目标文件大小应与源文件相当（~400-450KB）
    
    变更清单:
    - OCR: TrOCR双通道 → qwen3-vl:8b单模型（修改OCR章节描述）
    - AI: 本地qwen → 云端模型（修改AI出题描述）
    - 页面数: 35 → 28（修改所有版本号和计数引用）
    - ...
    """
)
```

**错误模式识别：**
| 症状 | 根因 | 修复 |
|:-----|:------|:------|
| 输出文件 65KB vs 源文件 445KB | 子代理没读全量HTML，自己"概括"了 | 显式要求读源文件+保持规模 |
| 只有3个 page-card vs 应有40个 | 子代理重写了HTML而非编辑 | 约束"不修改页面原型HTML结构" |
| 缺少phone/pad双端 | 子代理认为"精简"是对的 | 要求完整保留所有原型 |

### Step 4: 验证

```python
checks = {
    'page-card count': content.count('<div class="page-card">'),    # 应与源文件一致
    'phone-frame count': content.count('phone-frame'),
    'pad-frame count': content.count('pad-frame'),
    'version': 'v2.2' in content,
    'new text': '前置知识' in content,  # 确认新增文本存在
    'removed text': 'TrOCR' not in content,  # 确认删除文本不存在
    'file size': size > 400000,  # 应与源文件相当
}
for k, v in checks.items():
    sym = '✅' if v else '❌'
    print(f'  {sym} {k}: {v}')
```

### Step 5: 一致性审计（必做 — 用户明确要求的步骤）

在用户反馈全部应用到方案后、上传飞书之前，必须执行**系统性交叉审计**。用户2026-05-14明确批评了方案中页面清单与原型不一致的问题。

#### 5维交叉审计清单

| # | 维度 | 检查方法 | 典型失败案例 |
|:-:|:-----|:---------|:------------|
| 1 | **清单 vs 原型** | 页面清单每项 → 用 `content.count('<h3>N. 名称')` 验证存在 | 清单35页但原型只有27个编号 |
| 2 | **编号一致性** | h3编号、清单编号、section引用编号三者一致 | A/B/C字母编号 vs 1/2/3数字编号混用 |
| 3 | **删除内容全覆盖** | 用户说去掉某个功能/模式 → `content.count('关键词')` 验证=0（不只看描述，还要看phone/pad frame内的HTML文字） | "综合练习"从描述删了但phone frame内还显示 |
| 4 | **CSS/主题一致性** | 合并自子代理的片段 → 全文 `grep` 检查是否混入不同主题的CSS变量名（如 `--text-primary` vs `--text`） | OCR章节用暗色主题，其他用暖白主题 |
| 5 | **页面原型完整性** | `content.count('<div class="page-card">')` 应=清单数+其他非原型卡片（如知识点说明） | 子代理只输出了3个page-card而非全部 |

```python
# 审计执行模板
import re
with open('output.html', 'r') as f:
    content = f.read()

# 1. 提取清单中的页面数
inv_start = content.find('完整页面清单')
inv_pages = len(re.findall(r'<td>(\d+)</td>', content[inv_start:inv_start+5000]))

# 2. 提取原型h3标题
prototypes = re.findall(r'<h3>(\d+)\.\s*[^<]+</h3>', content)
prototype_nums = [int(x) for x in prototypes]

# 3. 对每个用户确认"去掉"的关键词做全覆盖检查
for keyword in ['综合练习', '要删除的模式名']:
    count = content.count(keyword)
    if count > 0:
        print(f"⚠️ '{keyword}' 还有 {count} 处残留！")

# 4. 检查主题一致性——不同子代理的主题变量
for var in ['--text-primary', '--bg-primary']:
    if var in content and '--text' not in content:
        print(f"⚠️ 混入了其他主题的CSS变量: {var}")

# 5. page-card完整性
pc_count = content.count('<div class="page-card">')
print(f"page-card: {pc_count}")
```

如果审计发现问题 → **不要上传**，先修复再重审。

#### 编号修正策略

当页面增加/删除导致编号错位时：

1. **删除一个页面**（如打卡日历嵌入首页）：该页面对应的page-card HTML块直接删除；其后的所有页面h3编号和前移到清单表一并更新
2. **页面顺序调整**：在清单表中调整顺序；对应调整原型部分的h3编号
3. **字母编号修正**：子代理生成的新页面常以A/B/C字母编号。合并后必须改为数字编号并插入正确位置

```python
# 编号修正示例
rename_map = {
    '<h3>A. 学科选择</h3>': '<h3>3. 学科选择</h3>',
    '<h3>B. 练习模式选择</h3>': '<h3>4. 练习模式选择</h3>',
}
for old, new in rename_map.items():
    assert content.count(old) == 1, f'{old} should appear exactly once'
    content = content.replace(old, new)
```

#### 风格一致：全文档使用统一主题

当从子代理合并OCR/导入等章节时，子代理可能使用不同的CSS主题。必须做CSS变量名映射：

```python
# 子代理的暗色主题 → 全局暖白主题 变量映射
theme_map = {
    'var(--bg-primary)': '#fdf8f0',
    'var(--bg-secondary)': '#f5f0e8',
    'var(--bg-card)': '#ffffff',
    'var(--border-color)': '#e0d8cc',
    'var(--text-primary)': '#2c2c2c',
    'var(--text-secondary)': '#666666',
    # ... 完整映射
}
for old, new in theme_map.items():
    merged_content = merged_content.replace(old, new)
```

### Step 6: 上传

与 Phase 5 相同 — 使用 drive/v1/files/upload_all 上传到飞书，并共享给用户。

---

## ⚠️ 设计转生产：清理框架伪影

### 关键原则

设计文档中的 phone-frame/pad-frame 双端并排展示模式是**设计阶段的产物**，绝不能让它们出现在最终落地的生产代码中。

### 识别泄漏信号

当生产环境中部署的页面出现以下特征，说明设计原型伪影泄漏：

| 信号 | 原文 | 根因 |
|:-----|:------|:------|
| 页面上下堆叠两个版本 | 手机框 + 平板框同时渲染 | 模板直接复用了设计文档的 frame-container 结构 |
| 显示"📱 手机视图 · 375px"等标签 | frame-label 显示设计标注 | 未去除设计阶段的标记文字 |
| 手机打开时两种布局混杂 | 两个 frame 上下排列 | 缺少设备检测逻辑 |

### 受影响组件

每个设计原型页面都包含：
- `.frame-container` — 包含两个子 frame 的容器
- `.phone-frame` / `.phone-inner` — 手机端 (375px) 框架
- `.pad-frame` / `.pad-inner` — 平板端 (600px) 框架
- `.frame-label` — 设计标注（如"📱 手机视图 · 375px"）

### 修复模式：CSS 媒体查询法（最小改动）

不修改模板 HTML，仅在 CSS 中添加响应式规则，让设备宽度决定显示哪个版本：

```css
/* 设计标注 — 生产环境隐藏 */
.frame-label { display: none; }

/* 手机 (≤480px): 显示 phone-frame，隐藏 pad-frame */
@media (max-width: 480px) {
  .pad-frame { display: none; }
  .phone-frame { display: block; }
}

/* 平板/桌面 (>480px): 显示 pad-frame，隐藏 phone-frame */
@media (min-width: 481px) {
  .phone-frame { display: none; }
  .pad-frame { display: block; }
}

/* 移除 design 阶段的 spacing */
.frame-container { display: block; padding: 0; }
```

### 更好的长期方案

CSS 隐藏方案虽然快，但两个版本的内容仍在 DOM 中。长期应：

1. **服务端检测**：在 Flask 的 `before_request` 中检测 `User-Agent`（或前端通过 JS 跳转），只渲染对应的模板
2. **或统一为单套响应式设计**：不再维护两套布局，用 CSS Grid/Flexbox + `container queries` 自适应
3. **设计文档生成阶段即标记**：在 HTML 模板中为双端原型添加 `data-role="design-only"` 属性，生产部署前用脚本批量剥离

### 验证清单

生产部署前逐一检查：

- [ ] 页面上没有 `.frame-label` 元素显示设计标注
- [ ] 手机端打开只显示 mobile 版本，不显示 pad 版本
- [ ] 平板/桌面端打开只显示 pad 版本
- [ ] 两种设备的核心功能（登录、做题、查看报告）均可正常操作

## 常见陷阱

### 1. 子代理将"更新文档"误解为"重写文档"

当前会话（2026-05-14 v2.1→v2.2）已验证陷阱：第一次委托子代理生成v2.2时，输出65KB（仅含3个页面原型）而非预期的445KB。根因：
- 子代理未读全量源文件
- 子代理认为"精简版"足够
- 缺少显式约束"不修改页面原型HTML结构"

**修复：** 第二次委托时的指令明确包含5条关键约束+预期文件规模下限，输出正确（448KB/28页面/全部phone+pad原型）。

### 3. 子代理输出CSS主题不一致

子代理可能生成不同主题的CSS。合并前需要：
- 统一CSS变量名（替换 OCR 主题的 `--text-primary` → `--text` 等）
- 确保页面组件类名一致（`.pg-card`/`.pg-btn` 等）

```python
ocr_adapted = ocr_content
ocr_adapted = ocr_adapted.replace('var(--text-primary)', 'var(--text)')
ocr_adapted = ocr_adapted.replace('var(--bg-secondary)', 'var(--bg2)')
# ... 对所有不兼容的变量名做替换
```

### 4. 页面边界定位不准

用 `find`/`rfind` 定位闭合标签时容易offset不准。建议：
- 先写验证断言
- 输出定位点附近的上下文确认
- 使用 `v2.count('<div class="page-card">')` 确认总数

### 3. 版本号遗漏

HTML的版本号可能出现在3-4处（header标题、副标题、footer、meta等）。用替换法统一更新。

### 4. 文件过大超过飞书限制

445KB HTML文件上传正常（<20MB）。但如果包含base64图片可能导致超大文件。大图片使用URL引用而非base64。

### 5. 页面清单与原型编号不一致

页面清单更新后，原型部分的编号必须同步更新。在Phase 1的清单表中直接列出所有页面，原型部分保持相同编号。

## 验证清单

- [ ] 所有页面在清单中有记录
- [ ] 清单中的"原型状态"与实际存在匹配
- [ ] h3编号与清单编号一致
- [ ] 每个页面有手机+Pad双端
- [ ] 版本号在所有位置一致
- [ ] 飞书上传后可以访问
- [ ] 飞书已共享给用户
- [ ] CSS主题统一，无颜色断裂

## 参考

- `sketch` skill — 一次性HTML原型（不适合大规格设计文档）
- `claude-design` skill — 单页面HTML设计（不适合35+页系统）
- `feishu-drive-sync-pipeline` — 飞书Drive同步管道
- `references/structured-feedback-iteration.md` — 实战案例：基于36条结构化反馈从v2.1迭代到v2.2（含反馈处理矩阵、子代理委托教训、变更执行策略）
- `references/html-merge-technique.md` — HTML大文件合并技术详解（section拆分、边界定位、CSS变量兼容）
