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

## Non-Interactive Use (Hermes Delegation)

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
