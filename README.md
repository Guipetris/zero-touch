# the-shaman-pipe

> Light the pipe. One command. Seven stages. Committed code.

*The pipe is also a pipeline. The shaman was informed. The shaman does not care.*

A Claude Code skill that drives a 7-stage autonomous pipeline with Opus oversight at every architectural gate, parallel Sonnet agents implementing against Opus-authored contracts, and a self-correcting retry loop that classifies failures before escalating to you.

```bash
/the-shaman-pipe "add stripe webhook handler"
/the-shaman-pipe --ticket GH-142
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

**You interact twice by default**: once in Stage 0 (what to build and which approach), and only again if a correction budget is exhausted. Everything else is autonomous. Enable `checkpoint_mode` to add optional pause points between stages — useful when you want to review the plan or contracts before they become irreversible.

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

Each feature runs in its own git worktree at `.shaman-pipe/worktrees/{slot}/`. Multiple features can run simultaneously in separate terminal sessions without branch conflicts. Worktrees are cleaned up automatically after commit or cancellation.

## State Management

The Shaman Pipe is fully self-contained. All state lives under `.shaman-pipe/` in your project (the directory is named after the pipe — the ceremonial one, and also the pipeline, yes, both):

```
.shaman-pipe/
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
npx skills add Guipetris/the-shaman-pipe

# Or manually
cp SKILL.md ~/.claude/skills/the-shaman-pipe/SKILL.md
```

## Flags

| Flag | Description |
|---|---|
| `--ticket {id}` | Fetch GitHub issue (`GH-142` or `142`) and skip Stage 0 |
| `--init` | Allow `git init` if the project has no repository |
| `--resume {run-id}` | Explicitly resume a specific interrupted run |

## Cancellation

Create `.shaman-pipe/CANCEL` at any time to cancel the active pipeline:

```bash
touch .shaman-pipe/CANCEL
```

The pipeline checks for the sentinel at the start of each stage, releases the worktree slot, archives the branch, and cleans up state.

## Checkpoints (optional)

Enable `checkpoint_mode` during first-run setup to add interactive pause points between every stage:

```
Stage N completes → summary shown → user confirms → Stage N+1 begins
```

At each checkpoint you get three options:

| Option | Effect |
|---|---|
| Proceed to Stage N | Continue pipeline normally |
| Pause — resume later | Halt gracefully; resume with `/the-shaman-pipe --resume {run-id}` |
| Cancel pipeline | Same as `.shaman-pipe/CANCEL` — clean teardown |

Checkpoints are designed for developers who want to inspect the Opus-authored contracts before implementation, or review the validation report before triggering the security review. The pipeline is fully autonomous by default — checkpoints are opt-in.

## Configuration

On first run, The Shaman Pipe asks three questions and writes answers to `.shaman-pipe/config.json`:

- **Delivery mode**: PR to target branch (recommended) or direct merge
- **Target branch**: auto-detected (main/master) or specify
- **Checkpoint mode**: fully autonomous (default) or pause between each stage for review

Config is project-local and added to `.gitignore` automatically.

## Ticket-Driven Mode

```bash
/the-shaman-pipe --ticket GH-142
```

Fetches the issue title, body, and labels via `gh issue view`, synthesizes `design.json` automatically, and skips Stage 0 entirely — straight to environment check and planning.

## License

Apache 2.0 — see [LICENSE](LICENSE).
