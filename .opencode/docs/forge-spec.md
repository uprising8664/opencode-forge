# Forge Specification

This document is the authoritative specification for the `.forge/` convention system used in opencode-forge. All agents (Architect, Smith, Scout, Warden) and commands (`/plan`, `/start-work`) reference this document as the single source of truth.

---

## Overview

`.forge/` is a directory-based planning and execution workflow for vanilla OpenCode. It provides a structured way to plan, track, and execute multi-step development work using four specialized agents and two user-facing commands — with no runtime code or external plugins required.

The workflow is governed entirely through agent prompts and file conventions documented here. The forge metaphor is intentional: the **Architect** designs the blueprint, the **Smith** hammers it into reality, the **Scout** investigates unknowns, and the **Warden** inspects the finished work.

Key principles:
- **Plans are immutable** — once written, a plan file cannot be changed. Mutable tracking is separated into a companion TODO file.
- **Two-file protocol** — every plan consists of a locked spec file and a mutable tracking file.
- **Wave-based execution** — tasks are grouped into waves; all tasks in a wave complete before the next begins.
- **Append-only notepads** — working memory accumulates; nothing is ever deleted mid-execution.
- **Verifiable outcomes** — every plan includes explicit QA scenarios and Smith writes evidence artifacts.

---

## Directory Structure

The `.forge/` directory lives at the root of the repository and contains all planning, tracking, and execution state.

```
.forge/
├── plans/           # Immutable plan specs (chmod 444 after Architect creates them)
│   └── {prefix}-manifest.md  # Multi-lane coordination manifest (not locked)
├── todos/           # Mutable progress tracking (one file per plan)
├── drafts/          # Architect working memory during planning (deleted after plan creation)
├── notepads/        # Append-only per-plan working memory
│   └── {plan-name}/
│       ├── learnings.md    # Patterns, conventions discovered
│       ├── issues.md       # Problems, gotchas encountered
│       ├── decisions.md    # Architectural choices made
│       └── problems.md     # Blocking issues needing resolution
├── evidence/        # QA verification artifacts
│   ├── task-{N}-{scenario-slug}.{ext}
│   └── final-qa/           # Final verification wave evidence
└── anvil.json       # Active execution state (gitignored)
```

### Subdirectory purposes

| Directory | Owner | Mutable? | Notes |
|-----------|-------|----------|-------|
| `plans/` | Architect | No | Plan files: chmod 444 after creation. Manifest files: not locked. |
| `todos/` | Smith | Yes | Checkboxes only |
| `drafts/` | Architect | Yes | Deleted after plan creation |
| `notepads/` | Smith | Append-only | Never overwrite existing content |
| `evidence/` | Smith | Yes | New files added; existing untouched |
| `anvil.json` | Smith | Yes | Gitignored; tracks live session state |

`.gitignore` must contain:
```
.forge/anvil.json
.forge/drafts/
```

---

## Plan File Format

Plan files live at `.forge/plans/{plan-name}.md` and are created by the Architect. After creation they are locked with `chmod 444`. Smith reads them but never modifies them.

A well-formed plan file follows this exact structure:

```markdown
# Plan: {Human-Readable Title}

**Plan name:** {plan-name}
**Created:** {ISO-8601 date}
**Agent:** architect

---

## TL;DR

One to three sentences describing what this plan accomplishes and why.

---

## Context

Background information the Smith needs to understand the problem domain, existing code structure, constraints, and any decisions already made. Include relevant file paths, API signatures, or architectural notes.

---

## Work Objectives

Numbered list of concrete deliverables:

1. Objective one — what must exist or work when this is done
2. Objective two
3. ...

---

## Verification Strategy

How to confirm the objectives are met. Written as testable scenarios.

### Scenario 1: {Scenario Name}
- **Given:** preconditions
- **When:** action taken
- **Then:** expected outcome
- **Evidence:** `.forge/evidence/task-N-{slug}.txt`

### Scenario 2: ...

---

## Execution Strategy

High-level approach for how Smith should implement the work. Not prescriptive line-by-line instructions, but enough context to make good decisions. May include: technology choices, patterns to follow, files to touch, things to avoid.

---

## TODOs

Tasks are grouped into numbered waves. All tasks in Wave N must complete before Wave N+1 begins.

### Wave 1 — {Wave Name}

| Key | Title | Type |
|-----|-------|------|
| todo:1 | {Task Title} | {implementation/research/qa} |
| todo:2 | {Task Title} | {implementation/research/qa} |

#### todo:1 — {Task Title}

**What:** Precise description of what this task produces. Concrete steps. Specify which tools to use.

**Must NOT do:**
- Explicit guardrails to prevent common errors
- Files that must not be touched
- Patterns that must not be introduced

**Agent profile:** Recommended agent category and brief reasoning (e.g., "unspecified-high — requires reading multiple files and writing implementation code")

**Parallelization:** Wave N. Blocks: [todo:X, todo:Y]. Blocked by: [todo:Z]. Or "No dependencies."

**References:**
- `path/to/file.ts` — why this file is relevant (existing pattern, type definitions, etc.)

**Acceptance:** Specific, verifiable condition that signals this task is done (e.g., "grep finds `export function login` in `src/auth.ts`", "npm test exits 0")

**QA Scenario:**
```
Scenario: {scenario name}
Tool: {Bash / Read / Grep}
Steps:
  1. {action}
  2. {check}
Expected: {outcome}
Evidence: .forge/evidence/task-N-{slug}.txt
```

**Commit:** `{type}({scope}): {message}`
Files: `path/to/changed/file.ts`, `path/to/other.ts`

#### todo:2 — {Task Title}

...

### Wave 2 — {Wave Name}

| Key | Title | Type |
|-----|-------|------|
| todo:3 | {Task Title} | {implementation/research/qa} |

...

---

## Final Verification Wave

Tasks Smith must run after all implementation waves complete. These confirm the full plan has been executed correctly.

| Key | Title |
|-----|-------|
| final-wave:F1 | {Verification Task Title} |
| final-wave:F2 | {Verification Task Title} |

#### final-wave:F1 — {Verification Task Title}

**What:** What to check and how.
**Evidence:** Path to write verification output.

---

## Commit Strategy

How to commit the work. Include:
- Commit message format
- Which files to stage
- Whether to push

---

## Success Criteria

Checklist of conditions that must all be true for the plan to be considered complete:

- [ ] Criterion one
- [ ] Criterion two
- [ ] All QA scenarios pass
- [ ] Final verification wave complete
```

---

## TODO File Format

TODO files live at `.forge/todos/{plan-name}.md`. This is the **only** file where Smith modifies checkboxes during execution. The plan file is never touched.

### Format

```markdown
# TODO: {Plan Name}

**Plan:** `.forge/plans/{plan-name}.md`
**Last updated:** {ISO-8601 timestamp}

---

## Wave 1 — {Wave Name}

- [ ] todo:1 — {Task Title}
- [ ] todo:2 — {Task Title}

## Wave 2 — {Wave Name}

- [ ] todo:3 — {Task Title}

## Final Verification Wave

- [ ] final-wave:F1 — {Verification Task Title}
- [ ] final-wave:F2 — {Verification Task Title}
```

### Rules

- Each task line follows the pattern: `- [ ] todo:N — Task Title`
- When a task completes: `- [x] todo:N — Task Title`
- Final verification tasks use `final-wave:FN` keys: `- [ ] final-wave:F1 — Task Title`
- Smith updates the `Last updated` timestamp whenever a checkbox changes
- **This file is the only mutable checkbox file.** The plan file (`.forge/plans/*.md`) is NEVER touched.
- The TODO file is created by the Architect at the same time as the plan file
- The TODO file is NOT locked (it remains writable so Smith can update it)

---

## Manifest File Format

Manifests coordinate multi-lane parallel plans. When work is split into two or more independent lanes that run in parallel, the Architect creates a manifest file to describe the lane structure, dependencies, and worktree setup.

- **Location:** `.forge/plans/{prefix}-manifest.md`
- **Created by:** the Architect when a plan is split into 2+ parallel lanes
- **Parsed by:** the `/setup-plan-worktrees` command to automate worktree creation
- **Not locked:** manifests are coordination documents, not immutable specs — they are NOT chmod 444

The manifest is created at the same time as the lane plan files and their TODO files. Each lane has its own plan file and TODO file following the same formats as single plans.

### Format

```markdown
# {Plan Name} - Manifest

## Overview
Coordinates parallel lane execution for {description}. Split into N lanes across M phases.

## Config
- **Prefix**: {prefix} | {name}
- **Project**: {project-name}
- **Integration Branch**: {prefix}-integration | none
- **Base Branch**: main

## Lane Structure

| Lane | Prefix | Plan File | Focus Area | Worktree Branch |
|------|--------|-----------|------------|-----------------|
| 1 | {prefix} | {prefix}-lane-1-{focus}.md | {description} | {prefix}-lane-1 |
| 2 | {prefix} | {prefix}-lane-2-{focus}.md | {description} | {prefix}-lane-2 |

> The `Prefix` column is optional. When all lanes share the Config prefix, the column may be omitted. When lanes have individual prefixes (e.g., subtask IDs), include it to override the Config prefix per lane.

## Dependency Graph

{ASCII art showing lane dependencies}

## Execution Order

### Phase 1 (Parallel — Start Immediately)
- **Lane 1**: {focus}
- **Lane 2**: {focus}

### Phase 2 (Sequential — After Phase 1)
- **Lane 3**: {focus} — depends on lanes 1 and 2

### Phase 3 (Merge)
- Merge all lane branches into integration branch or main

## Lanes

| Wave | Prefix | Plan File | Worktree | Base | Depends On |
|------|--------|-----------|----------|------|------------|
| 0 | {prefix} | {prefix}-lane-1-{focus}.md | {prefix}-lane-1 | main | — |
| 0 | {prefix} | {prefix}-lane-2-{focus}.md | {prefix}-lane-2 | main | — |
| 1 | {prefix} | {prefix}-lane-3-{focus}.md | {prefix}-lane-3 | integration | 1, 2 |

## Session Execution

{Instructions for running each lane in a separate OpenCode session}

## Success Criteria

- [ ] {criterion per lane}
- [ ] All lane branches merge cleanly

## Plan Status

| Lane | Prefix | Plan File | Status |
|------|--------|-----------|--------|
| 1 | {prefix} | {prefix}-lane-1-{focus}.md | 🔲 Created |
```

### Example

For a plan named `api-refactor` with prefix `PROJ-42`, split into two parallel lanes:

```markdown
# API Refactor - Manifest

## Overview
Coordinates parallel lane execution for API layer refactor. Split into 2 lanes across 2 phases.

## Config
- **Prefix**: PROJ-42
- **Project**: my-app
- **Integration Branch**: PROJ-42-integration
- **Base Branch**: main

## Lane Structure

| Lane | Plan File | Focus Area | Worktree Branch |
|------|-----------|------------|-----------------|
| 1 | PROJ-42-lane-1-routes.md | Route handlers | PROJ-42-lane-1 |
| 2 | PROJ-42-lane-2-services.md | Service layer | PROJ-42-lane-2 |

## Lanes

| Wave | Plan File | Worktree | Base | Depends On |
|------|-----------|----------|------|------------|
| 0 | PROJ-42-lane-1-routes.md | PROJ-42-lane-1 | main | — |
| 0 | PROJ-42-lane-2-services.md | PROJ-42-lane-2 | main | — |
```

When lanes map to individual subtasks with their own ticket IDs, use the per-lane `Prefix` column to override the Config prefix:

```markdown
## Config
- **Prefix**: PROJ-42
- **Integration Branch**: PROJ-42-integration
- **Base Branch**: main

## Lanes

| Wave | Prefix  | Plan File                    | Worktree       | Base | Depends On |
|------|---------|------------------------------|----------------|------|------------|
| 0    | PROJ-43 | PROJ-43-lane-1-routes.md     | PROJ-43-lane-1 | main | —          |
| 0    | PROJ-44 | PROJ-44-lane-2-services.md   | PROJ-44-lane-2 | main | —          |
```

### Rules

- The **Lanes table is the machine-readable contract** — it must include Wave, Plan File, Worktree, Base, and Depends On columns exactly as shown
- **Prefix** determines worktree directory and branch names. The Config `Prefix` is used for the manifest file name and integration branch. Each lane inherits the Config prefix by default. When the Lanes table includes an optional `Prefix` column, per-lane values override the Config prefix for that lane's plan file, worktree, and branch names (useful when lanes map to individual subtask tickets).
- **Base column values**: `main` (branch from main), `integration` (branch from integration branch), or a lane number (chained dependency)
- **Wave numbers** determine worktree creation order — wave 0 worktrees are created first, wave 1 after wave 0, etc.
- The manifest is created at the same time as the lane plan files and their TODO files
- Lane plan files follow the same format as single plans (see [Plan File Format](#plan-file-format) above)
- Each lane's TODO file follows the same format as single TODO files (see [TODO File Format](#todo-file-format) above)
- The manifest is **not locked** — it remains a mutable coordination document throughout the lifecycle

---

## Anvil State (anvil.json)

`anvil.json` lives at `.forge/anvil.json` and tracks the live execution state of the active plan. It is created or updated by Smith when execution begins and updated after each task completes. It is gitignored and ephemeral — it is not a record of history but a snapshot of current state.

### Complete JSON Schema

```json
{
  "active_plan": "string — path or name of the active plan",
  "started_at": "ISO-8601 timestamp",
  "plan_name": "string — short name matching the plan filename",
  "agent": "string — agent executing the plan (e.g. 'smith')",
  "session_ids": ["array of session ID strings"],
  "session_origins": {
    "session-id": "string — how the session started (e.g. 'user-started', 'resumed')"
  },
  "task_sessions": {
    "todo:N": {
      "task_key": "string — e.g. 'todo:1'",
      "task_label": "string — e.g. '1'",
      "task_title": "string — human-readable task name",
      "session_id": "string — session that worked on this task",
      "agent": "string — optional, agent category used",
      "category": "string — optional, task category",
      "updated_at": "ISO-8601 timestamp"
    }
  }
}
```

### Field descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `active_plan` | string | Yes | Relative path from repo root to the plan file, e.g. `.forge/plans/my-feature.md` |
| `started_at` | string | Yes | ISO-8601 timestamp when Smith first began executing this plan |
| `plan_name` | string | Yes | Short identifier matching the plan filename without extension, e.g. `my-feature` |
| `agent` | string | Yes | Agent role that is executing, always `"smith"` for normal execution |
| `session_ids` | array | Yes | All OpenCode session IDs that have been used to work on this plan |
| `session_origins` | object | Yes | Maps each session ID to a string explaining how it started |
| `task_sessions` | object | Yes | Maps task keys (e.g. `todo:1`) to session metadata objects |
| `task_sessions[key].task_key` | string | Yes | The task identifier, e.g. `todo:1` or `final-wave:F1` |
| `task_sessions[key].task_label` | string | Yes | Short numeric label, e.g. `"1"` |
| `task_sessions[key].task_title` | string | Yes | Human-readable title from the plan |
| `task_sessions[key].session_id` | string | Yes | The session ID that completed this task |
| `task_sessions[key].agent` | string | No | Agent category used for this task, e.g. `"unspecified-high"` |
| `task_sessions[key].category` | string | No | Task category label, e.g. `"implementation"` |
| `task_sessions[key].updated_at` | string | Yes | ISO-8601 timestamp when this task entry was last written |

### Example

```json
{
  "active_plan": ".forge/plans/auth-refactor.md",
  "started_at": "2026-04-08T10:00:00Z",
  "plan_name": "auth-refactor",
  "agent": "smith",
  "session_ids": ["ses_abc123", "ses_def456"],
  "session_origins": {
    "ses_abc123": "user-started",
    "ses_def456": "resumed"
  },
  "task_sessions": {
    "todo:1": {
      "task_key": "todo:1",
      "task_label": "1",
      "task_title": "Extract auth middleware",
      "session_id": "ses_abc123",
      "agent": "unspecified-high",
      "category": "implementation",
      "updated_at": "2026-04-08T10:30:00Z"
    }
  }
}
```

---

## Notepad Convention

Notepads live at `.forge/notepads/{plan-name}/` and provide append-only working memory for Smith during plan execution. They accumulate observations, learnings, and problems — nothing is ever deleted or overwritten.

### The four notepad files

| File | Purpose |
|------|---------|
| `learnings.md` | Patterns, conventions, and useful facts discovered during execution |
| `issues.md` | Problems and gotchas encountered (and how they were resolved, if resolved) |
| `decisions.md` | Architectural and implementation choices made, with rationale |
| `problems.md` | Blocking issues that remain unresolved; things that need follow-up |

### Rules

1. **Append-only**: Never overwrite existing content. Never use Edit to replace lines. Only add new sections at the end.
2. **One section per task entry**: Each time Smith writes to a notepad, it adds a new `##` section.
3. **Timestamp every entry**: The section heading includes the ISO-8601 date and the task ID.
4. **Write during execution**: Smith writes to notepads while working on tasks, not only at the end.

### Entry format

```
## [TIMESTAMP] Task: {task-id}
{content here}
```

Example:

```markdown
## [2026-04-08T10:45:00Z] Task: todo:2

Found that the existing auth module uses a non-standard token format.
The `verifyToken()` function expects base64url but the legacy endpoint
sends plain base64. Created a compatibility shim in `src/auth/compat.ts`.
```

### When to write what

- **learnings.md**: Write when you discover something non-obvious about the codebase, a pattern that will affect future tasks, or a convention you had to infer.
- **issues.md**: Write when you encounter an error, unexpected behavior, or a constraint that blocked progress. Include the fix if found.
- **decisions.md**: Write when you make a choice between two or more valid approaches. Record the option chosen and why.
- **problems.md**: Write when something is blocking and you cannot resolve it in the current task. These should be reviewed at the end of execution.

---

## Two-File Protocol

Every plan in `.forge/` is represented by exactly two files: an immutable plan spec and a mutable tracking file. This separation is fundamental to the forge workflow.

For multi-lane plans, the Architect also creates a manifest file — see [Manifest File Format](#manifest-file-format) below.

### What the Architect creates

When `/plan` or `@architect` is invoked, the Architect creates exactly two files:

1. **`.forge/plans/{name}.md`** — the full plan spec (TL;DR, Context, Work Objectives, Verification Strategy, Execution Strategy, TODOs, Final Verification Wave, Commit Strategy, Success Criteria)
2. **`.forge/todos/{name}.md`** — the tracking file with one checkbox per task

These two files always have the same base name. If the plan is `auth-refactor`, both files are named `auth-refactor`.

For multi-lane plans, the Architect additionally creates `.forge/plans/{prefix}-manifest.md` — a coordination document listing all lanes, their dependencies, and worktree setup metadata.

### Locking the plan

Immediately after writing both files, the Architect runs:

```bash
chmod 444 .forge/plans/{name}.md
```

This makes the plan file read-only at the filesystem level. It cannot be accidentally modified by any tool, shell command, or agent.

### What Smith may and must not touch

**Smith MAY modify:**
- `.forge/todos/*.md` — updating checkboxes as tasks complete
- `.forge/anvil.json` — updating execution state
- `.forge/notepads/**/*` — appending working memory entries
- `.forge/evidence/**/*` — writing new QA evidence files
- All other project files (source code, tests, configs)

**Smith MUST NEVER modify:**
- `.forge/plans/*.md` — these are read-only by convention and by filesystem permission

### Why this matters

If a plan could be modified mid-execution, the relationship between what was planned and what was built becomes ambiguous. Separating the immutable spec from the mutable tracking file means:

- The plan is always a reliable record of intent
- Checkboxes in the TODO file are the canonical record of progress
- The Warden can compare the original plan to the implementation without uncertainty

---

## Plan Immutability

Plans are immutable after creation. This is a hard rule with no exceptions during normal execution.

### Why plans cannot change

A plan represents a commitment about scope, approach, and acceptance criteria. If the plan changes mid-execution:

- Evidence written against old scenarios becomes invalid
- The Warden cannot determine whether the implementation matches the plan
- Partially-completed tasks may be retroactively redefined

### What to do when the plan is wrong

If Smith discovers the plan is incorrect, incomplete, or based on faulty assumptions, Smith **must stop execution** and tell the user:

> "The plan needs revision. I cannot continue with the current plan because [reason]. To proceed, the user should re-invoke Architect with the updated requirements."

The revision protocol is:

1. Smith **stops** — does not implement a workaround or deviate from the plan
2. Smith **documents** the problem in `.forge/notepads/{plan-name}/problems.md`
3. User re-invokes the Architect with updated context
4. Architect creates a **new plan** with a new name (e.g., `auth-refactor-v2`)
5. The old plan remains untouched as a record

### Minor clarifications vs. plan changes

If the plan is ambiguous but not wrong (e.g., a file path is not specified), Smith may make a reasonable interpretation and record the decision in `decisions.md`. This is not a plan change — it is implementation detail.

If the acceptance criteria or scope need to change, that is a plan change and requires re-invoking Architect.

---

## Evidence Convention

Evidence files are written by Smith to `.forge/evidence/` as proof that QA scenarios passed. They are referenced by the plan's Verification Strategy section and checked by the Warden during final review.

### Naming convention

Regular task evidence:
```
.forge/evidence/task-{N}-{scenario-slug}.{ext}
```

Where:
- `{N}` is the task number (matching `todo:N`)
- `{scenario-slug}` is a short kebab-case description of the scenario
- `{ext}` is `txt` for shell output, `json` for JSON data, `md` for structured reports

Examples:
```
.forge/evidence/task-1-config-valid.txt
.forge/evidence/task-2-api-returns-200.txt
.forge/evidence/task-3-schema-matches.json
```

### Final QA evidence

Evidence from the final verification wave lives in:
```
.forge/evidence/final-qa/
```

Examples:
```
.forge/evidence/final-qa/full-test-suite.txt
.forge/evidence/final-qa/integration-smoke.txt
.forge/evidence/final-qa/lsp-clean.txt
```

### Content requirements

Each evidence file must contain:
- The command or check that was run
- The actual output
- A pass/fail verdict

Example:
```
Command: npm test -- --grep "auth"
Exit code: 0

Output:
  auth middleware
    ✓ rejects requests without token (12ms)
    ✓ accepts valid JWT (8ms)
    ✓ rejects expired JWT (5ms)

  3 passing (45ms)

VERDICT: PASS
```

### Rules

- Evidence files are write-once — never overwrite or edit existing evidence
- If a scenario fails, write a new evidence file (e.g., `task-1-config-valid-retry.txt`) after fixing
- The plan's Verification Strategy section specifies the expected evidence file path; Smith must write to that exact path

---

## Wave-Based Execution

Plans organize tasks into numbered waves. Wave-based execution ensures that foundational work completes before dependent work begins.

### How waves work

1. Each plan's TODO section groups tasks under wave headings: `### Wave 1 — {Name}`, `### Wave 2 — {Name}`, etc.
2. Smith executes all tasks in Wave 1 before beginning Wave 2
3. A wave is complete when all its checkboxes in `.forge/todos/{plan-name}.md` are checked
4. The final verification wave runs after all numbered waves complete

### Default mode: sequential

By default, Smith executes tasks one at a time within a wave:

1. Read the task description from the plan
2. Implement the task
3. Run QA scenarios, write evidence
4. Update the TODO checkbox
5. Update `anvil.json`
6. Move to the next task

This is the safe default. It prevents conflicts when tasks share files.

### Parallel mode: concurrent subagents

When tasks within a wave are explicitly independent (they touch different files, have no shared state, and are marked as parallelizable in the plan), Smith may dispatch them as concurrent subagents using the `task()` tool.

To dispatch tasks in parallel:
- All tasks must be in the same wave
- They must not share any files being created or modified
- The plan's Execution Strategy should indicate they can run in parallel
- Smith collects all results before marking the wave complete and proceeding

### Wave completion criteria

A wave is considered complete when:
- All tasks in the wave have `[x]` checkboxes in the TODO file
- All required evidence files for those tasks exist in `.forge/evidence/`
- No unresolved blocking issues exist in `.forge/notepads/{plan-name}/problems.md`

Smith must not proceed to the next wave if any of these conditions are unmet.

---

## Lifecycle

The full lifecycle of a forge plan, from initiation to Warden approval:

1. **User initiates planning** — runs `/plan [topic]` or `@architect [topic]` to begin the planning session.

2. **Architect interviews** — gathers requirements through conversation. Uses `.forge/drafts/{name}-draft.md` as working memory while structuring the plan.

3. **Architect writes plan files** — creates `.forge/plans/{name}.md` (full plan spec) and `.forge/todos/{name}.md` (tracking file) simultaneously. For multi-lane plans, the Architect creates one plan file per lane, one TODO file per lane, and a manifest file (`.forge/plans/{prefix}-manifest.md`).

4. **Architect locks the plan** — runs `chmod 444 .forge/plans/{name}.md` immediately after writing the plan file.

5. **Architect cleans up and hands off** — deletes the draft file from `.forge/drafts/`, then tells the user: "Plan created. Run `/start-work` or `@smith` to begin execution."

6. **User begins execution** — runs `/start-work` or `@smith`. Smith reads `.forge/anvil.json` if it exists to check for an in-progress plan, or starts fresh.

7. **Smith initializes anvil** — creates or updates `.forge/anvil.json` with the active plan name, session ID, and start timestamp.

8. **Smith executes tasks** — for each task in the current wave:
   a. Reads the task description from `.forge/plans/{name}.md`
   b. Implements the task (writes code, creates files, etc.)
   c. Runs the QA scenarios described in the plan's Verification Strategy
   d. Writes evidence to `.forge/evidence/task-N-{slug}.txt`
   e. Updates the checkbox in `.forge/todos/{name}.md`
   f. Updates `task_sessions` in `.forge/anvil.json`

9. **Smith writes to notepads** — throughout execution, appends observations to the appropriate notepad file in `.forge/notepads/{plan-name}/`.

10. **Smith completes all waves** — after all numbered waves complete, runs the Final Verification Wave tasks. Writes final evidence to `.forge/evidence/final-qa/`.

11. **Smith hands off to Warden** — tells the user: "All tasks complete. Run `@warden` for final review."

12. **User runs Warden review** — `@warden` reads the plan, the TODO file, all evidence, and the implementation. Outputs a structured verdict: **APPROVE** (all success criteria met) or **REJECT** (with specific issues to fix).

13. **On REJECT** — Smith addresses the Warden's findings, updates evidence, and the user re-runs `@warden`.

14. **On APPROVE** — execution is complete. The plan file, TODO file, evidence, and notepads remain as a permanent record of the work.

---

## Agent Roles Reference

| Agent | Invocation | Responsibility |
|-------|-----------|----------------|
| **Architect** | `/plan` or `@architect` | Interviews user, writes plan + TODO file, locks plan, cleans up drafts |
| **Smith** | `/start-work` or `@smith` | Reads plan, executes tasks, writes evidence, updates TODO + anvil + notepads |
| **Scout** | `@scout` | Investigates unknowns: researches APIs, explores codebases, answers Architect/Smith questions |
| **Warden** | `@warden` | Reviews completed work against the plan; outputs APPROVE or REJECT verdict |

### Architect responsibilities
- Create complete, unambiguous plan files with actionable task descriptions
- Define verifiable acceptance criteria for every task
- Write TODO files that mirror the plan's task list exactly
- Lock plan files immediately after creation
- Remove draft files before handing off
- Create manifest files for multi-lane plans with correct Lanes table format

### Smith responsibilities
- Never modify `.forge/plans/*.md`
- Execute tasks in wave order
- Write evidence for every QA scenario
- Keep anvil.json current throughout execution
- Append to notepads; never overwrite
- Stop and escalate when the plan is wrong (do not improvise)

### Scout responsibilities
- Answer specific questions delegated by Architect or Smith
- Research external documentation, APIs, or existing code
- Return findings in a structured format so the delegating agent can continue

### Warden responsibilities
- Read the original plan — not the implementation — as the source of truth
- Check every success criterion from the plan's Success Criteria section
- Verify every evidence file exists and contains a PASS verdict
- Output a clear APPROVE or REJECT with line-item reasoning
- Never approve work that has missing evidence or unmet criteria
