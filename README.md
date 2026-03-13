# zero-touch

> A single command takes a feature from idea to committed code — fully automated.

A Claude Code skill that drives a 7-stage autonomous pipeline with Opus oversight at every architectural gate, parallel Sonnet agents implementing against Opus-authored contracts, and a self-correcting retry loop that classifies failures before escalating to you.

```bash
/zero-touch "add stripe webhook handler"
/zero-touch --ticket GH-142
```

## How It Works

```
Stage 0  Design       Opus     Socratic clarity loop → approach selection
Stage 1  Environment  Haiku    Git status, conflict scan, branch protection check
Stage 2  Planning     Opus     Architecture → unit contracts per file/component
Stage 3  Implement    Sonnet×N Parallel agents, each working against an Opus contract
Stage 4  Validation   Haiku    Tests, type check, lint, acceptance criteria
Stage 5  Review       Opus     Security, blast radius, code quality gate
Stage 6  Commit       Haiku    Conventional commit + PR or direct merge
```

**You interact twice**: once in Stage 0 (what to build and which approach), and only again if a correction budget is exhausted. Everything else is autonomous.

## Stage 0: Design Intelligence

Stage 0 is a two-phase design conversation — fully built-in, no external dependencies.

**Phase A — Clarity loop**: Opus-driven Socratic questions targeting the weakest clarity dimension (goal, constraints, acceptance criteria, codebase context) until ambiguity drops below 20%. Mathematical scoring with challenge agents at rounds 4, 6, and 8 to surface hidden assumptions.

**Phase B — Approach selection**: Opus proposes 2-3 architectural approaches with trade-offs. You pick one. Opus then authors per-unit contracts that Sonnet agents implement against.

## Self-Correcting Pipeline

Failures at any validation gate are classified before retrying:

| Type | Trigger | Fix Strategy |
|---|---|---|
| `SYNTAX` | Parse/compile error | Sonnet self-corrects (max 2/unit) |
| `LOGIC` | Test failure | Haiku diagnoses → Sonnet re-implements |
| `ARCH` | Contract violation | Opus revises contract → Sonnet retries |
| `AMBIGUOUS` | Unclear failure | Opus arbitrates contract vs implementation |

Per-unit budgets prevent infinite loops. When budgets exhaust, the pipeline surfaces a clear escalation report rather than silently degrading.

## Git Worktree Isolation

Each feature runs in its own git worktree at `.zero-touch/worktrees/{slot}/`. Multiple features can run simultaneously in separate terminal sessions without branch conflicts. Worktrees are cleaned up automatically after commit or cancellation.

## State Management

Zero-Touch is fully self-contained. All state lives under `.zero-touch/` in your project:

```
.zero-touch/
├── config.json                         # Project delivery config (PR vs direct merge)
├── worktrees/
│   ├── pool.json                       # Slot pool for concurrent features
│   └── {slot}/                         # Git worktrees, one per feature
└── state/
    └── {run-id}/
        ├── interview-state.json        # Stage 0 Phase A clarity loop progress
        ├── interview-spec.md           # Crystallized feature spec
        ├── design.json                 # Goals, constraints, chosen approach
        ├── feature-plan.json           # Stage 2 unit decomposition
        ├── contracts/{unit-id}.json    # Opus-authored per-unit contracts
        ├── corrections.json            # Correction budget tracking
        └── validation-report.json     # Stage 4 output for Stage 5 reference
```

No external tools, no MCP servers, no plugin dependencies. Only `git` and `gh` CLI required.

## Prerequisites

1. **`git`** — project must be a git repository (or pass `--init`)
2. **`gh` CLI authenticated** — for PR creation and branch protection checks
   ```bash
   gh auth status
   ```

That's it.

## Installation

```bash
# Via skills CLI
npx skills add Guipetris/zero-touch

# Or manually
cp SKILL.md ~/.claude/skills/zero-touch/SKILL.md
```

## Flags

| Flag | Description |
|---|---|
| `--ticket {id}` | Fetch GitHub issue (`GH-142` or `142`) and skip Stage 0 |
| `--init` | Allow `git init` if the project has no repository |
| `--resume {run-id}` | Explicitly resume a specific interrupted run |

## Cancellation

Create `.zero-touch/CANCEL` at any time to cancel the active pipeline:

```bash
touch .zero-touch/CANCEL
```

The pipeline checks for the sentinel at the start of each stage, releases the worktree slot, archives the branch, and cleans up state.

## Configuration

On first run, Zero-Touch asks two questions and writes answers to `.zero-touch/config.json`:

- **Delivery mode**: PR to target branch (recommended) or direct merge
- **Target branch**: auto-detected (main/master) or specify

Config is project-local and added to `.gitignore` automatically.

## Ticket-Driven Mode

```bash
/zero-touch --ticket GH-142
```

Fetches the issue title, body, and labels via `gh issue view`, synthesizes `design.json` automatically, and skips Stage 0 entirely — straight to environment check and planning.

## License

Apache 2.0 — see [LICENSE](LICENSE).
