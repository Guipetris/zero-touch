# Model Timing & HUD Enhancement

**Date:** 2026-03-13
**Status:** Draft
**Author:** Guilherme Petris + Claude

## Problem

The Shaman Pipe orchestrates three model tiers (Opus, Sonnet, Haiku) across seven stages but provides no visibility into how much time each tier consumes. Without this data, there is no way to verify that the right model is being used for the right stage, spot cost anomalies, or understand where pipeline time is spent.

## Goals

1. **Terminal summary at end of each run** — a rich table showing stage x model x duration, including per-agent breakdowns for parallel Stage 3 workers.
2. **Always-on HUD signal** — a compact rolling total of model-tier time within the 5-hour quota window, visible in both the active pipeline statusline and the fallback statusline.

## Non-Goals

- Real-time parallel agent counter (no registry mechanism needed).
- Per-token cost estimation (time is the proxy).
- Historical trending beyond the 5h window (future enhancement).

---

## Architecture

### Data Flow

```
SKILL.md stage transitions
  │  Bash("node shaman-timing.mjs --start-stage ...")
  ▼
shaman-timing.mjs CLI helper                         ← all JSON mutation here
  │  writes to:
  ▼
.shaman-pipe/state/{run_id}/model-timing.json        ← per-run timing log
  │
  │  SKILL.md at Stage 6 / cancel / fail:
  │  Bash("node shaman-summary.mjs --run-id abc123")
  ▼
shaman-summary.mjs                                   ← prints terminal table
  │  writes to:                                         + writes usage fragment
  ▼
~/.claude/hud/model-usage-runs/{run_id}.json         ← append-only fragment
  │
  ▼  (HUD reader, every prompt render)
shaman-hud.mjs                                       ← renders O:34m S:14m H:3m
```

### Components

| Component | Responsibility |
|-----------|---------------|
| `shaman-timing.mjs` | CLI helper that mutates `model-timing.json`. SKILL.md calls it via Bash — no JSON manipulation from the prompt. |
| `shaman-summary.mjs` | Invoked by SKILL.md at pipeline completion (Stage 6, cancel, fail). Reads `model-timing.json`, prints terminal table, writes usage fragment. |
| `shaman-hud.mjs` | Reads usage fragments at every prompt render. Appends `O:Xm S:Xm H:Xm` to both `shamanStatus()` (active run) and `fallback()` (idle). |

### Why a CLI helper instead of direct JSON writes from SKILL.md

SKILL.md is a prompt executed by an LLM. Having the LLM read-modify-write a growing JSON array 14+ times per run (7 stages x start/end) is fragile — a single malformed write corrupts all timing data. `shaman-timing.mjs` encapsulates all state mutation behind simple Bash calls:

```bash
node ~/.claude/hud/shaman-timing.mjs --init abc123 "my-feature"
node ~/.claude/hud/shaman-timing.mjs --start-stage 0 "Clarity Ritual" opus
node ~/.claude/hud/shaman-timing.mjs --end-stage 0 success
node ~/.claude/hud/shaman-timing.mjs --start-agent 3 webhook-handler sonnet
node ~/.claude/hud/shaman-timing.mjs --end-agent 3 webhook-handler
node ~/.claude/hud/shaman-timing.mjs --complete
```

The `--init` command creates `model-timing.json` and ensures the `model-usage-runs/` directory exists. Timestamps are captured by the script (`new Date().toISOString()`), not by the LLM.

---

## Data Model

### File 1: `.shaman-pipe/state/{run_id}/model-timing.json`

Written by `shaman-timing.mjs` during the run. One entry per stage.

```json
{
  "version": "1.0",
  "run_id": "abc123",
  "feature_slug": "my-feature",
  "status": "running",
  "stages": [
    {
      "stage": 0,
      "label": "Clarity Ritual",
      "model_tier": "opus",
      "started_at": "2026-03-13T10:00:00.000Z",
      "ended_at": "2026-03-13T10:07:30.000Z",
      "duration_ms": 450000,
      "exit_reason": "success"
    },
    {
      "stage": 3,
      "label": "The Summoning",
      "model_tier": "sonnet",
      "started_at": "2026-03-13T10:12:00.000Z",
      "ended_at": "2026-03-13T10:17:00.000Z",
      "wall_clock_ms": 300000,
      "duration_ms": 869000,
      "agents": [
        {
          "unit_id": "webhook-handler",
          "model_tier": "sonnet",
          "started_at": "2026-03-13T10:12:01.000Z",
          "ended_at": "2026-03-13T10:16:50.000Z",
          "duration_ms": 289000
        },
        {
          "unit_id": "webhook-validator",
          "model_tier": "sonnet",
          "started_at": "2026-03-13T10:12:02.000Z",
          "ended_at": "2026-03-13T10:17:00.000Z",
          "duration_ms": 298000
        },
        {
          "unit_id": "webhook-types",
          "model_tier": "sonnet",
          "started_at": "2026-03-13T10:12:01.000Z",
          "ended_at": "2026-03-13T10:16:43.000Z",
          "duration_ms": 282000
        }
      ]
    }
  ],
  "completed_at": null
}
```

**Schema rules:**

- `status`: `"running"` | `"completed"` | `"cancelled"` | `"failed"`
- `status` and `completed_at` are set atomically (same `writeFileSync` call) when the run finishes.
- `exit_reason`: `"success"` | `"correction"` | `"escalation"` — see mapping table below.
- `duration_ms` is computed at write time by `shaman-timing.mjs` (`new Date(ended_at) - new Date(started_at)`). This avoids `NaN` propagation when `ended_at` is null on a crash.
- For Stage 3 (parallel workers): `wall_clock_ms` = elapsed time for the stage, `duration_ms` = sum of all agent durations (aggregate model-time). The `agents[]` sub-array records per-unit timing. **HUD totals use `duration_ms` (aggregate), not `wall_clock_ms`**, so that parallel Sonnet consumption is reported accurately.
- Non-parallel stages omit `agents[]` and `wall_clock_ms`.
- Entries missing `ended_at` or `duration_ms` are skipped by readers (crash tolerance).

**`exit_reason` mapping:**

| Scenario | Value |
|----------|-------|
| Stage passes without any correction attempt | `"success"` |
| Stage passes after one or more correction attempts | `"correction"` |
| Correction budget exhausted, human intervention needed | `"escalation"` |

### File 2: `~/.claude/hud/model-usage-runs/{run_id}.json`

Append-only fragments. One file per completed run. Written by `shaman-summary.mjs` at pipeline completion.

```json
{
  "version": "1.0",
  "run_id": "abc123",
  "feature_slug": "my-feature",
  "status": "completed",
  "completed_at": "2026-03-13T10:25:25.000Z",
  "completed_at_epoch": 1773577525,
  "model_totals_ms": {
    "opus": 927000,
    "sonnet": 869000,
    "haiku": 163000
  }
}
```

**Why append-only fragments instead of a shared file:**

The worktree pool allows 3 simultaneous features. If two runs complete near-simultaneously and both write to a shared file, one clobbers the other. Append-only fragments eliminate shared mutable state — each run writes its own file, the HUD reader aggregates at read time.

**Window management:**

- The HUD reader determines the current 5h window start as: `reset_5h - 18000` (in epoch seconds) when `quota-cache.json` is available, falling back to `Math.floor(Date.now() / 1000) - 18000` (5h ago) if the cache is missing or stale.
- Fragments where `completed_at_epoch < window_start` are purged lazily by the HUD reader (delete files on each read).
- `completed_at_epoch` is stored as epoch seconds alongside the ISO string to avoid type conversion at read time.

---

## Terminal Summary

Printed by `shaman-summary.mjs` when invoked from SKILL.md at pipeline completion. **Not** from the Stop hook — the Stop hook fires at session end, which may be long after the pipeline finishes.

```
┌─────────────────────────────────────────────────────────────────┐
│  The Shaman Pipe · run abc123 · "add stripe webhook handler"    │
├───────┬────────────────────┬─────────┬──────────┬───────────────┤
│ Stage │ Ritual             │ Model   │ Duration │ Exit          │
├───────┼────────────────────┼─────────┼──────────┼───────────────┤
│   0   │ Clarity Ritual     │ Opus    │   7m 30s │ success       │
│   1   │ Environment Reading│ Haiku   │   0m 45s │ success       │
│   2   │ Contract Ceremony  │ Opus    │   4m 12s │ success       │
│   3   │ The Summoning      │ Sonnet  │  14m 29s │ success       │
│       │  ├─ webhook-handler│         │   4m 49s │               │
│       │  ├─ webhook-valid..│         │   4m 58s │               │
│       │  └─ webhook-types  │         │   4m 42s │               │
│   4   │ Validation Rite    │ Haiku   │   1m 20s │ success       │
│   5   │ Final Judgment     │ Opus    │   3m 45s │ success       │
│   6   │ The Sealing        │ Haiku   │   0m 38s │ success       │
├───────┴────────────────────┴─────────┴──────────┴───────────────┤
│  Totals   Opus: 15m 27s  Sonnet: 14m 29s  Haiku: 2m 43s       │
│  Wall clock: 23m 10s   Model time: 32m 39s                     │
└─────────────────────────────────────────────────────────────────┘
```

**Notes:**
- Stage 3 "Duration" shows aggregate `duration_ms` (sum of agent durations = 289000+298000+282000 = 869000 = 14m 29s), with per-agent rows indented below.
- Wall clock = `wall_clock_ms` for Stage 3 (300000 = 5m 00s) + `duration_ms` for all other stages (450000+45000+252000+80000+225000+38000 = 1090000 = 18m 10s) = 1390000 = 23m 10s.
- Model time = sum of `duration_ms` across all stages (uses agent-level totals for Stage 3) = 1959000 = 32m 39s.
- Stages with `exit_reason: "correction"` are highlighted in yellow (ANSI).
- Stages with `exit_reason: "escalation"` are highlighted in red (ANSI).
- Incomplete stages (crash/cancel) show "interrupted" in the Duration column.

---

## HUD Integration

### Always-on model-time segment

The model-time segment appears in **both** HUD states — during active pipeline runs and in the idle fallback. This is achieved by extracting the segment computation into a shared helper function.

**During active run (appended to `shamanStatus()`):**
```
Pipe ▸ 3/6 The Summoning  stripe-webhook  [wt-1]  |  O:15m S:14m H:3m
```

**Idle fallback (appended to `fallback()`):**
```
the-shaman-pipe  main  Sonnet 4.6  [5h 12%]  ctx 8%  |  O:15m S:14m H:3m
```

**`modelTimingSegment()` helper logic:**
1. Glob `~/.claude/hud/model-usage-runs/*.json`.
2. Parse each fragment. Purge files where `completed_at_epoch < window_start` (window start = `reset_5h - 18000` from `quota-cache.json`, or `Date.now()/1000 - 18000` as fallback).
3. Sum `model_totals_ms` across remaining fragments.
4. Format as `O:{m}m S:{m}m H:{m}m`. Omit tiers with 0 time.
5. If no data in window, return empty string (no `|` segment).

**Color coding:** Opus segment is yellow when Opus exceeds 50% of total model time across the window, red when exceeding 70%. This is a heuristic signal that the pipeline is Opus-heavy (expensive). Sonnet and Haiku are always green/default since their cost is lower.

---

## Instrumentation via `shaman-timing.mjs` CLI

All SKILL.md timing writes are simple Bash calls to a CLI helper. The helper handles all JSON read-modify-write operations, timestamp capture, and directory creation.

### CLI Commands

| Command | Description |
|---------|-------------|
| `--init <run_id> <feature_slug>` | Creates `model-timing.json` with `status: "running"`, empty `stages[]`. Ensures `~/.claude/hud/model-usage-runs/` directory exists. |
| `--start-stage <N> <label> <model_tier>` | Appends a new stage entry with `started_at = now`. |
| `--end-stage <N> <exit_reason>` | Sets `ended_at = now`, computes `duration_ms`. For Stage 3: also computes `wall_clock_ms` and aggregate `duration_ms` from agents. |
| `--start-agent <stage_N> <unit_id> <model_tier>` | Appends agent entry to `stages[N].agents[]` with `started_at = now`. |
| `--end-agent <stage_N> <unit_id>` | Sets agent's `ended_at = now`, computes `duration_ms`. |
| `--complete` | Sets `status: "completed"`, `completed_at = now`. |
| `--cancel` | Sets `status: "cancelled"`, `completed_at = now`. Marks any open stage/agent as interrupted. |
| `--fail` | Sets `status: "failed"`, `completed_at = now`. Marks any open stage/agent as interrupted. |

**State file location:** The CLI reads `CWD/.shaman-pipe/state/run-state.json` to discover the current `run_id`, then operates on `CWD/.shaman-pipe/state/{run_id}/model-timing.json`. The `--init` command takes `run_id` explicitly since `run-state.json` may not yet exist.

### SKILL.md Instrumentation Points

| Transition | SKILL.md Bash call |
|------------|-------------------|
| Stage 0 start | `node ~/.claude/hud/shaman-timing.mjs --init $RUN_ID "$FEATURE_SLUG"` then `--start-stage 0 "Clarity Ritual" opus` |
| Stage N-1 end → Stage N start | `--end-stage $((N-1)) success` then `--start-stage $N "$LABEL" $MODEL` |
| Stage 3 agent launch | `--start-agent 3 $UNIT_ID sonnet` |
| Stage 3 agent complete | `--end-agent 3 $UNIT_ID` |
| Stage 6 end (success) | `--end-stage 6 success` then `--complete` then `node ~/.claude/hud/shaman-summary.mjs --run-id $RUN_ID` |
| Cancellation | `--cancel` then `node ~/.claude/hud/shaman-summary.mjs --run-id $RUN_ID` |
| Unrecoverable failure | `--fail` then `node ~/.claude/hud/shaman-summary.mjs --run-id $RUN_ID` |

### Resume Flow

When a pipeline is resumed (`--resume`), `model-timing.json` from the previous partial run is preserved. The resumed stage gets a fresh `started_at` (the gap between crash and resume is not counted). This is correct — the gap represents idle time, not model time.

---

## New Files

| File | Location | Purpose |
|------|----------|---------|
| `shaman-timing.mjs` | `~/.claude/hud/` | CLI helper: all `model-timing.json` mutations. Called from SKILL.md via Bash. |
| `shaman-summary.mjs` | `~/.claude/hud/` | Terminal table printer + usage fragment writer. Called from SKILL.md at pipeline end. |
| `model-usage-runs/` | `~/.claude/hud/` | Directory of append-only per-run usage fragments. Created by `shaman-timing.mjs --init`. |

## Modified Files

| File | Change |
|------|--------|
| `SKILL.md` | Add `Bash("node shaman-timing.mjs ...")` calls at stage boundaries. Add `Bash("node shaman-summary.mjs ...")` at pipeline end / cancel / fail. |
| `shaman-hud.mjs` | Add `modelTimingSegment()` helper. Append its output to both `shamanStatus()` and `fallback()`. |

**Note:** `settings.json` does NOT need changes. The summary is triggered from SKILL.md, not the Stop hook. The Stop hook may optionally be extended later as a safety net for crash recovery, but that is out of scope for v1.

---

## Failure Modes

| Scenario | Handling |
|----------|----------|
| Crash mid-stage (no `ended_at`) | `shaman-timing.mjs` marks open entries as interrupted on `--cancel`/`--fail`. Readers skip entries without `duration_ms`. Terminal summary shows "interrupted" for incomplete stages. |
| Cancellation | SKILL.md calls `--cancel` then `shaman-summary.mjs`. Summary prints partial table with "(cancelled)" marker and writes a fragment with partial totals. |
| `quota-cache.json` missing/stale | HUD reader falls back to `Math.floor(Date.now() / 1000) - 18000` as window start. Self-heals on next quota refresh. |
| Concurrent run completions | Append-only fragments — no shared mutable state, no write race. |
| Fragment directory grows unbounded | Lazy purge: HUD reader deletes fragments with `completed_at_epoch` before window start. |
| `model-timing.json` not found | `shaman-summary.mjs` exits cleanly with no output — no pipeline ran. |
| `shaman-timing.mjs` fails mid-run | Timing data is best-effort. Pipeline continues regardless — timing writes are not in the critical path. SKILL.md should not gate on timing script exit codes. |
| `shaman-summary.mjs` fails at pipeline end | Usage fragment is not written. HUD shows stale data. Terminal summary is missing. Non-critical — does not affect committed code. |
| `model-usage-runs/` directory missing | `shaman-timing.mjs --init` creates it. If `shaman-summary.mjs` runs without it, it creates the directory before writing. |
| Version mismatch | Readers check `version` field. Unknown versions are skipped with a stderr warning. Future migrations branch on this field. |

---

## Implementation Order

1. `shaman-timing.mjs`: CLI helper with all commands (`--init`, `--start-stage`, `--end-stage`, `--start-agent`, `--end-agent`, `--complete`, `--cancel`, `--fail`)
2. `shaman-summary.mjs`: terminal table renderer + usage fragment writer
3. `shaman-hud.mjs`: add `modelTimingSegment()`, wire into `shamanStatus()` and `fallback()`
4. `SKILL.md`: add Bash calls at stage boundaries, pipeline end, cancellation, and failure flows
5. Cancellation/failure flows: add `--cancel`/`--fail` + summary invocation
6. Resume flow: verify `model-timing.json` preservation on `--resume`
