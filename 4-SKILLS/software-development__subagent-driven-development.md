---
name: subagent-driven-development
description: "Execute plans via delegate_task subagents (2-stage review + blinded spec compliance + 2-round escalation + Aider integration). v1.3.0: Aider as implementation engine for coding subagents."
version: 1.3.0
author: Hermes Agent (adapted from obra/superpowers)
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [delegation, subagent, implementation, workflow, parallel]
    related_skills: [writing-plans, requesting-code-review, test-driven-development]
---

# Subagent-Driven Development

## Overview

Execute implementation plans by dispatching fresh subagents per task with systematic two-stage review.

**Core principle:** Fresh subagent per task + two-stage review (spec then quality) = high quality, fast iteration.

**Critical insight (v1.2.0):** The same agent cannot effectively review its own work — confirmation bias causes it to rationalize micro-deviations with "I designed it this way" reasoning. The solution is role separation: the reviewer subagent must **never see the full design document**, only the acceptance criteria checklist. This eliminates the "I know what this was supposed to do" shortcut that suppresses discrepancy detection.

**2-round escalation rule (v1.2.0):** Any review FAIL gets exactly 2 fix-review cycles. If the 2nd review also FAILs, the issue **escalates to Hermes** (the orchestrator agent) — it does NOT loop indefinitely. This prevents token waste on tasks where the acceptance criteria are flawed or the subagent is not a good fit.

## The Confirmation Bias Trap

### Why it matters

When the same agent designs and implements, then reviews its own output:

```
Designer brain: "I chose this route structure for a good reason"
  → Reviewer brain (same agent): "That reason still makes sense"
    → Deviation missed
```

This is not a bug in the agent — it's a **structural problem** with self-review. The agent can't "unknow" the design rationale.

### The workaround: blinded reviewers

The reviewer subagent **must not** receive the full design document. It receives only:

1. **Atomic acceptance criteria** — binary PASS/FAIL conditions
2. **File paths** to check
3. **Request/response schemas** — exact field names and types

Everything else (design rationale, alternative approaches considered, trade-offs made) is **withheld from the reviewer**. This forces the reviewer to do a purely mechanical compliance check rather than an interpretive one.

## When to Use

Use this skill when:
- You have an implementation plan (from writing-plans skill or user requirements)
- Tasks are mostly independent
- Quality and spec compliance are important
- You want automated review between tasks

**vs. manual execution:**
- Fresh context per task (no confusion from accumulated state)
- Automated review process catches issues early
- Consistent quality checks across all tasks
- Subagents can ask questions before starting work

## The Process

### 1. Read and Parse Plan

Read the plan file. Extract ALL tasks with their full text and context upfront. Create a todo list:

```python
# Read the plan
read_file("docs/plans/feature-plan.md")

# Create todo list with all tasks
todo([
    {"id": "task-1", "content": "Create User model with email field", "status": "pending"},
    {"id": "task-2", "content": "Add password hashing utility", "status": "pending"},
    {"id": "task-3", "content": "Create login endpoint", "status": "pending"},
])
```

**Key:** Read the plan ONCE. Extract everything. Don't make subagents read the plan file — provide the full task text directly in context.

### 2. Per-Task Workflow

For EACH task in the plan:

#### Step 1: Dispatch Implementer Subagent

Use `delegate_task` with complete context:

```python
delegate_task(
    goal="Implement Task 1: Create User model with email and password_hash fields",
    context="""
    TASK FROM PLAN:
    - Create: src/models/user.py
    - Add User class with email (str) and password_hash (str) fields
    - Use bcrypt for password hashing
    - Include __repr__ for debugging

    FOLLOW TDD:
    1. Write failing test in tests/models/test_user.py
    2. Run: pytest tests/models/test_user.py -v (verify FAIL)
    3. Write minimal implementation
    4. Run: pytest tests/models/test_user.py -v (verify PASS)
    5. Run: pytest tests/ -q (verify no regressions)
    6. Commit: git add -A && git commit -m "feat: add User model with password hashing"

    PROJECT CONTEXT:
    - Python 3.11, Flask app in src/app.py
    - Existing models in src/models/
    - Tests use pytest, run from project root
    - bcrypt already in requirements.txt
    """,
    toolsets=['terminal', 'file']
)
```

#### Step 2: Dispatch Spec Compliance Reviewer (Blinded)

Dispatch a reviewer subagent that has **no access to the original design document** — only the acceptance criteria.

Critical difference from a normal review:

| Aspect | Current practice | v1.2.0 improvement |
|:-------|:-----------------|:--------------------|
| Reviewer's context | Full task spec + design doc | **Only acceptance criteria** + file paths |
| Review criteria | "Does this match the spec?" | **Only binary checks** from criteria list |
| Judgment | Interpretive ("close enough") | Mechanical ("this field exists or not") |
| Deviation tolerance | ~15% (can rationalize minor gaps) | ~0% (must hit every check) |

```python
delegate_task(
    goal="Verify acceptance criteria are met — DO NOT read the design document",
    context="""
    ACCEPTANCE CRITERIA (only source of truth for review):
    [ ] File exists: src/models/user.py
    [ ] Class User has fields: email (str), password_hash (str)
    [ ] email field has UNIQUE constraint
    [ ] __repr__ method returns "<User: email>"
    [ ] Tests in tests/models/test_user.py: 5+ test cases
    [ ] All tests pass: pytest tests/models/test_user.py -v

    ⚠️ CRITICAL RULE: Do not apply "this seems reasonable" judgment.
    ⚠️ CRITICAL RULE: Do not consider what the "intended behavior" was.
    ⚠️ CRITICAL RULE: Check each criterion as a binary YES/NO.

    OUTPUT: PASS or list of FAIL criteria with specific evidence.
    """,
    toolsets=['file']
)
```

**If spec issues found:** Fix gaps, then re-run spec review. Continue only when spec-compliant.
**2-round escalation:** If spec review FAILs twice, escalate to Hermes. Do not proceed to coding until spec is compliant.

#### Step 3: Dispatch Code Quality Reviewer

After spec compliance passes:

```python
delegate_task(
    goal="Review code quality for Task 1 implementation",
    context="""
    FILES TO REVIEW:
    - src/models/user.py
    - tests/models/test_user.py

    CHECK:
    - [ ] Follows project conventions and style?
    - [ ] Proper error handling?
    - [ ] Clear variable/function names?
    - [ ] Adequate test coverage?
    - [ ] No obvious bugs or missed edge cases?
    - [ ] No security issues?

    OUTPUT FORMAT:
    - Critical Issues: [must fix before proceeding]
    - Important Issues: [should fix]
    - Minor Issues: [optional]
    - Verdict: APPROVED or REQUEST_CHANGES
    """,
    toolsets=['file']
)
```

**If quality issues found:** Fix issues, re-review. Continue only when approved.

#### Step 4: Mark Complete

```python
todo([{"id": "task-1", "content": "Create User model with email field", "status": "completed"}], merge=True)
```

### 3. Final Review

After ALL tasks are complete, dispatch a final integration reviewer:

```python
delegate_task(
    goal="Review the entire implementation for consistency and integration issues",
    context="""
    All tasks from the plan are complete. Review the full implementation:
    - Do all components work together?
    - Any inconsistencies between tasks?
    - All tests passing?
    - Ready for merge?
    """,
    toolsets=['terminal', 'file']
)
```

### 4. Verify and Commit

```bash
# Run full test suite
pytest tests/ -q

# Review all changes
git diff --stat

# Final commit if needed
git add -A && git commit -m "feat: complete [feature name] implementation"
```

## Task Granularity

**Each task = 2-5 minutes of focused work.**

**Too big:**
- "Implement user authentication system"

**Right size:**
- "Create User model with email and password fields"
- "Add password hashing function"
- "Create login endpoint"
- "Add JWT token generation"
- "Create registration endpoint"

## Red Flags — Never Do These

- Start implementation without a plan
- Skip reviews (spec compliance OR code quality)
- Proceed with unfixed critical/important issues
- Dispatch multiple implementation subagents for tasks that touch the same files
- Make subagent read the plan file (provide full text in context instead)
- Skip scene-setting context (subagent needs to understand where the task fits)
- Ignore subagent questions (answer before letting them proceed)
- Accept "close enough" on spec compliance
- Skip review loops (reviewer found issues → implementer fixes → review again)
- Let implementer self-review replace actual review (both are needed)
- **Start code quality review before spec compliance is PASS** (wrong order)
- Move to next task while either review has open issues
- **Give the spec reviewer the full design document** (confirmation bias — reviewer will rationalize deviations)
- **Code for more than 30 minutes without an intermediate checkpoint** (long feedback loops compound micro-deviations)
- **Assume a single subagent can handle a task that spans multiple files with cross-cutting changes** (split into smaller tasks instead)
- **Loop review cycles more than 2 rounds without escalation** (token waste + diminishing returns)
- **Accept a partial PASS from the spec reviewer** — every criterion must pass, no "close enough"
- **Skip the escalation step and continue coding** after 2 failed reviews (broken acceptance criteria produce broken code)

## Handling Issues

### If Subagent Asks Questions

- Answer clearly and completely
- Provide additional context if needed
- Don't rush them into implementation

### If Reviewer Finds Issues

1. Fix the issues (same or new subagent)
2. Reviewer reviews again
3. **If review FAILs again: this is Round 2. Fix and re-review.**
4. **If Round 2 also FAILs: STOP. Escalate to Hermes (orchestrator). Do NOT attempt Round 3.**
5. The orchestrator (Hermes) determines: are the acceptance criteria wrong? Is the subagent not suited? Should the task be done differently?

### If Subagent Fails a Task

- Dispatch a new fix subagent with specific instructions about what went wrong
- Don't try to fix manually in the controller session (context pollution)

## Efficiency Notes

**Why fresh subagent per task:**
- Prevents context pollution from accumulated state
- Each subagent gets clean, focused context
- No confusion from prior tasks' code or reasoning

**Why two-stage review:**
- Spec review catches under/over-building early
- Quality review ensures the implementation is well-built
- Catches issues before they compound across tasks

**Cost trade-off:**
- More subagent invocations (implementer + 2 reviewers per task)
- But catches issues early (cheaper than debugging compounded problems later)

## Integration with Other Skills

### With writing-plans

This skill EXECUTES plans created by the writing-plans skill:
1. User requirements → writing-plans → implementation plan
2. Implementation plan → subagent-driven-development → working code

### With test-driven-development

Implementer subagents should follow TDD:
1. Write failing test first
2. Implement minimal code
3. Verify test passes
4. Commit

Include TDD instructions in every implementer context.

### With requesting-code-review

The two-stage review process IS the code review. For final integration review, use the requesting-code-review skill's review dimensions.

### With systematic-debugging

If a subagent encounters bugs during implementation:
1. Follow systematic-debugging process
2. Find root cause before fixing
3. Write regression test
4. Resume implementation

### With Aider (v1.3.0)

Aider (`aider-chat`) can serve as the implementation engine for the coding subAgent, replacing a manually-directed sequence of `write_file` + `terminal` calls. Aider provides:

- **Repo map** — Aider builds an index of the repo structure (`repo-map`) and uses it to identify which files need changes, making cross-file edits much more efficient than manual navigation.
- **Multi-file awareness** — A single `aider --message "implement X"` can create/modify multiple files in one pass, with Aider reasoning about import patterns and cross-file dependencies.
- **Auto git commits** — Aider commits after each change, creating a clean audit trail.
- **Map-refactor** — `--map-refactor` flag enables refactoring-aware edits where Aider analyzes dependency graphs before making changes.
- **Lint self-healing** — Aider automatically runs linters on modified files and fixes issues before committing.

**When to use Aider vs manual subAgent:**

| Factor | Manual subAgent | Aider-based subAgent |
|:-------|:----------------|:---------------------|
| Best for | Small single-file changes (Tier 2/3) | Multi-file features (Tier 0/1) |
| Setup cost | None | Needs venv + wrapper + model config |
| Cross-file awareness | Limited to what context provides | Full repo map (finds patterns across files) |
| Token efficiency | Higher per-call, but more calls | Lower per-call, but fewer calls |
| Git integration | Manual commit | Auto-commit every change |
| Model flexibility | Uses same model as orchestrator | Can use different model (DeepSeek for code, Ollama for simple) |

**Three-phase Aider integration flow:**

```
Phase 1 — Hermes prepares                              Aider role
  ├─ Initialize git repo (Aider requires git)            (passive)
  ├─ Write acceptance criteria file                      │
  ├─ Write .aider.conf.yml (model, flags, excludes)      │
  └─ Set up wrapper script (sources .env for API keys)   │
                                                         ▼
Phase 2 — SubAgent invokes Aider                    Aider active
  ├─ SubAgent calls: `aider-wrapper --message "..."`    │
  ├─ Aider reads repo, analyzes, edits files, commits   │
  └─ SubAgent verifies: tests pass + exit code 0        │
                                                         ▼
Phase 3 — Blind review SubAgent                    Hermes active
  ├─ Review SubAgent (D-Des gate): acceptance criteria  │
  │  vs code — same blinded protocol as manual flow     │
  └─ PASS → proceed / FAIL → 2-round fix escalation     │
```

**Model routing for Aider:**

| Model | When | Config |
|:------|:-----|:-------|
| `deepseek/deepseek-chat` | Main coding (complex refactoring, features) | `DEEPSEEK_API_KEY` in env |
| `ollama/qwen2.5:3b` | Simple edits, quick fixes | `OLLAMA_API_BASE=http://localhost:11434` |
| `deepseek/deepseek-reasoner` | Architecture decisions, design reviews | `DEEPSEEK_API_KEY` in env |

**Wrapper script pattern** (store at `~/.local/bin/aider-wrapper`):

```bash
#!/bin/bash
# Aider wrapper — sources environment variables then runs aider from venv
set -e
if [ -f "$HOME/.hermes/.env" ]; then
    source "$HOME/.hermes/.env"
fi
export OLLAMA_API_BASE="${OLLAMA_API_BASE:-http://localhost:11434}"
exec ~/.aider-venv/bin/aider "$@"
```

**Aider configuration file** (`.aider.conf.yml` at repo root):

```yaml
model: deepseek/deepseek-chat
yes: true
auto-commits: true
dirty-commits: true
git: true
map-refactor: true
lint: true
show-model-warnings: false
suggest-shell-commands: false
cache-prompts: true
analytics: false
```

**SubAgent dispatch template using Aider:**

```python
delegate_task(
    goal=f"""
    1. Run: cd {workdir} && aider-wrapper --message "Implement feature: {feature_desc}"
    2. Verify tests pass: cd {workdir} && pytest tests/ -q
    3. Check git log: cd {workdir} && git log --oneline -3
    4. Report: commit SHAs, files changed, test results
    """,
    context=f"""
    PROJECT: {project_name}
    AIDER WRAPPER: ~/.local/bin/aider-wrapper
    AIDER CONFIG: {workdir}/.aider.conf.yml
    MODEL: deepseek/deepseek-chat (main), ollama/qwen2.5:3b (simple tasks)
    GIT: already initialized at {workdir}

    ACCEPTANCE CRITERIA:
    [paste criteria from docs/acceptance-criteria-[task].md]

    CRITICAL: Aider will handle file changes. Do NOT use write_file/terminal to
    make code changes. Only use terminal to run aider-wrapper and verify results.
    """,
    toolsets=['terminal', 'file']
)
```

**Aider-specific pitfalls:**

- **Aider REQUIRES a git repo** — it refuses to run outside one. Use `mktemp -d && git init` for scratch work, or init if missing.
- **Pipeline pattern** — Aider commits incrementally and you may want to squash before final review. Use `git reset --soft HEAD~n` or let Aider's auto-commits stand.
- **`read_file()` line numbers** — When patching code that was written by Aider, remember that `read_file()` output is prefixed with `NUM|` line numbers. To get raw content for Aider, use `cat` via terminal.
- **Large dependency** — `aider-chat` is ~248MB installed (depends on scipy, litellm, numpy, etc.). Installation takes 3-5 minutes even on fast mirrors. Use a dedicated venv (`~/.aider-venv`).
- **WSL network** — In China, default PyPI may be slow. Use Aliyun mirror: `pip install -i https://mirrors.aliyun.com/pypi/simple/ aider-chat`. Setup `PIP_INDEX_URL` env var for convenience.
- **Model mismatch** — Aider's model IDs follow LiteLLM conventions, not provider-native. `deepseek/deepseek-chat` maps to DeepSeek V4 Flash; `ollama/qwen2.5:3b` to local Ollama.
- **Don't pipe to aider** — Unlike Codex CLI which hangs on stdin capture, Aider accepts file-based message input via `--message` or `--file`. Always use `--message "prompt"` not stdin pipe.
- **Over-aggressive editing** — Aider's repo map may cause it to modify files it shouldn't. Use `.aiderignore` to exclude generated/config files, or constrain with `--read` (read-only files).
- **Review still needed** — Aider produces code that compiles and passes its own tests, but the blinded D-Des review is still required. Aider's code is not exempt from the 2-round escalation protocol.

## Example Workflow

```
[Read plan: docs/plans/auth-feature.md]
[Create todo list with 5 tasks]

--- Task 1: Create User model ---
[Dispatch implementer subagent]
  Implementer: "Should email be unique?"
  You: "Yes, email must be unique"
  Implementer: Implemented, 3/3 tests passing, committed.

[Dispatch spec reviewer]
  Spec reviewer: ✅ PASS — all requirements met

[Dispatch quality reviewer]
  Quality reviewer: ✅ APPROVED — clean code, good tests

[Mark Task 1 complete]

--- Task 2: Password hashing ---
[Dispatch implementer subagent]
  Implementer: No questions, implemented, 5/5 tests passing.

[Dispatch spec reviewer]
  Spec reviewer: ❌ Missing: password strength validation (spec says "min 8 chars")

[Implementer fixes]
  Implementer: Added validation, 7/7 tests passing.

[Dispatch spec reviewer again]
  Spec reviewer: ✅ PASS

[Dispatch quality reviewer]
  Quality reviewer: Important: Magic number 8, extract to constant
  Implementer: Extracted MIN_PASSWORD_LENGTH constant
  Quality reviewer: ✅ APPROVED

[Mark Task 2 complete]

... (continue for all tasks)

[After all tasks: dispatch final integration reviewer]
[Run full test suite: all passing]
[Done!]
```

## Remember

```
Fresh subagent per task
Two-stage review every time
Spec compliance FIRST
Code quality SECOND
Never skip reviews
Catch issues early
```

**Quality is not an accident. It's the result of systematic process.**

## Further reading (load when relevant)

When the orchestration involves significant context usage, long review loops, or complex validation checkpoints, load these references for the specific discipline:

- **`references/context-budget-discipline.md`** — Four-tier context degradation model (PEAK / GOOD / DEGRADING / POOR), read-depth rules that scale with context window size, and early warning signs of silent degradation. Load when a run will clearly consume significant context (multi-phase plans, many subagents, large artifacts).
- **`references/gates-taxonomy.md`** — The four canonical gate types (Pre-flight, Revision, Escalation, Abort) with behavior, recovery, and examples. Load when designing or reviewing any workflow that has validation checkpoints — use the vocabulary explicitly so each gate has defined entry, failure behavior, and resumption rules.
- **`references/design-delivery-gap-analysis.md`** — Root cause analysis of the "design spec → delivered code" deviation problem. Contains the 7 root causes, compounding effect model, confirmation bias analysis, and multi-agent pattern proposal. Load when designing review workflows or debugging persistent design-delivery gaps.
- **`references/aider-integration-setup.md`** — Full Aider setup guide: installation (with China mirror), wrapper script, model routing, `.aider.conf.yml`, usage patterns (single-shot, file-based, constrained edit), subagent dispatch templates, and remediation for common pitfalls. Load when implementing Aider as a coding sub-agent for Tier 0/1 features.

Both references adapted from gsd-build/get-shit-done (MIT © 2025 Lex Christopherson).
