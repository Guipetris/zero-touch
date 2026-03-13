# zero-touch

> A single command takes a feature from idea to committed code — fully automated.

A Claude Code skill that chains a 7-stage autonomous pipeline with Opus oversight at every architectural gate, Sonnet agents implementing in parallel against Opus-authored contracts, and a self-correcting retry loop that classifies failures before escalating to you.

```bash
/zero-touch "add stripe webhook handler"
/zero-touch --ticket GH-142
```

## How It Works

```
Stage 0  Design       Opus     Socratic interview → approach selection
Stage 1  Environment  Haiku    Git status, conflict scan, branch protection check
Stage 2  Planning     Opus     Architecture → unit contracts per file/component
Stage 3  Implement    Sonnet×N Parallel agents, each working against an Opus contract
Stage 4  Validation   Haiku    Tests, type check, lint, acceptance criteria
Stage 5  Review       Opus     Security, blast radius, code quality gate
Stage 6  Commit       Haiku    Conventional commit + PR or direct merge
```

**You interact twice**: once in Stage 0 (what to build and which approach), and only again if a correction budget is exhausted and the pipeline needs guidance. Everything else is autonomous.

## Self-Correcting Pipeline

Failures at any validation gate are classified before retrying:

| Type | Trigger | Fix Strategy |
|---|---|---|
| `SYNTAX` | Parse/compile error | Sonnet self-corrects (max 2/unit) |
| `LOGIC` | Test failure | Haiku diagnoses → Sonnet re-implements |
| `ARCH` | Contract violation | Opus revises contract → Sonnet retries |
| `AMBIGUOUS` | Unclear failure | Opus arbitrates contract vs implementation |

Per-unit budgets prevent infinite loops. When budgets exhaust, the pipeline surfaces a clear escalation prompt rather than silently degrading.

## Git Worktree Isolation

Each feature runs in its own git worktree at `.zero-touch/worktrees/{slot}/`. Multiple features can run simultaneously in separate terminal sessions without branch conflicts. Worktrees are cleaned up automatically after commit or cancellation.

## Stage 0: Design Intelligence

Stage 0 is a two-phase design conversation powered by Opus:

**Phase A — Clarity loop**: Socratic questions targeting the weakest clarity dimension until ambiguity drops below 20% (mathematical scoring across goal, constraints, acceptance criteria).

**Phase B — Approach selection**: Opus proposes 2-3 architectural approaches with trade-offs. You pick one. Opus then authors per-unit contracts that Sonnet agents implement against.

This replaces `deep-interview → ralplan → autopilot` chained manually — it's the same quality gates in a single command.

## Prerequisites

1. **[oh-my-claudecode (OMC)](https://github.com/sgomez-dev/oh-my-claudecode)** — Zero-Touch uses OMC's state management (`state_write`/`state_read`) for stage tracking, `/cancel` support, and HUD integration.

2. **Register the state mode** — add `'zero-touch'` to OMC's state mode list. In your OMC installation:

   ```typescript
   // ~/.claude/plugins/marketplaces/omc/src/tools/state-tools.ts
   const EXTRA_STATE_ONLY_MODES = ['ralplan', 'omc-teams', 'deep-interview', 'zero-touch'] as const;
   ```

   Also update the compiled dist file at the same path but under `dist/`:
   ```javascript
   // dist/tools/state-tools.js
   const EXTRA_STATE_ONLY_MODES = ['ralplan', 'omc-teams', 'deep-interview', 'zero-touch'];
   ```

3. **`gh` CLI authenticated** — required for PR creation and branch protection checks.
   ```bash
   gh auth status
   ```

4. **Git repository** — the target project must be a git repo, or pass `--init` to create one.

## Installation

```bash
# Via skills CLI
npx skills add Guipetris/zero-touch

# Or manually: copy SKILL.md into your Claude skills directory
cp SKILL.md ~/.claude/skills/zero-touch/SKILL.md
```

## Configuration

On first run, Zero-Touch asks two configuration questions and writes answers to `.zero-touch/config.json` at your project root:

- **Delivery mode**: PR to target branch (recommended) or direct merge
- **Target branch**: detected automatically (main/master), or specify

Config is project-local and committed to `.gitignore` automatically.

## Flags

| Flag | Description |
|---|---|
| `--ticket {id}` | Ticket-driven mode — fetches GitHub issue and skips Stage 0 entirely |
| `--init` | Allow `git init` if the project has no repository |
| `--resume {run-id}` | Explicitly resume a specific interrupted run |

## Cancellation

```bash
/oh-my-claudecode:cancel
```

Cancels the active pipeline, releases the worktree slot, archives the branch, and clears OMC state.

## State Files

All runtime state lives under `.zero-touch/` at your project root:

```
.zero-touch/
├── config.json                    # Project-level delivery config
├── worktrees/                     # Git worktree pool
│   └── {slot}/                    # One per concurrent feature
│       └── agents/{unit-id}/      # Per-agent isolated worktrees
└── state/
    └── {run-id}/
        ├── design.json            # Stage 0 output (goals, constraints, approach)
        ├── feature-plan.json      # Stage 2 output (units + contracts)
        └── corrections.json       # Correction budget tracking
```

## License

Apache 2.0 — see [LICENSE](LICENSE).
