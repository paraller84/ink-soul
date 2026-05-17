---
name: opencode
description: "Delegate coding to OpenCode CLI (features, PR review)."
version: 1.2.0
author: Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Coding-Agent, OpenCode, Autonomous, Refactoring, Code-Review]
    related_skills: [claude-code, codex, hermes-agent]
---

# OpenCode CLI

Use [OpenCode](https://opencode.ai) as an autonomous coding worker orchestrated by Hermes terminal/process tools. OpenCode is a provider-agnostic, open-source AI coding agent with a TUI and CLI.

## When to Use

- User explicitly asks to use OpenCode
- You want an external coding agent to implement/refactor/review code
- You need long-running coding sessions with progress checks
- You want parallel task execution in isolated workdirs/worktrees

## Prerequisites

- OpenCode installed: `npm i -g opencode-ai@latest` or `brew install anomalyco/tap/opencode`
- Auth configured (see "Custom Provider Setup" below for non-OpenAI providers)
- Verify: `opencode providers list` should show available providers
- Git repository for code tasks (recommended)
- `pty=true` for interactive TUI sessions

## Custom Provider Setup (Non-OpenAI with OpenAI-Compatible API)

OpenCode has a **hardcoded model list** — it only sends known model names (like `gpt-4o-mini`, `claude-sonnet-4`) to the API endpoint. It does NOT dynamically discover models from the provider. This causes 400 errors when using providers that require specific model names.

**The workaround**: Run a model-name rewrite proxy between OpenCode and the provider. The proxy intercepts API requests, rewrites the `model` field from OpenCode's hardcoded name to the provider's actual model name, then forwards the request.

**Known limitation**: OpenCode validates model names before sending requests. With `control_account` overrides, `-m openai/gpt-4o-mini` still fails model validation. Set `OPENAI_API_KEY` and `OPENAI_BASE_URL` environment variables to bypass this validation entirely.

### Setup Steps (DeepSeek Example)

The proxy script lives at: `~/.hermes/skills/autonomous-ai-agents/opencode/scripts/ds-proxy.py`

```python
# Step 1: Start the proxy (see scripts/ds-proxy.py in this skill for the full file)
# Key design decisions:
#   - TARGET = "https://api.deepseek.com" (NO /v1 suffix — self.path includes /v1/)
#   - Separate do_GET returns /v1/models for model discovery
#   - do_POST rewrites model field to deepseek-v4-flash
#   - SO_REUSEADDR prevents port exhaustion on restart
```

```bash
# Step 2: Start the proxy
python3 ~/.hermes/skills/autonomous-ai-agents/opencode/scripts/ds-proxy.py 8099 &

# Step 3: Write provider credentials to OpenCode's provider database (needed for auth, but NOT for model routing)
python3 -c "
import sqlite3, time, os, json
auth_path = os.path.expanduser('~/.hermes/auth.json')
with open(auth_path) as f:
    key = json.load(f)['credential_pool']['deepseek'][0]['access_token']
db = os.path.expanduser('~/.local/share/opencode/opencode.db')
conn = sqlite3.connect(db)
now = int(time.time() * 1000)
conn.execute('INSERT OR REPLACE INTO control_account VALUES (?,?,?,?,?,?,?,?)',
    ('deepseek', 'deepseek', 'http://127.0.0.1:8099', key, '', 9999999999999, 1, now, now))
conn.commit()
"
```

```bash
# Step 4: Set env vars and verify
OPENAI_API_KEY="$(python3 -c "
import json
print(json.load(open('$HOME/.hermes/auth.json'))['credential_pool']['deepseek'][0]['access_token'])
")" \
OPENAI_BASE_URL="http://127.0.0.1:8099" \
opencode run --model 'openai/gpt-4o-mini' "Respond with exactly: DEEPSEEK_OK"
```

> **Note**: The `OPENAI_API_KEY` + `OPENAI_BASE_URL` env var override is required because OpenCode's hardcoded model list will reject `openai/gpt-4o-mini` when routed through `control_account` alone. The env vars bypass model validation entirely.

### Troubleshooting Custom Provider Setup

| Symptom | Cause | Fix |
|---------|-------|-----|
| `400 - model not supported` | OpenCode bypassed proxy | Check `OPENAI_BASE_URL` is set to proxy address |
| `Model not found: openai/gpt-4o-mini` | OpenCode's built-in model list rejected the combo | Set `OPENAI_API_KEY` + `OPENAI_BASE_URL` env vars |
| `Model not found: X/.` | Model name lacks `provider/` prefix | Use `--model openai/gpt-4o-mini` format |
| `Connection refused` | Proxy not running | Start proxy first |
| `Address already in use` | Old proxy holding port | `fuser -k 8099/tcp; sleep 1` then retry |
| `502 Bad Gateway` | Proxy can't reach upstream | Check API key validity / network |

### OpenCode Provider Database Schema

| Table | Purpose | Key Columns |
|-------|---------|-------------|
| `control_account` | Provider API keys & endpoints | `email` (provider name), `url` (API base URL), `access_token`, `token_expiry` |
| `account` | OpenCode cloud accounts (not AI providers) | `id`, `email`, `access_token` |

**Key insight**: The `control_account` table stores AI provider credentials. The `url` field is the API base URL — set this to your proxy URL for custom providers. The `email` field is a user-chosen provider identifier.

## Binary Resolution (Important)

Shell environments may resolve different OpenCode binaries. If behavior differs between your terminal and Hermes, check:

```
terminal(command="which -a opencode")
terminal(command="opencode --version")
```

If needed, pin an explicit binary path:

```
terminal(command="$HOME/.opencode/bin/opencode run '...'", workdir="~/project", pty=true)
```

## One-Shot Tasks

Use `opencode run` for bounded, non-interactive tasks:

```
terminal(command="opencode run 'Add retry logic to API calls and update tests'", workdir="~/project")
```

Attach context files with `-f`:

```
terminal(command="opencode run 'Review this config for security issues' -f config.yaml -f .env.example", workdir="~/project")
```

Show model thinking with `--thinking`:

```
terminal(command="opencode run 'Debug why tests fail in CI' --thinking", workdir="~/project")
```

Force a specific model:

```
terminal(command="opencode run 'Refactor auth module' --model openrouter/anthropic/claude-sonnet-4", workdir="~/project")
```

## Interactive Sessions (Background)

For iterative work requiring multiple exchanges, start the TUI in background:

```
terminal(command="opencode", workdir="~/project", background=true, pty=true)
# Returns session_id

# Send a prompt
process(action="submit", session_id="<id>", data="Implement OAuth refresh flow and add tests")

# Monitor progress
process(action="poll", session_id="<id>")
process(action="log", session_id="<id>")

# Send follow-up input
process(action="submit", session_id="<id>", data="Now add error handling for token expiry")

# Exit cleanly — Ctrl+C
process(action="write", session_id="<id>", data="\x03")
# Or just kill the process
process(action="kill", session_id="<id>")
```

**Important:** Do NOT use `/exit` — it is not a valid OpenCode command and will open an agent selector dialog instead. Use Ctrl+C (`\x03`) or `process(action="kill")` to exit.

### TUI Keybindings

| Key | Action |
|-----|--------|
| `Enter` | Submit message (press twice if needed) |
| `Tab` | Switch between agents (build/plan) |
| `Ctrl+P` | Open command palette |
| `Ctrl+X L` | Switch session |
| `Ctrl+X M` | Switch model |
| `Ctrl+X N` | New session |
| `Ctrl+X E` | Open editor |
| `Ctrl+C` | Exit OpenCode |

### Resuming Sessions

After exiting, OpenCode prints a session ID. Resume with:

```
terminal(command="opencode -c", workdir="~/project", background=true, pty=true)  # Continue last session
terminal(command="opencode -s ses_abc123", workdir="~/project", background=true, pty=true)  # Specific session
```

## Common Flags

| Flag | Use |
|------|-----|
| `run 'prompt'` | One-shot execution and exit |
| `--continue` / `-c` | Continue the last OpenCode session |
| `--session <id>` / `-s` | Continue a specific session |
| `--agent <name>` | Choose OpenCode agent (build or plan) |
| `--model provider/model` | Force specific model |
| `--format json` | Machine-readable output/events |
| `--file <path>` / `-f` | Attach file(s) to the message |
| `--thinking` | Show model thinking blocks |
| `--variant <level>` | Reasoning effort (high, max, minimal) |
| `--title <name>` | Name the session |
| `--attach <url>` | Connect to a running opencode server |

## Procedure

1. Verify tool readiness:
   - `terminal(command="opencode --version")`
   - `terminal(command="opencode auth list")`
2. For bounded tasks, use `opencode run '...'` (no pty needed).
3. For iterative tasks, start `opencode` with `background=true, pty=true`.
4. Monitor long tasks with `process(action="poll"|"log")`.
5. If OpenCode asks for input, respond via `process(action="submit", ...)`.
6. Exit with `process(action="write", data="\x03")` or `process(action="kill")`.
7. Summarize file changes, test results, and next steps back to user.

## PR Review Workflow

OpenCode has a built-in PR command:

```
terminal(command="opencode pr 42", workdir="~/project", pty=true)
```

Or review in a temporary clone for isolation:

```
terminal(command="REVIEW=$(mktemp -d) && git clone https://github.com/user/repo.git $REVIEW && cd $REVIEW && opencode run 'Review this PR vs main. Report bugs, security risks, test gaps, and style issues.' -f $(git diff origin/main --name-only | head -20 | tr '\n' ' ')", pty=true)
```

## Parallel Work Pattern

Use separate workdirs/worktrees to avoid collisions:

```
terminal(command="opencode run 'Fix issue #101 and commit'", workdir="/tmp/issue-101", background=true, pty=true)
terminal(command="opencode run 'Add parser regression tests and commit'", workdir="/tmp/issue-102", background=true, pty=true)
process(action="list")
```

## Session & Cost Management

List past sessions:

```
terminal(command="opencode session list")
```

Check token usage and costs:

```
terminal(command="opencode stats")
terminal(command="opencode stats --days 7 --models anthropic/claude-sonnet-4")
```

## Pitfalls

- **Hardcoded model list**: OpenCode only sends known model names (e.g., `gpt-4o-mini`, `claude-sonnet-4`) to the API. It does NOT dynamically discover models. Use env var override (`OPENAI_API_KEY` + `OPENAI_BASE_URL`) for custom providers.
- **Model name validation**: Even with `control_account` URL override, `-m provider/model` may fail because models are not registered under providers. `OPENAI_API_KEY` + `OPENAI_BASE_URL` env vars bypass this.
- **Provider name `deepseek/` is recognized** by OpenCode as a valid provider, but no models are registered under it.
- Interactive `opencode` (TUI) sessions require `pty=true`. The `opencode run` command does NOT need pty.
- `/exit` is NOT a valid command — it opens an agent selector. Use Ctrl+C to exit the TUI.
- PATH mismatch can select the wrong OpenCode binary/model config.
- If OpenCode appears stuck, inspect logs before killing:
  - `process(action="log", session_id="<id>")`
- Avoid sharing one working directory across parallel OpenCode sessions.
- Enter may need to be pressed twice to submit in the TUI (once to finalize text, once to send).

## Verification

Smoke test:

```
terminal(command="opencode run 'Respond with exactly: OPENCODE_SMOKE_OK'")
```

Success criteria:
- Output includes `OPENCODE_SMOKE_OK`
- Command exits without provider/model errors
- For code tasks: expected files changed and tests pass

## Rules

1. Prefer `opencode run` for one-shot automation — it's simpler and doesn't need pty.
2. Use interactive background mode only when iteration is needed.
3. Always scope OpenCode sessions to a single repo/workdir.
4. For long tasks, provide progress updates from `process` logs.
5. Report concrete outcomes (files changed, tests, remaining risks).
6. Exit interactive sessions with Ctrl+C or kill, never `/exit`.