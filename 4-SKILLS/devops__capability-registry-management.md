---
name: capability-registry-management
description: 体系化能力注册与管理系统 — 注册、审计、生命周期管理。统一管理所有端到端能力与一次性方案，确保可发现、可治理、可优化。
version: 1.4.0
author: Hermes Agent
metadata:
  hermes:
    tags: [registry, meta-management, governance, lifecycle, content-enrichment]
    related_skills: [feishu-drive-sync-pipeline, llm-wiki, personal-knowledge-base-management]
    use_cases:
      - 查看所有体系化能力清单
      - 注册新系统或归档新方案
      - 对某个能力进行优化/修复/废弃
      - 月度审计检查
      - 丰富现有系统的描述内容（角色定义、架构图等）
---

# 体系化能力注册与管理

## 概述

本技能管理两类不同的资产：

| 类型 | 标识 | 定义 | 管理位置 |
|------|------|------|---------|
| 🟢 **系统** | C000–C999 | 持续运行的端到端闭环（有 cron/定期维护） | `~/wiki/entities/capability-registry.md` |
| 📋 **解决方案** | S001–S999 | 一次性完成的方案（部署/迁移/技术验证，完成后封存） | `~/wiki/entities/solution-archive.md` |

**共同入口**: `~/wiki/index.md`（Registry 分区 + 解决方案台账分区）
**审计脚本**: `~/.hermes/scripts/capability-lint.py`（仅审计系统）

### 判断标准

**系统 (Cxxx)** — 必须同时满足：
- ✅ **端到端闭环** — 解决一个完整问题域，而非单点功能
- ✅ **持续运行** — 有 cron 定时任务或定期维护流程
- ✅ **有明确资产** — 文件/脚本/cron/skill 等多种资源组合
- ✅ **可复现** — 不是做了一次，而是有重复执行的模式

**解决方案 (Sxxx)** — 满足任一：
- ✅ **一次性完成** — 部署、迁移、技术验证、配置搭建
- ✅ **完成后封存** — 不再持续运行，但经验值得记录
- ✅ **可复用参考** — 未来类似场景可能再次需要

**两者都不算**：
- ❌ 单点技能（如一个独立的 shell 脚本）
- ❌ 工具本身（如 Ollama、ChromaDB——是组件，不是能力）
- ❌ 日常任务记录（应走飞书→Wiki 同步管道）

### 生命周期状态（仅系统适用）

| 状态 | 含义 | 典型时长 |
|------|------|---------|
| `proposed` | 已设计方案但未部署 | 设计阶段 |
| `active` | 正常运行 | 持续 |
| `needs-repair` | 已知问题待修复 | 按需 |
| `idle` | 90天无操作 | 建议确认是否废弃 |
| `deprecated` | 不再推荐使用 | 30天后自动归档 |
| `archived` | 已归档，仅作参考 | 永久 |
| `merged` | 已合并到另一个系统 | 永久 |

解决方案没有生命周期状态——只有「已记录」和「已删除」。

---

## 操作规程

### 1. 查看全景

```markdown
你: "查看能力全景"

我: 读取 ~/wiki/index.md
  → 展示 Registry 分区（当前系统）+ 解决方案台账分区
```

### 2. 查看详情

```markdown
你: "查看 C001" 或 "查看 S001" 或 "查看个人知识库 RAG"

系统 → 跳转到 capability-registry.md 对应锚点
       → 展示: 一句话定位 + 流程 + 资产 + 已知问题 + 健康检测

方案 → 跳转到 solution-archive.md 对应锚点
       → 展示: 解决什么问题 + 设计方案 + 结果 + 经验教训
```

### 3. 完整引导新系统（从规范到全链路部署）

```markdown
触发条件: 用户提供系统规范文档 / 你设计新系统方案 / 部署首次成功 / 审计发现

=== 全链路步骤（按顺序执行）===

第一步：判断分类
  持续运行？有 cron？有定期维护？ → 系统 (Cxxx)，继续以下所有步骤
  一次性方案？完成即封存？         → 解决方案 (Sxxx) → 走操作规程 #8

第二步：创建本地知识结构
  mkdir -p ~/wiki/raw/systems/<id>/
  创建文件：
    ├── index.md          ← 系统架构文档（定位、模块、目录结构、关联skill、变更日志）
    ├── architecture.md   ← 架构详情（模块说明、飞书路径映射、核心流程图）
    ├── changelog.md      ← 变更日志表（日期/操作/说明）
    └── templates/        ← 可选模板文件（如画像卡/日志/报告模板）

第三步：创建核心资产
  - 如果系统需要行为框架 → 创建 skill（skill_manage create, category: devops）
  - 如果系统需要脚本/配置 → 创建脚本（~/.hermes/scripts/ 或系统自有目录）
  - 初始化数据文件（如 self-observation-db.json 等）

第四步：评估飞书目录需求
  该新系统是否需要独立的飞书目录结构？
  → 需要：创建新文件夹（在 drive root 或已有父文件夹下）
    使用 lark_oapi 的 CreateFolderFileRequest:
    body = CreateFolderFileRequestBody()
    body.name = '📁文件夹名'
    body.folder_token = '<parent_token>'  # None = drive root
    req = CreateFolderFileRequest.builder().request_body(body).build()
    resp = client.drive.v1.file.create_folder(req)
    # 注意: resp.data.token（不是 node_token 也不是 file.token）

  → 需要子文件夹：迭代创建，依次传递 folder_token

第五步：创建系统文档
  - 在飞书目标文件夹下创建 docx 文档:
    POST /open-apis/docx/v1/documents
    body: {"title": "标题", "folder_token": "<folder_token>"}
    返回: document_id = doc_token

  - 用 feishu-md-writer.py 写入内容:
    python3 ~/.hermes/scripts/feishu-md-writer.py <doc_token> <content.md>

  - 记录所创建文档的 tokens（后续注册要用）

第六步：注册到能力注册表
  在 registry 中新增 ### CXXX 条目
  → 填写元数据 (名称/状态/域/标签/版本/阶段)
  → 描述核心模块/端到端流程
  → 列出所有涉及资产（知识库目录/skill/脚本/数据文件等）
  → 编写健康检测命令
  → 更新 index.md（在 Registry 分区增加 CXXX 链接）
  → 更新注册表头部统计（总条目数）
  → 更新关联能力的变更日志（如有）

第七步：接入飞书同步管道
  → 将新文档 token 添加到 ops-sync-feishu-docs.py 的 DOC_TOKENS 字典
  → 将新 ID 添加到 SYSTEM_IDS 列表
  → 运行 python3 ~/.hermes/scripts/ops-sync-feishu-docs.py 全量同步
  → 验证输出: "发现 N+1 个系统"（核对总数）+ "✅ 已更新: CXXX-..." 出现

第八步：全链路验证
  # 本地验证
  test -d ~/wiki/raw/systems/<id>/ && echo "OK"
  grep -q "CXXX" ~/wiki/entities/capability-registry.md && echo "OK"
  # Skill 验证（如创建）
  test -f ~/.hermes/skills/devops/<skill-name>/SKILL.md && echo "OK"
  # 飞书文档验证
  python3 -c "
  import urllib.request, json, os
  # 用 raw_content API 确认内容长度 > 0
  token = '<doc_token>'
  app_id='...'; app_secret='...'
  auth = json.dumps({'app_id': app_id, 'app_secret': app_secret}).encode()
  req = urllib.request.Request('https://open.feishu.cn/open-apis/auth/v3/tenant_access_token/internal', data=auth, headers={'Content-Type': 'application/json'})
  resp = json.loads(urllib.request.urlopen(req).read())
  t = resp['tenant_access_token']
  url = f'https://open.feishu.cn/open-apis/docx/v1/documents/{token}/raw_content'
  req = urllib.request.Request(url, headers={'Authorization': f'Bearer {t}'})
  resp = json.loads(urllib.request.urlopen(req).read())
  content = resp.get('data', {}).get('content', '')
  assert len(content) > 0, f'Docs {token} content empty!'
  print(f'OK — {len(content)} chars')
  "

第九步：记录到记忆
  → 保存一条 compact 记忆：系统名 + 核心资产路径 + 飞书tokens
  → 格式:"CXXX 系统名已上线(日期): skill(name)+知识库(path)+飞书目录(tokens). 系统文档<doc_token>."
```

=== 关键陷阱 ===
- 飞书创建文件夹时，`resp.data.token` 是字段名（不是 `node_token` 或 `file.token`）
- 创建文档需要 `folder_token` 参数（非必需但建议），否则文档落在 drive root
- 同步脚本更新后，必须关注输出中的系统数量——如果 sync 脚本显示数比预期少，说明 regex 匹配失败
- 注册表锚点 `<a id="cxxx"></a>` 必须紧贴 `## CXXX — 标题`（允许空白行但过多空白行会导致 sync 脚本静默跳过）

### 4. 优化/修复系统

```markdown
你: "优化 C001"

我: 加载 C001 完整定义
  → 运行健康检测项
  → 分析已知问题列表
  → 评估影响范围 (检查"关联能力"字段，看依赖链)
  → 制定修复/优化方案
  → 执行
  → 更新注册表 (updated + 变更日志)
  → 运行 python3 ~/.hermes/scripts/ops-sync-feishu-docs.py 同步云文档
```

### 4a. 内容丰富化 — 扩展系统描述细节

```markdown
触发条件: 系统已有基础注册信息，但某个描述章节过于简略
         （如角色定义只有寥寥几行、架构图太简单、缺少职责详解等）

典型场景:
  - 用户说"补充系统中设定角色的详细描述"
  - 用户说"把XX系统的描述写详细一点"
  - 发现某个系统条目在能力注册表中只有骨架，缺乏实质性内容

流程:
  ① 读取当前系统条目 (capability-registry.md 对应锚点)
  ② 确定需丰富的内容区域：
     - 角色定义表 → 扩展列（#/角色/图标/类型/职责详解）
     - 架构全景图 → 更新 ASCII 图反映最新结构
     - 角色阵容扩展规则 → 补充整体说明段落
  ③ 用 patch 工具精确替换：
     - 替换旧表格为新表格（加编号、加图标、加职责详解列）
     - 替换架构图中过时的角色数/类型描述
     - 补充阵容扩展规则（如适用）
  ④ 更新变更日志（新增 changelog entry）
  ⑤ 运行同步: python3 ~/.hermes/scripts/ops-sync-feishu-docs.py
  ⑥ 验证: 确认输出中 "✅ 已更新: CXXX-..." 包含目标系统

注意事项:
  - 保持原有 markdown 表格语法一致（对齐方式、分隔线）
  - 架构图 ASCII 宽度保持一致，避免错位
  - 表格内容如果跨多行，用普通 markdown 分行而非渲染为多行
  - 同步后检查飞书文档内容（raw_content API 验证）
```

### 5. 废弃系统

```markdown
你: "废弃 C002"

我: 标记 status: deprecated
  → 记录废弃日期
  → 通知关联能力 (如果被其他能力引用)
  → 30天后审计脚本自动建议：已到归档期
```

### 6. 合并系统

```markdown
你: "把 C001 合并到 C002"

我: C001 标记 merged → merged_into: C002
  → C002 更新资产清单 (加入 C001 的资产)
  → 更新索引
```

### 7. 月度审计（仅系统）

```markdown
Cron: 每月第1天 8:00

运行 ~/.hermes/scripts/capability-lint.py

检查项:
  1. 声明资产是否存在 (文件/脚本/cron/skill)
  2. 90天 idle 预警
  3. 180天 deprecated 自动归档
  4. 未注册的活跃 cron 任务
  5. ID 冲突

输出: 审计报告 → ~/wiki/reports/capability-lint-{date}.md
```

### 8. 新增解决方案

```markdown
触发条件: 你完成了一次性部署/迁移/技术验证，或已有方案需要归档

在 solution-archive.md 新增 ### SXXX 条目
  → 填写元数据 (名称/类型/完成日期/状态)
  → 描述解决的问题
  → 记录设计方案与结果
  → 说明失败原因 (如果适用)
  → 列出遗留资产 (源码压缩包路径/skill 引用)
  → 记录经验教训（供未来复用）
  → 说明何时可复用
  → 更新 index.md
  → 记录 log.md
```

---

## 模板

### 飞书系统文档模板

每个系统在飞书「我的系统」目录中的文档自动生成，固定结构为：

```markdown
## 🧠 基本逻辑

- 一句话定位
- 系统架构/数据流图
- 依赖关系（关联能力）
- 系统特定内容（状态表/分级/角色定义等）

## 📖 使用手册

- 启动与停止
- 资产清单
- 健康检测
- 已知问题/当前局限
- 变更日志
```

同步脚本（`ops-sync-feishu-docs.py`）从注册表提取 20+ 字段，按此模板自动拼装。同步时按需包含各系统特有的章节（如 C008 的系统状态表、C006 的采集指标表）。

### 系统模板

每条系统登记时包含：

```markdown
### CXXX — 系统名称

| 字段 | 值 |
|------|-----|
| **ID** | CXXX |
| **名称** | |
| **状态** | `proposed` / `active` / `needs-repair` / ... |
| **域** | |
| **创建** | YYYY-MM-DD |
| **更新** | YYYY-MM-DD |
| **标签** | |
| **阶段** | phase-X |
| **关联能力** | |

### 一句话定位

### 端到端流程

ASCII 流程图描述完整链路

### 涉及资产

| 类别 | 资产 | 路径/标识 | 状态 |

### 健康检测

```bash
# shell 命令检查关键资产是否存在
```

### 已知问题

| # | 问题 | 发现日期 | 严重度 | 说明 |

### 变更日志

| 日期 | 操作 | 说明 |
```

### 解决方案模板

```markdown
## SXXX — 方案名称

| 字段 | 值 |
|------|-----|
| **ID** | SXXX |
| **名称** | |
| **类型** | 部署 / 迁移 / 技术验证 / 参考架构 |
| **完成日期** | YYYY-MM-DD |
| **状态** | 已完成 / 已完成（参考） |
| **标签** | |

### 一句话

### 解决的问题

### 设计方案

### 结果

### 失败原因（如适用）

### 遗留资产

| 资产 | 路径 | 用途 |

### 何时可复用

### 经验教训

### 操作记录

| 日期 | 操作 | 说明 |
```

---

## 能力间依赖声明

如果系统 B 依赖系统 A 的结果，在 B 的资产表中注明。

```markdown
| 🔗 | 依赖 C001 | Wiki 仓库 | 使用 C001 的知识页面 |
```

审计脚本会在 monthly lint 中检查引用的依赖 ID 是否存在。

---

## 注意事项与陷阱

### 常见错误

| 陷阱 | 后果 | 正确做法 |
|------|------|---------|
| 把一次性方案注册为系统 | 月审报资产缺失、状态混乱 | 先判断：有 cron 吗？→ 否 → 走解决方案 |
| 把工具本身当成能力 | 注册表膨胀，失去管理价值 | 工具是组件，能力是使用工具解决完整问题的闭环 |
| 注册后不更新任何资产 | 审计告警，依赖链断裂 | 注册时一次填完所有资产表 |
| 技能/脚本未创建就注册 | 审计告警 "声明资产缺失" | 先实现，再注册 |

### 关键教训（从实践中来）

**教训一：用户明确区分「系统」和「解决方案」，不要在同一个注册表中混用**
- 向量检索部署框架最初被注册为 C003（系统），但它是一次性部署尝试，完成后就归档
- 用户指出应该单独管理 → 创造了两层体系
- 这个区分不仅是语义问题，影响月审逻辑、生命周期管理、依赖追踪

**教训二：注册表是元管理工具，不要过度设计**
- 前端时间不需要复杂的状态机——方案只有「已记录」和「已删除」
- 发现问题简单标出来即可，不需要自动化的修复流程

**教训三：判断分类时，问一句「这是持续运行的东西吗？」**
- 这比问「这是系统性能力吗？」更准确
- 系统的本质特征不是「复杂」或「重要」，而是「持续运行」

**教训四：注册表锚点格式 → 同步脚本静默失败**
- 手动编辑注册表时，可能在 `<a id="cxxx"></a>` 和 `## CXXX — 标题` 之间意外插入空白行
- `ops-sync-feishu-docs.py` 的 regex `\\n## CXXX` 只允许紧贴，遇到空白行会静默跳过该系统的同步
- 2026-04-29 实战：C008 被跳过 → 飞书文档内容为空 → 耗时 debug 才发现是 line 950 多了一个空白行
- **修复后的 regex**: `<a id="{sys_id.lower()}"></a>\\s*\\n## {sys_id}`（`\\s*` 允许零或多个空白字符）
- **预防**：新增/编辑系统后，检查锚点 `<a id="cxxx"></a>` 是否紧贴 `## CXXX — 标题`
- **检测**：同步脚本输出「发现 N 个系统」，核对数量是否等于预期；缺失系统表示 regex 匹配失败

**教训五：同步后必须验证飞书文档有内容**
- 同步脚本输出 "✅ 已更新" 不代表飞书文档有内容——C008 曾输出成功但文档为空
- 验证方法：用 raw_content API 确认内容长度 > 0
  ```python
  # 验证飞书文档有内容
  url = f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}/raw_content"
  req = urllib.request.Request(url, headers={'Authorization': f'Bearer {token}'})
  resp = json.loads(urllib.request.urlopen(req).read())
  content = resp.get('data', {}).get('content', '')
  assert len(content) > 0, f"文档 {doc_token} 内容为空!"
  ```

**教训六：注册表内表格字段名加粗格式不一致 → 审计脚本解析静默失败**
- `capability-lint.py` 的 regex 默认匹配 `**状态**`（加粗）格式，但 C005、C006 的注册表条目使用 `|| 状态 | \`active\` |`（非加粗）
- 2026-05-01 审计发现：C005、C006 正确显示为 `active`，但 lint 脚本输出 `status: unknown`
- 修复：regex 改为 `(?:\*\*)?状态(?:\*\*)?` 兼容两种格式（同样适配标签/更新/创建字段）
- **预防**：注册新系统时，表格字段名可以统一用 `| **字段** |` 格式（带加粗）
- **检测**：审计报告中有系统显示 `status: unknown` 但在注册表中实际有明文状态 → 属于 regex 未匹配
- 同步脚本输出 "✅ 已更新" 不代表飞书文档有内容——C008 曾输出成功但文档为空
- 验证方法：用 raw_content API 确认内容长度 > 0
  ```python
  # 验证飞书文档有内容
  url = f"https://open.feishu.cn/open-apis/docx/v1/documents/{doc_token}/raw_content"
  req = urllib.request.Request(url, headers={'Authorization': f'Bearer {token}'})
  resp = json.loads(urllib.request.urlopen(req).read())
  content = resp.get('data', {}).get('content', '')
  assert len(content) > 0, f"文档 {doc_token} 内容为空!"
  ```
