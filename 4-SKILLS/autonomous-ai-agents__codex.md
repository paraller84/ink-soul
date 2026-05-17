---
name: codex
description: "Delegate coding to OpenAI Codex CLI (features, PRs). v1.1.0: WSL pitfalls documented (Reconnecting errors, bwrap, stdin). Aider recommended as WSL alternative."
version: 1.1.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Coding-Agent, Codex, OpenAI, Code-Review, Refactoring]
    related_skills: [claude-code, hermes-agent]
---

# Codex CLI

Delegate coding tasks to [Codex](https://github.com/openai/codex) via the Hermes terminal. Codex is OpenAI's autonomous coding agent CLI.

## When to use

- Building features
- Refactoring
- PR reviews
- Batch issue fixing

Requires the codex CLI and a git repository.

## Prerequisites

- Codex installed: `npm install -g @openai/codex`
- OpenAI auth configured (see "First-Time Setup" below for diagnosis)
- **Must run inside a git repository** — Codex refuses to run outside one
- Use `pty=true` in terminal calls — Codex is an interactive terminal app

## First-Time Setup (Hermes-Agent Initiated)

When a user says "install Codex" or "set up Codex", follow this diagnostic
and provisioning workflow. **Do NOT install without verifying credentials
first** — npm install succeeds but Codex will fail at its first use.

### Step 1: Environment Check

```python
import subprocess, os

# Check Node/npm
node_v = subprocess.run("node --version", shell=True, capture_output=True, text=True, timeout=10)
npm_v = subprocess.run("npm --version", shell=True, capture_output=True, text=True, timeout=10)

# Check if already installed
codex_check = subprocess.run("codex --version 2>&1 || echo 'NOT_INSTALLED'",
    shell=True, capture_output=True, text=True, timeout=10)
```

### Step 2: Credential Diagnosis (check ALL sources)

```python
sources = []

# A: Environment variable
key = subprocess.run('echo "$OPENAI_API_KEY"',
    shell=True, capture_output=True, text=True, timeout=5)
if key.stdout.strip():
    sources.append(("OPENAI_API_KEY", f"{key.stdout.strip()[:12]}..."))

# B: Codex CLI OAuth
codex_auth = os.path.expanduser("~/.codex/auth.json")
if os.path.exists(codex_auth):
    sources.append(("~/.codex/auth.json", "EXISTS"))

# C: Hermes auth (for model.provider: openai-codex)
hermes_auth = os.path.expanduser("~/.hermes/auth.json")
if os.path.exists(hermes_auth):
    import json
    with open(hermes_auth) as f:
        auth = json.load(f)
    if 'openai-codex' in auth or 'openai' in auth:
        sources.append(("hermes auth", "CONFIGURED"))

return sources  # Empty list = no credentials found
```

### Step 3: User-Facing Ask (when no credentials found)

**Output format** — a concise summary of what was found + clean ask:

```
📋 Codex Setup Diagnosis
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Node:       v22.22.2 ✅
  npm:        10.9.7 ✅
  Codex CLI:  NOT_INSTALLED
  ─────────────────────────────────
  OPENAI_API_KEY:      🔴 empty
  ~/.codex/auth.json:  🔴 not found
  hermes auth:         🔴 not configured
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

⚠️  Codex CLI needs OpenAI credentials. Two options:

A) Give me your OpenAI API Key → I write it to ~/.bashrc
B) Run `npx codex login` in your terminal (browser OAuth) →
   token cached in ~/.codex/auth.json
```

**Install only after credentials are confirmed** — run `npm install -g @openai/codex`.

### Step 4: Post-Install Verification

```bash
codex --version                    # Should print version
# Do NOT use `echo '{}' | codex` for connectivity test — Codex captures stdin
# and will print "Reading additional input from stdin..." then hang.
# Instead, just verify --version and then try a real exec:
mkdir -p /tmp/codex-verify && cd /tmp/codex-verify && git init && \
  echo 'print("hello")' > test.py && git add -A && git commit -m "init" && \
  codex exec "Check that this file exists and print its content" --model gpt-4o
```

### WSL/Headless-Specific Notes

- **No browser in WSL** → OAuth login via `npx codex login` requires a browser
  window on the Windows host. Either:
  1. Use API Key (simpler, recommended for WSL)
  2. Or run `npx codex login` from Windows side and copy `~/.codex/auth.json`
     to WSL home
- **Node/npm via Hermes-managed** — Hermes bundles its own Node at
  `~/.hermes/node/bin/node`. `npm install -g` installs into this Hermes Node
  prefix by default.
- **`pty=true` is mandatory** — Codex is an interactive terminal app; without
  PTY it hangs or produces no output.
- **⚠️ WSL Runtime Issues (v0.130.0, verified 2026-05-17):**
  - **Reconnecting errors**: Codex may print `ERROR: Reconnecting... 2/5` and
    fail to connect to OpenAI API even when OPENAI_API_KEY is valid. This
    appears to be a sandbox initialization issue on WSL.
  - **Sandbox (bwrap) missing**: WSL lacks kernel-level bubblewrap support.
    Codex prints `warning: Codex could not find bubblewrap on PATH` and falls
    back to bundled bubblewrap, which may malfunction.
  - **Stdin capture**: `echo '{}' | codex` triggers "Reading additional input
    from stdin..." and blocks the command — pipe-based invocation doesn't work.
    Always use `codex exec "prompt"` with a string argument.
  - **Model auto-selection**: Codex may auto-select a model (e.g. gpt-5.5)
    instead of the configured default. Explicitly pass `--model gpt-4o`.
  - **Timeout on first exec**: Codex's first call on a fresh git repo may
    exceed 60s timeout. Use `timeout=120` in terminal calls.

## One-Shot Tasks

```
terminal(command="codex exec 'Add dark mode toggle to settings'", workdir="~/project", pty=true)
```

For scratch work (Codex needs a git repo):
```
terminal(command="cd $(mktemp -d) && git init && codex exec 'Build a snake game in Python'", pty=true)
```

## Background Mode (Long Tasks)

```
# Start in background with PTY
terminal(command="codex exec --full-auto 'Refactor the auth module'", workdir="~/project", background=true, pty=true)
# Returns session_id

# Monitor progress
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")

# Send input if Codex asks a question
process(action="submit", session_id="<id>", data="yes")

# Kill if needed
process(action="kill", session_id="<id>")
```

## Key Flags

| Flag | Effect |
|------|--------|
| `exec "prompt"` | One-shot execution, exits when done |
| `--full-auto` | Sandboxed but auto-approves file changes in workspace |
| `--yolo` | No sandbox, no approvals (fastest, most dangerous) |

## PR Reviews

Clone to a temp directory for safe review:

```
terminal(command="REVIEW=$(mktemp -d) && git clone https://github.com/user/repo.git $REVIEW && cd $REVIEW && gh pr checkout 42 && codex review --base origin/main", pty=true)
```

## Parallel Issue Fixing with Worktrees

```
# Create worktrees
terminal(command="git worktree add -b fix/issue-78 /tmp/issue-78 main", workdir="~/project")
terminal(command="git worktree add -b fix/issue-99 /tmp/issue-99 main", workdir="~/project")

# Launch Codex in each
terminal(command="codex --yolo exec 'Fix issue #78: <description>. Commit when done.'", workdir="/tmp/issue-78", background=true, pty=true)
terminal(command="codex --yolo exec 'Fix issue #99: <description>. Commit when done.'", workdir="/tmp/issue-99", background=true, pty=true)

# Monitor
process(action="list")

# After completion, push and create PRs
terminal(command="cd /tmp/issue-78 && git push -u origin fix/issue-78")
terminal(command="gh pr create --repo user/repo --head fix/issue-78 --title 'fix: ...' --body '...'")

# Cleanup
terminal(command="git worktree remove /tmp/issue-78", workdir="~/project")
```

## Batch PR Reviews

```
# Fetch all PR refs
terminal(command="git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'", workdir="~/project")

# Review multiple PRs in parallel
terminal(command="codex exec 'Review PR #86. git diff origin/main...origin/pr/86'", workdir="~/project", background=true, pty=true)
terminal(command="codex exec 'Review PR #87. git diff origin/main...origin/pr/87'", workdir="~/project", background=true, pty=true)

# Post results
terminal(command="gh pr comment 86 --body '<review>'", workdir="~/project")
```

## Rules

1. **Always use `pty=true`** — Codex is an interactive terminal app and hangs without a PTY
2. **Git repo required** — Codex won't run outside a git directory. Use `mktemp -d && git init` for scratch
3. **Use `exec` for one-shots** — `codex exec "prompt"` runs and exits cleanly
4. **`--full-auto` for building** — auto-approves changes within the sandbox
5. **Background for long tasks** — use `background=true` and monitor with `process` tool
6. **Don't interfere** — monitor with `poll`/`log`, be patient with long-running tasks
7. **Parallel is fine** — run multiple Codex processes at once for batch work

## Hermes → Codex Delegation Workflow (with Quality Gates)

When Hermes delegates implementation to Codex, the flow is a 3-phase handshake, NOT a one-shot fire-and-forget:

```
Phase 1 — Hermes prepares                                           Codex role
  ├─ Write data contract file (JSON Schema: fields, types, defaults) │  (passive)
  ├─ Write TDD RED test files (pytest, failing until implemented)    │
  └─ Write context brief (project path, constraints, style rules)    │
                                                                     ▼
Phase 2 — Codex implements                                     Codex active
  ├─ Reads contract + tests + context                              │
  ├─ Implements until `pytest tests/ -q` passes (TDD GREEN)        │
  └─ Codex writes implement_v1.json with files_modified list       │
                                                                     ▼
Phase 3 — Hermes quality gate                                 Hermes active
  ├─ Runs `pytest tests/ -q` — all pass?                          │
  ├─ Runs `quality_gate.py .` — A-class defects = 0?              │
  ├─ Either: PASS → proceed to Step 5 (delivery)                   │
  └─ Or: FAIL → delegate back to Codex with specific fix scope     │
```

### Delegation Template

```python
# Hermes side — delegate to Codex with quality gate
result = delegate_task(
    goal=f"""
    1. Read data contract from: {contract_path}
    2. Read test files from: {test_path}  
    3. Implement feature until all tests pass
    4. Run: cd {workdir} && pytest tests/ -q (must show zeros)
    5. Output: implement_v1.json with files_modified + pass status
    """,
    context=f"""
    Project: {project}
    Constraints: Use codex --full-auto mode for speed
    Note: Fields must match contract schema EXACTLY — no extra fields, no renamed fields
    """,
    toolsets=['terminal', 'file']
)

# After delegate_task returns, Hermes runs quality gate
gate_result = terminal(f"python3 ~/.hermes/scripts/quality_gate.py {workdir}")
if 'A-class' in gate_result.output.lower():
    # Classify and delegate fix back
    delegate_task(goal=f"Fix the following A-class quality defects: {extract_defects(gate_result.output)}")
```

### Pitfalls

- **Don't dump full context** — pass file paths, not file contents. Codex reads files itself.
- **Don't skip quality gate** — Codex can pass its own tests but still have hardcoded data, URL drift, or missing persistence. The quality gate catches what TDD doesn't.
- **Use worktree for parallel fixes** — `git worktree add -b fix/n /tmp/fix-n main` isolates each Codex session.
- **Verify fix landing** — Codex may fix the wrong file (same function name in different file). Always check `files_modified` against the actual template/route used by users (3-line cross-ref: navigation link → route → template → modified file).
- **Don't dump all gate findings back to Codex** — quality gate may report hundreds of items (实战: 426项首次扫描). Hermes must triage: isolate A-class (new code) vs B-class (pre-existing) before delegating fixes. Dumping 400+ findings confuses Codex and wastes tokens. Triage logic: check if finding's file is in `files_modified` list from Codex's response.
- **⚠️ WSL reliability (v1.1.0):** Codex v0.130.0 has significant runtime issues
  on WSL Ubuntu 24.04. The `Reconnecting...` error pattern, missing bubblewrap
  sandbox, and model auto-selection problems make it unreliable as a daily
  coding sub-agent. **Prefer Aider (`pip install aider-chat`) on WSL** — it has
  no sandbox dependency, supports more providers (including DeepSeek), and
  responds instantly without the ~20s Codex initialization overhead. Aider
  also supports `--model` with local Ollama models, making it a better fit for
  this environment's hybrid cloud+local strategy.
- **Don't pipe stdin to `codex`** — `echo 'prompt' | codex` triggers
  "Reading additional input from stdin..." and blocks. Always use
  `codex exec "prompt"` with a string argument.

## Linked Files

| File | Content |
|------|---------|
| `references/hermes-codex-delegation-workflow.md` | Detailed Hermes→Codex delegation pattern with JSON contract example, full delegate_task template, and worktree parallel fix example |
