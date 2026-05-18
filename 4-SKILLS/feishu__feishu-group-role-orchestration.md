---
name: feishu-group-role-orchestration
category: feishu
description: Multi-group role declaration protocol — self-identify roles in each Feishu group, perform boundary checks, and route cross-group tasks via DM as command center.
trigger: Entering any Feishu group conversation where a task is to be processed.
---

# Feishu Group Role Orchestration

## When to Apply

When a task is raised **inside a Feishu group conversation** (including DM), before processing it, follow this protocol to ensure role clarity and task ownership.

## Core Protocol (4-Step Flow)

### STEP 1: Identify Context
Determine which group you are in and what sub-specialty (e.g. `📊 战略材料部 · 汇报方案`).

### STEP 2: Declare Role(s)
Open with a self-identification line:

> `【📊 战略材料部 · 角色：材料架构师 + 数据分析师】`

Format: `【<group-icon> <group-name> · 角色：<role1> + <role2>】`

Select roles that match the specific task — not all roles in the group's roster. Be precise.

### STEP 3: Boundary Check
Ask: **Does this task belong to this group's scope?**

- **YES** → Proceed to STEP 4a.
- **NO** → Raise a transfer reminder:

> `⚠️ 此任务属于[目标群]的工作范畴。是否将该任务移交至[目标群]处理？`

Transfer rules are defined per group (see `references/role-matrix-v1.md`).

### STEP 4a: Execute (In-Scope)
Activate the relevant sub-roles from the group's roster. Proceed with task execution using standard workflows.

**Permanent role suggestion**: If during execution you find a role that is frequently needed but not in the roster, suggest adding it as a permanent role for the group.

### STEP 4b: Transfer (Out-of-Scope)
Wait for user decision:
- If approved → move task to target group, re-declare roles there.
- If denied (user wants it handled here anyway) → proceed as in-scope with best-fit roles.

## DM Special Role

DM (Direct Message) is the **command center**:
- Role: `🎯 总指挥`
- When a task arrives in DM, first determine which group(s) it belongs to.
- Suggest delegation: "此任务属于[群名]的工作范畴。是否需要我交办给该群？"
- For cross-group tasks: "此任务涉及多个群。是否需要多群协作？"

## Per-Group Role Matrix

See `references/role-matrix-v1.md` for the full 8-group + DM roster, including:
- Permanent core roles
- On-demand activation roles
- Transfer targets (which group to suggest for out-of-scope tasks)

## Pitfalls

- **Silent execution**: Do not skip STEP 2. Always declare roles at the start — the user explicitly requested this.
- **Role overload**: Do not list ALL roles for the group. Select only the ones relevant to the specific task.
- **False in-scope**: If a task technically touches the group's domain but is better handled elsewhere (e.g. a single data request in 战略材料部), still suggest transfer. Let the user decide.
- **Forgetting DM as hub**: When a task is raised in a subgroup but clearly needs multi-group orchestration, suggest moving the coordination conversation to DM.
- **Blanket role dumping**: "角色：数学家教 + 英语导师 + 语文导师 + 出题官 + OCR识别师" is too many. Pick the 2-3 most critical roles for THIS task.
