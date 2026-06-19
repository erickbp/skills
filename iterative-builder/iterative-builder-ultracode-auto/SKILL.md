---
name: iterative-builder-ultracode-auto
description: Autonomous iterative build workflow — plan one task at a time against actual codebase state, headless and loop-safe. Each task card is designed by code-architect reading the real codebase, implemented in a worktree, reviewed, and merged before the next task is planned, with no approval prompts. Runs one task per invocation and signals status (CONTINUE/COMPLETE/BLOCKED) for an external loop; halts for a human only for credentials, external/cloud checks, risky/destructive actions, or a task it cannot complete. Use when upfront planning would diverge from reality or when the sequence of work should emerge organically.
---

# Iterative Builder

Plan one task at a time against the actual codebase, build it, review it, commit it, then plan the next.

## Objective

- Produce task cards that are designed against the real codebase state, not a projected future state.
- Let the sequence of work emerge organically from completed work and remaining requirements.
- Run unattended — write the manifest and every task card to disk for state, resume, and audit, but proceed without approval prompts; halt for a human only when something genuinely requires one (credentials, an external/cloud check, a risky/destructive action, or a task that cannot be completed).
- Deliver fully reviewed, tested, committed code after each task before moving on.
- Fan out the read- and verify-heavy steps across parallel sub-agents to widen coverage — foreground by default, escalating to true background orchestration (the Workflow tool) on the heavy steps — while keeping the autonomous flow and the authoritative reviewer intact.

## Autonomous Operation

This skill runs unattended — typically headless inside a loop/bash wrapper. It never asks the user questions and never waits for approval. It writes the manifest and task cards to disk (for state, resume, and audit) and proceeds on its own through bootstrap, every task, and validation.

**One unit of work per invocation, then signal and stop.** Each invocation does exactly one of: (a) bootstrap + the first task, (b) one task, or (c) Phase 3 validation when no pending requirements remain. It then writes a status sentinel and stops. The external loop re-invokes for the next unit. Because each invocation is a fresh process, every task is designed against a fresh read of the repo (this is the Rule 14 benefit, achieved without a human).

**Status signal (read by the loop).** At the end of every invocation, write `TASKS/.ib-status` whose first line is one of `CONTINUE`, `COMPLETE`, or `BLOCKED` (optionally a one-line reason after it), and print a matching marker block:

- `CONTINUE` — a task merged (or bootstrap finished) and work remains, including pending validation. The loop re-invokes `/iterative-builder-ultracode-auto @TASKS/MANIFEST.md`.
- `COMPLETE` — all requirements are `done`/`deferred` and Phase 3 validation has finished. The loop stops (success).
- `BLOCKED` — a halt condition was hit (a "HUMAN INPUT REQUIRED" situation). The loop stops and alerts a human. Also write `TASKS/.ib-block.md` with the category, what is needed, the current state, and how to resume; print the same as the marker (the BLOCKED marker reads `HUMAN INPUT REQUIRED — <category>`).

Marker format:

```
=== ITERATIVE-BUILDER STATUS: <CONTINUE|COMPLETE|BLOCKED> ===
<one-line summary — e.g. "TASK-03 merged to main; 2 requirement(s) remain">
<CONTINUE: re-invoke with /iterative-builder-ultracode-auto @TASKS/MANIFEST.md>
<BLOCKED: category + what a human must do; see TASKS/.ib-block.md>
===
```

**Halt for a human ONLY when one of these is true** (write `BLOCKED`, preserve all state, do not merge or take the risky action, then stop):

1. **`credentials`** — a credential, secret, token, or login the run cannot supply is required to proceed.
2. **`external-check`** — something the code cannot verify must be confirmed against an external/cloud system (a deployed service, staging, a cloud resource, a third-party dashboard).
3. **`risk`** — a destructive or irreversible action needs human judgment (deleting data, dropping or destructively migrating a table, force-pushing, rewriting history, touching production — anything not safely reversible).
4. **`unfixable-task`** — the task cannot be completed cleanly on its own: the reviewer still FAILs after 5 fix cycles, the implementation sub-agent cannot succeed after bounded redesign, or build/tests fail on merge after fix attempts.

**Everything else proceeds without asking.** In particular, a genuine product-judgment gap (an ambiguous requirement, an unstated preference, a missing Open Question answer) is NOT a halt condition: pick the most reasonable interpretation, record it as an explicit assumption in the Decision Ledger (and the Adjustments Log when it shapes the build), and continue. Quality gates are not user questions and stay fully intact — the `feature-dev:code-reviewer` gate and its fix→re-review loop (Rule 15), the Step 2.3 card-quality critics, the Codex advisory second opinion (Rule 16), and the ultracode fan-out all still run.

## Multi-Agent Orchestration

Seven steps fan out across parallel sub-agents rather than running as a single pass: **1.1** (ground the repo), **1.2** (decision ledger), **1.3** (extract requirements), **2.2** (design the card), **2.3** (quality-check the card), **2.7** (review), and **3.1** (check success criteria). Each carries an **Ultracode fan-out** block with its recipe. Apply the fan-out automatically; do not ask the user whether to use it.

Two distinct mechanisms do this work — do not conflate them:

- **Default — foreground sub-agents (no opt-in needed).** Launch the step's agents as inline `Agent` calls in a single message, in the foreground (the pattern feature-dev and `/code-review` use), so results land in the same turn and flow into the next gate. This is always available, needs no opt-in, and is the default for all seven steps.
- **Heavy steps — the Workflow tool (true background "ultracode").** On the three heavy steps, when a threshold below is met, **call the Workflow tool** to fan out in the background instead. It runs asynchronously — the call returns a task id and the result arrives later via a notification, after which you aggregate and proceed to the same gate. Invoking this skill is itself the opt-in to the Workflow tool for these steps, so use it whenever a threshold is met; do not ask first. Below the thresholds, stay foreground.
- **Scale to the work (Rule 11):** match agent count to risk; collapse to one agent or skip fan-out for tiny, low-risk work.

| Step | Call the Workflow tool when… |
|------|------------------------------|
| **2.7** Review | the task's diff spans more than ~10 files or ~400 changed lines, or the reviewers are slow to run inline |
| **1.1** Ground the repo | the repo exceeds ~1,000 tracked source files, or is a monorepo with 3+ packages/workspaces |
| **3.1** Success criteria | there are more than ~8 success criteria, or per-SC verification needs slow builds/test suites |

The background fan-out must complete and be aggregated **within the same task** — before the step that follows, and never across the Step 10 hard stop. Results return to the same gate with the same conservative aggregation rule. The other four steps (1.2, 1.3, 2.2, 2.3) are lightweight and stay foreground by default — escalate them only in the rare exceptions their per-step recipes note (1.2 never escalates).

This augments the workflow; it never weakens it. Fan-out **feeds a step, never replaces a gate** — the former user-approval steps (1.5 manifest, 2.4 card, 3.2 gaps) now run autonomously (the orchestrator writes/decides and proceeds; see Autonomous Operation), and `feature-dev:code-reviewer` stays the single authoritative review gate: parallel reviewers only add breadth, aggregated conservatively (PASS only if every dimension passes; any FAIL is a FAIL with the union of findings), and no finding is ever dismissed, downgraded, or dropped (Rule 15). Codex stays advisory (Rule 16) and the per-task context reset stays (Rule 14). Fan-out parallelizes work **within a single step only** — it never parallelizes the per-task loop across tasks, and never creates execution waves; tasks are still designed, built, reviewed, and merged one at a time.

Read [references/ultracode-fanout.md](references/ultracode-fanout.md) for the per-step recipes, the full mechanism guidance, and the complete invariant list.

## Workflow

### Phase 1: Bootstrap

Establish the goal, capture decisions, extract requirements, and create the living manifest.

#### Step 1: Ground the goal and the repo

Before designing any task:

1. Read the input carefully and identify:
   - product goal
   - scope boundaries
   - likely success criteria
   - likely risk classes
2. Explore the repo with tools to verify:
   - Repo topology: monolith, service, monorepo, package, app
   - Affected subsystems: UI, API, domain logic, persistence, jobs, infra
   - Existing patterns: read 1-2 representative examples of similar features or modules to understand conventions
   - Data layer: ORM, migration framework, schema definitions, database type
   - API layer: framework, routing conventions, middleware stack
   - Test infrastructure: test framework, config, directory layout, naming conventions
   - Build/CI: build system, CI pipeline, deploy process
   - Dependencies: package manager, key libraries, version constraints
3. Do not invent repo facts that tools can verify. You must use tools to verify these facts.
4. If the repo cannot answer something and the answer is product intent, do not ask — make the most reasonable assumption, record it as an explicit assumption in the Decision Ledger (Open Questions / Assumptions, resolved-by-assumption) and proceed. Only halt (`BLOCKED`) if proceeding would require credentials, an external/cloud check, or a risky/destructive action (see Autonomous Operation).
5. For greenfield projects where no code exists yet, paths from the input document are treated as verified. Internal paths that follow established framework conventions should be labeled `(convention-based)` in Discovered Facts.

**Ultracode fan-out (multi-agent):** Run the repo sweep (item 2) as parallel read-only sub-agents launched in ONE message, each covering one angle of the Discovered Facts: (a) topology + build/CI + dependencies, (b) data/persistence layer, (c) API/service layer + routing/middleware, (d) test infrastructure + conventions, (e) 1-2 existing features closest to the goal. Each agent returns ONLY tool-cited facts (Glob/Grep/Read evidence) plus key file paths — no invented facts (Rule 1), no product-intent guesses. Orchestrator reads the surfaced key files, then synthesizes one Discovered Facts set; mark unverifiable items `unverified` and greenfield framework paths `(convention-based)`. If any angle reveals a product-intent gap, record it as an explicit assumption and proceed (item 4) — fan-out gathers facts; product-intent gaps become logged assumptions, never questions to the user, and never silent scope changes. Scale down for tiny/greenfield repos (fewer agents). When the repo exceeds ~1,000 tracked source files or is a monorepo with 3+ packages/workspaces, run this sweep via the Workflow tool in the background (true ultracode) — wait for it to complete, then synthesize one Discovered Facts set as below. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 2: Build a Decision Ledger

Create a `Decision Ledger` with four buckets:

- `Locked Decisions` — binding user choices and hard constraints
- `Coding Agent's Discretion` — real implementation freedom areas
- `Deferred / Out of Scope` — explicitly excluded work
- `Open Questions` — missing user judgment that materially changes the plan

Rules:

- Record `Open Questions` only for user-judgment gaps, not repo facts.
- Treat `Locked Decisions` as binding.
- Treat `Deferred / Out of Scope` as forbidden scope.
- Use `Coding Agent's Discretion` only for real implementation freedom.
- Do not silently guess: resolve each Open Question with an explicit, logged assumption (autonomous mode never asks).

**Scale the ledger to the task size.** For XS/S work, keep it minimal: keep all four buckets but make their content terse — a couple of short bullets where there's real content and `None` where a bucket is genuinely empty. Use at most a couple of terse bullets each, no multi-clause analysis. A self-resolvable implementation detail is not an `Open Question` (decide it in Discretion or leave it to implementation); record an `Open Question` only when a genuine user-judgment decision is at stake (resolved with a logged assumption), else record a one-line assumption. A simple change deserves a terse ledger (mostly `None`), not a multi-bullet survey — but keep all four buckets present.

Read [references/decision-ledger.md](references/decision-ledger.md) when building or validating the ledger.

**Ultracode fan-out (multi-agent):** First draft all four buckets in a single pass yourself. Then launch 2-3 completeness critics in parallel (one message, foreground), each independently re-reading the goal + Discovered Facts to flag, with tool evidence, missed `Open Questions`, mis-bucketed Locked-vs-Discretion items, and silently-assumed decisions that belong in `Locked` or `Open`. Merge and dedup every flag into the ledger — additively; a critic may add or re-bucket items but never delete an Open Question or downgrade a Locked decision. Critics surface user-judgment gaps only (never ask about repo facts) and never silently resolve a gap — every surviving gap becomes an explicit, logged assumption (never left for a user). For XS/S goals, SKIP this critic panel entirely (it manufactures ceremony on small tasks); use it only when the goal is large or risky enough to warrant it. The ledger still feeds the autonomous Step 1.5 manifest write. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 3: Extract requirements and success criteria

From the input and grounding, extract:

- **Numbered requirements**: `REQ-01`, `REQ-02`, ... — each a distinct deliverable or behavior the goal demands.
- **Success criteria**: `SC-01`, `SC-02`, ... — observable conditions that prove the goal is met.

Requirements should be atomic enough that each maps to roughly one task, but coarse enough to avoid ceremony. If a requirement clearly needs multiple tasks, that will emerge naturally during the per-task loop.

**Match the requirement count to the size of the work.** For a small, single-purpose input, prefer the fewest requirements that capture the deliverable — often just one or two — and keep success criteria to the few that actually prove the goal. Collapse trivially-coupled asks (work that one small task delivers together, e.g. a toggle button plus the state it flips) into a single requirement; do not pre-split a cohesive small change into separate REQs. Reserve fine-grained decomposition for genuinely large or separable work, where many requirements are correct (a multi-feature platform legitimately yields 8+).

**Ultracode fan-out (multi-agent):** Launch parallel extractors in ONE message (foreground), each deriving candidate `REQ-`/`SC-` items from the input + Step 1 grounding through one lens — (a) explicit asks, (b) implied behaviors, (c) cross-cutting/non-functional (security, performance, observability, migration/rollout, compliance). Each item cites tool evidence (Rule 1); no item is invented. The orchestrator merges, dedups, and reconciles numbering, then runs one completeness critic asking "what deliverable is implied but unlisted?" Keep `REQ`s atomic (~one task) yet coarse enough to avoid ceremony, and SCs observable (Rule 9). The critic only ADDS candidates — it is not a gate and drops nothing. Scale lenses to the input (Rule 11): for an XS/single-ask input, SKIP the extractor fan-out and the completeness critic entirely — just list the one or two obvious requirements inline; the multi-lens sweep is for rich, multi-feature specs where it prevents missed deliverables. The reconciled set feeds Step 4 and the autonomous Step 1.5 manifest write. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 4: Write the manifest

Write `TASKS/MANIFEST.md` using the manifest template. At this stage the manifest contains:

- Goal summary
- Discovered Facts
- Decision Ledger
- Success Criteria
- Requirements with status (all `pending`)
- Empty sections for: Completed Tasks, Adjustments Log, Remaining Work, Follow-up Items

For a small task, keep the manifest light in content, not structure: keep all sections present as the living-document scaffold, but keep their content terse — Discovered Facts is a few verified lines (not an exhaustive survey); empty sections (Completed Tasks, Adjustments Log, Follow-up Items) stay present, shown as "None" and ready for later updates; and Remaining Work can be a one-liner instead of restating each requirement. Do not pad sections with invented content, but do not drop them either.

Read [references/manifest-template.md](references/manifest-template.md) for the manifest format.

#### Step 5: Finalize manifest

Write `TASKS/MANIFEST.md` (Step 4) and proceed directly to Phase 2 — do not pause for approval (autonomous mode). The manifest is the on-disk source of truth and audit trail; it is not gated.

A human may steer the build between invocations by editing `MANIFEST.md` directly (add/remove/modify requirements, change locked decisions, resolve or override a logged assumption). On the next invocation the skill reconciles those edits (see Edge Cases → "Manifest edited between invocations"). Within an invocation, nothing waits on the user.

### Phase 2: Per-Task Loop

Repeat until all requirements are addressed.

#### Step 1: Select next requirement(s)

Choose the next pending requirement(s) from the manifest. Selection criteria:

- Prerequisites satisfied (prior tasks have landed on main)
- Logical ordering (foundational before dependent)
- User priority if expressed

A single task may address one or more related requirements. Mark selected requirements as `in-progress` in the manifest.

#### Step 2: Design the task card

Invoke `feature-dev:code-architect` with:

- The selected requirement(s) and their context
- The decision ledger
- The task card template ([references/task-card-template.md](references/task-card-template.md))
- Summaries of completed tasks (what was built, what files exist now)
- Coverage and must-haves guidance ([references/coverage-and-must-haves.md](references/coverage-and-must-haves.md))
- Any relevant anti-stub patterns ([references/anti-stub-patterns.md](references/anti-stub-patterns.md))

The code-architect reads the **current codebase** (including all previously merged work) and produces a task card. This is the key advantage over upfront planning — the card is designed against reality.

**Ultracode fan-out (multi-agent):** Replace the single architect with a judge panel — in ONE message, launch parallel foreground `feature-dev:code-architect` agents (multiple instances of the same agent), each with a different lens (minimal-change / clean-architecture / pragmatic-balance), each reading the **current** codebase (Rule 13) and citing tool evidence (Rule 1), each returning a complete candidate card. Score every candidate on the Step 2.3 dimensions — requirement coverage, anti-stub substance, XS/S/M sizing fit, locked-decision adherence — then synthesize ONE winning card, grafting the best elements of the runners-up. This designs a single card from several perspectives; it does not design multiple tasks at once (no execution waves). Scale the panel to the work (Rule 11): 1 architect for XS/low-risk, 2-3 for S/M or higher risk. The synthesized card is not final: it still flows into the 2.3 quality-check and the autonomous 2.4 finalize. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 3: Quality-check the card

Before finalizing the card, verify:

- No scope-reducing language in Goal or In Scope ("initial structure", "stub", "placeholder", "skeleton", "shell", "basic scaffold") for source-required behavior
- Substance constraints present in Artifacts for high-risk files
- Verification commands are concrete and executable
- Must-Haves include Truths (observable behavior), Artifacts (concrete deliverables), and Key Links (critical wiring)
- Truths describe behavior or invariants, not implementation steps
- Locked decisions from the ledger are honored
- The card does not implement deferred/out-of-scope items
- The card fits sizing guidelines (XS/S/M — not L or XL)

If the card fails quality checks, re-invoke code-architect with specific feedback.

**Ultracode fan-out (multi-agent):** Instead of one solo pass, launch the checklist as parallel adversarial critics in a SINGLE message (foreground), each attacking ONE dimension of the card and citing the exact card text/repo evidence (Rule 1): (a) scope-reducing language in Goal/In Scope; (b) stub risk vs. substance constraints in Artifacts for high-risk files; (c) Must-Haves shape — Truths are behavior not steps, and Truths/Artifacts/Key Links are all present; (d) Verification Commands concrete and executable; (e) locked-decision adherence + no deferred/out-of-scope leakage; (f) sizing (XS/S/M, not L/XL). Each returns PASS/FAIL + specific issues. Aggregate conservatively: PASS only if EVERY critic returns PASS; ANY FAIL = aggregate FAIL — re-invoke `feature-dev:code-architect` (loop back to Step 2) with the union of all issues, then re-run the critics. This is a pre-implementation planning-quality gate: it FEEDS Step 4 (finalize) and decides whether the card proceeds to implementation; it is NOT the `feature-dev:code-reviewer` code-review gate (Rule 15). Scale down for tiny/low-risk cards — fold critics into one pass; never add ceremony (Rule 11). See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 4: Finalize task card

Write the card to `TASKS/TASK-NN.md` and proceed to Step 5 (create worktree) — do not pause for approval (autonomous mode). Card quality is enforced autonomously by the Step 3 quality-check critics, which loop back to the architect on any FAIL; only a card that passes that gate reaches this step.

Revision feedback is honored only when a human has left it in the manifest between invocations (see Edge Cases → "Manifest edited between invocations"); the skill never stops to solicit it.

#### Step 5: Create worktree

```bash
git checkout main
git pull
git branch task/TASK-NN
git worktree add .worktrees/TASK-NN task/TASK-NN
```

#### Step 6: Implement

Launch an implementation sub-agent in the worktree with the finalized task card as its operating instructions. The sub-agent follows the Execution Protocol and Guardrails embedded in the task card.

#### Step 7: Review

**NEVER self-dismiss reviewer findings.** You are not qualified to judge whether a reviewer finding is valid — only the reviewer (via re-review after a fix) — or, after the max-5 halt, a human — can make that determination. Any orchestrator analysis of reviewer findings that concludes "this is fine", "false positive", "non-blocking", "backward compatible", or similar is a skill violation regardless of how reasonable the analysis seems. See the violation example below.

**Invoking the reviewer:** When you invoke `feature-dev:code-reviewer`, include this instruction in your prompt to the reviewer:

> End your review with a verdict line in exactly this format:
> `## Verdict: PASS` if no issues found, or `## Verdict: FAIL — N issue(s)` if issues found.

After the reviewer responds, read **only the verdict line** to decide the next action. Do not interpret the substance of individual findings to decide whether they "really" matter.

**Ultracode fan-out (multi-agent):** Inside the max-5 loop, replace the single `feature-dev:code-reviewer` call with parallel `feature-dev:code-reviewer` agents launched in ONE message, each scoped to one dimension — correctness/bugs, security, project-conventions (CLAUDE.md), simplicity/DRY, tests/coverage — and each ending with its own `## Verdict` line. Read only the verdict lines, then aggregate conservatively: aggregate PASS only if EVERY dimension is PASS; ANY FAIL = aggregate FAIL whose issue list is the UNION of all agents' findings. Scale agents to risk (Rule 11): a tiny/low-risk diff may use one reviewer. When the diff spans more than ~10 files or ~400 changed lines (or the reviewers are slow to run inline), fan the dimension reviewers out in the background via the Workflow tool (true ultracode) — wait for completion, then aggregate the verdicts at this same gate; the fix/re-review loop must still converge before Step 8 and never cross the Step 10 hard stop. Never self-dismiss, downgrade, or drop a finding — including dropping one dimension's finding because another agent disagrees (Rule 15); the fan-out adds breadth only and feeds, never replaces, the gate. The fix-then-re-review loop, max-5 cap, Codex advisory step (Rule 16), Review Gate Checklist, and the exit conditions (PASS, or halt-for-human after max-5) are UNCHANGED; only extend the checklist to list which dimensions were covered. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

**Loop:**

```
review_count = 0

while review_count < 5:
    review_count += 1
    invoke the feature-dev:code-reviewer fan-out (parallel, per-dimension — see above) on the modified files in the worktree
    aggregate the agents' ## Verdict lines (PASS only if every dimension PASSes; any FAIL = FAIL with the union of issues)

    if verdict == PASS:
        break → proceed to the Codex second-opinion step, then the Review Gate Checklist, then Step 8
    if verdict == FAIL:
        fix every reported issue in the worktree
        run build/tests to confirm fixes compile and pass
        continue loop (re-run the reviewer fan-out to verify fixes)

if review_count == 5 and last verdict still FAIL:
    HALT for human — write BLOCKED (category: unfixable-task), preserve the worktree, do not merge
    (see Edge Cases → Code-reviewer unfixable issues)
```

**Do not exit the loop early.** After fixing issues, you MUST re-invoke the reviewer before proceeding — a passing build is necessary but not sufficient. Fixes can introduce new issues (e.g., changing an API call may require updating its callers). Only proceed to Step 8 when the reviewer returns a PASS verdict. If it cannot reach PASS within 5 cycles, HALT for a human (`BLOCKED`, `unfixable-task`) — there is no interactive "accept known debt" path in autonomous mode.

> **VIOLATION EXAMPLE — "Self-Dismissal"**
>
> This is the exact failure pattern this rule prevents:
>
> 1. Reviewer returns `## Verdict: FAIL — 1 issue(s)` (e.g., dependency version mismatch, confidence ≥ 80)
> 2. Orchestrator reads the finding
> 3. Orchestrator writes its own analysis: "Looking at this more carefully, the reviewer flagged X but this is actually a false positive because Y"
> 4. Orchestrator skips re-review and proceeds directly to merge
> 5. The run never halts for a human (no `BLOCKED` signal)
>
> **Any response matching this pattern is a skill violation.** It does not matter if the analysis is correct. The orchestrator does not have authority to override the reviewer — only a re-review (PASS verdict) — or, after 5 failed cycles, a human via a `BLOCKED` halt — can clear a FAIL verdict.

**After a PASS verdict — Codex second opinion (runs automatically):**

Run this automatically once per task, after the loop above produces the first `## Verdict: PASS`. Do not ask the user — it always runs, and never gates the merge (`feature-dev:code-reviewer` remains the only authoritative gate).

1. **Locate the Codex companion runtime** (installed by the `codex` plugin):

   ```bash
   CODEX="$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)"
   ```

   If `$CODEX` is empty, Codex is not installed: record "Codex unavailable — see /codex:setup" in the Review Gate Checklist and proceed to merge. Do not block a task that already passed the reviewer on an advisory second opinion.

2. **Run a standard Codex review** on this task's committed changes, from inside the worktree:

   ```bash
   node "$CODEX" setup --json          # verify the CLI is present and authenticated
   cd .worktrees/TASK-NN
   node "$CODEX" review --wait --scope branch --base main

   # Tear down THIS task's review broker so we leave the system as we found it. `review` spawns a
   # per-worktree app-server-broker.mjs daemon (detached, no idle timeout) that the codex plugin's
   # session-end reaper cannot reclaim once this worktree is removed. Kill exactly that broker by the
   # PID it wrote to its own pid-file, after confirming the live process is that broker for THIS
   # worktree — never the editor/desktop Codex app-server. SIGTERM lets it unlink its socket + pid-file
   # and exit cleanly. Best-effort: a missing broker is a no-op and never affects the merge.
   WT="$(pwd -P)"
   find "${TMPDIR:-/tmp}" -maxdepth 2 -path '*/cxc-*/broker.pid' -type f 2>/dev/null | while IFS= read -r pf; do
     bpid="$(cat "$pf" 2>/dev/null)"; case "$bpid" in ''|*[!0-9]*) continue;; esac
     case "$(ps -ww -o command= -p "$bpid" 2>/dev/null)" in
       *app-server-broker.mjs*"--cwd $WT"*) kill -TERM "$bpid" 2>/dev/null || true ;;
     esac
   done
   ```

   The review returns JSON: `verdict` (`approve` | `needs-attention`), `summary`, `findings[]` (each with severity, file, line range, confidence, recommendation), and `next_steps`. (`main` is the same integration branch used in Steps 5 and 8.) If the run errors on setup or auth, record the message and "see /codex:setup" in the checklist and proceed to merge (advisory; it never blocks). After the review returns, the same block tears down this task's Codex broker — best-effort, and it never gates the merge.

3. **Codex findings are advisory — they are NOT authoritative.** Codex is a non-authoritative second opinion. Unlike `feature-dev:code-reviewer` (the authoritative gate you must never self-dismiss), Codex's findings are *just findings*: not directives, not binding suggestions, and its `approve`/`needs-attention` verdict does **not** gate the merge. Record the findings, then decide on their merits which — if any — are worth acting on. For every finding you decline to act on, give a one-line reason.

4. **If no Codex finding warrants action:** note that in the Review Gate Checklist and proceed to merge.

5. **If any Codex finding warrants action:** do NOT patch-and-merge directly, and do NOT treat Codex's text as the fix spec. Implement the change in the worktree, then re-run the fix → `feature-dev:code-reviewer` re-review loop (`review_count = 0`, max 5 cycles, no self-dismissal) until the reviewer returns `## Verdict: PASS`. On that PASS, proceed directly to the Review Gate Checklist and merge — **do not return to this Codex step.** The second opinion runs once per task, and the authoritative gate remains `feature-dev:code-reviewer`.

**Review Gate Checklist — required before proceeding to Step 8:**

Before moving to Step 8, output this checklist in your response. If the aggregate verdict is FAIL, you MUST NOT merge — HALT for a human (`BLOCKED`, `unfixable-task`).

```
### Review Gate
- Reviewer verdict: [PASS / FAIL]  (aggregate across the parallel dimension reviewers)
- Dimensions reviewed: [correctness, security, conventions, simplicity, tests]  (note any skipped + why)
- Issues reported: [0 / N]
- Exit condition: [clean report (PASS) / HALTED — BLOCKED: unfixable-task]
- Codex second opinion: [unavailable — see /codex:setup / ran: approve / ran: needs-attention — M finding(s)]
- Codex findings actioned: [n/a / none — judged advisory (one-line reason each) / addressed K, re-reviewed to PASS]
```

The only clean exit from Step 7 is a `feature-dev:code-reviewer` aggregate PASS verdict. If PASS cannot be reached within 5 cycles, the run HALTs for a human (`BLOCKED`, `unfixable-task`) — there is no interactive acceptance path. The Codex second opinion never gates the merge; you still exit Step 7 only via a `feature-dev:code-reviewer` PASS.

#### Step 8: Merge to main

```bash
cd /path/to/project
git merge --squash task/TASK-NN
# Run build verification
<build command from Discovered Facts>
# Run test verification
<test command from Discovered Facts>
# Commit
git commit -m "[TASK-NN] <title>"
# Clean up worktree
git worktree remove .worktrees/TASK-NN
git branch -d task/TASK-NN
```

If build or test verification fails, attempt to fix and re-run. If it still fails, HALT for a human — do not commit a failing merge (see Edge Cases → "Build or test failure on merge").

#### Step 9: Update manifest

After successful merge:

- Mark addressed requirements as `done (TASK-NN)`
- Add entry to Completed Tasks with summary of what was built
- Log any adjustments (scope changes, new discoveries, requirement modifications) in the Adjustments Log
- Update Remaining Work
- If new follow-up items were identified, add to Follow-up Items

#### Step 10: Signal status — HARD STOP

**This step is a mandatory stopping point — you MUST stop responding after writing the status signal below.** Do exactly one task per invocation. The external loop re-invokes the skill in a fresh process for the next task.

After updating the manifest in Step 9:

1. Write `TASKS/.ib-status` with first line `CONTINUE` (one task merged; more work — or validation — remains).
2. Print the `CONTINUE` status marker, then **stop — do not output anything else, do not continue to Step 11, do not design the next task, do not launch any sub-agents**:

```
=== ITERATIVE-BUILDER STATUS: CONTINUE ===
TASK-NN merged to main. <M> requirement(s) remain.
Re-invoke to continue: /iterative-builder-ultracode-auto @TASKS/MANIFEST.md
===
```

Replace `TASK-NN` with the task ID just completed and `<M>` with the count of pending requirements (use `0 — validation pending` when the last requirement just merged).

**Why this matters:** The manifest and task cards on disk contain all state needed to continue. Designing the next task in the same context risks stale assumptions about the codebase. Stopping after one task means the next invocation — a fresh process — designs against a fresh read of the repo. The orchestrator cannot and must not batch a second task into this invocation.

#### Step 11: Continue or finish (next invocation)

Step 10 already stopped this invocation, so this decision is made at the start of the NEXT invocation, when the loop re-runs the skill against the existing manifest:

- If all requirements are `done` or `deferred` and Phase 3 has not run yet, proceed to Phase 3 (validation).
- Otherwise, return to Phase 2 Step 1 and design the next task.

(How an invocation decides to bootstrap vs. resume is in Output Delivery.)

### Phase 3: Validation

#### Step 1: Check success criteria

For each success criterion (`SC-01`, `SC-02`, ...):

- Verify it is satisfied by the completed tasks
- Note which task(s) addressed it
- Flag any gaps

**Ultracode fan-out (multi-agent):** Launch one verifier agent per success criterion in a single message (foreground), each reading the merged `main` plus the task(s) claiming that SC and adversarially testing whether it is GENUINELY satisfied — not merely claimed. Each returns `met` / `gap` with observable evidence (file:line, passing test, or observed behavior per Rule 9), which task(s) addressed it, and `unverified` + reason where only runtime/external systems could prove it (Rule 1). Aggregate verbatim into one gap list: any `gap` or unproven `unverified` becomes a flagged gap; never mark an SC met without cited evidence. These verifiers check SC satisfaction, not code quality — they do not touch the Rule 15 review gate, and they only FEED Step 3.2, where the orchestrator decides autonomously (create tasks to close the gap / record an external-only gap / accept). Scale down for few/simple SCs. When there are more than ~8 success criteria, or per-SC verification needs builds/test suites that are slow to run serially, run the fan-out via the Workflow tool in the background (true ultracode) — wait for completion, then assemble the aggregated gap list for the Step 3.2 gap handling. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 2: Handle gaps

If success criteria have gaps, handle them autonomously:

- **Closable by building** (missing or incomplete code): add a new requirement (`REQ-NN`, `pending`) to the manifest, log it in the Adjustments Log, write `CONTINUE`, and stop — the next invocation designs a task for it (loop back to Phase 2).
- **Provable only by an external/runtime system** (needs a deployed service, staging, or a cloud resource the run cannot exercise): record it as `unverified` with the reason in the final report. If the goal genuinely cannot be called done without that human check, HALT (`BLOCKED`, `external-check`); otherwise note it as a follow-up and continue to finalize.
- Never silently accept an unmet, code-closable gap.

#### Step 3: Finalize

- Write final manifest state to `TASKS/MANIFEST.md`.
- Write `TASKS/.ib-status` with first line `COMPLETE`.
- Print the `COMPLETE` status marker followed by the summary report: tasks completed, requirements met, any deferred items, assumptions made, follow-up work, and any `unverified` success criteria awaiting an external check.

## Core Rules

1. **Verify repo facts with tools; do not guess.**
   - Use Glob, Grep, and Read before naming exact files, modules, tables, endpoints, commands, or conventions.
   - If something cannot be verified with tools (requires runtime, external service, staging environment), mark it `unverified` and explain why.
   - Do not invent exact filenames, modules, tables, endpoints, commands, or environment prerequisites unless verified by tools or explicitly provided in the input.
   - For greenfield projects where no code exists yet, paths from the input document (project names, endpoint routes) are treated as verified. Internal paths that follow established framework conventions (e.g., Controllers/, Entities/, Repositories/) should be labeled `(convention-based)` in Discovered Facts. Artifact paths for files that will be created are acceptable when they follow stated conventions.
   - In autonomous mode, do not ask: when a product-judgment gap remains, record an explicit assumption (Decision Ledger) and proceed (see Autonomous Operation and Rule 18). Still never invent repo facts — verify with tools or mark `unverified`.

2. **Preserve intent across iterations.**
   - Do not let implementation convenience override locked user decisions.
   - Do not quietly reintroduce deferred scope.
   - Do not reduce required scope when designing a task. If the requirement demands full behavior, the task card must deliver full behavior — not a structural shell, stub, skeleton, placeholder, or "initial structure."
   - Scope-reducing language ("initial structure", "not full flow", "stub", "shell", "placeholder", "skeleton", "basic scaffold") in a task's Goal or In Scope is a defect when the mapped requirement demands full behavior.
   - Carry forward stated constraints on behavior, API compatibility, latency, data model, dependency choices, rollout, security, compliance, and backward compatibility.

3. **Anchor cards to outcomes, not just work.**
   - Every requirement must be addressed by at least one task.
   - Every task must say what requirement(s) it addresses.

4. **Use goal-backward must-haves.**
   - Every task card must include `Truths`, `Artifacts`, and `Key Links`.
   - These are for execution safety, not documentation theater.

5. **Keep discovery honest.**
   - Resolve what you can during bootstrap and task design.
   - Only create discovery cards when the unknown genuinely requires runtime behavior, external systems, or broader investigation than the planning pass can support.

6. **Keep cards fresh-thread safe.**
   - A new agent should be able to execute a card without rediscovering the plan.
   - Include all necessary context in the card itself.

7. **Separate risky rollout work.**
   - Split schema preparation, compatibility, backfill, cutover, and cleanup when applicable.
   - Isolate breaking changes and coordinated rollouts.

8. **Make verification executable.**
   - Include concrete commands whenever possible.
   - If commands are inferred, label them best-effort.
   - List known prerequisites for running them.
   - Verification should include both structural checks (build, file existence) and at least one behavioral check when feasible (test execution, endpoint response, output validation).
   - For test tasks, verification commands must always include running the test suite.
   - For non-test implementation tasks, if Artifacts include substance constraints (`contains`, `min_lines`, `exports`), verification commands should include at least one content check that validates a substance constraint.
   - Prefer stack-native commands (e.g., `dotnet test`, `pytest`, `npm test`) over raw shell utilities.

9. **Make acceptance criteria observable.**
   - Criteria must describe externally visible behavior, contract guarantees, data invariants, or testable internal outcomes.
   - Avoid vague criteria like "works correctly" or "is production ready".

10. **Separate required work from optional work.**
    - Nice-to-have improvements, future cleanup, and post-launch hardening belong in Follow-up Items.
    - A behavior the requirement marks as required must not appear in Follow-up Items. If it cannot fit in the current task, create a new requirement for it.

11. **Keep the result compact.**
    - Do not add ceremony for tiny, low-risk changes.
    - For small work, keep the ledger and must-haves terse but still present: keep all four ledger buckets (empty ones shown as `None`) with a couple of bullets each and no speculative open questions (see [references/decision-ledger.md](references/decision-ledger.md) → Scale to the Task Size).
    - For small work, prefer the fewest requirements that capture the deliverable (often one or two) and keep success criteria minimal; keep manifest sections present but terse (empty sections shown as `None`, not padded and not dropped). Do not split a cohesive small change into multiple requirements; reserve fine-grained decomposition for large or separable work.

12. **Do not write implementation code during planning.**
    - The bootstrap phase and task card design only plan and decompose work.
    - Implementation happens in the worktree during the per-task loop.

13. **Design each task against the real codebase.**
    - The code-architect must read the current state of main (including all previously merged tasks) when designing each card.
    - Never design a card against a projected future state.

14. **One task per invocation; hard-stop after each.**
    - After merging a task, write the `CONTINUE` status (Step 10) and **stop responding**. Do not continue to Step 11 or begin the next task in the same invocation.
    - Each invocation is a fresh process driven by the external loop — that fresh process is what gives the next task a clean read of the repo (no `/clear` by a human needed).
    - Do not rationalize batching a second task ("still fresh", "context is small", "just one more task"). One unit of work per invocation, then signal and stop.
    - The manifest and task cards on disk are the source of truth — the context window is not.

15. **Never override the independent reviewer.**
    - `feature-dev:code-reviewer` is an independent quality gate. The orchestrator has zero authority to evaluate, dismiss, downgrade, or reinterpret its findings.
    - When the reviewer reports issues, the only valid actions are: fix and re-review, or — if PASS cannot be reached within 5 cycles — HALT for a human (`BLOCKED`, `unfixable-task`). There is no autonomous "accept the finding and proceed" path.
    - Rationalizing a finding away ("this is actually fine", "false positive", "backward compatible") is a skill violation.
    - This prohibition applies to `feature-dev:code-reviewer` only. The Codex second opinion is explicitly advisory — see rule 16.

16. **Codex review is advisory, not a gate.**
    - The Codex second opinion (Step 7) runs automatically after `feature-dev:code-reviewer` returns PASS, without asking the user, and never blocks the merge.
    - Codex findings are *just findings* — non-authoritative, not binding directives. Unlike reviewer findings, you may evaluate them on their merits and decide which (if any) to act on.
    - Acting on a Codex finding means implementing the change and re-passing `feature-dev:code-reviewer` — never patch-and-merge on Codex's say-so. The authoritative gate is always `feature-dev:code-reviewer`.
    - The Codex second opinion runs once per task.

17. **Multi-agent fan-out augments; it never weakens a gate.**
    - Seven steps fan out across parallel sub-agents (1.1, 1.2, 1.3, 2.2, 2.3, 2.7, 3.1) — automatically, in the foreground by default. The three heavy steps escalate to the Workflow tool (true background ultracode) when the thresholds in the Multi-Agent Orchestration section are met (2.7 on a large diff; 1.1 / 3.1 on a large repo or many SCs); the other four stay foreground by default (rare per-step exceptions aside). See the Multi-Agent Orchestration section and [references/ultracode-fanout.md](references/ultracode-fanout.md).
    - Fan-out feeds a step; it never weakens a gate. The former user-approval steps (1.5, 2.4, 3.2) now run autonomously (write/decide and proceed — see Autonomous Operation), and `feature-dev:code-reviewer` stays the single authoritative review gate — parallel reviewers add breadth only, aggregated conservatively (PASS only if every dimension passes; any FAIL = FAIL with the union of findings), with no finding ever dismissed or dropped (this extends Rule 15).
    - Fan-out parallelizes work within a single step only. It never parallelizes the per-task loop across tasks and never creates execution waves — tasks are designed, built, reviewed, and merged one at a time.
    - Scale the fan-out to the work (Rule 11): collapse to one agent or skip it for tiny, low-risk work.

18. **Operate autonomously; halt only when a human is truly required.**
    - Never ask the user a question and never wait for approval. Write the manifest and task cards to disk and proceed (see Autonomous Operation).
    - Resolve product-judgment gaps with explicit, logged assumptions (Rule 1) — never block on them.
    - Halt (`BLOCKED`) and stop only for: `credentials`, an `external-check` the code cannot perform, a `risk` (destructive/irreversible action), or an `unfixable-task`. For anything else, keep going.
    - Quality gates are not user questions: the reviewer gate (Rule 15), the Step 2.3 critics, the Codex advisory (Rule 16), and the fan-out (Rule 17) all stay in force.

## Discovery Cards

Create discovery cards only when planning cannot resolve an unknown up front.

Each discovery card must:

- state what is unknown
- state why planning could not resolve it
- produce a concrete output
- name which requirements it unblocks
- write findings to `DISCOVERY-NN.md`

Follow-on tasks that depend on a discovery card must reference the `DISCOVERY-NN.md` file in their Context Anchor.

A discovery card that could have been replaced by normal repo inspection is a defect.

## Sizing Rules

Use these buckets only as an internal check:

- `XS`: tiny, surgical, very low risk
- `S`: small, focused, standard PR
- `M`: moderate, still reviewable in one sitting

Do not create `L` or `XL` cards. Split the requirement further.

Splitting heuristics (guidelines, not hard rules — justify exceptions if needed):

- A card that creates more than ~12 new files or modifies more than ~8 existing files is likely L. Consider splitting by sub-domain, sub-feature, or artifact type.
- A card that implements more than ~5 independent use cases, handlers, or controllers is likely L. Split by functional area.
- A card that combines fundamentally different work types (e.g., REST endpoints + WebSocket orchestration, or DbContext + repositories + migration) should be split along the type boundary.
- Formulaic CRUD across many resource types (e.g., 6 admin resources x 4 operations = 24 use cases) exceeds the ~5 use case guideline even though each operation is simple. Split by resource sub-group rather than by operation type.

## Task ID Rules

- Always use the prefix `TASK-` followed by a zero-padded two-digit number starting at `01`: `[TASK-01]`, `[TASK-02]`, ..., `[TASK-99]`.
- If there are more than 99 tasks, continue with three digits: `[TASK-100]`, `[TASK-101]`, etc.
- Never use project-specific prefixes like `[PA-01]`, `[AUTH-01]`, or `[DB-01]`. The prefix is always `TASK-`.
- Number tasks sequentially in the order they are created during the per-task loop.

## Anti-Stub Patterns

When a task creates files that are high-risk for stubbing (API routes, data-fetching components, business logic modules), include at least one substance constraint in the Artifacts field of Must-Haves. Good constraints: `min_lines: 30`, `contains: prisma.*.findMany`, `exports: [handleSubmit, validateInput]`. The constraint should match the real work, not just file existence.

Read [references/anti-stub-patterns.md](references/anti-stub-patterns.md) for the full stub indicator reference.

## Edge Case Handling

### Manifest edited between invocations

Because the skill runs one task per invocation, a human can steer the build between iterations by editing `TASKS/MANIFEST.md` directly — the skill never stops to solicit this. On the next invocation, reconcile the edits before selecting a task:

1. **New requirements** (a human added `REQ-NN`): ensure status `pending`, log the addition in the Adjustments Log, and design a card for it when it becomes the next logical piece of work.
2. **Changed requirements** (text edited): keep the new text, log the change in the Adjustments Log with before/after; if a completed task only partially addressed the old text, note the delta and create a new requirement for it.
3. **Removed or deferred requirements**: move the description to Deferred / Out of Scope and skip it.
4. **Revised or overridden assumptions** (a human replaced a logged assumption or added a Locked Decision): treat the human's value as binding, log the override, and re-design any not-yet-merged card that depended on the old assumption.

Within an invocation, card quality is enforced autonomously by the Step 2.3 quality-check critics (which loop back to the architect on FAIL) — there is no interactive card rejection.

### Code-reviewer unfixable issues (5 cycles)

When the reviewer still returns FAIL after 5 fix cycles, HALT for a human — do not merge:

1. Mark the requirement `in-progress (TASK-NN — blocked)` in the manifest and log the unresolved findings in the Adjustments Log.
2. Leave the worktree `.worktrees/TASK-NN` and branch `task/TASK-NN` intact so a human can inspect or fix them.
3. Write `TASKS/.ib-status` = `BLOCKED` and `TASKS/.ib-block.md` (category `unfixable-task`, the union of unresolved findings, the worktree path, and how to resume), print the `BLOCKED` marker, and stop.

### Codex review unavailable or errors

When the Codex second opinion (Step 7) runs but Codex is not installed, not authenticated, or the run errors:

1. Record the exact message and "see /codex:setup" (it checks the CLI and auth, and can install via `npm install -g @openai/codex`).
2. Record "Codex unavailable — see /codex:setup" in the Review Gate Checklist.
3. Proceed to merge. The advisory second opinion never blocks a task that already passed `feature-dev:code-reviewer`. (Missing Codex auth is not a `credentials` halt — Codex never gates the merge.)

### Implementation failure

When the implementation sub-agent fails to complete the task:

1. Collect the failure context (what was attempted, what failed, error messages).
2. Re-invoke code-architect with the failure context to redesign the approach. Bound this to 2 redesign attempts.
3. If the requirement is too large, split it into smaller requirements and continue with the smallest viable piece.
4. If the requirement is blocked by an external factor (a credential, an external/cloud dependency), HALT for a human (`BLOCKED`, `credentials` or `external-check`) with the blocker recorded.
5. If it still cannot be implemented after the bounded redesigns and is not splittable, HALT for a human (`BLOCKED`, `unfixable-task`) with the failure context; leave the worktree intact.

### Build or test failure on merge

When the merge to main fails build or test verification:

1. Attempt to fix the failure in the worktree and re-run the full reviewer loop on the fix (a fix is code that must itself pass review).
2. Re-run build/test verification.
3. If it still fails, HALT for a human — do not commit a failing merge: undo the staged squash so `main` stays clean (the work remains on `task/TASK-NN` and in the worktree), record the failure details in the Adjustments Log, write `TASKS/.ib-status` = `BLOCKED` and `TASKS/.ib-block.md` (category `unfixable-task`), print the `BLOCKED` marker, and stop.

## Output Delivery

Write the output as a `TASKS/` directory in the project root containing:

- `MANIFEST.md` — the living manifest (updated after every task)
- `TASK-NN.md` — individual task cards (created as each task is designed)
- `.ib-status` — the loop status sentinel: first line `CONTINUE` / `COMPLETE` / `BLOCKED` (written at the end of every invocation; see Autonomous Operation)
- `.ib-block.md` — written only on `BLOCKED`: the halt category, what a human must do, current state, and how to resume

**Bootstrap vs. resume (decided at invocation start):** if `TASKS/MANIFEST.md` already exists, RESUME from it — never overwrite the manifest's history and never ask; reconcile any human edits (Edge Cases → "Manifest edited between invocations") and continue from the manifest's state (next pending task, or Phase 3 if none remain). Only when no `MANIFEST.md` exists do you bootstrap a fresh one (Phase 1).

The manifest is a living document. It starts with requirements only (Phase 1) and grows as tasks are designed, implemented, and completed (Phase 2). By the end, it provides a complete record of what was built, what changed, and what remains.

Read [references/manifest-template.md](references/manifest-template.md) for the manifest format.
Read [references/task-card-template.md](references/task-card-template.md) for the task card format.

## Final Quality Bar

### Per-task checks (before finalizing each card)

- repo grounding was completed and Discovered Facts use tool-verified evidence
- the card names what requirement(s) it addresses via the `Addresses` field
- must-haves include Truths (observable behavior), Artifacts (concrete deliverables), Key Links (critical wiring)
- Truths describe observable behavior or invariants, not implementation steps
- no scope-reducing language in Goal or In Scope for source-required behavior
- substance constraints present in Artifacts for files that are high-risk for stubbing
- verification commands and prerequisites are realistic and executable after the task completes
- the card is single-purpose and fits sizing guidelines
- locked decisions from the ledger are honored
- no deferred/out-of-scope items are implemented
- the context anchor explains why this task exists and what to read first
- no exact filenames, commands, or prerequisites were invented without evidence
- Change Safety and Failure Signals are present
- every type, interface, or artifact referenced is traceable to the task itself, a predecessor task, or an existing repo file
- the card is understandable in a fresh thread without access to this conversation

### Per-manifest checks (after each update)

- every requirement has a status (`pending`, `in-progress (TASK-NN)`, or `done (TASK-NN)`)
- completed tasks have summaries of what was built
- adjustments are logged with rationale
- remaining work accurately reflects what is left
- follow-up items contain only explicitly optional or post-launch work
- success criteria are tracked against completed tasks
- the manifest provides a complete record usable by someone joining the project

Return the result as ready-to-execute task files. Do not write implementation code during the planning and card design phases.
