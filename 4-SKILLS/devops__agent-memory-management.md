---
name: agent-memory-management
description: >-
  管理 Hermes Agent 持久化记忆的策略——防止记忆膨胀、定期瘦身、区分核心事实与历史日志。
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags:
      - memory
      - token-optimization
      - context-management
      - system-governance
    category: devops
---

# Agent 记忆管理策略

## 三层存储架构（2026-05-11 重构）

```
┌──────────────────────────────────────────────────────┐
│  Tier 1: MEMORY（每轮注入）    目标 ≤3K/15条          │
│  仅放「每轮都必须知道」的跨会话事实                     │
│  文件: ~/.hermes/memories/MEMORY.md + USER.md         │
├──────────────────────────────────────────────────────┤
│  Tier 2: SKILL（按需加载 + triggers）                 │
│  放「特定任务才需要」的流程/约定/陷阱                   │
│  文件: ~/.hermes/skills/<category>/<name>/SKILL.md    │
├──────────────────────────────────────────────────────┤
│  Tier 3: WIKI / 知识库（语义检索获取）                  │
│  放「需要时才查」的参考信息/历史记录                     │
│  文件: ~/wiki/ + ~/.hermes_tools/knowledge_base/      │
└──────────────────────────────────────────────────────┘
```

**迁移规则**：
- 固定的操作规程（Flask陷阱/C005流程/版本管理）→ Skill
- 参考数据（API参数/历史版本）→ Wiki/知识库
- 仅跨会话必备的事实（用户偏好/环境/架构决策）→ MEMORY
- 版本日志/Bug细节 → session_search 回溯


记忆系统（MEMORY.md）在每次会话的每轮消息中都注入到上下文中。如果记忆不断累积，会导致：
- 每轮消息的固定开销持续增长（从 3K → 19K+）
- 过时的操作记录（「文件夹已创建」「v4.3 已升级」）占用宝贵的上下文窗口
- 新的事实被淹没在历史日志中

## 核心原则：双层记忆结构

```
记忆总大小
├── 核心层 (~3K): 每次会话都必须知道的事实
│   包括: 架构规则、用户偏好、系统当前状态、API 要点
│   特征: 删除后会直接影响当前会话的响应质量
│
└── 历史层 (~10K+): 已完成的变更记录
     → 应该删除，通过 session_search 回溯
     特征: 删除后不会影响任何会话的响应（因为已是过去式）
```

## 什么该保存（核心事实）

| 类别 | 示例 | 必须保存 |
|:----|:-----|:--------|
| 用户偏好 | 全本地模型策略、只复制不删除原则 | ✅ |
| 架构规则 | 飞书教育目录的三层隔离设计 | ✅ |
| 模块当前状态 | C003 v6.0 / C006 active | ✅ |
| API 要点 | feishu-md-writer block_type 映射 | ✅ |
| 工作流要求 | 所有编码任务必须走 C005 | ✅ |

## 什么不该保存（历史日志）

| 类别 | 示例 | 应该删除 |
|:----|:-----|:--------|
| 一次性操作记录 | 「文件夹已创建」「token 已保存」 | ✅ |
| 已完成的版本历史 | v4.0 → v4.1 → v4.2 → v4.3 的详细变更日志 | ✅ |
| 已解决的问题报告 | 「依赖包缺失已修复」「权限已打通」 | ✅ |
| 已废弃的旧配置 | 旧的 token、旧的目录结构 | ✅ |
| 重复的信息 | 同一系统多个版本的独立条目 | ✅ |

## 定期瘦身流程

### 触发条件

- 记忆文件 > 5K 字符
- 或包含过时的版本历史（超过 2 个旧版本号）
- 或用户说「记忆太臃肿了」
- 或你发现每轮对话上下文开销 >10K（检查 `memories/MEMORY.md` + `USER.md` 的 `wc -c`）

### 三档分类法（L0/L1/L2）

瘦身时，将每条内容归入三档：

| 档位 | 范围 | 目标大小 | 特征 |
|:-----|:------|:--------:|:------|
| **L0** | USER.md (用户画像) | ≤ **2.5K** chars | 身份、角色、团队、核心偏好、通信/编码规则、生活作息。**每轮都需知道** |
| **L1** | MEMORY.md (技术笔记) | ≤ **5K** chars | 环境配置、架构决策、工具API要点、cron体系、系统间关系。**工作时大概率相关** |
| **L2** | Skills / wiki | 无上限 | 技术教训、踩坑记录、流程细节、参考数据。**按需加载** |

**L0和L1的总和应 ≤8K chars**，这是每轮对话的固定上下文开销。

### 执行步骤

**方案A（推荐 — 直接文件编辑）：** memory action 的 replace/remove 匹配不可靠（常因换行符/编码问题失败）。直接编辑文件是最彻底的方式：

```bash
# 1. 读取当前内容
cat ~/.hermes/memories/MEMORY.md | wc -c
cat ~/.hermes/memories/USER.md | wc -c

# 2. 用 write_file 直接覆盖（§ 为条目分隔符）
# USER.md 目标 ≤2.5K chars — 浓缩为 8-12 条，每条 150-300 字
# MEMORY.md 目标 ≤5K chars — 按模块合并（环境/Infra/CORE/Feishu API/Token追踪等）

# 3. 验证
wc -c ~/.hermes/memories/*.md
```

**方案B（备用 — 通过 memory tool）：**
```python
# 逐条 add + remove。注意：remove 需要精确子串匹配，对含换行的多行条目可能失败
memory(action="add", target="memory", content="浓缩后的条目")
# 然后尝试 remove 旧条目
memory(action="remove", target="memory", old_text="旧条目的唯一子串")
```

### 条目合并策略

合并同类项时遵循「从多到一」原则：

| 合并前（多条独立条目） | 合并后（一条） |
|:----------------------|:--------------|
| C003 v4.0 / v4.1 / v4.2 / v4.3 / v4.4 / v5.0 | `C003: 架构改为session模式。每日10题分布：...` |
| 飞书群A chat_id / 飞书群B chat_id / ... | `8部门飞书群: 📊(oc_...)/📋(oc_...)/✍️(oc_...)/...` |
| Bug修复记录 3条 | 只留修复结论（`feishu-task-manager: due字段类型bug→isinstance修复`），详细排查走 session_search |

### L2 → Skills 迁移规则

当从记忆修剪出L2内容时：
1. 先确认是否已有覆盖该领域的 skill（`skills_list` 检查）
2. 有 → 作为 pitfall/reference 追加到该 skill
3. 无 → 追加到现有的伞状 skill 或新建 class-level skill

迁移标记格式：`[从记忆移除 → 已迁移至 <skill-name>/references/<file>.md]`

### 典型瘦身收益

| 优化前 | 优化后 | 节省 |
|:-----:|:-----:|:----:|
| 19K / 40 条 | 2.8K / 11 条 | **-85%** |
| 51K / 135 条（USER+MEMORY合并） | 6.7K / 22 条 | **-87%** |
| 51K / 135 条（USER+MEMORY合并） | 8.3K / 20 条 | **-84%** | 最新实测（2026-05-15），见 `references/2026-05-15-memory-slimming.md` |
| 每轮注入 51K | 每轮注入 6.7K | **-44K/轮** |
| 日均50会话 × 30天 × 44K | — | **-66M tokens/月 ≈ $9-10/月** |

### 瘦身频率

- 每月1日自动检查（与约束注册表瘦身联动）
- 或发现单条内容 > 500 char
- 或用户反馈「响应变慢」/「上下文臃肿」

## 预防性措施

防止记忆重新膨胀：

1. **每次添加新记忆时，先检查是否已有同类条目**
   - 有 → 用 replace 更新而非 add 追加
   - 无 → 用 add 新增

2. **版本升级时删除旧版本条目**
   - v4.5 发布时 → 检查并删除 v4.0-v4.4

3. **每月至少检查一次记忆健康度**
   - 与 C006 运营监控联动

## 4. Advanced: External Memory Provider Integration

When the built-in MEMORY.md + USER.md system isn't enough (agent needs structured long-term recall, cross-session persona, or automated memory extraction), integrate an external memory provider.

### 架构模式

```
Hermes Agent (Python)
 └─ MemoryManager (always runs built-in + at most 1 external)
      └─ External MemoryProvider (plugin)
           ├─ HTTP client / SDK calls
           │
           ▼ HTTP
      Provider Gateway sidecar (Node.js/Go/Rust)
           └─ Storage backend (SQLite + vector / proprietary DB)
```

**关键特性：**
- MemoryManager 始终保留 built-in 提供者（MEMORY.md / USER.md），**外部提供者是附加的**
- 一次只能激活一个外部提供者（`memory.provider` 决定）
- 两个提供者独立运行，一个失败不影响另一个
- 外部提供者自动注入 `system_prompt_block` + `prefetch` 召回 + `sync_turn` 写入

### 激活外部提供者

```yaml
# config.yaml
memory:
  provider: <provider_name>      # 匹配 plugins/memory/<name>/ 目录名
  memory_enabled: true
  user_profile_enabled: true
```

### 生命周期

| Hermes 事件 | 回调 | 用途 |
|-------------|------|------|
| 每轮开头 | `prefetch(query)` | 召回相关记忆注入上下文 |
| 每轮结束 | `sync_turn(user, asst)` | 异步写入对话记录 |
| 会话结束 | `on_session_end(messages)` | 提取/聚合 |
| 卸载 | `shutdown()` | 清理连接 |

### 已知的外部提供者

| 提供者 | 插件位置 | 存储后端 | 特点 |
|--------|---------|---------|------|
| mem0 | `plugins/memory/mem0/` | Mem0 Platform API | 云端服务 |
| memory_tencentdb | `plugins/memory/memory_tencentdb/` | SQLite + sqlite-vec / TCVDB | 本地优先，四层记忆，可降本30-61% |
| honcho | `plugins/memory/honcho/` | Honcho API | 云端服务 |
| 其他（hindsight/holographic/byterover等） | 内置 | 各平台API | 可探索 |

详见 `references/memory-tencentdb-setup.md`（TencentDB 完整安装指南）。如需其他提供者的安装指南，请按同一格式补充参考文件。

## 已知陷阱

- `memory(action="add")` 总是追加新条目，不会覆盖旧的。需要先 `remove` 再 `add` 来实现更新
- `memory(action="replace")` 需要精确的 old_text 匹配，如果记忆条目被压缩过，old_text 可能找不到
- 直接编辑 `~/.hermes/memories/MEMORY.md` 是最彻底的清理方式，但需要确保格式正确（§ 分隔符）
- 用户偏好（target="user"）和记忆（target="memory"）是独立的存储，需要分别管理
