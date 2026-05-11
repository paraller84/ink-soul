---
name: flask-service-onboarding
category: devops
description: >-
  Systematic process for onboarding a pre-built Flask web service on WSL — taking a
  code-complete but never-started application from zero to verified running, then
  hardening it with management scripts, WSL auto-start, periodic Feishu content sync,
  cron job registration, and capability registry updates.
version: 1.8.0
---

# Flask Service Onboarding

## When to Use

Use this skill when a Flask web service has **code written but never started**, was **interrupted during startup**, is **running but returning errors/timeouts**, or is **running but needs WSL-hardening** (auto-start, cron automation, Feishu sync).

Typical triggers:
- User says "check the status of [service]" and it's not running
- User says "1" / "start it" / "resume from where we left off" for a pre-built service
- User says "continue" after the service is running and expects automation (auto-start, cron)
- Found a `~/edu-hub/<project>/` directory with `app.py`, templates, and data files but no running process
- Session history shows code was built but startup was interrupted (port conflict, timeout, etc.)
- **User reports "can't access" a URL that points to a known service — process exists but requests fail or hang**

## Related: Creating a New Subject Module (vs. Onboarding a Standalone Service)

If you need to **create a new subject module within the Edu-Hub unified app** (e.g., Chinese, Science), not onboard a standalone service: see `references/edu-hub-new-subject-module.md`. This covers cloning the English/C008 subject pattern — Blueprint registration in `app.py`, data model design, parser + state manager creation, and portal integration.

**Key distinction:** Standalone services get their own port and process. Subject modules get registered as Blueprints in the shared port-5002 app, sharing auth, database, and portal infrastructure.

### Related: Authentication Pattern

If the service has a parent login/setup flow using local JSON credential storage, see `references/auth-json-credential-pattern.md`. This covers the common "setup-then-login inconsistency" bug where the setup endpoint stores the password but not the username, and the authenticate function uses a hardcoded constant.

**Audit requirement (important!):** After creating a new subject module by referencing an existing one, do NOT skip Step 10 (Cross-Reference Audit) in that reference doc. The single most common mistake is copying the *concept* of a feature (e.g., "word-bank") without adapting it for the new subject's data model — which results in the new module's route silently falling through to the reference module's implementation.

### Related: Template JS Debugging

When the running service serves stale or broken JavaScript on template-rendered pages, see `references/jinja2-template-js-debugging.md`. Covers Jinja2 template vs Flask debug mode cache interaction, JS syntax verification with Node.js, and common Jinja2→JS escaping pitfalls (backslash chains, undefined fallback functions).

### Related: Web Speech API — iOS Speak Fix

When an educational page has a **「听发音」** button that works on desktop but is silent on iPad/iPhone, see `references/web-speech-api-ios-fix.md`. Covers the `onvoiceschanged` async loading pattern, `doSpeak()` fallback, en-US voice selection, and the iOS Safari quirks that cause silent first-speak.

### Related: Adding a Grammar Practice Module

When adding a **knowledge-points-based practice module** (grammar, vocabulary builders, etc.) with a fundamentally different learning model (mastery percentage vs SM-2), see `references/grammar-practice-module-pattern.md`. Covers: percentage-based mastery model, cloud LLM exam analysis pipeline, DOCX file parsing with stdlib, dual parent/student entry points, and integration with coins/auth/grade systems. Pattern demonstrated in C008 Grammar Module v1.0.

### Related: DOCX Parsing with stdlib Only

When you need to extract text from .docx files but python-docx/lxml cannot be installed, see `references/docx-parsing-stdlib.md`. Uses `zipfile` + `xml.etree.ElementTree` (stdlib only). Pattern demonstrated in C008 Grammar Module file upload.

### Related: Multi-Source Frontend Data Aggregation

When a single parent page needs to display data from **multiple Flask Blueprints** with different auth requirements and API formats (e.g., batch content from `english_bp`, generated exams from `eng_v2`, grammar analyses from `grammar_bp`), see `references/multi-source-frontend-aggregation.md`. Covers: multiple `load*()` functions with `Promise.all`, adding JSON API endpoints alongside page routes, confirm-and-redirect pattern after save, and auto-expand for collapsible content cards.

### Related: Content-Hash Dedup for Analysis Pipelines

When adding a **duplicate content detection** layer to any import/analysis pipeline (exam papers, documents, forms), see `references/content-hash-dedup-pipeline.md`. Covers SHA256 normalization, thread-safe check-before-LLM, backward compatibility with old records, and route-level flash message handling. Pattern demonstrated in C008 Grammar Module exam dedup.

### Related: Question Grading Field Routing

When a unified practice grading function (`_v4_grade_question`) handles multiple question types where each type stores its correct answer in a **different field** (`word` for spelling, `answer` for en2cn, `correct_answer` for multiple choice), see `references/question-grading-field-routing.md`. Covers: per-type field routing, frontend 'I don't know' button display fix, verification checklist for adding new question types, and `data.expected` API response field priority. Pattern demonstrated in C008 v4 en2cn grading fix (2026-05-06).

### Related: Flashcard / Memorization Mode

When adding a **flashcard (背题) mode** to an existing single-page quiz application — where students browse questions and answers together before testing — see `references/flashcard-mode-pattern.md`. Covers: independent storage, 3-level marking, "reveal on click" interaction, auto-advance, retry-forgotten flow, and the complete CSS/JS/HTML implementation pattern. Demonstrated in Aerospace Quiz flashcard mode (2026-05-06).

### Related: Multi-Blueprint Debugging Checklist

When parent records show different statistics than the practice engine, or practice repeatedly shows the same words — the likely cause is two JSON stores tracking the same concept (word mastery) via different code paths. See `references/dual-data-source-consistency.md` for detection checklist, fix patterns, and verification steps. Demonstrated in C008 v4 persistence + SM-2 progress fix (2026-05-06).

### Related: Per-Day Practice Metrics (Duration + Wrong Answers)

When you need to add **duration tracking** (how long each day's practice took) and **wrong-answer persistence** (click a day to see what was wrong per item) to an existing educational service, see `references/per-day-practice-metrics.md`. Covers: two-file persistence design (checkin.json + wrong_answers_daily.json), session `started_at` → duration calculation, multiple-sessions-per-day handling (MAX not SUM), wrong answer extraction from `sess["answers"]`, per-item detail API endpoint, and template modal overlay. Pattern demonstrated in C008 English system v3.7+ (2026-05-08).

### Related: Cross-Reference Consistency Audit

When **bug fixes have been applied** but the user reports they didn't take effect — or you need to verify which page users actually navigate to vs which page you fixed — see `references/cross-reference-consistency-audit.md`. Covers 4-layer route-template-navlink-userPath cross-reference, detection of dead routes, misdirected fixes, and orphaned code. Pattern demonstrated in C008 English module post-fix audit (2026-05-06).

### Related: Adding a Token Economy / Gamification Module

When adding a **coins/stars/points system** with JSON file persistence, see `references/json-micro-economy-pattern.md`. Covers the full pattern: account lazy-creation, transaction log FIFO, daily limits, level progression auto-detection, one-time milestone bonuses, and graceful try/except integration with existing practice flows. Pattern demonstrated in C008 v3.5 coins_manager.py.

### Related: Standalone Quiz Tool Pattern

When you need to build a **self-contained single-page quiz application** with embedded question data, multiple quiz modes (random/achievement/exam/wrong-review), localStorage persistence, and no server-side database — see `references/standalone-quiz-tool-pattern.md`. Covers: template structure, quiz mode filter functions, multi-choice grading, timer management, resume detection, dark theme CSS, and Flask Blueprint integration. Pattern demonstrated in C008 Aerospace Quiz tool (500 questions, 2026-05-07).

### Related: Educational Module Question Testing

When you need to **systematically verify every question type** in an educational module — grammar choice/fill_blank/conversion/error_correction, vocabulary spelling/en2cn/fill_first_letter, dictation — see `references/educational-module-testing.md`. Covers: 3-layer test architecture (direct engine + Flask test client + HTTP), lifecycle verification (start → submit → complete), question type completeness matrix, and common bugs detected by the method. Pattern demonstrated in English module full test (2026-05-07).

### Related: Vocabulary Mastery Viewer (Word Bank)

When adding a **read-only vocabulary overview page** showing all words/phrases/sentences with per-item mastery status, see `references/word-bank-pattern.md`. Covers data merging (content.json × SM-2 frequency), 4-state classification (new/learning/review/mastered), category + status filtering, search, paginated rendering, and the per-item error badge. Pattern demonstrated in C008 v3.5 word_bank.html + `/api/word-bank` endpoint.

## Prerequisites

- Flask app is in a known directory (typically `~/edu-hub/<project>/`)
- Service uses JSON files for data (no database)
- Service runs on a port (typically 5000)

## Step-by-Step Process

### Step 1: Quick State Survey

Before any action, survey the state:
```python
# 1. Check if process is running
ps aux | grep flask | grep -v grep
ss -tlnp | grep <port>

# 2. Check if code exists
ls <project-dir>/app.py
ls <project-dir>/templates/
ls <project-dir>/data/

# 3. Check last startup log
cat <project-dir>/data/server.log   # or find log files
```

### Step 2: Port & Process Check

- If port is free → proceed
- If port is occupied → identify the process. If it's the same service, it's already running (verify with `/health`). If it's another service, decide whether to use a different port or free it.
- If port shows nothing listening but `ps` shows a sleep/wait state → clean it up and restart

### Step 3: Dependency Verification

Check key imports used by `app.py`:
```bash
python3 -c "import flask; print(flask.__version__)"
python3 -c "import <custom_dep>; print(<custom_dep>.__version__)"
# For specific deps like eng-to-ipa, filelock, etc.
python3 -m pip show <package> 2>/dev/null || echo "NOT INSTALLED"
```

### Step 4: Data Integrity Check

Scan `data/content.json` and `data/state.json` for structural issues:
- **content.json**: Verify batch structure. Check sentences are properly split (not merged into one field). Validate en/cn fields.
- **state.json**: Verify exercise list exists. Check for orphan state entries (items not in content).
- **Recommended**: Spot-check a few items that look suspicious (e.g., sentences with embedded dashes or mixed content).

Fix data issues with `patch` tool. Follow the "only fix what's broken" principle — don't restructure data.

### Step 5: Clean Code Issues

Check for issues flagged by any existing `audit_report.json`:
- Unused imports: `import json`, `import os`, `abort` from Flask — remove if unused
- Missing global exception handler — low priority for onboarding, skip
- Hardcoded IPs — low priority, skip

Only fix P1 issues (things that would prevent the app from running correctly).

### Step 6: Start the Service

```bash
cd <project-dir>
# Use background process (NOT nohup/disown)
python3 app.py
# → Start with background=true in terminal tool
```

Poll the process to confirm it started:
```bash
sleep 2
curl -s http://localhost:<port>/health
```

### Step 7: Comprehensive Smoke Test

Run these tests in order. Each must pass before moving to the next.

**⚠️ Watch for stub endpoints:** If a response looks like `{"message": "待实现"}` or `{"message": "TODO"}`, the endpoint is a placeholder — it wasn't wired to actual business logic. This requires backend implementation, not just testing. Examples from real incidents: `POST /api/practice/start` returning only a session_id without generating questions, `GET /api/practice/current` returning a hardcoded stub.

```bash
echo "=== 1. /health ==="
curl -s http://localhost:<port>/health | python3 -m json.tool

echo "=== 2. Data endpoints ==="
curl -s http://localhost:<port>/api/batches
curl -s http://localhost:<port>/api/batches/<first_batch_id>
curl -s http://localhost:<port>/api/export-content

echo "=== 3. Core interaction flow ==="
# Start practice
curl -s -X POST http://localhost:<port>/api/start-practice \
  -H "Content-Type: application/json" \
  -d '{"batch_id":"<batch_id>","counts":{"words":2,"phrases":1,"sentences":1}}'

# Submit correct answer
curl -s -X POST http://localhost:<port>/api/submit-answer \
  -H "Content-Type: application/json" \
  -d '{"batch_id":"<batch_id>","category":"words","item_text":"<first_word>","user_answer":"<first_word>"}'

# Submit wrong answer
curl -s -X POST http://localhost:<port>/api/submit-answer \
  -H "Content-Type: application/json" \
  -d '{"batch_id":"<batch_id>","category":"words","item_text":"<second_word>","user_answer":"<wrong_answer>"}'

echo "=== 4. Records & progress ==="
curl -s http://localhost:<port>/api/progress
curl -s http://localhost:<port>/api/batches/<batch_id>/cards  # if cards endpoint exists

echo "=== 5. Frontend pages ==="
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/<page1>
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/<page2>
```

### Step 8: Troubleshoot Running-but-Broken Service

Use this section when the process exists and port is listening, but requests fail or hang.

#### 8a — Determine Error Type

Test both localhost and the external LAN IP in parallel to isolate the problem layer:

```bash
# Local access (tests Flask app logic, routes, templates)
curl -v --connect-timeout 5 --max-time 10 http://127.0.0.1:<port>/health
curl -v --connect-timeout 5 --max-time 10 http://127.0.0.1:<port>/v2/parent/

# External access (tests WSL networking)
curl -v --connect-timeout 5 --max-time 10 http://<lan-ip>:<port>/health
```

| Local response | External response | Likely cause |
|---------------|-------------------|--------------|
| 500 / error JSON | Timeout | Flask app error (routes, templates, imports) |
| 200 OK | Timeout | WSL networking issue (see 8g) |
| Timeout | Timeout | Process zombie / deadlock → kill & restart |
| 200 OK | 200 OK | Issue was transient / already fixed |

#### 8b — Verify Route Registration

The process may have started with incorrect code or partial imports. Compare running routes against expected:

```bash
# Check Flask URL map from the app module (not from the running process)
cd <project-dir> && python3 -c "
import sys; sys.path.insert(0, '.')
from app import app
for r in sorted(app.url_map.iter_rules(), key=lambda r: r.rule):
    print(f'{r.rule:50s} {list(r.methods)}')
"
```

If expected routes (e.g., `/v2/parent/`) appear in this dump but return 404 at runtime, the running process loaded different code. Kill and restart.

### Step 8c — Check Blueprint Template Resolution Order

Flask searches **app template folder first**, then Blueprint template folder. If both have `index.html`, the app's template wins silently. This causes Blueprint pages to render wrong content.

**Fix:** Rename Blueprint templates to avoid name collisions:
```python
# In routes.py — use a subfolder name
return render_template("v2/index.html")   # not "index.html"
# Then create: <blueprint-dir>/templates/v2/index.html
```

#### 8c-ii — Cross-Blueprint Template Name Collision

When **two different Blueprints** (e.g., `eng_v3` and `chinese_v3`) both have templates at the same relative path (e.g., `v3/base.html` and `v3/portal.html`), Flask's `DispatchingJinjaLoader` searches ALL Blueprint template folders in **registration order**. The Blueprint registered first wins — so the Chinese portal template `{% extends 'v3/base.html' %}` may resolve to the English `v3/base.html` (which has `{% block title %}英语学习{% endblock %}`).

**Symptoms:**
- A page renders with the **wrong content** (e.g., `/chinese/student/portal` shows `英语学习 - 学习门户` / English title)
- `curl` confirms the route hits the correct Blueprint handler, but the rendered HTML shows another Blueprint's content
- `/english/student/portal` works fine (its Blueprint is registered first)
- Only the **later-registered** Blueprint's pages are broken

**Root cause:** `DispatchingJinjaLoader` doesn't scope template resolution to the current Blueprint — it iterates all Blueprint loaders and returns the first match. Two Blueprints with the same subdirectory name (`v3/`) produce paths that collide.

**Diagnostic:**
```bash
# List all templates with the same relative path across Blueprints
find ~/edu-hub -path '*/templates/*' -name 'base.html' -o -name 'portal.html' | sort
```

**Fix — use a Blueprint-unique subdirectory prefix:**
```python
# ❌ Collides: both use "v3/"
render_template("v3/portal.html")      # eng_v3
render_template("v3/portal.html")      # chinese_v3 ← gets eng_v3's template

# ✅ No collision: unique prefix per Blueprint
render_template("eng_v3/portal.html")   # eng_v3: english/v3/templates/eng_v3/portal.html
render_template("chinese_v3/portal.html")  # chinese_v3: chinese/v3/templates/chinese_v3/portal.html
```

**Files to update:**
1. Move template files to new unique subdirectory
2. Update all `render_template()` calls in the Blueprint's `routes.py`
3. Update all `{% extends "..." %}` references in the templates themselves
4. **Always** clear `__pycache__` and restart after this change (trap #10)

#### 8d — Check for `@app.errorhandler(Exception)` Masking HTTP Errors

A broad exception handler at the app level catches all HTTPExceptions (including 404, 405) and turns them into 500 responses with JSON:

```python
# ❌ This catches werkzeug.exceptions.NotFound (404) → returns 500
@app.errorhandler(Exception)
def handle_exception(e):
    return jsonify({"error": "服务器内部错误", "detail": str(e)}), 500
```

**Diagnostic clue:** If curl returns HTTP 500 with `"detail":"404 Not Found..."` in JSON body, this handler is masking a 404. Either:
- Fix the route so it matches (check trailing slashes, blueprint prefixes)
- Or narrow the handler: `@app.errorhandler(500)` instead of `Exception`

#### 8e — Verify Import Path / sys.path for Blueprint Sub-modules

When a Blueprint sub-module (e.g., `content_engine.py`) does `from english.english_parser import ...`, it requires `english` to be discoverable as a top-level package. Flask apps often add the parent directory to sys.path inline:

```python
# in app.py — usually at the top
sys.path.insert(0, str(Path(__file__).parent.parent))
```

**Check:** From the project directory, verify the import chain works:
```bash
cd <project-dir> && python3 -c "
import sys
sys.path.insert(0, str(Path.cwd().parent))
from english.v2.routes import *
print('All imports OK')
" 2>&1
```

If this fails but `app.py` works when running directly, the sys.path manipulation in `app.py` is working at runtime but won't be visible via standalone import checks.

#### 8f — Process Zombie Detection

A process may show in `ps aux` and `ss -tlnp` but be completely unresponsive (hung on deadlock, blocked I/O, infinite loop in startup).

**Symptoms:**
- `curl` to localhost times out (even for `/health`)
- `strace` shows no syscall activity
- Process CPU usage is 0%

**Fix:** Kill and restart:
```bash
kill -9 <pid>
# Wait 2s for socket release
sleep 2
# Restart via background terminal
```

#### 8g — WSL External Access Diagnosis

If the service works on `127.0.0.1` but times out on the LAN IP:

**Find the actual WSL IP** (different from Windows host IP):
```bash
hostname -I
# Returns something like 172.19.x.x (WSL virtual network)
```

**The WSL IP vs Windows IP mismatch rule:**
- `localhost:<port>` — works from Windows browser (WSL auto-forwards)
- `172.19.x.x:<port>` — WSL vEthernet IP, usable from Windows
- `192.168.x.x:<port>` — **Windows LAN IP**, will NOT work unless port forwarding is set up

**To enable LAN IP access**, use the `wsl-port-forwarding` skill (covers portproxy, firewall rules, Mirrored mode, UAC tricks, and auto-start persistence):

```bash
# Quick check: audit existing port forwarding
bash ~/.hermes/skills/devops/wsl-port-forwarding/scripts/check-and-report-ports.sh <port>

# Or manually (must run as admin on Windows):
powershell
netsh interface portproxy add v4tov4 listenaddress=<windows-lan-ip> listenport=<port> connectaddress=<wsl-ip> connectport=<port>
```

Verify with:
```powershell
netsh interface portproxy show all
```

### Step 9: Post-Onboarding Automation (WSL)

After the service is verified running, harden it for WSL deployment:

#### 9a — Create Management Script

Create `~/.hermes/scripts/<project>-manager.sh` with start/stop/restart/status/logs subcommands:

```bash
SERVICE_DIR="$HOME/edu-hub/<project>"
LOG_FILE="$SERVICE_DIR/data/server.log"
PID_FILE="/tmp/<project>-flask.pid"
PORT=<port>

# Must handle:
# - PID detection via both pid file AND ss -tlnp port scan
# - status checks /health endpoint
# - graceful stop then force kill fallback
# - RestartSec delay
```

#### 9b — Configure WSL Auto-Start

WSL `systemd --user` services **do not work reliably** (exit code 216/GROUP due to cgroup constraints). Use `cron @reboot` instead:

```bash
(crontab -l 2>/dev/null | grep -v <project>; \
 echo "@reboot cd /home/openclaw/edu-hub/<project> && python3 app.py > data/server.log 2>&1") \
 | crontab -
```

Verify with: `crontab -l | grep @reboot`

#### 9c — Set Up Periodic Content Sync (if Feishu-connected)

If the service has a Feishu upload folder (like `英语/上传入口`), create a sync script:

1. Script path: `~/.hermes/scripts/<project>-feishu-sync.py`
2. **Must handle**: 
   - List folder docs via Feishu Drive API
   - Compare with state file (token + modified_time) to identify new/changed docs
   - Read doc content via `feishu_wiki_sync_mod.read_doc_content()`
   - Parse and import into service's data store
   - Move processed docs to backup folder via Feishu Move API
   - Track state in `<project>-feishu-sync-state.json`
3. Register a cron job via `cronjob` tool:
   - Schedule: `0 */2 * * *` (every 2 hours) or `0 6,18 * * *` (twice daily)
   - Script: `<project>-feishu-sync.py`
   - Deliver: `origin` (reports results)

#### 9d — Register Capability

Update the capability registry at `~/wiki/entities/capability-registry.md`:
- Register or update the service entry with status `active`
- Include all assets: scripts, templates, data files, crons, management script
- Add change log entry: `upgrade` type, description of what was set up

### Step 10: Update Memory

Update persistent memory:
- Replace any "planning" / "interrupted" / "code complete" entries with current running state
- Include: port, URL, data summary, verification status, cron jobs, auto-start method
- Keep it compact — this is operational context for future sessions

### Step 8h — LAN Access: EXTERNAL_URL Pattern

When a Flask app's HTML templates or redirects hardcode `localhost:<port>`, LAN users clicking those links will reach their own device's `localhost` — not the server. Fix this with a configurable `EXTERNAL_URL`:

```python
# In app.py, after app creation:
EXTERNAL_URL = os.environ.get("EXTERNAL_URL", "http://192.168.3.102")
```

**In the HTML template (triple-quoted string, not Jinja2 template):**

```python
# Use .replace() to avoid CSS curly-brace conflicts with .format()
LANDING_PAGE = """<a class="card" href="{EXTERNAL_URL}:5001/v2/">...</a>"""

@app.route("/")
def index():
    return LANDING_PAGE.replace("{EXTERNAL_URL}", EXTERNAL_URL)
```

**⚠️ Why .replace() not .format():** CSS rules use `{margin:0}` etc. — calling `.format()` on a string containing CSS will trigger a `KeyError` because those curly braces look like format placeholders. `.replace()` is safe because it only matches the exact literal `{EXTERNAL_URL}`.

**Files to check for hardcoded `localhost` or fixed IP (comprehensive sweep):**
```bash
grep -rn 'localhost:500[012]\|192\.168\.3\.102:500[012]' ~/edu-hub --include="*.py" --include="*.html"
```

The env var approach means IP changes require only: `export EXTERNAL_URL=http://new-ip` + restart — no code changes.

### Step 8i — .pyc Cache Trap: Code Changes Not Reflected After Restart

After patching `app.py`, old `.pyc` bytecode files can cause the running service to still serve the **old** code despite a restart.

**Symptoms:**
- `grep` confirms the new code is on disk
- `kill` + restart was done
- But `curl` response still shows old content (e.g., `localhost:5002` when code says `{EXTERNAL_URL}:5002`)

**Root cause:** Python compiler caches bytecode in `__pycache__/app.cpython-3xx.pyc`. If this cache predates your edit, old bytecode survives `kill -9` + restart unless the cache is removed.

**Fix — always clear __pycache__ before restarting after code changes:**
```bash
rm -rf ~/edu-hub/<project>/__pycache__/
rm -rf ~/edu-hub/<project>/v2/__pycache__/   # any blueprint subdirs too
```

### Step 8j — Old Process Kill Verification (crontab @reboot trap)

Services started via `cron @reboot` produce multiple layers:
```
/bin/sh -c cd /project && python3 app.py > log 2>&1    # shell wrapper (PID X)
└── python3 app.py                                       # actual process (PID Y)
```

**pkill -f "app.py" may not match the shell wrapper** because the `-f` pattern `app.py` matches the child process but the parent shell (`sh -c ... python3 app.py`) uses slightly different syntax. The parent `sh` process keeps holding the socket open.

**Correct kill sequence:**
```bash
# 1. Find ALL layers
ps aux | grep <port>
ss -tlnp | grep <port>      # shows which PID holds the socket

# 2. Kill by PID directly (not pkill)
kill -9 <actual-socket-holding-PID>

# 3. Verify port is free
ss -tlnp | grep <port> || echo "Port free"

# 4. Then remove __pycache__ and restart
```

**Proactive prevention:** Before restarting, verify the port is actually freed. Crontab @reboot processes are especially persistent because cron restarts them on the same port if the shell wrapper isn't killed.

### Step 0b – Verify Subsystem Scope Before Acting

When the user reports "X doesn't work in the Y module", **do not assume you know which subsystem they mean** — especially when there are multiple education subsystems (English/C008 grammar, Chinese/C011 handwriting, Math/C003). If you're mid-conversation and just fixed another issue, the user's next complaint may be about a **different subsystem**.

**Pitfall**: The user says "语法练习不显示题目" — you just fixed `/chinese/v2/` issues, so you assume it's about Chinese. But they were referring to **English grammar** (`/student/grammar/`), a completely different Blueprint.

**Prevention — before acting on any bug report:**
1. Pause and ask yourself: *Which subsystem does this bug belong to?* Map the symptom to the correct subsystem:
   - "语文/生字/古诗" → Chinese (C011)
   - "英语/语法/单词" → English grammar (C008)
   - "数学/应用题" → Math (C003)
2. **If ambiguous, ask the user for the exact URL or page name.** A one-line clarification saves 30+ minutes of fixing the wrong system.
3. **If the user just mentioned one subsystem and now mentions a different symptom, do NOT default to the previous subsystem.** The new bug may be in a wholly different subsystem.
4. Keep a mental checklist of recently active subsystems — this session had both Chinese v2.0 and English grammar active, making scope confusion especially likely.

### Step 0c – Template Context Variable Mismatch (Silent Empty Content)

When a route handler passes a single dict object to `render_template()` (e.g., `session=session_data`) but the template references data fields as **individual variables** (e.g., `{{ question_text }}`, `{{ kp_name }}`, `{{ current_index }}`), the page renders with **blank/empty content** instead of a Jinja2 error.

**Why it's silent**: Jinja2's `Undefined` object handles `{{ variable or 'default' }}` gracefully — it returns the default. But `{{ question_text }}` (without `or` fallback) evaluates to the empty string `''`. The page loads, navigation works, but the actual question/answer area is empty — matching symptoms like "不显示题目".

**Diagnosis pipeline — three checks for blank-content bugs:**

| Check | Command | What it reveals |
|-------|---------|-----------------|
| 1. Template scan | `grep -n '{{' template.html \| head -20` | List all expected context variables |
| 2. Route handler | Read the `render_template(...)` call | Which variables are actually passed |
| 3. Comparison | Diff the two lists | Variables in template but NOT in handler = blank content |

**Example from real incident (English grammar practice):**
```python
# ❌ Route passes one dict — template expects N individual variables
return render_template("practice.html", session=session_data)

# Template uses: {{ question_text }}, {{ kp_name }}, {{ options }}, {{ feedback }}
# Solution — extract from the dict and pass individually:
return render_template("practice.html",
    question_text=current_q.get("question_text", ""),
    kp_name=resolved_kp_name,
    options=formatted_options,
    feedback=built_feedback,
    session_id=session_id,
    current_index=current_index + 1,
    total_questions=len(questions),
    ...
)
```

**Where to look — common variable categories that get missed when wrapping in a `session` object:**
- Display fields: `question_text`, `question_id`, `kp_name`, `options`
- Progress indicators: `current_index`, `total_questions`
- Feedback/results: `is_complete`, `correct_count`, `wrong_count`, `accuracy`
- Navigation: `session_id`, `back_url`

**Prevention — after writing/fixing a route handler that calls `render_template()`:**
1. `grep -oP '\{\{[.\s]*?\w+' template.html | sort -u` — extract all template variable names
2. Check each extracted name appears in the `render_template()` call's keyword arguments
3. Pay special attention to templates that use `{{ var or 'default' }}` — the `or` mask hides the problem

### Step 8c-iii — API Stub Trap (New Blueprint with Incomplete Implementation)

When a new Blueprint or feature is created by **copying a template structure** (routes, templates, static files) but the **actual backend business logic** was never wired up, the API endpoints return stubs like:
```json
{"message": "待实现"}
{"message": "TODO"}
{"correct": true, "message": "待实现"}
```

**Why it happens**: The developer created the routing framework and frontend templates first (which work and render correctly), then planned to implement the backend APIs later but never did. The Blueprint appears complete from a file-count perspective but is non-functional.

**Diagnosis** — check API endpoints for stub responses:
```bash
# After Smoke Test Step 7 reveals unexpected behavior, inspect response body
curl -s http://localhost:5002/path/to/api/endpoint
# If you see "待实现" or "TODO" in the response, it's a stub
```

**Common stub locations in practice systems:**
| Endpoint | Stub symptom |
|----------|-------------|
| `POST /api/practice/start` | Creates session_id but no questions |
| `GET /api/practice/current` | Returns `{"message": "待实现"}` |
| `POST /api/practice/submit` | Returns `{"correct": true}` hardcoded |
| `GET /api/practice/info` | Endpoint entirely missing |

**Fix pattern — create a session manager:**
```python
# practice/session_manager.py — single file that wires up question generators
_SESSIONS: dict[str, dict] = {}

def create_session(batch_id, student_id):
    content = load_content()
    questions = generate_questions(batch)  # call existing generators
    _SESSIONS[session_id] = {"questions": questions, "current_idx": 0, ...}
    return {"session_id": session_id, "total": len(questions), "first_question": ...}

def get_current_question(session_id):
    sess = _SESSIONS[session_id]
    q = sess["questions"][sess["current_idx"]].copy()
    q.pop("answer")  # hide answer from student
    return q

def submit_answer(session_id, user_answer):
    q = _SESSIONS[session_id]["questions"][current_idx]
    correct = grade(q["answer"], user_answer)
    _SESSIONS[session_id]["current_idx"] += 1
    return {"correct": correct, "next": next_question or None, "completed": done}
```

Then wire up the stub endpoints in `routes.py`:
```python
from .practice.session_manager import create_session, get_current_question, submit_answer

@bp.route("/api/practice/start", methods=["POST"])
def api_practice_start():
    result = create_session(batch_id, student_id)
    return jsonify(result)

@bp.route("/api/practice/current", methods=["GET"])
def api_practice_current():
    result = get_current_question(session_id)
    return jsonify(result)

@bp.route("/api/practice/submit", methods=["POST"])
def api_practice_submit():
    result = submit_answer(session_id, user_answer)
    return jsonify(result)
```

## Known Traps

1. **Port 5000 conflicts**: If port 5000 is taken by another Flask app, either use `kill` on that process or change port in `app.run()`.
2. **No process but port in use**: Could be a stale socket from a killed process — wait 60s or use `sudo fuser -k <port>/tcp`.
3. **eng-to-ipa try/except**: This library uses a try/except pattern, so it won't error if missing — but phonetics will show `[音标待补充]`. Check by calling `api/batches/<id>/cards` and inspecting the `phonetic` field.
4. **Data files with merged content**: Especially `sentences` — if the source text used a format without proper line breaks, entries from one doc may merge into single records. Always spot-check sentences field.
5. **Flask dev server warning**: The "WARNING: This is a development server" message is normal for this context (LAN-only, family use). Ignore it.
6. **Restart after WSL shutdown**: The service won't survive WSL restart. Use `cron @reboot` (not systemd --user — that fails with status 216/GROUP on WSL2 due to cgroup constraints).
7. **Feishu API rate limits**: When creating sync scripts, avoid rapid-fire API calls. Space out token checks and document reads. The Feishu Drive API is reliable but not designed for bursty local scripts.
8. **Management script PID detection**: A Flask process started via `nohup` or `background=true` may not have a PID file. The manager script must fall back to `ss -tlnp` port search with `grep -oP 'pid=\\K[0-9]+'` to find the process by port.
9. **Cron @reboot WSL timing**: `@reboot` fires when cron starts, which happens after WSL boot. For services that depend on network (Feishu API), the network may not be ready — the Flask app's API will work on localhost immediately, but Feishu sync will fail on first run if boot takes longer. This is acceptable (the next scheduled run will pick up).
10. **.pyc cache survives restart**: Old `__pycache__/app.cpython-*.pyc` files cause stale code after code edits + restart. Always run `rm -rf __pycache__/` before restarting after patching. This applies to ALL subdirectories with blueprints too (e.g., `v2/__pycache__/`).
11. **Crontab @reboot processes are multi-layered**: pkill may miss the parent `sh -c` wrapper that holds the socket. Kill by PID from `ss -tlnp` output, not by pattern. Verify port is freed before restart.
12. **CSS curly braces break `.format()`**: When using triple-quoted HTML strings with CSS (like `{margin:0}`), `.format()` will throw `KeyError`. Use `.replace()` for placeholder substitution instead.
13. **WSL mirrored networking vs portproxy**: If WSL is in mirrored mode, do NOT add portproxy rules — they actively break LAN access by forwarding to non-existent old virtual IPs. Detect with: `ip addr | grep inet | grep -v 127.0.0.1` — if WSL IP matches Windows LAN IP, it's mirrored mode.
14. **Feature removal follows a 4-layer pattern**: When removing a feature (e.g., photo recognition), always clean in this order: (1) `__init__.py` imports → (2) `routes.py` endpoints → (3) templates/JS/CSS → (4) source file deletion. Stop at each layer to verify no residual errors before proceeding. Missing any layer = silent 500 errors or broken UI.
15. **Jinja2 inline JS onClick handler escaping trap**: Complex backslash escaping inside Jinja2 templates that generate HTML onclick handlers is error-prone. The template file's backslashes pass through Jinja2 unchanged and must be valid JS. A common pitfall: `\\''` in `onclick="loadDetail('' + id + '')"` renders to `onclick="loadDetail(\'id\')"` which is invalid JavaScript (bare `\'` outside a string context). **Safer pattern**: assign escaped values to JS variables first, then use simple string concatenation in the onclick:
    ```javascript
    var escaped = opt.replace(/'/g, "\\'");
    choiceHtml += '<button onclick="selectChoice(this, ' + "'" + escaped + "'" + ')">...</button>';
    ```
    This avoids ambiguous backslash sequences in the template file.
16. **Flask Jinja2 template cache with debug=False**: When `debug=False` (production), Jinja2 caches compiled templates in-memory. Unlike Python `.pyc` bytecode (trap #10), there is no on-disk cache to clean. The fix is ensuring the **old process is fully dead** before restarting:
    - Use `ss -tlnp | grep <port>` to find the PID holding the socket
    - `kill <PID>` directly (not `pkill` which may miss the shell wrapper)
    - Verify port is free before starting new process
    - If the manager script reports "already running" on the new process, the old one wasn't killed
17. **Verify all function references in JS fallback/error paths**: An undefined function call in a JS exception handler (e.g., calling `showPracticeDone()` instead of `completePractice()`) causes the entire `<script>` block to fail silently — the page stays stuck on "正在加载题目..." with no visible error. After editing templates with non-trivial JS, extract the rendered JS and run `node --check` to catch syntax errors. Then verify the app browser loads and functions correctly end-to-end.

18. **Undefined variable in route handler after refactoring**: When code is refactored — especially blocks that were moved into a called function (e.g., `pe.submit_answer()` already handles retry mode internally) — a leftover check that references a variable from the unreachable code path (`session_obj.get("mode")`) silently causes a 500 error because the variable was never defined in the current scope. **Prevention**: After removing a code block that defined a variable, audit ALL remaining references to that variable in the same function. **Diagnosis**: This error doesn't appear in compilation — it only triggers at runtime when the specific code path executes. Use Flask's test client to submit requests through the problematic endpoint and capture the traceback.

19. **Educational quiz page UX pattern (dictation/practice)**: Quiz pages should use a **manual-advance** pattern rather than auto-advance: (a) Answer submission hides the submit and skip buttons and shows &quot;下一题 →&quot;, (b) User manually clicks to advance, giving time to read feedback, (c) Wrong answers show the correct answer in the feedback, (d) An &quot;❓ 不知道&quot; (I don&#39;t know / skip) button reveals the correct answer immediately without submitting to the grading API — the answer is shown, the next button appears, and the API is never called for that attempt, (e) The last question&#39;s next button changes to &quot;查看成绩 →&quot;, (f) Both dictation and practice pages should share this same UX pattern, (g) The backend API must return `{&quot;correct&quot;: bool, &quot;expected&quot;: str, &quot;done&quot;: bool}` to support this, (h) A **back button** (`‹`) in the header navigates to the portal/home page with a `confirm(&#39;确定退出当前练习？&#39;)` dialog to prevent accidental exits, (i) A footer link &quot;← 返回学习门户&quot; provides an additional navigation path, (j) The three buttons (提交答案 / ❓不知道 / 下一题 →) are laid out in a single flex row, with only the applicable button(s) visible at each state — submit + skip before answering, next only after feedback is shown.

(k) **`answered` vs `hasResult` trap in quiz button logic**: When controlling button visibility in `showQuestion()`, NEVER use `answered` (whether an option has been selected, i.e., `qid in q.answers`) as the sole condition for hiding the submit button. Selecting an option sets `q.answers[qid] = [label]` IMMEDIATELY, which makes `answered` true BEFORE the user has clicked submit. This causes the submit button to disappear as soon as any option is clicked — the user can select but never submit:

```javascript
// ❌ BROKEN — submit button disappears as soon as an option is clicked
document.getElementById('submitBtn').style.display = (result === undefined && !answered) ? 'inline-block' : 'none';
//                              because answered=true (option selected) → !answered=false → display='none'

// ✅ CORRECT — only use hasResult (whether the answer was submitted/graded)
const hasResult = result !== undefined;
document.getElementById('submitBtn').style.display = hasResult ? 'none' : 'inline-block';
//                             hasResult=false (not yet submitted) → always visible regardless of selection
```

**All four buttons** should use `hasResult` consistently:
- `submitBtn`: visible when no result → `hasResult ? 'none' : 'inline-block'`
- `revealBtn` (查看答案): same condition as submitBtn
- `nextBtn`: visible only when has result AND not last question
- `finishBtn`: visible only when has result AND is last question

(l) **"查看答案" (reveal answer) button pattern**: In addition to the "❓不知道" skip button (which reveals the answer without API submission), provide a "👁 查看答案" button that behaves differently — it DOES count as wrong (submits to the grading/recording system) and shows the correct answer:

```javascript
function revealAnswer() {
  const q = state.currentQuiz;
  const question = q.questions[q.idx];
  const qid = question.id;
  if (q.results[qid] !== undefined) return;  // already answered

  q.answers[qid] = [];    // no answer selected
  q.results[qid] = false;  // mark as wrong
  q.score.wrong++;
  if (!state.wrongSet[qid]) state.wrongSet[qid] = 0;
  state.wrongSet[qid]++;

  saveState();
  showQuestion();  // re-renders with feedback + correct answer
}
```

**Style**: Use `.btn-warning` (orange/amber, not red or blue) to distinguish it from submit (primary) and exit (danger) buttons:
```css
.btn-warning { background:#e65100; color:#fff; }
.btn-warning:hover { background:#ff8f00; }
```

**UX comparison**:
| Button | Counts as wrong | Shows answer | Triggers API | Use case |
|--------|:---------------:|:------------:|:------------:|----------|
| ❓ 不知道 | No | Yes | No | Student genuinely doesn't know, wants to learn |  
| 👁 查看答案 | Yes | Yes | Yes | Student wants to see but pay the penalty (like peek at answer key) |

21. **API response wrapper extraction**: When an internal API function returns a uniform wrapper dict ({'found': bool, 'result': dict, 'error': str}) but route handlers treat the wrapper as the data object, the template/all callers get zero values and silent failures. This is a common design pattern in Python APIs that want to distinguish "found but empty" vs "not found", but routes MUST extract the inner dict:\n\n    ```python\n    # ❌ Broken: passes wrapper to template\n    analysis = ea.get_analysis(analysis_id)  # returns {'found': True, 'analysis': {...}}\n    return render_template('result.html', analysis=analysis)\n    #      Template sees: analysis.get('questions') → None\n\n    # ✅ Fixed: extract inner dict\n    analysis_result = ea.get_analysis(analysis_id)\n    if isinstance(analysis_result, dict) and analysis_result.get('found'):\n        analysis = analysis_result['analysis']\n    else:\n        analysis = None\n    return render_template('result.html', analysis=analysis)\n    ```\n\n    **Prevention:** When calling any internal API that returns a dict with keys like `found`, `success`, `data`, `result`, `error`, immediately check for the wrapper pattern and extract the inner payload.\n\n22. **Server-side data injection vs client-side fetch for auth-gated data**: When a page needs data from a Blueprint with `@require_role`, don't make the browser JS fetch from that Blueprint — the session cookie may not propagate cross-Blueprint. Instead, inject the data server-side:\n\n    ```python\n    # In the route handler (same Blueprint as the page template)\n    from other_blueprint import some_api\n    data = some_api()  # called server-side, no auth issue\n    return render_template('page.html', injected_data_json=data)\n    ```\n\n    ```html\n    <!-- In template: Jinja2 tojson + safe produces valid JS -->\n    <script>\n    var DATA = {{ injected_data_json | tojson | safe }};\n    // Use DATA directly, no fetch() needed\n    </script>\n    ```\n\n    This avoids race conditions, auth failures, and extra API calls. Trade-off: larger initial payload. Choose when the data is small (<100KB) and the user expects immediate rendering.

    ```javascript
    // ❌ Broken: plain HTML form POST shows JSON in browser
    <form method="POST" action="{{ url_for('bp.some_api') }}">
        <button type="submit">开始</button>
    </form>

    // ✅ Correct: JS intercepts, handles JSON, redirects
    <form class="js-submit">
        <button type="submit">开始</button>
    </form>
    <script>
    document.querySelector('.js-submit')?.addEventListener('submit', function(e) {
        e.preventDefault();
        fetch(this.action, {method:'POST', headers:{'Content-Type':'application/json'}, body:'{}'})
        .then(function(r) {
            if (!r.ok) return r.json().then(function(d) { throw new Error(d.error || '请求失败'); });
            return r.json();
        })
        .then(function(d) {
            var sid = d.session_id || (d.data && d.data.session_id);
            if (sid) window.location.href = '/practice/' + sid;
            else throw new Error('未获取到会话');
        })
        .catch(function(err) { alert(err.message); });
    });
    </script>
    ```

    **Event delegation approach**: If there are many forms (e.g., a grid of knowledge point cards), use event delegation on the parent container rather than per-form listeners:
    ```javascript
    document.querySelector('.kp-grid')?.addEventListener('submit', function(e) {
        if (e.target && e.target.matches('.kp-card')) {
            e.preventDefault();
            // ... fetch handling ...
        }
    });
    ```

    **Always test with real auth**: JSON form submission bugs only surface when a user with proper session/auth clicks the form. Anonymous curl tests or `require_role`-gated routes won't reproduce the issue without a valid session.

23. **JS `API_BASE = ''` in Blueprint templates causes silent 404s**: When a Flask Blueprint has a `url_prefix` (e.g., `/english/student`), and the template's JavaScript uses a hardcoded empty `API_BASE = ''` to build fetch URLs, requests go to root-relative paths (`/api/v4/status`) instead of prefixed paths (`/english/student/api/v4/status`). The `.catch(() => {})` in the JS silently swallows the 404, so the UI just shows `-` or "加载失败" with no console errors. **Same issue applies to `window.location.href` redirects** — hardcoded `/v4/practice/session?session=...` becomes dead 404 links. After fixing API_BASE, audit ALL hardcoded paths with `grep -rn "'/api/v4\\|\"'/v4/'\\|\"'/student/'" templates/` to catch both fetch URLs and redirect targets. **Diagnosis clue:** Status bar shows all fields as `-`, calendar shows "加载失败", but `curl` with the correct prefix returns valid data. **Fix — pass the API base from the Flask route:**

    ```python
    # In routes.py
    @bp.route("/portal")
    def portal():
        api_base = url_for("bp.portal").replace("/portal", "")
        return render_template("portal.html", api_base=api_base)
    ```

    ```javascript
    // In template — use the server-provided prefix, not hardcoded ''
    const API_BASE = '{{ api_base }}';
    fetch(API_BASE + '/api/v4/status')  // → /english/student/api/v4/status
    ```

    **Why `replace` not `rstrip`:** `str.rstrip("portal")` strips characters individually (trailing 'l', 'a', 'r', etc.) not the whole word. Use `str.replace("/portal", "")` for exact substring removal.

24. **Cross-Blueprint template path collision (v2 template overrides v3)**: When two Blueprints (`eng_v2` and `eng_v3`) have templates at the same relative path (e.g., both have `student-records/index.html`), Flask's `DispatchingJinjaLoader` searches Blueprint template folders in **registration order** and returns the first match. A route handler in `eng_v3` calling `render_template("student-records/index.html")` can resolve to `eng_v2/templates/student-records/index.html` if `eng_v2` was registered first. **Symptom:** Newly created template in v3's directory is never rendered — the page shows different HTML, layout, or API calls from a different Blueprint. **Fix — use Blueprint-unique template paths:**

    ```python
    # ❌ Collides — same relative path in two Blueprints
    render_template("student-records/index.html")

    # ✅ Unique — prefix with Blueprint-specific subdirectory
    render_template("v3/student_records.html")
    ```

    **Always** clear `__pycache__` and restart after renaming templates to avoid cached old resolution order.

27. **Question display fallback chain — verify the full data-to-eye path**: When a question or prompt text is displayed incorrectly (showing nothing, showing generic text, revealing the answer), the bug may span 4 layers:
    - **Data layer**: What is the original content (cn, en, category fields)?
    - **Backend layer**: How are questions generated? What fields are set (prompt, cn, word, en)?
    - **API layer**: What fallback chain is used for `question_text`? What happens when fields are empty?
    - **Frontend layer**: How does the template decide what to display? Are there condition branches that depend on data presence?
    
    **The most common cause of "no information" questions**: `_enrich_question()` overwrites `q["cn"]` with a fallback prompt (like "请输入单词的正确拼写"), making `data.cn` always truthy. The frontend template checks `if (data.cn)` and displays `data.cn` as the question text — but the fallback prompt contains no word information. The actual word is in `data.word` or `data.en`, which is never displayed because the `data.cn` check succeeds and bypasses the `data.word` fallback.
    
    **Fix chain (must fix all layers):**
    1. Backend: Keep original `cn` — don't overwrite with fallback prompt (`_enrich_question()`)
    2. API: Ensure `question_text` fallback chain includes `word` and `en` as final fallbacks
    3. Frontend: Handle empty `cn` gracefully — show `data.word` instead of generic text
    
    **Verification:** After fixing, trace one question through the full cycle: create a session :arrow_right: API response :arrow_right: rendered HTML. The user should always see either a Chinese translation OR the English word — never a generic instruction without context.

28. **English category/status names in Chinese UI context**: When a template constructs Chinese text by concatenating template variables (e.g., `'请输入' + data.category + '的正确拼写'`), any English field value like `"word"`, `"phrase"`, `"sentence"` will render directly as English in the Chinese sentence. The user sees `"请输入word的正确拼写"` instead of `"请输入单词的正确拼写"`.
    
    **Fix — always map English names to Chinese before using them in UI text:**
    ```javascript
    var catNames = {'word':'单词', 'phrase':'词组', 'sentence':'短句'};
    promptLabel.textContent = '请输入' + (catNames[data.category] || data.category || '单词') + '的正确拼写';
    ```
    
    **Audit:** Search for ALL concatenations that mix Chinese text with template variables in `.html` files:
    ```bash
    grep -rn "'请输入\|'请选择\|'请写出\|'请在" ~/edu-hub --include="*.html" | grep -v 'cn_instruction' | grep '+'
    ```
    
    **True, final root cause for persistent "请输入word" bugs in C008 (2026-05-06):** The original bug (prompt containing answer text) → fixed by removing {en} from fallback → exposed cn overwrite bug → fixed by keeping original cn → exposed missing category name mapping → fixed by adding catNames dict. 4 layers, 4 fixes, all necessary.

29. `str.rstrip()` vs `str.replace()` for URL prefix trimming`

30. **Unescaped single quotes inside single-quoted JS string — silent script failure**: When building HTML strings via JS string concatenation, a single quote (`'`) inside a single-quoted JS string terminates the string prematurely. The remaining code becomes syntax errors, and the ENTIRE `<script>` block fails to parse — no JS runs at all. The page renders as bare HTML (no interactive elements), with no visible error unless the browser console is opened.\n\n    **Classic example — building an onclick handler with an empty-string argument:**\n    ```javascript\n    // ❌ BROKEN: the first ' closes the outer string, remaining code is garbage\n    tags.innerHTML = '<button onclick=\"selectCategory('')\">全部</button>';\n    //                        The parser sees: string ends at ', then ' is a new empty string, ) is unexpected\n\n    // ✅ FIXED: escape single quotes with backslash\n    tags.innerHTML = '<button onclick=\"selectCategory(\\'\\')\">全部</button>';\n    ```\n\n    **Why the error is silent**: The JS engine doesn't throw a runtime error — it fails at **parse time**. The `<script>` tag's content is never evaluated, so no code runs at all. No init(), no event handlers, no error in console (some engines report a generic \"Unexpected string\" or empty error message).\n\n    **Diagnosis — quick check for dead `<script>` block:**\n    1. Test a simple global: type `typeof MODES` or `typeof ALL_QUESTIONS` in the browser console → returns `undefined` (script didn't load)\n    2. Check for DOM elements rendered by JS: are mode cards, dynamic content, or interactive elements missing?\n    3. Verify syntax with Node.js: `node -e \"new Function(scriptContent)\"` — catches parse-time errors the browser silently swallows\n    4. Look for this specific pattern: `onclick=\"functionName('')\"` — the double-single-quote `''` inside a single-quoted JS string.\n\n    **Prevention — when building HTML strings with event handlers in JS:**\n    - Prefer **backtick template literals** (`) over single quotes for multi-line HTML — backticks don't conflict with single quotes inside HTML attributes\n    - When using single quotes, escape ALL internal single quotes with `\\'`\n    - For empty-string arguments in onclick, use `\\'\\'` not `''`\n    - After any HTML-within-JS construction, run `node --check` on a generated script to verify syntax\n\n    **Related patterns that also cause silent parse failure:**\n    - Unmatched backticks in template literals\n    - Unclosed string in inline data (large JSON arrays embedded in `<script>`)\n    - JSX-like syntax in a non-transpiled script (angle brackets inside template literals confuse some parsers)\n\n31. **Dual-registration Blueprint trap — same module on two ports with different url_prefix**: When a Flask Blueprint (`eng_v3`) is registered in TWO separate apps — one standalone (no prefix, port 5000) and one unified (with prefix `/english/student`, port 5002) — hardcoded paths in templates silently break on the prefixed deployment. This is the **most common root cause of "I fixed it but it still doesn't work"** bugs in the Edu-Hub ecosystem.

    **The two-registration pattern in Edu-Hub:**
    ```python
    # english/app.py (standalone, port 5000) — NO prefix
    app.register_blueprint(eng_v3)
    # → /portal, /coins-rules, /api/v4/status

    # edu-hub/app.py (unified, port 5002) — WITH prefix
    app.register_blueprint(eng_v3, url_prefix="/english/student")
    # → /english/student/portal, /english/student/coins-rules
    ```

    **Symptom**: User says "问题依然存在" after you deployed a fix. You restart the right process, but the links/fetch URLs still don't work. The fix template works on port 5000 but not on port 5002.

    **Diagnosis**: Ask yourself: *Which app does the user actually access?* Don't default to the standalone app. Check running processes:
    ```bash
    ss -tlnp | grep -E '500[012]'
    ```
    If edu-hub (port 5002) is running, test with the `/english/student` prefix:
    ```bash
    curl -s http://localhost:5002/english/student/portal-v4 | head -5
    ```

    **Root cause — hardcoded paths in templates**: Template HTML elements (footer links, status bar `<a>` tags, JS `window.location.href` targets) use hardcoded absolute paths like `<a href="/coins-rules">`. These work in the standalone app (no prefix, path is literally `/coins-rules`) but fail in edu-hub (path should be `/english/student/coins-rules`).

    **Fix — always use `{{ url_for() }}` in Jinja2 templates, never hardcoded paths:**
    ```html
    <!-- ❌ Breaks under url_prefix -->
    <a href="/coins-rules">积分规则</a>
    <a href="/word-bank">我的词库</a>
    <a href="/student/grammar/">语法练习</a>

    <!-- ✅ Works in both standalone and prefixed deployments -->
    <a href="{{ url_for('eng_v3.coins_rules_page') }}">积分规则</a>
    <a href="{{ url_for('eng_v3.word_bank_page') }}">我的词库</a>
    <a href="{{ url_for('grammar_bp.grammar_portal') }}">语法练习</a>
    ```

    **Cross-Blueprint url_for works**: `url_for('grammar_bp.grammar_portal')` resolves correctly even though `grammar_portal` is in a different Blueprint. Flask resolves Blueprint names globally.

    **Same rule applies to JS redirects**: If you have `window.location.href = '/v4/practice/session?session=' + sid;`, it must be `window.location.href = API_BASE + '/v4/practice/session?session=' + sid;` where `API_BASE` comes from `{{ api_base }}` (set by the route handler using `url_for`).

    **Verification — test under BOTH registrations before declaring done:**
    ```python
    # Test standalone (mimics port 5000)
    from english.app import app as standalone_app
    with standalone_app.test_client() as c:
        # ... set up session, test routes

    # Test edu-hub (mimics port 5002)
    from edu_hub.app import app as hub_app
    with hub_app.test_client() as c:
        with c.session_transaction() as sess:
            sess[...session setup...]
        resp = c.get('/english/student/portal')
        assert '/english/student/coins-rules' in resp.data.decode()
    ```

    **Proactive prevention**: Before applying ANY template fix in the English/C008 module, decide which app to target:
    - If the user's URL contains `/english/`, they're using edu-hub (5002) → use `url_for()`
    - If the user's URL is just `:5000/`, they're using standalone → hardcoded paths also work, but `url_for()` is safer for future-proofing
    - **When in doubt, always use `url_for()`** — it works under both registrations.

    **Restart the right process**: Fixing templates and restarting the wrong port will not help. After changes:
    ```bash
    # Kill the correct process
    kill <edu-hub-pid>
    # Or
    kill <standalone-pid>
    # Then start the one the user actually uses
    ```
26. **Global question quality gate — validate at session entry, not at each generator**: When malformed questions (missing `word`, `en`, `cn`, or `question_text`) reach the frontend, the symptom is "请输入单词的正确拼写" with no actual word — a generic instruction without context. Fixing each generator individually is brittle because new generators or edge cases (grammar types `conversion`/`error_correction`, LLM-generated questions) introduce new failure modes.

    **Instead, add a single validation/repair gate where all questions enter the session store:** Validate by question type, repair cross-field (word↔en, prompt from question_text), reject unfixable ones, log warnings. Code pattern:
    ```python
    def _v4_validate_question(q, index):
        qtype = q.get("type", "")
        # Cross-field repair
        if not q.get("word") and q.get("en"): q["word"] = q["en"]
        if not q.get("en") and q.get("word"): q["en"] = q["word"]
        if not q.get("prompt") and q.get("cn") and q["cn"] != q.get("en",""):
            q["prompt"] = q["cn"]
        if not q.get("prompt") and q.get("question_text"):
            q["prompt"] = q["question_text"][:200]
        # Type-specific checks (return None = reject)
        if qtype in ("choice","multiple_choice"):
            if not q.get("options"): return None
        elif qtype == "fill_first_letter":
            if not q.get("word") and not q.get("en"): return None
            if not q.get("same_cn_words"): q["type"] = "spelling"
        elif qtype in ("spelling","en2cn"):
            if not q.get("word"): return None
        else:
            if not (q.get("question_text") or q.get("word") or q.get("en") or q.get("cn")):
                return None
        return q

    def _v4_create_session(questions, metadata):
        valid = []
        rejected = 0
        for i, q in enumerate(questions):
            fixed = _v4_validate_question(q, i)
            if fixed: valid.append(fixed)
            else: rejected += 1
        if not valid:
            valid.append({"type":"spelling","word":"placeholder",...})
        _V4_SESSIONS[sid] = {"questions": valid, ...}
    ```

    **Design principles: one gate, not many fixes; repair before reject; reject with logging; extreme fallback when all questions invalid.** Also applies to frontend: when adding a new question type (e.g., grammar `conversion`), add it to BOTH the validation function and the frontend rendering switch — not just the generator.

27. **Unified practice session: question type normalization across domain engines**: When a unified practice session template (e.g., `practice_session_v4.html`) handles question rendering, submission, and grading, it must normalize question types from all domain-specific engines. Grammar questions from `grammar.practice_engine.generate_question()` use `type: 'choice'`, but the unified session checks for `type: 'multiple_choice'`. This mismatch causes three silent failures: (a) choice options are hidden and the input field is shown instead, (b) submit button logic falls to default (reads text input instead of selected option), (c) grading falls to default (string comparison instead of option match).

    **Fix chain — audit all three layers when adding a new question type:**
    1. **Rendering** (`renderQuestion()` in template): Add `qType === 'choice'` to the choice condition, or normalize the `type` field server-side to a common vocabulary.
    2. **Submission** (`submitAnswer()` in template): Add the same condition to the answer-collection logic so options are read correctly.
    3. **Grading** (`_v4_grade_question()` in routes.py): Add the same condition so the correct option comparison is used.

    **Additionally**, ensure `question_text` is always included in the API response. Grammar questions carry their text in `question_text` or `original_text`, which must be forwarded in the response payload — it was missing in the `api_v4_practice_current()` response, causing blank content even for `fill_blank` type questions.

    **Diagnosis clue:** When a specialized/grammar practice session loads but shows only the prompt text ("请在空白处填入正确答案" or "请输入正确答案") with no actual question content, check two things: (a) Is `question_text` present in the API response (line ~1310 of routes.py)? (b) Is the question `type` value (e.g., `"choice"`) handled by the template's `renderQuestion()` conditions?

    **Prevention — when adding a new practice engine (grammar, math, science) to a unified session:** Define a `QUESTION_TYPE_VOCABULARY` map at the top of the unified engine that translates domain-specific types to the unified session's expected types:
    ```python
    # unified_practice.py
    TYPE_NORMALIZATION = {
        "choice": "multiple_choice",       # grammar → unified
        "fill_blank": "fill_blank",         # grammar → unified (already exists)
        "spelling": "spelling",             # word engine → unified (already exists)
        "en2cn": "en2cn",                  # word engine → unified (already exists)
        "fill_first_letter": "fill_first_letter",  # word engine → unified
    }
    # In the API response:
    resp[&quot;type&quot;] = TYPE_NORMALIZATION.get(q.get(&quot;type&quot;), q.get(&quot;type&quot;, &quot;spelling&quot;))
    ```

    **Additionally, the `fill_first_letter` question type** (generated for words with multiple English synonyms sharing the same Chinese meaning, like 飞机/plane/aeroplane) must NOT pre-fill the first letter in the input field. Setting `value=&quot;{{ first_letter }}&quot;` effectively gives away the answer. Instead, use the placeholder attribute to show the first letter as a hint, and leave the value empty:
    ```javascript
    // ❌ Wrong: pre-fills first letter, revealing too much
    html += '&lt;input value=&quot;' + fl + '&quot;&gt;';

    // ✅ Right: shows first letter as placeholder hint only
    html += '&lt;input placeholder=&quot;' + fl + '&hellip;&quot; value=&quot;&quot;&gt;';
    ```

    **Also ensure the unified practice session template always includes:** a back button with confirmation dialog in the header, a &quot;不知道&quot; skip button that reveals the answer without API submission, a footer link returning to the portal, and progress counter + progress bar. These are non-negotiable UX elements for any educational practice page.

32. **Session default student ID vs student management ID mismatch (coins & data modification)**: When modifying student-specific data (coins, progress, records), the student ID used by the practice/coins layer (`session.get("student_id", DEFAULT_STUDENT)`) may differ from the ID in `v2/students.json`. C008 English system has two ID systems:

    - **v2 student management** (`v2/students.json`): Uses custom IDs like `stu_7fcee189` for 叶宇凡
    - **v3/v4 practice + coins layer** (`v3/coins_manager.py`, `v3/data_manager.py`): Uses `DEFAULT_STUDENT = "stu_default"` as fallback when no session cookie is present

    **The `_get_student()` resolution chain:** `@require_role("student")` → `session.get("student_id", dm.DEFAULT_STUDENT)`. If the user hasn't established a session (no student portal login), `_get_student()` returns `"stu_default"`.

    **Fix pattern — always trace the actual student ID before modifying data:**

    ```bash
    # 1. Check which student the system actually uses at runtime
    grep -n "DEFAULT_STUDENT\|_get_student\|session.get.*student_id" v3/routes.py | head -5

    # 2. Check the coins file for the DEFAULT student, not the v2/students.json ID
    ls data/coins/stu_default.json   # ← this is what the API reads
    ls data/coins/stu_7fcee189.json   # ← this is what v2 thinks the student is
    ```

    **Prevention:** Before creating/modifying any student-specific data file (coins, progress, state), read `_get_student()` from the routes module (or check `data_manager.DEFAULT_STUDENT`) to determine the actual active student ID. Do NOT take the student ID from `v2/students.json` — that's the parent-facing management layer, not the session layer.
