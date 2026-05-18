---
name: aider-coding
description: "Delegate coding to Aider CLI (features, PRs, refactoring). v1.0.0 — WSL-native DeepSeek/Ollama integration. Uses SEARCH/REPLACE edit format with auto-commit, lint, and repo-map."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, wsl]
metadata:
  hermes:
    tags: [Coding-Agent, Aider, DeepSeek, Ollama, Code-Review, Refactoring, WSL]
    related_skills: [codex, subagent-driven-development]
---

# Aider CLI

Delegate coding tasks to [Aider](https://aider.chat/) via the Hermes terminal. Aider is an AI pair programming CLI that works directly with git repos.

## When to use

- Building features (multi-file changes)
- Refactoring with repo awareness (auto repo-map)
- Batch issue fixing (multiple files in one session)
- First choice for WSL coding work (reliable, no sandbox dependency)
- Replaces Codex as the primary coding sub-agent on WSL

## Prerequisites

- Aider installed in `~/.aider-venv`: `~/.aider-venv/bin/aider`
- Wrapper script at `~/.local/bin/aider-wrapper`
- DeepSeek V4 Flash configured (DEEPSEEK_API_KEY in `~/.hermes/.env`)
- **Must run inside a git repository** — Aider refuses to run outside one
- Files to edit MUST be listed on the command line

## C005 工作流集成

Aider 作为 C005 编码 SubAgent 的标准用法（多Agent 角色分离协议）：

### 角色定位

| 角色 | 工具 | 职责 |
|:----:|:----:|:------|
| 设计者+验收条件定义者 | Hermes | Step 0-2 + Step 2.7 原子验收条件 |
| **编码执行者** | **Aider（aider-wrapper）** | Step 3 工程实现 |
| 审核者 | SubAgent（冷酷审查员） | D-Des 设计符合性门禁 |
| 决策者 | Hermes | 2 轮审核后 FAIL 升级处理 |

### 验收条件驱动的委托模式

```python
# Hermes 端 — 产出验收条件文件
docs/acceptance-criteria-[task].md  # 可二进制判定的检查清单

# 委托 Aider 执行 Step 3
delegate_task(
    goal=f"""使用 Aider (aider-wrapper) 实现验收条件"",
    context=f"""
    验收条件: {criteria_path}
    文件清单: {files_list}
    约束: 每条验收条件必须满足
    验证: cd {workdir} && python -m pytest tests/ -q
    """
)
```

### 两阶段修复循环

```
Aider 编码 → 审核 SubAgent 审查
  ├─ ✅ ALL PASS → 进入 Step 4 质量门禁
  └─ ❌ FAIL → Aider 修补（第1轮）
       └─ ❌ FAIL → Aider 再修补（第2轮）
            └─ ❌ FAIL → 升级 Hermes 决策（v4.4.0 2轮升级机制）
```

### WSL 环境选择指南

| 条件 | 推荐选择 | 原因 |
|:----|:---------|:-----|
| 在 WSL 上工作 | **Aider** | Codex v0.130 在 WSL 有 sandbox/bwrap 缺失、启动超时问题 |
| 非 WSL (Linux/Mac) | Codex 或 Aider | Codex sandbox 可用 |
| 需要 repo-map 跨文件感知 | Aider | 原生支持，Codex 需手动组装上下文 |
| 纯 Python 单文件修改 | 两者均可 | Aider 非 TTY SEARCH/REPLACE 有文件名 artifact 风险 |

Use the wrapper script for all invocations:

```bash
aider-wrapper --message "instruction" file1.py file2.py
```

The wrapper sources DEEPSEEK_API_KEY from `~/.hermes/.env` and invokes aider with correct flags.

### Key flags (hardcoded in wrapper)

| Flag | Purpose |
|------|---------|
| `--model deepseek/deepseek-chat` | DeepSeek V4 Flash |
| `--yes-always` | Auto-approve all edits |
| `--api-key deepseek=$KEY` | Auth token |
| `--no-suggest-shell-commands` | Disable shell command suggestions |
| `--no-gitignore` | Skip TTY check for .gitignore |

### Custom flags (pass through)

```bash
aider-wrapper --model deepseek/deepseek-chat --message "..." file.py
aider-wrapper --verbose --message "..." file.py
aider-wrapper --read reference.md --message "..." file.py  # Read reference files
```

## One-Shot Tasks (Create New File)

**IMPORTANT**: The target file MUST be listed on the command line:

```bash
# ✅ Correct — file listed explicitly
aider-wrapper --message "Create hello.py with a greeting" hello.py

# ❌ Wrong — no file specified, aider will say "Please add hello.py to the chat"
aider-wrapper --message "Create hello.py with a greeting"
```

## One-Shot Tasks (Modify Existing Files)

```bash
# Single file
aider-wrapper --message "Add error handling to fetch_data()" src/fetcher.py

# Multiple files (repo-map handles cross-file awareness)
aider-wrapper --message "Extract parse_config() to a shared module" src/main.py src/config_parser.py
```

## One-Shot Tasks (With Reference Context)

```bash
# Read reference file as context
aider-wrapper --read specs/api.md --message "Implement the /users API endpoint" src/routes.py
```

## Background Mode (Long Tasks)

```bash
terminal(command="aider-wrapper --message 'Refactor the auth module to use JWT' auth.py auth_handler.py",
         workdir="~/project", timeout=300)
```

## Delegation from Hermes (SubAgent Pattern)

```python
# Hermes side — delegate to Aider subagent
result = delegate_task(
    goal=f"""
    1. Read the acceptance criteria from: {criteria_path}
    2. Use Aider (aider-wrapper) to implement the feature
    3. Files to create/modify: {files_list}
    4. Verify: cd {workdir} && python -m pytest tests/ -q
    5. Output: implement_result.json with files_modified + pass status
    """,
    context=f"""
    Project: {project}
    Invocation: aider-wrapper --read {criteria_path} --message "Implement from criteria" {files_list}
    Constraints: All acceptance criteria must be met, tests must pass
    Note: target files must be listed on command line even for new files
    """,
    toolsets=['terminal', 'file']
)
```

## Pitfalls

- **⚠️ Files must be listed on CLI** — Aider in non-TTY mode won't create/edit files that aren't explicitly named. A single omission → "Please add hello.py to the chat"
- **⚠️ Non-TTY SEARCH/REPLACE artifacts (v0.86.2)** — When Aider's model produces conversational text before the SEARCH/REPLACE block (e.g. "Now produce SEARCH/REPLACE blocks."), Aider incorrectly parses it as a filename. The resulting file has the wrong name (e.g. "Now produce SEARCH/REPLACE blocks.md_to_html.py") and the intended file is created empty. **Fix**: rename the artifact file to the intended name and check git diff.
- **⚠️ `--map-refactor` unsupported in v0.86.2** — This flag (`aimap-refactor` in newer Aider versions) causes `unrecognized arguments` error. Don't set it in config.
- **no TTY needed** — Unlike Codex, Aider works fine without `pty=true` in non-interactive mode
- **auto git commit** — Aider automatically commits changes (can override with `--no-auto-commits`)
- **API key** — Pass via `--api-key deepseek=KEY` or environment variable. The `.env` file in the project directory is auto-read if present.

## Jinja2 Template Pitfalls

Aider 生成的 Jinja2 模板容易混入 Python 原生语法，造成运行时 500 错误。

### `abs()` vs `|abs`

```jinja
{# ❌ 错误 — Jinja2 没有全局 abs() 函数 #}
<div class="total">{{ abs(total_coins) }}</div>

{# ✅ 正确 — 使用 Jinja2 的 abs filter #}
<div class="total">{{ total_coins|abs }}</div>
```

**根因**: Jinja2 不是 Python。内置函数如 `abs()`、`len()`、`str()`、`round()` 在模板中不可用。部分有等价 filter（`|abs`, `|length`, `|string`, `|round`），部分需在 Flask 侧通过 `app.jinja_env.globals` 注册或传入模板上下文。

### 常见陷阱对照表

| Python 表达式 | Jinja2 等价写法 | 说明 |
|:--------------|:----------------|:-----|
| `{{ abs(x) }}` | `{{ x\|abs }}` | filter 语法 |
| `{{ len(x) }}` | `{{ x\|length }}` 或 `{{ x.count() }}` | filter 或方法 |
| `{{ str(x) }}` | `{{ x\|string }}` 或直接 `{{ x }}` | auto-escape 已处理 |
| `{{ min(a,b) }}` | `{{ [a,b]\|min }}` | 用 list filter |
| `{{ dict.keys() }}` | `{{ dict\|list }}` 或 `{% for k in dict %}` | 遍历用 for |

### 验证方法

Aider 完成模板修改后，用 test_client 触发渲染验证：

```python
with app.test_client() as c:
    # 模拟登录
    with c.session_transaction() as sess:
        sess['parent_logged_in'] = True
    # 渲染页面
    r = c.get('/parent/coin-overview')
    assert r.status_code == 200, f"Template error: {r.status_code}"
    # 如果 500，检查日志定位具体行号
```

建议在验收条件中添加一条「所有修改后的模板在 test_client 下 GET 返回 200」。
