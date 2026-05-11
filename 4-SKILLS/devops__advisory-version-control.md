---
name: advisory-version-control
description: >-
  C007 Advisory Council — version control module (version_control.py).
  Snapshots, rollback, changelog, and auto-pruning for 5 types of configuration
  spread across advisory_*.py modules. Used when deploying updates, recovering
  from bad changes, or auditing configuration state history.
version: 1.0.0
author: Hermes Agent
metadata:
  hermes:
    tags:
      - c007
      - version-control
      - rollback
      - snapshot
      - operations
    category: devops
---

# C007 顾问团系统 — 版本控制

> 管理顾问团5类配置的版本快照、回滚、变更日志。

## 使用时机

- **部署新版本后**：执行 `--create` 保存基线快照
- **配置改错了**：用 `--rollback` 回滚到已知正常版本
- **排查问题**：用 `--changelog` 看谁在什么时候改了配置
- **审计配置历史**：用 `--list` 查看所有快照及配置概览
- **手动清理**：`--prune` 触发旧版本清理

## 底层原理

`version_control.py` 位于 `~/.hermes/scripts/version_control.py`，通过动态导入 + 模块命名空间写回实现配置管理。

### 5类快照配置项

| 源文件 | 快照的配置 | 取值方式 |
|--------|-----------|---------|
| `advisory_roles.py` | `RESIDENT_CONSULTANTS` + `DYNAMIC_CONSULTANTS` | 直接读取模块变量 |
| `advisory_orchestrator.py` | `AdvisoryOrchestrator.PHASES` (stage prompts) | 读取类属性 |
| `speaker_selector.py` | `SPEAKER_MODE` + `rotation_rules` | try/except 降级（文件可能不存在） |
| `advisory_memory.py` | `WorkingMemory.max_rounds` (WORKING_MEMORY_ROUNDS) | inspect 读取 __init__ 默认参数 |
| `advisory_budget.py` | `WATERMARK_80/90/95` 水位阈值 | 读取类常量 |

### 架构要点

- **快照存储**：`~/.hermes/advisory_memory/version_snapshots/snapshot_<version>.json`
- **变更日志**：`~/.hermes/advisory_memory/version_changelog.jsonl`（追加式 JSONL）
- **保留策略**：自动保留最新 5 个版本（`MAX_VERSIONS=5`），创建新版本时触发清理
- **实验比例**：`EXPERIMENT_RATIO=0.1`（10% 流量走新版本，用于 A/B 自动生效场景）
- **原子回滚**：rollback 会先预检全部 5 类配置，任一不可恢复则中止，不写任何模块

## CLI 命令

```bash
# 查看所有命令
python3 ~/.hermes/scripts/version_control.py --help

# 创建快照
python3 ~/.hermes/scripts/version_control.py --create <version>

# 回滚到指定版本
python3 ~/.hermes/scripts/version_control.py --rollback <version>

# 列出所有快照（含配置概览）
python3 ~/.hermes/scripts/version_control.py --list

# 查看变更日志
python3 ~/.hermes/scripts/version_control.py --changelog

# 手动清理旧版本
python3 ~/.hermes/scripts/version_control.py --prune
```

## Python API

```python
from version_control import create_snapshot, rollback, list_snapshots, get_changelog

# 创建快照 → 返回文件路径
path = create_snapshot("v3.1.0")

# 回滚 → 返回 bool
success = rollback("v3.0.0")

# 列出快照 → list[dict]
snapshots = list_snapshots()

# 查看最近变更 → list[dict]
entries = get_changelog(limit=10)
```

## 关键设计决策（不可违背）

### 1. 优雅降级：speaker_selector.py 的动态可用性
`speaker_selector.py` 现已存在（`advisory_session.py` v3.0+ 直接导入）。`_collect_speaker_selector()` 仍使用 try/except 捕获 `ImportError` 和 `ModuleNotFoundError`，以 `_present=False` 标记（保留降级路径）。`_list` 概览在文件不可用时显示"轮值: 未配置"。回滚时如果文件不存在则跳过该模块的写回。**保留 try/except 降级，即使文件当前存在**。

### 2. 原子回滚：先检后写
rollback 内部的执行顺序是：① 加载快照 → ② 预检全部 5 类（逐项确认可恢复）→ ③ 写回到模块命名空间。任何单项失败都会中止整个操作，不写任何修改。**禁止改为逐个回滚**。

### 3. 动态模块加载
```python
_ensure_scripts_importable()  # sys.path.insert(0, SCRIPTS_DIR)
_reload_modules()             # importlib.reload(sys.modules[...])
```
这两个函数必须在 `create_snapshot()` 开头调用，确保捕获最新状态。**禁止跳过**。

### 4. 清理策略
`_prune_old_snapshots()` 按快照文件内的 `created_epoch` 排序，保留最新的 MAX_VERSIONS(5) 个。损坏的快照文件也参与清理。**禁止手动删除快照文件而不触发清理**。

## 常见坑

| 问题 | 表现 | 解决方法 |
|:----|:-----|:---------|
| `NameError: name 'importlib' is not defined` | 创建快照时崩溃 | 在文件顶部的 import 块加入 `import importlib` |
| `speaker_selector` 模块不存在时查询失败 | 列表显示"轮值: 未配置"是正常行为 | 模块已存在，保留降级代码供异常回滚时兼容 |
| 回滚后模块状态未立即生效 | advisory_orchestrator 还在用旧 PHASES | `importlib.reload(sys.modules['advisory_orchestrator'])` 或重启 Hermes 会话 |
| 快照文件损坏 | `--list` 显示 "corrupted" | 手动删除文件或用 `--prune` 触发清理 |
