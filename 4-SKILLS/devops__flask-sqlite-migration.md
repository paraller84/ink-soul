---
name: flask-sqlite-migration
category: devops
description: >-
  将 Flask 应用的 JSON 文件存储迁移到 SQLite 数据库的完整模式。
  包含统一数据层设计、幂等迁移脚本、向后兼容层、零破坏策略。
version: 1.2.0
---

# Flask JSON→SQLite 迁移模式

## 何时使用

当现有 Flask 应用使用 JSON 文件存储数据，需要迁移到 SQLite 时：

- 多个子系统各自独立使用 JSON 文件（状态文件分散）
- 需要统一数据层供所有子系统共享
- 需要在迁移过程中保持向后兼容（零破坏）
- 需要同时添加统一认证或其他跨子系统功能
- 新子系统需要复用已有的数据结构

**不要用于**：
- 全新 Flask 应用（直接用 SQLite 即可）
- 已有 SQLAlchemy/ORM 的应用
- 仅修改单个 Blueprint 的简单功能

## 核心架构模式

### 三层设计

```
┌─────────────────────────────────────────────────┐
│  兼容层 (compat_layer.py)                       │
│  旧接口签名保持不变，底层切换到 SQLite          │
│  load_state() → load_state_compat()            │
├─────────────────────────────────────────────────┤
│  数据层 (database.py)                           │
│  SQLite 单例 + Schema 初始化 + CRUD 便利函数    │
│  12+ 张系统表 + 子系统扩展表                    │
│  WAL 模式 + busy_timeout + 线程安全             │
├─────────────────────────────────────────────────┤
│  迁移层 (migrate*.py)                           │
│  幂等迁移引擎 + 版本追踪 + 日志                 │
│  JSON 读取 → 校验 → SQLite 写入 → 标记已迁移    │
│  迁移后 JSON 保留不动（备份角色）                │
└─────────────────────────────────────────────────┘
```

### 数据库 Schema 关键设计

```sql
-- 迁移追踪表（幂等性保障）
schema_migrations(version UNIQUE, description, applied_at, status)

-- 子系统注册表（扩展性）
subsystems(name UNIQUE, label, version, enabled, metadata)

-- 通用业务表（可跨子系统复用）
parents, students, batches, batch_items
learning_states, practice_records

-- 子系统专用表（扩展表 naming convention: {subsystem}_*）
math_knowledge, math_wrong_questions, math_daily_logs
```

### Schema 设计原则

1. **所有表用 `IF NOT EXISTS`** — 幂等初始化
2. **FK 约束在迁移期间允许 NULL** — 迁移时数据来源可能不一致
3. **有意义的 UNIQUE 约束** — 去重保障
4. **索引覆盖查询模式** — 按 student_id、batch_id 等高频字段建索引
5. **WAL 模式** — 支持读写并发，无需 filelock

## 数据库连接管理

```python
# 核心模式：线程局部单例连接
_local = threading.local()

def get_db() -> sqlite3.Connection:
    if not hasattr(_local, "conn") or _local.conn is None:
        _local.conn = sqlite3.connect(str(_DB_PATH), check_same_thread=False)
        _local.conn.row_factory = sqlite3.Row
        _local.conn.execute("PRAGMA journal_mode=WAL;")
        _local.conn.execute("PRAGMA busy_timeout=5000;")
        _local.conn.execute("PRAGMA foreign_keys=ON;")
    return _local.conn

def close_db():
    if hasattr(_local, "conn") and _local.conn is not None:
        _local.conn.close()
        _local.conn = None
```

## CRUD 便利函数

在 Flask 的每个请求上下文启动时执行 `init_db()`，结束时执行 `close_db()`：

```python
# 在 app.py 中注册
from data import database as db
db.init_db()
@app.teardown_appcontext
def _close_db(exception):
    db.close_db()
```

提供 6 个核心 CRUD 函数（详见 references/database-api.md）：
- `fetch_one(sql, params)` — 单行查询
- `fetch_all(sql, params)` — 多行查询
- `insert(table, data)` — 插入
- `update(table, data, where, where_params)` — 更新
- `delete(table, where, params)` — 删除

## 迁移脚本结构

### 迁移引擎（migrate.py）

```python
def run_migration(version, description, json_paths, migrate_func, rename_after=True):
    """运行单个迁移，支持：
    - 幂等性：检查 schema_migrations 表
    - 错误回滚：标记 status=failed
    - JSON 重命名：迁移后 .json → .json.migrated
    """
    
def run_all_migrations(migrations: list) -> dict:
    """批量运行多个迁移（每个迁移独立追踪）"""
```

### 子迁移器结构

每个子系统有自己的 `migrate_{subsystem}.py`：

```python
# 路径配置
_ENGLISH_DATA_DIR = Path(__file__).parent.parent / "english" / "data"

# 单个迁移函数
def _migrate_content() -> bool:
    ...

# 主入口
def run_english_migrations() -> dict:
    return run_all_migrations([
        {"version": "v4.0-english-content",
         "description": "批次内容迁移",
         "json_paths": [_CONTENT_FILE],
         "migrate_func": _migrate_content,
         "rename_after": False},  # False = JSON 保留
        ...
    ])
```

### 导入路径处理

迁移脚本可能从多种上下文中调用（manager.py、CLI、import），需要稳固定位项目根目录：

```python
import sys
from pathlib import Path
_HUB_DIR = str(Path(__file__).parent.parent)
if _HUB_DIR not in sys.path:
    sys.path.insert(0, _HUB_DIR)

from data.database import get_db, insert, fetch_one  # 绝对导入
from data.migrate import run_migration, _read_json_file  # 绝对导入
```

⚠️ **不要使用相对导入**（如 `from ..data.database`）— 迁移脚本可能不在包上下文中运行。

## 导入路径双模式策略（Blueprint 内的 SQLite 辅助函数）

SQLite 辅助函数（如 `_is_sqlite_ready()`、`_get_learning_states_from_sqlite()`）通常放在 Blueprint 模块中。这类模块**可能从两种上下文中被导入**：

1. **Flask 包上下文**（`python app.py` 启动）：相对导入 `from ...data.database import ...` 工作正常
2. **独立运行**（测试、脚本、repl）：相对导入失败（`attempted relative import beyond top-level package`）

如果辅助函数用 `try/except` 包裹相对导入，失败会被**静默吞掉**，函数返回 False/空数据，且 **JSON fallback 会无声掩盖问题**。

### `_get_sqlite_db()` 双模式辅助模式

```python
# 在 Blueprint 模块的底部（SQLite 重写覆盖区）定义

_SQLITE_MODULE = None

def _get_sqlite_db():
    """获取 data.database 模块，兼容包内和独立运行上下文"""
    global _SQLITE_MODULE
    if _SQLITE_MODULE is not None:
        return _SQLITE_MODULE

    # 尝试 1：相对导入（Flask 包上下文）
    try:
        import sys as _sys
        _HUB_DIR = str(_Path(__file__).resolve().parent.parent.parent)
        # 注意：这里用 __file__ 推导项目的根目录，然后加到 sys.path
        if _HUB_DIR not in _sys.path:
            _sys.path.insert(0, _HUB_DIR)
    except Exception:
        pass

    # 尝试 2：绝对导入（已在 sys.path 中）
    try:
        from data.database import execute, get_db, fetch_one, fetch_all
        _SQLITE_MODULE = type('_SQLiteDB', (), {
            'execute': staticmethod(execute),
            'get_db': staticmethod(get_db),
            'fetch_one': staticmethod(fetch_one),
            'fetch_all': staticmethod(fetch_all),
        })()
        return _SQLITE_MODULE
    except ImportError:
        pass

    return None
```

然后在所有 SQLite 读/写辅助函数中使用：

```python
def _is_sqlite_ready() -> bool:
    db = _get_sqlite_db()
    if db is None:
        return False
    try:
        row = db.fetch_one("SELECT COUNT(*) as cnt FROM learning_states")
        return row and row["cnt"] > 0
    except Exception:
        return False
```

### 关键原则

- **不要**在各函数内部分别写 `from ...data.database import ...` — 分散且难以修复
- **不要**只在 try 块内写相对导入 — 失败无声，JSON fallback 掩埋问题
- **一定要**在所有 SQLite 辅助函数中统一使用 `_get_sqlite_db()` — 一处修复全局生效
- **一定要**在 `python3 -c` 和 Flask `app.run()` 两种上下文中分别验证读路径能返回真实数据
- **一定要**在 Phase 1 写入数据后，验证 Phase 2 读路径是否真正从 SQLite 读取（而非 JSON fallback）

### 验证方法

```bash
# 验证 SQLite 读路径是否真实工作
cd ~/edu-hub && python3 -c "
from data.database import init_db, fetch_one
init_db()
row = fetch_one('SELECT COUNT(*) as cnt FROM learning_states')
print(f'learning_states: {row[\"cnt\"]} rows')

from english.v3.data_manager import _get_sqlite_db, _is_sqlite_ready
db = _get_sqlite_db()
print(f'_get_sqlite_db() returns: {\"OK\" if db else \"NONE\"}')
print(f'_is_sqlite_ready(): {_is_sqlite_ready()}')

# 直接验证 get_memory_params 使用 SQLite
from english.v3.data_manager import get_memory_params
params = get_memory_params('stu_default')
print(f'get_memory_params(stu_default): {len(params)} words from SQLite')
"
```

如果 `get_memory_params` 返回 0 但 SQLite 表有数据，说明相对导入仍是无声失败的。

## 向后兼容模式

### 双轨并行策略

```python
# 在现有的 state_manager.py 末尾追加 SQLite 后端

# 1. 函数名重定义（覆盖旧的 import）
import sys as _sys_mod

def load_state():
    """SQLite-first, fallback to JSON"""
    try:
        from ..data.compat_layer import load_state_compat
        if load_state_compat.is_sqlite_ready():
            return load_state_compat()
    except ImportError:
        pass
    return _original_load_state()  # 回退到 JSON
```

### 兼容层（compat_layer.py）

从 SQLite 组装回旧 JSON 输出格式：

```python
def load_state_compat() -> dict:
    """从 SQLite 组装回 state.json 格式"""
    result = {"version": "4.0", "students": {}}
    students = fetch_all("SELECT * FROM students")
    for stu in students:
        # 重建嵌套结构：students → batches → categories → items
        ...
    return result
```

## 陷阱

### T6: SQLite schema 已存在但无人写入

**发现场景**：system/data/database.py 中有 learning_states、practice_records 等完整表格 schema，系统也正常运行，但 SQLite 表中**永远为空**——所有读写操作仍走 JSON。

**诊断方法**：
```bash
sqlite3 data/edu-hub.db "SELECT COUNT(*) FROM learning_states;"
# 返回 0 → 确认问题
```

然后追踪该系统的所有写入路径：
```bash
# 搜索所有可能写入 SQLite 的操作
grep -rn "INSERT.*learning_states\|INSERT.*practice_records" system/
# 搜索 vs JSON 写入
grep -rn "save_state\|_write_json\|json.dump" system/ | grep -v "migration\|backup"
```

**根因**：迁移只完成了 schema 创建和只读路径（SQLite-first 读），但**写路径从未补全**。新功能迭代中所有人继续写 JSON，SQLite 表永远是空表。

**修复**（分三步，每步可独立验证）：

```
阶段1 — 补全写路径：在现有 JSON 写入路径旁，增加 SQLite INSERT/UPDATE（双写过渡）
  关键：try/except 包裹 SQLite 写入，不阻塞 JSON 主路径
  验证：sqlite3 查询确认表有数据

阶段2 — 读路径切换：将读路径从 JSON 优先切换为 SQLite 优先
  关键：保留 JSON fallback（SQLite 出错或为空时自动回退）
  验证：关闭 JSON 文件后系统仍能正常运行

阶段3 — 历史迁移：批量迁移现有 JSON 数据 → SQLite
  关键：幂等迁移脚本（ON CONFLICT DO NOTHING）
  验证：迁移前后数据行数一致
```

**原则**：分阶段渐进，每阶段可独立验证和回退。SQLite 写入必须先于读路径切换存在。

**常见触发原因**：
- 先建了 SQLite schema（设计阶段），但开发迭代中持续写 JSON（快速原型阶段）
- 新增模块时未意识到有 SQLite 统一数据层存在
- 双写 try/except pass 让写入失败被静默吞掉，开发者以为 SQLite 写成功了

### T1: FK 约束在迁移期间失败

**发现场景**：迁移 `learning_states` 表时，`item_id` 引用 `batch_items(id)`，但某些 items 在 state.json 和 content.json 的键名不匹配，导致 `item_id=NULL` 被 FK `NOT NULL` 拒绝。

**根因**：`CREATE TABLE learning_states (item_id INTEGER NOT NULL REFERENCES batch_items(id))` — `NOT NULL` 阻止 NULL 值插入。

**修复**：让 FK 字段在迁移期间允许 NULL：
```sql
item_id INTEGER REFERENCES batch_items(id)   -- 去掉 NOT NULL
```

**原则**：迁移期间的 FK 字段应允许 NULL。数据一致性通过迁移脚本的业务逻辑保证，不应依赖数据库级约束在迁移阶段执行。

### T2: 学生 ID 在不同 JSON 文件中不一致

**发现场景**：`state.json` 使用 `students.stu_default`，而 `students.json` 使用 `stu_7fcee189`。迁移 `learning_states` 时 `REFERENCES students(id)` 失败。

**修复**：迁移函数中增加 ID 映射逻辑：

```python
existing = fetch_one("SELECT id FROM students WHERE id=?", [state_sid])
if not existing:
    first_stu = fetch_one("SELECT id FROM students ORDER BY id LIMIT 1")
    actual_sid = first_stu["id"] if first_stu else state_sid
```

### T3: Stale __pycache__ + 旧进程残留

**发现场景**：修改 `app.py` 注册新 Blueprint 后，重启服务后新路由返回 404。`python3 -c "from auth.routes import auth_bp; print(auth_bp.url_map)"` 显示路由已注册，但运行时 404。

**根因**：旧 Python 进程仍在运行（`pkill -f` 未匹配到），且 `__pycache__` 缓存旧字节码。

**修复**：
```bash
# 1. 强制杀死所有相关进程
kill -9 $(ss -tlnp | grep 5002 | grep -oP 'pid=\K[0-9]+' | head -1)

# 2. 清除所有 __pycache__
find ~/edu-hub -type d -name __pycache__ -exec rm -rf {} +

# 3. 等待端口释放
sleep 2

# 4. 重启
python3 scripts/edu-hub-manager.py start
```

### T4: `patch replace_all=true` 导致串联破坏

**发现场景**：对 `migrate_english.py` 执行 `replace_all=true` 将变量名 `sid` 替换为 `actual_sid`，导致 `for sid` 和 `sid = stu.get("id")` 都被替换，产生 `actual_actual_sid` 双重命名。

**原则**：`replace_all=true` 仅当替换字符串在所有上下文中语义一致时使用。变量名替换应逐处确认或用 `read_file` 整体重写。

### T7: 双写阶段变量名覆盖（Variable Shadowing）

**发现场景**：在 state_manager.py 的 `record_answer()` 中增加 SQLite 写入时，使用 `item` 作为 SQLite 查询结果变量，覆盖了函数中原有的 `item`（表示状态字典的变量）。结果 `ON CONFLICT` 写入的 `status` 和 `wrong_count` 来自 SQLite 行而非真实状态。

**根因**：Python 的动态变量赋值在嵌套函数或同一函数内允许重新绑定。当一个函数的局部变量 `item` 已经被赋值为业务数据字典后，再用 `item = execute(...)` 会**覆盖**原始引用。后续代码引用 `item["status"]` 时，读取的是 SQLite 行的属性（可能不存在或不同）。

**诊断方法**：
```bash
# 检查双写代码中是否有变量名重用
grep -n "item = execute\|bi_row = execute" modified_file.py
# 附近是否有 `item` 用作业务数据的代码？
grep -n "item\[.*status\|item\[.*wrong" modified_file.py
```

**修复**：使用区分度高的变量名（`bi_row`, `db_item`, `_sql_row` 等）：
```python
# 错误
item = execute("SELECT id FROM batch_items WHERE ...").fetchone()
if item:
    # item 已不是状态字典！

# 正确
bi_row = execute("SELECT id FROM batch_items WHERE ...").fetchone()
if bi_row:
    execute("INSERT ...", [student_id, bi_row["id"], item["status"]])  # item 仍是状态字典
```

**预防**：在 Step 3 实现阶段检查所有 SQLite 写入代码，确认 SQLite 查询结果变量名与函数已有的业务变量名不重复。详见 `references/dual-write-phase1-pattern.md`。

### T9: Flask `g` 上下文 — 独立脚本中 get_db() 失败

**发现场景**：独立运行的 Python 脚本（如出题管线 `agent_pipeline.py`）调用 `from db import get_db` 后执行 `get_db()`，抛出 `RuntimeError: Working outside of application context`。

**根因**：`db.py` 使用 Flask 的 `g` 对象（`from flask import g`）存储数据库连接。`g` 仅在 Flask 请求/应用上下文中可用，独立脚本中没有。

**两种连接模式对比**：

| 模式 | 适用场景 | 独立脚本支持 |
|:-----|:---------|:-------------|
| `threading.local()` + 自定义单例 | 自定义数据层 | ✅ 开箱可用 |
| Flask `g` 对象 | Flask Blueprint 模式 | ❌ 需要 app_context() |

**修复**（选择一种）：

**方案A（推荐——让脚本包装上下文）**——适用于已经使用 Flask `g` 的现有项目：
```python
from app import create_app
app = create_app()
with app.app_context():
    db = get_db()
    # 所有数据库操作放在这里
```

**方案B（不推荐——改造 db.py）**——修改现有 db.py 支持双模式：
```python
from flask import g
import threading

_local = threading.local()

def get_db():
    try:
        # 优先使用 Flask g（web上下文）
        if 'tutor_db' in g:
            return g.tutor_db
    except RuntimeError:
        pass  # 不在Flask上下文中
    
    # 回退到线程本地存储（独立脚本）
    if not hasattr(_local, 'conn') or _local.conn is None:
        _local.conn = _create_connection()
    return _local.conn
```

**预防**：在新项目中，如果预见到需要独立脚本操作数据库，从一开始就使用 `threading.local()` 模式而非 Flask `g` 模式。

**发现场景**：独立运行的 SQLite 脚本（migration、seed、debug）执行了 INSERT 但忘记调用 `db.commit()`。脚本打印了"已创建N条记录"，但脚本退出后数据全部消失。

**根因**：Python sqlite3 默认不开启自动提交（autocommit=False，除了某些特定执行模式）。不在 `with` 块或显式调用 `commit()` 时，INSERT/UPDATE/DELETE 在事务中执行，脚本结束未提交则自动回滚。

**诊断方法**：
```bash
sqlite3 path/to/db.db "SELECT COUNT(*) FROM target_table;"
# 返回 0，但脚本打印了"创建N条" → 确认缺少 commit
```

**修复**：
```python
# 在所有 INSERT/UPDATE/DELETE 后统一调用
db.commit()
print("DONE")  # commit 后再确保

# 或使用 with 上下文（自动提交/回滚）
with db:
    db.execute("INSERT INTO ...", (...))
```

**预防**：写完数据操作后 `db.commit()` 再打印完成信息。先用 `sqlite3` 命令行验证数据可见再确认。

### T5: 纯 `python3 -c` 测试 vs 运行时路径不同

迁移脚本的 import 路径依赖于 `sys.path` 包含项目根目录。从 `edu-hub-manager.py`（`scripts/` 目录）调用时 CWD 不同。始终在 `edu-hub/` 目录或用 `__file__` 推导。

## 验证清单

迁移完成后必须验证：

```
□ SQLite 表数量正确
□ 数据行数与 JSON 一致（或合理地少于 JSON，如去重后）
□ 现有 state_manager API 返回的结构包含相同字段
□ 所有 API 端点返回 HTTP 200（非 404/500）
□ Auth 端点返回正确的 JSON（非 HTML 404 页面）
□ 旧 JSON 文件仍然存在且未被修改
□ 管理脚本的 start/stop/status/logs/migrate 命令正常工作
```

## 参考文件

| 文件 | 说明 |
|:----|:------|
| `references/database-api.md` | database.py CRUD 函数参考 |
| `references/edu-hub-v4-migration.md` | 此会话的 Edu-Hub v4.0 迁移完整架构 |
| `references/dual-write-phase1-pattern.md` | Phase 1 双写过渡模式实战代码参考（含 variable shadowing 陷阱） |
| `references/sqlite-relative-import-trap.md` | SQLite 辅助函数相对导入无声失败的发现、根因、修复全记录 |
