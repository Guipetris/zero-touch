# Implementation Plan: Model Timing & HUD Enhancement

**Spec:** `docs/superpowers/specs/2026-03-13-model-timing-hud-design.md`
**Date:** 2026-03-13
**Status:** Ready for execution

---

## Overview

Six implementation steps, ordered by dependency chain. Each step is independently testable.
The plan follows the spec's "Implementation Order" exactly.

**New files:** 2 (`shaman-timing.mjs`, `shaman-summary.mjs`)
**Modified files:** 2 (`shaman-hud.mjs`, `SKILL.md`)
**New directory:** 1 (`~/.claude/hud/model-usage-runs/`)

---

## Step 1: `shaman-timing.mjs` — CLI Helper

**File:** `~/.claude/hud/shaman-timing.mjs`
**Dependencies:** None (first in chain)
**Model routing for executor:** Sonnet (standard Node.js CLI, well-defined spec)

### What to implement

A Node.js CLI script (ES module, `#!/usr/bin/env node`) that encapsulates all `model-timing.json`
state mutations. Follow the patterns established in `~/.claude/hud/skill-tracker.mjs`:
- Same import style (`node:fs`, `node:path`, `node:os`)
- Same `readJSON`/`writeJSON`/`ensureDir` helper pattern
- Same `process.argv` dispatch switch at the bottom

### CLI commands (8 total)

| Command | Args | Behavior |
|---------|------|----------|
| `--init` | `<run_id> <feature_slug>` | 1. Derive state dir from CWD: `${CWD}/.shaman-pipe/state/${run_id}/`. 2. Create `model-timing.json` with `{ version: "1.0", run_id, feature_slug, status: "running", stages: [], completed_at: null }`. 3. Ensure `~/.claude/hud/model-usage-runs/` directory exists (`mkdirSync recursive`). |
| `--start-stage` | `<N> <label> <model_tier>` | 1. Read `run-state.json` from CWD to get `run_id`. 2. Read `model-timing.json`. 3. Append to `stages[]`: `{ stage: N, label, model_tier, started_at: new Date().toISOString(), ended_at: null, duration_ms: null, exit_reason: null }`. 4. Write back. |
| `--end-stage` | `<N> <exit_reason>` | 1. Find stage entry where `stage === N` and `ended_at === null`. 2. Set `ended_at = now`, compute `duration_ms = new Date(ended_at) - new Date(started_at)`. 3. For Stage 3 specifically: compute `wall_clock_ms` from stage start/end, compute `duration_ms` as sum of all `agents[].duration_ms` (only those with non-null duration). 4. Set `exit_reason`. 5. Write back. |
| `--start-agent` | `<stage_N> <unit_id> <model_tier>` | 1. Find stage entry where `stage === N`. 2. Initialize `agents: []` if not present. 3. Append `{ unit_id, model_tier, started_at: now, ended_at: null, duration_ms: null }`. 4. Write back. |
| `--end-agent` | `<stage_N> <unit_id>` | 1. Find agent in `stages[N].agents[]` by `unit_id`. 2. Set `ended_at = now`, compute `duration_ms`. 3. Write back. |
| `--complete` | (none) | Set `status: "completed"`, `completed_at: now`. Single `writeFileSync`. |
| `--cancel` | (none) | Set `status: "cancelled"`, `completed_at: now`. Mark any open stage/agent entries as interrupted: set `exit_reason: "interrupted"`, leave `duration_ms: null`. |
| `--fail` | (none) | Same as `--cancel` but `status: "failed"`. |

### Key design decisions

- **State file discovery:** `--init` takes `run_id` directly. All other commands read `CWD/.shaman-pipe/state/run-state.json` to discover `run_id`, then operate on `CWD/.shaman-pipe/state/${run_id}/model-timing.json`.
- **Stage N is parsed as integer** from argv: `parseInt(args[0], 10)`.
- **Idempotent guards:** `--start-stage` should warn to stderr (not crash) if a stage entry for N already exists with a non-null `started_at`. `--end-stage` should warn if no matching open entry found.
- **No `process.exit(1)` on missing state files** — timing is best-effort. Print warning to stderr, exit 0.
- **`writeFileSync` atomicity:** Write to a temp file then rename, same pattern as pool.json in SKILL.md. Use `writeFileSync(path + '.tmp', ...)` then `renameSync(path + '.tmp', path)`.

### Acceptance criteria

- [ ] `node shaman-timing.mjs --init test123 "my-feature"` creates `model-timing.json` with correct schema and creates `~/.claude/hud/model-usage-runs/`
- [ ] `--start-stage 0 "Clarity Ritual" opus` appends a stage entry with `started_at` populated, `ended_at` null
- [ ] `--end-stage 0 success` computes `duration_ms` correctly (non-zero positive integer)
- [ ] `--start-agent 3 webhook-handler sonnet` + `--end-agent 3 webhook-handler` records agent timing
- [ ] `--end-stage 3 success` computes `wall_clock_ms` (stage wall clock) and `duration_ms` (sum of agents)
- [ ] `--complete` sets status to "completed" and `completed_at` to ISO string
- [ ] `--cancel` marks open entries as interrupted, sets status "cancelled"
- [ ] `--fail` marks open entries as interrupted, sets status "failed"
- [ ] Missing state files produce stderr warning, exit 0 (no crash)
- [ ] All timestamps are captured by the script, not passed as arguments

### Testing strategy

Create a temp directory structure mimicking `.shaman-pipe/state/{run_id}/` and `run-state.json`.
Run the full lifecycle sequence:

```bash
# Setup
mkdir -p /tmp/test-timing/.shaman-pipe/state
echo '{"active":true,"run_id":"test123","stage":0}' > /tmp/test-timing/.shaman-pipe/state/run-state.json

# Test init
cd /tmp/test-timing && node ~/.claude/hud/shaman-timing.mjs --init test123 "my-feature"
# Verify: cat .shaman-pipe/state/test123/model-timing.json — should have version, run_id, empty stages

# Test stage lifecycle
node ~/.claude/hud/shaman-timing.mjs --start-stage 0 "Clarity Ritual" opus
# Verify: stages[0] has started_at, no ended_at

sleep 1
node ~/.claude/hud/shaman-timing.mjs --end-stage 0 success
# Verify: stages[0] has ended_at, duration_ms > 0

# Test agent lifecycle (Stage 3)
node ~/.claude/hud/shaman-timing.mjs --start-stage 3 "The Summoning" sonnet
node ~/.claude/hud/shaman-timing.mjs --start-agent 3 webhook-handler sonnet
node ~/.claude/hud/shaman-timing.mjs --start-agent 3 webhook-types sonnet
sleep 1
node ~/.claude/hud/shaman-timing.mjs --end-agent 3 webhook-handler
node ~/.claude/hud/shaman-timing.mjs --end-agent 3 webhook-types
node ~/.claude/hud/shaman-timing.mjs --end-stage 3 success
# Verify: wall_clock_ms and duration_ms both populated, duration_ms = sum of agents

# Test cancel with open entries
node ~/.claude/hud/shaman-timing.mjs --start-stage 4 "Validation Rite" haiku
node ~/.claude/hud/shaman-timing.mjs --cancel
# Verify: stage 4 has exit_reason "interrupted", status "cancelled"

# Verify model-usage-runs dir exists
ls ~/.claude/hud/model-usage-runs/
```

---

## Step 2: `shaman-summary.mjs` — Terminal Table + Usage Fragment

**File:** `~/.claude/hud/shaman-summary.mjs`
**Dependencies:** Step 1 (reads `model-timing.json` written by `shaman-timing.mjs`)
**Model routing for executor:** Sonnet

### What to implement

A Node.js CLI script that:
1. Reads `model-timing.json` for a given run
2. Prints a formatted terminal table (Unicode box drawing + ANSI colors)
3. Writes a usage fragment to `~/.claude/hud/model-usage-runs/{run_id}.json`

### CLI interface

```
node ~/.claude/hud/shaman-summary.mjs --run-id <run_id>
```

The script reads `CWD/.shaman-pipe/state/{run_id}/model-timing.json`. If not found, exit 0 with no output (spec: "exits cleanly with no output").

### Terminal table format

Match the spec's box-drawing table exactly. Key rendering rules:

- **Column widths:** Stage (5), Ritual (20), Model (7), Duration (8), Exit (13)
- **Duration format:** `Xm Ys` (e.g., `7m 30s`). Helper: `function fmtDuration(ms) { const s = Math.floor(ms/1000); return \`${Math.floor(s/60)}m ${s%60}s\`; }`
- **Stage 3 agents:** Indented with tree characters (`├─` for non-last, `└─` for last). Unit IDs truncated to fit Ritual column.
- **Incomplete stages:** Show "interrupted" in Duration column
- **ANSI colors:**
  - `exit_reason: "correction"` -> yellow (`\x1b[33m`)
  - `exit_reason: "escalation"` -> red (`\x1b[31m`)
  - Default -> no color
- **Totals row:** Sum `duration_ms` per model tier. Format: `Opus: Xm Ys  Sonnet: Xm Ys  Haiku: Xm Ys`
- **Wall clock:** Sum `wall_clock_ms` for Stage 3 + `duration_ms` for all other stages
- **Model time:** Sum `duration_ms` across all stages (uses agent-level totals for Stage 3)

### Usage fragment writer

After printing the table, write `~/.claude/hud/model-usage-runs/{run_id}.json`:

```json
{
  "version": "1.0",
  "run_id": "...",
  "feature_slug": "...",
  "status": "completed|cancelled|failed",
  "completed_at": "<ISO>",
  "completed_at_epoch": <epoch_seconds>,
  "model_totals_ms": {
    "opus": <ms>,
    "sonnet": <ms>,
    "haiku": <ms>
  }
}
```

- `completed_at_epoch`: `Math.floor(new Date(completed_at).getTime() / 1000)`
- Only include model tiers that have non-zero totals in `model_totals_ms`
- Ensure `~/.claude/hud/model-usage-runs/` exists before writing (defensive `mkdirSync`)

### Acceptance criteria

- [ ] `node shaman-summary.mjs --run-id test123` prints a correctly formatted box-drawing table to stdout
- [ ] Stage 3 agents appear as indented rows with tree characters
- [ ] Correction stages are yellow, escalation stages are red
- [ ] Incomplete stages show "interrupted"
- [ ] Totals row shows correct per-model sums
- [ ] Wall clock vs model time are computed correctly (wall_clock_ms for Stage 3, duration_ms for others)
- [ ] Usage fragment is written to `~/.claude/hud/model-usage-runs/{run_id}.json` with correct schema
- [ ] `completed_at_epoch` is epoch seconds (not milliseconds)
- [ ] Missing `model-timing.json` -> clean exit with no output, exit code 0
- [ ] Cancelled/failed runs produce partial tables with "(cancelled)" or "(failed)" marker in header

### Testing strategy

Create a mock `model-timing.json` with known values (use the example from the spec) and verify:

1. **Happy path:** Full 7-stage completed run -> table matches spec example
2. **Cancelled run:** Only stages 0-2 completed, stage 3 open -> "interrupted" shown
3. **Correction exit:** Stage with `exit_reason: "correction"` -> yellow ANSI in output
4. **No file:** Run with nonexistent run_id -> no output, no error
5. **Fragment verification:** After each test, read the fragment file and verify `model_totals_ms` sums

---

## Step 3: `shaman-hud.mjs` — Add `modelTimingSegment()`

**File:** `~/.claude/hud/shaman-hud.mjs` (modify existing)
**Dependencies:** Step 2 (reads usage fragments written by `shaman-summary.mjs`)
**Model routing for executor:** Sonnet

### What to implement

Add a `modelTimingSegment()` function and wire it into both `shamanStatus()` and `fallback()`.

### Changes to `shaman-hud.mjs`

#### 1. Add imports

Add `readdirSync`, `unlinkSync` to the existing `node:fs` import (line 14):

```javascript
import { existsSync, readFileSync, readdirSync, unlinkSync } from "node:fs";
```

#### 2. Add `modelTimingSegment()` function

Insert after the `trunc()` / `colorPct()` helpers (around line 61), before the stdin section.

Logic:
1. Define `USAGE_DIR = join(HOME, ".claude/hud/model-usage-runs")`.
2. If directory doesn't exist, return `""`.
3. Determine window start:
   - Read `~/.claude/quota-cache.json`. If `reset_5h` exists, `windowStart = reset_5h - 18000`.
   - Fallback: `windowStart = Math.floor(Date.now() / 1000) - 18000`.
4. Read all `.json` files in `USAGE_DIR` via `readdirSync`.
5. For each file:
   - Parse JSON. Skip if `version !== "1.0"` (warn to stderr).
   - If `completed_at_epoch < windowStart`: delete the file (lazy purge), skip.
   - Accumulate `model_totals_ms.opus`, `.sonnet`, `.haiku`.
6. Format result:
   - Convert ms to minutes: `Math.round(ms / 60000)`.
   - Build segments: `O:{m}m`, `S:{m}m`, `H:{m}m`. Omit tiers with 0.
   - If no data, return `""`.
7. Color coding for Opus segment:
   - Compute `opusPct = opus / (opus + sonnet + haiku) * 100`.
   - `opusPct > 70` -> red. `opusPct > 50` -> yellow. Otherwise default.
   - Sonnet and Haiku are always default/green.
8. Return `" | " + segments.join(" ")` (note the ` | ` separator with pipe character).

#### 3. Wire into `shamanStatus()`

At the end of the `shamanStatus()` function (line 121), before returning, append the model timing segment:

```javascript
// Current return (line 121):
return `${CYAN}Pipe${RESET} ...${slotLabel}`;

// Change to:
const timing = modelTimingSegment();
return `${CYAN}Pipe${RESET} ...${slotLabel}${timing}`;
```

Apply similarly to the Stage 0 branch (line 114).

#### 4. Wire into `fallback()`

In `fallback()`, before the final `return parts.join("  ")` (line 239), append:

```javascript
const timing = modelTimingSegment();
if (timing) parts.push(timing.trim());
// Or alternatively, since timing already includes " | ", append directly to the output string
```

**Important nuance:** `shamanStatus()` returns a single string, so append `timing` directly. `fallback()` uses a `parts[]` array joined by double-space, so the `|` separator should be part of the timing string itself. The simplest approach: if `modelTimingSegment()` returns non-empty, push the entire `"| O:Xm S:Xm"` string (with the pipe) as a part.

#### 5. Handle the stdin flow issue

Currently `shamanStatus()` returns early before stdin is read (line 247). The `modelTimingSegment()` function does NOT need stdin — it reads files. This is correct; no change needed to the main flow.

### Acceptance criteria

- [ ] When `~/.claude/hud/model-usage-runs/` has fragment files within the 5h window, HUD output includes `O:Xm S:Xm H:Xm` segment
- [ ] Tiers with 0 minutes are omitted from display
- [ ] Opus segment turns yellow when Opus > 50% of total, red when > 70%
- [ ] Fragments outside the 5h window are deleted (lazy purge)
- [ ] When no fragments exist (or directory missing), no timing segment appears (no stray `|`)
- [ ] Timing segment appears in both active-pipeline and idle-fallback HUD states
- [ ] `quota-cache.json` missing -> falls back to `Date.now() - 18000s`
- [ ] Invalid/corrupt fragment files are skipped without crashing

### Testing strategy

1. Create mock fragment files in `~/.claude/hud/model-usage-runs/`:
   - One with `completed_at_epoch` = now (within window)
   - One with `completed_at_epoch` = 6 hours ago (outside window, should be purged)
2. Run `echo '{}' | node ~/.claude/hud/shaman-hud.mjs` and verify timing segment appears
3. Verify the stale fragment was deleted after the run
4. Test with empty directory -> no timing segment
5. Test with directory missing -> no timing segment, no crash

---

## Step 4: `SKILL.md` — Add Timing Instrumentation

**File:** `/Users/guilherme.petris/Documents/the-shaman-pipe/SKILL.md`
**Dependencies:** Step 1 (the CLI commands must exist)
**Model routing for executor:** Sonnet (careful text editing)

### What to implement

Insert `Bash("node ~/.claude/hud/shaman-timing.mjs ...")` calls at every stage boundary in SKILL.md.
These are prompt-level instructions — the LLM executing SKILL.md will run them as Bash commands.

### Instrumentation points (14 insertion locations)

**Important principle from spec:** "SKILL.md should not gate on timing script exit codes." All timing Bash calls should be fire-and-forget. Append `|| true` to each call so timing failures never block the pipeline.

#### Stage 0 start (after "Initialize run state", ~line 213)

Insert after the `Write(".shaman-pipe/state/run-state.json", ...)` block:

```
Initialize model timing:
Bash("node ~/.claude/hud/shaman-timing.mjs --init {run-id} \"{feature_slug}\" || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 0 \"Clarity Ritual\" opus || true")
```

#### Stage 0 -> Stage 1 transition (~line 625)

Insert before the `Write(".shaman-pipe/state/run-state.json", { stage: 1, ... })`:

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 0 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 1 \"Environment Reading\" haiku || true")
```

**For `exit_reason`:** The spec defines three values: `success`, `correction`, `escalation`. At the end of Stage 0, always use `success` (Stage 0 doesn't have a correction loop — it has the Socratic ambiguity loop, which is not a "correction"). For later stages that enter the Adaptive Correction System, the exit_reason should reflect the outcome. This is a **user decision point** — see "Key decisions requiring user input" below.

#### Stage 1 -> Stage 2 transition (~line 780)

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 1 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 2 \"Contract Ceremony\" opus || true")
```

#### Stage 2 -> Stage 3 transition (~line 950)

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 2 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 3 \"The Summoning\" sonnet || true")
```

#### Stage 3 per-agent start (inside the per-agent TaskCreate block, ~line 992)

Before each agent TaskCreate, add:

```
Bash("node ~/.claude/hud/shaman-timing.mjs --start-agent 3 {unit-id} sonnet || true")
```

#### Stage 3 per-agent complete (after agent validation, ~line 1056)

After each agent's inline contract gate passes:

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-agent 3 {unit-id} || true")
```

#### Stage 3 -> Stage 4 transition (~line 1144)

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 3 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 4 \"Validation Rite\" haiku || true")
```

#### Stage 4 -> Stage 5 transition (~line 1290)

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 4 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 5 \"Final Judgment\" opus || true")
```

#### Stage 5 -> Stage 6 transition (~line 1401)

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 5 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 6 \"The Sealing\" haiku || true")
```

#### Stage 6 completion (after cleanup, before clearing run-state, ~line 1518)

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 6 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --complete || true")
Bash("node ~/.claude/hud/shaman-summary.mjs --run-id {run-id} || true")
```

### Ticket mode and spec mode

For `--ticket` mode (skip Stage 0), the `--init` call happens at Stage 1 start instead.
The `--start-stage` begins at 1 instead of 0. Insert init+start at the ticket mode Stage 1 entry point (~line 475).

For `--spec` mode, Stage 0 still runs (spec-seeded mode), so the instrumentation is the same as interactive mode.

### Acceptance criteria

- [ ] Every stage boundary has paired `--end-stage` / `--start-stage` calls
- [ ] Stage 3 agent lifecycle has `--start-agent` before TaskCreate and `--end-agent` after contract gate
- [ ] Stage 6 completion has `--end-stage 6` + `--complete` + `shaman-summary.mjs` invocation
- [ ] All timing Bash calls include `|| true` (non-blocking)
- [ ] Ticket mode has `--init` at its entry point
- [ ] Timing calls use the correct `run-id` variable (matching SKILL.md's existing `{run-id}` convention)
- [ ] The `--init` call comes AFTER `mkdir -p .shaman-pipe/state/{run-id}/contracts` (directory must exist)

### Testing strategy

This step modifies a prompt file, not executable code. Testing is done via:
1. **Manual review:** Read the modified SKILL.md and verify every stage boundary has timing calls
2. **Dry run:** Execute a real pipeline run (`/the-shaman-pipe "test feature"`) and verify `model-timing.json` populates
3. **Grep audit:** `grep -c "shaman-timing" SKILL.md` should return ~14 (2 per stage transition + agent calls)

---

## Step 5: Cancellation & Failure Flows

**File:** `/Users/guilherme.petris/Documents/the-shaman-pipe/SKILL.md`
**Dependencies:** Steps 1, 2, 4
**Model routing for executor:** Sonnet

### What to implement

Add `--cancel`/`--fail` + `shaman-summary.mjs` invocation to the three termination flows in SKILL.md.

#### Cancellation flow (line ~2243, "Pipeline Cancellation" section)

Insert after Step 6 (write partial run log) and before Step 7 (clear run state):

```
Step 6b: Record timing data:
  Bash("node ~/.claude/hud/shaman-timing.mjs --cancel || true")
  Bash("node ~/.claude/hud/shaman-summary.mjs --run-id {run-id} || true")
```

#### REJECT verdict (line ~1380, Stage 5 REJECT action)

Insert after step 3 (release pool slot) and before step 4 (clear run state):

```
Bash("node ~/.claude/hud/shaman-timing.mjs --fail || true")
Bash("node ~/.claude/hud/shaman-summary.mjs --run-id {run-id} || true")
```

#### Checkpoint "Cancel pipeline" response (line ~1924)

The checkpoint cancel enters the same cancellation flow, so the above cancellation instrumentation covers it.

### exit_reason tracking for correction stages

When a stage passes after corrections were applied (Stages 3, 4, 5), the `--end-stage` call should use `correction` instead of `success`. When correction budget is exhausted and it escalates, use `escalation`.

**Implementation approach:** In the Adaptive Correction System section (~line 1526), after a successful correction:
- Track a flag (e.g., `had_corrections = true` in the stage's context)
- At `--end-stage` call: use `correction` if corrections were applied, `success` otherwise
- At escalation: use `escalation`

**This is a user decision point** — see "Key decisions" below.

### Acceptance criteria

- [ ] Cancellation flow calls `--cancel` then `shaman-summary.mjs`
- [ ] REJECT verdict calls `--fail` then `shaman-summary.mjs`
- [ ] Terminal summary is printed for cancelled/failed runs (partial table)
- [ ] Usage fragment is written for cancelled/failed runs (partial totals)
- [ ] All calls include `|| true`

### Testing strategy

1. Start a pipeline, cancel via `touch .shaman-pipe/CANCEL` during Stage 2
2. Verify `model-timing.json` shows status "cancelled", open stages are "interrupted"
3. Verify terminal summary prints with partial data
4. Verify usage fragment exists in `~/.claude/hud/model-usage-runs/`

---

## Step 6: Resume Flow Verification

**File:** `/Users/guilherme.petris/Documents/the-shaman-pipe/SKILL.md` (minor addition)
**Dependencies:** Steps 1, 4
**Model routing for executor:** Sonnet

### What to implement

Verify and document that `model-timing.json` is preserved on resume. The spec states:

> "When a pipeline is resumed (--resume), model-timing.json from the previous partial run is preserved. The resumed stage gets a fresh started_at (the gap between crash and resume is not counted)."

### Changes needed

In the Resume mode section (~line 94), add an instruction:

```
When resuming, model-timing.json is preserved from the previous partial run.
The resumed stage gets a fresh --start-stage call (fresh started_at).
Do NOT call --init again on resume — the timing file already exists.
```

The `--start-stage` call at the resumed stage's entry point naturally handles this because:
1. The previous stage's `--end-stage` was never called (crash), so it has `ended_at: null`
2. The resumed stage gets a new `--start-stage` entry with fresh `started_at`
3. Readers skip entries without `duration_ms`, so the crashed stage entry is harmlessly ignored

### Verification needed

Confirm that `shaman-timing.mjs --start-stage` does NOT require the previous stage to be closed. It simply appends a new entry. This means a crash at Stage 3 followed by resume at Stage 3 creates two Stage 3 entries — one with null `ended_at` (skipped by readers) and one with fresh timing.

### Acceptance criteria

- [ ] Resume mode documentation in SKILL.md mentions timing preservation
- [ ] `--start-stage` works even if a prior entry for the same stage exists with null `ended_at`
- [ ] Readers (shaman-summary.mjs, shaman-hud.mjs) skip entries where `duration_ms` is null
- [ ] No `--init` call in the resume path

### Testing strategy

1. Start a run, complete stages 0-2, simulate crash (kill the session)
2. Resume via `--resume`
3. Verify `model-timing.json` has stages 0-2 with valid timing + a stage 3 entry with null `ended_at` from the crash
4. After resumed Stage 3 completes, verify a second Stage 3 entry with valid timing
5. Verify the terminal summary shows stages 0-2 + resumed Stage 3 (skipping the crashed entry)

---

## Key Decisions Requiring User Input

These are design decisions where the spec is ambiguous or the implementor needs guidance.
The executor should ask the user (not guess) for these 5 items.

### Decision 1: `exit_reason` tracking through the correction system

The spec defines `exit_reason` as `success | correction | escalation`, but SKILL.md's Adaptive Correction System doesn't currently track whether corrections were applied at the stage level.

**Options:**
- (A) **Simple:** Always use `success` for `--end-stage`. The `exit_reason` refinement for `correction`/`escalation` is deferred to a follow-up. The table still shows all timing data; only the Exit column loses fidelity.
- (B) **Inline flag:** Add a comment in the correction system section instructing the LLM to track a `had_corrections` mental flag and use it in the `--end-stage` call. Fragile — the LLM may forget.
- (C) **State in timing file:** Have `shaman-timing.mjs` expose a `--mark-correction <N>` command that sets a flag on the stage entry. The `--end-stage` call reads this flag. Robust but adds a 9th CLI command.

**Recommendation:** Option A for v1, Option C queued for v1.1.

### Decision 2: Should `shaman-summary.mjs` print to stderr or stdout?

The spec says "prints terminal table." Stdout is natural for display, but if SKILL.md captures the output for any reason, it could be noisy.

**Recommendation:** Stdout. The Bash calls in SKILL.md don't capture output (no `$(...)` wrapper), so stdout is clean.

### Decision 3: Duration format — `7m 30s` vs `7:30` vs `7.5m`

The spec example uses `7m 30s`. Confirm this is preferred over more compact alternatives.

### Decision 4: `modelTimingSegment()` rounding

Minutes are shown as integers (`O:15m`). When a tier has 89 seconds of usage (1.48 minutes), should it show `O:1m` or `O:2m`?

**Recommendation:** `Math.round()` — 89s shows as `O:1m`, 91s shows as `O:2m`.

### Decision 5: Table width and truncation

The spec example table is ~65 chars wide. On narrow terminals, this could wrap. Should the executor add terminal width detection and truncation, or is fixed-width acceptable for v1?

**Recommendation:** Fixed-width for v1. Claude Code terminals are typically wide enough.

---

## Execution Order Summary

```
Step 1: shaman-timing.mjs        [NEW FILE]     ~150 lines    No dependencies
Step 2: shaman-summary.mjs       [NEW FILE]     ~180 lines    Depends on Step 1 data model
Step 3: shaman-hud.mjs           [MODIFY]       ~40 lines     Depends on Step 2 fragment format
Step 4: SKILL.md instrumentation  [MODIFY]       ~30 insertions Depends on Step 1 CLI interface
Step 5: Cancel/fail flows         [MODIFY]       ~10 insertions Depends on Steps 1, 2
Step 6: Resume flow verification  [MODIFY]       ~5 lines      Depends on Steps 1, 4
```

Steps 1-3 form the "data pipeline" (write -> summarize -> display).
Steps 4-6 form the "integration layer" (hook into SKILL.md execution flow).

Steps 1 and 2 can be implemented by the same executor in sequence.
Step 3 is a separate concern (HUD rendering) — can be a parallel executor.
Steps 4-6 all modify SKILL.md — must be sequential, ideally same executor.

---

## File Reference

| File | Path | Action |
|------|------|--------|
| `shaman-timing.mjs` | `~/.claude/hud/shaman-timing.mjs` | CREATE |
| `shaman-summary.mjs` | `~/.claude/hud/shaman-summary.mjs` | CREATE |
| `shaman-hud.mjs` | `~/.claude/hud/shaman-hud.mjs` | MODIFY |
| `SKILL.md` | `/Users/guilherme.petris/Documents/the-shaman-pipe/SKILL.md` | MODIFY |
| `model-usage-runs/` | `~/.claude/hud/model-usage-runs/` | CREATE (dir, by shaman-timing.mjs --init) |
| `model-timing.json` | `.shaman-pipe/state/{run_id}/model-timing.json` | CREATE (per-run, by shaman-timing.mjs) |
| Design spec | `docs/superpowers/specs/2026-03-13-model-timing-hud-design.md` | READ ONLY (reference) |
| `skill-tracker.mjs` | `~/.claude/hud/skill-tracker.mjs` | READ ONLY (pattern reference) |
| `quota-cache.json` | `~/.claude/quota-cache.json` | READ ONLY (by modelTimingSegment) |
