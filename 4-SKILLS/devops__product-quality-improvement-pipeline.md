---
name: product-quality-improvement-pipeline
category: devops
description: >-
  全链路产品质量改进管线 — 结合 C007 顾问团产品分析 + 测试专家创建测试案例 +
  文档归档飞书 + C005 编码工作流实施 + 全系统回归测试 + 缺陷迭代修复。
  适用于对已有系统进行系统性质量审查和改进的端到端流程。
version: 1.1.0
metadata:
  hermes:
    tags: [quality, testing, c007, c005, audit, regression, pipeline]
    related_skills: [advisory-council-workflow, code-development-workflow, dogfood]
---

# 全链路产品质量改进管线

## 适用场景

- 用户需要对已有系统进行全面的功能审计和质量评估
- 用户发现系统有多个问题，希望系统性地发现并修复
- 任务涉及多阶段：产品分析 → 测试设计 → 开发实施 → 回归验证
- 需要协调 C007（顾问团分析）和 C005（编码实施）两条工作流

## 禁止场景

- 单一 Bug 修复（走 C005 Tier 0 极速通道即可）
- 新建系统（走 C005 完整流程）
- 信息查询或配置调整

## 全流程概览

```
Phase 1 ── 产品经理审计
  ├─ 梳理系统功能清单（模块/API/页面/数据）
  ├─ 按功能完备性/用户体验/角色设计/技术架构 4 维度评估
  └─ 输出: 产品优化建议文档（P0/P1/P2 分级）

Phase 2 ── 测试专家审计
  ├─ 实际系统探测（API/页面/边界条件测试）
  ├─ 按 P0/P1/P2 分级记录真实 Bug
  ├─ 输出完整测试案例矩阵（TC-CM/TC-PR/TC-RP 等分类）
  └─ 输出回归测试清单（修复后验证项目）

Phase 3 ── 文档归档
  ├─ 将产品分析文档同步到飞书「顾问团报告」文件夹
  ├─ 将测试案例文档同步到飞书「顾问团报告」文件夹
  └─ 更新 memory 记录版本变更

Phase 4 ── C005 实施
  ├─ 按 P0→P1→P2 顺序执行
  ├─ 每个修复项独立验证后端 API
  └─ 修复后立即更新 memory

Phase 5 ── 全系统回归测试
  ├─ 运行预设的回归测试清单
  ├─ 每项标记 PASS/FAIL
  └─ 对 FAIL 项进行迭代修复

Phase 6 ── 数据恢复
  ├─ 恢复测试中删除的测试数据
  └─ 确认系统回到可用状态
```

## 测试专家角色扩展（2026-04-29 实战验证）

在 Phase 2 中，可以扩展 C007 顾问团阵容，新增「测试专家」角色。其审计策略如下：

### 测试路径规划

```
探测顺序:
  1. API 层测试（curl 逐个端点）
  2. 页面渲染测试（browser + curl）
  3. 边界条件测试（负数/空值/超大值/特殊字符）
  4. 权限测试（未登录调用 v2 API / v1 API 绕过鉴权）
```

### 关键发现技巧（实战总结）

**技巧1：前端 JS URL 路径校验**
```javascript
// 常见的 Bug 模式：Blueprint 前缀缺失
// 前端调用: fetch('/api/v2/session/xxx/heartbeat')
// 实际路由: /v2/api/v2/session/xxx/heartbeat  (蓝图前缀 /v2)
// 检查方法: grep -n "fetch(" templates/*.html | grep -v "v2/api"
```

**技巧2：Mock/SAMPLE 数据检测**
```javascript
// 用 curl 检查页面源码中是否包含硬编码的模拟数据
grep -n "SAMPLE_QUESTIONS\|MOCK_DATA\|模拟数据" templates/*.html
// 或用 browser 工具渲染后检查内容是否与 API 数据一致
```

**技巧3：双入口权限验证**
```bash
# v1 API 可能绕过 v2 鉴权 - 用相同操作测试两个入口
curl -X POST 'http://localhost:5000/api/batches' ...     # v1 → 可能无需鉴权
curl -X POST 'http://localhost:5000/v2/api/v2/content/import' ...  # v2 → 需 parent 角色
```

**技巧4：参数校验 Fuzz 测试**
```python
# 常见的缺少校验的参数
tests = [
    {"total_count": -5, "correct_count": 10},  # 负数+超额
    {"total_count": 0, "correct_count": 0},     # 零值
    {"total_count": None},                      # null
    {"total_count": "abc"},                     # 字符串
]
```

**技巧5：Browser 渲染 vs API 返回不一致**
- API 返回 200 不代表页面渲染正确
- 模板中的模拟数据（如 `var SAMPLE_QUESTIONS`）只能通过 browser 发现
- 加载态（loading spinner）、空态（empty state）、错误态（error）只能用 browser 验证

**技巧6：实际输出 vs 设计常量一致性校验 — 发现计分异常**
- 在测试教育系统的练习模式（大挑战/每日任务）时，不只看 API 返回 200
- **必须校验实际生成的题量是否符合代码中的设计常量**（如 `CHALLENGE_BASE_QUESTIONS`）
- 检测方法：创建一次练习，检查 `question_count` 或 `len(questions)` 是否在预期范围内
- 实战案例：大挑战设计常量 `CHALLENGE_BASE_QUESTIONS = (30, 40)`，但实际因错题/生词不足只生成 5-7 题 — 题量异常导致计分异常，但 API 始终返回 200
- 修复：确保短缺失使用 `shortfall` 填充逻辑补齐到设计值，并用常量而非随机范围保证一致性

### 实战发现的 Bug 类型分布

| 类型 | 占比 | 典型例子 |
|:----|:---:|:---------|
| 硬编码模拟数据 | 15% | v2默写页使用 SAMPLE_QUESTIONS 从未对接真实API |
| URL路径错误 | 10% | 蓝图前缀缺失导致心跳从未生效 |
| 参数校验缺失 | 20% | 接受负数/超额记录（正确率200%） |
| 功能缺失 | 30% | 无删除/编辑/持久化 |
| 错误信息模糊 | 10% | "未找到有效内容"不指出具体问题 |
| 权限绕过 | 10% | v1 API可绕过v2 parent鉴权 |
| 配置错误 | 5% | 健康检查模块路径配置错误 |

## 详细执行路径

### Phase 1: 产品经理审计（C007 角色）

**角色**: 以「资深产品经理」身份运做（可用 C007 顾问团框架）

**步骤**:

1. **功能清单梳理**：系统化浏览所有 Python 文件、HTML 模板、API 端点、数据文件
2. **五维度审计**：
   - 功能完备性：哪些功能缺失？（如试卷无持久化、无内容编辑、无删除接口）
   - **用户体验 — 交互意图清晰度**：每个交互元素是否让用户明确知道「该做什么」？
     - ✅ 教育练习系统尤其高危：题型标签可能被用户忽略，prompt 文本（如仅显示英文句子）不暗示操作
     - ✅ 检查点：输入框是否有明确的占位符提示？题型说明是否足够醒目？空态/首次使用是否引导？
     - ✅ 实战案例：C008 英语 en2cn 题显示 "I have two dogs."，用户截图问"不知道题问的是什么"——因题型标签小且 prompt 自身不暗示操作，应加醒目操作指引
   - 角色设计：家长/学生权限是否合理？（如 v1 API 可绕过 v2 鉴权）
   - 技术架构：模块划分是否清晰？（如 v1/v2 双轨制、参数校验缺失）
   - **数据质量**：LLM 自动生成的题库中是否存在空答案题？检查条件：`answer="" AND options=[]`，若有则来自 PDF 图片盲区。详见 `references/llm-exam-analysis-image-blindness.md`。
   - **数据源一致性**：系统的读写路径是否指向同一个数据源？检查条件：找出所有 JSON 数据文件 → 对比每个子系统的写入源和读取源是否一致。典型陷阱：v4 练习写入 `word_frequency.json` 但家长报告读 `state.json`，导致报告显示 0 数据。详见 `references/data-source-migration-audit.md`。
   - **判题正确性**：每种题型的预期答案字段是否与判题函数使用的字段一致？检查条件：对每种题型追踪 生成→存储→判题 三条路径，确认答案字段在传递中未发生映射错误。典型陷阱：en2cn 题型答案在 `answer` 字段（中文），判题函数从 `word` 字段读取（英文），输入正确中文永远判错。详见 `references/grading-field-mapping.md`。
   - **数据内容质量**：cn 字段虽然存在但值与 en 相同（英文原文），导致 en2cn 题型无法使用。检查条件：对所有双语条目检查 `cn == en`。详见 `references/bilingual-content-quality-audit.md`。
   - **题型生命周期管理**：移除某个题型时，需系统性检查 6 层代码（生成→判题→重练→模板→数据→死代码）。若发现某题型功能已作废但仍存在代码残留，记录为 P1 优化项。详见 `references/question-type-removal-pattern.md`。
3. **分级输出**：P0(致命) / P1(重要) / P2(优化) 三级建议
4. **格式**：Markdown 文档，含问题描述、影响评估、修复方案建议

### Phase 2: 测试专家审计（新增 C007 角色）

**角色**: 以「资深系统测试专家」身份运作

**步骤**:

1. **环境探测**：先跑一遍所有 API 端点（GET/POST/DELETE），检查返回码
2. **页面渲染检查**：逐页访问确认能正常渲染
3. **边界测试**：负数参数、空值、超大值、特殊字符
4. **权限测试**：未登录访问、跨版本 API 调用
5. **Bug 记录**：按 P0/P1/P2 分级，每条包含：
   - ID、模块、严重度
   - 复现步骤
   - 实际 vs 预期行为
   - （可选）截图或 API 响应示例
6. **测试案例矩阵**：按模块分类（CM=内容管理, PR=练习, RP=报告, WB=错题本, EX=试卷, VI=视觉, UM=用户管理, SEC=安全），每个案例含：
   - 测试ID、测试项、前置条件、输入、预期结果、状态
7. **回归测试清单**：列出 P0 修复后必须验证的关键路径

### Phase 3: 文档归档

1. 将产品分析文档和测试案例文档写入本地 `artifacts/<task-id>/` 目录
2. 通过飞书 API 创建文档到「我的系统/顾问团报告」文件夹
3. 使用 `feishu-md-writer.py` 写入 Markdown 内容
4. 记录文档 token 到 memory，供后续每日同步

### Phase 4: C005 实施

**注意**: 此阶段在用户明确要求"执行"后启动（C007 熔断规则）

1. 按 P0→P1→P2 顺序处理
2. 每个修复项：
   - 定位问题所在的文件和行号
   - 确定修复方案（patch / 新增 API / 修改模板）
   - 实施修复
   - 单独验证 API 响应
3. P0 项可并行修复（使用 delegate_task 或 execute_code 批量编排）
4. 每次修复后记录到 memory

### Phase 4.1: Fix Landing Verification（新增）

> 核心教训：同一功能可能有多个实现页面（如 material-bank vs material-library），
> 但用户只通过导航链接访问其中一个。修复错页面 = 未修复。

在每次 C005 实施完成后、质量门禁扫描之前，必须验证修复是否落在用户实际访问的页面上。

**三线交叉验证法**：

```bash
# 线1：导航链接 — 用户实际点击什么？
grep -oP 'href="([^"]+)"' templates/parent/index.html | grep root_bp_prefix

# 线2：路由注册 — 哪些页面实际存在？
grep '@bp.route' routes.py

# 线3：文件路径 — 修复修改了什么文件？
ls -la fix_files_changed.txt   # 或 git diff --name-only
```

**验证步骤**：

1. **识别导航入口**：从家长/学生门户页找到所有导航链接（`href` 值）
2. **映射到路由**：每个导航链接对应哪个 Blueprint + 路由注册
3. **映射到模板**：该路由 `render_template` 渲染什么文件
4. **与修复文件交叉比对**：修复的文件是否与该路由渲染的模板文件一致

**常见陷阱**：

| 陷阱 | 表现 | 后果 |
|:----|:-----|:-----|
| 双页面同名不同渲染 | `material-bank` 和 `material-library` 不同路径渲染不同模板 | 修复了 A 但用户访问 B |
| 有路由无导航 | `/english/parent/material-bank` 注册了但门户没有链接 | 修复对用户完全不可见 |
| 导航链接指向非标准路径 | 如 `/english/parent/exam-manage` 而非 `/exam-manage` | Blueprint 前缀被忽略 |

**检查示例**：

```bash
# 1. 提取导航中的所有 href
grep -oP 'href="/english/parent/[^"]*"' templates/parent/index.html

# 2. 检查每个导航路径对应的路由
for route in import records material-library exam-manage wrong-book; do
    echo "=== $route ==="
    grep -A1 "@eng_v2.route(\"/$route\"" routes.py
done

# 3. 检查该路由渲染什么模板
grep -A2 "@eng_v2.route(\"/$route\"" routes.py | grep render_template

# 4. 确认修复文件 == 该模板文件
```

**修复验证清单**：

- [ ] 所有导航链接已提取
- [ ] 每个链接的 Blueprint + 路由已定位
- [ ] 路由渲染的模板文件已确认
- [ ] 本次修复的文件与模板文件一致（或属于该模板引用的子模板/API）
- [ ] 如果发现修复文件与导航路径不匹配 → 立即转移修复到正确文件

### Phase 4.5: 质量门禁扫描（新增）

在实施完成后、回归测试之前，运行自动化质量门禁扫描新增/修改的系统。

**执行方式**：
```bash
python3 ~/.hermes/scripts/quality_gate.py --target ~/edu-hub/ai-family-tutor
```

**输出**：JSON 报告生成在项目根目录 `quality_gate_report.json`（非 artifacts/ 下）

**门禁策略**：
- 门禁扫描会发现**三类问题**：新引入的、预先存在的、行为预期（如认证）
- 所有阻断级问题必须处理 → 进入 Phase 4.6

**⚠️ 首跑量级陷阱**：首次在现实项目上运行 quality_gate.py 会产生大量发现（实战：AI家庭教师项目首跑 426 项，其中 A 类 4、B 类 30、C 类 392）。这是正常的——绝大多数是遗留存量问题。不要被总数吓到，执行 Phase 4.6 分类后，真正的阻断项（A 类新引入）通常是个位数。

### 6维质量门禁定义（D1-D6）

质量门禁扫描覆盖以下6个维度，对应8类历史高发缺陷：

| 门禁 | 检测内容 | 检测方法 | 目标缺陷 |
|:----|:---------|:---------|:---------|
| **D1** | 硬编码模拟数据 | 搜索 SAMPLE / MOCK / 模拟数据 / 示例题 模式在 .py/.html/.js 中 | 硬编码兜底未替换 |
| **D2** | URL路由一致性 | 对比前端 href 与后端 Blueprint 路由注册，每三个月蓝图前缀匹配 | 死链接/蓝图前缀缺失 |
| **D3** | 数据持久化完整性 | 每个 submit/answer 处理器 → 检查是否有 write/record/INSERT/update 调用 | 幽灵评分（数据未持久化） |
| **D4** | 模板渲染变量追踪 | 模板变量 → render_template 参数 → 上游数据源，验证管道每环节变量存在 | 空洞渲染（模板条件字段缺失） |
| **D5** | 输入参数校验 | 路由函数是否有 input validation / 边界测试覆盖负数/空值/超大值 | 边界缺失 |
| **D6** | 数据契约合规 | API 响应字段名/类型 vs 契约 Schema 一致性校验 | 数据契约漂移 |

**A/B/C/D 缺陷分类标准**：

```
门禁发现问题
  ├─ A类: 新代码引入的问题 → 必须100%修复，阻断交付
  ├─ B类: 预先存在的遗留问题 → 按用户意愿修复
  │   ├─ 用户选择"全部清理" → 进入遗留项清理流程
  │   └─ 用户选择"只修新代码" → 记入基线文档，不阻断
  ├─ C类: 行为预期问题 → 记录为基线，不修复
  │   ├─ 认证要求（403/401）
  │   ├─ 静态分析假阳性（变量实际已传递但分析器不识别）
  │   └─ 无效路径回退行为
  └─ D类: 非阻断警告 → 记录但允许交付（如硬编码初始值）
```

**自进化机制**：发现新的缺陷模式（非D1-D6覆盖的），写入 `defect_patterns.json` 注册表，分配 D编号，下次扫描自动包含新检测。

**实现脚本**：`~/.hermes/scripts/quality_gate.py` — 6维扫描，输出 JSON 报告。在后续开发任务 Step 3→Step 4 之间运行。

### Phase 4.6: 遗留项分类与清理（新增）

质量门禁报告中的问题需要分类处理，不能一刀切。分类标准如下：

```
门禁发现问题
  ├─ A类: 新代码引入的问题 → 必须100%修复，阻断交付
  ├─ B类: 预先存在的遗留问题 → 按用户意愿修复
  │   ├─ 用户选择"全部清理" → 进入遗留项清理流程
  │   └─ 用户选择"只修新代码" → 记入基线文档，不阻断
  ├─ C类: 行为预期问题 → 记录为基线，不修复
  │   ├─ 认证要求（403/401）
  │   ├─ 静态分析假阳性（变量实际已传递但分析器不识别）
  │   └─ 无效路径回退行为
  └─ D类: 非阻断警告 → 记录但允许交付（如硬编码初始值）
```

**遗留项清理流程（B类）**：

```bash
1. 获取完整问题清单（JSON报告）
2. 分类：哪些是A类（新引入）vs B类（遗留）vs C类（预期）
3. 对于B类：
   a. L1静态扫码问题 → 模板变量补全、死链修复、函数引用补全
   b. L1静态扫码问题（模板变量不匹配）→ 在render_template补充缺失变量，设置安全默认值
   c. 重定向路径修复 → 处理路径比较中的尾杠问题
   d. 死链修复 → 替换真实路由或javascript:void(0)
4. 对每个修复项：先读完整文件 → 理解上下文 → 小范围patch
5. 每修复一批 → 重启服务 → 重新运行门禁验证
6. 重复直至剩余问题均为C类（预期行为）或D类（警告）
```

**常见遗留问题模式**：

| 模式 | 根因 | 修复方案 |
|:----|:-----|:---------|
| 模板变量不匹配 | render_template缺失模板使用的变量名 | 补全变量，设置安全默认值（空字符串/空列表/0） |
| 函数引用未定义 | JavaScript函数被调用但未实现 | 添加安全兜底实现或移除调用 |
| 重定向404 | catch-all路径映射未覆盖无尾杠情况 | 修复path.rstrip('/')比较逻辑 |
| 死链接href="#" | UI占位符残留 | 替换为真实路由或javascript:void(0) |
| 静态资源404 | 旧CSS/JS路径失效 | 检查文件存在性或移除引用 |
| 认证路由"内容过小" | 未登录返回的403/重定向页面 | 预期行为，记录为基线 |

**基线文档**：
对最终的C类问题，在交付物中记录：
```json
{
  "baseline_issues": [
    {"check": "L2-001", "count": 76, "reason": "认证路由需要session，预期行为"},
    {"check": "L1-007", "count": 1, "reason": "静态分析假阳性，变量实际已传递"}
  ]
}
```

### Phase 5: 全系统回归测试

1. 运行 Phase 2 的回归测试清单
2. 对于复杂的端到端测试（如 v2 默写页），使用 browser 工具检查渲染
3. 每项标记 ✅ PASS 或 ❌ FAIL
4. 对 FAIL 项进行迭代修复（小问题直接修复，大问题回 Phase 4）

### Phase 6: 数据恢复

1. 恢复测试中删除的测试数据（重新导入或调用重建 API）
2. 确认所有批次恢复到可用状态
3. 输出最终测试统计：总测试数、通过数、失败数

## 输出预算与质量要求

| 产出物 | 内容 | 格式 |
|--------|------|------|
| 产品优化建议 | P0/P1/P2 分级，每条含描述+影响+方案 | Markdown 文档 |
| 测试案例矩阵 | 模块分类，每条含前置+输入+预期+状态 | Markdown 文档 |
| 回归测试清单 | 修复后验证项，PASS/FAIL 标记 | Markdown 文档 |
| 最终统计 | 总测试数 / 通过 / 失败 | 控制台输出+memory |

## 常见陷阱

1. **跳过实际探测直接写测试案例** — 必须先跑一遍真实 API，否则会漏掉运行时错误
2. **测试中删除数据后忘记恢复** — Phase 6 必须有数据恢复步骤
3. **只做 API 测试不做页面测试** — Browser 渲染问题（如模拟数据、CSS 错误）只能通过浏览器发现
4. **产品分析和测试案例脱节** — 测试案例应覆盖产品分析中发现的所有 P0 项
5. **忽略权限测试** — v1 API 可能绕过 v2 鉴权，这是常见安全隐患
7. **LLM 试卷分析的图片盲区** — 当审核一个「从试卷分析自动生成题库」的系统时，检查题库中是否有空答案题（答案和选项都为空）。这些通常来自 PDF 中"根据图片内容"的题型，LLM 无法看到图片。详见 `references/llm-exam-analysis-image-blindness.md`。
8. **难度分层的题型可能永远不可达** — 当系统按难度分层配置题型池时（如 `QUESTION_TYPES = {"easy": ["choice","fill_blank"], "hard": ["error_correction"]}`），普通练习（默认 easy）永远不会生成 hard-only 题型。测试时必须显式测试所有难度层级，不能只测 easy。检测方法：遍历模板中所有 `type` 字段，与难度池交叉对比。

## 参考文件

| 文件 | 内容 |
|------|------|
| `references/interaction-intent-check.md` | 交互意图清晰度检查清单 |
| `references/navigation-path-cross-ref.md` | 导航路径交叉引用审计 |
| `references/llm-exam-analysis-image-blindness.md` | LLM 试卷分析图片盲区陷阱 |
| `references/quality-gate-baseline-triage.md` | 质量门禁基线分类策略 |
| `references/data-source-migration-audit.md` | 数据源迁移审计 — 写入新文件但读取旧文件的检测与修复 |
| `references/grading-field-mapping.md` | 判题函数字段映射不一致 — 多题型系统中答案字段与判题字段不匹配的检测与修复 |
| `references/bilingual-content-quality-audit.md` | 双语内容数据质量审计 — cn==en 导致 en2cn 题型无声失效的检测与修复 |
| `references/question-type-removal-pattern.md` | 从教育系统移除题型 — 六层重构检查清单（生成→判题→重练→模板→数据→死代码） |
| `references/defect-pattern-registry.md` | 缺陷模式注册表 — 自进化门禁系统，包含 D001-D004 初始模式定义 |
| `templates/quality_gate_template.py` | 6维质量门禁扫描脚本模板 — 可复制并扩展。生产级实现在 `~/.hermes/scripts/quality_gate.py`，已在 AI家庭教师项目验证（首扫描426项发现） |
| `~/wiki/guides/development-testing-key-points.md` | 6 类开发/测试检查点 — 模板变量传递、标志消费链、五层数据流验证、题型分支测试等 |
| `references/api-level-batch-scanner-pattern.md` | API层批量扫查模式 — 直接调后端引擎生成所有输出，程序化验证数据完整性 |

## 关联技能

- `advisory-council-workflow` (C007) — 顾问团 4 阶段询问
- `code-development-workflow` (C005) — 编码实施工作流
- `dogfood` — 探索性浏览器 QA 测试
- `batch-execution-optimization` — 多步骤任务批量执行
