---
name: the-shaman-pipe
description: >
  Single-command pipeline from idea to committed code. Runs a 7-stage fully autonomous workflow:
  Opus-driven design (Socratic clarity interview + approach selection), Opus planning, parallel
  Sonnet implementation against Opus-authored contracts, Haiku validation, Opus security review,
  and Haiku commit — with git worktree isolation and a self-correcting retry loop that classifies
  failures before escalating. Use when the user says "the-shaman-pipe", "shaman pipe", "tsp",
  "full pipeline", "end-to-end", or wants a feature implemented from idea to committed code
  with no manual steps.
license: Apache-2.0
compatibility: Requires git and gh CLI (authenticated)
metadata:
  author: Guipetris
  version: "1.0.0"
---

<Purpose>
The Shaman Pipe takes a single command and autonomously produces committed code through seven sequential, gated stages. Opus owns all design, planning, and review decisions. Sonnet agents implement in parallel against Opus-authored contracts. Haiku handles mechanical validation and environment checks. Every failure routes to an adaptive correction chain with per-unit budgets before escalating to a human. Git worktree isolation enables multiple features to run simultaneously without session collision.

```
/the-shaman-pipe "add stripe webhook handler"
/the-shaman-pipe --ticket GH-142
```
</Purpose>

<Use_When>
- User wants a complete feature implemented end-to-end from a brief description
- User says "the-shaman-pipe", "shaman pipe", "tsp", or "full pipeline"
- Feature requires multiple files, coordinated design, and validation
- User wants contract-enforced implementation with Opus oversight at every gate
- User wants parallel feature development with worktree isolation
- User has a GitHub/Linear ticket and wants automated implementation
</Use_When>

<Do_Not_Use_When>
- User wants a single file fix or small change -- use executor or ralph
- User wants to explore options or brainstorm -- use omc-plan
- User wants only requirements gathering with no implementation -- address the clarity loop directly without the full pipeline
- User wants only a plan without execution -- use ralplan
- User says "just do it" with a fully specified task -- use autopilot
- Project has no git repository and user does not want one initialized
</Do_Not_Use_When>

<Why_This_Exists>
The existing `deep-interview -> ralplan -> autopilot` pipeline chains three independent skills with manual handoffs between them. The Shaman Pipe replaces manual handoffs with gated stages, adds contract-per-unit enforcement so Sonnet agents cannot deviate from Opus's design, introduces worktree-based isolation for concurrent features, and includes a surgical correction system that classifies failures and applies the minimum intervention needed. The result is a single command that produces committed code with architectural integrity enforced at every boundary.
</Why_This_Exists>

<Execution_Policy>
- Eight stages execute sequentially: Brainstorming (pre-stage) -> 0 (Design Refinement) -> 1 (Environment) -> 2 (Planning) -> 3 (Implementation) -> 4 (Validation) -> 5 (Review) -> 6 (Commit)
- Brainstorming ALWAYS runs first — for bare descriptions, --spec, and --ticket invocations alike. It produces a committed spec doc that feeds Stage 0.
- Stage 0 always runs in spec-seeded mode (condensed 3-question refinement), since brainstorming guarantees a spec exists.
- Each stage must pass its gate before the next begins
- Failures at any gate enter the adaptive correction chain before re-attempting or escalating
- The only required human touchpoints are Brainstorming (collaborative design dialogue), Stage 0 (targeted refinement questions), and escalation gates when correction budgets are exhausted
- Optional checkpoint stops between stages can be enabled via `config.json.checkpoint_mode`; when active, the pipeline pauses at each stage boundary for user review before proceeding
- Model routing: Opus (Brainstorming, Stage 0, 2, 5), Sonnet (Stage 3), Haiku (Stages 1, 4, 6)
- Run state is written to .shaman-pipe/state/run-state.json on every stage transition
- Cancel at any time by creating `.shaman-pipe/CANCEL` (e.g. `touch .shaman-pipe/CANCEL`); worktree slot is released, branch archived, state cleared
</Execution_Policy>

<Prerequisites>
```
git must be available and the project must be a git repository (or --init flag must be passed).

gh CLI must be authenticated for branch protection checks and PR creation.
  Run: gh auth status
  If not authenticated, Stage 1 branch protection check falls back gracefully (warns, uses config).

No other external dependencies. The Shaman Pipe manages its own state under .shaman-pipe/.
```
</Prerequisites>

<Steps>

## Invocation

Parse `{{ARGUMENTS}}` to determine invocation mode. **All modes route through brainstorming first.**

**Interactive mode:** `/the-shaman-pipe "feature description"`
- Proceeds to Brainstorming pre-stage with the feature description as seed
- Brainstorming explores, clarifies, designs, writes spec, review loop, user gate
- Spec feeds into Stage 0 (condensed refinement)

**Ticket mode:** `/the-shaman-pipe --ticket {id}`
- Strip any `GH-` or `gh-` prefix from `{id}` before calling the API (e.g. `GH-142` → `142`)
- Fetch ticket via `gh issue view {id} --json title,body,labels`
- Set `"source": "github"`, `"ticket_id": "{id}"`
- Proceeds to Brainstorming pre-stage with the ticket title, body, and labels as seed context
- Brainstorming validates/refines the ticket requirements, writes spec, review loop, user gate
- Spec feeds into Stage 0 (condensed refinement)

**Spec mode:** `/the-shaman-pipe --spec {path}`
- Read the spec document at `{path}` via `Read("{path}")`
- Set `"source": "brainstorm_spec"`, `"spec_path": "{path}"`
- Proceeds to Brainstorming pre-stage with the spec content as seed context
- Brainstorming validates the existing spec, fills gaps, review loop, user gate
- Spec feeds into Stage 0 (condensed refinement)

**Resume mode:** If `.shaman-pipe/state/run-state.json` exists:
- Read state via `Read(".shaman-pipe/state/run-state.json")`
- Extract `run_id` and `stage` from state
- Read `.shaman-pipe/state/{run-id}/design.json` if it exists
- Resume from the last completed stage
- **Timing preservation:** `model-timing.json` from the previous partial run is preserved. Do NOT call `--init` again. The resumed stage gets a fresh `--start-stage` call (gap between crash and resume is not counted). Entries without `ended_at` from the crashed stage are harmlessly skipped by readers.

**Flags:**
- `--init` : Allow git init if no .git directory exists
- `--ticket {id}` : Ticket-driven mode (ticket seeds brainstorming)
- `--spec {path}` : Spec-driven mode (spec seeds brainstorming)
- `--resume {run-id}` : Explicit resume of a specific run

---

## Namespace Bootstrap (run once, idempotent)

Before any stage executes, ensure the `.shaman-pipe/` namespace exists. This is idempotent -- safe to run on every invocation.

**Step 1:** Check if `.shaman-pipe/` directory exists at the project root.

**Step 2:** If it does not exist, create the full directory tree:

```
Bash("mkdir -p .shaman-pipe/worktrees .shaman-pipe/state .shaman-pipe/logs")
```

**Step 3:** Write `config.json` with defaults if it does not exist:

```json
{
  "delivery_mode": "pr",
  "target_branch": "main",
  "pool_size": 3,
  "auto_merge_on_ci_pass": false,
  "test_command": null,
  "package_manager": null,
  "checkpoint_mode": false
}
```

Auto-detect target branch: check if `main` exists (`git branch --list main`), fall back to `master`, fall back to current branch.

```
Write(".shaman-pipe/config.json", configJson)
```

**Step 4:** Write `pool.json` with available slots if it does not exist. Read `pool_size` from `config.json` (default 3) to determine slot count:

```javascript
// Read pool_size from config.json (default 3 if not set)
const poolSize = config.pool_size || 3;
const slots = Array.from({length: poolSize}, (_, i) => ({
  "id": `feature-${i+1}`, "status": "available", "run_id": null,
  "branch": null, "claimed_at": null, "last_heartbeat": null
}));
```

Example with default pool_size of 3:
```json
{
  "slots": [
    { "id": "feature-1", "status": "available", "run_id": null, "branch": null, "claimed_at": null, "last_heartbeat": null },
    { "id": "feature-2", "status": "available", "run_id": null, "branch": null, "claimed_at": null, "last_heartbeat": null },
    { "id": "feature-3", "status": "available", "run_id": null, "branch": null, "claimed_at": null, "last_heartbeat": null }
  ]
}
```

```
Write(".shaman-pipe/worktrees/pool.json", poolJson)
```

**Step 5:** Update `.gitignore` to exclude worktrees and state:

```
Bash("grep -q '.shaman-pipe/worktrees/' .gitignore 2>/dev/null || printf '\n# The Shaman Pipe\n.shaman-pipe/worktrees/\n.shaman-pipe/state/\n' >> .gitignore")
```

Only `.shaman-pipe/worktrees/` and `.shaman-pipe/state/` are excluded. `config.json` and `logs/` can be committed.

---

## Pre-Stage: Brainstorming (Opus)

**Model:** Opus
**Input:** Feature description, ticket data, or existing spec (depending on invocation mode)
**Output:** Committed spec document at `docs/superpowers/specs/YYYY-MM-DD-{topic}-design.md`
**Gate:** Spec must pass automated review loop AND user approval

Brainstorming always runs before Stage 0. It replaces the cold-start Socratic interview with the `/brainstorming` skill's collaborative design process, which produces a higher-quality spec through natural dialogue, scope decomposition, and automated review.

### Invoke the Brainstorming Skill

Invoke `/brainstorming` via the `Skill` tool, passing context based on invocation mode:

**Interactive mode:** Invoke brainstorming with the feature description. Brainstorming explores the codebase, asks one question at a time, proposes 2-3 approaches with a recommendation, presents the design section-by-section, writes a spec doc, runs the spec review loop, and waits for user approval.

**Ticket mode:** Before invoking brainstorming, present the fetched ticket context:
```
I'm starting the design phase for this GitHub ticket:

Title: {ticket.title}
Labels: {ticket.labels}

{ticket.body}

Let me run the brainstorming process to validate and refine these requirements into a proper spec.
```
Then invoke brainstorming. The ticket content gives brainstorming a head start — it will validate assumptions, identify gaps, and refine into a full spec rather than starting from scratch.

**Spec mode:** Before invoking brainstorming, present the existing spec:
```
I'm reviewing this existing spec to validate and refine it:

{spec content}

Let me run the brainstorming process to check for gaps and ensure this is ready for implementation.
```
Then invoke brainstorming. The spec seeds the dialogue — brainstorming validates rather than discovers, focusing on gaps and unstated assumptions.

### Brainstorming Produces

The brainstorming skill handles its own full flow:
1. Project context exploration
2. Clarifying questions (one at a time, scope decomposition if needed)
3. 2-3 approaches with trade-offs and recommendation
4. Design presented section-by-section with user approval
5. Spec document written to `docs/superpowers/specs/YYYY-MM-DD-{topic}-design.md` and committed
6. Automated spec review loop (up to 5 iterations)
7. User review gate — user approves the written spec before proceeding

### After Brainstorming Completes

Record the spec path for Stage 0:
```
spec_path = "docs/superpowers/specs/YYYY-MM-DD-{topic}-design.md"
```

Update run state to transition to Stage 0:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 0,
  "run_id": "{run-id}",
  "phase": "spec-seeded",
  "feature_slug": "{slug}",
  "spec_path": "{spec_path}",
  "started_at": "<ISO timestamp>"
})
```

Proceed to Stage 0 in spec-seeded mode with the brainstorming output.

---

## Stage 0 -- Design Refinement (Opus)

**Model:** Opus
**Input:** Brainstorming spec document (always available — brainstorming pre-stage guarantees it)
**Output:** `.shaman-pipe/state/{run-id}/design.json`
**Gate:** Ambiguity score must be <= 20%

> **Note:** Stage 0 always runs in spec-seeded mode. The brainstorming pre-stage guarantees a spec exists at `spec_path` before Stage 0 begins. The full Socratic interview (Phase A cold-start) is no longer used — brainstorming handles the exploratory design dialogue. Stage 0 performs targeted refinement: scoring the spec's clarity per dimension, asking at most 3 focused questions to fill gaps, then extracting the structured design.json for downstream stages.

### Generate Run ID

```
run_id = "sp-" + first 8 characters of a UUID
```

Generate via:
```
Bash("python3 -c \"import uuid; print('sp-' + uuid.uuid4().hex[:8])\"")
```

Generate feature slug from description: kebab-case, max 40 characters, alphanumeric and hyphens only.

### Create Run State Directory

```
Bash("mkdir -p .shaman-pipe/state/{run-id}/contracts")
```

### Initialize run state

```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 0,
  "run_id": "{run-id}",
  "phase": "A",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Initialize model timing

```
Bash("node ~/.claude/hud/shaman-timing.mjs --init {run-id} \"{slug}\" || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 0 \"Clarity Ritual\" opus || true")
```

### Phase A: Socratic Clarity Loop

The Shaman Pipe implements Socratic questioning directly — no external skill required.

#### Step 1: Detect Project Type

Spawn a Haiku agent to inspect the codebase:

```
Agent(
  subagent_type="general-purpose",
  model="haiku",
  prompt="Check if this directory has existing source code, package files (package.json, pyproject.toml, go.mod, Cargo.toml, pom.xml), or git history. Return JSON: { type: 'brownfield'|'greenfield', evidence: string, relevant_areas: string[] }"
)
```

- **Brownfield**: source files exist AND the feature description references modifying or extending existing functionality
- **Greenfield**: new project or standalone new feature with no existing code context

For brownfield projects, also spawn a targeted explore to map relevant codebase areas:

```
Agent(
  subagent_type="general-purpose",
  model="haiku",
  prompt="Map the codebase areas most relevant to: '{feature description}'. Return file paths, key modules, and existing patterns (auth, data access, routing, etc.) that the feature will touch or extend."
)
```

Store the result as `codebase_context` for use in question generation.

#### Step 2: Initialize Interview State

Write interview state to `.shaman-pipe/state/{run-id}/interview-state.json`:

```json
{
  "interview_id": "{run-id}",
  "type": "greenfield|brownfield",
  "initial_idea": "{feature description}",
  "rounds": [],
  "current_ambiguity": 1.0,
  "threshold": 0.2,
  "codebase_context": null,
  "challenge_modes_used": []
}
```

#### Step 3: Announce to User

Present to the user:

```
Starting design clarity interview. I'll ask targeted questions to understand your feature before building anything. Ambiguity score updates after each answer — we proceed once it drops below 20%.

Feature: "{feature description}"
Project type: {greenfield|brownfield}
Current ambiguity: 100%
```

#### Step 4: Interview Loop

Repeat until `current_ambiguity <= 0.20` OR user exits early OR round 20 reached:

**4a. Generate the next question.**

Identify the dimension with the LOWEST clarity score (start all at 0.0). Build the question targeting that dimension:

| Dimension | Question Style | Weight (greenfield) | Weight (brownfield) |
|---|---|---|---|
| Goal Clarity | "What exactly happens when...?" | 40% | 35% |
| Constraint Clarity | "What are the boundaries?" | 30% | 25% |
| Success Criteria | "How do we know it works?" | 30% | 25% |
| Context Clarity | "How does this fit the existing system?" | N/A | 15% |

Questions must expose assumptions, not gather feature lists. One question per round — never batch.

**Challenge agent injections** (applied to question generation, each used at most once):

- **Round 4+** (Contrarian): "Challenge the user's core assumption. Ask 'What if the opposite were true?' or 'What if this constraint doesn't actually exist?'"
- **Round 6+** (Simplifier): "Probe whether complexity can be removed. Ask 'What's the simplest version that would still be valuable?'"
- **Round 8+, ambiguity > 30%** (Ontologist): "The ambiguity is still high. Ask 'What IS this, really?' or 'If you described this in one sentence to a colleague, what would you say?'"

Track which modes have been used in `challenge_modes_used` to prevent repetition.

**For brownfield questions**: consult `codebase_context` first. Never ask the user about something the codebase already reveals.

**4b. Ask the question via AskUserQuestion.**

```
Round {n} | Targeting: {weakest_dimension} | Ambiguity: {score}%

{question}
```

Include contextually relevant options plus a free-text option.

**4c. Score ambiguity using Opus (temperature 0.1 for consistency).**

Prompt Opus:

```
Score clarity for this {greenfield|brownfield} feature specification across dimensions (0.0 = no clarity, 1.0 = crystal clear):

Feature: {initial_idea}

Interview transcript:
{all rounds Q&A so far}

Score each dimension:
1. Goal Clarity: Is the primary objective unambiguous? Can you state it in one sentence without qualifiers?
2. Constraint Clarity: Are boundaries, limitations, and non-goals clear?
3. Success Criteria: Could you write a passing test right now? Are acceptance criteria concrete and testable?
{4. Context Clarity [brownfield only]: Do we understand the existing system well enough to modify it safely?}

For each dimension return:
- score: float 0.0-1.0
- justification: one sentence
- gap: what's still unclear (omit if score >= 0.9)

Return JSON only.
```

Calculate ambiguity:

```
Greenfield: ambiguity = 1 - (goal × 0.40 + constraints × 0.30 + criteria × 0.30)
Brownfield: ambiguity = 1 - (goal × 0.35 + constraints × 0.25 + criteria × 0.25 + context × 0.15)
```

**4d. Report progress to user.**

```
Round {n} complete.

| Dimension        | Score | Weight | Weighted | Gap           |
|------------------|-------|--------|----------|---------------|
| Goal             | {s}   | {w}%   | {s×w}    | {gap or "—"}  |
| Constraints      | {s}   | {w}%   | {s×w}    | {gap or "—"}  |
| Success Criteria | {s}   | {w}%   | {s×w}    | {gap or "—"}  |
| Context          | {s}   | {w}%   | {s×w}    | {gap or "—"}  |
| **Ambiguity**    |       |        | **{score}%** |           |

{score <= 20 ? "Clarity threshold met. Proceeding to approach design." : "Next question targets: {weakest_dimension}"}
```

**4e. Update interview state.**

Append the round to `interview-state.json`:

```json
{
  "round": {n},
  "question": "{question text}",
  "answer": "{user answer}",
  "scores": { "goal": {s}, "constraints": {s}, "criteria": {s}, "context": {s} },
  "ambiguity": {score},
  "challenge_mode": "{mode or null}"
}
```

Write updated state: `Write(".shaman-pipe/state/{run-id}/interview-state.json", updatedState)`

**4f. Check soft limits.**

- **Round 3+**: If user says "enough", "let's go", "build it", "proceed" — allow early exit.
- **Round 10**: Show soft warning: "10 rounds complete. Ambiguity: {score}%. Continue interviewing or proceed with current clarity?"
- **Round 20**: Hard cap. Proceed with current clarity. Note the risk in design.json.

**Early exit warning (if ambiguity > 20%):**

```
Current ambiguity is {score}% (threshold: 20%). Areas still unclear:
  {list weakest dimensions and their gaps}

Proceeding may require rework. Continue anyway?
```
Options: `["Yes, proceed to approach design", "Ask 2-3 more questions", "Cancel"]`

#### Step 5: Crystallize Interview Spec

When ambiguity ≤ 20% (or early exit accepted), use Opus to generate the spec and write to `.shaman-pipe/state/{run-id}/interview-spec.md`:

```markdown
# Interview Spec: {feature-slug}

## Metadata
- Run ID: {run-id}
- Rounds: {count}
- Final Ambiguity: {score}%
- Type: {greenfield|brownfield}
- Status: {PASSED | EARLY_EXIT}

## Goal
{crystal-clear goal statement}

## Constraints
- {constraint 1}
- {constraint 2}

## Non-Goals
- {excluded scope 1}

## Acceptance Criteria
- [ ] {testable criterion 1}
- [ ] {testable criterion 2}

## Assumptions Exposed
| Assumption | Challenge Applied | Resolution |
|---|---|---|
| {assumption} | {challenge mode} | {decision} |

## Technical Context
{brownfield: codebase findings}
{greenfield: technology constraints}
```

Write: `Write(".shaman-pipe/state/{run-id}/interview-spec.md", specContent)`

#### Step 6: Translation Layer

Extract fields from the interview spec and map to `design.json` fields:

| Interview spec section | design.json field |
|---|---|
| Goal section | `goals[]` (split into discrete statements) |
| Constraints section | `constraints[]` |
| Acceptance Criteria section | `acceptance_criteria[]` |
| Technical Context / codebase findings | `scope[]` (directories referenced) |
| Non-Goals section | `restricted[]` (directories explicitly excluded) |
| Final Ambiguity Score | `ambiguity_score` |

**Resume checkpoint:** After Phase A translation completes and before Phase B begins, write a partial `design.json` so that a crash or interruption during approach selection can be detected and resumed:

```json
{
  "run_id": "{run-id}",
  "feature_slug": "{feature-slug}",
  "ambiguity_score": {score},
  "goals": [...],
  "constraints": [...],
  "acceptance_criteria": [...],
  "scope": [...],
  "restricted": [...],
  "source": "interview",
  "ticket_id": null,
  "chosen_approach": null
}
```

```
Write(".shaman-pipe/state/{run-id}/design.json", partialDesignJson)
```

On resume: if `design.json` exists but `chosen_approach` is `null`, skip Phase A and re-enter Phase B to re-present approaches to the user.

### Phase B: Approach Design

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 0,
  "run_id": "{run-id}",
  "phase": "B",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

**Step 1: Opus proposes 2-3 architectural approaches.**

For each approach, include:
- Name (short, descriptive)
- Description (2-3 sentences)
- Trade-offs (pros and cons)
- Explicitly rejected alternatives and why

Present approaches to user with trade-off analysis. The user selects one.

Use `AskUserQuestion` to present:

```
Your design is clear (ambiguity: {score}%). I've identified {N} architectural approaches:

**Approach 1: {name}**
{description}
+ {pro}
- {con}

**Approach 2: {name}**
{description}
+ {pro}
- {con}

{**Approach 3: {name}** (if applicable)}

Which approach should I implement?
```

Options: `["Approach 1: {name}", "Approach 2: {name}", "Approach 3: {name}"]`

**Step 2: First-run config questions (only if `.shaman-pipe/config.json` still has defaults).**

If this is the first run (config has not been user-confirmed), ask three questions:

Question 1 via `AskUserQuestion`:
```
How should completed features be delivered?
```
Options: `["PR to target branch (recommended)", "Direct merge to target branch"]`

Question 2 via `AskUserQuestion`:
```
What is the target branch?
```
Options: `["{detected-main-branch}", "Other (specify)"]`

Question 3 via `AskUserQuestion`:
```
Would you like checkpoint stops between stages?

With checkpoints ON, the pipeline pauses after each stage and shows you what was built
before asking whether to proceed. Useful if you want to review the plan, contracts, or
implementation before they become irreversible. You can still cancel at any time via
`touch .shaman-pipe/CANCEL` regardless of this setting.
```
Options: `["No — run fully autonomous (recommended)", "Yes — pause between each stage for review"]`

Persist answers to `.shaman-pipe/config.json`:
```
Read(".shaman-pipe/config.json")
Edit(".shaman-pipe/config.json", old_delivery_mode, new_delivery_mode)
Edit(".shaman-pipe/config.json", "\"checkpoint_mode\": false", "\"checkpoint_mode\": {true|false per answer}")
```

### Write design.json

After Phase A translation and Phase B selection, write the complete `design.json`:

```json
{
  "run_id": "sp-a3f2c1d9",
  "feature_slug": "stripe-webhook-handler",
  "ambiguity_score": 0.14,
  "goals": ["validate stripe signature", "persist webhook event", "return 200"],
  "constraints": ["must not modify billing schema", "idempotent on retry"],
  "acceptance_criteria": ["signature validation test passes", "event persisted to events table"],
  "chosen_approach": "webhook-first with idempotency key",
  "rejected_approaches": ["polling-based", "queue-backed async"],
  "scope": ["src/payments/"],
  "restricted": ["src/billing/schema.ts"],
  "source": "interview",
  "ticket_id": null
}
```

```
Write(".shaman-pipe/state/{run-id}/design.json", designJson)
```

---

## Stage 0: Spec-Seeded Behaviour (`--spec` mode)

When invoked with `--spec {path}`, Stage 0 replaces blank-slate discovery with a targeted refinement loop. The brainstorm spec is treated as prior work — already explored, already approved by the user — so Opus refines rather than discovers.

### Step 1: Parse the spec

Read the spec document and extract:
- **Goal** — what the feature/change is meant to accomplish
- **Constraints** — tech stack, scope limits, non-goals stated in the spec
- **Acceptance criteria** — any success conditions or test cases mentioned
- **Proposed approaches** — any design options or recommendations in the spec

Assess each dimension on the same 0–100% clarity scale used in Phase A. A well-written brainstorm spec will typically land at 30–60% ambiguity overall (goal is clear, implementation details are not).

### Step 2: Targeted refinement (Phase A condensed)

Run a condensed Phase A using only the ambiguity dimensions that the spec left unresolved:
- Skip any dimension already scored ≤ 20% ambiguous
- Ask at most **3 targeted questions** total (vs. the normal 8-round ceiling)
- Challenge agent fires once if any answer introduces a new unknown
- Exit when overall ambiguity ≤ 20% or question budget is exhausted

Summarise the spec to the user at the start:
> "I've read the brainstorm spec at `{path}`. I have a clear picture of the goal and constraints. I need to clarify a few implementation details before we pick an approach."

### Step 3: Approach selection (Phase B — unchanged)

Run standard Phase B. If the spec already proposes approaches, surface them as the candidate set and add any Opus-generated alternatives. User selects; Opus authors contracts as normal.

### Ambiguity score in design.json

Set `"ambiguity_score"` to the final post-refinement value (not the spec's initial score), so the PR body and review stage reflect actual remaining uncertainty.

---

## Stage 1 -- Environment Check (Haiku)

**Model:** Haiku
**Input:** `design.json`, project environment
**Output:** Claimed worktree slot with feature branch
**Gate:** All 7 checks must pass (warnings allowed, errors block)

Check for cancel sentinel: if `.shaman-pipe/CANCEL` exists, enter cancellation flow (see Cancellation section).
If `config.checkpoint_mode` is `true`, run the Stage 0→1 checkpoint (see Checkpoint Gate section).

End Stage 0 timing, start Stage 1:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 0 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 1 \"Environment Reading\" haiku || true")
```

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 1,
  "run_id": "{run-id}",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Check 1: Git Repo Check

```
Bash("test -d .git && echo 'OK' || echo 'NO_GIT'")
```

- If `NO_GIT` and `--init` flag was NOT passed: **ERROR**. Stop pipeline.
  > "No git repository found. Run with --init flag to initialize one, or navigate to a git repository."
- If `NO_GIT` and `--init` flag WAS passed:
  ```
  Bash("git init")
  ```
  Generate a stack-appropriate `.gitignore` (detect by existing files: `package.json` -> Node, `requirements.txt`/`pyproject.toml` -> Python, `Cargo.toml` -> Rust, etc.).
  ```
  Bash("git add -A && git commit -m 'Initial commit'")
  ```

### Check 2: Worktree Pool Bootstrap

Verify `.shaman-pipe/` namespace exists. If not, run the Namespace Bootstrap section above. If `pool.json` exists, verify slot integrity:

```
Read(".shaman-pipe/worktrees/pool.json")
```

Validate: all slots have required fields (`id`, `status`, `run_id`, `branch`, `claimed_at`, `last_heartbeat`). If corrupted, regenerate with defaults.

### Check 3: Branch Protection Check

```
Bash("gh api repos/{owner}/{repo}/branches/{target_branch}/protection 2>/dev/null && echo 'PROTECTED' || echo 'NOT_PROTECTED_OR_UNAVAILABLE'")
```

Extract `{owner}/{repo}` from:
```
Bash("git remote get-url origin 2>/dev/null | sed -E 's|.*github.com[:/]([^/]+/[^/.]+).*|\\1|'")
```

- If `PROTECTED`: override `config.json` delivery mode to `"pr"` regardless of user preference.
- If `NOT_PROTECTED_OR_UNAVAILABLE`: log warning, use `config.json` value. Do not block.

### Check 4: Clean Working Tree

```
Bash("git status --porcelain")
```

- If output is non-empty: **ERROR**. Stop pipeline.
  > "Uncommitted changes detected. Commit or stash changes before running the-shaman-pipe."

### Check 5: Conflict Scan

Read active worktrees from `pool.json`. For each claimed slot, compare its `design.json` scope against the current run's scope:

```
Read(".shaman-pipe/worktrees/pool.json")
```

For each claimed slot with a `run_id`:
```
Read(".shaman-pipe/state/{other-run-id}/design.json")
```

Compare `scope[]` arrays for directory-level overlap. If overlap detected: **WARN** (do not block).

> "Warning: Active run {other-run-id} overlaps scope directory '{dir}'. File-level conflicts will be caught at Stage 3 merge."

### Check 6: Duplication Scan

Search the codebase for existing implementations that overlap the feature scope. Use Grep to search for key terms from `design.json` goals:

```
Grep(pattern="{key term from goal}", path="{scope directory}")
```

If significant overlap detected: **WARN** (do not block).

> "Warning: Existing implementation found at {file}:{line} that may overlap with this feature."

### Check 7: Pool Slot Claim

**Atomic claim protocol:**

Step 1: Read current pool state:
```
Read(".shaman-pipe/worktrees/pool.json")
```

Step 2: Find first slot with `status: "available"`.

Step 3: If no available slot, check for stale slots (heartbeat older than 10 minutes):
```python
stale = now - last_heartbeat > 600 seconds
```
If stale slot found, release it with warning:
```
> "Warning: Releasing stale slot {slot-id} (last heartbeat: {timestamp}). Previous run {run-id} may have crashed."
```

Step 4: If still no available slot, queue and wait:
```
Bash("sleep 5")
```
Then retry from Step 1. Do not fail.

Step 5: Atomic claim via temp file + rename:
```
Bash("cat > .shaman-pipe/worktrees/pool.{pid}.tmp << 'POOL_EOF'\n{updated pool.json with slot claimed}\nPOOL_EOF\nmv .shaman-pipe/worktrees/pool.{pid}.tmp .shaman-pipe/worktrees/pool.json")
```

The claimed slot should contain:
```json
{
  "id": "feature-{N}",
  "status": "claimed",
  "run_id": "sp-a3f2c1d9",
  "branch": "feature/sp-a3f2c1d9-stripe-webhook-handler",
  "claimed_at": "2026-03-13T10:00:00Z",
  "last_heartbeat": "2026-03-13T10:00:00Z"
}
```

### Create Worktree

After all checks pass:

```
Bash("git worktree add .shaman-pipe/worktrees/{slot-id} -b feature/sp-{run-id}-{feature-slug}")
```

This creates an isolated working directory with its own feature branch.

---

## Stage 2 -- Planning (Opus)

**Model:** Opus
**Input:** `design.json` + codebase context
**Output:** `feature-plan.json` + `contracts/{unit-id}.json` for each unit
**Gate:** All contracts pass independent Critic review

Check for cancel sentinel: if `.shaman-pipe/CANCEL` exists, enter cancellation flow (see Cancellation section).
If `config.checkpoint_mode` is `true`, run the Stage 1→2 checkpoint (see Checkpoint Gate section).

End Stage 1 timing, start Stage 2:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 1 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 2 \"Contract Ceremony\" opus || true")
```

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 2,
  "run_id": "{run-id}",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Opus Challenge-the-Brief Gate

Before generating any contracts, Opus evaluates `design.json` for soundness:

Read the design:
```
Read(".shaman-pipe/state/{run-id}/design.json")
```

Explore the codebase within the declared scope:
```
Glob(pattern="**/*", path="{scope directories}")
Read({key files in scope})
```

Opus must answer these questions before proceeding:
1. Are the goals achievable within the declared scope?
2. Do the constraints conflict with each other?
3. Are the acceptance criteria testable?
4. Would implementing this feature cause architectural harm to existing code?

If concerns are found, Opus surfaces them and proposes modified scope. Does NOT silently proceed.

Use `AskUserQuestion` if scope modification requires user approval:
```
I've identified a concern with the design:
{concern description}

Proposed modification: {modification}

Proceed with modification, or continue with original scope?
```

### Generate Feature Plan

Opus reads `design.json` and the full codebase context within scope, then produces `feature-plan.json`:

```json
{
  "run_id": "sp-a3f2c1d9",
  "units": [
    {
      "unit_id": "unit-event-schema",
      "files": ["src/payments/types.ts"],
      "depends_on": []
    },
    {
      "unit_id": "unit-webhook-handler",
      "files": ["src/payments/webhook.ts"],
      "depends_on": ["unit-event-schema"]
    }
  ],
  "merge_order": ["unit-event-schema", "unit-webhook-handler"],
  "parallelism": 2,
  "restricted_files": ["src/billing/schema.ts"]
}
```

**Constraints on the feature plan:**
- `merge_order` must be a valid topological sort of the dependency graph (no cycles)
- `files[]` must be disjoint across units (no file appears in two units)
- `restricted_files` must include all files from `design.json.restricted`
- `parallelism` should not exceed the number of independent units (units with no unresolved dependencies)

```
Write(".shaman-pipe/state/{run-id}/feature-plan.json", featurePlanJson)
```

### Generate Contract Set

One contract per unit. Each contract is Opus's design decision -- not a transcription of the feature description.

**Contract schema:**

```json
{
  "unit_id": "unit-webhook-handler",
  "files": ["src/payments/webhook.ts"],
  "inputs": { "request": "WebhookRequest", "signature": "string" },
  "outputs": { "response": "WebhookResponse", "persisted": "WebhookEvent" },
  "function_signatures": ["handleWebhook(req: WebhookRequest): Promise<WebhookResponse>"],
  "side_effects": ["INSERT into events table", "no external calls"],
  "dependencies": ["unit-event-schema"],
  "restricted_from": ["src/billing/schema.ts"]
}
```

**Contract fields explained:**
- `unit_id`: Unique identifier matching the feature plan unit
- `files`: Exact files this unit will create or modify
- `inputs`: Named input parameters with their types
- `outputs`: Named output values with their types
- `function_signatures`: Exact function signatures the implementation must expose
- `side_effects`: All side effects this unit performs (DB writes, API calls, file I/O)
- `dependencies`: Other units this unit depends on (must be implemented first or in parallel with merge)
- `restricted_from`: Files this unit's agent is forbidden from touching

Write each contract:
```
Write(".shaman-pipe/state/{run-id}/contracts/{unit-id}.json", contractJson)
```

### Opus's Explicit Powers in Stage 2

1. **Challenge the brief** -- if `design.json` is ambiguous or would cause architectural harm, Opus surfaces this before generating contracts. Can propose modified scope. Does not silently proceed.
2. **Own the contracts** -- every contract is Opus's design decision. Sonnet agents cannot override contract shapes.
3. **Set the blast radius** -- `restricted_from` field in each contract enforces which files agents cannot touch. Enforced at the Sonnet contract gate (Stage 3), not on trust.

### Critic Pass (Independent Contract Review)

After Opus authors all contracts, run an independent Critic validation against `design.json`. This is a separate evaluation pass -- Opus reviewing its own contracts without an independent pass is insufficient.

**Critic checks:**

1. **Coverage check:** Do contracts collectively cover all acceptance criteria from `design.json`?
   - For each acceptance criterion, at least one contract must address it through its `outputs`, `side_effects`, or `function_signatures`.

2. **Approach adherence:** Does the `chosen_approach` from Phase B reflect in the contract design?
   - Contract structure should implement the selected approach, not an alternative.

3. **Internal consistency:** Are any contracts internally contradictory?
   - No contract should declare an output that conflicts with another contract's input expectations.
   - No circular dependencies in the dependency graph.

4. **Restriction enforcement:** Are restricted files honored in all contracts?
   - Every file in `design.json.restricted` must appear in every contract's `restricted_from`.
   - No contract's `files[]` should overlap with `restricted_files` from the feature plan.

**Critic execution:**

Read all contracts and design.json, then evaluate each check. For each failed check, produce a structured failure:

```json
{
  "check": "coverage",
  "failed_criterion": "event persisted to events table",
  "reason": "No contract declares a side effect for persisting events",
  "affected_contracts": ["unit-webhook-handler"]
}
```

**Rewrite loop:** Failed contracts are sent back to Opus for rewriting with the Critic's failure context. Opus rewrites only the affected contracts. The Critic re-validates. This loop runs at most 3 times. If contracts still fail after 3 Critic iterations, the pipeline halts with a structured error:

> "Contract design failed Critic review after 3 iterations. Last failures: {failure list}. Manual review required."

Stage 3 does NOT begin until all contracts pass the Critic.

---

## Stage 3 -- Implementation (Sonnet x N, parallel)

**Model:** Sonnet (one per unit, up to `parallelism` concurrent)
**Input:** Per-agent: its contract JSON + files listed in contract as codebase context
**Output:** Implemented code on sub-branches, merged into feature branch
**Gate:** All units pass inline contract gate + merge protocol completes

Check for cancel sentinel: if `.shaman-pipe/CANCEL` exists, enter cancellation flow (see Cancellation section).
If `config.checkpoint_mode` is `true`, run the Stage 2→3 checkpoint (see Checkpoint Gate section).

End Stage 2 timing, start Stage 3:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 2 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 3 \"The Summoning\" sonnet || true")
```

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 3,
  "run_id": "{run-id}",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Agent Progress Tracking

Initialize agent progress (visible in HUD statusline):
```
Write(".shaman-pipe/state/{run-id}/agent-status.json", {
  "phase": "implementing",
  "total": {number of units from feature-plan.json},
  "parallelism": {parallelism from feature-plan.json},
  "agents": [
    { "id": "{unit-id}", "status": "pending" }
    // ... one entry per unit from feature-plan.json
  ],
  "merge_progress": 0
})
```

Report to user:
> "Stage 3: The Summoning -- {total} agents, parallelism {P}"

### Branch Topology

```
main
 +-- feature/sp-{run-id}-{slug}                         <-- main feature branch
      (lives in .shaman-pipe/worktrees/{slot-id}/)
      |
      +-- feature/{run-id}/unit-{unit-id-1}              <-- agent sub-branch
          (.shaman-pipe/worktrees/{slot-id}/agents/{unit-id-1}/)
      +-- feature/{run-id}/unit-{unit-id-2}              <-- agent sub-branch
          (.shaman-pipe/worktrees/{slot-id}/agents/{unit-id-2}/)
```

**IMPORTANT:** Git does not allow multiple branches checked out simultaneously in a single worktree folder. Parallel agents MUST each have their own worktree path. Each agent sub-branch is created via `git worktree add` into a dedicated subfolder, not `git checkout -b` in the shared slot folder. After merge, agent sub-worktrees are removed.

Pool slots remain at 3 (3 simultaneous features). Within each slot, N agent sub-worktrees are created temporarily for Stage 3 and cleaned up after the merge protocol completes.

### Per-Agent Lifecycle

For each unit in `feature-plan.json`, spawn a Sonnet agent via TaskCreate.

Before spawning each agent, record timing:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --start-agent 3 {unit-id} sonnet || true")
```

```
TaskCreate(
  subagent_type="general-purpose",
  model="sonnet",
  description="Implement unit {unit-id} per contract",
  context={
    "contract": {contract JSON},
    "codebase_files": {content of files listed in contract.files + dependency contract files},
    "worktree_path": ".shaman-pipe/worktrees/{slot-id}",
    "run_id": "{run-id}",
    "unit_id": "{unit-id}"
  }
)
```

**Before spawning agents, create sub-worktrees for each unit:**

```
Bash("git worktree add .shaman-pipe/worktrees/{slot-id}/agents/{unit-id} -b feature/{run-id}/unit-{unit-id} feature/sp-{run-id}-{slug}")
```

Each agent gets a fully isolated working directory. The sub-branch is created off the feature branch HEAD at Stage 3 start.

**Agent instructions (injected into each TaskCreate):**

```
You are implementing unit "{unit-id}" for the-shaman-pipe run "{run-id}".

Your contract:
{contract JSON}

Instructions:
1. Your isolated worktree is at: {agent_worktree_path}
   (.shaman-pipe/worktrees/{slot-id}/agents/{unit-id}/)
2. All your work happens in this directory -- do NOT navigate outside it
3. Implement the files listed in your contract
4. Your implementation MUST match:
   - Function signatures exactly as declared in contract.function_signatures
   - Return shapes matching contract.outputs
   - Side effects matching contract.side_effects
   - You MUST NOT touch any file in contract.restricted_from
5. After implementation, commit your changes:
   git add {contract.files}
   git commit -m "feat({unit-id}): implement {unit-id} per contract"
6. Report completion

RESTRICTED FILES (do NOT read, modify, or import from):
{contract.restricted_from}
```

**Parallelism control:** Spawn agents up to the `parallelism` value from the feature plan. If `parallelism` is 2 and there are 4 units, spawn the first 2, wait for them to complete, then spawn the next 2 (respecting dependency order).

Units with unresolved dependencies must wait for their dependencies to complete and merge first.

**Agent status updates:** Before spawning each batch, update `agent-status.json` -- set batch agents to `"running"` and write the full file. Report to user which agents are starting:
> "Batch {N}: Starting agents {unit-id-1}, {unit-id-2}..."

### Inline Contract Gate (per agent, Sonnet validates)

After each agent reports completion, validate its output against the contract:

**Step 1: Signature check**
```
Grep(pattern="{each function_signature}", path="{worktree}/{contract.files}")
```
Verify every declared function signature exists in the implemented files.

**Step 2: Return shape check**
Read the implemented files and verify return types match `contract.outputs`.

**Step 3: Side effects check**
Verify declared side effects are present (e.g., DB operations, API calls).

**Step 4: Restricted files check**
```
Bash("cd .shaman-pipe/worktrees/{slot-id}/agents/{unit-id} && git diff HEAD --name-only")
```
Verify no file in `contract.restricted_from` appears in the diff.

If all checks pass, record agent timing completion and update status:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-agent 3 {unit-id} || true")
```
Update `agent-status.json`: set agent {unit-id} status to `"completed"` and write the full file.
Report: > "Agent {unit-id} completed -- contract gate passed"

If any check fails: update agent status to `"correcting"` in `agent-status.json`, then enter the Adaptive Correction System (see below) for this unit. After correction succeeds, update status to `"completed"`.

### Heartbeat

Update the pool slot heartbeat every 5 minutes during **all stages that hold a slot (Stages 1–6)**. Update at each stage transition and periodically during long-running stages (Stage 3 implementation, Stage 4 validation, Stage 5 review).

```
Read(".shaman-pipe/worktrees/pool.json")
```
Update the claimed slot's `last_heartbeat` to current ISO timestamp.
```
Write(".shaman-pipe/worktrees/pool.json", updatedPoolJson) -- via atomic temp+rename
```

A slot with no heartbeat update for 10 minutes is considered stale and may be reclaimed by another invocation.

### Merge Protocol (sequential, dependency-sorted)

Once ALL agents report done and pass their contract gates, a dedicated merge sequence executes in `merge_order` from the feature plan.

Update agent status for merge phase:
```
Read(".shaman-pipe/state/{run-id}/agent-status.json")
```
Set `"phase": "merging"`, `"merge_progress": 0`. Write back.
Report: > "All {N} agents completed. Starting merge protocol."

**For each unit in merge_order:**

After each successful unit merge, increment `merge_progress` in `agent-status.json` and write back.

**Step 1: Merge sub-branch into feature branch**
```
Bash("cd {worktree_path} && git checkout feature/sp-{run-id}-{slug} && git merge feature/{run-id}/unit-{unit-id} --no-ff -m 'merge: unit-{unit-id} into feature branch'")
```

**Step 2: Re-run contract gate on merged result**
Run the full contract gate (signatures, return shapes, side effects, restricted files) on the merged state -- not just the unit in isolation.

**Step 3: If merge conflict**
Sonnet merge agent resolves using contract definitions as ground truth. Contracts define the correct interface, not the conflicting code.

```
Bash("cd {worktree_path} && git diff --name-only --diff-filter=U")
```

For each conflicted file, read the conflict markers:
```
Read("{worktree_path}/{conflicted_file}")
```

Resolve by selecting the implementation that matches the contract for the file's owning unit. Edit the file to remove conflict markers:
```
Edit("{worktree_path}/{conflicted_file}", conflict_section, resolved_section)
```

Then complete the merge:
```
Bash("cd {worktree_path} && git add {resolved_files} && git commit -m 'resolve: merge conflict in {files} using contract ground truth'")
```

**Step 4: Re-run contract gate after resolution**
If the resolved merge still fails the contract gate, the conflicting units are re-implemented sequentially (not in parallel) with explicit awareness of each other's contracts:

```
TaskCreate(
  subagent_type="general-purpose",
  model="sonnet",
  description="Re-implement unit {unit-id} sequentially after merge conflict",
  context={
    "contract": {contract JSON},
    "conflicting_contract": {other unit's contract JSON},
    "merged_state": {current merged code},
    "worktree_path": "{worktree_path}",
    "instruction": "Implement with awareness of the conflicting unit's contract. Your implementation must coexist."
  }
)
```

**Step 5: Repeat for each unit in merge_order**

After all units are merged, the feature branch contains the complete implementation.

---

## Stage 4 -- Validation (Haiku)

**Model:** Haiku
**Input:** Merged feature branch in worktree
**Output:** Validation report (pass/fail per check)
**Gate:** All validation checks pass (failures enter correction chain)

Check for cancel sentinel: if `.shaman-pipe/CANCEL` exists, enter cancellation flow (see Cancellation section).
If `config.checkpoint_mode` is `true`, run the Stage 3→4 checkpoint (see Checkpoint Gate section).

End Stage 3 timing, start Stage 4:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 3 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 4 \"Validation Rite\" haiku || true")
```

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 4,
  "run_id": "{run-id}",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Step 1: Detect Package Manager

Check for lockfiles in the worktree:

```
Bash("cd {worktree_path} && ls package-lock.json yarn.lock pnpm-lock.yaml poetry.lock Pipfile.lock Cargo.lock go.sum 2>/dev/null")
```

| Lockfile | Package Manager | Install Command |
|---|---|---|
| `package-lock.json` | npm | `npm ci` |
| `yarn.lock` | yarn | `yarn install --frozen-lockfile` |
| `pnpm-lock.yaml` | pnpm | `pnpm install --frozen-lockfile` |
| `poetry.lock` | poetry | `poetry install` |
| `Pipfile.lock` | pipenv | `pipenv install` |
| `Cargo.lock` | cargo | `cargo build` |
| `go.sum` | go | `go mod download` |

Override with `config.json.package_manager` if set.

### Step 2: Install Dependencies

```
Bash("cd {worktree_path} && {install_command}")
```

### Step 3: Test Suite Check

If no test files exist in the project:

```
Bash("cd {worktree_path} && find . -name '*.test.*' -o -name '*.spec.*' -o -name 'test_*' -o -name '*_test.*' | head -5")
```

If no tests found, spawn a Sonnet agent to generate a baseline test suite covering the contracts:

```
TaskCreate(
  subagent_type="general-purpose",
  model="sonnet",
  description="Generate baseline test suite for the-shaman-pipe contracts",
  context={
    "contracts": {all contract JSONs},
    "design": {design.json},
    "worktree_path": "{worktree_path}",
    "instruction": "Generate tests that verify each acceptance criterion from design.json. Cover function signatures, return shapes, and side effects from contracts. Not comprehensive -- enough to validate acceptance criteria."
  }
)
```

### Step 4: Run Validation Suite

**Tests:**
```
Bash("cd {worktree_path} && {test_command}")
```

Auto-detect test command if `config.json.test_command` is null:
| Indicator | Test Command |
|---|---|
| `package.json` with `test` script | `npm test` |
| `pytest.ini` or `conftest.py` | `pytest` |
| `Cargo.toml` | `cargo test` |
| `go.mod` | `go test ./...` |

**Type check:**
| Indicator | Type Check Command |
|---|---|
| `tsconfig.json` | `npx tsc --noEmit` |
| `mypy.ini` or `[mypy]` in setup.cfg | `mypy .` |
| `pyrightconfig.json` | `pyright` |

```
Bash("cd {worktree_path} && {typecheck_command}")
```

**Lint:**
| Indicator | Lint Command |
|---|---|
| `.eslintrc*` or `eslint.config.*` | `npx eslint .` |
| `ruff.toml` or `[tool.ruff]` | `ruff check .` |
| `Cargo.toml` | `cargo clippy` |

```
Bash("cd {worktree_path} && {lint_command}")
```

### Step 5: Acceptance Criteria Mapping

For each acceptance criterion in `design.json`, Haiku maps it to a specific test result or observable output:

```
| Criterion | Evidence | Status |
|---|---|---|
| "signature validation test passes" | test_webhook.py::test_signature_validation PASSED | PASS |
| "event persisted to events table" | test_webhook.py::test_event_persistence PASSED | PASS |
```

Criteria without corresponding test results are flagged for Stage 5 (Opus review).

### Validation Report

Produce a structured validation report:

```json
{
  "tests": { "status": "pass", "passed": 12, "failed": 0, "skipped": 1 },
  "typecheck": { "status": "pass", "errors": 0 },
  "lint": { "status": "pass", "warnings": 3, "errors": 0 },
  "acceptance_criteria": [
    { "criterion": "signature validation test passes", "status": "pass", "evidence": "test_webhook.py::test_signature_validation" },
    { "criterion": "event persisted to events table", "status": "pass", "evidence": "test_webhook.py::test_event_persistence" }
  ]
}
```

Persist the report for Stage 5 to reference on resume:
```
Write(".shaman-pipe/state/{run-id}/validation-report.json", validationReport)
```

Any failure enters the Adaptive Correction System. Success gates Stage 5.

---

## Stage 5 -- Review (Opus)

**Model:** Opus
**Input:** Merged feature branch, design.json, all contracts
**Output:** Verdict (APPROVE, FIX_MINOR, RE-PLAN, REJECT)
**Gate:** Verdict must be APPROVE to proceed to Stage 6

Check for cancel sentinel: if `.shaman-pipe/CANCEL` exists, enter cancellation flow (see Cancellation section).
If `config.checkpoint_mode` is `true`, run the Stage 4→5 checkpoint (see Checkpoint Gate section).

End Stage 4 timing, start Stage 5:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 4 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 5 \"Final Judgment\" opus || true")
```

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 5,
  "run_id": "{run-id}",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Three-Lens Review

Opus reviews the implementation against the contracts it designed -- closed-loop validation with authorship context.

**Lens 1: Security**
- Trust boundaries: Does the implementation properly validate inputs at boundaries?
- Input validation: Are all external inputs sanitized?
- Secrets exposure: Are credentials, API keys, or tokens hardcoded?
- OWASP surface: SQL injection, XSS, CSRF, path traversal, etc.
- Blast radius: Does the implementation touch anything beyond its declared scope?

**Lens 2: Code Quality**
- Contract adherence: Does the implementation match contract signatures, inputs, outputs, and side effects?
- Anti-patterns: God objects, deep nesting, magic numbers, copy-paste duplication
- Coupling: Does the implementation introduce tight coupling to external modules not declared in the contract?

**Lens 3: Acceptance Criteria**
- Compare each criterion from `design.json` against the actual implementation
- Does the code actually deliver what Stage 0 specified?
- Reference the Stage 4 validation report for test evidence

### Verdicts

**`APPROVE`** -- All three lenses pass. Proceed to Stage 6.

**`FIX_MINOR`** -- Small issues found that Sonnet can fix with direction.

Opus produces a structured fix list:
```json
{
  "verdict": "FIX_MINOR",
  "fixes": [
    {
      "file": "src/payments/webhook.ts",
      "line": 42,
      "issue": "Missing input validation on webhook signature",
      "fix_guidance": "Add signature length check before crypto.verify call"
    }
  ]
}
```

Action: Spawn Sonnet to apply fixes in the worktree:
```
TaskCreate(
  subagent_type="general-purpose",
  model="sonnet",
  description="Apply FIX_MINOR patches per Opus review",
  context={
    "fixes": {fix list from Opus},
    "worktree_path": "{worktree_path}"
  }
)
```

After Sonnet patches: re-run Stage 4 (Validation) -> re-run Stage 5 (Review). This loop is bounded by the correction budget.

**`RE-PLAN`** -- Fundamental contract violation or architecture mismatch.

Opus rewrites ONLY the affected contracts. Stage 3 re-runs for those units ONLY -- not the full implementation.

Action:
1. Opus identifies affected unit IDs
2. Opus rewrites their contracts in `.shaman-pipe/state/{run-id}/contracts/{unit-id}.json`
3. New sub-branches are created for affected units: `feature/{run-id}/unit-{unit-id}-v2`
4. Sonnet agents re-implement only those units
5. Merge protocol replays the full merge order but skips units that are already cleanly merged (detected via `git merge-base --is-ancestor`)
6. Re-enter at Stage 4

**`REJECT`** -- Unacceptable security risk or feature would cause architectural harm.

Action:
1. Write structured rejection rationale to `.shaman-pipe/state/{run-id}/rejection.json`:
   ```json
   {
     "verdict": "REJECT",
     "reason": "Implementation introduces SQL injection vulnerability in webhook handler that cannot be patched without redesigning the data access layer",
     "affected_files": ["src/payments/webhook.ts"],
     "recommendation": "Redesign data access to use parameterized queries throughout"
   }
   ```
2. Archive the feature branch (do not delete)
3. Release the pool slot
4. Record timing and print summary:
   ```
   Bash("node ~/.claude/hud/shaman-timing.mjs --fail || true")
   Bash("node ~/.claude/hud/shaman-summary.mjs --run-id {run-id} || true")
   ```
5. Clear run state: `Bash("rm -f .shaman-pipe/state/run-state.json")`
6. Pipeline terminates. Human intervention required.

> "Pipeline REJECTED at Stage 5 review. Reason: {reason}. Branch preserved at feature/sp-{run-id}-{slug} for inspection. See .shaman-pipe/state/{run-id}/rejection.json for details."

---

## Stage 6 -- Commit (Haiku)

**Model:** Haiku
**Input:** Approved feature branch, design.json, config.json
**Output:** PR or merged commit
**Gate:** Successful PR creation or merge

Check for cancel sentinel: if `.shaman-pipe/CANCEL` exists, enter cancellation flow (see Cancellation section).
If `config.checkpoint_mode` is `true`, run the Stage 5→6 checkpoint (see Checkpoint Gate section).

End Stage 5 timing, start Stage 6:
```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 5 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --start-stage 6 \"The Sealing\" haiku || true")
```

Update run state:
```
Write(".shaman-pipe/state/run-state.json", {
  "active": true,
  "stage": 6,
  "run_id": "{run-id}",
  "feature_slug": "{slug}",
  "started_at": "<ISO timestamp>"
})
```

### Read Config

```
Read(".shaman-pipe/config.json")
```

Determine delivery mode: `"pr"` or `"merge"`.

### Commit Message Format

Build the commit message from `design.json`:

```
feat({scope}): {feature_slug as human-readable description}

Implements contract set sp-{run-id}. {Closes #{ticket_id}. | ""}

Contracts: {comma-separated unit-id list from feature-plan.json}
Approach: {chosen_approach from design.json}
Ambiguity score at design: {ambiguity_score * 100}%

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
```

Where `{scope}` is derived from the primary scope directory in `design.json` (e.g., `payments` from `src/payments/`).

### PR Mode

```
Bash("cd {worktree_path} && git push -u origin feature/sp-{run-id}-{slug}")
```

```
Bash("cd {worktree_path} && gh pr create --title 'feat({scope}): {description}' --body \"$(cat <<'PR_EOF'\n## Summary\n\nImplements contract set sp-{run-id}.{ticket_close_line}\n\n## Contracts\n{unit list with descriptions}\n\n## Approach\n{chosen_approach}\n\nAmbiguity score at design: {ambiguity_score * 100}%\n\n## Acceptance Criteria\n{criteria list with pass/fail from Stage 4}\n\nCo-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>\nPR_EOF\n)\" --base {target_branch}")
```

If `config.json.auto_merge_on_ci_pass` is true:
```
Bash("gh pr merge --auto --squash")
```

### Direct Merge Mode

```
Bash("git checkout {target_branch} && git merge feature/sp-{run-id}-{slug} --no-ff -m \"$(cat <<'MSG_EOF'\nfeat({scope}): {description}\n\nImplements contract set sp-{run-id}.{ticket_close_line}\n\nContracts: {unit-id list}\nApproach: {chosen_approach}\nAmbiguity score at design: {ambiguity_score * 100}%\n\nCo-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>\nMSG_EOF\n)\"")
```

### Cleanup

**Step 1: Remove agent sub-worktrees** (must happen before branch deletion)
```
Bash("git worktree list --porcelain | grep '.shaman-pipe/worktrees/{slot-id}/agents' | awk '/worktree/{print $2}' | xargs -r -I{} git worktree remove {} --force")
```

**Step 2: Remove feature worktree** (must happen before branch deletion — worktree references the branch)
```
Bash("git worktree remove .shaman-pipe/worktrees/{slot-id} --force")
```

**Step 3: Archive feature branch and sub-branches** (only after worktrees removed)
```
Bash("git branch --list 'feature/{run-id}/*' | xargs -r git branch -D")
Bash("git branch -D feature/sp-{run-id}-{slug} 2>/dev/null; git push origin --delete feature/sp-{run-id}-{slug} 2>/dev/null || true")
```

**Step 4: Release pool slot**

Read pool.json, update the claimed slot:
```json
{
  "id": "feature-{N}",
  "status": "available",
  "run_id": null,
  "branch": null,
  "claimed_at": null,
  "last_heartbeat": null
}
```

Write via atomic temp+rename:
```
Bash("cat > .shaman-pipe/worktrees/pool.{pid}.tmp << 'POOL_EOF'\n{updated pool.json}\nPOOL_EOF\nmv .shaman-pipe/worktrees/pool.{pid}.tmp .shaman-pipe/worktrees/pool.json")
```

**Step 4: Archive run state**

Compile the run log:
```json
{
  "run_id": "sp-a3f2c1d9",
  "feature_slug": "stripe-webhook-handler",
  "started_at": "2026-03-13T10:00:00Z",
  "completed_at": "2026-03-13T10:45:00Z",
  "stages_completed": [0, 1, 2, 3, 4, 5, 6],
  "delivery": { "mode": "pr", "pr_url": "https://github.com/...", "branch": "feature/sp-a3f2c1d9-stripe-webhook-handler" },
  "units": ["unit-event-schema", "unit-webhook-handler"],
  "corrections": { "total": 0, "by_type": {} },
  "ambiguity_score": 0.14,
  "review_verdict": "APPROVE"
}
```

```
Write(".shaman-pipe/logs/{run-id}.json", runLogJson)
```

**Step 5: Record timing and print summary**

```
Bash("node ~/.claude/hud/shaman-timing.mjs --end-stage 6 success || true")
Bash("node ~/.claude/hud/shaman-timing.mjs --complete || true")
Bash("node ~/.claude/hud/shaman-summary.mjs --run-id {run-id} || true")
```

**Step 6: Print pipeline completion report**

Read the run log, agent status, and model timing data. Print a structured completion report to the user:

```
═══════════════════════════════════════════════════
  The Shaman Pipe · Complete
═══════════════════════════════════════════════════

  Feature:      {feature_slug}
  Run ID:       {run-id}
  Delivery:     {PR created at {url} | Merged to {target_branch}}

  Agents:       {N} units ({parallelism} parallel)
                {list each unit-id and its contract gate result}

  Corrections:  {total corrections} ({by_type breakdown if any})
  Review:       {APPROVE | CONDITIONAL | REJECT}

  Model usage shown in the timing table above (from shaman-summary.mjs).
  Run archived to .shaman-pipe/logs/{run-id}.json
═══════════════════════════════════════════════════
```

Read `.shaman-pipe/state/{run-id}/agent-status.json` for the agent details. Include each agent's unit-id, status, and whether corrections were needed. This gives the user a complete picture of what happened during the pipeline.

**Step 7: Clear run state**

```
Bash("rm -f .shaman-pipe/state/run-state.json")
Bash("rm -f .shaman-pipe/state/{run-id}/agent-status.json")
```

---

## Adaptive Correction System

The correction system is invoked from Stages 3, 4, and 5 whenever a gate fails. Entry point is always Haiku classification.

### Failure Classifier (Haiku)

Takes raw failure output (test failure, lint error, contract gate failure, review finding) and classifies it:

| Type | Description | Examples |
|---|---|---|
| `SYNTAX` | Lint, formatting, missing imports, type annotations | `TS2304: Cannot find name`, `E302 expected 2 blank lines`, missing import statement |
| `LOGIC` | Incorrect behavior, broken contract, wrong return shape | Test assertion failure, wrong output type, missing side effect |
| `ARCH` | Blast radius violation, wrong abstraction, cross-unit dependency | Modified restricted file, introduced circular dependency, wrong module boundary |
| `AMBIGUOUS` | Failure does not map to the contract (contract may be wrong) | Test expects behavior not in contract, error in generated test itself |

### Correction Chains

**SYNTAX chain:**
```
Step 1: Sonnet self-corrects with failure output injected
  TaskCreate(
    subagent_type="general-purpose",
    model="sonnet",
    context={
      "contract": {original contract},
      "failure_type": "SYNTAX",
      "failure_summary": "{lint/type error message}",
      "previous_attempts": 0,
      "correction_guidance": "Fix the syntax issue. Do not change logic or signatures.",
      "worktree_path": "{worktree_path}",
      "files_to_fix": ["{affected files}"]
    }
  )

Step 2 (if still fails): Haiku re-classifies (may have been misclassified)
  - If reclassified as LOGIC -> enter LOGIC chain
  - If still SYNTAX -> human escalation
```

**LOGIC chain:**
```
Step 1: Sonnet self-corrects with contract + failure
  TaskCreate(
    subagent_type="general-purpose",
    model="sonnet",
    context={
      "contract": {original contract},
      "failure_type": "LOGIC",
      "failure_summary": "{test failure or contract gate violation}",
      "previous_attempts": 0,
      "correction_guidance": "Implementation does not match contract. Fix to match declared signatures, outputs, and side effects.",
      "worktree_path": "{worktree_path}",
      "files_to_fix": ["{affected files}"]
    }
  )

Step 2 (if still fails): Opus diagnoses root cause, rewrites correction prompt
  Opus reads: contract + implementation + failure + Sonnet's first attempt
  Opus produces: specific diagnosis and rewritten correction guidance

Step 3: Sonnet retries with Opus diagnosis
  TaskCreate(
    subagent_type="general-purpose",
    model="sonnet",
    context={
      "contract": {original contract},
      "failure_type": "LOGIC",
      "failure_summary": "{failure details}",
      "previous_attempts": 1,
      "correction_guidance": "{Opus diagnosis: e.g., 'Implementation returns raw DB row, contract expects transformed DTO. Apply mapping in handleWebhook before return.'}",
      "worktree_path": "{worktree_path}",
      "files_to_fix": ["{affected files}"]
    }
  )

Step 4 (if still fails): Human escalation with full correction history
```

**ARCH chain:**
```
Step 1: Opus re-evaluates affected contracts
  Opus reads the failure, the implementation, and the contract
  Opus rewrites contracts for impacted units (narrower blast radius, different abstraction)

Step 2: Sonnet agents for those units re-implement against new contracts
  New sub-branches: feature/{run-id}/unit-{unit-id}-v2

Step 3 (if still fails): RE-PLAN path in Stage 5
```

**AMBIGUOUS chain:**
```
Step 1: Opus reads original contract + failure + implementation
  Opus decides: is the contract wrong, or is the implementation wrong?

  If contract wrong:
    Opus rewrites the contract
    Sonnet retries against new contract

  If implementation wrong:
    Reclassify as LOGIC
    Enter LOGIC chain
```

### Correction Budget

**Per-unit maximums:**

| Type | Per-unit max |
|---|---|
| SYNTAX | 2 |
| LOGIC | 2 |
| ARCH | 1 |
| AMBIGUOUS | 1 |

**Global ceiling:** `3 x number_of_units` total correction attempts across all units and types.

Track budgets in `.shaman-pipe/state/{run-id}/corrections.json`:
```json
{
  "units": {
    "unit-webhook-handler": {
      "SYNTAX":    { "attempts": 1, "max": 2, "history": ["lint error: unexpected token"] },
      "LOGIC":     { "attempts": 0, "max": 2, "history": [] },
      "ARCH":      { "attempts": 0, "max": 1, "history": [] },
      "AMBIGUOUS": { "attempts": 0, "max": 1, "history": [] }
    }
  },
  "global_total": 1,
  "global_ceiling": 6
}
```

### Budget Exhaustion

When a per-unit budget or the global ceiling is exhausted, produce a structured escalation report:

```json
{
  "escalation": true,
  "run_id": "sp-a3f2c1d9",
  "unit_id": "unit-webhook-handler",
  "stage": 4,
  "failure_type": "LOGIC",
  "attempts_made": 2,
  "correction_history": [
    {
      "attempt": 1,
      "failure_summary": "handleWebhook returns raw DB row instead of WebhookResponse",
      "correction": "Sonnet self-correct",
      "result": "Still returns wrong shape"
    },
    {
      "attempt": 2,
      "failure_summary": "handleWebhook returns raw DB row instead of WebhookResponse",
      "correction": "Opus diagnosed: missing DTO mapping layer",
      "result": "DTO mapping added but type mismatch on status field"
    }
  ],
  "branch_preserved": "feature/sp-a3f2c1d9-stripe-webhook-handler",
  "recommendation": "Manual review of src/payments/webhook.ts:42 - the WebhookResponse type definition may need updating to match the actual DB schema"
}
```

Write to run state:
```
Write(".shaman-pipe/state/{run-id}/escalation.json", escalationJson)
```

Pipeline halts. Branch is preserved for manual inspection. Pool slot remains claimed (manual release required after resolution).

> "Correction budget exhausted for unit {unit-id} ({failure_type}). {attempts_made} attempts made. Branch preserved at feature/sp-{run-id}-{slug}. See .shaman-pipe/state/{run-id}/escalation.json for full history."

### Correction Context (injected into every retry)

Every correction attempt receives structured context, not raw error output:

```json
{
  "contract": {
    "unit_id": "unit-webhook-handler",
    "function_signatures": ["handleWebhook(req: WebhookRequest): Promise<WebhookResponse>"],
    "outputs": { "response": "WebhookResponse" }
  },
  "failure_type": "LOGIC",
  "failure_summary": "return shape missing status field - expected WebhookResponse with status, got object without it",
  "previous_attempts": 1,
  "correction_guidance": "Opus diagnosis: implementation returns raw DB row, contract expects transformed DTO. Apply mapping in handleWebhook before return."
}
```

</Steps>

<Pool_Management>

### pool.json Structure

```json
{
  "slots": [
    {
      "id": "feature-1",
      "status": "claimed",
      "run_id": "sp-a3f2c1d9",
      "branch": "feature/sp-a3f2c1d9-stripe-webhook-handler",
      "claimed_at": "2026-03-13T10:00:00Z",
      "last_heartbeat": "2026-03-13T10:42:00Z"
    },
    {
      "id": "feature-2",
      "status": "available",
      "run_id": null,
      "branch": null,
      "claimed_at": null,
      "last_heartbeat": null
    },
    {
      "id": "feature-3",
      "status": "available",
      "run_id": null,
      "branch": null,
      "claimed_at": null,
      "last_heartbeat": null
    }
  ],
  "default_pool_size": 3
}
```

### Atomic Claim Protocol

Race conditions can occur when two `/the-shaman-pipe` invocations run simultaneously. The atomic claim protocol prevents double-claiming:

```
Step 1: Read pool.json
Step 2: Find first available slot
Step 3: Write updated pool (with slot claimed) to temp file:
          .shaman-pipe/worktrees/pool.{pid}.tmp
        where {pid} is the current process ID
Step 4: Atomic rename: mv pool.{pid}.tmp pool.json
Step 5: Re-read pool.json to verify OUR run_id is in the claimed slot
         (if another process won the race, our claim was overwritten)
Step 6: If verification fails, retry from Step 1
```

The rename operation is atomic on POSIX filesystems -- either it fully replaces the file or it does not. This prevents partial writes.

### Heartbeat Protocol

Active runs update `last_heartbeat` every 5 minutes during all stages that hold a slot (Stages 1-6):

```
Read(".shaman-pipe/worktrees/pool.json")
-- Update claimed slot's last_heartbeat to current ISO timestamp
Write via atomic temp+rename
```

**Stale detection:** A slot is stale if:
- `status` is `"claimed"`
- `last_heartbeat` is older than 10 minutes (2x heartbeat interval)

Stale slots are auto-released with a warning when another run needs a slot (Check 7 in Stage 1).

### Queue Behavior

When all slots are claimed and no stale slots exist:

```
Step 1: Log "All pool slots claimed. Queuing..."
Step 2: Wait 5 seconds: Bash("sleep 5")
Step 3: Re-read pool.json
Step 4: Check for stale slots (release if found)
Step 5: Check for available slots (claim if found)
Step 6: If still no slot, repeat from Step 2
```

No hard timeout on queuing. The pipeline waits indefinitely for a slot. The user can create `.shaman-pipe/CANCEL` to exit the queue.

</Pool_Management>

<Checkpoint_Gate>

## Checkpoint Gate (optional — `config.json.checkpoint_mode`)

When `checkpoint_mode` is `true`, the pipeline pauses at each stage boundary. The gate fires **after** the cancel sentinel check and **before** the run-state update for the incoming stage. It presents a summary of the just-completed stage and asks the user whether to proceed.

### Checkpoint Summaries

**Stage 0 → Stage 1 (Design complete)**

Present via `AskUserQuestion`:
```
Design complete.

Feature:             {feature_slug}
Ambiguity:           {ambiguity_score * 100}%
Chosen approach:     {chosen_approach}
Goals:               {goals count}
Acceptance criteria: {criteria count}
Scope:               {scope directories}

Proceed to environment check (Stage 1)?
```
Options: `["Proceed to Stage 1: Environment Check", "Pause — resume later", "Cancel pipeline"]`

---

**Stage 1 → Stage 2 (Environment ready)**

Present via `AskUserQuestion`:
```
Environment ready.

Branch:         feature/sp-{run-id}-{slug}
Worktree slot:  {slot-id}
Checks passed:  7/7
{Any warnings, or "No warnings"}

Proceed to planning (Stage 2)?
```
Options: `["Proceed to Stage 2: Planning", "Pause — resume later", "Cancel pipeline"]`

---

**Stage 2 → Stage 3 (Planning complete)**

Present via `AskUserQuestion`:
```
Planning complete.

Units decomposed:              {unit count}
Contracts authored + approved: {unit count}
Parallelism:                   {parallelism}
Merge order:                   {merge_order list}

Proceed to implementation (Stage 3)?
```
Options: `["Proceed to Stage 3: Implementation", "Pause — resume later", "Cancel pipeline"]`

---

**Stage 3 → Stage 4 (Implementation complete)**

Present via `AskUserQuestion`:
```
Implementation complete.

Units merged:      {unit count}
Corrections used:  {global_total}/{global_ceiling}

Proceed to validation (Stage 4)?
```
Options: `["Proceed to Stage 4: Validation", "Pause — resume later", "Cancel pipeline"]`

---

**Stage 4 → Stage 5 (Validation passed)**

Present via `AskUserQuestion`:
```
Validation passed.

Tests:               {passed}/{total} passing
Type check:          {pass | N errors}
Lint:                {pass | N warnings, N errors}
Acceptance criteria: {N}/{total} mapped to passing tests

Proceed to security and quality review (Stage 5)?
```
Options: `["Proceed to Stage 5: Review", "Pause — resume later", "Cancel pipeline"]`

---

**Stage 5 → Stage 6 (Review verdict)**

Present via `AskUserQuestion`:
```
Review complete.

Verdict:             APPROVE
Security:            clean
Code quality:        clean
Acceptance criteria: all met

Proceed to commit / PR creation (Stage 6)?
```
Options: `["Proceed to Stage 6: Commit", "Pause — resume later", "Cancel pipeline"]`

> Note: Stage 6 checkpoint only fires on `APPROVE`. `FIX_MINOR` and `RE-PLAN` loop back through Stages 3–5 and will re-trigger their own checkpoints; `REJECT` terminates the pipeline directly without a checkpoint.

---

### Checkpoint Responses

**"Proceed to Stage N"** — Continue normally. The run-state update executes immediately after.

**"Pause — resume later"** — Pipeline halts gracefully. `run-state.json` retains the current `stage` value (the stage about to begin), so `/the-shaman-pipe --resume {run-id}` re-enters at the correct point. The worktree slot remains claimed; heartbeat stops. The slot becomes stale after 10 minutes and may be auto-released if another run needs it.

**"Cancel pipeline"** — Enters the same cancellation flow as the `.shaman-pipe/CANCEL` sentinel: release worktree slot, archive branch, clear run-state, write partial run log to `.shaman-pipe/logs/{run-id}.json`.

</Checkpoint_Gate>

<Tool_Usage>
- `Write(".shaman-pipe/state/run-state.json", ...)` / `Read(".shaman-pipe/state/run-state.json")` / `Bash("rm -f .shaman-pipe/state/run-state.json")` -- self-contained run state for resume and cancellation
- `TaskCreate(subagent_type="general-purpose", model="sonnet", ...)` -- spawn parallel Sonnet agents in Stage 3
- `SendMessage(task_id, ...)` -- communicate with spawned agents if needed
- `TaskGet(task_id)` / `TaskList()` -- monitor agent completion in Stage 3
- `Agent(subagent_type="general-purpose", model="haiku", ...)` -- brownfield/greenfield detection and codebase mapping in Stage 0 Phase A
- `AskUserQuestion` -- Stage 0 Phase B approach selection and config questions
- `Bash(...)` -- git commands (worktree, branch, merge, status), package manager, gh CLI, file operations
- `Read(...)` -- read design.json, feature-plan.json, contracts, pool.json, config.json, implemented files
- `Write(...)` -- write design.json, feature-plan.json, contracts, pool.json, config.json, run logs
- `Edit(...)` -- modify config.json, pool.json, resolve merge conflicts
- `Glob(...)` -- map codebase files within scope directories
- `Grep(...)` -- search codebase for existing implementations (duplication scan), verify function signatures (contract gate)
</Tool_Usage>

<Failure_Modes>

| Scenario | Behavior |
|---|---|
| Stage 0 interrupted | Resume from run-state.json if it exists. `Read(".shaman-pipe/state/run-state.json")` returns stage and run_id. Re-read `.shaman-pipe/state/{run-id}/design.json` if Phase A completed. Resume Phase B if design.json exists but no chosen_approach. |
| Stage 3 agent crash | Heartbeat expires after 10 minutes with no update. Next `/the-shaman-pipe` invocation or stale check releases the slot with a warning. Branch is archived, not deleted. Run state preserved for debugging. |
| Stage 6 merge fail (direct mode) | Target branch left in pre-merge state. Run logs record the failure. Manual resolution required -- no auto-rollback of a partial merge. Pool slot released. |
| Budget exhausted | Structured escalation report written to `.shaman-pipe/state/{run-id}/escalation.json`. Branch preserved for manual inspection. Pool slot remains claimed. Pipeline halts with human-readable summary. |
| Pool full (all slots claimed) | Check stale slots first (heartbeat > 10 min). If stale found, release and claim. If no stale, queue: wait 5 seconds, retry. Repeat until slot available. User can `/cancel` to exit queue. |
| Cancel during any stage (`.shaman-pipe/CANCEL` sentinel) | Release worktree slot (set status to available). Archive feature branch. Clear run state (`rm -f .shaman-pipe/state/run-state.json`). Write partial run log to `.shaman-pipe/logs/{run-id}.json` with `stages_completed` reflecting progress. Remove sentinel (`rm -f .shaman-pipe/CANCEL`). |
| `gh` CLI not authenticated | Stage 1 branch protection check falls back gracefully (warns, uses config.json value). Stage 6 PR creation fails with clear error pointing to `gh auth login`. |
| Critic pass fails 3 times | Pipeline halts at Stage 2. Contracts are fundamentally flawed. Human review of design.json and contracts required. |
| RE-PLAN issued twice for same units | Second RE-PLAN exhausts the ARCH correction budget for those units. Escalation report produced. |
| Worktree creation fails | Likely disk space or git state issue. Clear error with `git worktree list` output for debugging. Pool slot released. |

</Failure_Modes>

<Examples>

<Good>
Greenfield invocation:
```
User: /the-shaman-pipe "add REST API for user preferences with CRUD operations"

Stage 0: Phase A clarity loop runs 6 rounds, ambiguity drops to 16%
         Opus proposes: (1) Express + Prisma, (2) Fastify + Drizzle, (3) Hono + raw SQL
         User selects Approach 1
         design.json written with 3 goals, 2 constraints, 4 acceptance criteria

Stage 1: All 7 checks pass, pool slot feature-1 claimed, worktree created

Stage 2: Opus creates 3 units: unit-schema, unit-handlers, unit-routes
         Critic pass validates coverage of all 4 acceptance criteria
         feature-plan.json written with merge_order respecting dependencies

Stage 3: 2 Sonnet agents run in parallel (unit-schema + unit-handlers)
         unit-routes waits for unit-handlers (dependency)
         All contract gates pass, merge protocol completes cleanly

Stage 4: npm test passes, tsc --noEmit clean, eslint clean
         All acceptance criteria mapped to test results

Stage 5: Opus APPROVE across all three lenses

Stage 6: PR created, slot released, state archived
```
Why good: Full pipeline with no corrections needed. Clean pass through all gates.
</Good>

<Good>
Correction chain activation:
```
Stage 4 validation fails: test_preferences.ts::test_update assertion error
  Haiku classifies: LOGIC (return shape wrong)
  Budget check: LOGIC 0/2 for unit-handlers, global 0/6

  Correction attempt 1: Sonnet self-corrects
    Still fails: return shape still wrong

  Correction attempt 2: Opus diagnoses
    "Implementation updates DB but returns stale object. Must re-query after UPDATE."
    Sonnet retries with Opus diagnosis
    Test passes

  Stage 4 re-run: all pass
  Stage 5: Opus APPROVE
```
Why good: Correction chain used minimum intervention. Budget tracked correctly. Opus diagnosis was specific and actionable.
</Good>

<Good>
Concurrent features:
```
Session A: /the-shaman-pipe "add user preferences API"
  Claims feature-1 slot
  Running Stage 3...

Session B: /the-shaman-pipe "add notification service"
  Check 5 (conflict scan): no scope overlap with Session A
  Claims feature-2 slot
  Running Stage 2...

Both complete independently. No worktree collision.
```
Why good: Pool isolation works. Scope overlap checked but no conflict.
</Good>

<Bad>
Proceeding with high ambiguity:
```
Stage 0: Phase A clarity loop runs 3 rounds, ambiguity at 45%
Pipeline proceeds to Stage 1 anyway
```
Why bad: Hard gate at 20% ambiguity was bypassed. This will cause contract failures downstream because the goals and acceptance criteria are unclear.
</Bad>

<Bad>
Ignoring contract gate:
```
Stage 3: Agent modifies src/billing/schema.ts despite restricted_from
Pipeline proceeds to Stage 4 without catching the violation
```
Why bad: Contract gate must catch restricted file violations. This is a blast radius breach that will cascade through later stages.
</Bad>

<Bad>
Unlimited correction retries:
```
Stage 4: LOGIC failure on unit-handlers
  Sonnet retries 5 times with no Opus diagnosis
  Still failing after 5 attempts
```
Why bad: LOGIC budget is 2 per unit. After attempt 1, Opus should diagnose. After attempt 2, budget is exhausted and escalation report is produced. Never retry blindly.
</Bad>

</Examples>

<Escalation_And_Stop_Conditions>
- **Ambiguity > 20% after Stage 0 Phase A:** Do not proceed. Continue the clarity loop or ask user for more detail.
- **Stage 1 environment check fails (error, not warning):** Stop immediately with clear error message.
- **Critic pass fails 3 times:** Stop at Stage 2. Contracts are fundamentally flawed.
- **Per-unit correction budget exhausted:** Produce escalation report. Halt pipeline for that unit.
- **Global correction ceiling reached (3 x N units):** Produce escalation report. Halt entire pipeline.
- **Stage 5 REJECT verdict:** Hard stop. Archive branch. Write rejection rationale.
- **User says "stop", "cancel", "abort" or creates `.shaman-pipe/CANCEL`:** Release slot, archive branch, clear state.
- **Stage 6 merge conflict in direct mode:** Log failure, leave target branch clean, require manual resolution.
</Escalation_And_Stop_Conditions>

<Final_Checklist>
- [ ] Run ID generated and state directory created
- [ ] Run state written to .shaman-pipe/state/run-state.json at Stage 0 start
- [ ] Namespace `.shaman-pipe/` bootstrapped (idempotent)
- [ ] `.gitignore` updated with `.shaman-pipe/worktrees/` and `.shaman-pipe/state/`
- [ ] Stage 0: Phase A clarity loop completed with ambiguity <= 20%
- [ ] Stage 0: design.json written with all required fields
- [ ] Stage 0: User selected architectural approach via AskUserQuestion
- [ ] Stage 1: All 7 environment checks passed
- [ ] Stage 1: Pool slot claimed atomically
- [ ] Stage 1: Git worktree created in claimed slot
- [ ] Stage 2: feature-plan.json written with valid dependency graph
- [ ] Stage 2: All contracts written and passed Critic review
- [ ] Stage 2: Restricted files enforced in all contracts
- [ ] Stage 3: Sonnet agents spawned per parallelism setting
- [ ] Stage 3: All agents passed inline contract gate
- [ ] Stage 3: Merge protocol completed in dependency order
- [ ] Stage 3: Heartbeat updated during execution
- [ ] Stage 4: Dependencies installed in worktree
- [ ] Stage 4: Tests, type check, and lint all pass
- [ ] Stage 4: Each acceptance criterion mapped to evidence
- [ ] Stage 5: Opus APPROVE verdict across all three lenses
- [ ] Stage 6: PR created or direct merge completed
- [ ] Stage 6: Pool slot released
- [ ] Stage 6: Run state archived to `.shaman-pipe/logs/{run-id}.json`
- [ ] Run state cleared (rm .shaman-pipe/state/run-state.json) at Stage 6 completion
- [ ] No worktree collision with other active runs
- [ ] Correction budgets respected (per-unit and global ceiling)
</Final_Checklist>

<Advanced>

## Configuration

`.shaman-pipe/config.json` -- set during first Stage 0 run, persisted per project:

```json
{
  "delivery_mode": "pr",
  "target_branch": "main",
  "pool_size": 3,
  "auto_merge_on_ci_pass": false,
  "test_command": null,
  "package_manager": null
}
```

`test_command` and `package_manager` are auto-detected if null. Set explicitly to override auto-detection (e.g., `"test_command": "npm run test:integration"` for non-standard test scripts).

## Resume

If interrupted at any stage, re-invoke `/the-shaman-pipe` (no arguments needed). The skill reads `.shaman-pipe/state/run-state.json` and resumes from the last completed stage:

| Interrupted At | Resume Behavior |
|---|---|
| Stage 0 Phase A | `interview-state.json` exists in `.shaman-pipe/state/{run-id}/`. Re-invoke `/the-shaman-pipe` to continue from the last completed round. |
| Stage 0 Phase B | design.json exists but no chosen_approach. Re-present approach selection. |
| Stage 1 | Re-run all 7 checks (idempotent). |
| Stage 2 | If feature-plan.json exists, check for contracts. Resume Critic pass if contracts exist. |
| Stage 3 | Check which units completed (sub-branches exist with commits). Re-spawn only incomplete units. |
| Stage 4 | Re-run full validation suite (idempotent). |
| Stage 5 | Re-run full review (idempotent). |
| Stage 6 | Check if PR exists or merge completed. Resume cleanup if delivery done. |

## Model Routing Summary

| Stage | Model | Rationale |
|---|---|---|
| 0 - Design Intelligence | Opus | Architecture decisions, approach design, ambiguity evaluation |
| 1 - Environment Check | Haiku | Mechanical checks, no creative reasoning needed |
| 2 - Planning | Opus | Contract design, dependency analysis, blast radius decisions |
| 3 - Implementation | Sonnet | Code generation per contract, parallel execution |
| 3 - Merge Protocol | Sonnet | Conflict resolution using contracts as ground truth |
| 4 - Validation | Haiku | Run commands, collect results, map criteria |
| 4 - Test Generation | Sonnet | Generate baseline tests when none exist |
| 5 - Review | Opus | Security, quality, and acceptance review with authorship context |
| 6 - Commit | Haiku | PR creation, merge, cleanup -- mechanical operations |
| Correction: Classify | Haiku | Failure classification |
| Correction: Self-correct | Sonnet | Code fixes per guidance |
| Correction: Diagnose | Opus | Root cause analysis for LOGIC and ARCH failures |

## State Management

The Shaman Pipe manages its own state under `.shaman-pipe/state/run-state.json`. No external dependencies.

**Schema:**
```json
{
  "active": true,
  "stage": 0,
  "run_id": "sp-a3f2c1d9",
  "phase": "A",
  "feature_slug": "stripe-webhook-handler",
  "started_at": "2026-03-13T10:00:00Z"
}
```

**Lifecycle:**
- Written at Stage 0 start
- Updated on every stage transition (stage number incremented)
- Cleared (file deleted) at Stage 6 completion or on cancellation

**Cancellation sentinel:** Create `.shaman-pipe/CANCEL` to request cancellation. The pipeline checks for this file at the start of each stage and enters the cancellation flow if found.

**Resume:** On `/the-shaman-pipe` invocation with no arguments, if `run-state.json` exists, read it to determine `run_id` and `stage`, then resume from the appropriate stage.

**Scope clarification:** `.shaman-pipe/state/run-state.json` stores the active/phase signal needed for resume and cancellation.

## Ticket-Driven Mode

`/the-shaman-pipe --ticket GH-142` fetches the issue and synthesizes `design.json` automatically:

```
Bash("gh issue view 142 --json title,body,labels,assignees")
```

Map ticket fields to design.json:
- `title` -> primary goal
- `body` -> goals, constraints, acceptance criteria (parsed from markdown checkboxes)
- `labels` -> scope hints (e.g., label "payments" -> scope ["src/payments/"])

Set `"source": "github"` and `"ticket_id": "GH-142"`.

Skip Phase A (Socratic loop) and Phase B (approach selection) entirely. Opus still runs the Challenge-the-Brief gate in Stage 2 to validate the ticket's requirements are sufficient.

If the ticket lacks acceptance criteria, Opus generates them and presents to user for confirmation before proceeding.

## Worktree Lifecycle

```
Creation:   git worktree add .shaman-pipe/worktrees/{slot-id} -b feature/sp-{run-id}-{slug}
Usage:      All Stage 3 agents work inside this directory
Cleanup:    git worktree remove .shaman-pipe/worktrees/{slot-id} --force
```

The worktree is a full git working directory with its own HEAD, index, and working tree. Changes in the worktree do not affect the main working directory. This enables concurrent feature development without session collision.

## Correction Budget Tracking

Track correction attempts per-unit and globally in run state at `.shaman-pipe/state/{run-id}/corrections.json`:

```json
{
  "units": {
    "unit-webhook-handler": {
      "SYNTAX": { "attempts": 0, "max": 2, "history": [] },
      "LOGIC": { "attempts": 1, "max": 2, "history": [
        {
          "attempt": 1,
          "stage": 4,
          "failure_summary": "handleWebhook returns raw DB row",
          "correction": "Sonnet self-correct",
          "result": "Still returns wrong shape",
          "timestamp": "2026-03-13T10:30:00Z"
        }
      ]},
      "ARCH": { "attempts": 0, "max": 1, "history": [] },
      "AMBIGUOUS": { "attempts": 0, "max": 1, "history": [] }
    }
  },
  "global_total": 1,
  "global_ceiling": 6
}
```

Before each correction attempt:
1. Check per-unit budget for the failure type
2. Check global ceiling
3. If either exhausted, produce escalation report instead of retrying

## Pipeline Cancellation

Create `.shaman-pipe/CANCEL` to cancel the active run (e.g. `touch .shaman-pipe/CANCEL`). The pipeline checks for this sentinel at the start of each stage (1 through 6). When detected:

```
Step 1: Read .shaman-pipe/state/run-state.json to get run_id and current stage
Step 2: Remove agent sub-worktrees for current stage (if Stage 3):
          git worktree list --porcelain | grep '.shaman-pipe/worktrees/{slot-id}/agents' | awk '/worktree/{print $2}' | xargs -r -I{} git worktree remove {} --force
Step 3: Remove feature worktree:
          git worktree remove .shaman-pipe/worktrees/{slot-id} --force
Step 4: Delete sub-branches and feature branch:
          git branch --list 'feature/{run-id}/*' | xargs -r git branch -D
          git branch -D feature/sp-{run-id}-{slug} 2>/dev/null || true
Step 5: Release pool slot (set status back to "available" in pool.json)
Step 6: Write partial run log to .shaman-pipe/logs/{run-id}.json
Step 6b: Record timing and print summary:
          Bash("node ~/.claude/hud/shaman-timing.mjs --cancel || true")
          Bash("node ~/.claude/hud/shaman-summary.mjs --run-id {run-id} || true")
Step 7: Clear run state:
          Bash("rm -f .shaman-pipe/state/run-state.json")
Step 8: Remove sentinel:
          Bash("rm -f .shaman-pipe/CANCEL")
```

</Advanced>

Task: {{ARGUMENTS}}
