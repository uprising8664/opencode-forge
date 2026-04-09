# OpenCode Forge

A plugin-agnostic planning and execution workflow for vanilla OpenCode. Pure markdown and JSONC — zero runtime code, zero dependencies.

---

## What is OpenCode Forge?

OpenCode Forge encodes structured planning and execution conventions directly into agent prompts and commands. It gives you a repeatable workflow for breaking down complex work into plans, executing them task-by-task, and verifying the results — without requiring any plugins, runtime code, or external tooling.

It works through four specialized agents and two slash commands that coordinate via the `.forge/` directory convention. Everything is markdown and JSONC configuration. There are no `.ts`, `.js`, or `.py` files.

---

## Quick Start

1. **Clone this repo** (or copy the `.opencode/` directory into an existing project):
   ```bash
   cp -r opencode-forge/.opencode /your-project/
   ```

2. **Plan your work** — run `/plan` to start a planning session with Architect:
   ```
   /plan add user authentication
   ```

3. **Execute the plan** — run `/start-work` to begin implementation with Smith:
   ```
   /start-work
   ```

4. **Review the results** — invoke Warden for a structured code review:
   ```
   @warden
   ```

---

## One-Shot Installation Prompt

Open OpenCode in your project, start a new session, and paste this prompt. It will install Forge and confirm the setup:

```
Please install OpenCode Forge into this project by running the following steps:
Run this command to copy the Forge configuration into the project:
   git clone https://github.com/uprising8664/opencode-forge.git /tmp/opencode-forge && cp -r /tmp/opencode-forge/.opencode ./.config && rm -rf /tmp/opencode-forge
```

---

## Agents

### Architect — Strategic Planner

| | |
|---|---|
| **File** | `.opencode/agents/architect.md` |
| **Model** | `github-copilot/claude-opus-4.6` · Temperature 0.3 |
| **Mode** | Primary |
| **Invoke** | `/plan [topic]` or `@architect` |
| **Permissions** | Full access |

Architect interviews you in structured rounds — objectives and scope, technical constraints, then verification strategy — before generating any files. It maintains working drafts during the interview and transitions to plan generation only when all five clearance criteria are met.

Outputs: an immutable plan at `.forge/plans/{name}.md` (locked with `chmod 444`) and a mutable TODO tracker at `.forge/todos/{name}.md`.

### Smith — Task Executor

| | |
|---|---|
| **File** | `.opencode/agents/smith.md` |
| **Model** | `github-copilot/claude-sonnet-4.6` · Temperature 0.2 |
| **Mode** | Primary (default agent) |
| **Invoke** | `/start-work [plan-name]` or `@smith` |
| **Permissions** | Full access |

Smith reads plans and implements them task-by-task, wave-by-wave. For each task, Smith reads the spec, implements the work, runs every QA scenario, writes evidence artifacts, checks off the TODO, and updates `anvil.json` session state.

Supports session resumption — if interrupted, `/start-work` picks up where Smith left off by reading `anvil.json` and finding the first unchecked task. Optionally dispatches parallel subagents for independent tasks within a wave.

### Scout — Read-Only Researcher

| | |
|---|---|
| **File** | `.opencode/agents/scout.md` |
| **Model** | `github-copilot/claude-haiku-4.5` · Temperature 0.1 |
| **Mode** | Subagent |
| **Invoke** | `@scout [question]` |
| **Permissions** | Read-only (write, edit, task, todowrite denied; bash requires approval) |

Scout gathers information and reports findings. It explores codebases with `read`, `glob`, and `grep`; searches the web with `webfetch` and `websearch`; and analyzes patterns across files. Every finding includes source, evidence, and relevance.

Scout does not write files, generate code, or suggest fixes. It reports what it finds.

### Warden — Read-Only Reviewer

| | |
|---|---|
| **File** | `.opencode/agents/warden.md` |
| **Model** | `github-copilot/claude-sonnet-4.6` · Temperature 0.1 |
| **Mode** | Subagent |
| **Invoke** | `@warden` or `@warden review {plan-name}` |
| **Permissions** | Strictest (only read, glob, grep, question allowed; everything else denied) |

Warden runs a 5-check review protocol against the plan that specified the work:

1. **Plan Compliance** — Are all success criteria and work objectives satisfied?
2. **TODO Completeness** — Are all checkboxes marked done?
3. **Code Quality** — Type safety, error handling, debug artifacts, hardcoded values?
4. **Scope Fidelity** — Missing work or scope creep vs. the plan?
5. **Evidence** — Do all referenced evidence files exist and contain `VERDICT: PASS`?

Warden issues a binary verdict: **APPROVE** or **REJECT**. No conditional approvals. If rejected, Smith fixes the issues and Warden reviews again.

---

## Commands

### `/plan [topic]`

Routes to Architect. Starts a structured planning session.

```
/plan refactor the database layer
/plan add OAuth2 login flow
/plan migrate from REST to GraphQL
```

Architect interviews you, then generates `.forge/plans/{name}.md` and `.forge/todos/{name}.md`.

### `/start-work [plan-name]`

Routes to Smith. Begins or resumes plan execution.

```
/start-work                    # Resume active plan (reads anvil.json)
/start-work auth-migration     # Start a specific plan
```

If `anvil.json` exists, Smith resumes from the last incomplete task. If no plans exist, Smith tells you to run `/plan` first.

### `/setup-plan-worktrees <manifest-file>`

Sets up git worktrees from a plan manifest for parallel lane execution. Routes to Smith.

```
/setup-plan-worktrees PROJ-42-manifest.md
```

When Architect creates multi-lane parallel plans, it generates a manifest at `.forge/plans/{prefix}-manifest.md`. This command reads the manifest and:

1. Creates an integration branch (if specified)
2. Creates git worktrees for each lane as siblings to the main worktree
3. Distributes plan files to each worktree's `.forge/plans/` directory

After setup, open a new OpenCode session in each wave-0 worktree and run `/start-work`.

---

## The `.forge/` Directory

All planning, tracking, and execution state lives in `.forge/` at the repository root.

```
.forge/
├── plans/           # Immutable plan specs (chmod 444). Also contains manifest files (`*-manifest.md`) for multi-lane coordination.
├── todos/           # Mutable progress tracking (checkboxes)
├── drafts/          # Architect working memory (temporary, deleted after plan creation)
├── notepads/        # Append-only per-plan working memory
│   └── {plan-name}/
│       ├── learnings.md    # Patterns and conventions discovered
│       ├── issues.md       # Errors and blockers encountered
│       ├── decisions.md    # Choices made during execution
│       └── problems.md     # Unresolved blocking issues
├── evidence/        # QA verification artifacts
│   └── final-qa/   # Final verification wave evidence
└── anvil.json       # Active execution state (gitignored)
```

**Git tracking**: `.forge/plans/` is tracked in git — plans are the source of truth and worth preserving. Everything else (`todos/`, `drafts/`, `notepads/`, `evidence/`, `anvil.json`) is gitignored as working state.

See `.opencode/docs/forge-spec.md` for the complete specification of file formats, lifecycle rules, and conventions.

---

## Workflow

A typical session follows this cycle:

```
  ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
  │  /plan   │────▶│ Execute │────▶│ Review  │────▶│  Done   │
  │@architect│     │ @smith  │     │ @warden │     │         │
  └─────────┘     └─────────┘     └─────────┘     └─────────┘
       │               │               │
       │               │          REJECT? ──▶ @smith fixes ──▶ @warden again
       │               │
       │          @scout as needed
       │
  Interview rounds
  (objectives → constraints → verification)
```

1. **Plan** — `/plan` or `@architect` interviews you about scope, constraints, and verification. Generates an immutable plan and a mutable TODO tracker.
2. **Execute** — `/start-work` or `@smith` works through tasks wave-by-wave. Each task is implemented, QA'd, evidenced, and checked off before moving to the next.
3. **Research** — `@scout` answers questions at any point. Use it during planning to investigate the codebase, or during execution when Smith needs context about an unfamiliar library.
4. **Review** — `@warden` runs the 5-check protocol. Returns APPROVE or REJECT with specific findings.
5. **Fix** — If Warden rejects, Smith addresses the specific issues, then Warden reviews again.
6. **Resume** — If a session is interrupted, `/start-work` resumes from where Smith left off. Anvil state tracks the active plan and completed tasks.

### Multi-Lane Parallel Execution

For large tasks, Architect can split work into parallel lanes — each with its own plan, worktree, and OpenCode session:

1. **Plan** — `/plan` creates multiple lane plans plus a manifest
2. **Set up worktrees** — `/setup-plan-worktrees <manifest>` creates isolated worktrees per lane
3. **Execute in parallel** — Run `/start-work` in each worktree simultaneously
4. **Merge** — Merge lane branches when all lanes complete
5. **Review** — `@warden` reviews the merged result

---

## Configuration

### Models

Change `model` and `fallback_models` in `.opencode/opencode.jsonc` under each agent key:

```jsonc
"architect": {
  "model": "github-copilot/claude-opus-4.6",
  "fallback_models": ["github-copilot/claude-sonnet-4.6", "github-copilot/gpt-4o"]
}
```

### Permissions

Adjust the `permission` block in each agent's `.md` frontmatter (`.opencode/agents/*.md`):

```yaml
permission:
  read: allow
  write: deny    # deny, allow, or ask
  edit: deny
  bash: ask      # "ask" prompts the user for approval each time
```

### Agent Behavior

Edit the markdown body of any agent file in `.opencode/agents/`. The frontmatter controls config; the markdown body controls behavior.

### Default Agent

Change which agent handles unrouted messages in `.opencode/opencode.jsonc`:

```jsonc
"default_agent": "smith"
```

---

## Project Structure

```
.opencode/
├── opencode.jsonc           # Agent config — models, modes, temperatures, fallbacks
├── agents/
│   ├── architect.md         # Planner agent prompt
│   ├── smith.md             # Executor agent prompt
│   ├── scout.md             # Research agent prompt
│   └── warden.md            # Reviewer agent prompt
├── commands/
│   ├── plan.md              # /plan → routes to Architect
│   ├── start-work.md        # /start-work → routes to Smith
│   └── setup-plan-worktrees.md  # /setup-plan-worktrees → routes to Smith
└── docs/
    └── forge-spec.md        # .forge/ convention specification
```

---

## License

MIT
