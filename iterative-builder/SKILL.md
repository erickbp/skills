---
name: iterative-builder
description: Iterative build workflow — plan one task at a time against actual codebase state. Each task card is designed by code-architect reading the real codebase, auto-approved unless sensitive or complex, implemented in a worktree, reviewed, and merged before the next task is planned. Use when upfront planning would diverge from reality or when the sequence of work should emerge organically.
---

# Iterative Builder

Plan one task at a time against the actual codebase, build it, review it, commit it, then plan the next.

## Objective

- Produce task cards that are designed against the real codebase state, not a projected future state.
- Let the sequence of work emerge organically from completed work and remaining requirements.
- Keep the user in the loop on consequential work — the orchestrator proceeds with its recommended task card automatically, pausing for explicit approval before implementation only when a card is sensitive or complex (high risk, high-blast-radius, or blocked on an open decision).
- Deliver fully reviewed, tested, committed code after each task before moving on.
- Fan out the read- and verify-heavy steps across parallel sub-agents to widen coverage — foreground by default, escalating to true background orchestration (the Workflow tool) on the heavy steps — while keeping the user-approval gates (the one-time manifest gate, the conditional per-task card gate, and the validation gate) and the authoritative reviewer intact.

## Multi-Agent Orchestration

Seven steps fan out across parallel sub-agents rather than running as a single pass: **1.1** (ground the repo), **1.2** (decision ledger), **1.3** (extract requirements), **2.2** (design the card), **2.3** (quality-check the card), **2.7** (review), and **3.1** (check success criteria). Each carries an **Ultracode fan-out** block with its recipe. Apply the fan-out automatically; do not ask the user whether to use it. (Throughout, a dotted id `X.Y` means Phase X, Step Y — e.g. `2.7` is Phase 2, Step 7; the per-step table in [references/ultracode-fanout.md](references/ultracode-fanout.md) maps each id to its phase and step name.)

Two distinct mechanisms do this work — do not conflate them:

- **Default — foreground sub-agents (no opt-in needed).** Launch the step's agents as inline `Agent` calls in a single message, in the foreground (the pattern feature-dev and `/code-review` use), so results land in the same turn and flow into the next gate. This is always available, needs no opt-in, and is the default for all seven steps.
- **Heavy steps — the Workflow tool (true background "ultracode").** On the three heavy steps, when a threshold below is met, **call the Workflow tool** to fan out in the background instead. It runs asynchronously — the call returns a task id and the result arrives later via a notification, after which you aggregate and proceed to the same gate. Invoking this skill is itself the opt-in to the Workflow tool for these steps, so use it whenever a threshold is met; do not ask first. Below the thresholds, stay foreground.
- **Scale to the work (Rule 11):** match agent count to risk; collapse to one agent or skip fan-out for tiny, low-risk work.

| Step | Call the Workflow tool when… |
|------|------------------------------|
| **2.7** Review | the task's diff spans more than ~10 files or ~400 changed lines, or the reviewers are slow to run inline |
| **1.1** Ground the repo | the repo exceeds ~1,000 tracked source files, or is a monorepo with 3+ packages/workspaces |
| **3.1** Success criteria | there are more than ~8 success criteria, or per-SC verification needs slow builds/test suites |

The background fan-out must complete and be aggregated **within the same task** — before the step that follows, and never across the Step 10 boundary (the per-task soft re-ground or a periodic hard reset). Results return to the same gate with the same conservative aggregation rule. The other four steps (1.2, 1.3, 2.2, 2.3) are lightweight and stay foreground by default — escalate them only in the rare exceptions their per-step recipes note (1.2 never escalates).

This augments the workflow; it never weakens it. Fan-out **feeds a gate, never replaces one** — every user-approval gate stays: 1.5 and 3.2 as hard gates, and 2.4 as the conditional card gate (auto-proceed on routine cards; pause on sensitive/complex ones). `feature-dev:code-reviewer` stays the single authoritative review gate: parallel reviewers only add breadth, aggregated conservatively (PASS only if every dimension passes; any FAIL is a FAIL with the union of findings), and no finding is ever dismissed, downgraded, or dropped (Rule 15). Codex stays advisory (Rule 16) and the context reset stays (Rule 14 — a soft re-ground each task, a hard `/clear` periodically). Fan-out parallelizes work **within a single step only** — it never parallelizes the per-task loop across tasks, and never creates execution waves; tasks are still designed, built, reviewed, and merged one at a time.

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
   - Default/integration branch: the branch tasks merge into (e.g., via `git symbolic-ref --short refs/remotes/origin/HEAD` or `git rev-parse --abbrev-ref HEAD`). Record it in Discovered Facts; `main` in the worktree/merge commands (Steps 5, 7, 8) stands for it.
3. Do not invent repo facts that tools can verify. You must use tools to verify these facts.
4. If the repo cannot answer something and the answer is product intent, ask the user before proceeding.
5. For greenfield projects where no code exists yet, paths from the input document are treated as verified. Internal paths that follow established framework conventions should be labeled `(convention-based)` in Discovered Facts.

**Ultracode fan-out (multi-agent):** Run the repo sweep (item 2) as parallel read-only sub-agents launched in ONE message, each covering one angle of the Discovered Facts: (a) topology + build/CI + dependencies, (b) data/persistence layer, (c) API/service layer + routing/middleware, (d) test infrastructure + conventions, (e) 1-2 existing features closest to the goal. Each agent returns ONLY tool-cited facts (Glob/Grep/Read evidence) plus key file paths — no invented facts (Rule 1), no product-intent guesses. Orchestrator reads the surfaced key files, then synthesizes one Discovered Facts set; mark unverifiable items `unverified` and greenfield framework paths `(convention-based)`. If any angle reveals a product-intent gap, escalate to the user (item 4) — fan-out only gathers facts, it never decides intent or replaces the 1.5 manifest approval gate. Scale down for tiny/greenfield repos (fewer agents). When the repo exceeds ~1,000 tracked source files or is a monorepo with 3+ packages/workspaces, run this sweep via the Workflow tool in the background (true ultracode) — wait for it to complete, then synthesize one Discovered Facts set as below. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 2: Build a Decision Ledger

Create a `Decision Ledger` with four buckets:

- `Locked Decisions` — binding user choices and hard constraints
- `Coding Agent's Discretion` — real implementation freedom areas
- `Deferred / Out of Scope` — explicitly excluded work
- `Open Questions` — missing user judgment that materially changes the plan

Rules:

- Ask questions only for user-judgment gaps, not repo facts.
- Treat `Locked Decisions` as binding.
- Treat `Deferred / Out of Scope` as forbidden scope.
- Use `Coding Agent's Discretion` only for real implementation freedom.
- If important questions remain unanswered, do not silently guess.

**Scale the ledger to the task size.** For XS/S work, keep it minimal: keep all four buckets but make their content terse — a couple of short bullets where there's real content and `None` where a bucket is genuinely empty. Use at most a couple of terse bullets each, no multi-clause analysis. A self-resolvable implementation detail is not an `Open Question` (decide it in Discretion or leave it to implementation); raise an `Open Question` only when a genuine user-judgment decision is actually blocked, else record a one-line assumption. A simple change deserves a terse ledger (mostly `None`), not a multi-bullet survey — but keep all four buckets present.

Read [references/decision-ledger.md](references/decision-ledger.md) when building or validating the ledger.

**Ultracode fan-out (multi-agent):** First draft all four buckets in a single pass yourself. Then launch 2-3 completeness critics in parallel (one message, foreground), each independently re-reading the goal + Discovered Facts to flag, with tool evidence, missed `Open Questions`, mis-bucketed Locked-vs-Discretion items, and silently-assumed decisions that belong in `Locked` or `Open`. Merge and dedup every flag into the ledger — additively; a critic may add or re-bucket items but never delete an Open Question or downgrade a Locked decision. Critics surface user-judgment gaps only (never ask about repo facts) and never silently guess a resolution — unresolved items stay as `Open Questions` for the user. For XS/S goals, SKIP this critic panel entirely (it manufactures ceremony on small tasks); use it only when the goal is large or risky enough to warrant it. The ledger still feeds the Step 1.5 approval gate. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 3: Extract requirements and success criteria

From the input and grounding, extract:

- **Numbered requirements**: `REQ-01`, `REQ-02`, ... — each a distinct deliverable or behavior the goal demands.
- **Success criteria**: `SC-01`, `SC-02`, ... — observable conditions that prove the goal is met.

Requirements should be atomic enough that each maps to roughly one task, but coarse enough to avoid ceremony. If a requirement clearly needs multiple tasks, that will emerge naturally during the per-task loop.

**Match the requirement count to the size of the work.** For a small, single-purpose input, prefer the fewest requirements that capture the deliverable — often just one or two — and keep success criteria to the few that actually prove the goal. Collapse trivially-coupled asks (work that one small task delivers together, e.g. a toggle button plus the state it flips) into a single requirement; do not pre-split a cohesive small change into separate REQs. Reserve fine-grained decomposition for genuinely large or separable work, where many requirements are correct (a multi-feature platform legitimately yields 8+).

**Ultracode fan-out (multi-agent):** Launch parallel extractors in ONE message (foreground), each deriving candidate `REQ-`/`SC-` items from the input + Step 1 grounding through one lens — (a) explicit asks, (b) implied behaviors, (c) cross-cutting/non-functional (security, performance, observability, migration/rollout, compliance). Each item cites tool evidence (Rule 1); no item is invented. The orchestrator merges, dedups, and reconciles numbering, then runs one completeness critic asking "what deliverable is implied but unlisted?" Keep `REQ`s atomic (~one task) yet coarse enough to avoid ceremony, and SCs observable (Rule 9). The critic only ADDS candidates — it is not a gate and drops nothing. Scale lenses to the input (Rule 11): for an XS/single-ask input, SKIP the extractor fan-out and the completeness critic entirely — just list the one or two obvious requirements inline; the multi-lens sweep is for rich, multi-feature specs where it prevents missed deliverables. The reconciled set feeds Step 4 and the 1.5 approval gate — fan-out never replaces that gate. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

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

#### Step 5: Present manifest for approval

Present the manifest to the user. The user may:

- Approve as-is
- Add, remove, or modify requirements
- Add or change locked decisions
- Clarify open questions

Incorporate feedback and update `TASKS/MANIFEST.md` before proceeding.

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

**Ultracode fan-out (multi-agent):** Replace the single architect with a judge panel — in ONE message, launch parallel foreground `feature-dev:code-architect` agents (multiple instances of the same agent), each with a different lens (minimal-change / clean-architecture / pragmatic-balance), each reading the **current** codebase (Rule 13) and citing tool evidence (Rule 1), each returning a complete candidate card. Score every candidate on the Step 2.3 dimensions — requirement coverage, anti-stub substance, sizing fit (XS/S/M, or L only under the L allowance), locked-decision adherence — then synthesize ONE winning card, grafting the best elements of the runners-up. This designs a single card from several perspectives; it does not design multiple tasks at once (no execution waves). Scale the panel to the work (Rule 11): 1 architect for XS/low-risk, 2-3 for S/M or higher risk. The synthesized card is not auto-approved by the panel: it still flows into the 2.3 quality-check and the 2.4 card decision (auto-proceed or pause). See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 3: Quality-check the card

Before presenting to the user, verify:

- No scope-reducing language in Goal or In Scope ("initial structure", "stub", "placeholder", "skeleton", "shell", "basic scaffold") for source-required behavior
- Substance constraints present in Artifacts for high-risk files
- Verification commands are concrete and executable
- Must-Haves include Truths (observable behavior), Artifacts (concrete deliverables), and Key Links (critical wiring)
- Truths describe behavior or invariants, not implementation steps
- Locked decisions from the ledger are honored
- The card does not implement deferred/out-of-scope items
- The card fits sizing guidelines (XS/S/M, or L only under the Sizing Rules "L allowance"; never XL), and its `Size`/`Cohesion` fields are set and consistent with the actual artifact list

If the card fails quality checks, re-invoke code-architect with specific feedback.

**Ultracode fan-out (multi-agent):** Instead of one solo pass, launch the checklist as parallel adversarial critics in a SINGLE message (foreground), each attacking ONE dimension of the card and citing the exact card text/repo evidence (Rule 1): (a) scope-reducing language in Goal/In Scope; (b) stub risk vs. substance constraints in Artifacts for high-risk files; (c) Must-Haves shape — Truths are behavior not steps, and Truths/Artifacts/Key Links are all present; (d) Verification Commands concrete and executable; (e) locked-decision adherence + no deferred/out-of-scope leakage; (f) sizing — `Size` is XS/S/M, or L only when the L allowance holds (verify `Cohesion`=uniform, Risk=low, and an anti-stub substance constraint on every repeated artifact against the actual artifact list, not self-attested); never XL. Each returns PASS/FAIL + specific issues. Aggregate conservatively: PASS only if EVERY critic returns PASS; ANY FAIL = aggregate FAIL — re-invoke `feature-dev:code-architect` (loop back to Step 2) with the union of all issues, then re-run the critics. This is a pre-user planning-quality gate: it FEEDS the Step 4 card decision and never replaces it; it is NOT the `feature-dev:code-reviewer` code-review gate (Rule 15). Scale down for tiny/low-risk cards — fold critics into one pass; never add ceremony (Rule 11). See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 4: Decide on the task card (auto-proceed by default)

The orchestrator proceeds with its recommended card automatically and pauses for explicit user approval **only when the card is sensitive or complex**. This is a conditional gate, not a removed one — read the card's own fields to decide; the rule is a mechanical field check, not a judgment call.

**Pause for explicit approval when ANY of these hold:**

- `Risk Level: high` (breaking changes, data migration, security-sensitive), OR
- high-blast-radius per Rule 7 — schema/migration/cutover/breaking change, security-boundary, or cross-protocol work, OR
- `Change Safety` is `coordinated`, `cleanup-later`, or `unknown`, OR
- an unresolved `Open Question` or a Locked-decision conflict materially bears on this card (the design hinges on a user-judgment call that is not settled).

When pausing, present the card and wait. The user may:

- Approve the card
- Request changes (re-invoke code-architect with feedback)
- Defer the requirement (move to Deferred, update manifest)
- Inject a new requirement (add to manifest, design card for it)

**Otherwise — a routine card (Risk `low`/`medium`, Change Safety `additive`/`reversible`/`feature-flagged`, not high-blast-radius, no blocking ambiguity) — auto-proceed:** present the card followed by a one-line non-blocking note — `Implementing TASK-NN now — reply to intervene or edit the card` — and continue directly to Step 5 without waiting. The user can still intervene (the note invites it), and the Step 7 review gate is unchanged, so a flawed card is still caught before merge.

The same risk bar governs the high-blast-radius post-merge pause in Step 10 (Rule 7), so a high-blast-radius task pauses both before implementation (here) and after merge.

#### Step 5: Create worktree

Run from the primary worktree (the project root). `main` in this and the following steps stands for the repo's **default/integration branch** as detected during Step 1 grounding — substitute it if the repo uses `master` or another name.

```bash
# Preconditions (first task / resume): the target must be a git repo on a clean tree.
git rev-parse --is-inside-work-tree >/dev/null 2>&1 || git init   # greenfield: init, then make an initial commit before the first checkout
test -z "$(git status --porcelain)" || { echo "Uncommitted changes on main — commit or stash before continuing"; exit 1; }

git checkout main
[ -n "$(git remote)" ] && git pull --ff-only   # skip for local-only / greenfield repos with no upstream

# Reclaim any leftover state from a crashed or abandoned prior run before creating the worktree.
git worktree prune
# If .worktrees/TASK-NN or task/TASK-NN already exists, STOP and ask the user whether to resume the
# existing worktree (it may hold uncommitted work) or discard it — do NOT blindly force-recreate
# (see Edge Cases → "Merge conflict or leftover git state").
git branch task/TASK-NN
git worktree add .worktrees/TASK-NN task/TASK-NN
```

#### Step 6: Implement

Launch an implementation sub-agent in the worktree with the finalized task card as its operating instructions. The sub-agent follows the Execution Protocol and Guardrails embedded in the task card.

#### Step 7: Review

**NEVER self-dismiss reviewer findings.** You are not qualified to judge whether a reviewer finding is valid — only the reviewer (via re-review after a fix) or the user can make that determination. Any orchestrator analysis of reviewer findings that concludes "this is fine", "false positive", "non-blocking", "backward compatible", or similar is a skill violation regardless of how reasonable the analysis seems. See the violation example below.

**Invoking the reviewer:** When you invoke `feature-dev:code-reviewer`, include this instruction in your prompt to the reviewer:

> End your review with a verdict line in exactly this format:
> `## Verdict: PASS` if no issues found, or `## Verdict: FAIL — N issue(s)` if issues found.

After the reviewer responds, read **only the verdict line** to decide the next action. Do not interpret the substance of individual findings to decide whether they "really" matter.

**Ultracode fan-out (multi-agent):** Inside the max-5 loop, replace the single `feature-dev:code-reviewer` call with parallel `feature-dev:code-reviewer` agents launched in ONE message, each scoped to one dimension — correctness/bugs, security, project-conventions (CLAUDE.md), simplicity/DRY, tests/coverage — and each ending with its own `## Verdict` line. Read only the verdict lines, then aggregate conservatively: aggregate PASS only if EVERY dimension is PASS; ANY FAIL = aggregate FAIL whose issue list is the UNION of all agents' findings. Scale agents to risk (Rule 11): a tiny/low-risk diff may use one reviewer. When the diff spans more than ~10 files or ~400 changed lines (or the reviewers are slow to run inline), fan the dimension reviewers out in the background via the Workflow tool (true ultracode) — wait for completion, then aggregate the verdicts at this same gate; the fix/re-review loop must still converge before Step 8 and never cross the Step 10 boundary. Never self-dismiss, downgrade, or drop a finding — including dropping one dimension's finding because another agent disagrees (Rule 15); the fan-out adds breadth only and feeds, never replaces, the gate. The fix-then-re-review loop, max-5 cap, Codex advisory step (Rule 16), Review Gate Checklist, and the two exit conditions are UNCHANGED; only extend the checklist to list which dimensions were covered. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

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
    escalate to user (see Edge Cases → Code-reviewer unfixable issues)
```

**Do not exit the loop early.** After fixing issues, you MUST re-invoke the reviewer before proceeding — a passing build is necessary but not sufficient. Fixes can introduce new issues (e.g., changing an API call may require updating its callers). Only proceed to Step 8 when the reviewer returns a PASS verdict or the user explicitly accepts known debt.

> **VIOLATION EXAMPLE — "Self-Dismissal"**
>
> This is the exact failure pattern this rule prevents:
>
> 1. Reviewer returns `## Verdict: FAIL — 1 issue(s)` (e.g., dependency version mismatch, confidence ≥ 80)
> 2. Orchestrator reads the finding
> 3. Orchestrator writes its own analysis: "Looking at this more carefully, the reviewer flagged X but this is actually a false positive because Y"
> 4. Orchestrator skips re-review and proceeds directly to merge
> 5. User is never consulted
>
> **Any response matching this pattern is a skill violation.** It does not matter if the analysis is correct. The orchestrator does not have authority to override the reviewer — only a re-review (PASS verdict) or explicit user acceptance can clear a FAIL verdict.

**After a PASS verdict — Codex second opinion (always runs, no prompt):**

Run this once per task, automatically, after the loop above produces the first `## Verdict: PASS`. It runs without asking the user. It is advisory and never gates the merge — `feature-dev:code-reviewer` remains the only authoritative gate.

1. **Locate the Codex companion runtime** (installed by the `codex` plugin):

   ```bash
   CODEX="$(ls -d ~/.claude/plugins/cache/openai-codex/codex/*/scripts/codex-companion.mjs 2>/dev/null | sort -V | tail -1)"
   ```

   If `$CODEX` is empty, Codex is not installed: tell the user to run `/codex:setup`, record "Codex unavailable — see /codex:setup" in the Review Gate Checklist, and proceed to merge. Do not block a task that already passed the reviewer on an advisory second opinion.

2. **Run a standard Codex review** on this task's committed changes, from inside the worktree:

   ```bash
   node "$CODEX" setup --json          # verify the CLI is present and authenticated
   cd .worktrees/TASK-NN
   node "$CODEX" review --wait --scope branch --base main

   # Tear down THIS task's review broker so it doesn't leak (a now-idle $TMPDIR/cxc-* dir may linger until the OS tmp reaper clears it). `review` spawns a
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

   The review returns JSON: `verdict` (`approve` | `needs-attention`), `summary`, `findings[]` (each with severity, file, line range, confidence, recommendation), and `next_steps`. (`main` is the same integration branch used in Steps 5 and 8.) If the run errors on setup or auth, surface the message, point the user to `/codex:setup`, record it in the checklist, and proceed to merge. After the review returns, the same block tears down this task's Codex broker — best-effort, and it never gates the merge.

3. **Codex findings are advisory — they are NOT authoritative.** Codex is a non-authoritative second opinion. Unlike `feature-dev:code-reviewer` (the authoritative gate you must never self-dismiss), Codex's findings are *just findings*: not directives, not binding suggestions, and its `approve`/`needs-attention` verdict does **not** gate the merge. Present the findings, then decide on their merits which — if any — are worth acting on. For every finding you decline to act on, give a one-line reason.

4. **If no Codex finding warrants action:** note that in the Review Gate Checklist and proceed to merge.

5. **If any Codex finding warrants action:** do NOT patch-and-merge directly, and do NOT treat Codex's text as the fix spec. Implement the change in the worktree, then re-run the fix → `feature-dev:code-reviewer` re-review loop (`review_count = 0`, max 5 cycles, no self-dismissal) until the reviewer returns `## Verdict: PASS`. On that PASS, proceed directly to the Review Gate Checklist and merge — **do not return to this Codex step.** The second opinion runs once per task, and the authoritative gate remains `feature-dev:code-reviewer`.

**Review Gate Checklist — required before proceeding to Step 8:**

Before moving to Step 8, output this checklist in your response. If the verdict is FAIL and there is no user quote, you MUST NOT proceed.

```
### Review Gate
- Reviewer verdict: [PASS / FAIL]  (aggregate across the parallel dimension reviewers)
- Dimensions reviewed: [correctness, security, conventions, simplicity, tests]  (note any skipped + why)
- Issues reported: [0 / N]
- Exit condition: [clean report / user accepted — quote user message]
- Codex second opinion: [ran: approve / ran: needs-attention — M finding(s) / unavailable — see /codex:setup]
- Codex findings actioned: [n/a / none — judged advisory (one-line reason each) / addressed K, re-reviewed to PASS]
```

The only two exit conditions from Step 7 are: (a) the reviewer returns a PASS verdict, or (b) the user explicitly accepts known issues (quote their message in the checklist). The Codex second opinion does not add a third exit condition — it never gates the merge; you still exit Step 7 only via a `feature-dev:code-reviewer` PASS (or explicit user acceptance).

#### Step 8: Merge to main

```bash
cd <project root>
git merge --squash task/TASK-NN
# Stop on conflict markers — do NOT commit a conflicted squash.
test -z "$(git diff --name-only --diff-filter=U)" || { echo "Squash merge conflict — resolve or 'git merge --abort' and escalate"; exit 1; }
# Run build verification
<build command from Discovered Facts>
# Run test verification
<test command from Discovered Facts>
# Commit
git commit -m "[TASK-NN] <title>"
# Clean up worktree
git worktree remove .worktrees/TASK-NN
git branch -D task/TASK-NN   # -D (not -d): a squash merge does not mark the branch as merged, so -d would fail
```

If build or test verification fails, attempt to fix. If the fix fails, escalate to the user. If `git merge --squash` reports a conflict (only possible when `main` advanced out-of-band during the task), do not commit — see Edge Cases → "Merge conflict or leftover git state".

#### Step 9: Update manifest

After successful merge:

- Mark addressed requirements as `done (TASK-NN)`
- Add entry to Completed Tasks with summary of what was built
- Log any adjustments (scope changes, new discoveries, requirement modifications) in the Adjustments Log
- Update Remaining Work
- If new follow-up items were identified, add to Follow-up Items

#### Step 10: Re-ground for the next task

After updating the manifest in Step 9, refresh working state before designing the next task. The **default is an in-context soft re-ground** (no human round-trip); a full `/clear` is reserved for a periodic checkpoint. This keeps the build moving without re-paying a human `/clear` after every task, while still guaranteeing each card is designed against current reality.

**Soft re-ground (default — do this between every task):**

1. Re-read `TASKS/MANIFEST.md` from disk — re-anchor on requirement statuses, completed-task summaries, and the Decision Ledger.
2. Re-scan the files the just-merged task changed (and the next requirement's subsystem) with fresh reads or a scoped Step 1.1 grounding sub-agent — cite file evidence for what the code looks like NOW.
3. Explicitly discard pre-merge snapshots: name the earlier file reads that are now stale and superseded.
4. State which assumptions you are dropping.

The soft re-ground is **mechanical and evidence-citing — not a judgment call.** Do not skip it with "still fresh" / "context is small" / "just one more task." Design freshness never depends on the orchestrator's memory: the architect still re-reads current `main` per card (Rule 13).

**Periodic hard reset (the `/clear` checkpoint):** force a full context reset when EITHER — context utilization is nearing the harness's auto-compaction (reset cleanly from disk *before* a lossy summary happens), OR `K` tasks (default 5) have completed since the last hard reset. To hard-reset, print this handoff and then **stop — do not design the next task or launch sub-agents**:

```
---
TASK-NN complete and merged to main. Periodic context reset (K tasks since last reset, or nearing the context limit).

To continue, run these two commands:
1. /clear
2. /iterative-builder @TASKS/MANIFEST.md
---
```

Replace `TASK-NN` with the task ID just completed.

**Human checkpoint (decoupled from the reset):** the per-task human touchpoint is the Step 2.4 card decision, *before* any code is written — now conditional (auto-proceed on routine cards; pause for approval on sensitive/complex ones, on the same Rule 7 bar as below). After a merge, surface a brief non-blocking note ("TASK-NN merged; re-grounding and continuing to the next task — reply to intervene or edit the manifest") and continue. EXCEPTION: for high-blast-radius work (the same Step 2.4 high-blast-radius bar — Rule 7), pause for an explicit human OK after merge before continuing (this is the same work that also pauses at the Step 2.4 card decision).

**Never let the relaxed reset become batched or parallel execution.** Tasks are still designed → built → reviewed → merged strictly one at a time (Rule 17): no card for the next task before this one merges, no overlapping worktrees.

**Why this matters:** The manifest and task cards on disk contain all state needed to continue; the context window is not the source of truth. The soft re-ground re-anchors on that disk state and discards stale snapshots so the next task is planned against current reality, and Rule 13 forces the architect to re-read current `main` regardless — so the per-task human `/clear` is redundant for freshness. The periodic hard reset bounds context rot (and pre-empts a lossy auto-compaction) without paying a human round-trip every task.

#### Step 11: Continue or finish

If all requirements are `done` or `deferred`, proceed to Phase 3. Otherwise, return to Step 1.

### Phase 3: Validation

#### Step 1: Check success criteria

For each success criterion (`SC-01`, `SC-02`, ...):

- Verify it is satisfied by the completed tasks
- Note which task(s) addressed it
- Flag any gaps

**Ultracode fan-out (multi-agent):** Launch one verifier agent per success criterion in a single message (foreground), each reading the merged `main` plus the task(s) claiming that SC and adversarially testing whether it is GENUINELY satisfied — not merely claimed. Each returns `met` / `gap` with observable evidence (file:line, passing test, or observed behavior per Rule 9), which task(s) addressed it, and `unverified` + reason where only runtime/external systems could prove it (Rule 1). Aggregate verbatim into one gap list: any `gap` or unproven `unverified` becomes a flagged gap; never mark an SC met without cited evidence. These verifiers check SC satisfaction, not code quality — they do not touch the Rule 15 review gate, and they only FEED Step 3.2, where the USER decides (create tasks / accept / defer). Scale down for few/simple SCs. When there are more than ~8 success criteria, or per-SC verification needs builds/test suites that are slow to run serially, run the fan-out via the Workflow tool in the background (true ultracode) — wait for completion, then assemble the aggregated gap list for the Step 3.2 gate. See [references/ultracode-fanout.md](references/ultracode-fanout.md).

#### Step 2: Handle gaps

If success criteria have gaps:

- Present gaps to the user
- User decides: create additional tasks (loop back to Phase 2), accept as-is, or defer

#### Step 3: Finalize

- Write final manifest state to `TASKS/MANIFEST.md`
- Report summary: tasks completed, requirements met, any deferred items, follow-up work

## Core Rules

1. **Verify repo facts with tools; do not guess.**
   - Use Glob, Grep, and Read before naming exact files, modules, tables, endpoints, commands, or conventions.
   - If something cannot be verified with tools (requires runtime, external service, staging environment), mark it `unverified` and explain why.
   - Do not invent exact filenames, modules, tables, endpoints, commands, or environment prerequisites unless verified by tools or explicitly provided in the input.
   - For greenfield projects where no code exists yet, paths from the input document (project names, endpoint routes) are treated as verified. Internal paths that follow established framework conventions (e.g., Controllers/, Entities/, Repositories/) should be labeled `(convention-based)` in Discovered Facts. Artifact paths for files that will be created are acceptable when they follow stated conventions.
   - Prefer asking one round of clarifying questions over producing cards built on assumptions.

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
   - High-blast-radius work is: schema/migration, breaking-contract, cutover, security-boundary (authn/authz/trust-boundary), and cross-protocol work (a new transport or protocol surface). This is the single canonical list that the Step 2.4 and Step 10 pause triggers and the Sizing blast-radius rules all refer to as "per Rule 7".

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

14. **Re-ground between tasks; hard-reset periodically.**
    - After merging a task, do an in-context soft re-ground (re-read the manifest, re-scan changed files with cited evidence, discard stale snapshots) and continue — no human `/clear` per task. The Step 2.4 card decision remains the per-task human touchpoint — conditional: auto-proceed on routine cards, pause for approval on sensitive/complex ones.
    - Force a full `/clear` hard reset periodically: when context nears the harness's auto-compaction (reset cleanly from disk first — a disk-backed reset is lossless, a harness summary is lossy) or every K tasks (default 5) since the last reset.
    - The soft re-ground is mechanical and evidence-citing, never a judgment call. Do not skip it with "still fresh" / "context is small" / "just one more task."
    - Never let the loosened reset become batched/parallel execution: tasks are still designed, built, reviewed, and merged strictly one at a time (Rule 17) — no overlapping worktrees, no card for the next task before this one merges.
    - The manifest and task cards on disk are the source of truth — the context window is not. Design freshness is guaranteed by Rule 13 (the architect re-reads current `main`) regardless of reset cadence.
    - For high-blast-radius work (Rule 7), pause for an explicit human OK after merge before continuing.

15. **Never override the independent reviewer.**
    - `feature-dev:code-reviewer` is an independent quality gate. The orchestrator has zero authority to evaluate, dismiss, downgrade, or reinterpret its findings.
    - When the reviewer reports issues, the only valid actions are: fix and re-review, or present to user for explicit acceptance.
    - Rationalizing a finding away ("this is actually fine", "false positive", "backward compatible") is a skill violation.
    - This prohibition applies to `feature-dev:code-reviewer` only. The Codex second opinion is explicitly advisory — see rule 16.

16. **Codex review is advisory, not a gate.**
    - The Codex second opinion (Step 7) runs automatically after `feature-dev:code-reviewer` returns PASS — with no prompt — and never blocks the merge.
    - Codex findings are *just findings* — non-authoritative, not binding directives. Unlike reviewer findings, you may evaluate them on their merits and decide which (if any) to act on.
    - Acting on a Codex finding means implementing the change and re-passing `feature-dev:code-reviewer` — never patch-and-merge on Codex's say-so. The authoritative gate is always `feature-dev:code-reviewer`.
    - The Codex second opinion runs once per task.

17. **Multi-agent fan-out augments; it never weakens a gate.**
    - Seven steps fan out across parallel sub-agents (1.1, 1.2, 1.3, 2.2, 2.3, 2.7, 3.1) — automatically, in the foreground by default. The three heavy steps escalate to the Workflow tool (true background ultracode) when the thresholds in the Multi-Agent Orchestration section are met (2.7 on a large diff; 1.1 / 3.1 on a large repo or many SCs); the other four stay foreground by default (rare per-step exceptions aside). See the Multi-Agent Orchestration section and [references/ultracode-fanout.md](references/ultracode-fanout.md).
    - Fan-out feeds a gate; it never replaces one. Every user-approval gate stays — 1.5 and 3.2 as hard gates, 2.4 as the conditional card gate (auto-proceed on routine cards; pause on sensitive/complex ones) — and `feature-dev:code-reviewer` stays the single authoritative review gate — parallel reviewers add breadth only, aggregated conservatively (PASS only if every dimension passes; any FAIL = FAIL with the union of findings), with no finding ever dismissed or dropped (this extends Rule 15).
    - Fan-out parallelizes work within a single step only. It never parallelizes the per-task loop across tasks and never creates execution waves — tasks are designed, built, reviewed, and merged one at a time. This holds for two independent reasons, both independent of how much context the model has: (a) **no upfront multi-card decomposition** — projecting several cards ahead designs against a future codebase state that will not exist when they run (the Rule 13 freshness reason), and that projection drift is context-independent; (b) **no concurrent execution** — strictly serial design→build→review→merge keeps each diff reviewable against a known base and bounds blast radius (a risk/reviewability reason). An earlier `task-splitter` design with execution waves and a wave-merge protocol was deliberately removed for these reasons; a larger context window does not revive it.
    - Scale the fan-out to the work (Rule 11): collapse to one agent or skip it for tiny, low-risk work.

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
- `L`: large, allowed only for cohesive low-risk work that reads as one pattern applied many times (see the L allowance)

Never create `XL` cards. Split the requirement further.

**Two kinds of split — keep them straight.** A *volume proxy* splits work that is simply too much to review in one pass; a *blast-radius rule* splits work that is too risky to land in one merge. Volume proxies relax for a cohesive, low-risk card (it reviews as one pattern + N near-identical applications, so it stays approvable in one sitting even when the file count is high); blast-radius rules never relax, because their constraint is rollback safety and coordinated rollout, not how much code fits in one pass.

**The `L` allowance (volume only).** A card may be `L` only when ALL hold:
- (a) one uniform Change Type repeated across resources (CRUD handlers, DTOs, mappers, repository methods) — not mixed work types;
- (b) Risk Level low — additive only; no schema/migration, breaking-contract, security-boundary, or cross-protocol work;
- (c) every repeated artifact carries an anti-stub substance constraint, so volume cannot hide stubs;
- (d) the diff reviews as one pattern + N near-identical applications, so a human can still approve it in one sitting by checking the pattern once.

A card failing any of (a)–(d) stays `XS`/`S`/`M` and is split below. The `L` allowance never licenses risky, mixed, or rollout work into one card.

Splitting heuristics (guidelines, not hard rules — justify exceptions if needed):

- *(volume proxy — relaxable)* More than ~12 new files or ~8 modified files is likely L; split by sub-domain, sub-feature, or artifact type **unless** it meets the L allowance. File count alone no longer forces a split for cohesive low-risk work — the residual limit is one-sitting human review.
- *(volume proxy — relaxable by heterogeneity, not count)* More than ~5 *heterogeneous* use cases/handlers/controllers — different logic, failure modes, or risk — split by functional area. *N uniform* handlers sharing one pattern may stay together as `L` regardless of N. The deciding factor is heterogeneity, not the count.
- *(blast-radius rule — never relax)* Fundamentally different work types (e.g., REST endpoints + WebSocket orchestration, or DbContext + repositories + migration) split along the type boundary regardless of available context — the constraint is coordinated rollout and a clean revert (see Rule 7).
- *(default: keep together)* Formulaic CRUD across many resources (e.g., 6 resources x 4 operations = 24 use cases) is cohesive uniform work — keep it in one `L` card under the allowance by default, since each operation is simple and the diff reads as one pattern x N. Split only when a per-resource difference adds real risk (one resource needs auth others don't, a soft-delete cascade, a tenant-scoping invariant), then split by resource sub-group, not by operation type.

## Task ID Rules

- Always use the prefix `TASK-` followed by a zero-padded two-digit number starting at `01`: `[TASK-01]`, `[TASK-02]`, ..., `[TASK-99]`.
- If there are more than 99 tasks, continue with three digits: `[TASK-100]`, `[TASK-101]`, etc.
- Never use project-specific prefixes like `[PA-01]`, `[AUTH-01]`, or `[DB-01]`. The prefix is always `TASK-`.
- Number tasks sequentially in the order they are created during the per-task loop.

## Anti-Stub Patterns

When a task creates files that are high-risk for stubbing (API routes, data-fetching components, business logic modules), include at least one substance constraint in the Artifacts field of Must-Haves. Good constraints: `min_lines: 30`, `contains: prisma.*.findMany`, `exports: [handleSubmit, validateInput]`. The constraint should match the real work, not just file existence.

Read [references/anti-stub-patterns.md](references/anti-stub-patterns.md) for the full stub indicator reference.

## Edge Case Handling

### User rejects a task card

This applies whether the card paused for approval (a sensitive/complex card) or the user intervened on the non-blocking note of an auto-proceeding card. When the user rejects or requests changes to a task card:

1. Collect the user's feedback.
2. Re-invoke code-architect with the original requirements plus the user's feedback.
3. Present the revised card.
4. If the requirement itself is the issue, offer to defer it (move to Deferred in the ledger, update manifest).

### User injects a new requirement

When the user wants to add work mid-build:

1. Add the new requirement to the manifest with a new `REQ-NN` identifier.
2. Log the addition in the Adjustments Log with rationale.
3. Set status to `pending`.
4. Design a task card for it when it becomes the next logical piece of work.

### Requirements change mid-build

When existing requirements change:

1. Update the requirement text in the manifest.
2. Log the change in the Adjustments Log with before/after and rationale.
3. If a completed task partially addressed the old requirement, note what delta remains.
4. Create new requirements for the delta work if needed.

### Code-reviewer unfixable issues (5 cycles)

When the code reviewer identifies issues that cannot be fixed after 5 cycles:

Present to the user with three options:

1. **Proceed with known debt** — document the issues in the manifest Adjustments Log and continue.
2. **Abandon the task** — remove the worktree and delete the branch (`git worktree remove --force .worktrees/TASK-NN; git branch -D task/TASK-NN`), revert the requirement to `pending`, and redesign. (Force is safe here — the user chose to discard the work.)
3. **Pause for manual fix** — the user fixes the issues manually, then resume the workflow.

### Codex review unavailable or errors

When the Codex second opinion (Step 7) runs but Codex is not installed, not authenticated, or the run errors:

1. Surface the exact message and point the user to `/codex:setup` (it checks the CLI and auth, and can install via `npm install -g @openai/codex`).
2. Record "Codex unavailable — see /codex:setup" in the Review Gate Checklist.
3. Proceed to merge. The advisory second opinion never blocks a task that already passed `feature-dev:code-reviewer`.

### Implementation failure

When the implementation sub-agent fails to complete the task:

1. Collect the failure context (what was attempted, what failed, error messages).
2. Re-invoke code-architect with the failure context to redesign the approach.
3. If the requirement is too large, split it into smaller requirements.
4. If the requirement is blocked by an external factor, defer it and log the blocker.

### Build or test failure on merge

When the merge to main fails build or test verification:

1. Attempt to fix the failure in the worktree.
2. Re-run verification.
3. If the fix fails, present to the user with the failure details and options:
   - Fix manually and continue
   - Abandon the task and redesign
   - Proceed with the failure acknowledged (only for non-critical test failures)

### Merge conflict or leftover git state

When `git merge --squash` reports a conflict, or Step 5 finds a pre-existing `.worktrees/TASK-NN` or `task/TASK-NN` (a crashed or abandoned prior run):

1. **Conflict:** do not commit. Run `git merge --abort`, then present the conflicting files and options: resolve manually and continue, or abandon the task and redesign. A conflict only arises when `main` advanced out-of-band during the task (serial execution otherwise prevents it).
2. **Leftover worktree/branch:** run `git worktree prune`; if state remains, ask the user whether to resume the existing worktree (it may hold uncommitted work from the crash) or discard it (`git worktree remove --force .worktrees/TASK-NN; git branch -D task/TASK-NN`) and recreate. Never force-recreate blindly.

### Resuming from an edited or malformed manifest

When invoked with an existing `TASKS/MANIFEST.md` (resume):

1. Confirm it parses and has its required sections (Goal, Requirements with parseable statuses, Decision Ledger). If it is truncated, missing sections, or has unparseable statuses, STOP and present the problem with options: point to a VCS/backup copy, repair the manifest, or re-bootstrap.
2. Reconcile any edits the user made between sessions (requirement text, statuses, locked decisions, deferrals) before selecting the next task — apply the same handling as the mid-build edge cases above.
3. Never overwrite the manifest's existing history.

## Output Delivery

Write the output as a `TASKS/` directory in the project root containing:

- `MANIFEST.md` — the living manifest (updated after every task)
- `TASK-NN.md` — individual task cards (created as each task is designed)

**Bootstrap vs. resume (decided at invocation start):** if `TASKS/MANIFEST.md` already exists (e.g., you were invoked with `@TASKS/MANIFEST.md`), RESUME from it — do **not** re-run Phase 1 bootstrap, and never overwrite the manifest's history. First confirm the manifest parses and has its required sections (Goal, Requirements with parseable statuses, Decision Ledger); if it is truncated, missing required sections, or otherwise malformed, STOP and present the problem to the user (point to a VCS/backup version, repair, or re-bootstrap) rather than resuming against an incomplete document. Reconcile any edits the user made between sessions (see Edge Cases → "Resuming from an edited or malformed manifest"), then continue from the manifest's state: select the next pending requirement (Phase 2 Step 1), or proceed to Phase 3 if none remain. Only when no `MANIFEST.md` exists do you bootstrap a fresh one (Phase 1). If a `TASKS/` directory exists without a manifest, confirm with the user before overwriting.

The manifest is a living document. It starts with requirements only (Phase 1) and grows as tasks are designed, implemented, and completed (Phase 2). By the end, it provides a complete record of what was built, what changed, and what remains.

Read [references/manifest-template.md](references/manifest-template.md) for the manifest format.
Read [references/task-card-template.md](references/task-card-template.md) for the task card format.

## Final Quality Bar

### Per-task checks (before presenting each card)

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

- every requirement has a status (`pending`, `in-progress (TASK-NN)`, `done (TASK-NN)`, or `deferred`)
- completed tasks have summaries of what was built
- adjustments are logged with rationale
- remaining work accurately reflects what is left
- follow-up items contain only explicitly optional or post-launch work
- success criteria are tracked against completed tasks
- the manifest provides a complete record usable by someone joining the project

Return the result as ready-to-execute task files. Do not write implementation code during the planning and card design phases.
