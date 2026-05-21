---
name: feishu-drive-directory-standard
description: 飞书云盘完整目录标准 — 6大板块分类、文件命名规范（含生命周期状态标签）、生成路由规则、三区归档策略、知识库同步策略
version: 2.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [feishu, drive, directory, naming, routing, knowledge-base]
    use_cases:
      - 所有生成的HTML/文档/PPT/报告按规则归入飞书对应目录
      - 生成文件自动同步到本地知识库RAG系统
      - 各飞书群的产出自动归档到部门群产出目录
      - 项目开发文档归档到项目/对应子目录
---

# 飞书云盘目录标准 v1.0

## 一、顶层结构

飞书云盘下 **3个顶层文件夹**（不可变更）：

| 文件夹 | Token | 用途 |
|--------|-------|------|
| 📂 系统文档 | `LQx1fl0v8lB1XFdAI29czWeznVh` | 永久性文档、Wiki、知识库 |
| 📂 **Hermes生成文件** | `Otppfr9EelPIawdezL2csUXCnoh` | **所有AI生成的产出归档至此** |
| 📂 内容工厂出品 | `OaHdfQM9flT1vudSxmFcHVRMnEg` | C014内容工厂周度发布 |

> 🔴 **强制规则**：所有 Agent 生成的文档/文件/PPT/报告等，默认上传至 `Hermes生成文件` 下对应的子分类目录。不可直接上传到顶层或其他无关位置。

## 二、Hermes生成文件 — 6大板块

```
Hermes生成文件/
├── 📁 项目/                              # 1️⃣ 项目开发
│   ├── AI家庭教师/                       # AI家庭教师项目文档
│   │   ├── _archive/                    # 📦 历史版本归档（只读）
│   │   └── _drafts/                     # 🗑️ 草稿/过程文件（超7天清理）
│   ├── Pad端设计原型 2026-05-16/         # Pad端原型
│   ├── Token管理体系/                    # Token追踪体系
│   └── 数学模块原型/                     # 数学模块原型
│
├── 📁 工作汇报/                          # 2️⃣ 工作汇报
│   ├── EAST数据治理/                     # EAST相关汇报/排查/报告
│   ├── 竞聘材料/                         # 竞聘PPT/演讲稿
│   ├── EAST二季度工作会/                 # EAST二季度专项
│   ├── 中银保信合规风控/                 # 中银保信方案
│   └── 保险数据报送项目(苏总)/            # 苏总委托项目
│
├── 📁 部门群产出/                        # 3️⃣ 部门群产出（含顾问团）
│   ├── 📊战略材料部/                     # 各部门生成内容自动归档
│   ├── 📋会议情报部/
│   ├── ✍️内容市场部/
│   ├── 💡战略研究部/
│   ├── 🎓教育产品部/
│   ├── 📬对外联络部/
│   ├── 🗓️办公室/
│   ├── 🔧系统运维部/
│   └── 🧠顾问团/                        # 顾问团产出（合并自原两处）
│
├── 📁 日常运营/                          # 4️⃣ 日常运营
│   └── 待办看板/                         # 任务看板/项目仪表盘
│
├── 📁 个人/                              # 5️⃣ 个人
│   ├── 健康管理/                         # 体检报告/健康方案
│   └── 儿子教育/                         # 教育计划/学习材料
```

> 📌 **每个项目/板块目录**根节点下可选建两个特殊子目录：
> - `_archive/` — 已确认的历史版本归档（只读，永久保留）
> - `_drafts/` — 草稿/过程文件（周期性自动清理，见生命周期规则）
> 根目录只保留当前活跃的最终版文件（≤5个）。

### 各目录Token速查表

> **⚠️ 2026-05-18 实测纠正**：此前记录的"Token漂移/404"是误报，实际原因是用了错误的API端点。Token本身稳定，请使用正确的端点（见附录B）。
>
> **🆕 2026-05-20 新增**：每个板块下新增 `_archive/` 和 `_drafts/` 子目录，实现三区生命周期管理。

| 路径 | 当前Token（仅供快速定位，使用前验证） |
|------|------|
| Hermes生成文件 | `Otppfr9EelPIawdezL2csUXCnoh` |
| Hermes生成文件/_archive | `FK7pf0Y4OlVRBIds5JAc4sPmFnQg` |
| Hermes生成文件/_drafts | `RzlIfI9zlgr7m5sJ6RucQZCnsntg` |
| 项目 | `F8apfVtErlmfEvdPWXGcF6mZn2g` |
| 项目/_archive | `NiHKf1YihlJhBsd5MPGctXRznPc` |
| 项目/_drafts | `QUzmf6wxAlTAA1sCQPBcKjP7nkc` |
| 项目/AI家庭教师 | `H04qfCSORlT7FFdNQBdcDANwn8d` |
| 工作汇报 | 见`list_folder`获取最新 |
| 工作汇报/_archive | `PTZkfNQMIlX96TdN3VecSMCUtnvc` |
| 工作汇报/_drafts | `SJ3wfQ5OlXYeJhdKWN8cQ1k0nNf` |
| 工作汇报/EAST数据治理 | 见`list_folder`获取最新 |
| 工作汇报/竞聘材料 | 见`list_folder`获取最新 |
| 部门群产出 | 见`list_folder`获取最新 |
| 日常运营 | 见`list_folder`获取最新 |
| 日常运营/_archive | `FWjHfL3LglFvDJdtAvVctjHvYnpf` |
| 日常运营/_drafts | `WdvNfJ22yl6LLcdJC3Sc7hTrnja` |
| 日常运营/待办看板 | 见`list_folder`获取最新 |
| 部门群产出/🧠顾问团 | `RRkofnlQOlnN3VdrZ7mcAra4nYd` |
| 个人 | 见`list_folder`获取最新 |
| 个人/_archive | `TELBfhfKNlQQ78droAacdT510ntd` |
| 个人/_drafts | `R5MxfdqoPotSS0svOnocNF9fnid` |
| 个人/健康管理 | 见`list_folder`获取最新 |
| 个人/儿子教育 | 见`list_folder`获取最新 |

## 三、文件命名规范

### 命名格式

```
[类型前缀]_[内容简述]_v[版本]_[状态标签]_[YYYYMMDD].[格式]
```

> 示例：`设计_题库架构_v2_FINAL_20260512.html`

### 类型前缀速查

| 前缀 | 适用场景 | 示例 |
|------|---------|------|
| `设计` | 系统设计/架构方案 | `设计_题库架构_v2_FINAL_20260512.html` |
| `方案` | 实施方案/解决方案 | `方案_EAST治理_v1_FINAL_20260518.html` |
| `测试` | 测试案例/测试报告 | `测试_全链路_v1_FINAL_20260517.html` |
| `原型` | UI/UX原型 | `原型_家长中心_v2_FINAL_20260515.html` |
| `调研` | 调研报告/竞品分析 | `调研_AI落地方案_v1_FINAL_20260510.html` |
| `材料` | 汇报材料/PPT素材 | `材料_EAST二季度_v4_FINAL_20260516.html` |
| `日报` | 每日简报 | `日报_20260516_EAST进展_FINAL.html` |
| `周报` | 周度报告 | `周报_2026W20_系统运维_FINAL.html` |
| `分析` | 数据分析/审计报告 | `分析_Token消耗_v1_FINAL_20260520.html` |

### 状态标签

| 标签 | 含义 | 存放位置 | 清理策略 |
|------|------|---------|---------|
| **FINAL** | 用户确认的最终交付件 | 根目录（最新）→ _archive（旧版） | 永久保留 |
| **WIP** | 工作中，尚未确认 | 根目录 | 下一版生成时旧版标记OBSOLETE移_archive |
| **DRAFT** | 草稿/一次性过程输出 | _drafts/ | 超7天自动清理 |
| **OBSOLETE** | 已废弃/被新版取代 | _archive/ | 永久保留（可追溯） |

### 版本号规则

- 初版标 `v1`
- 后续迭代 `v1 → v2 → v3`
- 重大重构建议新起前缀（如 `设计_v2`）

## 四、路由规则 — 生成内容应去向

| 内容类型 | 目标目录 | 规则 |
|---------|---------|------|
| AI家庭教师设计文档 | Hermes生成/项目/AI家庭教师/ | 所有AI家庭教师相关设计/方案/测试文档 |
| EAST数据治理汇报 | Hermes生成/工作汇报/EAST数据治理/ | 所有EAST相关风险排查/汇报/材料 |
| 竞聘PPT/演讲稿 | Hermes生成/工作汇报/竞聘材料/ | 贺馨怡等竞聘相关文件 |
| 中银保信相关 | Hermes生成/工作汇报/中银保信/ | (如无子文件夹可直接放) |
| 顾问团审核报告 | Hermes生成/部门群产出/🧠顾问团/ | C007顾问团审计/建议/分析 |
| 待办看板/仪表盘 | Hermes生成/日常运营/待办看板/ | 任务看板/项目仪表盘 |
| 体检报告/健康方案 | Hermes生成/个人/健康管理/ | 个人健康相关 |
| 儿子教育计划/材料 | Hermes生成/个人/儿子教育/ | 三年级教育相关 |
| 系统文档类(永久) | 系统文档/对应子目录/ | Wiki/操作手册/架构参考 |
| 内容工厂发布 | 内容工厂出品/ | C014周度发布内容 |
| 部门群消息产物 | Hermes生成/部门群产出/对应部门/ | 自动归档到各部门目录 |

## 五、生命周期管理（三区模型）

每个文件经历 **创建 → 活跃 → 定稿 → 归档/淘汰** 的生命周期。每个项目/板块目录下通过三区分离管理：

### 三区定义

| 区域 | 目录 | 包含内容 | 文件数量 | 清理策略 |
|:---|:-----|:--------|:--------|:--------|
| 🔥 **热区 Active** | 根目录 | 当前活跃的最终版文件（最新 FINAL） | ≤5个 | 生成新版时旧版移入 _archive |
| 📦 **温区 Archive** | `_archive/` | 历史 FINAL + OBSOLETE 文件 | 不限 | 永久保留，只移不删 |
| 🗑️ **冷区 Drafts** | `_drafts/` | DRAFT 草稿/过程文件 | 不限 | 超7天自动清理 |

### 生命周期流转规则

| 当前状态 | 触发条件 | 下一步 | 执行时机 |
|---------|---------|-------|---------|
| WIP（未确认） | 生成新版或用户废弃 | 标记 OBSOLETE → 移 _archive | 立即 |
| WIP → **FINAL** | 用户确认"可以了" | 文件名中 WIP 升级为 FINAL | 确认时 |
| FINAL（当前版） | 生成新版 FINAL | 当前版 FINAL → 移 _archive | 生成新版时 |
| DRAFT（草稿） | 超 7 天 | 自动删除 | 每周一 cron |
| OBSOLETE（已废弃） | — | 常驻 _archive，不删除 | — |

### 生成纪律（Agent 必须遵守）

每次生成文件时，附加以下生命周期步骤：

```
0. 确定文件状态：新生成→WIP / 用户确认→FINAL / 草稿→DRAFT
0a. 检查目标目录中是否已有同主题旧版FINAL → 有则移_archive
0b. 命名中必须包含状态标签+日期
1. 上传到根目录（FINAL/WIP）或 _drafts/（DRAFT）
```

## 六、生成流程（固化步骤）

每次生成文件时，必须按以下步骤执行：

```
0. 生命周期预检 → 确定文件状态（FINAL/WIP/DRAFT）
   0a. 检查目标目录是否有同主题旧版FINAL → 有则移入 _archive/
   0b. 命名必须包含状态标签 + 日期
1. 判断内容类型 → 参照路由规则表确定目标目录
2. 生成文件（默认HTML暗色科技风，PPT除外）
3. 上传到飞书云盘对应目录（FINAL→根目录，DRAFT→_drafts/）
4. 重命名确认：用户确认后，WIP→FINAL（不影响已上传文件，下次生成时生效）
5. 【可选】同步到本地知识库目录（见附录）
6. 点击分享按钮生成分享链接
7. 将链接发送到对应对话中
```

### 清理时序（自动化）

| 频率 | 执行内容 | 方式 |
|:---|:--------|:----|
| 每次生成时 | 旧版FINAL移_archive | Agent手动 |
| 每周一06:00 | 扫描_drafts/，删除超7天DRAFT | cron脚本 |
| 每月1日 | 报告各目录文件分布 | cron脚本 |

## 七、最佳实践

1. **先查Token再上传**：每次上传前用正确端点验证目标目录Token（见附录B）
2. **Token稳定不漂移**：2026-05-18实测，Token本身稳定（Hermes生成文件Otppfr9EelPIawdezL2csUXCnoh及其子目录均有效）。若遇404，先检查API端点是否正确，而非假定Token漂移
3. **文件夹不存在则创建**：按需创建子目录，遵循6大板块结构
4. **不直接放根目录**：所有文件必须有明确的归类目录
5. **部门群产出自动归档**：各部门群生成的内容直接放入对应的部门产出目录
6. **根目录保持整洁**：每个板块/项目的根目录活跃文件 ≤5个，超出立即归档
7. **文件名即元数据**：文件名必须包含 类型+内容+版本+状态+日期，一眼可辨文件状态
8. **DRAFT 不占用根目录**：草稿/过程文件始终放入 `_drafts/`，根目录只放 FINAL
9. **旧版不删除只归档**：FINAL 文件绝不删除，只移入 `_archive/`；仅 DRAFT 可清理

## 附录：知识库同步（已配置）

### 同步架构

```
Hermes生成文件(飞书云盘)
        │
        ├──→ 本地 ~/.hermes/output/    (每次生成时保存HTML/DOC副本)
        │
        └──→ cron(每6h) → ~/.hermes/scripts/kb_sync_hermes.py
                    │
                    ▼
        /mnt/g/WPSDocument/Hermes生成/  (KB扫描源)
                    │
                    ▼
        ChromaDB 向量化 → 语义检索
```

### 已配置的组件

| 组件 | 状态 | 说明 |
|------|------|------|
| 本地镜像目录 `~/wiki/raw/hermes-generated/` | ✅ | 文件本地副本 |
| KB扫描源 `/mnt/g/WPSDocument/Hermes生成/` | ✅ | KB增量管道自动索引 |
| 同步脚本 `kb_sync_hermes.py` | ✅ | `~/.hermes/scripts/` |
| 定时同步cron | ✅ | 每6小时自动同步 |

### 使用方式

生成文件时执行以下步骤：

```python
# Step 1: 上传到飞书对应目录
# (见路由规则表，API端点见 references/feishu-api-endpoint-guide.md)
```

## 附录B: API 端点速查

> 见 `references/feishu-api-endpoint-guide.md` — 已验证有效的Tokens、正确/错误端点对照、upload_all 5字段细节。
# Step 2: 同时保存本地副本
output_path = os.path.expanduser("~/.hermes/output/filename.html")
with open(output_path, "w") as f:
    f.write(html_content)

# Step 3: 通知KB同步（可选，cron会自动处理）
import subprocess
subprocess.run(["python3", os.path.expanduser("~/.hermes/scripts/kb_sync_hermes.py")])

# Step 4: 发送飞书链接
# ...
```

