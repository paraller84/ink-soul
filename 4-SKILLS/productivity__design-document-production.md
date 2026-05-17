---
name: design-document-production
category: productivity
description: 系统化HTML设计文档生产工作流 — 页面清单分析→原型生成→版本合并→飞书上传。适用于UI/UX系统、教育平台、管理后台等任何需要多页面原型+设计规格说明的场景。
tags: [design, prototype, html, feishu, ui-ux, design-doc]
---

# HTML 设计文档生产工作流

## 定位

当需要为系统/功能创建**完整设计说明书**（含页面原型、数据模型、API设计、规则说明等）时使用。区别于一次性HTML原型（用 `sketch`/`claude-design`），本工作流面向**多页面、多版本、需迭代**的大规格设计文档。

## 7 种设计文档类型总览

设计任务可能产出以下 **7 种类型**之一或多个。本 skill 主要覆盖类型 A/B/C/G，类型 D 见 `education-pad-ui`，类型 E 见 `html-info-doc-production`，类型 F 见 `systematic-testing`。但所有类型的内容要求、格式规范、交付纪律以本 skill 为统一标准。

| # | 类型 | 适用场景 | 输出格式 |
|:-:|:-----|:---------|:---------|
| **A** | 系统蓝图设计说明书 | 系统级完整方案，含全部页面+数据+API+规则+分期 | HTML 信息图 |
| **B** | 数据结构/数据模型 | 表结构+ER关系+DDL+迁移，三项强制门禁（见数据模型节） | HTML 信息图 |
| **C** | 业务规则/算法方案 | 出题策略/积分规则/难度自适应等纯逻辑设计 | HTML 信息图 |
| **D** | UI 交互原型 | 可操作的 HTML 原型，用户测试用（独立文件集） | 独立 HTML 文件集 |
| **E** | 系统架构设计 | 服务拓扑+技术选型+成本+风险+双轨回滚 | HTML 信息图 |
| **F** | 测试方案设计 | E2E 用例+黄金数据+门禁+CI | 飞书 docx/HTML |
| **G** | 物理/机械零件图纸 | 物理实物（航模/工具/3D打印等）的独立零件三视图 | HTML 信息图 |

> 📎 **标准化模板参考**：`references/design-doc-template-v1.0.html` — 内含每类文档的完整必须内容清单、格式规范、陷阱和实例。

## 触发条件

用户要求设计/重设计一个系统的完整方案，且明确或暗示：
- 需要看到所有页面原型
- 需要手机+Pad/PC双端展示
- 需要输出为HTML文档（飞书可预览或本地查看）
- 需要多版本迭代
- 需要输出**数据模型/ER关系/架构设计**的结构化HTML文档（无UI原型需求）

## 用户格式偏好（2026-05-16 明确确认）

信息量大、需要结构型表达的内容使用 **HTML 格式（暗色科技风主题）** 输出，而非纯文字表格。此偏好适用于所有技能和场景，不限于设计文档。

## 两个交付模式

⚠️ **关键区别 — 用户偏好的区分：**

| 模式 | 交付物 | 适用场景 | 用户用词 |
|:-----|:-------|:---------|:---------|
| **A: 单一大设计文档** | 1个HTML包含全部页面phone+pad双端 + 数据模型/API/规则 | 系统级完整设计说明书 | "设计文档""系统方案" |
| **B: 独立可交互原型** | N个独立HTML文件，各对应一个页面/场景，可交互 | UI原型确认、用户测试 | "给我看原型""HTML输出" |
| **C: 数据结构/架构文档** | 1个自包含HTML，以表格+ER图+DDL为主，无phone/pad框架 | ER关系、数据库设计、系统架构 | "数据结构设计""数据模型" |

**模式B的核心纪律（基于用户纠正）：**
- **HTML文件本身是主要交付物**，不要只传截图+写文档描述
- 每个HTML是自包含的独立原型（可浏览器直接打开、点击交互）
- 上传HTML文件到飞书文件夹（用 `upload_all` API）
- 飞书文档只是**辅助说明**（概述+对照表），不是主交付物
- 用户实际是在浏览器中打开HTML文件测试交互，不是在飞书文档中看描述

## 模式B工作流

### Phase B0: 准备
1. 确定需要原型的页面清单
2. 确定设计规范（色彩/尺寸/布局模式）
3. 创建独立HTML目录（如 `pad-prototypes/`）

### Phase B1: 生成HTML原型
每个原型是独立的、可交互的HTML文件：
```html
<!-- 自包含的HTML原型，可直接双击打开 -->
<!DOCTYPE html>
<html>
<head>
  <style>/* 完整样式，无外部依赖 */</style>
</head>
<body>
  <!-- 可交互的UI，带JavaScript行为 -->
  <!-- 含hover/click/状态切换等交互反馈 -->
</body>
</html>
```

### Phase B2: 交付到飞书
1. 创建文件夹（`项目` → `Pad端设计原型 YYYY-MM-DD`）
2. 上传所有HTML文件 → 上传PNG截图作为预览参考
3. 创建飞书文档 → 写辅助说明（概述+各页对照表+交互要点）
4. 分享文件夹和文档给用户

### 模式B陷阱
- ❌ 不要只传截图到飞书，把HTML留在本地
- ❌ 不要在飞书文档中长篇描述每个页面（HTML原型自身就是最好的描述）
- ❌ 不要创建单一大HTML（模式B是独立文件，每个原型自包含）
- ✅ 飞书文档应该短小精悍：核心设计原则 + HTML文件对照表

## 模式C工作流：数据结构/架构设计文档

### 适用场景

当用户需求聚焦于**数据模型、表结构、ER关系、架构设计**等非UI内容时使用。输出为1个自包含HTML，以表格+ER图+SQL DDL为主，不包含phone/pad框架。

### Phase C0: 触发识别

| 用户用语 | 示例 |
|:---------|:------|
| "数据结构设计" | "先完成数据结构的设计" |
| "数据模型" | "数据库表结构是什么" |
| "ER关系" | "表间关系，是否存在冗余或关系不清" |
| "系统架构" | "三库隔离架构" |

### Phase C1: 格式规范

使用暗色科技风主题，以表格为主，避免phone/pad框架：

```css
/* 数据结构文档主题 */
body { background: #0d1117; color: #c9d1d9; }
h2 { color: #f0883e; border-left: 4px solid #f0883e; }
h3 { color: #7ee787; }
code { background: #161b22; color: #ffa657; }
table { border: 1px solid #30363d; }
th { background: #161b22; color: #8b949e; }
tr:hover td { background: #161b22; }
pre { background: #161b22; border: 1px solid #30363d; }
.note { background: #1f6feb0d; border-left: 3px solid #58a6ff; }
.warn { background: #d299220d; border-left: 3px solid #ffa657; }
```

### Phase C2: 内容结构

```
1. 架构总览（三库/模块关系图）
2. ER关系图（每库的表间关系，用缩进或ASCII表示）
3. 每张表的字段明细（表名、列、类型、中文释义、约束、唯一键、示例）
4. 完整DDL（可直接执行的CREATE TABLE + CREATE INDEX）
5. 关键约束和业务规则（唯一键粒度、默认值、兼容策略）
6. 数据流（OCR导入映射、练习引擎出题链路）
7. 迁移方案（旧表→新表、默认值兼容、幂等执行）
8. 已消除的设计缺陷对照表
```

### ⚠️ 数据模型表文档质量门禁（2026-05-16 用户纠正）

设计数据模型表时，**每一张表**必须满足以下三个条件——缺一不可：

**条件一：完整字段定义**

每列必须包含：字段名、类型、中文释义、约束、示例值。格式示例：

| 字段名 | 类型 | 中文释义 | 约束 | 示例 |
|:-------|:-----|:---------|:-----|:-----|
| lesson_number | INTEGER | **课号**，全册唯一（1~28） | NOT NULL | 18 |
| recognition_type | TEXT | **认字类型**：'recognize'=会认, 'write'=会写 | DEFAULT 'recognize' | 'write' |

**条件二：表和字段的中文释义**

- 每张表必须有中文说明段落（`<p class="note">存储...</p>`）
- 每个字段必须有中文释义列
- 唯一键需附业务含义说明（如「同学生同年级下课号唯一」）
- 字段中的枚举值需全部列出（如 recognition_type: 'recognize' / 'write'）

**条件三：ER 关系 + 完整表覆盖**

- ER 关系图中**必须列出本文档中定义的所有表**，不能遗漏
- 新增表后必须同步更新 ER 图
- 验证方法：grep ER 图中的表名总数 vs DDL 中的表名总数，应一致

**失败案例（来自真实会话）：**
- ❌ 缺少中文释义 → 「设计中还是缺字段和表的中文解释」
- ❌ ER 图遗漏 training_chars/training_words → 与 DDL 不一致
- ❌ 缺少课文全文表 → 「缺少了课文全文信息的记录」

**自查清单：**

```
□ 每张表的字段都有「中文释义」列
□ 每张表前有用途说明段落
□ 枚举字段列出所有可能值
□ UNIQUE 约束有业务语义解释
□ ER 图列出了文档中全部表（用 grep 对比表名列表与 ER 图内容）
□ 从业务场景反推：用户做练习时需要哪些数据关联？是否全部在表中？
  — 例如：课本库 → 需要（单元/课/字/词/课文全文）
  — 漏掉课文全文 → 用户会指出
```

## 模式G工作流：物理/机械零件图纸

### 适用场景

当需求涉及**物理实物**（航模、机器人、工具架、3D打印件、机械结构等）需要制作技术图纸时使用。输出为 HTML 信息图，包含每个零件的独立三视图。

### ⚠️ 关键区分：分别的零件图 ≠ 组装总装图（2026-05-17 用户纠正）

**这是物理图纸设计中最容易犯的错误，也是用户明确纠正过的点：**

| 概念 | 含义 | 用户用词 |
|:-----|:------|:---------|
| **分别的示意图** | 每个零件单独画，各自标注尺寸/材料/工艺 | "分别的示意图" |
| **组装在一起的示意图** | 多个零件拼在一起，显示装配关系 | "组装在一起的示意图" |

**默认规则**：除非用户明确要求"总装图"或"装配图"，**优先输出分别的零件图**——每个零件从多个角度独立绘制，包含完整的尺寸标注。

### Phase G0: 触发识别

| 用户用语 | 示例 |
|:---------|:------|
| "飞机模型"、"航模"、"零件" | "把每个零件画出来" |
| "机械图纸"、"蓝图"、"技术图纸" | "我要分别的示意图" |
| "制作/加工/裁切" | "这个零件怎么裁" |

### Phase G1: 零件清单梳理

列出所有需要画图的零件，确认每个零件的材料、数量、尺寸范围：

```markdown
| # | 零件名 | 材料 | 大致尺寸 | 数量 |
|:-:|:-------|:-----|:---------|:----:|
| A | 主翼 | 木板 | 展250×弦52→35mm | 1块 |
| B | 平衡翼 | 木板 | 100×25mm | 1块 |
```

### Phase G2: 生成零件图纸

**每个零件必须满足以下三视图标准（缺一不可）：**

| 视图 | 代号 | 展示内容 | 必须标注 |
|:-----|:----|:---------|:---------|
| **俯视图** | ◩ 俯视 | 从正上方看到的形状（轮廓+对称性） | 翼展/宽度/弦长/梯形收窄 |
| **前视图** | ▤ 前视 | 从正前方看到的形状（截面/厚度/对称） | 截面尺寸/厚度/对称线 |
| **侧视图** | ▣ 侧视 | 从侧面看到的形状（翼型/厚度渐变） | 弦长/最厚点/前缘后缘处理 |

**SVG 绘制规范：**

```svg
<!-- 每个视图用独立 SVG 画布，含以下元素： -->
<svg viewBox="0 0 200 140">
  <!-- 1. 零件轮廓 — 用突出版本颜色的填充+描边 -->
  <polygon points="..." class="part-fill-wood"/>
  <!-- 2. 中心线 — 虚线，用于对称零件 -->
  <line class="centerline"/>
  <!-- 3. 尺寸线 — 橙色/黄色，两端有短竖线 -->
  <line class="dim-line"/>
  <text class="dim-text">← 翼展 250mm →</text>
  <!-- 4. 标注文字 — 小号灰白文字 -->
  <text class="label-text">前缘磨圆</text>
</svg>
```

**统一颜色方案：**

```css
/* 零件填充 */
.part-fill-wood { fill:#3d2c1e; stroke:#d4a047; stroke-width:1; }
.part-fill-metal { fill:#2a3d5c; stroke:#54a0ff; stroke-width:1; }
.part-fill-plastic { fill:#1e3d2e; stroke:#2ed573; stroke-width:1; }

/* 标注 */
.centerline { stroke:#2a3148; stroke-width:0.7; stroke-dasharray:3,3; }
.dim-line { stroke:#ffaa00; stroke-width:0.8; }
.dim-text { font-family:'Courier New',monospace; font-size:8px; fill:#ffaa00; }
.label-text { font-family:'Segoe UI',sans-serif; font-size:9px; fill:#c8d6e5; }
```

### Phase G3: 组装关系（可选）

仅当用户明确要求"总装图"时，在零件图之后附加一个**总装关系图**，显示零件之间的相对位置和安装顺序。不画在零件图上。

### Phase G4: 交付物结构

```html
文档结构：
1. 标题 + 零件明细汇总表
2. 每个零件一个卡片，每卡片内3个视图（俯/前/侧）的grid
3. 每个视图有独立的尺寸标注
4. 每个零件卡片底部有裁切/制作注意事项
5. 最后附零件明细汇总表（材料/尺寸/数量/备注）
```

### 常见陷阱

- ❌ **零件图里包含装配信息** — 零件图只画该零件自己，不画其他零件
- ❌ **缺少尺寸标注** — 每张图必须有一到两个关键尺寸标注，用于裁切
- ❌ **三视图缺角** — 俯/前/侧缺任何一个，用户无法理解零件立体形状
- ❌ **不讲清楚"这是从上往下看还是从侧面看"** — 每个视图必须标注名称（◩ 俯视图 / ▤ 前视图 / ▣ 侧视图）
- ❌ **默认用户要的是总装图** — 用户说"示意图"时，先确认是"分别的零件图"还是"组装在一起的总装图"
- ✅ **每个零件一个独立 SVG 画布**，标注完整尺寸，标注文字用中文

> ⚠️ **设计文档生成后，两步缺一不可：**
> 1. **上传到飞书** — 上传 HTML/docx 到对应项目文件夹（不是只存本地）
> 2. **发送链接到对话** — 将文档的打开链接发送到当前对话（不只是说"已上传"）
>
> ❌ 不允许只生成本地文件 → 用户看不到
> ❌ 不允许只上传不发送链接 → 用户不知道在哪
> ❌ 不允许只发送链接不上传 → 文件不在云盘

**执行顺序**（所有模式通用）：
### Phase C3: 交付物位置

遵循默认规则——文件自动放入对应项目文件夹：

```项目根目录/
  └── docs/
      └── <topic>-design-v<version>.html
```

**三轨交付（强制执行，不可跳过）：**
- 本地：项目 `docs/` 目录（开发引用用）
- 飞书：对应项目文件夹（用户访问用）— 遵循 `feishu-drive-directory-standard` 路由规则
- KB同步：本地副本 → KB增量管道 → ChromaDB可检索（保存到 `~/.hermes/output/` 后cron自动同步）
- skill 引用：复制到技能 `references/` 目录（后续引用用）
- **不需要每次确认用户是否要飞书版，对于活跃项目默认自动上传**

## 🔴 交付目录路由规则（强制遵守）

> 所有生成的文件（HTML/PPT/docx/PNG等）必须放入 `Hermes生成文件` 对应的子目录。**不可直接上传到顶层或无关位置**。
> 
> 打开 `feishu-drive-directory-standard` skill 查看完整6大板块目录结构和路由规则表。
>
> **快速路由参考：**
> | 内容类型 | 目标目录 |
> |:---------|:---------|
> | 项目设计/架构/原型 | `Hermes生成/项目/` |
> | 工作汇报/EAST/竞聘 | `Hermes生成/工作汇报/` |
> | 顾问团报告/审计 | `Hermes生成/顾问团报告/` |
> | 日常运营/看板 | `Hermes生成/日常运营/` |
> | 个人/健康/教育 | `Hermes生成/个人/` |
> | 部门群产出自动归档 | `Hermes生成/部门群产出/` |

## 交付执行纪律（2026-05-16 用户反馈后加入）

> ⚠️ 两步缺一不可，违反后必须主动道歉并补做：
> 1. **上传到飞书** — 创建 docx 或上传 HTML 到对应项目文件夹
> 2. **发送链接到对话** — 将文档的打开链接发送到当前群聊/DM
>
> 不允许停留在本地、不允许只发路径不发链接、不允许只上传不发消息。

> **🔴 默认规则（2026-05-17 用户明确告知）：**
> 将文档放入对应项目文件夹 + 发送链接是**默认规则**，不需要每次重申。仅供存放位置有变更或例外情况时才需确认。
> 此规则已固化到记忆，无需在每次生成文档时复述一遍"我即将上传并发送链接"——直接执行即可。

**两种交付方式的选择：**

| 交付物类型 | 适用场景 | 飞书操作方式 |
|:-----------|:---------|:-------------|
| **飞书 Docx 文档** | 数据结构/架构/设计规范（需持续编辑） | POST /docx/v1/documents 创建 → feishu-md-writer.py 写内容 → share type=docx |
| **HTML/PNG 文件** | 原型截图、交互原型 | upload_all API (multipart) → share type=file |
| **文件夹** | 多个相关文件归集 | POST /drive/v1/files/create_folder → share type=folder |

**飞书 Docx 文档交付标准流程（2026-05-16 验证）：**

```python
# Step 1: 创建文档
body = json.dumps({"title": "文档名称 v3.0", "folder_token": project_folder_token}).encode()
req = urllib.request.Request("https://open.feishu.cn/open-apis/docx/v1/documents", data=body, headers=headers)
doc_token = json.loads(urllib.request.urlopen(req).read())["data"]["document"]["document_id"]

# Step 2: 写入内容（必须从 ~/.hermes/scripts/ 运行）
# cd ~/.hermes/scripts && python3 feishu-md-writer.py <doc_token> <md_path>
# 验证：用 docx API 检查 child blocks 数量 > 0

# Step 3: 分享给用户（type=docx）
body = json.dumps({"member_type": "openid", "member_id": user_openid, "perm": "full_access"}).encode()
req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx",
    data=body, headers=headers)

# Step 4: 分享给群聊（type=docx, member_type=openchat）
body = json.dumps({"member_type": "openchat", "member_id": group_chat_id, "perm": "view"}).encode()
req = urllib.request.Request(
    f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx",
    data=body, headers=headers)

# Step 5: 发送链接到对话
link = f"https://rcncjr4hag8s.feishu.cn/docx/{doc_token}"
```

**上传完成后的验证（必做）：**
```python
# 验证文档有内容
req = urllib.request.Request(f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}/blocks/{doc_token}", headers=headers)
children = json.loads(urllib.request.urlopen(req).read())["data"]["block"]["children"]
assert len(children) > 0, "❌ 文档内容为空！feishu-md-writer 可能失败"

# 验证分享完成
req = urllib.request.Request(f"https://open.feishu.cn/open-apis/drive/v1/permissions/{doc_token}/members?type=docx&page_size=50", headers=headers)
members = json.loads(urllib.request.urlopen(req).read())["data"]["items"]
assert any(m["member_id"] == user_openid for m in members), "❌ 未分享给用户"
```

**文件夹 token 漂移陷阱** — 先用 parent 文件夹列表 API 获取实际 token，不要依赖注册表中的缓存值：

```python
def resolve_folder_token(folder_name, parent_token, headers):
    url = f"https://open.feishu.cn/open-apis/drive/v1/files?folder_token={parent_token}&page_size=100"
    resp = json.loads(urllib.request.urlopen(urllib.request.Request(url, headers=headers)).read())
    for f in resp["data"]["files"]:
        if f["type"] == "folder" and f["name"] == folder_name:
            return f["token"]
    return ""
```

### 模式C陷阱
- ❌ 不要加入phone/pad框架——数据结构文档不需要UI原型
- ❌ 不要冗余的单元/课说明——核心是表结构和关系
- ❌ 不要跳过ER关系直接写DDL——关系图是理解基础
- ❌ 不要忽略UDI key和索引策略——这是实际可执行DDL的关键部分
- ✅ 用 `UNIQUE(...)` 和 `CREATE INDEX` 显式声明约束

### 参考实例

- `ai-family-tutor-data-architecture` skill 的 `references/data-structure-design-v3.html` — 三库独立设计的最佳实践案例

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

> 📎 独立规范文件: `references/design-audit-checklist.md` — 包含完整审计表格 + 执行代码模板 + 跨skill引用说明

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

### 2. 子代理输出CSS主题不一致

子代理可能生成不同主题的CSS。合并前需要：
- 统一CSS变量名（替换 OCR 主题的 `--text-primary` → `--text` 等）
- 确保页面组件类名一致（`.pg-card`/`.pg-btn` 等）

```python
ocr_adapted = ocr_content
ocr_adapted = ocr_adapted.replace('var(--text-primary)', 'var(--text)')
ocr_adapted = ocr_adapted.replace('var(--bg-secondary)', 'var(--bg2)')
# ... 对所有不兼容的变量名做替换
```

### 3. 页面边界定位不准

用 `find`/`rfind` 定位闭合标签时容易offset不准。建议：
- 先写验证断言
- 输出定位点附近的上下文确认
- 使用 `v2.count('<div class="page-card">')` 确认总数

### 4. 版本号遗漏

HTML的版本号可能出现在3-4处（header标题、副标题、footer、meta等）。用替换法统一更新。

### 4. 文件过大超过飞书限制

445KB HTML文件上传正常（&lt;20MB）。但如果包含base64图片可能导致超大文件。大图片使用URL引用而非base64。

### 5. 页面清单与原型编号不一致

页面清单更新后，原型部分的编号必须同步更新。在Phase 1的清单表中直接列出所有页面，原型部分保持相同编号。

### 6. File 类型文件不支持直接共享到群聊

飞书 Permission API 对不同类型的对象行为不同：

| 对象类型 | `type=` | 共享用户 | 共享群聊 |
|---------|---------|---------|---------|
| docx 文档 | `type=docx` | ✅ `openid` | ✅ `openchat` |
| 文件夹 | `type=folder` | ✅ | ✅ |
| **file (HTML/PNG)** | **`type=file`** | **✅** | **❌ 400** |

共享 file 到群聊时 `POST /drive/v1/permissions/{token}/members?type=file` 返回 400。根因是飞书对 file 类型不支持直接设定群聊权限。

**解决方案**：不通过 Permission API 共享 file 到群聊，改为通过 IM 消息发送链接：

```python
# ❌ 不可行
body = {'member_type': 'openchat', 'member_id': 'oc_xxx', 'perm': 'view'}

# ✅ 可行：通过 send_message 或 IM 消息发送链接
link = f'https://rcncjr4hag8s.feishu.cn/file/{file_token}'
```

用户点击链接后自动跳转到飞书文件预览页（拥有该文件访问权限的前提下）。

**影响范围**：所有模式 B（HTML/PNG 文件）和模式 C（HTML 文档）的交付流程。

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
- `education-pad-ui` — Pad端交互原型设计规范
- `html-info-doc-production` — HTML信息图文档生产的通用模板骨架
- `references/structured-feedback-iteration.md` — 实战案例：基于36条结构化反馈从v2.1迭代到v2.2
- `references/html-merge-technique.md` — HTML大文件合并技术详解
- `references/design-doc-template-v1.0.html` — **设计方案标准化作业模板**：基于AI家庭教师6轮会话沉淀的6种设计文档类型的完整内容要求、格式规范、陷阱和实例。所有设计文档生产前应参考此模板确定所需类型和内容清单。
