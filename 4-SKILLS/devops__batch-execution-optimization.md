---
name: batch-execution-optimization
title: 批量执行优化 — 减少来回调用
description: 多步骤任务（3+步）采用 execute_code 批量编排，减少独立 terminal/cron/API 调用次数。核心原则：一次认证、一次规划、一次写、一次验证。
category: devops
trigger: 需要执行 3 步以上的独立操作（如创建多个文件、多次 API 调用、多次验证）
---

# 批量执行优化工作流

## 核心原则

在接到需要 **3 步以上** 的多步骤任务时，不要串行执行一个个独立工具调用。应该：

1. **先规划** — 在 execute_code 中把所有要写入的内容、要调用的 API、要创建的 cron 在内存中准备好
2. **批量写** — 一次性 write_file 或 patch 所有文件
3. **单次验证** — 一次 execute_code 验证所有模块的加载和功能
4. **最小化网络往返** — API 认证只需一次，不要每次 terminal 都重新获取 token

## 🔴 陷阱：不要对同一文件的顺序依赖变更使用批量

### 何时不应该批量

批量执行的适用前提是：**各写入/修改操作之间无依赖**。

当满足以下任一条件时，**不应该批量，而应该顺序执行 patch**：

| 场景 | 正确做法 | 错误做法 |
|:-----|:---------|:---------|
| 多个 patch 作用于同一文件 | 顺序 patch，每次验证后再下一个 | 在 execute_code 中一次性生成所有 patch 内容 |
| 后续 patch 依赖前一个 patch 的 diff 位置 | 按依赖顺序串行执行 | 批量执行 → 第二个 patch 的旧文本已不存在 |
| 修改正在运行的配置文件 | 每步修改后重启/重载检查 | 批量写入后才发现中间状态崩溃 |
| 单文件大规模功能新增（300+行） | 按功能区域分批 patch（CSS→HTML→JS→集成） | 一次写入整个修改 → 难以定位错误 |

**判断准则**：如果所有修改集中在 1-2 个文件且修改间有语义依赖链，走顺序 patch；如果涉及 3+ 个独立文件（如一个配置文件 + 一个脚本 + 一个 cron 定义），走批量。

### 本 session 的实战案例（2026-05-20 行号污染事故）

在批量替换6个模板文件的深色值到浅色值时，使用 `read_file()` 读取内容后直接传给 `write_file()`，导致所有文件被行号前缀污染：

```python
# ❌ 错误做法 — 行号前缀会嵌入文件内容
content = read_file("template.html")  # 返回 "1|<html>\\n2|<head>..."
write_file("template.html", content)  # 内容变成 "1|<html>\\n2|<head>..."

# ✅ 正确做法
content = open("template.html").read()  # 原始内容
# 或
result = terminal("cat template.html")
content = result['output']

write_file("template.html", content)  # 干净写入
```

`read_file()` 的 `NUM|` 前缀是给人读的渲染格式，不是可回写的原始内容。**永远不要将 read_file() 的输出直接传给 write_file()**。

---

### 本 session 的实战案例（2026-05-06 航空航天题库闪卡模式）

对一个 168KB 的单页 HTML 文件添加闪卡模式，所有改动在同一文件中且前后依赖：

① CSS 添加（.flashcard-* 样式）
② HTML 添加（闪卡视图区）  → 依赖 CSS 类名
③ JS MODES 数组添加          → 依赖 HTML 中 mode-grid
④ JS selectMode 更新          → 依赖 ② 的新 HTML 元素 ID
⑤ JS startQuiz 路由           → 依赖 ③ 的 mode.id
⑥ JS 闪卡核心函数             → 依赖 ②/④/⑤
⑦ init 集成                   → 依赖 ⑥

特点是：
- **每步不可跳过验证** — 中间任一步语法错误，整个页面崩溃（单文件单 script 块）
- **因果链紧密** — 后续函数名被前一步引用，批量生成无法感知中间状态
- **最终验证依赖全路径** — 只有所有 patch 完成后才能浏览器测试

## 典型反模式（不要这样做）

❌ 串行独立调用：
```
terminal("list folder A")  → 启动 Python、认证、调用 API
terminal("list folder B")  → 又启动 Python、认证、调用 API  
write_file "file1.py"
terminal("test file1")     → 再次启动 Python
patch "file2.py"
terminal("test file1+2")   → 又一次启动
cronjob create
terminal("verify all")     → 最后一次
```

## 优化模式（应该这样做）

```
execute_code:
  1. 一次获取 token（全局变量复用）
  2. 调用所有需要的 API（遍历目录、获取信息）
  3. 在内存中构建所有文件内容（tokens.json、脚本.py、补丁内容）
  4. 用 write_file 一次写入所有文件
  5. 用 patch 一次应用所有修改
  6. 在同一个进程中验证所有模块加载

  输出: 准备好要执行的 write_file/patch/cron 指令清单

→ 然后执行输出清单（批量写入 + 批量 patch + 单次 cron）
→ 最后单次 execute_code 验证
```

## 具体做法

### 数据收集阶段（execute_code）

```python
from hermes_tools import web_search, terminal, write_file, patch, read_file

# 1. 一次认证，多次使用
token_data = get_token_once()  # 自行实现缓存

# 2. 在内存中构造所有输出
outputs = []

# 3. 收集所有需要的信息
for item in items_to_process:
    result = call_api(token_data, item)
    outputs.append(result)

# 4. 构建文件内容
file1_content = build_file1(outputs)
file2_content = build_file2(outputs)
patch1 = build_patch1(...)
patch2 = build_patch2(...)

# 5. 输出具体要执行的指令清单
print(json.dumps({
    "write_files": [
        {"path": "path/to/file1", "content": file1_content},
        {"path": "path/to/file2", "content": file2_content},
    ],
    "patches": [...],
    "crons": [...],
}))
```

### 执行阶段（主流程中）

根据 execute_code 的输出，在主流程中直接调用：
- `write_file()` — 每个文件一次调用
- `patch()` — 每个修改一次调用
- `cronjob()` — 每次创建一次调用

### 验证阶段（execute_code）

```python
# 单次 Python 进程验证所有模块
import sys, importlib
for mod_name, mod_path in modules:
    spec = importlib.util.spec_from_file_location(...)
    # 测试导入
    # 测试关键函数
    # 测试 token 加载
print("✅ 全部验证通过")
```

## 回退隔离机制（防 patch 崩溃）

在修改 live 文件前，**必须**创建隔离副本。防止 patch 失败导致文件损坏。

### 标准流程

```
修改前：cp target.py target.py.edit    # 创建隔离副本
         │
         ├─ 在副本上执行所有 patch
         │
         ├─ 验证通过 → mv target.py.edit target.py   # 原子替换
         │
         └─ 验证失败 → rm target.py.edit             # 丢弃副本，零影响
```

### 封装工具

将上述流程封装到 `execute_code` 中：

```python
import os, shutil, subprocess

def safe_patch(filepath: str, old_str: str, new_str: str) -> bool:
    \"\"\"回退隔离的 patch 操作\"\"\"
    edit_path = filepath + ".edit"
    
    # 1. 创建副本
    shutil.copy2(filepath, edit_path)
    
    # 2. 在副本上 patch
    with open(edit_path) as f:
        content = f.read()
    if old_str not in content:
        os.remove(edit_path)
        print(f"❌ old_string 未找到: {filepath}")
        return False
    content = content.replace(old_str, new_str, 1)
    with open(edit_path, 'w') as f:
        f.write(content)
    
    # 3. 验证语法
    result = subprocess.run(
        [sys.executable, '-c', f'import py_compile; py_compile.compile({repr(edit_path)}, doraise=True)'],
        capture_output=True, text=True
    )
    if result.returncode != 0:
        os.remove(edit_path)
        print(f"❌ 语法错误: {result.stderr[:100]}")
        return False
    
    # 4. 原子替换
    os.replace(edit_path, filepath)
    print(f"✅ 安全 patch 成功: {os.path.basename(filepath)}")
    return True
```

### 适用场景

| 场景 | 必须用回退隔离 | 建议 |
|:----|:------------:|:-----|
| 单个 patch 修改 < 10 行 | ⚠️ 建议 | 低风险，但养成习惯 |
| 多个 patch 顺序作用于同一文件 | **✅ 必须** | 防止第N个patch崩溃拖垮前面M个 |
| 跨文件 patch（主版本+镜像版本） | **✅ 必须** | 一个失败不影响另一个 |
| 首次修改不熟悉的文件 | **✅ 必须** | 不可预测的缩进/编码问题 |

- [ ] API 认证是否只做了一次？（排除必要的不同服务认证）
- [ ] 是否有 3 个以上的独立 terminal 调用？→ 应该合并
- [ ] 文件写入是否可以合并到一次 execute_code 的输出中？
- [ ] 是否有一边写一边改（先写文件、再发现错误、再 patch 修复）？→ 应该在 execute_code 中一次完成
