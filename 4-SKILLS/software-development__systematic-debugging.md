---
name: systematic-debugging
description: "4-phase root cause debugging: understand bugs before fixing."
version: 1.2.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
metadata:
  hermes:
    tags: [debugging, troubleshooting, problem-solving, root-cause, investigation]
    related_skills: [test-driven-development, writing-plans, subagent-driven-development]
---

# Systematic Debugging

## Overview

Random fixes waste time and create new bugs. Quick patches mask underlying issues.

**Core principle:** ALWAYS find root cause before attempting fixes. Symptom fixes are failure.

**Violating the letter of this process is violating the spirit of debugging.**

## The Iron Law

```
NO FIXES WITHOUT ROOT CAUSE INVESTIGATION FIRST
```

If you haven't completed Phase 1, you cannot propose fixes.

## When to Use

Use for ANY technical issue:
- Test failures
- Bugs in production
- Unexpected behavior
- Performance problems
- Build failures
- Integration issues

**Use this ESPECIALLY when:**
- Under time pressure (emergencies make guessing tempting)
- "Just one quick fix" seems obvious
- You've already tried multiple fixes
- Previous fix didn't work
- You don't fully understand the issue

**Don't skip when:**
- Issue seems simple (simple bugs have root causes too)
- You're in a hurry (rushing guarantees rework)
- Someone wants it fixed NOW (systematic is faster than thrashing)

## The Four Phases

You MUST complete each phase before proceeding to the next.

---

## Phase 1: Root Cause Investigation

**BEFORE attempting ANY fix:**

### 1. Read Error Messages Carefully

- Don't skip past errors or warnings
- They often contain the exact solution
- Read stack traces completely
- Note line numbers, file paths, error codes

**Action:** Use `read_file` on the relevant source files. Use `search_files` to find the error string in the codebase.

### 2. Reproduce Consistently

- Can you trigger it reliably?
- What are the exact steps?
- Does it happen every time?
- If not reproducible → gather more data, don't guess

**Action:** Use the `terminal` tool to run the failing test or trigger the bug:

```bash
# Run specific failing test
pytest tests/test_module.py::test_name -v

# Run with verbose output
pytest tests/test_module.py -v --tb=long
```

### 3. Check Recent Changes

- What changed that could cause this?
- Git diff, recent commits
- New dependencies, config changes

**Action:**

```bash
# Recent commits
git log --oneline -10

# Uncommitted changes
git diff

# Changes in specific file
git log -p --follow src/problematic_file.py | head -100
```

### 4. Gather Evidence in Multi-Component Systems

**WHEN system has multiple components (API → service → database, CI → build → deploy):**

**BEFORE proposing fixes, add diagnostic instrumentation:**

For EACH component boundary:
- Log what data enters the component
- Log what data exits the component
- Verify environment/config propagation
- Check state at each layer

Run once to gather evidence showing WHERE it breaks.
THEN analyze evidence to identify the failing component.
THEN investigate that specific component.

### 5. Trace Data Flow

**WHEN error is deep in the call stack:**

- Where does the bad value originate?
- What called this function with the bad value?
- Keep tracing upstream until you find the source
- Fix at the source, not at the symptom

**Action:** Use `search_files` to trace references:

```python
# Find where the function is called
search_files("function_name(", path="src/", file_glob="*.py")

# Find where the variable is set
search_files("variable_name\\s*=", path="src/", file_glob="*.py")
```

### 6. Ghost Grading — Answers Judged But Never Persisted

**WHEN the user reports that daily practice repeats the same questions, or that mastery stats never change despite completing exercises:**

The practice engine may be **grading answers and returning results correctly, but never writing the results to the persistence layer**. This creates a phantom system where every session starts fresh.

**This is distinct from "data source divergence" above** — divergence means two stores have different data; ghost grading means **no data ever reaches any store**.

**Diagnostic checklist:**

1. **Check the submit handler for persistence calls**
   ```bash
   # Look for the submit handler in the practice engine
   grep -n "def.*submit\|def.*answer" routes.py
   # Read the function — does it call any write/record/save/persist function?
   grep -n "record_answer\|update_memory\|save_session\|record_test\|init_word_frequency" routes.py
   ```

2. **Two-request persistence test**
   ```python
   # 1. Submit an answer
   POST /api/practice/submit {"answer": "test", "session_id": "X"}
   # 2. Immediately re-read the session/state
   GET /api/practice/current?session=X
   # If the same question comes back at index 0 → persistence failed
   ```

3. **Restart + retry test**
   - Complete a practice session
   - Restart the server
   - Start a new practice session
   - **If the first 10 questions are identical to the previous session** → SM-2 model never updated (persistence gap)

4. **Direct file check**
   ```bash
   # Check the persistent store directly, not via API
   cat path/to/persistence/store.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(sum(1 for v in d.values() if v.get('total_tests',0)>0))"
   ```

5. **Trace ONE submit handler end-to-end**
   ```
   User clicks "submit" → JavaScript fetches /api/practice/submit
     → Flask route handler → grading logic → returns response
     → Is there a call to record_answer/update_memory_curve/init_word_frequency/record_test_result?
       ↓                        ↓
    YES (good)                NO (GHOST GRADING!)
                               ↓
                      Answers are graded in-memory,
                      response sent to frontend,
                      then **discarded forever**.
   ```

**Real-world example (C008 English v4, 2026-05-06):**

The v4 unified practice engine (`unified_practice.py`) generated mixed word+grammar questions. `api_v4_practice_submit()` in `v3/routes.py` graded each answer and stored results in `_V4_SESSIONS[sid]["answers"]` (in-memory dict). But **no function in the submit path called `dm.init_word_frequency()` or `dm.record_test_result()`** — the SM-2 data in `word_frequency.json` stayed at `repetitions=0` for all 301 words.

Result: every daily practice selected the same top-of-list words because SM-2 never advanced. Parent records showed 0 mastered/learning even though the child had practiced 100+ answers.

**Fix pattern:**
```python
# AFTER grading, BEFORE returning response:
def api_v4_practice_submit():
    grade = _v4_grade_question(q, user_answer)
    is_correct = grade["correct"]
    
    # ── PERSISTENCE GATE ──
    # For word-type questions, persist to SM-2 model
    qtype = q.get("type", "")
    if qtype in ("spelling", "en2cn", "fill_first_letter"):
        word_en = q.get("word") or q.get("en") or ""
        if word_en:
            dm.init_word_frequency(word_en, batch_id, student_id)
            dm.record_test_result(word_en, is_correct, student_id)
    # ──────────────────────
```

**Prevention — add to every new practice engine's review checklist:**
- [ ] Submit handler calls a persistence function after grading
- [ ] The persistence function writes to the SAME store the reporting UI reads from
- [ ] Restart the server and verify practice data survives

### 7. Data Source Divergence — Statistics Don't Match Reality

### 8. Frontend-Backend API Data Contract Drift

**WHEN the user reports that ALL answers are marked wrong, every submission fails silently, or the system appears to 'judge' but never correctly:**  

The root cause is often not the grading logic itself, but a mismatch in the **data contract** between frontend and backend — both sides work correctly when called individually, but they disagree on what the API input/output looks like.

**Diagnostic checklist:**

1. **Capture the exact request body the frontend sends**
   ```javascript
   // Add to submitAnswer() before the fetch:
   console.log('Sending:', JSON.stringify({question_id: q.id, answer: userAnswer}));
   ```
   Or use browser DevTools Network tab to inspect the request payload.

2. **Compare against what the backend expects**
   ```python
   # In the Flask route handler:
   data = request.get_json(force=True, silent=True) or {}
   print(f"RECEIVED: {data.keys()}")  # Compare keys
   print(f"EXPECTING: question_id or answers")
   ```

3. **Check the response path too**
   ```javascript
   // Frontend reads:
   result.correct       // exists?
   result.correct_answer // exists? or is it result.expected_answer?
   result.solution      // exists?
   ```
   ```python
   # Backend returns:
   return jsonify({
       "correct": ...,
       "expected_answer": ...,  # Frontend might read result.correct_answer!
   })
   ```

4. **Common drift patterns**

   | Pattern | Frontend sends/reads | Backend returns/expects | Symptom |
   |---------|---------------------|------------------------|---------|
   | Wrap drift | `{question_id, answer}` | `{answers: [{id, answer}]}` | Backend gets empty `answers[]` → 400 → `.correct` is undefined → all wrong |
   | Text key rename | `result.correct_answer` | returns `expected_answer` | Frontend shows empty/undefined response |
   | Boolean key rename | `data.correct` (boolean flag) | returns `is_correct` (boolean flag) | **All answers judged wrong** despite DB showing correct records — backend comparison works correctly, only the response key name differs |
   | Nesting diff | `{session_id, index, answer}` | `{session_id, data: {answer}}` | Backend reads wrong path |
   | Type coercion | sends `answer: "10"` | expects `answer: 10` (int) | `"10" == 10` might fail depending on comparison |

5. **Root cause pattern — Bilateral correctness with mismatched contract**

   This is special because **both sides are individually correct** — the frontend correctly constructs its request, the backend correctly processes what it expects. The bug is only in the **agreement between them**, not in either implementation.

   **Unlike typical bugs where one side is wrong**, you can't find it by reading either side alone. You must read BOTH sides simultaneously.

6. **Systematic fix — add a compatibility layer**

   ```python
   # In the backend, accept both formats:
   answers = data.get("answers", [])
   if not answers:
       qid = data.get("question_id", "")
       user_answer = data.get("answer", "")
       if qid and user_answer is not None:
           answers = [{"id": qid, "answer": user_answer}]
   ```

   This future-proofs the API without requiring frontend changes.

7. **Prevention — contract-first development**

   For any new API endpoint shared between frontend and backend:
   - [ ] **Write the data contract first** — a comment or docstring at both ends
   - [ ] **Check both ends simultaneously** — don't test frontend/backend separately
   - [ ] **Add a single-request test** — use curl to send the EXACT format the frontend sends
   - [ ] **Verify field names match** — especially `correct_answer` vs `expected_answer`, `id` vs `question_id`

### 9. JS Inline Script Syntax Error Bisection

**WHEN a Single-Page App (SPA) embedded in an HTML `<script>` tag renders as a blank skeleton — no JavaScript executes, dynamic elements (mode cards, category tags, quiz cards) don't appear, but the static HTML renders fine:**

The entire `<script>` block may have a syntax error that prevents parsing. Since the error stops ALL code from running, no individual function fails — the page is completely static with no visible error.

**Why it's silent:** The browser's `onerror` handler catches the parse failure but may not surface it in the console in a useful way. The page just "doesn't work" with zero obvious errors to the user.

**Diagnostic checklist:**

1. **Check if any JS variables are defined** using browser console:
   ```javascript
   typeof ALL_QUESTIONS   // Returns "undefined" → script didn't execute at all
   typeof MODES           // Same check for any other top-level variable
   document.getElementById('modeGrid')?.innerHTML || 'NOT FOUND'
   // If modeGrid exists in HTML but is empty AND MODES is undefined → script failed
   ```

2. **Validate the full script with Node.js** (works on the rendered HTML file):
   ```bash
   node -e "
   const fs = require('fs');
   const html = fs.readFileSync('template.html', 'utf8');
   const match = html.match(/<script>([\s\S]*?)<\/script>/);
   if (match) {
       try {
           new Function(match[1]);
           console.log('PARSE OK');
       } catch(e) {
           console.log('PARSE ERROR:', e.message);
       }
   }
   "
   ```

3. **If the error is "Unexpected string" with no useful line number** (because a large data array is on a single ~111K line, or the script is >128KB), use **function-boundary bisection**:
   ```javascript
   const positions = {
       'func1': script.indexOf('function func1()'),
       'func2': script.indexOf('function func2()'),
       'func3': script.indexOf('function func3()'),
   };
   
   // Test progressively larger portions to find the error region
   const upTo = script.substring(0, positions.func2);
   try { new Function(upTo); console.log('OK to func2'); }
   catch(e) { console.log('Error before func2'); }
   ```

4. **Once the error region is isolated, look for quoting issues:**
   - **Unescaped single quotes inside single-quoted JS strings**: `'...selectCategory('')...'` — the `'')` closes the outer string at `'`, leaving `')\>...'` as invalid syntax
   - **Fix**: escape inner quotes as `\'` — `'...selectCategory(\'\')...'`
   - **Template literals vs string concatenation**: Template literals (backticks) handle `'` and `"` safely. Use them when building innerHTML with inline onclick handlers.

5. **Common suspicious patterns** that create this bug:
   ```javascript
   // ❌ BROKEN — single quotes inside single-quoted string
   tags.innerHTML = '... onclick="selectCategory('')">全部</button> ...';
   //    The '') closes the outer string at the first '
   
   // ✅ FIXED — escape inner single quotes
   tags.innerHTML = '... onclick="selectCategory(\'\')">全部</button> ...';
   
   // ✅ ALSO WORKS — use template literal (backticks) for the outer string
   tags.innerHTML = `... onclick="selectCategory('')">全部</button> ...`;
   ```
   
   **Always check**: Any HTML attribute value that contains `'` (single quote) inside a JS string that's delimited by `'`.

6. **Verification** — confirm the fix with Node.js:
   ```bash
   node -e "
   const fs = require('fs');
   const html = fs.readFileSync('template.html', 'utf8');
   const m = html.match(/<script>([\s\S]*?)<\/script>/);
   new Function(m[1]) ? console.log('FIX CONFIRMED') : console.log('STILL BROKEN');
   "
   ```

7. **Prevention** — add to frontend template review checklist:
   - [ ] After editing templates with inline `<script>`, run `node --check` on extracted JS
   - [ ] Pay special attention to any HTML attribute value inside a JS string literal
   - [ ] Use template literals (backticks) for complex HTML-building strings that contain `'` or `"`
   - [ ] After fix, restart Flask app AND verify browser loads correctly (trap: template cache)

### 9a. Browser Console Fix-Before-Write Workflow

**When you've found a JS syntax error in an inline `<script>` via `new Function()` bisection (section #9), use the browser console to test fixes in-memory before writing to disk:**

```javascript
// 1. Read the raw script and apply the fix in-memory
var src = document.querySelector('script').textContent;
src = src.replace('bad_code_here', 'fixed_code_here');

// 2. Validate the fix
try {
    new Function(src);
    console.log('FIX CONFIRMED - no syntax errors');
} catch(e) {
    console.log('STILL BROKEN:', e.message);
}

// 3. If confirmed, write the same replacement to the source file
```

**This avoids the edit-flush-restart cycle during debugging.** Only write to disk when the browser confirms the fix.

### 9b. Flask Template Cache Trap

**When the JS fix has been applied to the template file but the browser still shows the old (broken) behavior:**

Flask caches templates in production mode (`debug=False`). The running server process holds the old version in memory. The fix won't take effect until the Flask process is restarted.

**Restart steps:**
```bash
# 1. Kill all Flask/WSGI processes on the port
pkill -f "python3 app.py"    # Adjust pattern as needed

# 2. Wait for the port to free
lsof -i :5002                # Should show nothing

# 3. Restart the Flask app
cd ~/edu-hub && python3 app.py > /dev/null 2>&1 &

# 4. Verify new content is served
curl -s http://localhost:5002/quiz/ | grep -c "FIXED_PATTERN"
```

**Alternative (if Flask is behind a process manager):**
```bash
sudo systemctl restart flask-app    # Systemd service
# or
pm2 restart flask-app               # PM2 process manager
```

**Trap to remember:** A `write_file` or `patch` to a template file does NOT refresh a running Flask instance. You must always restart the server process for template changes to take effect in production mode.

### 10. Silent Rendering Collapse — Data Vanishes in the Pipeline

**WHEN the user reports that the UI renders with NO interactive elements (no input fields, no choice buttons, no submit button), yet the page loads with a question text and blank space where controls should be:**

The root cause is **data dropped at an intermediate pipeline layer** — the frontend template has a conditional that renders nothing when a required field is missing (`null`/`undefined`/`None`). No error appears because the conditional simply evaluates to `false`.

**Unlike Ghost Grading (#6) where data is graded but never persisted, this is a full data-loss chain: data is generated, then lost at a storage/retrieval junction, and the template silently shows nothing.**

**The symptom is consistent:** every question, every session, the same blank.

**Diagnostic checklist:**

1. **Inspect the rendered HTML directly** — don't guess from the browser's visual behavior:
   ```html
   <!-- Look for empty conditional blocks -->
   <!-- Choice questions -->
           
   <!-- Fill questions -->
           
   ```
   If `<div>` containers exist but are empty → the template conditional evaluated to `false`.

2. **Identify what field the template conditional requires:**

   ```jinja
   {# pattern A: choice questions need both type AND choices #}
   {% if question.type in ('choice',) and question.choices %}
     ...render choices...
   {% endif %}

   {# pattern B: fill questions only need type #}
   {% if question.type in ('fill', 'calc', 'word_problem') %}
     ...render input...
   {% endif %}
   ```
   
   - Pattern A renders **nothing** when `question.choices` is `None`/`undefined`, even when `type='choice'` is correct.
   - Pattern B renders **nothing** when `question.type` doesn't match any known type.

3. **Trace the field backward through the pipeline:**
   
   ```
   template renders {question.choices}
        ↑
   get_current_question() returns question dict
        ↑
   practice_answers table stores fields
        ↑
   start_practice() / create_session() inserts row
        ↑
   generate() builds question dict from template or DB
        ↑
   Template/hardcoded data (source of truth)
   ```

   At EACH junction, check: does the field survive?
   
   | Junction | Check |
   |----------|-------|
   | Source → generate() | Does `generate()` include the field in the returned dict? |
   | generate() → storage | Does the INSERT statement include the field? |
   | storage → retrieval | Does the SELECT return the field? |
   | retrieval → route | Does `get_current_question()` include it in the return dict? |
   | route → template | Does `render_template()` pass it? |
   | template → render | Does the conditional use `and question.field` (not just `question.type`)? |

4. **Common pipeline breaks:**

   | Break point | Pattern | Fix |
   |------------|---------|-----|
   | `generate()` drops field | Dict spread creates a new dict without `answer`/`choices`: `{'type': t['type'], ...}` | Add the field: `'choices': t.get('choices'), 'answer': t.get('answer', '')` |
   | Storage column doesn't exist | Table schema was never updated for the new field | `ALTER TABLE ... ADD COLUMN` |
   | Storage uses new column, retrieval doesn't read it | `SELECT *` works, but the function only extracts specific keys | Add the field to the return dict |
   | Template requires both `type` AND field | `{% if type == 'choice' and choices %}` — `choices` might be `None` while `type` is correct | Fix upstream to always provide the field, or add a fallback |

5. **Form-vs-JSON submission mismatch — the companion bug**

   When the UI renders correctly but submitting does nothing (page stays on the same question, or the user gets a raw JSON response):
   
   ```
   Form POST (content-type: application/x-www-form-urlencoded)
        ↓
   Backend reads request.get_json() → None → empty dict → no data processed
        ↓
   Returns JSON response → browser displays raw JSON instead of redirecting
   ```

   **Fix pattern:** accept both formats in the route handler:
   ```python
   if request.is_json:
       data = request.get_json() or {}
   else:
       data = request.form or {}
   ```

   **Prevention — add to practice engine review checklist:**
   - [ ] For any form-answering flow, verify the backend reads `request.form` (not just `request.get_json`)
   - [ ] Trace ONE submission: form POST → backend handler → redirect → next question render

**Real-world example (AI家庭教师语文练习, 2026-05-16):**

The `practice_engine.QuestionGenerator.generate()` method built template question dicts with `type`, `text`, `choices`, `difficulty`, `subject` — but **dropped `answer` (the correct answer field)**. Then `start_practice()` stored the question but the `practice_answers` table had no `answer_choices` column. `get_current_question()` returned a dict with `type='choice'` but no `choices`. The Jinja2 template required `question.type in ('choice',) and question.choices` → evaluated to `False` → rendered empty `<div>`.

Additionally, the answer submission route used `request.get_json()` which returned `None` for form POSTs → no data processed → no redirect → empty response.

**Fix chain:**
```
1. generate() → add 'answer': t.get('answer', '')
2. practice_answers → ALTER TABLE ADD COLUMN answer_choices TEXT
3. start_practice() → store json.dumps(choices)
4. get_current_question() → parse and return choices
5. answer route → accept request.form as fallback
```

**Verification:** Start a new practice session (old sessions with missing data should be cleared). The rendered HTML should show `<li>` elements for choices or `<input>` for fill, and form submission should redirect to the next question.

**WHEN the user says "the stats don't match what I see" (mastered count, progress %, daily practice word selection):**

The most likely root cause is **not a bad algorithm but a bad data source** — the UI reads from Data Store A while the practice engine writes to Data Store B, and they've diverged.

**This pattern is the `systematic-debugging` equivalent of checking 'is the wire plugged in' before blaming the computer.**

**Diagnostic checklist:**

1. **Find every data store that tracks the same concept**
   ```bash
   grep -rn "mastered\|learning\|repetitions\|memory_curve\|word_frequency" --include="*.py" --include="*.json" . | grep -v __pycache__ | grep -v ".pyc"
   ```

2. **Count entries in each store**
   ```python
   # For JSON stores:
   import json, os
   for f in ["store_a.json", "store_b.json"]:
       data = json.load(open(f))
       print(f, len(data))
   ```

3. **Compare overlapping keys between stores**
   ```python
   keys_a = set(...)
   keys_b = set(...)
   print(f"Only in A: {len(keys_a - keys_b)}")
   print(f"Only in B: {len(keys_b - keys_a)}")
   ```

4. **For each key, compare the "status" field**
   - Same word, different status in each store? → divergence
   - One store has data, other doesn't? → missing persistence path

5. **Check each version's persistence path separately**
   ```text
   Version-specific code smell:
   - v1 writes to store A
   - v2 writes to store B  
   - v3 writes to store B (good)
   - v4/v5: check if they write ANYWHERE or just grade in-memory

   If a version has a submit handler but no call to record_answer/update_word_frequency/
   record_test_result → it's the version that broke the stats.
   ```

6. **Trace ONE complete word through the full chain**
   - Start: content.json → practice engine generates question
   - Middle: user answers → submit handler → grading
   - End: is the result written to the same store the UI reads from?

**Real-world example (C008 English, 2026-05-06):**

| Data Store | File | Words | Purpose |
|-----------|------|-------|---------|
| state.json (old) | english/data/state.json | 29 | v1/v2 dictation, **read by parent records page** |
| word_frequency.json (new) | english/data/word_frequency.json | 301 | **v4 practice engine** writes here for SM-2 |
| memory_curve.json (v3) | english/data/v3/memory_curve.json | 0 bytes | v3 SM-2 (disused) |

**Root cause:** `api_v4_practice_submit()` graded answers and returned them to the frontend, but **never called `record_test_result()` to persist to `word_frequency.json`**. The UI reads from `state.json` which only has 29 words (all `new`), while the engine uses `word_frequency.json` with 301 words (all `repetitions=0`). Both show 0 mastered for different reasons, and the daily practice engine keeps selecting the same top-of-list words because SM-2 never advances.

**Fix:** Add `dm.init_word_frequency()` + `dm.record_test_result()` in `api_v4_practice_submit()` after grading.

### Phase 1 Completion Checklist

- [ ] Error messages fully read and understood
- [ ] Issue reproduced consistently
- [ ] Recent changes identified and reviewed
- [ ] Evidence gathered (logs, state, data flow)
- [ ] **Data source divergence checked** — if UI shows stats, verify the store it reads matches the store the engine writes to
- [ ] Problem isolated to specific component/code
- [ ] Root cause hypothesis formed

**STOP:** Do not proceed to Phase 2 until you understand WHY it's happening.

---

## Phase 2: Pattern Analysis

**Find the pattern before fixing:**

### 1. Find Working Examples

- Locate similar working code in the same codebase
- What works that's similar to what's broken?

**Action:** Use `search_files` to find comparable patterns:

```python
search_files("similar_pattern", path="src/", file_glob="*.py")
```

### 2. Compare Against References

- If implementing a pattern, read the reference implementation COMPLETELY
- Don't skim — read every line
- Understand the pattern fully before applying

### 3. Identify Differences

- What's different between working and broken?
- List every difference, however small
- Don't assume "that can't matter"

### 4. Understand Dependencies

- What other components does this need?
- What settings, config, environment?
- What assumptions does it make?

---

## Phase 3: Hypothesis and Testing

**Scientific method:**

### 1. Form a Single Hypothesis

- State clearly: "I think X is the root cause because Y"
- Write it down
- Be specific, not vague

### 2. Test Minimally

- Make the SMALLEST possible change to test the hypothesis
- One variable at a time
- Don't fix multiple things at once

### 3. Verify Before Continuing

- Did it work? → Phase 4
- Didn't work? → Form NEW hypothesis
- DON'T add more fixes on top

### 4. When You Don't Know

- Say "I don't understand X"
- Don't pretend to know
- Ask the user for help
- Research more

---

## Phase 4: Implementation

**Fix the root cause, not the symptom:**

### 1. Create Failing Test Case

- Simplest possible reproduction
- Automated test if possible
- MUST have before fixing
- Use the `test-driven-development` skill

### 2. Implement Single Fix

- Address the root cause identified
- ONE change at a time
- No "while I'm here" improvements
- No bundled refactoring

### 3. Verify Fix

```bash
# Run the specific regression test
pytest tests/test_module.py::test_regression -v

# Run full suite — no regressions
pytest tests/ -q
```

### 4. If Fix Doesn't Work — The Rule of Three

- **STOP.**
- Count: How many fixes have you tried?
- If < 3: Return to Phase 1, re-analyze with new information
- **If ≥ 3: STOP and question the architecture (step 5 below)**
- DON'T attempt Fix #4 without architectural discussion

### 5. If 3+ Fixes Failed: Question Architecture

**Pattern indicating an architectural problem:**
- Each fix reveals new shared state/coupling in a different place
- Fixes require "massive refactoring" to implement
- Each fix creates new symptoms elsewhere

**STOP and question fundamentals:**
- Is this pattern fundamentally sound?
- Are we "sticking with it through sheer inertia"?
- Should we refactor the architecture vs. continue fixing symptoms?

**Discuss with the user before attempting more fixes.**

This is NOT a failed hypothesis — this is a wrong architecture.

### 6. The Iceberg Pattern — When Fixes Reveal Hidden Layers

A special case of the "3+ fixes" rule: sometimes each fix IS correct for the layer it addressed, but the bug has **multiple layers of indirection** that were previously masked by the very bug you just fixed.

**How to detect an iceberg bug:**

| Signal | What it means |
|--------|---------------|
| Fix #1 resolved the symptom but a related symptom appears | Fix #1 was correct for its layer but uncovered the next layer |
| The new symptom involves a DIFFERENT code path than Fix #1 | The underlying issue is a chain, not a single link |
| Each fix touches a different file/module/layer | The bug spans the full data flow (data :arrow_right: backend :arrow_right: API :arrow_right: frontend) |
| User says "still broken" with different specifics | The visible symptom changed because the root cause is at a deeper layer |

**The iceberg debugging protocol:**

When a fix for user-reported symptom X is deployed but user reports symptom Y (related but different):

```
Step 1: Map the FULL data flow chain
  data source :arrow_right: backend processing :arrow_right: API response :arrow_right: frontend rendering :arrow_right: user eyes

Step 2: Mark where each fix landed on this chain
  Fix #1 :arrow_right: back end (changed prompt text)     [layer 2 of 4]
  Fix #2 :arrow_right: API (changed fallback chain)       [layer 3 of 4]
  Fix #3 :arrow_right: frontend (added category mapping)  [layer 4 of 4]
  :arrow_left: If a layer is unmarked, that's where the remaining bug lives

Step 3: For the unmarked layer, ask:
  - What does the data look like entering this layer?
  - What transformation happens in this layer?
  - What does the output look like?

Step 4: Verify EVERYTHING flows through the full chain
  Don't assume "I fixed the backend so the frontend will work"
  Trace one complete item from data source to rendered output
```

**Real-world example (C008 English daily practice, 2026-05-06):**

| Fix # | Symptom after fix | Layer | Correct? |
|-------|-------------------|-------|----------|
| Original bug | "请输入单词 'apple' 的正确拼写" — answer in prompt | Backend (prompt text) | :white_check_mark: but... |
| Fix #1 | "请输入单词的正确拼写" — no answer shown, but no word either | Backend (cn overwrite) | :white_check_mark: revealed next layer |
| Fix #2 | API returns question_text correctly | API (fallback chain) | :white_check_mark: revealed next layer |
| Fix #3 | **"请输入word的正确拼写"** — category "word" used in Chinese text | Frontend (category:arrow_right:Chinese) | :white_check_mark: revealed next layer |
| Fix #4 | "He is the ___ (smart) student." — no instruction on what to do | Frontend (question hint) | :white_check_mark: |

Each fix was individually correct. The bug had 4 layers of indirection that were only revealed one at a time. **The iceberg protocol would have caught all 4 in one pass by tracing the full data flow chain before making any fix.**

**Prevention:** Before ANY fix for a bug involving user-facing text (question prompts, labels, error messages), trace the complete chain from data source :arrow_right: backend processing :arrow_right: API response :arrow_right: frontend rendering. Map all 4 layers. Only then propose fixes.

---

## Red Flags — STOP and Follow Process

If you catch yourself thinking:
- "Quick fix for now, investigate later"
- "Just try changing X and see if it works"
- "Add multiple changes, run tests"
- "Skip the test, I'll manually verify"
- "It's probably X, let me fix that"
- "I don't fully understand but this might work"
- "Pattern says X but I'll adapt it differently"
- "Here are the main problems: [lists fixes without investigation]"
- Proposing solutions before tracing data flow
- **"One more fix attempt" (when already tried 2+)**
- **Each fix reveals a new problem in a different place**

**ALL of these mean: STOP. Return to Phase 1.**

**If 3+ fixes failed:** Question the architecture (Phase 4 step 5).

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Issue is simple, don't need process" | Simple issues have root causes too. Process is fast for simple bugs. |
| "Emergency, no time for process" | Systematic debugging is FASTER than guess-and-check thrashing. |
| "Just try this first, then investigate" | First fix sets the pattern. Do it right from the start. |
| "I'll write test after confirming fix works" | Untested fixes don't stick. Test first proves it. |
| "Multiple fixes at once saves time" | Can't isolate what worked. Causes new bugs. |
| "Reference too long, I'll adapt the pattern" | Partial understanding guarantees bugs. Read it completely. |
| "I see the problem, let me fix it" | Seeing symptoms ≠ understanding root cause. |
| "One more fix attempt" (after 2+ failures) | 3+ failures = architectural problem. Question the pattern, don't fix again. |

## Quick Reference

| Phase | Key Activities | Success Criteria |
|-------|---------------|------------------|
| **1. Root Cause** | Read errors, reproduce, check changes, gather evidence, trace data flow | Understand WHAT and WHY |
| **2. Pattern** | Find working examples, compare, identify differences | Know what's different |
| **3. Hypothesis** | Form theory, test minimally, one variable at a time | Confirmed or new hypothesis |
| **4. Implementation** | Create regression test, fix root cause, verify | Bug resolved, all tests pass |

## Reference Files

- `references/grading-field-mismatch.md` — Tracing the "expected answer" field across all layers (data source → backend grading → API response → frontend display). Use when a grading/practice system judges correct input as wrong.
- `references/flask-web-debugging.md` — Flask-specific error patterns: TemplateNotFound, UndefinedError in templates, broken url_for endpoints, blueprint route conflicts, debug-mode traceback reading, and module reloader behavior. Use when debugging a Flask web app that returns 500/404 with an HTML traceback.

## Hermes Agent Integration

### Investigation Tools

Use these Hermes tools during Phase 1:

- **`search_files`** — Find error strings, trace function calls, locate patterns
- **`read_file`** — Read source code with line numbers for precise analysis
- **`terminal`** — Run tests, check git history, reproduce bugs
- **`web_search`/`web_extract`** — Research error messages, library docs

### With delegate_task

For complex multi-component debugging, dispatch investigation subagents:

```python
delegate_task(
    goal="Investigate why [specific test/behavior] fails",
    context="""
    Follow systematic-debugging skill:
    1. Read the error message carefully
    2. Reproduce the issue
    3. Trace the data flow to find root cause
    4. Report findings — do NOT fix yet

    Error: [paste full error]
    File: [path to failing code]
    Test command: [exact command]
    """,
    toolsets=['terminal', 'file']
)
```

### With test-driven-development

When fixing bugs:
1. Write a test that reproduces the bug (RED)
2. Debug systematically to find root cause
3. Fix the root cause (GREEN)
4. The test proves the fix and prevents regression

## Real-World Impact

From debugging sessions:
- Systematic approach: 15-30 minutes to fix
- Random fixes approach: 2-3 hours of thrashing
- First-time fix rate: 95% vs 40%
- New bugs introduced: Near zero vs common

**No shortcuts. No guessing. Systematic always wins.**
