---
name: external-proposal-evaluation
description: >-
  Evaluate an external proposal/pattern against existing internal systems.
  Systematic process: role mapping → existing system audit → feasibility
  assessment → integration recommendation → plan document.
  Output: gap analysis + go/no-go + integration plan.
version: 1.2.0
author: Hermes Agent
metadata:
  hermes:
    tags:
      - c007
      - advisory
      - proposal-evaluation
      - gap-analysis
      - integration-planning
    category: devops
---

# External Proposal Evaluation

> **Prerequisite**: This skill is a subclass of C007 Advisory Council work.
> Load `advisory-council-workflow` first for the general council process,
> then use this skill for the specific evaluation methodology.

## When to Use

Triggers — any of the following:
- User brings a structured proposal/playbook/pattern from outside and asks "how does this fit?"
- User shares a link (Qwen chat, blog, forum, etc.) containing a proposed approach
- User cites a blog post, paper, or another agent's approach and wants integration assessment
- A team member proposes a new workflow that may overlap with existing systems
- User asks "should we adopt this?" about an external methodology

Do NOT use for:
- Simple yes/no questions about an external tool (use `web_search`)
- Evaluating dependencies/packages (use `web_search` + `terminal`)
- Starting a new system from scratch (use `advisory-council-workflow` directly)

## Pre-Step: External Source Extraction

When the proposal comes as a link (not structured text):

1. **Browser-first attempt**: `browser_navigate(url)` → `browser_console(document.body.innerText)` to extract full text content
2. **Fallback**: If navigation fails, try `web_extract(url)`
3. **Reconstruction**: If content is fragmented (typically due to dynamic rendering), piece together by reading the extracted text sequentially — Qwen shared chat pages are usually one long conversation with markdown code blocks interspersed
4. **Identify the core proposal**: Look for the section where the AI responds to the user's actual question — this is the structured proposal you need to evaluate. Ignore preliminary search results and unrelated conversation history.

**Key challenges with Qwen share links:**
- Anti-bot protection may limit visible DOM — `browser_navigate` shows title only
- Content is rendered client-side — need `browser_console(document.body.innerText)` to bypass
- The conversation may contain multiple AI responses — find the one that answers the user's actual question
- Code blocks are typically rendered as plain text with a 3-backtick header visible

## Core Methodology (4-Step)

### Step 1: Role & Process Mapping

Map each proposed role/step to existing internal equivalents:

```markdown
| Proposed Role/Step | Existing Equivalent | Coverage | Gap |
|:-------------------|:--------------------|:--------:|:---:|
| Role A | System X's Role Y | ✅ Full | — |
| Role B | — | ❌ Missing | Needs new capability |
| Process C | System Z's Step W | ⚠️ Partial | Different output format |
```

**Coverage levels:**
- ✅ **Full** — same purpose, same output, different name only
- ⚠️ **Partial** — overlaps but has different scope/output/format
- ❌ **Missing** — no existing equivalent in any system

### Step 2: Existing System Audit

For each proposed function, check if existing infrastructure already supports it:

| Audit Target | What to Check |
|:-------------|:--------------|
| **Knowledge Base** (ChromaDB + Wiki) | Does KB content already cover this? What directory/source would it go in? |
| **Capability Registry** | Is this already registered as a capability? |
| **Skills** | Is there already a skill covering this pattern? |
| **Cron/Infrastructure** | Is there already a recurring pipeline? |
| **Advisory Council** | Does C007 already have a role covering this? |

**Key question**: "Is the storage/retrieval infrastructure already in place even if content is empty?" — This distinguishes "pipeline ready, needs content" from "nothing exists."

### Step 3: Integration Feasibility Assessment

Five integration approaches to evaluate:

| Approach | When to Choose | Cost | Risk |
|:---------|:---------------|:----:|:----:|
| **Merge** — Extend existing system | Existing system covers 60%+ of proposal | Low | May bloat existing system |
| **Parallel** — Run both independently | Proposal covers 30-60% and has unique value | Medium | Duplication risk |
| **Replace** — Deprecate old, adopt new | Proposal covers 90%+ and is strictly better | High | Migration cost |
| **Ignore** — Don't adopt | Proposal covers <30% or doesn't fit constraints | None | Missed opportunity |
| **Extract** — Adopt specific value points, reject framework | Proposal has 1-3 genuinely novel ideas but the overall framework is over-engineered/doesn't fit | Very low | May miss synergistic value |

**Decision framework:**
```
Proposal overlap with existing:
  ├─ <30% → Evaluate Ignore vs Extract (is there a kernel worth absorbing?)
  ├─ 30-60% → Evaluate Merge vs Parallel
  ├─ 60-90% → Merge is usually right
  └─ 90%+ → Consider Replace if strictly better
```

**Extract pattern** (new — most common with generic proposals from other agents):
```
Proposal is rejected as a whole (too many commands, too much state management,
doesn't fit architecture).
  ↓
But it has 1-3 genuinely valuable kernels:
  • Kernel A: e.g., progress visualization concept (adopt)
  • Kernel B: e.g., task dashboard overview (consider)
  • Kernel C: e.g., resume checkpoint (reject — already covered)
  ↓
For each adopted kernel, decide: behavioral convention vs code?
  • Behavioral convention (preferred): Add a section to an existing skill
    describing the pattern. Zero code, zero state files, zero commands.
    The agent commits to following the pattern in its responses.
  • Code implementation: Only when the behavior requires persistent state,
    cross-session data, or automation that cannot be done via convention.
  ↓
Implement the adopted kernels. Reject the framework container.
```

### Step 4: Produce Structured Plan Document

Output format (copy this template):

```markdown
# External Proposal Evaluation Report

## 1. Proposal Summary
- One-line description of the proposed approach
- Source (who brought it, where from)
- Key claims/benefits stated by the proposal

## 2. Role & Process Mapping
| Proposed | Existing | Coverage | Gap |
|----------|----------|:--------:|:----|
| ... | ... | ✅/⚠️/❌ | ... |

## 3. Existing Infrastructure Audit
| Area | Status | Detail |
|:-----|:------:|:-------|
| Knowledge Base | ✅/⚠️/❌ | What exists, what's missing |
| Capability Registry | ✅/⚠️/❌ | Already registered? |
| Skills | ✅/⚠️/❌ | Overlapping skills? |
| Cron/Pipelines | ✅/⚠️/❌ | Existing automated flows? |
| Advisory Council | ✅/⚠️/❌ | Role coverage? |

## 4. Integration Decision
- **Recommended approach**: Merge / Parallel / Replace / Ignore
- **Rationale**: Why this approach over others

## 5. Integration Plan (if Merge/Parallel)
- What to modify in existing systems
- What to create new
- What to leave unchanged
- Staged delivery (Phase 1/2/3)
- Risks and mitigations

## 6. User Decision Points
- List 2-4 questions requiring user confirmation
- Each with clear options
```

## Practical Example

See the advisory council report at:
`~/wiki/guides/advisory-council-reports/system-knowledge-sync-plan.md`
(Or Feishu doc `HXGKdkonWo7LEFxiWlFc0vCjnLc`)

This was a real evaluation of a "Knowledge Engineer" proposal against C005 v3.0, resulting in a system knowledge sync pipeline.

## Common Pitfalls

| Pitfall | Symptom | Fix |
|:--------|:--------|:----|
| **Role name match trap** | "They have a PM role, so does C007 → no gap" without checking actual function | Compare by **function** (what they do), not by **title** |
| **KB content vs KB pipeline confusion** | "KB can't support this" when pipeline exists but content is empty | Distinguish: infrastructure-ready vs content-filled |
| **Over-engineering the integration** | Proposing merge when the proposal adds <30% new value | Use the overlap % framework — if <30%, the answer is "ignore" or "extract" |
| **Skipping the "should we?" question** | Jumping straight to "how to integrate" | Run through C007's 4-stage Socratic process first (goal → constraints → tradeoffs → verify) |
| **Building code when convention suffices** | Implementing a new subsystem (state files, commands, CRUD endpoints) for something that could be a behavioral pattern in a skill | Default to **behavioral convention** (skill section + agent commitment to follow it). Only escalate to code when the behavior needs persistent cross-session state or automated execution. A good heuristic: if the external proposal uses 10+ commands and persistent state files for what is essentially "show progress", a 15-line skill section + behavioral commitment is better. |

## Relationship to Other Skills

- **advisory-council-workflow** — Parent skill. Use for the general C007 process (Socratic questions, internal review, Feishu archiving). This skill adds the specific evaluation methodology.
- **capability-registry-management** — Use to check if proposed capability is already registered. Use to register any newly identified gaps.
- **code-development-workflow** — After the evaluation produces a plan, hand off implementation to C005.
