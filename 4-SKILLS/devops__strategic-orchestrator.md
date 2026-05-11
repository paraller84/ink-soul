---
name: strategic-orchestrator
description: 战略指挥官架构——任务分析→分解→复杂度预测→模型路由→结果聚合。我(DeepSeek R1)负责战略层，DeepSeek V4云模型负责执行层。
version: 2.0.0
tags:
  - orchestration
  - multi-agent
  - routing
  - cloud-models
  - deepseek-v4
  - architecture
---

# 战略指挥官架构 v2（Strategic Commander Orchestrator）

> **v2.0.0 迁移说明**：2026-05-09 基于 DeepSeek-V4 原生 1M 上下文能力，整体架构从「本地模型优先」迁移到「全云端模型优先」。
> 本地模型（qwen2.5:3b / qwen3:8b / qwen3.5:9b）已退役为主力，仅视觉（qwen3-vl:4b）和嵌入（qwen3-embedding）因无云端替代保留本地。
> 迁移理由：V4-Flash 缓存命中仅 0.02元/百万token，比本地推理更便宜更快。

## 核心哲学

```
我 = 唯一的 Agent（DeepSeek R1，云端）
 ├─ 职责：分析、分解、决策、聚合
 ├─ 优势：强推理、全局视野、战略判断
 ├─ 工作模式：在 3 种方法论间切换
 └─ 专注：所有任务的入口和出口

DeepSeek V4 云模型 = 执行层（原生 1M 上下文 @ api.deepseek.com/v1）
 ├─ deepseek-v4-flash（优先级1）→ 快速通用，缓存命中 0.02元/M
 ├─ deepseek-v4-pro（优先级2）→ 强推理（1.6T参数），缓存命中 0.2元/M
 ├─ deepseek-chat（优先级3）→ 前代通用备选（128K上下文）
 └─ deepseek-reasoner（优先级4）→ 前代推理备选（128K上下文）

本地模型 = 辅助资源（Ollama @ 192.168.3.102:11434）
 └─ qwen3-vl:4b → 视觉理解（DeepSeek 无视觉模型）
 └─ qwen3-embedding:0.6b → 向量嵌入（DeepSeek 无嵌入API）
```

### 架构中的两类实体：系统 vs 角色

**这是一个单 Agent 架构**——只有我一个 Agent（DeepSeek R1）。所有 C 编号能力分为两类，不是独立 Agent：

| 类型 | 定义 | 举例 | 特征 |
|:----|:-----|:-----|:------|
| 🟢 **系统（System）** | 持续运行的**完整闭环** | C001知识库 / C003数学辅导 / C008英语教育 / C010人生罗盘 / C011语文 / C012飞书待办 | 有专属代码、cron、数据持久化、独立功能域 |
| 👤 **角色/方法论（Work Mode）** | 我在特定任务中**扮演的专业身份** | C004战略指挥官 / C007顾问团 / C005编码工作流 / C015质量门禁 | 我一人分饰多角，按流程行动，不产生独立Agent |

**关键理解**：
- C003/C008/C011 是**系统**——它们是独立运行的 Web 应用（Flask），有自己数据库和 cron，我对话中调用它们
- C004/C007/C005 是**方法论**——是我做事的流程规范，不是"其他 Agent"
- C002/C006/C009 是**管道**——横切所有系统，只做同步/监控/归档
- DeepSeek V4 云模型是**计算资源**——通过 Hermes cloud_models 配置，按路由策略自动选择

## 复杂度分类标准（v2：云端模型路由）

| 级别 | 典型上下文 | 任务特征 | 路由目标 | 价格（缓存命中） |
|:---:|:----------:|:---------|---------|:---------------:|
| **L3** | 0-16K | 简单快速（问答/格式化） | deepseek-v4-flash（主）→ deepseek-v4-pro（备） | 0.02元/M |
| **L2** | 16K-64K | 中等复杂度（代码/摘要） | deepseek-v4-flash（主）→ deepseek-v4-pro（备） | 0.02元/M |
| **L1** | 64K-128K | 复杂推理（架构/分析） | deepseek-v4-pro（主）→ deepseek-v4-flash（备） | 0.2元/M |
| **L0** | 128K-1M | 超大上下文（整书/全量文档） | deepseek-v4-flash（主）→ deepseek-v4-pro（备） | 0.02元/M |
| **LV** | 图像 | 视觉理解/OCR | qwen3-vl:4b（本地）→ deepseek-v4-flash（备） | 本地 |

### 路由决策流程（v2）

```
用户输入
  ↓
[战略指挥官] 分析任务：
  1. 评估上下文大小（token量）
  2. 评估推理深度（简单 vs 复杂）
  3. 判断是否涉及视觉
  ↓
  ├─ L3/L2 → deepseek-v4-flash（最快最便宜）
  │   ├─ 快速问答、简单格式化、翻译、提取
  │   └─ 直接响应，无需委托
  │
  ├─ L1 → deepseek-v4-pro（强推理）
  │   ├─ 系统设计、多步规划、架构决策
  │   └─ 需要战略级思考的跟我自己处理
  │
  ├─ L0 → deepseek-v4-flash（原生1M，缓存极便宜）
  │   ├─ 整本技术文档分析、全库代码审查
  │   └─ 超过128K的任务自动路由到此层
  │
  ├─ LV → qwen3-vl:4b（仅视觉）
  │
  └─ 混合 → 分解后按子任务路由到不同层级
  ↓
[战略指挥官] 聚合结果 → 统一输出
```

### 缓存定价策略

V4-Flash 的缓存机制是**成本优化关键**：

| 场景 | 价格 | 推荐策略 |
|:----|:----:|:---------|
| 缓存命中 | **0.02元/百万token** | 尽可能复用稳定上下文（常用文档、系统 prompt） |
| 缓存未命中 | 1元/百万token | 首次处理或大幅变动的上下文 |
| 价格倍率 | **50x** | 鼓励缓存复用，避免重复截断 |

**实操建议**：
- 处理高频重复文档时，让 V4-Flash 走缓存路径
- V4-Pro 缓存命中 0.2元/M，适合需要强推理但可不频繁切换上下文的场景
- 简单任务坚持走 V4-Flash（L3/L2），即使未命中成本也最低

## 执行模式（v2）

### 模式A：直接路由（L3/L2 简单-中等任务）
```python
# Hermes 路由系统自动处理，无需手动指定模型
# 会话模型已设置为 deepseek-v4-flash @ 1M
# 路由根据上下文大小自动切换层级
```

### 模式B：战略处理（L1 复杂推理）
复杂推理任务由我（DeepSeek R1）直接处理，必要时使用 `delegate_task` 委派子代理：
```python
delegate_task(
    goal="分析代码库架构并生成优化方案",
    context="""使用 deepseek-v4-pro 进行深度分析...""",
    toolsets=["terminal", "file", "web"]
)
```

### 模式C：超大上下文（L0 > 128K）
路由系统自动将 >128K 的任务指向 deepseek-v4-flash（原生1M），deepseek-v4-pro 作为备选。

### 模式D：视觉任务
```python
# 调用 qwen3-vl:4b（本地，因 DeepSeek 无视觉模型）
# 通过 vision 辅助任务或直接调用 Ollama API
```

## Hermes 配置要点（v2）

### cloud_models 配置结构（~/.hermes/config.yaml）
```yaml
cloud_models:
  deepseek_v4_flash:
    provider: deepseek
    model: deepseek-v4-flash
    context_window: 1000000    # 原生1M
    max_tokens: 384000         # 最大输出384K
    timeout: 600
    fallback_priority: 1
  deepseek_v4_pro:
    provider: deepseek
    model: deepseek-v4-pro
    context_window: 1000000
    max_tokens: 384000
    timeout: 900               # 1.6T参数，推理时间更长
    fallback_priority: 2
  deepseek_chat:               # 前代备选
    context_window: 128000
    fallback_priority: 3
  deepseek_reasoner:            # 前代备选
    context_window: 128000
    fallback_priority: 4
```

### 路由层级配置
```yaml
router:
  routing_levels:
    level_0_ultra_context:    # 128K-1M
      primary: deepseek-v4-flash
      fallback: deepseek-v4-pro
    level_1_large_context:    # 64K-128K
      primary: deepseek-v4-pro
      fallback: deepseek-v4-flash
    level_2_medium_context:   # 16K-64K
      primary: deepseek-v4-flash
      fallback: deepseek-v4-pro
    level_3_small_context:    # 0-16K
      primary: deepseek-v4-flash
      fallback: deepseek-v4-pro
```

### 辅助任务迁移
```yaml
# 从本地 Ollama 迁移到 DeepSeek 云
auxiliary:
  web_extract:    ollama → deepseek-v4-flash
  compression:    ollama → deepseek-v4-flash
  approval:       ollama → deepseek-v4-flash
  vision:         qwen3-vl:4b 保留本地（无云端替代）
  embedding:      qwen3-embedding 保留本地（无云端替代）
```

## Pitfalls

### ❌ 沿用本地模型的思维定势
**旧行为**：优先考虑上下文大小能否放进本地模型（256K→128K→64K→16K 分级）
**新行为**：V4-Flash 和 V4-Pro 都原生支持 **1M 上下文**，路由分层主要是**成本优化**而非容量限制

### ❌ 把系统和角色混为一谈
当用户问"有哪些角色"或"有哪些Agent"时，切勿列出 C001/C003/C008 这类**系统**。

**正确做法**：
```
我（DeepSeek R1）是唯一的 Agent。我拥有 3 种工作模式：
1. 默认战略指挥官模式（C004）
2. 顾问团询问模式（C007）
3. 编码工作流模式（C005）

此外还有独立运行的 9 个系统（C001知识库 / C003数学 / C008英语 / ...），
但它们是持续运行的应用，不是我扮演的角色。
```

### ❌ 把 DeepSeek V4-Pro 和 deepseek-reasoner 搞混
- **V4-Pro** (deepseek-v4-pro)：V4 系列，原生 1M 上下文，温度 0.3，适合通用强推理
- **deepseek-reasoner** (R1)：前代模型，128K 上下文，温度 0.3，仅作极端备选（prio=4）

### ❌ 忽略缓存定价差异
V4-Flash 缓存命中 0.02元/M vs 未命中 1元/M（50x差）。处理高频重复文档时，优先让 V4-Flash 走缓存路径，而非切到 V4-Pro。

## 从 v1（本地模型）到 v2（云端模型）的变更记录

| 项目 | v1（本地优先） | v2（云端优先） |
|:----|:-------------|:--------------|
| 主力模型 | qwen3.5:9b (256K) / qwen3:8b | deepseek-v4-flash (1M) |
| 推理模型 | qwen3:14b | deepseek-v4-pro (1M) |
| 备选模型 | 无（本地全量） | deepseek-chat / deepseek-reasoner（低优先级） |
| 上下文上限 | 256K（本地模型瓶颈） | **1M**（云端天然支持） |
| 视觉 | qwen3-vl:4b（本地） | 仍用本地（无云端替代） |
| 嵌入 | qwen3-embedding（本地） | 仍用本地（无云端替代） |
| 路由策略 | 按容量分级（256K→128K→64K→16K） | 按成本分级（flash→pro→前代备选） |
| 成本 | 全部本地（电费） | 缓存命中时比本地更便宜 |
| 执行桥 | local-executor.py（本地 Ollama） | Hermes 原生路由（cloud_models） |

## 不在此技能范围内的

- **已有自动化系统**（C001 知识库、C003 数学辅导等）：这些有独立的 cron job 和管理方式，战略指挥官**不干预**它们的独立运行
- **Hermes Agent 配置细节**：cloud_models 和路由配置的完整语法见 `hermes-agent` 技能

## 验证命令

```bash
# 验证当前 Hermes 路由配置
hermes config 2>/dev/null | head -20

# 直接验证 YAML 完整性
python3 -c "
import yaml
cfg = yaml.safe_load(open('/home/openclaw/.hermes/config.yaml'))
rl = cfg['router']['routing_levels']
for k, v in rl.items():
    ctx = f'{v.get(\"min_context\",0)}-{v.get(\"max_context\",\"\")}' if 'min_context' in v else 'vision'
    print(f'{k:30s} | {ctx:15s} → {v[\"primary_model\"]:20s} (fallback: {v[\"fallback_model\"]})')
"
```
