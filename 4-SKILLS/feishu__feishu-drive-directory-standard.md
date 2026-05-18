---
name: feishu-drive-directory-standard
description: 飞书云盘完整目录标准 — 6大板块分类、文件命名规范、生成路由规则、知识库同步策略
version: 1.0.0
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
│   └── 待办看板/                         # 任务看板/仪表盘
│
├── 📁 个人/                              # 5️⃣ 个人
│   ├── 健康管理/                         # 体检报告/健康方案
│   └── 儿子教育/                         # 教育计划/学习材料
```

### 各目录Token速查表

> **⚠️ 2026-05-18 实测纠正**：此前记录的"Token漂移/404"是误报，实际原因是用了错误的API端点。Token本身稳定，请使用正确的端点（见附录B）。

| 路径 | 当前Token（仅供快速定位，使用前验证） |
|------|------|
| Hermes生成文件 | `Otppfr9EelPIawdezL2csUXCnoh` |
| 项目 | `F8apfVtErlmfEvdPWXGcF6mZn2g` |
| 项目/AI家庭教师 | `H04qfCSORlT7FFdNQBdcDANwn8d` |
| 工作汇报 | 见`list_folder`获取最新 |
| 工作汇报/EAST数据治理 | 见`list_folder`获取最新 |
| 工作汇报/竞聘材料 | 见`list_folder`获取最新 |
| 部门群产出 | 见`list_folder`获取最新 |
| 日常运营 | 见`list_folder`获取最新 |
| 日常运营/待办看板 | 见`list_folder`获取最新 |
| 部门群产出/🧠顾问团 | `RRkofnlQOlnN3VdrZ7mcAra4nYd` |
| 个人 | 见`list_folder`获取最新 |
| 个人/健康管理 | 见`list_folder`获取最新 |
| 个人/儿子教育 | 见`list_folder`获取最新 |

## 三、文件命名规范

### 命名格式

```
[类型前缀]_[内容简述]_[版本号].[格式]
```

### 类型前缀速查

| 前缀 | 适用场景 | 示例 |
|------|---------|------|
| `设计` | 系统设计/架构方案 | `设计_AI家庭教师题库架构_v2.0.html` |
| `方案` | 实施方案/解决方案 | `方案_EAST数据质量治理_v1.html` |
| `测试` | 测试案例/测试报告 | `测试_AI家庭教师用户端_v1.html` |
| `原型` | UI/UX原型 | `原型_Pad端学习模式_v1.html` |
| `调研` | 调研报告/竞品分析 | `调研_企业AI Agent落地方案.html` |
| `材料` | 汇报材料/PPT素材 | `材料_EAST二季度工作会_v4.html` |
| `日报` | 每日简报 | `日报_20260516_EAST进展.html` |
| `周报` | 周度报告 | `周报_2026W20_系统运维.html` |
| `分析` | 数据分析/审计报告 | `分析_Token消耗周报_v1.html` |
| `设计` | 视觉/交互设计 | `设计_Pad端设计系统_v1.html` |

### 版本号规则

- 初版不标版本号，或标 `v1`
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

## 五、生成流程（固化步骤）

每次生成文件时，必须按以下步骤执行：

```
1. 判断内容类型 → 参照路由规则表确定目标目录
2. 生成文件（默认HTML暗色科技风，PPT除外）
3. 上传到飞书云盘对应目录
4. 【可选】同步到本地知识库目录（见附录）
5. 点击分享按钮生成分享链接
6. 将链接发送到对应对话中
```

## 六、最佳实践

1. **先查Token再上传**：每次上传前用正确端点验证目标目录Token（见附录B）
2. **Token稳定不漂移**：2026-05-18实测，Token本身稳定（Hermes生成文件Otppfr9EelPIawdezL2csUXCnoh及其子目录均有效）。若遇404，先检查API端点是否正确，而非假定Token漂移
3. **文件夹不存在则创建**：按需创建子目录，遵循6大板块结构
4. **不直接放根目录**：所有文件必须有明确的归类目录
5. **部门群产出自动归档**：各部门群生成的内容直接放入对应的部门产出目录

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

