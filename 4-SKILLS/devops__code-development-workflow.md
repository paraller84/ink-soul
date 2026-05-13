---
name: code-development-workflow
description: >-
  标准化编码开发工作流 v3.0 — 交付物驱动模式。
  将任务分解为原子步骤，每步输出结构化交付物文件，
  通过路径引用传递依赖，消除上下文膨胀。
  支持选择性重跑、血缘追踪和可审计的执行链路。
  继承 v2.2 的全云端模型策略和负载等级体系。
version: 4.0.2
author: Hermes Agent
metadata:
  hermes:
    tags:
      - devops
      - workflow
      - coding
      - code-review
      - testing
      - architecture
      - deliverable-driven
    category: devops
---
# 标准化编码开发工作流 v3.0 — 交付物驱动模式

>
> 所有涉及编码的任务，必须按此流程执行。
>
> v3.0 核心改进：**交付物驱动模式** — 步骤间通过结构化交付物文件显式传递依赖，仅保留路径引用而非全量历史累积，支持选择性重跑和血缘追踪。
>
> v2.2 保留原则：全云端模型策略（所有角色 DeepSeek R1）和负载等级体系继续沿用。

---

## 第一节（前置）：进度可视化面板（全流程导航）

> **任务执行期间，每次完成一个步骤后，必须展示进度可视化面板。**
>
> 这是对所有 C005 等级（Tier 0-3）的**强制行为要求**。
>
> 核心原则：**用户永远知道「我在哪」「到哪了」「还剩什么」**。
> 解决千问方案提到的「任务层级过深导致上下文丢失」问题的精简化方案。
>

### 1.0 每次步骤切换时的进度面板模板

**执行完每个 C005 步骤后**，在开始下一步之前，输出以下格式的进度面板：

```
📊 C005 进度 — [任务短描述] (Tier N)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 0: ✅ 需求分析            已完成
  Step 1: ✅ 代码学习            已完成
  Step 2: 🔄 架构设计            ◀ 进行中
  Step 2.5: ❏ 骨架搭建         待开始
  Step 2.6: ❏ UI/UE设计(按需)   待开始
  Step 3: ❏ 工程实现            待开始
  Step 4: ❏ 质量测试            待开始
  Step 5: ❏ 集成与交付          待开始
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 当前位置: Step 2 — 架构设计子代理执行中
📊 完成度: 2/8 步 (25%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ 已完成: 需求确认完成 / 代码学习交付物已输出
  🔮 下一步: 架构交付物 → 骨架搭建 → UI/UE设计(按需) → 编码
  ⚠️ 风险/待确认: 无
```

### 1.1 层级深度时的增强面板

当执行 Tier 2/3 的子代理并行任务时，展示嵌套进度：

```
📊 C005 进度 — C008语法模块增强 (Tier 2)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 0: ✅ 需求分析
  Step 1: ✅ 代码学习
    ├─ ▶ 子代理 A (架构视角): ✅ 交付物已输出
    └─ ▶ 子代理 B (实现视角): ✅ 交付物已输出
  Step 2: ✅ 架构设计            [[design_v1.json]]
  Step 2.5: ❏ 骨架搭建
  Step 2.6: ❏ UI/UE设计(按需)
  Step 3: 🔄 工程实现            ◀ 进行中
  Step 4: ❏ 质量测试
  Step 5: ❏ 集成与交付
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📍 当前位置: Step 3 — 子代理编码中 (等待返回)
📊 完成度: 4/8 步 (50%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  📂 交付物: req_analysis_v1.json → learn_arch_v1.json → learn_impl_v1.json → design_v1.json
  🔮 下一步: 骨架完成 → 编码 → 质量门禁 → 集成交付
```

### 1.2 非 C005 复杂任务的通用进度面板

当执行非 C005 但涉及多步骤的复杂任务（3+ 步骤）时，自动展示：

```
📊 任务进度 — [任务描述]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✅ 1/4: 数据收集与确认
  ✅ 2/4: 方案设计
  🔄 3/4: 执行实施           ◀ 当前
  ❏ 4/4: 验证与交付
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  📊 进度: 50%
  💡 当前操作: 正在执行方案...
```

### 1.3 执行规则

| 规则 | 说明 |
|:----|:------|
| **频率** | 每个 C005 步骤完成后、开始下一步前必须展示 |
| **非 C005 任务** | 3+ 步骤的复杂任务同样使用通用进度面板 |
| **Tier 0** | 极速通道可简化为单行进度「📊 Step: 需求→实现→测试→交付」，免面板 |
| **更新方式** | 不持久化到文件，每次展示基于当前上下文中的步骤状态计算 |
| **修复循环** | 测试修复循环期间不展示新面板，用「Testing: 修复循环 #3」追加标记 |
| **面板长度** | 不超过 15 行，紧凑优先 |

### 1.3a 跨任务总览（任务启动前置检查）

**在启动 C005 任务（Step 0）或任何复杂任务（3+ 步骤）时**，先执行一次轻量级跨任务检查。

**执行方式**：
1. 调用 `session_search()` 无query查询，获取近期活跃会话
2. 检查记忆中有无未完成标记（如 `[PAUSED]`、`[INCOMPLETE]` 状态）
3. 仅消耗 1 次查询，不做额外开销

**有未完成任务时**，在 Step 0 输出前附加跨任务速览块：

```
📋 跨任务速览
  ⏳ C008 语法模块增强 (Tier 2) — 上次做到 Step 3 (编码中) [4h前]
  ⚠️ 当前新任务与此无关，确认是否优先处理旧任务？
```

**无未完成任务时**，不输出该块，保持简洁。

**不查询的场景**：
- 简单问答（1-2 轮对话）
- 快速回复
- 用户明确说「先做这个，别的不管」

**与进度面板的关系**：跨任务速览是**进入面板之前**的前置检查，进度面板是**执行过程中**每步的状态展示。两者各自独立、互不冲突。

### 1.4 与千问方案的差异

| 维度 | 千问「分层任务导航系统」 | 本进度可视化方案 |
|:-----|:------------------------|:----------------|
| 实现方式 | 11+ 个斜杠命令 + 状态文件 + checkpoint | **纯行为约定**，零代码 |
| 用户负担 | 用户必须记忆命令并手动导航 | **无需记忆**，自动展示 |
| 持久化 | 每个任务写 JSON 状态文件 | 依赖上下文记忆，不额外持久化 |
| 跨会话恢复 | `/resume_at` 命令 | 通过 `session_search` 实现 |
| 子代理 | 不支持嵌套 | delegate_task 天然隔离 |
| 适用场景 | 通用 Hermes Agent 用户 | **专为 C005 工作流优化** |

---

**核心原则**：将任务分解为原子步骤，每步输出结构化交付物文件，通过路径引用传递依赖，消除上下文膨胀。

### 1.1 交付物定义规范

每个步骤的输出为**机器可解析的结构化 JSON 文件**，保存在 `~/edu-hub/artifacts/<task_id>/` 目录下。

```json
{
  "schema_version": "1.0",
  "step_id": "analysis_v1",
  "parent_step_id": null,
  "status": "success",
  "next_step": "design",
  "generated_at": "2026-04-29T20:00:00",
  "checksum": "sha256:abc123...",
  "result": { /* 步骤专有数据 */ },
  "metadata": {
    "model": "DeepSeek R1",
    "execution_seconds": 45,
    "token_cost": 12500
  }
}
```

| 字段 | 类型 | 必填 | 说明 |
|:----|:----|:----:|:-----|
| `schema_version` | string | ✅ | 交付物格式版本 |
| `step_id` | string | ✅ | 全局唯一步骤标识（如 `learn_arch_v1`） |
| `parent_step_id` | string\|null | ✅ | 上游交付物 ID，起点为 `null` |
| `status` | string | ✅ | `success` / `partial` / `error` |
| `next_step` | string\|null | 可选 | 建议的下游步骤名称 |
| `generated_at` | iso8601 | ✅ | 生成时间 |
| `checksum` | string | ✅ | 内容校验（`sha256:hex` 格式） |
| `result` | object | ✅ | 步骤专有结构化数据 |
| `metadata` | object | 可选 | 模型、耗时、Token 等 |

### 1.2 执行契约

每个步骤在执行前必须满足以下契约：

```
┌─ 步骤执行前 ──────────────────────────────────────┐
│  1. 输入校验：验证 parent_step_id 对应的交付物文件    │
│     存在且 checksum 匹配（防损坏/篡改）              │
│  2. 上下文注入：仅加载当前步骤 System Prompt +       │
│     上游交付物文件路径，不加载全量历史内容            │
│  3. 环境准备：创建 artifacts/<task_id>/ 子目录       │
└───────────────────────────────────────────────────┘
┌─ 步骤执行后 ──────────────────────────────────────┐
│  1. 交付物写入：原子写入（.tmp→rename）至指定路径    │
│  2. 上下文清除：删除当前上下文中的临时工具输出、       │
│     中间推理、调试日志，仅保留交付物路径引用          │
│  3. 血缘注册：记录 step_id → parent_step_id 映射     │
└───────────────────────────────────────────────────┘
```

### 1.3 交付物血缘图谱

所有步骤的交付物构成有向无环图（DAG），通过 `parent_step_id` 追踪：

```
req_analysis_v1 (parent: null)
├── learn_arch_v1 (parent: req_analysis_v1)
│   └── design_v1 (parent: learn_arch_v1)
└── learn_impl_v1 (parent: req_analysis_v1)
       └── design_v1 (同上)
              ├── skeleton_v1 (parent: design_v1)  [强制]
              │   └── ui_ue_design_v1 (parent: skeleton_v1)  [按需]
              │        └── implement_v1 (parent: ui_ue_design_v1 | skeleton_v1)
              └── implement_v1 (parent: skeleton_v1)  [无UI时]
                     └── test_v1 (parent: implement_v1)
                            └── integrate_v1 (parent: test_v1)
```

血缘图谱存储：`~/edu-hub/artifacts/<task_id>/lineage.json`

```json
{
  "task_id": "c008-v2-upgrade",
  "steps": {
    "req_analysis_v1": { "parent": null, "artifact_path": "req_analysis_v1.json", "status": "success" },
    "learn_arch_v1": { "parent": "req_analysis_v1", "artifact_path": "learn_arch_v1.json", "status": "success" }
  }
}
```

### 1.4 错误处理与选择性重跑

```
步骤执行失败？
  ├─ 生成 error 交付物到 artifacts/<task_id>/<step_id>.error.json
  │   ├─ step_id: 同正常交付物
  │   ├─ status: "error"
  │   ├─ error: { "type": "validation_failed" | "ollama_timeout" | "build_error",
  │   │           "detail": "具体错误信息",
  │   │           "recovery_hint": "检查 X 文件 Y 行" }
  │   └─ 写入 lineage.json 标记 status=error
  │
  └─ 重跑策略：
      ├─ 从失败的 step_id 开始重跑
      ├─ 自动重新验证其 parent_step_id 的交付物
      ├─ 该步骤下游的所有状态标记为 stale
      └─ 无需重跑上游已成功的步骤

选择性重跑命令:
  python3 ~/.hermes/scripts/c005-retry.py \\
    --task-id <task_id> \\
    --from-step <step_id> \\
    [--force]  # 强制重跑即使 status=success
```

### 1.5 存储基础设施

```bash
~/edu-hub/artifacts/
  └── <task_id>/                          # 每次 C005 执行创建一个
      ├── lineage.json                    # 血缘图谱
      ├── req_analysis_v1.json            # Step 0 交付物
      ├── learn_arch_v1.json              # Step 1A 交付物
      ├── learn_impl_v1.json              # Step 1B 交付物
      ├── design_v1.json                  # Step 2 交付物
      ├── skeleton_v1.json                # Step 2.5 交付物 [强制]
      ├── ui_ue_design_v1.json            # Step 2.6 交付物 [按需]
      ├── skeleton_v1/                   # Step 2.5 骨架（目录结构）
      │   ├── manifest.json              # 骨架清单
      │   ├── python_skeleton/           # Python 骨架文件
      │   ├── template_skeleton/         # HTML 骨架文件
      │   └── dependency_graph.md        # 方法调用/页面引用关系
      ├── implement_v1.json               # Step 3 交付物
      ├── test_v1.json                    # Step 4 交付物
      ├── integrate_v1.json               # Step 5 交付物
      ├── test_v1.error.json              # 错误交付物 (如有)
      └── logs/
          └── step_implement_v1.log        # 步骤执行日志

# 自动清理：任务完成后保留 7 天，cron 清理
```

---

## 二、入口前置检查：C007 Mode D（Bug 修复专用）

> ⚠️ **🔴 强制铁律：所有 Bug 修复必须先经 C007 顾问团前置审核** 🔴
>
> 用户明确要求（2026-05-06）：
> > 「后续的所有对系统问题修复的内容，务必请顾问团队介入，对问题进行总结，
> > 并控制好交付团队的整体质量，现在修复的问题越来越多」
>
> **忽视此规则的典型代价**（已在本项目实战验证）：
> 1. **表层修复掩盖深层根因** — 同一个 CSS display 问题连续 3 次修复才根除，
>    前两次都是「看起来简单直接修了」，结果只是移动了症状位置
> 2. **修复次数越多，用户信任度越低** — 每次反馈「还是不行」，都意味之前
>    的工作做了无用功
> 3. **质量螺旋下降** — 表面修好的同时可能引入新问题，不经根因分析的修复
>    无法判断是否带了新 Bug
>
> **执行铁律（违反一次记录一次）**：
> 1. **不得以「修复很简单」为由跳过 C007** — 「简单」只是你的判断，意味着
>    你没做根因分析就动手了
> 2. **不得以「之前修过类似的」为由跳过 C007** — 如果上次修复后问题还在，
>    说明之前的根因分析是错的，更不应重复错误
> 3. **所有 Bug 修复，无论大小，必须先由 C007 首席策略做一次问题总结**
>    — 哪怕只有 1 轮对话（3 个问题问清症状+影响范围+复现步骤）
> 4. **一旦用户说「又出现」或描述中提到「之前也遇到过」** — 自动升级为
>    Mode D 完整 5 阶段流程，跳过简化路径
> 5. **pre_c005_check 函数硬编码 bug_fix → route_to_c007_mode_d** —
>    此路由不可被「我觉得这是 Tier 0」绕过

**所有 Bug 修复任务，必须先走 C007 Mode D，再进入 C005 等级决策。**

> **用户沟通提示**：在告知用户需要 C007 Mode D 时，先简要解释流程（1-2句话）：
> 「C007 是顾问团系统，由多个专家角色做根因分析后再动手编码。模式 D 是 Bug 修复专用流程——先做根因分析再出修复方案，防止修了表面没修根因。」
> 用户可能不熟悉「C007 Mode D」这个内部术语，直接抛术语会让用户困惑（如反问「什么是C007 D？」）。先解释再推进。

**治理模式**: 模式 C（标记后跑）— 仅修 Bug 本身，不重构老代码。在修改过的文件中加入 `# TODO(refactor): ` 标记，并在交付物中记录「此文件尚有 N 处待治理」。

```python
def pre_c005_check(task_type: str) -> str:
    \"\"\"
    入口前置检查：判断是否需先走 C007 Mode D。

    返回: "continue_c005" | "route_to_c007_mode_d"
    \"\"\"
    bug_fix_keywords = [\"bug\", \"修复\", \"错误\", \"异常\", \"500\", \"404\",
                        \"不对\", \"不行\", \"坏了\", \"有问题\", \"错了\",
                        \"不跳转\", \"不显示\", \"不生效\", \"报错\"]

    if task_type == "bug_fix":
        return "route_to_c007_mode_d"
    return "continue_c005"
```

**规则**：
- 用户报告「系统有问题」→ **先启动 C007 Mode D**（问题总结→根因分析→方案设计）
- 用户要求「新增功能」→ 直接进入 C005 等级决策
- 用户要求「配置调整/文档修改」→ 直接进入 C005 Tier 0

C007 Mode D 输出根因分析报告和修复方案后，C005 再根据方案分配负载等级。

### 负载等级决策（入口评估）

**进入 C005 流程前，先评估修改范围，确定 C005 等级：**

```
输入：需求分析完成后，已知修改目标
  ↓
评估：
  ├─ 修改行数（估算）
  ├─ 涉及文件数
  ├─ 是否跨模块/涉及API变更
  ├─ 是否涉及交互页面功能（前端界面、H5应用、交互式组件）
  └─ 是否有现成测试用例
  ↓
┌─────────────────────────────────────────────────────────────┐
│                      C005 负载等级决策树                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  < 20 行 + 单文件 + 无依赖 + 有测试用例                       │
│    → ★☆☆ Tier 0: 极速通道                                    │
│       (直接实现→测试→交付，跳过全部子代理)                     │
│       适用: 文档修正、配置调整、简单 BUG 修复                   │
│                                                              │
│  20-100 行 + 1-2 文件                                        │
│    → ★★☆ Tier 1: 轻量级                                      │
│       (1×学习→实现→测试，跳过架构设计)                         │
│       适用: 单一功能增强、非关键路径修改                        │
│                                                              │
│  100-500 行 + 2-5 文件                                        │
│    → ★★★ Tier 2: 标准级                                      │
│       (2×学习→架构→实现→测试，当前 v1.3 的简化版)              │
│       适用: 模块内部重构、中等功能开发                          │
│                                                              │
│  >500 行 或 跨模块 或 API变更 或 数据库结构变更                │
│    → ★★★★ Tier 3: 完整级                                     │
│       (3×学习→架构→实现→测试，全流程)                          │
│       适用: 系统级重构、新系统搭建                              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────────┘

> **💡 纯视觉/UI 主题改造的加速路径**
>
> 当任务类型为**纯 CSS/视觉改造**（不改逻辑、不改API、不改数据结构）时，
> 即使规模匹配 Tier 2/3，也有权跳过 Step 1（代码学习）和 Step 2（架构设计）：
>
> ```
> 正常流程：Step 0 → Step 1(学习) → Step 2(设计) → Step 2.5(骨架) → Step 2.6(UI) → Step 3(实现) → ...
> 加速路径：Step 0 → Step 2.5(骨架) → Step 2.6(UI/UE设计) → Step 3(实现) → Step 4(测试) → Step 5(交付)
>                     ↑
>               跳过 Step 1 和 Step 2
>               （因为无逻辑代码需要学习，无架构需要设计）
> ```
>
> **判定条件**（全部满足才可加速）：
> 1. 所有变更文件仅涉及 HTML/CSS（无 .py 修改）
> 2. 无 API 端点、路由、数据库变更
> 3. 无 JavaScript 逻辑变更
> 4. 用户已确认视觉方向（通过 Step 0）
>
> Step 4 质量测试仍须执行，但测试重点从"逻辑正确性"转为"页面渲染完整性"。

**自动化辅助**：执行入口时可运行 `pre-c005-assess` 脚本自动推荐等级：

```bash
python3 ~/.hermes/scripts/pre-c005-assess.py \
  --target-files "path/to/file1.py path/to/file2.py" \
  --description "修改说明"
```

---

## 三、按等级的完整流程定义（交付物驱动版）

## 老代码治理策略：渐进边界法

> 每次 C005 任务改造现有代码时，不仅要交付新功能，还要将被触碰的代码提升到当前标准。
> 核心原则：不欠新债，逐步还旧债。

### 改幅分级

| 改幅 | 对应 Tier | 治理模式 | 要求 |
|:----|:---------|:---------|:-----|
| **大改** (>100行) | Tier 2+ | **模式 A**: 先修后改 | Step 2.4 合规审计 → 重构骨架 → 填充逻辑 |
| **小改** (20-100行) | Tier 1 | **模式 B**: 只修不动 | 仅修复直接触碰的代码行，其余打标记 |
| **Bug修复** | C007 Mode D | **模式 C**: 标记后跑 | 仅修 Bug，文件中留 TODO(refactor) 标记 |

### 红线规则（必须修复，不可放行）

| 违规项 | 检查方式 | 对应陷阱 |
|:-------|:---------|:---------|
| `href="#"` 在导航链接中 | `grep -n 'href="#' templates/*.html` | 陷阱28 |
| 硬编码 API 路径 | `grep -nP "['"]/api/v[0-9]/" routes.py` | B3 |
| 目录与内置模块同名 | `find . -type d -name 'math\|os\|sys\|io'` | 陷阱1 |
| 模板变量名与 route 参数不匹配 | 骨架阶段自动比对 | 陷阱26 |
| 数据字段双重语义 | 代码审查 | 陷阱23 |
| 双返回路径只改其一 | `grep -n 'return jsonify' *.py | 陷阱29 |

### 治理节奏

```
每次 C005 任务 = 为老代码做一次「局部翻新」

  time →  老代码健康度
  T1: 修功能A → 清理 file1.py (硬编码修复+type hints)    80%  ↑
  T2: 修功能B → 清理 file2.py (return路径统一)           85%  ↑
  T3: 新功能C → 清理 file3.py (目录改名)                 90%  ↑
  T4: Bug修复 → 标记 file4.py (TODO refactor)           90%  →
                —— 健康度总体上升，不是下降 ——
```

---

### ★☆☆ Tier 0: 极速通道

**条件**: < 20 行 + 单文件 + 无依赖 + 有测试用例

**治理模式**: 模式 B（只修不动）— 仅修复直接触碰的代码行，其余不动。

**交付物模式**: 极速通道视为单一原子步骤，输出交付物直接到 `artifacts/<task_id>/step_tier0_v1.json`

**流程**:
```
我(DeepSeek R1) → 直接分析(理解代码上下文)
   → 验证输入交付物(如有父步骤)
   → 直接实现(patch/写代码)
   → 直接测试(run 现有测试用例)
   → 输出交付物: {step_id: "tier0_v1", parent_step_id: null, status: "success",
       result: {files_modified: [...], test_results: {...}}}
   → 交付结果
```

**模型**: 我本身（DeepSeek R1）— 不创建任何子代理

**输出预算**: 无限制（单次会话，无中间报告）

**Step 5 集成交付**: 仅 Memory（一句摘要），跳过 Registry/Skill 更新

**禁止**: 跳过测试步骤 — 至少跑一次现有测试用例

---

### ★★☆ Tier 1: 轻量级

**条件**: 20-100 行 + 1-2 文件

**治理模式**: 模式 B（只修不动）— 仅修复直接触碰的代码行（硬编码路径、死链接、类型注解缺失），其余代码打 `# TODO(refactor): ` 标记。

**交付物模式**: 每个子步骤输出独立交付物到 `artifacts/<task_id>/`

**流程**:

#### Step 0: 需求分析（我 - DeepSeek R1）
- 理解需求 → 逻辑拆解 → 列出约束条件
- **等待用户确认后才进入下一阶段**
- 输出交付物 `artifacts/<task_id>/req_analysis_v1.json`
  ```json
  {
    "step_id": "req_analysis_v1",
    "parent_step_id": null,
    "status": "success",
    "result": {
      "requirements": ["...", "..."],
      "constraints": ["..."],
      "scope": "功能增强" | "缺陷修复",
      "has_ui": false,
      "target_files": ["path/to/file.py"]
    },
    "next_step": "learn"
  }
  ```
  `next_step` 根据任务类型决定：`"learn"`（正常）、`"skeleton"`（C007 加速）、`"implement"`（绿地加速）

**环境自动检测**：Step 0 输出后立即执行，注入到 Step 1 的 context 中：
```bash
python3 ~/.hermes/scripts/environment-audit.py --context
```
检测结果包含：平台类型/GPU状态/Ollama版本/磁盘空间/端口状态等已知限制，加入后续步骤的 context 中作为 `platform_constraints` 约束。

#### Step 1: 代码学习（1个子代理）

**前置步骤：Token Savior 调用者分析（v3.9 新增）**
在派发代码学习子代理前，先用 Token Savior 对 target_files 中的关键函数/类执行 `get_dependents` 分析，识别被广泛调用的公共符号。这些符号的签名应标记为「不可修改」，纳入 `no_touch_zones`。

```python
# Step 0 交付物 target_files 中的关键符号
# 用 Token Savior 检查每个符号的被调用范围：
for symbol in key_symbols:
    dependents = get_dependents(symbol)
    if len(dependents) >= 3:
        no_touch_zones.append(f"{symbol}（被 {len(dependents)} 处调用，不可修改签名）")
```

分析结果合并到子代理的 context 中：

```python
context = f"""
任务描述: {读取 artifacts/<task_id>/req_analysis_v1.json 中的 requirements}
目标文件: {读取 artifacts/<task_id>/req_analysis_v1.json 中的 target_files}
修改目标: {读取 artifacts/<task_id>/req_analysis_v1.json 中的 scope}

📌 Token Savior 调用者分析：
{json.dumps(token_savior_no_touch_zones, indent=2, ensure_ascii=False)}

⚠️ 交付物约束:
1. 输出为结构化 JSON 交付物格式
2. 总输出不超过 1,000 tokens
3. 写入 artifacts/<task_id>/learn_v1.json
"""
```

**模型**: **DeepSeek R1**（云端模型，所有角色统一使用云端推理）

**输出交付物** (`artifacts/<task_id>/learn_v1.json`):
```json
{
  "step_id": "learn_v1",
  "parent_step_id": "req_analysis_v1",
  "status": "success",
  "result": {
    "key_findings": ["发现1", "发现2", "发现3"],
    "no_touch_zones": ["函数X（被N处调用，不可修改签名）"],
    "implementation_notes": "实现时注意事项",
    "token_savior_verified": true
  },
  "next_step": "implement"
}
```

#### Step 2: 架构设计
**跳过** — Tier 1 不涉及复杂架构变更。

#### Step 2.5: 骨架搭建（强制）

**适用**: 所有 Tier 1+ 任务，Step 2(架构设计)之后必经步骤。

**模型**: 我（DeepSeek R1）— 不创建子代理，直接完成

**产出目录**: `artifacts/<task_id>/skeleton_v1/`

**产出物**:
```
skeleton_v1/
  ├── python_skeleton/   # .py骨架（import + class + def签名 + route + return结构）
  ├── template_skeleton/ # .html骨架（extends + block + 变量引用）
  └── dependency_graph.md
```

**骨架验证清单**:
```
□ 所有 import 已就位（可执行）
□ 所有 class/def 已定义（含类型注解）
□ 所有 route 装饰器 + url_for 已使用
□ 所有 return 结构已定型（多路径一致）
□ compile() 语法检查通过
□ 无 hardcoded 路径
□ 无 href="#" 占位
□ 目录名不与 Python 内置模块冲突
□ 模板变量名与 route 参数名一致
```

**输出交付物** (`artifacts/<task_id>/skeleton_v1.json`):
```json
{
  "step_id": "skeleton_v1",
  "parent_step_id": "design_v1",
  "status": "success",
  "result": {
    "files_created": ["path/to/engine.py", "path/to/template.html"],
    "syntax_check": "PASS",
    "hardcoded_paths_found": 0,
    "variable_mismatches_found": 0,
    "return_path_consistency": "PASS"
  },
  "next_step": "ui_ue_design|implement"
}
```

#### Step 2.6: UI/UE 设计（按需）

**条件**: 仅当任务涉及交互页面功能时启用（由 Step 0 需求分析判定）

**模型**: 我（DeepSeek R1）— 不创建子代理，直接完成

**输出交付物** (`artifacts/<task_id>/ui_ue_design_v1.json`):
```json
{
  "step_id": "ui_ue_design_v1",
  "parent_step_id": "req_analysis_v1",
  "status": "success",
  "result": {
    "interaction_flow": "...",
    "page_layout": "...",
    "components": [...],
    "visual_style": "...",
    "implementation_notes": "..."
  },
  "next_step": "implement"
}
```

#### Step 3: 代码实现（1个子代理）

```python
# 注入内容：仅交付物路径
context = f"""
学习交付物: artifacts/<task_id>/learn_v1.json
{如有UI设计: UI/UE交付物: artifacts/<task_id>/ui_ue_design_v1.json}
需求: {从 req_analysis_v1.json 读取}

⚠️ 约束:
1. 修改完成后写入 artifacts/<task_id>/implement_v1.json
2. 交付物中包含 files_modified 和 verification_steps
3. 不可调用 clarify
"""
```

**模型**: **DeepSeek R1**（云端模型，所有角色统一使用云端推理）

**后置验证（强制）**: 每次 patch() 后对修改文件执行：
```bash
# 语法检查 + 硬编码路径 + 内置模块冲突检测
python3 ~/.hermes/scripts/post-patch-verify.py path/to/file.py
# 同一文件 ≥ 3 次 patch → 读全量后用 write_file 重写
```

**输出交付物** (`artifacts/<task_id>/implement_v1.json`):
```json
{
  "step_id": "implement_v1",
  "parent_step_id": "learn_v1",
  "status": "success",
  "result": {
    "files_modified": [{"path": "...", "change_type": "create|patch|delete"}],
    "patch_count_per_file": {"file.py": 2},
    "post_patch_verify": "PASS",
    "verification_steps": ["python3 -c \"...\""],
    "patches_applied": 3
  },
  "next_step": "test"
}
```

#### Step 4: 质量测试（1个子代理 + 我独立验证）

⚠️ **子代理自验证不是完整测试**。子代理返回后，我必须独立编写并运行正式测试套件（阶段 B）。

**阶段 A: 子代理测试**

```python
context = f""""
实现交付物: artifacts/<task_id>/implement_v1.json
验证步骤: {从 implement_v1.json 读取 verification_steps}

⚠️ 约束:
1. 测试结果写入 artifacts/<task_id>/test_v1.json
2. PASS/FAIL 逐条记录
3. 发现问题 → 在子代理内修复→重新测试，只返回最终 PASS
""""
```

**输出交付物** (`artifacts/<task_id>/test_v1.json`):
```json
{
  "step_id": "test_v1",
  "parent_step_id": "implement_v1",
  "status": "success",
  "result": {
    "test_results": [
      {"name": "语法检查", "status": "PASS"},
      {"name": "导入检查", "status": "PASS"},
      {"name": "功能测试", "status": "PASS", "detail": "3/3 通过"}
    ],
    "coverage": "85%",
    "fix_cycles": 1
  },
  "next_step": "integrate"
}
```

**阶段 B: 我独立验证**

子代理测试通过后，我必须编写并运行独立的正式测试套件（覆盖象限 A+B 至少），验证通过后才进入 Step 5。测试文件独立于子代理的验证代码，确保不被子代理的"启动检查"或"简单运行验证"所替代。

**阶段 C: 自动化质量门禁（强制）**

所有涉及 edu-hub 教育系统的 Tier 1+ 任务，**必须**在 Step 4 末尾运行自动化质量门禁扫描。详见 `references/quality-gate-integration.md`。

门禁能力：
- L1 静态扫描（7项，<1min）：死链/API存根/模板冲突/导入/硬编码/函数引用/变量不匹配
- L2 路由冒烟（5项，<3min）：可达性/内容完整性/导航连通性/重定向/静态资源
- 硬门禁：P0 错误 → exit code 1 → 阻断进入 Step 5

> **⚠️ 纯视觉改造的门禁覆盖规则**
>
> 对于 CSS-only/视觉主题改造，质量门禁可能报告与本次变更无关的预存问题
> （死链 `href="#"`、模板变量不匹配、路由认证 403 等）。
> 按 `references/quality-gate-pre-existing-issues.md` 的 Step 2 分类方法：
> - 问题文件不在本次 target_files 中 → **非阻断，记录在案**
> - 问题文件在本次 target_files 中 → **必须修复后才可进入 Step 5**
> - L2 认证 403 属于正常鉴权 → **始终忽略**
>
> 在交付物中记录分类结果并在 `blocking_override_justified` 字段置 true。

```
# 强制调用方式
import subprocess, os, sys
quality_gate = os.path.expanduser("~/edu-hub/quality_gate/run_quality_gate.py")
if os.path.exists(quality_gate):
    result = subprocess.run(
        ["python3", quality_gate, "--system", system_id, "--task-id", task_id],
        capture_output=True, timeout=600
    )
    print(result.stdout.decode() if result.stdout else "")
    if result.returncode != 0:
        print(f"🔴 质量门禁阻断：{system_id} 存在 P0 错误，请修复后重试")
        sys.exit(1)
```

#### Step 5: 集成与交付（我 - DeepSeek R1）
- 验证 test_v1.json status=success
- 将通过的代码集成
- 输出最终交付物 `artifacts/<task_id>/integrate_v1.json`
- **保存 Memory**（一句变更摘要）
- 交付结果
- **不更新** Registry 和 Skill

---

### ★★★ Tier 2: 标准级

**条件**: 100-500 行 + 2-5 文件

**治理模式**: 模式 A（先修后改）— Step 2.4 合规审计 → Step 2.5 重构骨架 → Step 3 填充新逻辑。

**交付物模式**: 各子代理独立输出交付物，通过血缘图谱 DAG 进行依赖管理

#### Step 0: 需求分析（我 - DeepSeek R1）
- 同 v1.3，等待用户确认。
- 输出交付物 `artifacts/<task_id>/req_analysis_v1.json`（格式同 Tier 1 Step 0）

#### Step 1: 代码学习（2个并行子代理，各输出独立交付物）

每个子代理输出交付物到独立路径：

```python
# 子代理 A — 架构视角
context_a = f"""
目标文件: {files}
交付物路径: artifacts/<task_id>/learn_arch_v1.json

⚠️ 交付物约束:
1. 学习模块依赖、数据流向、外部接口
2. 输出为结构化 JSON 交付物格式
3. 写入 artifacts/<task_id>/learn_arch_v1.json
"""

# 子代理 B — 实现视角
context_b = f"""
目标文件: {files}
交付物路径: artifacts/<task_id>/learn_impl_v1.json

⚠️ 交付物约束:
1. 学习代码风格、边界条件、已有陷阱
2. 仅报告与 learn_arch_v1 的差异部分
3. 写入 artifacts/<task_id>/learn_impl_v1.json
"""
```

**分区读取规则（严格）**:
```
当目标文件 > 400 行时，按行分区：

子代理 A: 读 1-200 行（配置层 + 数据模型 + 接口定义）
子代理 B: 读 200+ 行（核心算法 + 主流程 + 辅助函数）

重叠率目标: < 10%
```

#### Step 2: 架构设计（1个子代理）

**前置步骤：Token Savior 影响链分析（v3.9 新增）**

在派发架构设计子代理前，先使用 Token Savior MCP 工具自动分析修改目标的影响链，生成机器验证的依赖图谱，注入到子代理的 context 中。

**执行流程：**

```
Step 0 交付物 (req_analysis_v1.json) 中的 target_files
  ↓
从 target_files 提取关键符号（函数名、类名、路由处理器名）
  ↓
并行调用 Token Savior 工具：
  ├─ get_dependents(symbol)     — 谁调用了这个符号（影响范围）
  ├─ get_dependencies(symbol)   — 这个符号依赖了谁（修改边界）
  ├─ get_change_impact(symbol)  — 完整影响链分析（transitive）
  ├─ find_symbol(name)          — 精确定位每个符号的位置
  └─ get_symbol_cluster(name)   — 功能聚类（同一功能区的相关符号）
  ↓
生成结构化的 token-savior-impact-report，包含：
  {
    "verified_dependents": [{"symbol": "caller_func", "file": "app.py:50", "confidence": 1.0}],
    "verified_no_touch_zones": [{"symbol": "shared_func", "reason": "被15处调用", "file": "..."}],
    "call_chains": [{"from": "new_feature", "to": "legacy_api", "path": ["A.py", "B.py", "C.py"]}],
    "risks": ["符号 X 有 8 个隐含调用者，修改签名需全局更新"]
  }
  ↓
影响报告注入到架构设计子代理的 context 中
```

**注入方式**：在 Step 2 子代理的 context 中，将 Token Savior 影响报告作为独立上下文块追加：

```python
context = f"""
...
📌 Token Savior 影响链分析（机器验证）:
{json.dumps(impact_report, indent=2, ensure_ascii=False)}

⚠️ 影响报告中的 verified_dependents 和 verified_no_touch_zones
   是机器从 AST 依赖图推导的，必须纳入 patch_dependency_graph
"""
```

**适用条件**：
- Token Savior MCP 服务器已运行（`hermes mcp list` 确认 token-savior 状态为 enabled）
- edu-hub 项目已索引完成（`list_projects` 显示 active）
- 当 Token Savior 不可用时，回退到人工分析（原有流程不变）

> 完整工具能力映射、已知限制（symbol 参数期望函数名而非模块路径、get_routes 仅限 Next.js）、典型执行时序见 `references/token-savior-integration.md`。

**输出交付物** (`artifacts/<task_id>/design_v1.json`):
```json
{
  "step_id": "design_v1",
  "parent_step_id": "req_analysis_v1",
  "status": "success",
  "result": {
    "modules": [{"name": "模块A", "responsibility": "...", "dependencies": ["模块B"]}],
    "interfaces": [{"name": "API_X", "method": "POST", "path": "/api/v2/..."}],
    "data_flow": "描述数据流",
    "patch_dependency_graph": {
      "nodes": ["file_A.py", "file_B.py"],
      "edges": [{"from": "file_A.py", "to": "file_B.py", "type": "depends_on"}],
      "token_savior_verified": true,
      "impact_report_ref": "artifacts/<task_id>/token_savior_impact_v1.json"
    },
    "implementation_priority": ["Task A", "Task B"],
    "risks": ["风险1：..."],
    "tool_assisted": ["token_savior"]
  "next_step": "audit"
  }
}
```

#### Step 2.4: 合规审计（模式 A 专属）

> **目标**：在动手重构前，扫描 target_files 中违反现行规范的部分，列出「必须修复」「推荐修复」「仅标记」三类。

**执行方式**（我直接执行，不创建子代理）：

```bash
# 1. 扫描硬编码路径
grep -nP "['"]/api/v[0-9]/" routes.py
grep -nP "['"]http://localhost" *.py

# 2. 扫描死链接
grep -n 'href="#[^ ]' templates/*.html

# 3. 扫描无类型注解的函数
for f in target_files/*.py; do
  grep -nP "^def \w+\([^)]*\):" "$f" | grep -v ": int\|: str\|: bool\|: list\|: dict"
done

# 4. 扫描双返回路径
for f in target_files/*.py; do
  grep -c "return jsonify" "$f" | grep -v ":1$"
done
```

**输出交付物** (`artifacts/<task_id>/audit_v1.json`):
```json
{
  "step_id": "audit_v1",
  "parent_step_id": "design_v1",
  "status": "success",
  "result": {
    "must_fix": [
      {"file": "routes.py:50", "issue": "硬编码API路径 '/api/v2/list'", "principle": "P3 url_for强制"},
      {"file": "templates/portal.html:22", "issue": "href="#" 死链接", "principle": "P8 永不覆盖"}
    ],
    "recommend_fix": [
      {"file": "engine.py:15", "issue": "函数 generate() 缺类型注解", "principle": "P5 骨架优先"}
    ],
    "mark_only": [
      {"file": "state.py:30-80", "issue": "get_state() 变量命名不规范", "principle": "P3 命名规范"}
    ]
  },
  "next_step": "skeleton"
}
```

#### Step 2.5: 骨架搭建（强制 + 含老代码重构）

**骨架产出**:
- 输出 `artifacts/<task_id>/skeleton_v1/` 目录
- Python: import + class/def签名 + route装饰器 + return结构
  — 先按合规审计修复 mandatory 项（硬编码→url_for、href→真实路径）
  — 再搭新骨架
- HTML: extends + block + 变量引用名 + url_for
- 类型覆盖矩阵：后端输出类型 ↔ 前端处理路径
- 依赖关系图：完整方法调用/页面引用
- 验证：compile() 语法检查 + 模板文件存在性 + 变量名一致性

**输出交付物** (`artifacts/<task_id>/skeleton_v1.json`):
```json
{
  "step_id": "skeleton_v1",
  "parent_step_id": "design_v1",
  "status": "success",
  "result": {
    "files_created": ["path/to/engine.py", "path/to/template.html"],
    "syntax_check": "PASS",
    "hardcoded_paths_found": 0,
    "variable_mismatches_found": 0,
    "return_path_consistency": "PASS",
    "type_coverage_matrix": {"types": [...], "gaps": [...]}
  },
  "next_step": "ui_ue_design|implement"
}
```

#### Step 2.6: UI/UE 设计（按需）

**条件**: 仅当任务涉及交互页面功能时启用

**输出交付物** (`artifacts/<task_id>/ui_ue_design_v1.json`):
```json
{
  "step_id": "ui_ue_design_v1",
  "parent_step_id": "skeleton_v1",
  "status": "success",
  "result": {
    "interaction_flow": "交互流程与用户操作路径（含多状态分支）",
    "page_layout": "页面布局与区域划分",
    "components": ["组件清单"],
    "visual_style": "视觉风格规范",
    "state_management": {"loading": "...", "empty": "...", "error": "...", "boundary": "..."},
    "responsive_behavior": "响应式行为说明",
    "implementation_notes": "实现注意事项"
  },
  "next_step": "implement"
}
```

#### Step 3: 代码实现（1个子代理 + 依赖图调度）

```python
# 注入：仅交付物路径
context = f"""
设计交付物: artifacts/<task_id>/design_v1.json
{如有UI: UI/UE交付物: artifacts/<task_id>/ui_ue_design_v1.json}
学习交付物A: artifacts/<task_id>/learn_arch_v1.json
学习交付物B: artifacts/<task_id>/learn_impl_v1.json
需求: {从 req_analysis_v1.json 读取}

⚠️ 约束:
1. 写入 artifacts/<task_id>/implement_v1.json
2. 交付物包含 files_modified 和 verification_steps
3. 每次 patch() 后运行后置验证：
   python3 ~/.hermes/scripts/post-patch-verify.py <修改的文件>
4. 同一文件 ≥ 3 次 patch → 读全量后用 write_file 重写
"""
```

**依赖图调度**:
```
Task A (无依赖) ─┐
Task B (无依赖) ─┤→ delegate_task(tasks=[A,B]) 并行
                  │
Task C (依赖A)   ─┘→ 等A完成后单独派C
所有任务输出独立交付物到 artifacts/<task_id>/task_A_v1.json 等
```

#### Step 4: 质量测试（1个子代理 + 我独立验证）

⚠️ **子代理自验证不是完整测试**。子代理返回后，我必须独立编写并运行正式测试套件（阶段 B），覆盖所有 4 个象限。

**阶段 A: 子代理测试**

```python
context = f""""
实现交付物: artifacts/<task_id>/implement_v1.json
验证步骤: {从 implement_v1.json 读取}

⚠️ 约束:
1. 测试矩阵预定义（实现前即定义完整用例清单）
2. 写入 artifacts/<task_id>/test_v1.json
3. 发现问题 → 子代理内修复循环，只返回最终 PASS
""""
```

**阶段 B: 我独立验证**

子代理返回 test_v1.json status=success 后，我必须独立编写并运行正式测试套件，覆盖 4 个象限（UT/IT-API/IT-HTML/BT），见下方"测试矩阵模板"。测试文件必须新创建，独立于子代理的"启动验证"代码，确保真正的全覆盖。

**不可跳过阶段 B 的场景**（即使子代理报告全部 PASS）：
- 新增 3+ 个文件
- 涉及 API 端点变更
- 涉及前端页面新增
- 用户明确要求"测试全覆盖"

**修复循环隔离**:
```
测试发现问题？
  ├─ 小问题(1-3行修改) → 子代理内修复→重新测试
  │                       (子代理内循环，父上下文零开销)
  │                       test_v1.json 标记 fix_cycles=N
  │
  └─ 大问题(架构缺陷) → delegate_task(新建子代理)
                          (全新上下文，加载交付物路径)
                          (只返回最终 PASS 结果)
```

#### Step 5: 集成与交付（我 - DeepSeek R1）
- 验证 test_v1.json status=success
- 代码集成
- 输出最终交付物 `artifacts/<task_id>/integrate_v1.json`
- **现有能力增强** → 增量更新：Skill 版本号升版 + Memory
- 交付结果
- **不更新** Registry（除非新增独立能力）

---

### ★★★★ Tier 3: 完整级

**条件**: >500 行 或 跨模块 或 API变更 或 数据库结构变更

**治理模式**: 模式 A（先修后改）— 同 Tier 2，合规审计 + 骨架重构前置。

**交付物模式**: 全流程交付物 DAG，每个子代理独立输出，通过 lineage.json 追踪完整血缘

#### Step 0: 需求分析
- 同 Tier 2 Step 0，输出 `req_analysis_v1.json`

#### Step 1: 代码学习（3个并行子代理，各输出独立交付物）

```python
# 子代理 A — 架构视角 → artifacts/<task_id>/learn_arch_v1.json (1-200行)
# 子代理 B — 实现视角 → artifacts/<task_id>/learn_impl_v1.json (200-450行)
# 子代理 C — 测试视角 → artifacts/<task_id>/learn_test_v1.json (450+行)
```

**分区读取规则（严格执行）**:
```
当目标文件 > 400 行：
  子代理 A: 读 1-200 行（配置层 + 数据模型 + 接口定义）
  子代理 B: 读 200-450 行（核心算法/策略函数）
  子代理 C: 读 450+ 行（主流程 + 辅助函数 + 镜像版本差异）
重叠率目标: < 10%
```

**输出交付物**: 每个子代理独立 JSON 文件，B/C 通过 `diffs_from_A` / `diffs_from_AB` 字段标记差异。

#### Step 2: 架构设计
- 同 Tier 2 Step 2，输出 `design_v1.json`

#### Step 2.5: 骨架搭建（强制）
- 同 Tier 2 Step 2.5，输出 `skeleton_v1.json` + `/skeleton_v1/` 目录

#### Step 2.6: UI/UE 设计（按需）
- 同 Tier 2 Step 2.6，输出 `ui_ue_design_v1.json`

#### Step 3: 代码实现
- 同 Tier 2 Step 3，支持依赖图调度的并行任务交付物

#### Step 4: 质量测试
- 同 Tier 2 Step 4，输出 `test_v1.json`
- 修复循环隔离策略同 Tier 2

#### Step 5: 集成与交付（完整版）

```python
# 验证所有上游交付物 status=success
artifacts = ["req_analysis_v1.json", "design_v1.json", "implement_v1.json", "test_v1.json"]
for a in artifacts:
    validate_artifact(artifacts/<task_id>/a)  # 校验 checksum + status

# 生成最终交付物
integrate_v1.json = {
  "step_id": "integrate_v1",
  "parent_step_id": "test_v1",
  "status": "success",
  "result": {
    "change_type": "new_capability" | "enhancement" | "bugfix",
    "files_changed": [...],
    "delivery_actions": [...]
  }
}
```

```

## 四、模型选择矩阵（汇总）

| 角色 | Tier 0 | Tier 1 | Tier 2 | Tier 3 |
|:----|:------:|:-----:|:-----:|:------:|
| 需求分析（我） | DeepSeek R1 | DeepSeek R1 | DeepSeek R1 | DeepSeek R1 |
| 学习 A（架构） | — | — | **DeepSeek R1** | **DeepSeek R1** |
| 学习 B（实现） | — | DeepSeek R1 | DeepSeek R1 | DeepSeek R1 |
| 学习 C（测试） | — | — | — | DeepSeek R1 |
| 架构设计 | — | — | **DeepSeek R1** | **DeepSeek R1** |
| **骨架搭建** | — | **我** | **我** | **我** |
| **UI/UE 设计** | — | **我（按需）** | **DeepSeek R1（按需）** | **DeepSeek R1（按需）** |
| 代码实现 | 我 | DeepSeek R1 | **DeepSeek R1** | **DeepSeek R1** |
| 质量测试 | 我 | DeepSeek R1 | DeepSeek R1 | DeepSeek R1 |
| 集成交付 | 我 | 我 | 我 | 我 |

**关键原则**:
- **全部角色统一使用 DeepSeek R1 云端模型** — 本地模型不再参与 C005 工作流
- 原本地模型（qwen3:14b、qwen3:8b）角色全部迁移至云端，确保推理能力一致性
- 我（DeepSeek R1）兼任需求分析/集成交付/极速通道操作
- **所有步骤输出交付物到 `artifacts/<task_id>/`** — 路径引用而非全量内容传递

**降级方案**（云端不可用）:
- 全部暂停工作流，待云端恢复后继续
- 无降级到本地模型的通路 —— 所有角色强制云端

---

## 五、子代理通信协议（交付物版）

### 5.1 交付物输出格式

所有子代理输出 **结构化 JSON 交付物文件** 到指定路径，而非在 context 中返回。格式：

```json
{
  "step_id": "learn_arch_v1",
  "parent_step_id": "req_analysis_v1",
  "status": "success",
  "next_step": "design",
  "generated_at": "2026-04-29T20:00:00",
  "checksum": "sha256:abc123def456",
  "result": {
    "key_findings": ["发现1", "发现2", "发现3"],
    "no_touch_zones": ["函数X（被N处调用，不可修改签名）"],
    "impact_assessment": "此修改会影响/不影响XX功能",
    "diffs_from_others": "与前面子代理的不同发现（如有）",
    "confidence": "high/medium/low"
  },
  "metadata": {
    "model": "DeepSeek R1",
    "execution_seconds": 45,
    "token_cost": 12500
  }
}
```

### 5.2 输出预算

| 角色 | 最大 tokens | 最大要点数 | 交付物路径 |
|:----|:----------:|:---------:|:----------|
| 需求分析 | 1,000 | 10 | `req_analysis_v1.json` |
| 学习代理 (每个) | 1,500 | 10 | `learn_*_v1.json` |
| 架构代理 | 3,000 | 15 | `design_v1.json` |
| 骨架搭建 | 1,500 | 10 | `skeleton_v1/` + `skeleton_v1.json` |
| UI/UE 设计代理 | 2,000 | 12 | `ui_ue_design_v1.json` |
| 实现代理 | 不限（但子代理内部完成验证） | — | `implement_v1.json` |
| 测试代理 | 2,000 | 20（含每条PASS/FAIL） | `test_v1.json` |

### 5.3 差异报告规则

- 子代理 A：完整报告 → `learn_arch_v1.json`
- 子代理 B：**仅报告与 A 的不同发现** → `learn_impl_v1.json` 中通过 `diffs_from_A` 字段
- 子代理 C：**仅报告与前两者的不同发现** → `learn_test_v1.json` 中通过 `diffs_from_AB` 字段
- **不得重复已报告内容**

### 5.4 固定模版

每个 delegate_task 的 context 末尾固定追加以下提示：

```
⚠️ 交付物约束:
1. 输出为结构化 JSON 文件到指定路径
2. 总输出不超过预算
3. 包含 step_id, parent_step_id, status, checksum 字段
4. 仅报告新发现，不重复已有内容
5. 不可调用 clarify 询问用户
6. 如发现问题，在子代理内部修复循环，只返回最终 PASS 结果
```

---

## 测试矩阵模板（Step 4 必备）

任何 Tier 2/3 的 Step 4 质量测试必须覆盖以下 4 个象限。Tier 1 至少覆盖象限 A+B。

```
象限 A: 单元测试 (UT) — 核心逻辑验证
  ├── 生成器/计算器测试: 每个生成函数至少验证输出格式和范围
  ├── 校验函数测试: 精确匹配、模糊匹配、边界格式
  ├── 空输入/无效输入测试: 空字符串、null、越界值
  └── 错误答案拒收测试: 确保非正确答案返回 False

象限 B: 集成测试 (IT-API) — API 端点验证
  ├── 健康检查: /health 返回 status=ok
  ├── 数据读取端点: 每个 GET 端点返回预期结构，含关键字段
  ├── 数据写入端点: POST 请求成功处理，返回预期结构
  ├── 输入校验: 空 body、缺失必填字段 → 返回 400
  └── 幂等性: 相同请求重复提交，结果一致

象限 C: 集成测试 (IT-HTML) — 页面渲染验证
  ├── 每个页面路由返回 HTTP 200
  ├── 页面包含标题关键字（Confirm the page title renders in the HTML）
  └── 关键 UI 元素关键字检查（Submit button, Navigation links, etc.）

象限 D: 边界测试 (BT) — 极端场景
  ├── 空数据状态: 无数据时页面不崩溃，显示友好提示
  ├── 无效参数: URL 参数缺失/错误时仍返回 200（非 500）
  ├── 并发/顺序依赖: 缓存状态导致的测试顺序问题（清理缓存策略）
  ├── 部分提交: 非完整数据集的提交
  └── 状态容错: 底层文件不存在/损坏时的优雅降级
```

**测试用例命名规范:**
```
UT-1xx: 生成器/计算逻辑
UT-2xx: 校验/验证函数
UT-3xx: 数据组装/格式化
UT-4xx: 核心业务流程
UT-5xx: 缓存/持久化
UT-6xx: 状态管理/外部依赖
IT-7xx: API集成测试
IT-8xx: HTML页面测试
BT-9xx: 边界/异常场景
```

**关键陷阱（从实战总结）:**

1. **测试顺序导致缓存污染** — 前面的测试如果修改了缓存文件（如不同参数生成练习），后面的 API 测试会读到脏数据。**修复**: API 集成测试前清除缓存，或每个独立测试使用 `force_refresh=True`
2. **相对导入在独立测试中失败** — `from . import module` 在直接 `python3 test.py` 运行时无父包上下文。**修复**: 在测试文件顶部设置 `sys.path.insert(0, ...)` 并尝试绝对导入做 fallback
3. **服务器热加载 vs 代码修改** — 运行中的 Flask 进程缓存了已导入模块。修改源代码后需重启服务器才能生效。测试应设计为可独立运行（不依赖运行中服务器）和可集成运行（依赖服务器）两种模式
4. **空断言错误消息无法定位** — `assert condition` 失败时不显示任何调试信息。**修复**: 始终使用 `assert condition, f"调试信息: {实际值}"` 格式

5. **Prompt 中包含答案（教育系统特有反模式）** — 当同一个函数返回的值既被用作"用户看到的提示文字"又被用作"正确答案"时，会出现提示文字中包含答案的 Bug。典型场景：`get_fallback_prompt()` 同时作为拼写题的 `prompt` 和 en2cn 题的 `answer` 的生成源。当数据缺失时（如 `cn==en`），fallback 提示文字中嵌入了单词本身，导致用户直接看到答案。
   **排查方法**：
   - 追踪数据流：哪些函数同时出现在 `prompt`/`cn` 字段和 `answer` 字段？
   - 检查所有 fallback/备用路径：`grep -rn 'f"请输入.*{en}.*正确拼写' *.py`
   - 验证 `cn==en` 的数据不会导致答案泄露：遍历所有词条，检查中文释义等于英文原文的情况
   - 确认前端模板渲染时 `qText.textContent = data.cn` 不会把含有答案的 fallback 提示暴露给用户
   **修复**：将"用户看到的提示文字"和"正确答案"拆分为两个不同的逻辑路径。提示文字中不应包含待猜测的答案。**预防**：在 Step 4 测试中增加"prompt 不含 answer"的断言检查：确保所有 question 的 `prompt`/`cn` 字段不包含其 `answer` 字段的内容

---

## 修复循环隔离策略

### 6.1 问题

v1.3 中测试发现的问题 → 打回 Step 2+3 → 修复 → 再测 → 循环 → **在父上下文中迭代，每次循环膨胀上下文**

### 6.2 解决方案

```
Step 3 (实现) → 输出 implement_v1.json → Step 4 (测试)
  ↓
测试发现问题？
  ├─ 小问题(1-3行修改) → 子代理内修复→重新测试
  │                       (子代理内循环，父上下文零开销)
  │                       更新 test_v1.json 标记 fix_cycles=N
  │
  └─ 大问题(架构缺陷) → delegate_task(新建子代理)
                          (全新上下文，仅加载交付物路径)
                          (只返回最终 PASS + 更新交付物)
  ↓
Step 5 集成交付: 读取 test_v1.json 验证 status=success
```

### 6.3 触发阈值

| 信号 | 动作 |
|:----|:-----|
| 测试发现 1-2 个边界遗漏 | 子代理内修复 |
| 测试发现逻辑错误 | 子代理内修复 |
| 测试发现架构问题（依赖缺失/接口不匹配） | **委托新子代理** |
| 同一文件 patch ≥ 2 次仍未通过 | **委托新子代理** |
| 修复后仍有 3+ 项 FAIL | **委托新子代理** |

---\n\n## Step 5.5: 系统知识自动同步（C009 集成）\n\n> **所有 Tier 的 Step 5 完成后，自动触发系统知识同步。**\n> 关联能力：C009 — 系统知识自动同步管道\n\n### 触发方式\n\n在每个 Tier 的 Step 5 末尾，追加以下调用：\n\n```python\n# 系统知识自动同步（C009）\nimport subprocess\nsync_script = os.path.expanduser(\"~/.hermes/scripts/system-knowledge-sync.py\")\nif os.path.exists(sync_script):\n    subprocess.run([\"python3\", sync_script, \"--task-id\", task_id],\n                   capture_output=True, timeout=30)\n```\n\n**无需等待完成** — sync 脚本独立执行，不影响 Step 5 返回结果。\n\n### 各 Tier 的行为\n\n| Tier | 同步内容 | 说明 |\n|:----|:---------|:------|\n| Tier 0 | 仅 changelog (铁层) | 极速通道无架构知识，只记录变更 |\n| Tier 1 | 金层+银层+changelog | 学习发现中的陷阱会被分类提取 |\n| Tier 2 | 金层+银层+铁层+changelog | 架构设计的模块/接口/数据流全部提取 |\n| Tier 3 | 全量 | 完整知识提取+向量化 |\n\n### 冷启动检测\n\n在 Step 0 末尾，**增加一个前置检查**：\n\n```python\n# 冷启动检测：目标系统是否有知识文档？\nimport os\nsystem_id = get_system_id_from_task(task_id)  # task_id 前缀映射\nsystem_index = os.path.expanduser(f\"~/wiki/raw/systems/{system_id}/index.md\")\nif not os.path.exists(system_index):\n    # 触发冷启动分析（轻量级，≤2K tokens）\n    subprocess.run([\"python3\", sync_script, \"--cold-start\",\n                    \"--system-id\", system_id, \"--skip-vectorize\"],\n                   capture_output=True, timeout=30)\n```\n\n### 手工操作\n\n当 C005 未完成（中途失败）但交付物已有时，可手动执行同步：\n\n```bash\n# 正常补跑\npython3 ~/.hermes/scripts/system-knowledge-sync.py --task-id <task_id>\n\n# 强制补跑（即使 C005 状态不是 success）\npython3 ~/.hermes/scripts/system-knowledge-sync.py --task-id <task_id> --force\n\n# 历史回溯（一次性补全所有已有系统）\npython3 ~/.hermes/scripts/system-knowledge-sync.py --backfill\n```\n\n### 知识存储位置\n\n```\n~/wiki/raw/systems/<C00X>/\n  ├── index.md             # 系统概览\n  ├── architecture.md      # 架构知识（金层+银层，差异写入）\n  ├── changelog.md         # 变更历史（追加，task_id 去重）\n  └── findings_iron.json   # 铁层知识（上限100条 FIFO）\n```\n\n### 前提条件\n\n- Ollama 运行中 + `qwen3-embedding:0.6b` 已加载（用于向量化）\n- 脚本 `~/.hermes/scripts/system-knowledge-sync.py` 已部署\n- 目录 `~/wiki/raw/systems/` 已存在\n\n---\n\n## 七、集成交付差异化策略

### 7.1 交付等级矩阵

```
修改类型判断:
  ├─ 新增端到端能力 → 全量更新: Registry + Skill + Memory
  │   (示例: 新增 C006 运营监控系统)
  │
  ├─ 现有能力增强 → 增量更新: Skill 版本号升版 + Memory
  │   (示例: C003 v4.4 → v4.5)
  │
  └─ 一次性缺陷修复 → 仅 Memory: 一句变更摘要
      (示例: 修复某个函数的边界条件)
      Memory: "修复了 X 函数的边界条件 Y"
```

### 7.2 交付三重奏模板（仅 Tier 2+ 的新增能力）

```
1. Registry: 版本号 + 变更日志条目 + 新增资产行
2. Skill: 版本号 + 组件表新行 + 用法说明/示例命令
3. Memory: 一句摘要(action=add, target=memory)
```

### 7.3 Tier 边界检查

```
交付前自检:
  □ 此修改是否是独立的新系统/能力？
    → 是 → Registry + Skill + Memory
    → 否 → 继续检查
  □ 此修改是否增强了现有能力的可交互性？
    → 是 → Skill + Memory
    → 否 → 仅 Memory
```

---

## 八、旧版迁移对照

| 旧版 v2.2 | 新版 v3.0 | 变化 |
|:----------|:----------|:-----|
| 步骤间全量 context 传递 | **交付物文件 + 路径引用** | **新增** 消除上下文膨胀 |
| 子代理输出通过 delegate_task 返回 | **子代理写入独立交付物文件** | **新增** 持久化可追溯 |
| 失败后全量重跑 | **选择性重跑：仅重跑失败步骤及其依赖链** | **新增** 节省 70%+ 重跑成本 |
| 无数据血缘追踪 | **lineage.json 血缘图谱 + parent_step_id** | **新增** 全链路可审计 |
| 无交付物校验 | **checksum 字段 + 输入验证契约** | **新增** 防损坏/篡改 |
| 所有编码走同一流程 | 4 级负载等级 | 继承 v2 |
| <50 行→快速通道 | <20 行→Tier 0, 20-100 行→Tier 1 | 继承 v2 |
| 全云端模型策略 | 全云端模型策略（不变） | 继承 v2 |
| 子代理云端 DeepSeek R1 | 子代理云端 DeepSeek R1（不变） | 继承 v2 |
| 修复循环在父上下文 | 子代理内修复 + 隔离委托 | 继承 v2 |
| 交付必更新 Registry+Skill+Memory | 差异化交付 | 继承 v2 |

---

## 九、预期效果

| 指标 | v2.2 基准 | v3.0 预估 | 减少 |
|:---|:--------:|:---------:|:----:|
| 父上下文输入 tokens（Tier 2 典型） | ~350K | ~120K | **-65%** |
| 子代理间重复信息传递 | ~60% | ~5%（仅路径引用） | **-92%** |
| 失败重跑范围 | 全量 | 单步+下游 | **-70%+** |
| 可审计性 | 无 | 全链路血缘图谱 | **新增** |
| 交付物持久化 | 无 | artifacts/ 目录 (保留7天) | **新增** |
| 错误恢复 | 手动重来 | 选择性重跑 + 错误交付物 | **新增** |
| 全流程时长（Tier 2） | ~1 小时 | ~50 分钟 | **-17%** |

---

## 新增陷阱 1：Python 内置模块命名冲突

**发现场景**：将 `~/edu-hub/math/` 作为 Flask Blueprint 的包目录，导致 `from math.v2 import math_v2` 失败，错误 `'math' is not a package`。

**根因**：Python 有内置 `math` 模块，与本地目录 `~/edu-hub/math/` 同名。即使通过 `sys.modules.pop('math')` 或自定义 `sys.meta_path` 钩子尝试覆盖，内置模块仍在启动时被缓存到 `sys.modules` 中。更致命的是，**Python 标准库（`random.py`、`email.utils` 等）内部从 `math` 导入函数**（`from math import log, exp, pi, e, ceil`），强行覆盖 `sys.modules['math']` 会使整个标准库崩溃。

**排查标志**：
```
ImportError: cannot import name 'log' from 'math' (/path/to/your/math/__init__.py)
```
这个错误出现在看似不相关的标准库导入中（如 `import random`），很容易误判为 Python 环境问题。

**解决方案**：**重命名目录**，不可与任何 Python 内置模块同名。检查方式：
```bash
python3 -c "import sys; print([m for m in sys.builtin_module_names if '内置模块名' in m])"
```

常见冲突名单（避免用作 Blueprint/包目录名）：
`math`, `os`, `sys`, `io`, `json`, `re`, `time`, `datetime`, `random`,
`collections`, `itertools`, `functools`, `pathlib`, `typing`, `logging`,
`http`, `socket`, `email`, `string`, `textwrap`, `base64`, `hashlib`,
`uuid`, `copy`, `types`, `enum`, `abc`, `inspect`, `ast`, `tokenize`

**替代方案**：加后缀（`math_sys`、`my_io`）或用子包名（`edu_math`、`app_os`）。

## 新增陷阱 2：Flask Blueprint url_prefix 替换而非拼接

**发现场景**：Blueprint 定义时指定 `url_prefix="/student"`，注册时又指定 `url_prefix="/english"`，期望最终路径为 `/english/student/portal`，实际却是 `/english/portal`。

**根因**：Flask 在注册 Blueprint 时，注册时的 `url_prefix` **完全替换** Blueprint 内部的 `url_prefix`，而非拼接。Flask 源码中 `BlueprintRegisterState` 类只保存注册时传入的 `url_prefix`，丢弃定义时的值。

```python
# Blueprint 定义
eng_v3 = Blueprint("eng_v3", __name__, url_prefix="/student")
eng_v3.add_url_rule("/portal", view_func=login_required(portal))

# 注册
app.register_blueprint(eng_v3, url_prefix="/english")
# 结果 URL：/english/portal  （不是 /english/student/portal）
```

**解决方案**：Blueprint 定义时**不设置** `url_prefix`，统一在注册时指定：
```python
eng_v3 = Blueprint("eng_v3", __name__)  # 不设 url_prefix
# 或者定义时用完整路径
eng_v3 = Blueprint("eng_v3", __name__, url_prefix="/english/student")
app.register_blueprint(eng_v3)  # 注册时不覆盖
```

**排查标志**：期望的 URL 路径比实际少了一级，或 Blueprint 内的 `url_for` 生成错误路径。

## 新增陷阱 7：路由处理器中引用未定义变量导致 500（NameError 无声陷阱）

**发现场景**：用户在练习页面点击「提交」或「不知道」按钮，前端收到 `{"error":"服务器内部错误"}`（HTTP 500），Flask 日志显示 `NameError: name 'session_obj' is not defined`。

**根因**：在 `routes.py` 中重写或修改路由处理器时，引入了一个在代码上下文之外定义的变量引用。例如：

```python
@bp.route("/api/practice/submit", methods=["POST"])
def api_practice_submit():
    ...
    result = pe.submit_answer(session_id, user_answer, response_seconds)

    # ❌ session_obj 从未定义！但编译器不报错（语法上有效，运行时才抛 NameError）
    if "error" not in result and result.get("done") is False and session_obj.get("mode") == "retry":
        ...
```

**为什么编译器不报错**：Python 是动态语言，`session_obj` 在语法检查时只是一个合法的标识符引用，只有在执行到这一行时才会发现它不在任何作用域中。

**诊断方法**：

1. **查看 Flask 日志**（不是 curl 返回的 JSON）：
   ```bash
   tail -50 ~/edu-hub/edu-hub.log   # 搜索 "Traceback" 或 "NameError"
   ```
   Flask 500 错误会在服务端完整打印调用栈，包含行号和变量名。

2. **用 test_client 模拟**（不需要浏览器）：
   ```python
   with app.test_client() as c:
       with c.session_transaction() as sess:
           sess['v2_role'] = 'student'
       r = c.post('/api/practice/submit', json={...})
       if r.status_code == 500:
           print('500 错误!')  # 然后检查服务端日志
   ```

3. **代码审查清单**——哪个变量可能未定义？检查：
   - 变量名拼写是否与当前作用域中的变量一致
   - 该变量是否在当前函数的代码路径中赋值（可能被 if/else 分支跳过）
   - 该变量是否来自 `_SESSIONS.get()` 等调用——那次调用的返回值是否赋值给了变量

**修复**：删除未定义变量的引用行，或将逻辑移到变量已定义的范围内。

**预防**：在 C005 Step 4（质量测试）中，对所有路由处理器增加一次 **Python 作用域扫描**——逐行检查每个函数内引用的所有局部变量名是否都已赋值。特别关注那些看起来像从别处复制粘贴过来的代码块。

## 新增陷阱 20：patch replace_all 导致 Python 代码严重损坏（区别于模板损坏）

**发现场景**（2026-05-06 C008 阶段1实现）：在 `data_manager.py` 上使用 `patch(..., replace_all=True)` 来替换 `data[student_id][word_en] = p` 附近的代码块，期望只修改 `update_memory_curve()` 函数。结果同时损坏了 `ensure_word_memory()` 和 `_get_learning_states_from_sqlite()` 两个不相关的函数。

**根因**：Python 文件中常见的变量名（`p`, `item`, `data`）在不同函数中出现多次，但**语义不同**：
- `p` 在 `ensure_word_memory()` 中是 `params` 字典（`p["interval"]` 等不存在）
- `p` 在 `update_memory_curve()` 中是 SM-2 参数对象（有 `interval`, `ease_factor` 等）

`replace_all=True` 会匹配所有出现位置，无论语义是否相同。

**典型症状**（Python 文件）：
- 不相关函数被注入了 SQL 语句块（SQLite 写入代码出现在错误的位置）
- 变量名被替换为另一个函数的变量（`default` → `p`）
- `return` 语句被替换为错误的返回对象（`return default` → `return p`）
- 函数被凭空插入或删除
- 代码语法检查失败

**Python 文件 vs HTML 模板的损坏区别**：

| 维度 | HTML 模板（原陷阱20） | Python 代码（本陷阱） |
|:-----|:---------------------|:---------------------|
| 损坏表现 | 重复定义、标签截断 | 变量名语义错误、SQL块植入 |
| 检查方法 | 视觉检查、grep 结构标签 | **语法检查**（`compile/ast.parse`） |
| 修复难度 | 低（视觉定位） | 中高（需理解函数边界） |
| 恢复方法 | 单处 patch | 需重写整个函数或文件 |

**修复方法**：

1. **立即检查文件完整性**：用 `ast.parse()` 检查语法
   ```bash
   python3 -c "import ast; ast.parse(open('damaged.py').read()); print('Syntax OK')"
   ```
   如果语法检查失败 → 已严重损坏，不要用 patch 修 patch

2. **恢复策略**：
   - 如果被损坏的函数可以定位 → 用足够长的上下文 patch 修复每个函数
   - 如果损坏跨多个函数（变量名错乱+代码植入）→ **用 `write_file` 重写整个文件**
   - 如果有原始文件备份或 git → `git checkout -- file.py`

3. **重写文件时的注意事项**：
   - 如果文件较大（>500 行），分段重写前先 `read_file` 全量读取
   - 重写时保留原始的 imports 和模块级变量（`DATA_DIR`, `_LOCKS` 等）
   - **先写正确版本，再应用正确的 patch**（一条一条，用唯一上下文）

**预防**：

1. **`replace_all=True` 在 Python 文件上禁止使用**，除非你已验证文件中确实只有 1 处匹配
2. **优先用足够长的上下文**使 old_string 唯一（至少包含 3 行前后文 + 函数签名）
3. **每次 patch 后立即做语法检查**：
   ```bash
   python3 -c "import ast; ast.parse(open('file.py').read()); print('OK')"
   ```
4. **如果同一个文件的 patch 次数 ≥ 3 次** — 切换为 `read_file` + `write_file` 一次性重写
5. **在 patch 前先 grep 确认 old_string 的唯一性**：
   ```bash
   grep -c "old_string_pattern" file.py
   # 返回值 > 1 → 不能用 replace_all，必须加上下文
   ```

## 新增陷阱 21：CSS 文件多次 patch 导致重复定义/结构损坏

**发现场景**：在同一个 CSS 文件上连续执行多次 `patch` 操作后，文件出现了重复的 `body{}`、`.nav-brand{}` 等选择器，部分选择器内容被截断混入其他选择器中。

**根因**：CSS 文件的选择器结构相对松散（嵌套深度浅、分号结尾），`patch` 工具的上下文匹配算法可能在多次操作后产生边界模糊。当多次 patch 的区间有重叠或接近重叠时，后续 patch 的匹配可能落在错误的位置。

**修复方法**：当 CSS 文件被 3 次或更多次 patch 后出现奇怪行为时，使用 `write_file` 重写整个文件，而不是用 patch 修复 patch。

**预防**：
1. 对 CSS 文件的修改尽量一条 patch 完成所有变更
2. 如果需要在同一个文件中做多次不连续的修改，考虑分批而不是分次 patch（先用 `read_file` 读全量，然后一次性 `write_file`）
3. 每次 patch 后运行语法检查：`python3 -c "import cssutils; cssutils.parseFile('file.css')"`（如果 cssutils 可用）或至少检查文件是否被截断

## 新增陷阱 26：模板变量名不匹配（静默空渲染）

**发现场景**：知识点 API 返回 56 条数据，但学生端门户页面显示「还没有语法知识点数据」的空白状态。API 正常、路由正常、模板正常渲染，就是数据为 0。

**根因**：路由处理器传递模板变量时用了错误的变量名，模板中的循环迭代器找不到数据：

```python
# 路由
return render_template("portal.html", 
    points=points_with_mastery)  # 模板期望 knowledge_points

# 模板
{% if knowledge_points %}  ← 这是 False！
    {% for kp in knowledge_points %}  ← 不会执行
{% else %}
    <p>还没有数据</p>  ← 显示这个
{% endif %}
```

**为什么 Flask 不报错**：Jinja2 对未使用的模板变量和未定义的模板变量都**静默忽略**。`points` 被传入但没有在模板中使用 → Jinja2 不报错。`knowledge_points` 未传入 → Jinja2 认为它是 `None`/`False`。两边都无声无息。

**排查模式**（API 返回正确数据但页面空白时的诊断路径）：
```
页面空白 → curl API 检查 → API 正常返回 N 条数据
                                       ↓
             检查路由的 render_template 参数名 → 与模板的变量名比对
                                       ↓
                 发现 route 传 points，模板读 knowledge_points
```

**修复**：在 `render_template()` 中添加别名参数，或用模板期望的变量名：
```python
return render_template("portal.html",
    points=points_with_mastery,                    # 保留原有（其他代码可能依赖）
    knowledge_points=points_with_mastery,           # 添加模板对应的变量名
)
```

**预防**：
1. 在 C005 Step 4 质量测试中，对每个页面路由增加**模板变量名验证** — 检查 `render_template()` 的关键字参数名与模板中引用的变量名是否匹配
2. 用自动化脚本扫描模板文件中的 `{% for x in ... %}` 和 `{{ x }}` 引用，与路由中的 `render_template()` 参数交叉比对
3. 在代码审查中增加"模板变量名一致性"检查点

---

## 新增陷阱 28：交付后用户路径验证缺失（说"完成"前先走一遍用户的路）

**发现场景**：完成了跨模块的语法模块改造（SM-2、题库、错题本、LLM生成、星星币等），API测试全部通过、页面200 OK、数据正确，汇报"已完成"。用户反馈：**"家长端依然没有看到上传试卷的功能"**。

**根因**：验证了 API 端点、单元测试、HTML 渲染，但**没有模拟用户的完整操作路径**。实际问题是：
1. 家长端导航菜单的「试卷分析」链接是 `href="#"`（死链）
2. 路由传参名与模板变量名不匹配（`stats` vs `analyzed_exams`）
3. 两个模板文件（`exams_list.html`、`analyze_result.html`）根本不存在

**API 测试全部 200 OK 掩盖了这些问题** — 因为路由存在 → 返回 200，但页面内容是空的（模板条件判断为 False）或链接点不进去。

**交付前验证清单**（Step 5 末尾，汇报"已完成"之前必做）：

```python
# 验证路径 1: 用户导航路径 — 点击每个链接，确认能到达目标页面
def verify_user_paths(app, session_role):
    with app.test_client() as c:
        with c.session_transaction() as sess:
            sess["v2_role"] = session_role
        
        # 获取页面
        resp = c.get("/parent/grammar/")
        html = resp.data.decode()
        
        # 提取所有导航链接
        import re
        links = re.findall(r'href="([^"]+)"', html)
        for link in links:
            if link.startswith("#") or link.startswith("http"):
                continue
            # 验证每个链接可访问
            r = c.get(link)
            if r.status_code != 200:
                print(f"⚠️ 死链: {link} -> {r.status_code}")
            
            # 验证页面有实际内容（不是空状态因变量名不匹配）
            body = r.data.decode()
            if len(body) < 100:
                print(f"⚠️ 页面过短: {link} ({len(body)} chars)")
```

```bash
# 验证路径 2: 导航链接不是死链（简单版）
curl -s -o /dev/null -w "%{http_code}" http://localhost:5002/parent/grammar/analyze
curl -s -o /dev/null -w "%{http_code}" http://localhost:5002/parent/grammar/exams
curl -s -o /dev/null -w "%{http_code}" http://localhost:5002/parent/grammar/llm-questions
```

```bash
# 验证路径 3: 模板文件存在性 — 路由 exists != 模板 exists
# grep 路由中 render_template 的模板路径
grep -r "render_template" routes.py | grep ".html"
# 与实际文件清单比对
ls -la templates/parent/
ls -la templates/student/
```

**验证路径 4**: 模板变量名一致性 — 比比看
```bash
# 从路由提取 render_template 参数名
grep -A1 "render_template" routes.py | grep -v "render_template" | grep -oP '\w+(?==\w)'

# 从模板提取引用的变量名
grep -oP '\{\{[^}]+\}\}' templates/*.html | grep -oP '\w+' | sort -u

# 交叉比对：模板用了但路由没传的变量
```

**预防**：
- 在 C005 Step 5「集成与交付」阶段末尾，增加**用户路径验证步骤**
- 验证清单包括：导航链可点击、模板文件存在、模板变量名与路由参数名一致
- 对涉及前端页面的任务，交付物中必须包含"用户导航路径验证"结果
- 关键原则：**API 全部 200 OK ≠ 用户功能可用**

**这个陷阱和陷阱 26（模板变量名不匹配）的区别**：陷阱 26 发现的是单个路由的模版变量名问题，陷阱 28 关注的是**交付后缺乏系统性的用户路径验证**——不检查导航链接、不检查模板文件存在性、不模拟用户完整操作流程。（重构引发的 NameError）

**发现场景**：重构测试文件时，将某个测试中的变量 `sid` 改为 `sid2`（因为与原数据冲突），后续测试用例未同步更新，导致 `NameError: name 'sid' is not defined`。

**根因**：顺序执行的测试脚本中，变量从定义到被引用存在隐式依赖链。当修改前面的变量名时，后面引用该变量的代码不会自动更新：

```python
# 改造前
sid = 'test_stu'
r = mm.record_answer(sid, 'kp', True)     # 定义 sid
due = mm.get_due_knowledge_points(sid)    # 引用 sid → 没问题
path = mm._mastery_path(sid)               # 清理也引用 sid

# 改造后（为了与其他测试隔离，改成了 sid2）
sid2 = 'test_stu_v2'
r = mm.record_answer(sid2, 'kp', True)    # 定义为 sid2
due = mm.get_due_knowledge_points(sid)    # ❌ 仍是旧变量名 sid！
path = mm._mastery_path(sid)               # ❌ 同上
```

**修复检查清单**（当测试间的变量名做任何重命名时）：
```
□ 变量定义后的所有引用都已更新
□ 变量在测试函数的末尾清理代码中也已更新
□ 测试间的共享 fixture/state 变量也检查了
□ 运行全部测试，而不仅仅是修改的那一个
```

**预防**：
1. 测试变量重命名后，用 `grep -n 'sid\\|old_varname' test_file.py` 检查所有引用
2. 在 C005 的测试编写指南中明确：每个独立测试用例使用唯一的变量名前缀/后缀，避免测试间的隐式依赖
3. 测试文件最后应包含**全部测试运行**，而非仅单条运行（可以暴露跨测试的变量污染）

---

## 新增陷阱：函数双返回路径只改其一（静默陷阱）

**发现场景**（C003 v7.0 实施 2026-05-12）：改造 `api_practice_today()` 时，在返回体中添加了 `questions` 字段。但函数有 **两条返回路径** —— `if bundle:` 路径和兜底生成路径。只在第二条路径加了字段，第一条没改。用户反馈页面空白。

```python
# 第一个返回路径（if bundle:）—— 缺了 questions 字段！
return jsonify({
    "calculations": calculations,
    "word_problems": word_problems,
    "date": bundle.get("date", ""),
    "title": bundle.get("title", ""),
    # ← 没有 questions！
})

# 第二个返回路径（else/兜底）—— 改了
return jsonify({
    "calculations": calculations,
    "word_problems": word_problems,
    "questions": q_list,  # ✅ 有
    "date": bundle.get("date", ""),
    "title": bundle.get("title", ""),
})
```

前端会 `if (!data.questions)` 判断条件为真，显示空状态。

**排查方法**：搜索函数的 `return jsonify` 或 `return dict` 的所有实例：
```bash
# 在 function X 中找到所有 return
grep -n "return jsonify\|return {" routes.py | head -20
# 对比每个返回体的字段集是否一致
```

**预防清单**：
1. 修改任何返回体时，先 `grep -n "return"` 找到该函数的**所有 return 语句**
2. 逐条确认每个 return 都包含了新字段
3. 最简单的预防：将公共字段提取为变量，然后 `return jsonify({**common_fields, ...specific_fields})`

**排查标志**：修改了 API 返回体后，前端读取 API 不报错（HTTP 200）但页面功能异常。

## 新增陷阱 33：路由函数参数顺序与业务函数不匹配（静默瘫痪陷阱）

**发现场景**（2026-05-13 C008 语法模块修复）：启动练习正常 → 提交答案返回 500。Flask 日志显示 `submit_answer()` 内部参数错乱。检查发现路由调用传入了 4 个位置参数，但函数只接受 3 个 —— 且因为超出了函数参数个数，Python 直接抛 `TypeError`。

```python
# ❌ 路由传了 4 个参数，函数只接受 3 个
@grammar_bp.route("/student/grammar/submit", methods=["POST"])
def submit_answer():
    ...
    result = pe.submit_answer(student_id, session_id, question_index, answer)
    #                                 ^^ 多了一个 student_id！
    # TypeError: submit_answer() takes 3 positional arguments but 4 were given

# ✅ 正确
    result = pe.submit_answer(session_id, question_index, answer)
```

**更隐蔽的变体**：函数同样接受 3 个参数，但顺序错了：
```python
def submit_answer(session_id, question_index, user_answer):
    ...

# ❌ 参数数量匹配但语义错位
result = pe.submit_answer(student_id, session_id, question_index)
# student_id → session_id (第一个参数位置)，完全语义颠倒

# ✅ 正确
result = pe.submit_answer(session_id, question_index, answer)
```

**根因**：路由函数内先 `_get_student_id()` 获取额外变量，顺手就把这个变量传进了业务函数。Python 按位置传参，一旦多传或少传了一个，API 整个瘫痪。

**排查**：查看 500 错误的 Flask 日志，对比路由调用参数列表和业务函数签名。注意：修改 `submit_answer()` / `complete_practice()` 等核心 API 时，务必确认路由调用行的参数列表与函数签名严格一致。

**预防**：参见 `references/flask-api-mismatch-patterns.md` 的「模式B」。

## 新增陷阱 34：HTML 表单直接 POST 到 JSON-only API（请求方式不匹配陷阱）

**发现场景**（2026-05-13 C008 语法模块修复）：练习页面 `submitBtn` 通过常规 HTML `<form>` 提交到后端 `submit_answer()` 路由。后端用 `request.get_json()` 解析请求体，但 form 提交的是 `application/x-www-form-urlencoded` —— 结果 `get_json()` 返回 `None`，所有参数丢失，API 返回「缺少必要参数」。

```python
# 后端
data = request.get_json(silent=True) or {}  # form 提交时 → {}
session_id = data.get("session_id")          # → None
```

```html
<!-- 前端 -->
<form method="POST" action="/submit">
    <input name="session_id" value="...">
    <button type="submit">提交</button>
</form>
```

当 form 没有 JavaScript 拦截时，浏览器自动用 `application/x-www-form-urlencoded` 格式提交，后端 `get_json()` 无法解析。

**现场诊断**：
1. 前端点提交 → 浏览器直接跳转到 `/submit` 页面显示 `{"error":"缺少必要参数"}`（因为后端返回 JSON，不是重定向）
2. 检查请求的 Content-Type → `application/x-www-form-urlencoded`（不是 `application/json`）
3. 后端日志无 `submit_answer` 异常（因为 `get_json(silent=True)` 静默返回 None，抛出 HTTP 400，不是 500）

**修复**：
- 前端：用 `fetch()` + `Content-Type: application/json` 发送请求
- 后端：同时接受 `request.is_json` + `request.form` 两种格式（向下兼容）

详见 `references/flask-api-mismatch-patterns.md` 的「模式A」。

## 新增陷阱 22：CSS 插值错误——`.replace()` 与 CSS 花括号冲突

**发现场景**：在 Flask 视图函数中使用 Python 字符串模板生成 HTML（如渲染门户页面），模板中包含 CSS 样式时使用 `.format()` 替代 `{EXTERNAL_URL}` 失败。

**根因**：CSS 规则使用花括号 `{}` 定义选择器，如 `body{font-size:16px}`。当 Python 字符串调用 `.format()` 方法时，这些花括号会被识别为格式化占位符，触发 `KeyError`。

```python
# ❌ 失败：CSS 花括号 {margin:0} 被当作 format 占位符
LANDING_PAGE = """<style>*{margin:0}</style><a href="{EXTERNAL_URL}">...</a>"""
return LANDING_PAGE.format(EXTERNAL_URL=url)

# ✅ 正确：用 .replace() 代替 .format()
LANDING_PAGE = """<style>*{margin:0}</style><a href="{EXTERNAL_URL}">...</a>"""
return LANDING_PAGE.replace("{EXTERNAL_URL}", EXTERNAL_URL)
```

**排查标志**：`KeyError: 'margin'` 或类似的花括号内容作为错误键。

**预防**：只要 HTML 字符串中包含 CSS（或任何使用花括号的语言），就必须使用 `.replace()` 而非 `.format()` 进行变量替换。

## 新增陷阱 23：数据字段与显示字段的语义混合（Flask 数据流反模式）

**发现场景（C008 2026-05-06）**：同一个 Bug 类重复出现 3 次。每次修复看似正确（功能测试通过），但用户在另一场景下遇到相同问题。最终发现所有表面不同的症状指向同一个根因——`_enrich_question()` 用 fallback 提示覆盖了 `cn` 字段。

**根因模式**：一个**JSON 数据字段**被赋予了双重语义——既是「业务数据」（如中文释义），又是「显示提示」（如 fallback 提示文字）。不同代码路径对同一个字段的读写使用的是不同语义，导致不一致：

```python
# ❌ 反模式：数据字段被覆盖为显示文本
def _enrich_question(q, word_info):
    # cn 本应是中文释义（业务数据）
    q["cn"] = dm.get_fallback_prompt(...)  
    # 但现在 cn 变成了 "请输入单词的正确拼写"（显示文本）
    # 原始中文释义丢失了！

# 然后在前端：
# qText.textContent = data.cn  
# → 用户看到显示文本，看不到业务数据
```

**排查标志**：
- 同一个 JSON key 在不同代码路径中被写入不同类型的数据（业务值 vs 显示文本）
- 后端 API 返回的字段在前端被直接渲染为可见内容，但字段名暗示它是业务数据
- 某个字段的值可能是"请输入..."之类的中文显示提示

**预防清单**：

1. **每个数据字段只承载一种语义**：
   - `cn` → 永远存中文释义（可空）
   - `prompt` → 存显示提示文本
   - `display_text` → 存前端显示的内容
   - 不要复用业务数据字段来存储显示提示

2. **`_enrich_question` 类函数**：只添加新字段，不覆盖已有字段。如果原有字段需要不同值，加新字段

3. **验证方法**：代码审查时追踪 JSON 响应中每个字段的**完整写路径**——所有写入该字段的代码位置、类型是否一致

4. **修复范围判定**：当同一个 JSON 字段在不同路径被写入不同类型数据时，至少检查所有读该字段的代码位置（前端模板、API 响应、路由处理），确保修复覆盖全部消费方

**参考**: `advisory-council-workflow` skill 的 `references/bug-fix-mode-d-worked-examples.md` 案例 3

## 新增陷阱 24：并行 worker 超时但文件已写完（虚惊陷阱）

**数据源容量陷阱（v3.9 新增）**：当设置练习引擎目标题量时，必须验证三类数据源（新词/已掌握/错题）的可用量之和能否达到目标。否则代码声明 100 题，实际只出 17 题。详见 `references/data-source-capacity-verification.md`。

**发现场景**：在 delegate_task 中使用 3 个并行 worker 实施大型项目（如 C011 v2.0 全量重写），1 个 worker 在 600 秒后返回 `status: "timeout"`，但检查文件系统发现所有文件已成功写入。

**根因**：Hermes Agent 的 subagent 有 600 秒超时限制。worker 可能已经完成了所有文件写入操作，但清理/返回阶段耗时过长导致超时，或最后一次工具调用恰好卡在超时边界。**超时不等于失败**——文件可能已完整写入，worker 只是没来得及返回完成状态。

**排查方法**：
```bash
# 1. 检查文件是否已创建
find ~/edu-hub/chinese-v2 -type f -name '*.py' -o -name '*.js' -o -name '*.html' | sort

# 2. 检查文件完整性（没有被截断）
wc -l path/to/suspected/file.py

# 3. 检查语法
python3 -c "compile(open('path/to/file.py').read(), 'file.py', 'exec')"
```

**修复**：如果文件已存在且完整，直接使用即可，无需重跑 worker。如果部分文件缺失（如模板文件写了一半），仅补写缺失文件而非全量重跑。

**预防**：
- 意识到 600 秒超时是平台限制，不可消除
- 优先使用 `write_file` 做原子写入（文件写完即持久化），而非 `patch`（需多次调用）
- 超时后先检查文件系统，再决定是否重跑

## 新增陷阱 31：类型/交互模式实现覆盖检查缺失（静默空渲染陷阱）

**发现场景**（2026-05-12 C003 数学模块）：练习页面没有输入答案的地方。API 返回 `word_problem` 类型题目，但前端只处理了 `choice`/`fill`/`calc` 三种类型。

**根因**：题目生成引擎输出多种类型（`choice`, `fill`, `calc`, `word_problem` 等），但前端做题页面只实现了部分类型的输入框/交互组件。未处理的类型被**静默跳过**——不报错、不显示错误信息、用户看到空白。

```javascript
// ❌ 错误：存在未处理的类型分支
if (q.type === 'choice') { renderChoice(q); }
else if (q.type === 'fill') { renderFill(q); }
else if (q.type === 'calc') { renderCalc(q); }
// word_problem 被静默忽略，不渲染任何内容

// ✅ 正确：所有类型必须有处理路径，或显式报错
switch (q.type) {
    case 'choice': renderChoice(q); break;
    case 'fill': renderFill(q); break;
    case 'calc': renderCalc(q); break;
    case 'word_problem': renderTextInput(q); break;  // 新增处理
    default: console.error('未知题目类型:', q.type);  // 兜底捕获
}
```

**排查标志**：
- 用户说"页面上没有输入框/没有答案的地方"
- API 返回的内容类型比前端处理的多
- 前端代码中有 `if/else if` 链但缺少 `else`/`default` 兜底

**预防清单**（每次涉及类型系统/交互模式枚举时执行）：

**步骤 1：在 Step 0（需求分析）中，枚举所有类型/交互模式**

```markdown
类型清单（2026-05-12 C003 数学模块实战案例）：
┌─────────────┬──────────┬──────────┐
│ 类型         │ 后端输出 │ 前端处理 │
├─────────────┼──────────┼──────────┤
│ choice      │ ✅       │ ✅       │
│ fill        │ ✅       │ ✅       │
│ calc        │ ✅       │ ✅       │
│ word_problem│ ✅       │ ❌ 缺失  │  ← 本次发现
└─────────────┴──────────┴──────────┘
```

**步骤 2：在 Step 2（架构设计）中，输出「类型实现覆盖矩阵」**

作为架构交付物的一部分，创建一个 **类型 vs 实现** 矩阵，逐行验证：

```json
{
  "type_coverage_matrix": {
    "title": "类型实现覆盖矩阵",
    "types": [
      {"name": "choice",     "engine_outputs": true, "frontend_handles": true,  "edge_to_frontend": "renderChoice()"},
      {"name": "fill",       "engine_outputs": true, "frontend_handles": true,  "edge_to_frontend": "renderFill()"},
      {"name": "calc",       "engine_outputs": true, "frontend_handles": true,  "edge_to_frontend": "renderCalc()"},
      {"name": "word_problem", "engine_outputs": true, "frontend_handles": false, "action": "需要增加 text input 渲染路径"}
    ],
    "gaps": [
      {"type": "word_problem", "missing_handling": "前端无 text input 渲染", "fix": "增加 text input 框 + 确认按钮"}
    ]
  }
}
```

**步骤 3：在 Step 3（实现）中，按矩阵逐行实现每一行**

每一个缺失的 `frontend_handles: false` 都必须成为实现任务的一项。

**步骤 4：在 Step 4（测试）中，生成每个类型的测试题目**

```python
# 测试每个类型都有可用的渲染路径
for q_type in ['choice', 'fill', 'calc', 'word_problem']:
    resp = c.get(f'/api/practice/next?type={q_type}')
    html = resp.data.decode()
    if q_type == 'word_problem':
        assert 'input' in html, f"word_problem 类型缺少输入框"
```

**相关的 Pre-implementation Audit**：

在每个涉及前端交互的任务开始编码前，运行以下预审计：

```bash
# 审计 A：枚举后端输出的所有类型
grep -oP "'type':\\s*'[^']+'" engine/*.py | sort -u | sed "s/'//g"

# 审计 B：枚举前端处理的所有类型
grep -oP "q\\.type\\s*===\\s*'[^']+'" templates/*.html | sort -u | sed "s/q\\.type\\s*===\\s*'//g; s/'$//"

# 审计 C：对比差异
# 后端有但前端没有的 = 缺失处理
```

## 新增陷阱 32：英文分类名直接拼接中文句子（Flask 模板国际化反模式）

**发现场景（C008 2026-05-06）**：用户报告"请输入word的正确拼写"，以为是单词 `word` 的问题，实际是 `data.category` 字段值为 `"word"`（英文），被直接拼接进中文句子 `'请输入' + data.category + '的正确拼写'`，产生了"请输入word的正确拼写"。

**根因**：Flask 模板中直接使用英文值拼接中文句子，没有做翻译映射。

```javascript
// ❌ 错误：data.category = "word" → "请输入word的正确拼写"
promptLabel.textContent = '请输入' + (data.category || '单词') + '的正确拼写';

// ✅ 正确：增加英→中映射
var catNames = {'word':'单词', 'phrase':'词组', 'sentence':'短句'};
promptLabel.textContent = '请输入' + (catNames[data.category] || data.category || '单词') + '的正确拼写';
```

**更隐蔽的版本（这个 Bug 发现的前提）**：这个 Bug 在此之前**一直存在但从未被触发**，因为 `_enrich_question()` 始终用 fallback 提示覆写 `data.cn`，导致前端模板跳过 `else` 分支。当 `_enrich_question()` 被修复（保留原始 cn）后，`data.cn` 为空 → 触发了 else 分支 → 暴露了这个潜伏的 Bug。

**排查清单**：

1. 搜索模板中所有 `+ data.category` 或 `+ category` 的拼接：
   ```bash
   grep -rn "['\"]请输入['\"].* + " templates/*.html | grep -v "catNames\|cat_"
   ```

2. 检查模板中所有英文值直接出现在中文句子中的情况（不限于 category）：
   ```bash
   grep -rn "['\"]请输入['\"]" templates/*.html | grep -v "单词\|词组\|短句\|catNames"
   ```

3. 关注 `|| '单词'` 这类 fallback 模式——如果 fallback 是中文但路径值却是英文，就是一个隐藏的国际化 Bug

4. 检查所有 `else` 分支和边界路径。一个从未被测试到的 `else` 分支可能包含隐藏 Bug。对于修复 `_enrich_question()` 这类改变"哪个分支被触发"的修改，必须验证完整渲染路径。

**预防**：在 C005 Step 4 质量测试中，对模板中的字符串拼接设置为**变量名必须包含翻译映射表**的模式检查。

**参考**：`advisory-council-workflow` skill 的 `references/bug-fix-mode-d-worked-examples.md` 案例 3

## 新增陷阱 29：模板占位符替换导致完整句子（语法题变翻译题）

**发现场景**：语法练习的规则名词复数知识点中，模板文本 `"I have two {noun_pl}."` 中的 `{noun_pl}` 被替换为具体名词（如 "dogs"），导致题目显示为完整句子"I have two dogs."，同时给出选项 "book" / "dogs" / "the book"。学生不知道这是选择题（选出正确复数形式），以为是翻译题。

**根因**：`_render_question()` 中的 `_fill_template()` 会**替换模板文本中的所有占位符**。当模板文本本身包含 `{noun_pl}` 时，替换后就变成完整句子，不再有空白占位。而答案和选项同样进行了替换，导致正确选项"dogs"已经显示在句子中。

```python
# ❌ 错误模板设计
{
    "type": "choice",
    "template": "I have two {noun_pl}.",      # 被替换为 "I have two dogs."
    "answer": "{noun_pl}",                     # 被替换为 "dogs"
    "choices": ["{noun_sg}", "{noun_pl}", "the {noun_sg}"],  # 被替换为 ["book","dogs","the book"]
}

# ✅ 正确模板设计
{
    "type": "choice",
    "template": "I have two ___.",             # 文字占位符，不参与替换
    "answer": "{noun_pl}",                     # 正确选项
    "choices": ["{noun_sg}", "{noun_pl}", "the {noun_sg}"],  # 替换后保留空白
}
```

**关键规则**：
- **模板文本（template）中不应该包含占位符** — 使用文字 `___` 作为视觉空白
- **答案（answer）和选项（choices）中使用占位符** — 这些需要动态替换为随机值
- `{noun_pl}`、`{noun_sg}` 等占位符只应出现在 answer 和 choices 中，不应出现在 template 中

**修复检查清单**：
```bash
# 检查所有模板中是否在 template 字段中错误使用了占位符
grep -E '"template":.*\{noun_|"template":.*\{verb_|"template":.*\{name\}' practice_engine.py
# 输出示例: "template": "I have two {noun_pl}."  ← 错误！应改为 "I have two ___."
```

**排查标志**：选择题的完整句子中包含正确选项本身。改错题则不同——改错题故意呈现错误形式，这是正确的。

**相关文件**：语法练习引擎 `~/edu-hub/english/grammar/practice_engine.py` 中的 `_QUESTION_TEMPLATES`

**发现场景**：C007 顾问团评审说「语法模块已建好，功能完整」，C005 实施完成后用户说「我依然没有看到上传文件的功能」。实际原因是：

1. 语法管理页面的所有导航链接都是 `href="#"`（死链），用户点不到「试卷分析」
2. 路由传 `stats` 字典但模板用独立变量名（如 `{{ analyzed_exams }}`），统计卡片始终显示 0
3. 路由引用了两个不存在的模板（`exams_list.html`、`analyze_result.html`）
4. 英语家长主页的「试卷导入」标签页只有粘贴文本，没有文件上传入口

**根因**：v1 快速原型阶段常用 `href="#"` 做 UI 占位，上线后未补全。新增功能时假设「父页面已有入口」，但入口从未真正可用。

**预防** — 在 Step 0 需求分析完成后、Step 3 编码之前，执行**三个预审计**：

### 审计 A：导航链接完整性
```bash
# 提取所有 href 值，筛选 #
grep -oE 'href="[^"]*"' templates/parent/*.html | sort -u | grep '#'
```

### 审计 B：模板变量一致性
```python
# route 传参名 ← → 模板变量名 对比
# 特别注意 stats 字典 vs 独立变量的模式差异
```

### 审计 C：模板文件存在性
```bash
grep -oP "render_template\('[^']+\.html" routes.py | sed "s/render_template('//" | \
  while read f; do [ -f "templates/$f" ] || echo "❌ 缺失: templates/$f"; done
```

参见 `references/edu-hub-extension-audit.md` 完整清单。

质量门禁预存问题处理：参见 `references/quality-gate-pre-existing-issues.md`。

**补救措施**（如果已经上线后发现）：对于 `href="#"` 的死链，将每个 `#` 替换为真实路由。对于缺失模板，参考 `references/edu-hub-extension-audit.md` 的模板模式创建。

**高风险信号**：任何 `href="#"` 在非品牌标题元素上——这是一个 v1 占位残留，必须修复。`#` 在导航链接中**永远**不应该出现在生产代码中。

**发现场景**：在本会话中，C007 顾问团已经产出了完整的架构方案文档（含数据流图、目录结构、API 设计、技术选型）。随后启动 C005 实施时，严格按照流程做了 Step 1（代码学习）和 Step 2（架构设计），但这两个步骤的内容与顾问团产出高度重复。

**根因**：C005 的标准流程假设没有前置方案文档，需要从零学习代码、设计架构。但当 C007 已产出完整方案时，学习+设计步骤变成重复劳动。

**加速路径**：
```
正常流程：C007→C005 Step 0→Step 1(学习)→Step 2(设计)→Step 2.5(UI)→Step 3(实现)→...
加速路径：C007→C005 Step 0→直接Step 2.5(骨架)→Step 2.6(UI/UE设计)→Step 3(实现)→...
                    ↑
              跳过 Step 1 和 Step 2
              因为架构已在 C007 交付物中定义
```

**触发条件**（必须全部满足才可加速）：
- C007 交付物包含：数据模型定义、API端点设计、目录结构、技术选型
- 用户已确认上述方案（通过 C007 流程）
- 代码学习无特殊风险（无未知的技术债务、无遗留系统接口）

**执行方式**：
- Step 0（需求分析）正常执行，记录 task_id
- Step 0 交付物中 `next_step` 设置为 `"ui_ue_design"` 而非 `"learn"`
- Step 2.5 根据 C007 交付物搭建骨架
- Step 2.6 直接引用 C007 交付物路径
- 在 Step 3 context 中注明："架构设计见 C007 顾问团报告 [链接]"

### Template-first confirmation mode (用户偏好)

当任务涉及**新的交互页面、模板、UI/UE设计**时，用户要求在编码实现前先展示设计/模板。这是本场景的固定流程：

```markdown
正确的改造节奏（用户偏好）:
Step 1: 设计模板/策略文档 → 用户确认
  （展示YAML模板设计、HTML页面布局、交互流程）
Step 2: 架构对齐检查（对照同类模块）→ 用户确认
  （输出对比表，标明对齐/不对齐项）
Step 3: 编码实现 → 测试 → 交付
```

跳过 Step 1 直接编码会导致返工。用户原话：**"你先设计模板，在开始改造执行前先和我说一下"**。

适用场景：教育系统模块改造、新增交互页面、新增题型模板。对其他有明显 UI/UE 设计的任务同样适用。

### 与跟 C007 无关的独立加速：绿地项目（Greenfield）：
纯CSS/视觉改造（不改逻辑、不改API、不改数据结构）时，
即使规模匹配 Tier 2/3，也有权跳过 Step 1（代码学习）和 Step 2（架构设计）：

```
正常流程：Step 0 → Step 1(学习) → Step 2(设计) → Step 2.5(骨架) → Step 2.6(UI) → Step 3(实现) → ...
加速路径：Step 0 → Step 2.5(骨架) → Step 2.6(UI/UE设计) → Step 3(实现) → Step 4(测试) → Step 5(交付)
                    ↑
              跳过 Step 1 和 Step 2
              （因为无逻辑代码需要学习，无架构需要设计）```
```

**判定条件**（全部满足才可加速）：
1. 不涉及修改任何现有 .py/.html/.js 文件（全部新建）
2. 不涉及与现有系统的深度集成（纯外部 API 调用 + 文件输出）
3. 需求规格明确（Step 0 可在一轮内完成确认）
4. 复杂度确定性高（无非确定的技术风险需要前期验证）

> **与 C007 加速路径的区别**：C007 加速是被动跳过（有证据告诉我不需要学习），绿地加速是主动跳过（客观上没有东西可学）。

**不适用场景**：
- C007 方案文档不够详细（仅有功能描述无技术设计）
- 涉及复杂的已有代码修改（必须学习现有代码结构）
- 需要与遗留系统深度集成

---

## 新增陷阱 23：路由处理器中引用未定义变量导致 500（NameError 无声陷阱）

**发现场景**：练习/听默提交答案后，页面不跳到下一题，始终显示第一题。

**根因**：代码使用 `_SESSIONS = {}` 作为 L1 内存缓存，每次操作后通过 `save_session()` 写入文件（L2），但**未同步更新 `_SESSIONS` dict**。下一次 `api_current()` 读取 `_SESSIONS` 时拿到的是旧数据。

```python
# 错误写法
session["current_index"] += 1
save_session(session_id, session)  # L2 已更新
# _SESSIONS[session_id] = session  ← 缺少这一行！

# 正确写法
session["current_index"] += 1
save_session(session_id, session)
_SESSIONS[session_id] = session  # 同步 L1 缓存
```

**排查标志**：前端提交后进度不动，但后端日志显示 `current_index` 在文件中有更新。重启服务后恢复正常（因为 L1 缓存重建自 L2）。

**预防**：每次 `save_session()` 后立即更新 `_SESSIONS`。参见 `references/flask-in-memory-cache.md`。
- `references/post-patch-safety.md` — 工具安全规范（patch/write_file 安全规则、后置验证脚本）
- `references/interactive-prototype-methodology.md` — 交互式 HTML 原型方法论
- `references/embedded-data-quality-audit.md` — 嵌入式 JS 数据质量审计方法论
- `references/handwriting-recognition-architecture.md` — 手写识别架构参考（TrOCR + qwen3-vl 双层混合方案）
- `references/edu-hub-new-module-checklist.md` — 在 edu-hub 单体 Flask 应用中新增子系统模块的检查清单（Blueprint注册、导航入口注入、常见陷阱）
- `references/edu-hub-extension-audit.md` — 在 edu-hub 已有模块上扩展功能前的预审计清单（导航链接完整性、模板变量一致性、模板文件存在性、父级页面入口验证）

## 新增陷阱 4：一次性修复遗漏深层根因（用户声称“未修复”的回归）

**发现场景**：用户报告练习/听默提交后页面不跳转下一题。第一次修复改动了前端 JS 的 `loadNextQuestion()` 调用，但**未触及根因** — 后端 `_SESSIONS` 内存缓存未同步。用户测试后反馈“还是没修复”。

**常见模式**：修复看似简单的 Bug 时，容易停留在**表层修复**（页面显示行为），而忽略深层根因（数据一致性问题）。

**根因分析框架**：

```
Bug 报告：“提交后页面不动”
  ↓
表层分析：
  ├─ 点击提交 → 调用 submit_answer() → API 返回 200
  ├─ 然后调用 loadCurrentQuestion() → 重新获取当前题目
  └─ 结果：还是第一题
  ↓
表层修复思路（❌ 错误）：
  └─ 修改前端 loadCurrentQuestion() 里的 index 计算
  ↓
深层分析（✅ 正确）：
  ├─ API 返回的 data.index 是多少？→ 始终是 1
  ├─ 服务端 current_index 是否递增了？→ 在文件里递增了
  ├─ 那 API 为什么还返回旧值？→ 因为读了内存缓存
  └─ 内存缓存为什么没更新？→ save_session() 后没同步 _SESSIONS
```

**排查清单**（遇到“修复了但问题依旧”时逐项检查）：

1. **数据流追踪**：从用户交互到数据持久化的完整路径 trace
   - 前端点击 → fetch API → 后端路由 → 业务函数 → 数据修改 → 持久化 → 下次读取路径
2. **双存储一致性**：检查是否有 `_CACHE = {}` + 文件/DB 双存储
   - 每次写操作后是否同步了内存缓存？
   - 每次读操作是从哪一端读的？
3. **重启测试**：重启服务后问题是否消失？
   - 如果重启后正常 → 内存缓存问题（L1 desync）
   - 如果重启后依旧 → 文件/DB 持久化问题（L2 not written）
4. **API 响应验证**：不要只测前端显示，直接 curl 测试 API 返回值
   - `curl api/batches/M1U1` → 确认数据已变更
   - `curl api/current?session=X` → 确认当前索引
5. **用户回归测试计时**：修复交付后，让用户验证时**记录执行日志**（服务端和前端），而非仅“看着正常”。如果用户说“还是不行”，立刻要求提供重现步骤和 API 响应截图。

**预防** — 在 Step 3 实现阶段，对所有涉及“读→改→写→再读”的代码路径，增加一致性验证：改后立即读取验证。在 Step 4 测试中增加 API 级回归测试（模拟完整操作序列）：

```python
# Step 4 测试用例示例：模拟用户完整操作链
def test_submit_advances_index():
    # 1. 创建会话
    session = start_practice(...)
    initial_idx = get_current_index(session)

    # 2. 提交答案
    submit_answer(session, "test_answer")

    # 3. 确认索引已递增
    new_idx = get_current_index(session)
    assert new_idx == initial_idx + 1, f"提交后索引应递增，期望 {initial_idx+1}，实际 {new_idx}"

    # 4. 确认前端 get_current API 返回正确索引
    current = api_practice_current(session)
    assert current['index'] == new_idx + 1  # API 返回 1-based
```

## 新增陷阱 5：__pycache__ 导致旧代码残留 + Flask 模板缓存（两个独立机制）

**发现场景**：修改了 Flask 应用代码文件后，重启服务仍运行旧版本逻辑，`print` 调试输出未出现。

**根因**：Python 有两个独立的缓存机制可能导致旧代码残留：

### 机制 A: Python `.pyc` 缓存（已在 v3.0 记录）

Python 启动时缓存已编译的 `.pyc` 文件到 `__pycache__/` 目录。当只修改代码而不清理缓存时——尤其通过 `kill` 旧进程再启动，而非进程内重载——**.pyc 文件可能残留旧版本**。在 Flask 开发场景中，如果旧进程被杀时正在写入缓存，新进程启动时可能读取到破损的旧缓存。

**强制清理命令**：
```bash
find ~/edu-hub -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null
# 然后重启服务
```

### 机制 B: Jinja2 模板字节码缓存（新增）

当 `app.run(debug=False)` 时，Flask 启用 Jinja2 模板编译缓存。`.html` 模板被编译为 Python 字节码并缓存到独立位置。与 `.py` 缓存不同，**进程重启不一定清除模板缓存**。即使文件内容已更新，Flask 仍读取缓存中旧版本的编译结果。

**排查命令**：
```bash
# 确认文件已更新
grep -n 'NewFunction' path/to/template.html
# 确认服务端实际上在输什么
curl -s http://localhost:PORT/path | grep -o 'NewFunction\|OldFunction'
# 模板缓存位置
find /tmp -name '*jinja*' -type f 2>/dev/null | head -10
```

**解决**：
1. 确认杀对了进程（`ss -tlnp | grep PORT` 找实际占用 PID）
2. 开发时可临时添加 `app.jinja_env.auto_reload = True`
3. 参见 `references/jinja2-template-caching.md` 详情

**预防**：两种缓存都要考量。修改 `.html` 模板时怀疑缓存 → 用 `curl | grep` 验证，不要只看浏览器。

## 新增陷阱 6：修复验证必须覆盖完整数据路径（"一次修复，全局验证"原则）

**发现场景**：用户反复报告同一个练习提交 Bug，虽然在代码中修复了后端 `_SESSIONS` 同步，但只验证了 API 响应正确，未验证前端是否收到更新数据。用户测试后说"还是没修复"。

**完整数据路径**：
```
用户操作 → 前端 JS → fetch API → 后端路由 → 业务函数 → 持久化(L2)
                                                                     ↓
用户期待结果 ← 前端渲染 ← HTTP响应 ← 路由返回 ← 下一请求 ← 读取缓存/持久化
```

**验证检查清单（修复后必做）**：

```python
# 验证路径 1: 持久化层 — 数据是否写入了文件/DB？
verify_persistence: 直接 cat/读数据库确认数据已变更

# 验证路径 2: API 层 — 后端返回了什么？
curl /api/current-session/X   # 确认 current_index 已递增
curl /api/sessions/X          # 确认 session 状态已更新

# 验证路径 3: 内存缓存层 — L1 缓存是否也更新了？
# 重启服务后验证（重启重建 L1），看问题是否消失
# 如果重启后正常 → L1 缓存未同步

# 验证路径 4: 前端层 — 通过 curl API 模拟前端请求序列
# 模拟完整用户操作链：
step1: POST /api/submit  {"session_id":"X","answer":"test"}
step2: GET  /api/current?session=X
  # verify: step2 返回的 index > step1 前的 index
  # verify: step2 返回的 question 不是 step1 的重复
```

**全路径验证伪代码（Step 4 测试模板）**：
```python
class TestFullDataPath:
    def setup_method(self):
        self.session_id = create_test_session(batch="M1U1")
        self.initial_current = get_current(self.session_id)
    
    def test_submit_updates_persistence(self):
        """验证持久化层已更新"""
        submit_answer(self.session_id, "test")
        stored = read_session_file(self.session_id)
        assert stored["current_index"] == 1
    
    def test_submit_updates_api_response(self):
        """验证 API 返回的数据是最新的"""
        submit_answer(self.session_id, "test")
        current = api_get_current(self.session_id)
        assert current["index"] == 1, f"API 返回旧索引: {current['index']}"
    
    def test_submit_advances_frontend_state(self):
        """验证前端后续操作使用更新后的数据"""
        # 模拟用户完整的做题流程
        for i in range(3):
            q = get_current_question(self.session_id)
            submit_answer(self.session_id, "placeholder_ans")
            verify_index_advanced(self.session_id, i+1)
```

**常见遗漏模式（实战总结）**：
```
1. 修了后端代码，验证了 API → 没验证 persistence → 重启后数据丢了
2. 修了持久化层，验证了文件 → 没验证 L1 缓存 → 用户还读到旧数据  
3. 修了 L1 缓存，验证了 API → 没验证前端 JS 处理 → 前端缓存了旧结果
4. 修了本次请求路径 → 没验证后续请求（submit→current→submit→current 循环） → 推进一次后卡住
5. 修了单个用户场景 → 没验证多用户/并发场景 → 数据交叉污染
```

**根因分析框架（"修复了但用户说没修好"时使用）**：

```python
def diagnose_unfixed_bug(user_steps: str) -> dict:
    """诊断用户报告"根本没修复"的 Bug"""
    suspects = {
        "代码没生效": lambda: is_new_process_running(),
        "修了表层没修根因": lambda: has_root_cause_beneath(fix),
        "验证了单层没验全链": lambda: verify_full_data_path(fix),
        "数据源有残留": lambda: check_old_data_pollution(),
    }
    return {k: v() for k, v in suspects.items() if v()}
```

**预防** — 在 C005 Step 4（质量测试）中，所有涉及"读→改→写→再读"的代码路径，必须包含完整的全路径验证测试用例，覆盖持久化、API、内存缓存三层路径。Tier 1 至少验证 API+持久化，Tier 2+ 覆盖全路径。

**发现场景**：修改了 Flask 应用代码文件后，重启服务仍运行旧版本逻辑，`print` 调试输出未出现。

**根因**：Python 启动时缓存已编译的 `.pyc` 文件到 `__pycache__/` 目录。当只修改代码而不清理缓存时——尤其通过 `kill` 旧进程再启动，而非进程内重载——**.pyc 文件可能残留旧版本**。在 Flask 开发场景中，如果旧进程被杀时正在写入缓存，新进程启动时可能读取到破损的旧缓存。

**强制清理命令**：
```bash
find ~/edu-hub -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null
# 然后重启服务
```

**预防**：在服务重启脚本中自动清理缓存。

## 九、大规模并行升级技术（Tier 3 实战模式）

> 当 Tier 3 任务涉及 **4+ 个独立工作流**（如本会话的 C008 v4 升级：核心模型 + 练习引擎 + 家长端 + 学生端），使用以下模式批量编排。

### 9.1 工作流拆分原则

将变更按**文件依赖关系**拆分为独立工作流，确保工作流之间**无共享文件写入冲突**：

```python
# 工作流拆分示例（C008 v4 实战）
workstreams = [
    {
        "name": "A: 核心数据模型",
        "files": ["mastery_model.py", "knowledge_base.py", "checkin_manager.py", "state_manager.py"]
    },
    {
        "name": "B: 练习引擎",
        "files": ["unified_practice.py(NEW)", "data_manager.py", "coins_manager.py"]
    },
    {
        "name": "C: 家长端",
        "files": ["routes.py", "report_engine.py", "import_page.html(NEW)", "material_library.html(NEW)"]
    },
    {
        "name": "D: 学生端",
        "files": ["routes.py", "portal_v4.html(NEW)", "practice_session_v4.html(NEW)", "practice_result_v4.html(NEW)"]
    }
]
```

**冲突检测**：如果两个工作流修改同一文件 → 合并为一个工作流，或拆分为串行依赖。

### 9.2 分批调度模式

max_concurrent_children 限制为 3，4+ 工作流需分批：

```python
# 第一批：3 个并行（优先级：无外部依赖的工作流优先）
first_batch = [workstream_A, workstream_B, workstream_C]  # 无依赖冲突

# 第二批：剩余 1 个（可以与第一批并行，只要文件不冲突）
second_batch = [workstream_D]
```

### 9.3 Context 构建要点

每个子代理的 context 必须包含：

1. **工作流专有需求** — 仅该工作流的变更内容
2. **完整文件路径** — 所有 `~/edu-hub/...` 绝对路径，确保子代理能找到文件
3. **现存代码参考** — 说明从哪里读参考（如 `参考 ~/edu-hub/english/v3/practice_engine.py 当前逻辑`）
4. **质量要求** — 明确要求类型提示、docstring、兼容性
5. **禁止操作** — `不要使用任何第三方库，只使用 Python 标准库`

### 9.4 集成验证

所有工作流并行完成后，执行全量集成验证：

```python
# 1. 语法检查 — 对所有修改/新建文件执行 compile()
for f in all_files:
    compile(open(f).read(), f, 'exec')

# 2. 导入检查 — 在项目根目录下尝试 import 所有修改的模块
# 3. 路由注册验证 — 检查 app.url_map 确认所有新路由已注册
# 4. 运行时验证 — 启动服务，curl 测试关键端点
# 5. 质量门禁 — 运行 run_quality_gate.py
```

### 9.5 服务器重启陷阱

在启动集成测试前，**必须先检查并杀死旧进程**，否则新代码不生效：

```bash
# 检查端口占用
ss -tlnp | grep <PORT>

# 杀死旧进程
kill -9 <PID>

# 在新进程中启动
cd ~/edu-hub && python3 app.py
```

**根因**：运行中的 Flask 进程将已导入模块缓存在内存中。修改源代码后，即使新进程启动，如果旧进程还占用端口，新进程会因 `Address already in use` 而启动失败，而旧进程仍在提供旧版本响应。

**全量重启命令**：
```bash
pkill -f 'python3 app.py' 2>/dev/null
sleep 1
find ~/edu-hub -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null
python3 app.py &
sleep 3
curl -s http://localhost:<PORT>/health | grep -q '"status":"ok"'
```

---

## 十、骨架先行方法论（v4.0 新增 — 习惯驱动，非检查驱动）

> **核心理念**：大部分 Bug 的根因不在编码层面，而在设计层面——目录结构、命名、接口签名、模板变量名。骨架正确了，填空时就出不了大错。
>
> **转变**：Step 2 → Step 2.5(骨架搭建) → Step 3(逻辑填充) 变成**默认路径**。
>
> **关键区分**：不是事后的「检查点」，而是事前的「习惯」。

### 10.1 事后检查 vs 事前习惯

```
检查点思维：
  写完代码 → 审查 → 发现 Bug → 修复
  代价已经产生，修改成本高

习惯思维：
  开始写之前 → 骨架已经正确 → 填空时不会跑偏
  零代价（根本不会产生 Bug）
```

骨架层面发现的错误是 0 成本修改（还没写逻辑），代码实现阶段发现骨架问题意味着要重构已写完的代码。

### 10.2 Step 2.5：骨架搭建规范

**每个 Tier 1+ 的任务，Step 2(架构设计) 之后必须经过 Step 2.5(骨架搭建) 才能进入 Step 3(编码实现)。**

**模板来源**：使用 skill 内置骨架模板快速生成，见 `templates/` 目录：
- `templates/blueprint-skeleton.py` — Flask Blueprint + Route 骨架
- `templates/page-skeleton.html` — HTML 页面骨架（含 loading/empty/error 状态）
- `templates/api-skeleton.js` — JS API 调用骨架（完整 try/catch）

**产出物：可编译但无逻辑的代码框架**

```
artifacts/<task_id>/skeleton_v1/
  ├── python_skeleton/     # Python 骨架（import + class + def签名 + route装饰器）
  ├── template_skeleton/   # HTML 骨架（extends + block + 变量引用名）
  ├── static_skeleton/     # JS/CSS 骨架（函数签名 + API 调用路径）
  └── dependency_graph.md  # 方法调用/页面引用关系图
```

#### Python 骨架规范

```python
# ✅ 正确：所有结构就位，方法体留空
from flask import Blueprint, render_template, request, jsonify
from .models import Student, PracticeSession

class PracticeEngine:
    """练习引擎"""
    def generate_questions(self, student_id: str, count: int) -> list:
        raise NotImplementedError

    def evaluate_answer(self, question_id: str, answer: str) -> dict:
        raise NotImplementedError

bp = Blueprint("practice", __name__, url_prefix="/api/practice")

@bp.route("/generate", methods=["POST"])
def api_generate():
    data = request.get_json()
    # TODO: call PracticeEngine.generate_questions()
    return jsonify({"questions": []})  # return 结构定型

@bp.route("/submit", methods=["POST"])
def api_submit():
    data = request.get_json()
    # TODO: call PracticeEngine.evaluate_answer()
    return jsonify({"result": {}, "next_question": {}})
```

**骨架验证**：
```
□ 所有 import 语句已就位（可执行 import）
□ 所有 class/def 已定义（含类型注解）
□ 所有 route 装饰器 + url_for 已就位
□ 所有 return 结构已定型（多路径结构一致）
□ compile() 语法检查通过
□ 无 hardcoded 路径
□ 无 href="#" 占位
□ 目录名不与 Python 内置模块冲突
```

#### HTML 骨架规范

```html
{# ✅ 正确：extends + block + 变量引用名 #}
{% extends "base.html" %}
{% block title %}练习页面{% endblock %}
{% block content %}
<div class="practice-container">
    <h1>{{ page_title }}</h1>
    {% for question in questions %}
    <div class="question-card">
        <p>{{ question.text }}</p>
        <input type="text" name="answer" class="answer-input">
    </div>
    {% endfor %}
    <nav>
        <a href="{{ url_for('practice.index') }}">返回</a>
    </nav>
</div>
{% endblock %}
```

**骨架验证**：
```
□ 所有 extends + block 结构已就位
□ 模板变量名与 route render_template 参数名一致
□ 所有 url_for 已使用（无硬编码路径）
□ 所有导航链接确认可达（非死链）
□ 模板文件物理存在（grep route → 确认文件存在）
```

#### 方法调用/页面引用关系图

```
┌─ api_generate() ──────────────────────────┐
│  ├── PracticeEngine.generate_questions()   │
│  └── jsonify()                             │
├─ api_submit() ─────────────────────────────┤
│  ├── PracticeEngine.evaluate_answer()      │
│  └── AnswerChecker.check()                 │
└─ portal() → render_template("portal.html") ┘
```

### 10.3 各 Tier 的骨架要求

| Tier | Step 2.5 要求 |
|:----|:-------------|
| Tier 0 | ❌ 跳过（<20 行无需骨架） |
| Tier 1 | ✅ 轻量骨架：单文件的 import + def 签名 |
| Tier 2 | ✅ 完整骨架：多文件的目录 + import + class + route + 模板 |
| Tier 3 | ✅ 完整骨架 + 关系图 + 类型覆盖矩阵 |

### 10.4 骨架与 Step 3 编码的关系

```
骨架搭建：
  .py 文件 ← import + class + def签名 + route装饰器 + return结构 → 填空更不会走偏
  .html 文件 ← extends + block + 变量引用 → 填空时变量名正确
  .js 文件 ← 函数签名 + API 调用路径 → 填空时数据流正确
```

Python 的 `compile()`、Jinja2 的模板语法、浏览器的 JS 解析器会自动约束编码层面的错误。真正危险的错误在设计层面——方法调对了但语法正确的、模板变量名不匹配但 Jinja2 静默忽略的、页面链接了但路径不对的。骨架阶段就是解决这些问题的。

### 10.5 工具默认行为（习惯化为内置约束）

| 工具 | 默认行为 |
|:----|:---------|
| patch() | **.py 文件拒绝 replace_all=True**（除非文件中确实只有 1 处匹配） |
| patch() | **.md 文件含多副本相似内容（如 Tier 1/2/3 定义）也禁止 replace_all=True** — 会交叉命中 JSON 结构 |
| patch() | 替换后**自动 compile() 语法检查** |
| patch() | 同一文件 ≥ 3 次 → **提示切换 write_file** |
| write_file() | 目标文件已存在 → **提示「覆盖/新建版本」** |
| 骨架创建 | 自动生成 import + 类型注解 + url_for |
| 骨架创建 | 自动检查**目录名与 Python 内置模块冲突** |

---

## 十一、开发原则（12条，习惯化）

每条原则不是贴在墙上的标语，而是**骨架/流程/工具的默认行为**。

### 11.1 架构原则（嵌入 Step 2 + 2.5）

```
原则1：迁移必幂等
  案例：B7 状态迁移重跑即炸
  习惯化：migrate() 默认含 IF NOT EXISTS / CREATE IF NOT EXISTS
          Step 2 交付物必须包含幂等性验证方案

原则2：改旧必兼容
  案例：B3 硬编码路径失效
  习惯化：骨架强制 url_for()，无 hardcoded 路径
          旧路由保留重定向（301），不直接删除

原则3：命名不撞车
  案例：B6 连字符文件名、B2 模板目录同名
  习惯化：骨架创建文件时自动检查：
          - 是否与 Python 内置模块同名
          - 是否与其他 Blueprint 模板目录同名

原则4：数据字段单一语义
  案例：陷阱23 cn 被覆盖为显示文本
  习惯化：骨架阶段定义每个 JSON 字段的单一职责
          不允许同一字段在不同路径写入不同类型
```

### 11.2 编码原则（嵌入 Step 2.5 + 工具默认行为）

```
原则5：骨架优先，填空在后
  案例：陷阱26 模板变量名不匹配、陷阱31 类型覆盖缺失
  习惯化：Step 2.5 是必经步骤
          route 骨架 + 模板骨架同步产出
          类型覆盖矩阵为骨架交付物必有字段

原则6：写完必验证
  案例：B1 patch() 静默失败
  习惯化：工具默认执行 compile() 语法检查
          replace_all=True 在 .py 上被直接拒绝

原则7：先范围后实现
  案例：陷阱29 双返回路径只改其一
  习惯化：修改前 grep 所有 return jsonify 路径
          骨架阶段列出所有返回路径并保证结构一致
```

### 11.3 数据原则（嵌入 Step 3 + 4）

```
原则8：永不覆盖（版本链）
  案例：B11 飞书文档覆盖原文件
  习惯化：工具默认阻止对已存在 doc_token 的写入
          必须显式指定新建版本（v2/v3）

原则9：全链追溯
  案例：陷阱6 修了 API 没验持久化
  习惯化：测试骨架默认三层验证（持久化→API→前端）
```

### 11.4 流程原则（嵌入 Step 0 + 5）

```
原则10：环境清单先行
  案例：B8 WSL 不自启、B9 Ollama GPU 不兼容
  习惯化：Step 0 完成后自动检查目标平台特殊限制

原则11：配置集中管理
  案例：B10 OLLAMA_HOST 散落在 4 个脚本中
  习惯化：新建脚本时默认从 config.yaml / .env 读取配置

原则12：类型覆盖矩阵
  案例：陷阱31 word_problem 类型前端没处理
  习惯化：骨架交付物必须包含「后端类型↔前端处理」映射表
```

---

## 十二、关键约定（v4.0 新增 + v3.0 继承）

1. **交付物优先** — 所有步骤先定义输入/输出 Schema，再执行（避免自由文本描述）
2. **路径引用** — 步骤间仅传递交付物文件路径，不传递全量内容
3. **校验前置** — 每个步骤执行前验证上游交付物的 checksum 和 status
4. **失败即停** — 交付物缺失或损坏时立即报错，不允许基于不完整数据推测
5. **血缘必录** — 每个交付物必须记录 parent_step_id，构成完整血缘图谱
6. **原子写入** — 交付物通过 .tmp→rename 方式写入，防止写入中断导致损坏
7. **未经用户确认不进入下一阶段** — Step 0 未确认前，不做架构/编码（继承 v2）
8. **子代理隔离执行** — 各角色在独立上下文中运行，互不污染（继承 v2）
9. **问题反馈闭环** — 测试发现的问题必须回溯到架构+工程修复（继承 v2）
10. **只新增不破坏** — 集成时遵守"只复制不删除"原则（继承 v2）
11. **学习先行** — 修改已有代码必须先学习，输出"不可触碰区域"（继承 v2）
12. **子代理不可用 clarify** — 所有信息在 context 中给足（继承 v2）

---

## 十三、约束学习闭环（v4.2 新增）

> 每次测试或用户反馈暴露的设计层面问题，都应评估是否补充为流程约束。
> 但约束存在「维护成本 + 执行成本」—— 不加控会导致流程臃肿、效率下降。

### 13.1 约束分级：按成本分为 4 档

```text
                        执行成本（token/时间）
                            低 ←────────→ 高
  预防价值     ┌─────────────────────────────────┐
    高        │ 🔧 内置       │ 🛡️ 审计门禁    │
              │ (0 token)     │ (~300 tokens)   │
              │ 工具级强制     │ 独立脚本/步骤   │
              ├────────────────┼─────────────────┤
    低        │ 📋 清单项     │ 📚 参考文档     │
              │ (~50 tokens)  │ (按需加载)      │
              │ Step内检查项   │ 仅出问题时查阅  │
              └────────────────┴─────────────────┘
```

| 等级 | 标识 | 执行成本 | 维护成本 | 何时添加 | 示例 |
|:----|:----|:--------|:--------|:---------|:-----|
| **🔧 内置** | 工具级 | 0 token | 高（改工具代码） | 频繁触发 + 自动化可行 | `patch()` 拒绝 replace_all |
| **🛡️ 门禁** | Step/脚本 | ~300 tokens | 中（维护脚本） | 偶发但影响大 | Step 2.4 合规审计 |
| **📋 清单** | Step 内检查项 | ~50 tokens | 低（加一行） | 规则简单 + 不易忘 | 骨架阶段检查 url_for |
| **📚 参考** | 引用文档 | 按需 | 低（写文档） | 场景特化 + 不需要每次都检 | JSON字段语义冲突排查 |

### 13.2 约束添加决策树

每次测试暴露或用户反馈一个设计层面的问题后：

```text
问题：测试发现的某个设计缺陷
  ↓
询问：这个缺陷能否被一条明确的规则预防？
  ├─ 否 → 不用加约束（技术/业务能力问题，非流程问题）
  └─ 是 → 继续
          ↓
         这条规则能否由工具自动执行？
         ├─ 是 → 🔧 内置（改工具代码）
         └─ 否 → 继续
                 ↓
                这条规则被违反的频率多高？
                ├─ 每次任务都有此风险 → 🛡️ 门禁（独立步骤/脚本）
                └─ 偶尔遇到 → 继续
                            ↓
                           这条规则是否容易被忘记？
                           ├─ 是 → 📋 清单（Step内检查项）
                           └─ 否 → 📚 参考（放文档即可）
```

### 13.3 约束注册表

所有约束记录在 `references/constraint-registry.md`，格式：

```yaml
- id: C001
  name: 禁止 replace_all 在 .py
  weight: 🔧 内置
  source: B1 patch静默失败
  added: 2026-05-14
  trigger: patch() on .py → 自动检查
  hits: 0  # 触发次数，用于评估是否降级/升级

- id: C008
  name: 模板变量名交叉比对
  weight: 📋 清单
  source: 陷阱26
  added: 2026-05-14
  stage: Step 2.5 骨架验证
  hits: 3
```

约束注册表中的 `hits`（命中次数）用于定期评估：
- **hits=0** 连续 3 个月 → 降级（📋→📚 或 删除）
- **hits>5/月** → 升级（📋→🛡️ 或 🔧）
- **所有约束** 首次添加后 1 个月自动评估，之后每季度评估

### 13.4 添加时机

| 触发场景 | 添加时机 |
|:---------|:---------|
| **Step 4 测试**发现的设计缺陷 | 当前 C005 任务交付物中记录，C005 任务结束后立即更新 Skill |
| **用户测试反馈**的设计问题 | 收到反馈 → 评估 → 更新 Skill（当前会话内完成） |
| **C007 审核**中发现的设计漏洞 | C007 交付物中记录，转入 C005 时自动带约束 |

### 13.5 约束总量控制

| 指标 | 上限 | 超限处理 |
|:----|:----|:---------|
| 🔧 内置 | 无限制（0 成本） | — |
| 🛡️ 门禁 | ≤ 5 条 | 超限时合并或降级某条 |
| 📋 清单 | ≤ 20 条 | 超限时移除非高频项至 📚 |
| 📚 参考 | 无限制（按需加载） | — |

**每月 1 日自动瘦身**：检查器读取 `constraint-registry.md`，对 `hits=0` 的约束标记「待降级」，对超限类别提示「请合并」。
13. **每轮 delegate_task 最多 3 个并行任务**（继承 v2）
14. **涉及交互页面必须先完成 UI/UE 设计**（继承 v2）
15. **子代理自验证不能替代独立测试** — 子代理返回后我必须编写并运行覆盖全部 4 个象限的独立测试套件（UT/IT-API/IT-HTML/BT），子代理的"启动检查"或"简单运行验证"不视为正式测试。Tier 1 至少覆盖 A+B，Tier 2/3 全部覆盖。测试文件独立新建，不被子代理验证代码替代。
