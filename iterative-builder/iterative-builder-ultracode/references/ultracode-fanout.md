# Ultracode Multi-Agent Fan-Out

Several iterative-builder-ultracode steps are read-heavy or verify-heavy: they get better when several agents work in parallel — covering more angles, or checking each other's work. This reference defines how those steps fan out. The skill applies it automatically at the steps listed below; do not ask the user first.

Two mechanisms do the work, and they are NOT the same thing. The **default is foreground sub-agents** — inline `Agent` calls in a single message; this is always available and needs no opt-in. The **escalation is the Workflow tool** (true background "ultracode"), reserved for the three heavy steps (2.7, 1.1, 3.1) when the thresholds below are met; it runs asynchronously, and invoking this skill is itself the opt-in to call it. The other four steps (1.2, 1.3, 2.2, 2.3) stay foreground by default — 1.2 never escalates; 1.3, 2.2, and 2.3 only in the rare exceptions noted at each step.

Fan-out **augments** the workflow — it never weakens it. Every user-approval gate stays, `feature-dev:code-reviewer` stays the single authoritative review gate, the optional Codex second opinion stays advisory, and the hard context reset after each task stays. Read the Invariants section before changing any recipe.

## Steps that fan out

| Step | Phase | Pattern |
|------|-------|---------|
| 1.1 Ground the goal and the repo | Bootstrap | parallel multi-angle repo sweep |
| 1.2 Build a Decision Ledger | Bootstrap | completeness-critic panel |
| 1.3 Extract requirements & success criteria | Bootstrap | multi-lens extractors + completeness critic |
| 2.2 Design the task card | Per-task loop | architect judge-panel |
| 2.3 Quality-check the card | Per-task loop | adversarial critic fan-out |
| 2.7 Review | Per-task loop | parallel per-dimension reviewers |
| 3.1 Check success criteria | Validation | per-criterion verifier fan-out |

No other step fans out. The user-approval gates (1.5, 2.4, 3.2), worktree/merge mechanics (2.5, 2.8), manifest writes (1.4, 2.9, 3.3), implementation (2.6), and the context reset (2.10) stay single-threaded.

## Mechanism

- **Default — inline parallel sub-agents (no opt-in needed).** Launch the step's agents in a SINGLE message so they run concurrently in the FOREGROUND (the same pattern feature-dev and `/code-review` use). Foreground keeps the result synchronous so it flows straight into the next gate in the same turn. This is the default for all seven steps.
- **Escalation — the Workflow tool (true background ultracode).** On the three heavy steps, **call the Workflow tool** when its concrete threshold is met: Step 2.7 Review when the diff spans more than ~10 files or ~400 changed lines (or the reviewers are slow inline); Step 1.1 when the repo exceeds ~1,000 tracked source files or is a monorepo with 3+ packages/workspaces; Step 3.1 when there are more than ~8 success criteria or per-SC verification needs slow builds/test runs. The Workflow tool runs asynchronously — the call returns a task id and the result arrives later via a notification; wait for completion, then aggregate. Results return to the SAME gate with the SAME aggregation rule, must finish **within the same task** (before the next step, never across the Step 10 reset), and invoking this skill is itself the opt-in to call it. Below the thresholds — and on steps 1.2, 1.3, 2.2, 2.3 by default — stay foreground.
- **Scale to the work (Rule 11 — keep compact).** Match the number of agents to risk and surface area. A tiny or low-risk step collapses to one or two agents, or skips fan-out entirely. Never add ceremony a small change does not warrant.

## Invariants (non-negotiable)

- **Fan-out feeds a gate; it never replaces one.** The user-approval gates remain: 1.5 manifest approval, 2.4 task-card approval, 3.2 gap decision. Parallel agents widen coverage and improve phrasing; the user still decides.
- **`feature-dev:code-reviewer` stays the single authoritative review gate (Rule 15).** Parallel reviewers only add breadth. Aggregate conservatively: aggregate PASS only if EVERY dimension PASSes; ANY FAIL is an aggregate FAIL whose issue list is the UNION of all findings. Never dismiss, downgrade, or drop a finding — including dropping one dimension's finding because another agent disagrees. Never add a verifier/triage agent that filters reviewer findings.
- **Codex stays advisory (Rule 16).** Optional, once per task, after the first aggregate PASS, never a gate.
- **The hard context reset stays (Rule 14).** A background fan-out is confined to the current step of the current task and must be aggregated before the step that follows; it never runs across the reset.
- **Repo facts stay tool-verified (Rule 1).** Every agent cites tool evidence (path + matched line) for every fact; unverifiable items are labeled `unverified` with a reason, never guessed.
- **Fan-out is within a step, never across tasks.** It never creates execution waves or parallel task grouping — tasks are still designed, implemented, reviewed, and merged one at a time (Rule 14). Only the work inside a single step is parallelized.

---

## Step 1.1 — Ground the goal and the repo (parallel multi-angle sweep)

**When to fan out:** Always, for the repo-exploration sub-step (Step 1, item 2) — apply it automatically, do not ask the user. Scale to the repo (Rule 11): for a tiny or greenfield repo, collapse to 1-2 agents or skip fan-out entirely; for a large/monorepo, use the full set and escalate to the Workflow tool per the threshold below.

**Mechanism:**
- *Default* — launch all angle-readers as inline sub-agents in a SINGLE message, in the FOREGROUND, so the synthesis happens in this same turn and flows straight into Step 4 (write manifest) and the Step 1.5 approval gate.
- *Large-repo escalation* — when the repo exceeds ~1,000 tracked source files or is a monorepo with 3+ packages/workspaces (a foreground sweep would be slow), use the Workflow tool (true ultracode) to fan out and verify in the background. Wait for it to complete, then return all angle reports to the same synthesis point. Same gate, same invariants.

**Parallel agent roles (map 1:1 to Step 1's grounding list and the manifest's Discovered Facts):**
- **(a) Topology + build/CI + dependencies** — repo shape (monolith/service/monorepo/package/app), build system, CI pipeline, deploy process, package manager, key libraries + version constraints. Cites: root manifests (package.json / *.csproj / pyproject.toml / go.mod), CI configs, lockfiles.
- **(b) Data / persistence layer** — ORM, migration framework, schema/entity definitions, database type. Cites: schema files, migration dirs, entity/model files.
- **(c) API / service layer** — framework, routing conventions, middleware stack, service boundaries. Cites: route/controller files, middleware registration, DI/startup wiring.
- **(d) Test infrastructure + conventions** — test framework, config, directory layout, naming conventions, how to run the suite. Cites: test configs, an example test file, test scripts.
- **(e) Existing similar feature(s)** — read 1-2 representative features closest to the product goal end-to-end to capture real conventions the new work must match. Cites: the concrete files that make up those features.

**Per-agent contract (enforces Rule 1):**
- Verify with Glob/Grep/Read before naming any file, module, table, endpoint, command, or convention. Every reported fact carries its tool evidence (path + matched line).
- Return a tight structure: `Verified facts` (fact → evidence), `Key files` (paths the orchestrator should read itself), `Unverified` (needs runtime/external/staging — say why), `Possible product-intent gap` (questions the repo cannot answer).
- Do NOT invent facts, do NOT decide product intent, do NOT propose requirements or task cards. Read-only grounding only — no code is written (Rule 12).

**Aggregation rule (conservative union):**
- Orchestrator reads the `Key files` each agent surfaced, then writes ONE Discovered Facts set in the manifest's shape (topology / relevant paths / data layer / test setup / build-CI / key conventions / other).
- A fact enters Discovered Facts only with tool evidence behind it. Anything an agent could not verify is recorded `unverified` with the reason. For greenfield repos, input-document paths are treated as verified and internal framework-convention paths are labeled `(convention-based)`.
- If two agents conflict on a fact, treat it as unresolved: re-verify with a direct Read rather than picking one — never average or guess.

**Escalation (preserve Step 1, item 4):** Collate every `Possible product-intent gap` across agents into the Decision Ledger's Open Questions and ask the user in one round BEFORE writing the manifest. Fan-out gathers facts; it never answers product intent on the user's behalf.

**Invariant reminder:** This is a bootstrap step — there is no worktree, no code-reviewer, no Codex, no context reset here. The fan-out FEEDS Step 4 and the Step 1.5 manifest-approval gate; it does not replace that gate, and it does not let any agent assert an unverified or intent-level fact.

## Step 1.2 — Decision Ledger: completeness-critic panel

**When:** After you have drafted all four ledger buckets (`Locked Decisions`, `Coding Agent's Discretion`, `Deferred / Out of Scope`, `Open Questions`) in a single pass. The fan-out critiques an existing draft; it never authors the first draft. Apply it automatically, no asking. This step is foreground-only — no Workflow tool, no opt-in needed.

**Mechanism:** Inline parallel sub-agents launched in a SINGLE message, run in the FOREGROUND (so the Step 1.5 user gate still works). This is bootstrap planning — there is no worktree, no `feature-dev:code-reviewer`, and no Codex at this step, so those gates do not apply here. This is a lightweight panel; do not escalate to the Workflow tool for a ledger (reserve true background ultracode for the heavy fan-outs — Step 2.7 Review and Steps 1.1 / 3.1 on large repos).

**Scale to the work (Rule 11 — keep compact):** tiny/low-risk goal → 1 critic or skip entirely; typical goal → 2 critics; large/multi-subsystem goal → 3 critics. Never add ceremony a one-paragraph goal does not warrant.

**Parallel critic roles** (each gets the goal/input, the Discovered Facts from Step 1.1, and the current ledger draft; each works independently and cites tool evidence per Rule 1):
- **Missed-question critic** — re-reads the goal + Discovered Facts hunting for user-judgment gaps the draft left out: contradictions in the input, external constraints the repo cannot resolve, behavior/compat/rollout/dependency choices the user has not actually fixed. Proposes new `Open Questions`. Per the decision-ledger conventions, flags ONLY user-judgment gaps — anything a tool can answer is out of bounds and must instead be verified and recorded as a Discovered Fact, not raised as a question.
- **Mis-bucketing critic** — audits each existing entry for the right bucket: a "decision" with no user mandate that was parked in `Locked` (should be `Coding Agent's Discretion` or an `Open Question`); a genuinely binding constraint hiding in `Discretion`; a `Deferred` item that is actually required for safety (per decision-ledger, surface that conflict as an `Open Question`). Proposes re-bucketings, each with a one-line reason.
- **Hidden-assumption critic** (add only for larger goals) — surfaces decisions the draft made *silently* — load-bearing choices implied by the plan but written down nowhere (e.g., an assumed data format, an assumed auth model, an assumed rollout with no flag). Each becomes either a `Locked Decision` (if the input truly fixed it — cite the line) or an `Open Question` (if it does not). This directly enforces "do not silently guess."

**Aggregation rule (conservative, additive-only):**
- Take the UNION of all critic flags; dedup by the item they concern.
- A critic may ADD an `Open Question` or RE-BUCKET an item; a critic may NOT delete an existing `Open Question`, downgrade a `Locked Decision`, or move required work into `Deferred`. The merge only ever tightens the ledger.
- Apply re-bucketings only when a critic gives a concrete reason; on a genuine conflict between critics (e.g., Locked vs. Open for the same item), keep the MORE conservative placement — `Open Question` over an assumed `Locked`, `Locked` over a permissive `Discretion`.
- Resolve nothing on the user's behalf. Every surviving `Open Question` stays open and rides into the Step 1.5 manifest-approval gate, where the user resolves it. Critics never invent an answer, and the orchestrator never quietly picks a default for a high-impact gap (a safe default for a low-impact ambiguity may be recorded as an explicit assumption, per the decision-ledger rules).

**Note on escalation:** If grounding the ledger keeps colliding with a genuinely huge/unmapped repo, that is a Step 1.1 (grounding) problem, not a ledger problem — let Step 1.1's fan-out handle the heavy lifting (it is the heavy step that escalates to the Workflow tool); keep 1.2 a compact inline panel over the already-discovered facts.

## Step 1.3 — Extract requirements & success criteria (multi-lens fan-out)

**When to fan out:** Always at 1.3, automatically — do not ask the user. **Scale to the input (Rule 11):** for a tiny/single-ask input, run 2 extractors (explicit + implied) or even do it inline with no sub-agents; for a rich multi-feature spec, run all 4 lenses. Never add ceremony.

**Mechanism:** Inline parallel sub-agents launched in a SINGLE message, run in the FOREGROUND. This is a planning step (Rule 12, no implementation code), so the heavyweight Workflow tool is NOT used here — reserve true-background ultracode for the heavy fan-outs (Step 2.7 review; Steps 1.1 / 3.1 on large repos). Foreground keeps results synchronous so the orchestrator can reconcile and hand a single clean set to the 1.5 gate.

**Inputs given to every extractor (identical packet):** the raw input/goal document, the Step 1.1 Discovered Facts (tool-verified repo grounding), and the Step 2 Decision Ledger (so `Deferred / Out of Scope` items are NOT proposed as requirements, and `Locked Decisions` shape SCs).

**Parallel extractor roles (each independent, one lens):**
- **L1 — Explicit asks:** every deliverable/behavior the input literally states. Cite the source line/section per item.
- **L2 — Implied behaviors:** behaviors the explicit asks logically require but don't spell out (e.g., "list X" implies pagination/empty-state; "accept uploads" implies size/type limits). Cite the explicit ask each inference hangs off.
- **L3 — Cross-cutting / non-functional:** security (authn/authz, input validation, secrets), performance/latency, observability (logging, metrics, alerts), migration/rollout/backfill, compliance/data-retention, backward-compat. Honor coverage-and-must-haves.md: annotate NFRs not verifiable by task-level commands as `(design-for)` and flag them for Follow-up performance testing; surface security asks so they can later become Truths, not buried Constraints.
- **L4 — (optional, large inputs) Data/contract surface:** entities, schemas, external contracts/integrations implied by the goal. Skip for small inputs.

Each extractor returns a flat list of candidate items in the form `REQ: <one deliverable/behavior>` and `SC: <observable condition>`, **with a tool/source citation per item** (input quote or Discovered-Facts reference). Extractors do NOT assign final numbers and do NOT prune each other — divergence is desired.

**Orchestrator aggregation (deterministic, conservative):**
1. **Union, then dedup** semantically equivalent candidates across lenses (keep the clearest phrasing; merge citations).
2. **Reconcile atomicity (Rule 11 + step invariant):** split a candidate that bundles clearly separable deliverables; **merge** trivially-coupled fragments so each `REQ` maps to ~one task but stays coarse enough to avoid ceremony. Multi-task requirements are fine — that emerges in the per-task loop.
3. **Enforce SC observability (Rule 9):** rewrite any SC that says "works correctly"/"is production ready" into an externally observable/testable condition; drop SCs that merely restate a REQ with no observable signal.
4. **Assign final sequential numbers** `REQ-01..`, `SC-01..` only after dedup/split/merge settle.
5. **Drop nothing silently:** if a candidate is excluded (e.g., it matches `Deferred / Out of Scope`), note it in the Decision Ledger rather than discarding it.

**Completeness critic (single pass, after reconcile):** one sub-agent reads the reconciled `REQ`/`SC` set plus the input and Discovered Facts and answers only: "what deliverable is implied by the goal but unlisted?" (common gaps: error/empty/edge states, auth on a new endpoint class, migration for a schema change, an observable SC for a stated behavior). It returns **candidate additions only** — it has NO authority to remove or downgrade existing items and is NOT a review gate (it does not stand in for `feature-dev:code-reviewer`, Rule 15). The orchestrator folds accepted additions back through the dedup/number step.

**Invariant: feeds, never replaces, the gate.** The reconciled set + the critic's proposed additions are presented at the 1.5 manifest-approval gate. The user still approves/edits/adds/removes; the fan-out only widens coverage and improves phrasing. This output also feeds Step 4 (manifest write) and downstream Step 1 requirement selection.

**Escalate to the Workflow tool only if:** the input is so large that extractor prompts would exceed comfortable context, or grounding must be re-derived per lens on a large repo. In that case fan out the extractors as background Workflow agents but still return their candidate lists to the SAME orchestrator reconcile + 1.5 gate — the gate and the no-drop aggregation rule are unchanged.

## Step 2.2 — Architect judge-panel (design the task card)

**Goal of the fan-out:** widen the design search so the card that reaches the user is the synthesis of several architectural takes, not one architect's first instinct — without weakening Rule 13, sizing, or the 2.3/2.4 gates. This parallelizes perspectives on a *single* card; it never designs multiple tasks at once (no execution waves — see Invariants).

### Mechanism
- **Default (this step): inline parallel sub-agents in a SINGLE message, foreground.** Launch every `feature-dev:code-architect` lens in one assistant turn so they run concurrently; foreground so the orchestrator blocks on their return and the synthesized card still hits the 2.3 check and the 2.4 user gate in the same flow. This is the same pattern feature-dev / `/code-review` use.
- 2.2 is NOT a heavy-fan-out step. Reserve the Workflow tool (true background ultracode) for 2.7 Review and for 1.1 / 3.1 on large repos. Use it here only if the repo is large enough that parallel full-codebase reads are slow — and even then, the panel must return to the same 2.3 → 2.4 path.

### Scale the panel (Rule 11 — keep compact)
- **XS / clearly low-risk requirement:** 1 architect (no panel). Never add ceremony for a one-file change.
- **S / typical:** 2 architects (minimal-change + pragmatic-balance).
- **M / higher-risk / integration-heavy / multiple requirements:** 3 architects (add clean-architecture).
- Never exceed 3 here; if the work seems to need more lenses, the requirement is probably too big — re-check sizing and consider splitting (Sizing Rules) instead of adding agents.

### Parallel agent roles (lenses)
Give every architect the SAME inputs from Step 2.2 — selected requirement(s) + context, the Decision Ledger, the task-card template, completed-task summaries, coverage-and-must-haves guidance, anti-stub patterns. Each is instructed to read the **current state of main including all merged work** (Rule 13 — never a projected state) and to cite tool evidence (Glob/Grep/Read) for every repo fact it asserts (Rule 1). Each returns ONE complete candidate card in the template format. The only difference is the lens:
- **minimal-change** — smallest correct diff; reuse existing patterns/files; fewest new artifacts. Guards against over-engineering and L-creep.
- **clean-architecture** — proper layering/boundaries and seams; flags where a minimal change would entrench debt. Guards against shortcuts that violate locked architectural decisions.
- **pragmatic-balance** — ships the full required behavior with sane structure; explicitly the anti-stub lens (substance constraints on high-risk artifacts; no "initial structure / stub / placeholder"). Guards against shallow coverage.

### Aggregation rule (orchestrator synthesizes — this is NOT a gate)
The orchestrator (not a sub-agent) scores each candidate against the Step 2.3 dimensions and synthesizes the winner. This selection feeds a gate; it never replaces one.
1. **Disqualify** any candidate that violates a hard rule: invents an unverified repo fact (Rule 1), designs against a projected state (Rule 13), implements deferred/out-of-scope work, breaks a Locked Decision, or sizes to L/XL.
2. **Score** survivors on: requirement coverage (every Addresses REQ fully delivered, no shallow coverage per coverage-and-must-haves.md), anti-stub substance (high-risk Artifacts carry `min_lines`/`contains`/`exports`), sizing fit (lands in XS/S/M), locked-decision adherence, and Must-Haves quality (Truths = observable behavior not steps; Artifacts concrete; Key Links real wiring).
3. **Synthesize ONE card**, taking the highest-scoring candidate as the base and grafting stronger elements from the runners-up (e.g., a tighter substance constraint, a missed Key Link, a better non-goal). Record a one-line "panel note" naming which lens won and what was grafted, for the 2.4 presentation.
4. The synthesized card MUST still fit XS/S/M. If every candidate came back L/XL, do not shrink by deleting required behavior — split the requirement (Sizing Rules) and re-run the relevant per-task steps.

### Invariants preserved
- **Rule 13** — every architect reads the real current codebase; none designs against a projected future state. The synthesis adds no facts beyond what the candidates tool-verified.
- **Rule 1** — each candidate cites tool evidence; disqualify any invented fact rather than carrying it into the synthesis.
- **Gates intact** — the synthesized card is unapproved input: it flows into Step 2.3 quality-check and then the Step 2.4 user-approval gate unchanged. On rejection, the "User rejects a task card" edge case is unchanged (re-run the panel with the user's feedback).
- **Sizing** — cards stay XS/S/M; the panel never licenses an L/XL card.
- **No review authority touched** — this step designs cards; it does not touch `feature-dev:code-reviewer` (Rule 15) or the Codex second opinion (Rule 16).

### When to escalate to the Workflow tool
Only on a large repo where N concurrent full-codebase architect reads are too slow or too heavy for foreground inline agents. Then fan the lenses out via the Workflow tool (background) and have it return all candidates to the orchestrator, which performs the same scoring/synthesis and routes the winner into the same 2.3 → 2.4 path. The escalation changes only where the agents run — never the aggregation rule or the gates.

## Step 2.3 — Quality-check the card (adversarial critic fan-out)

**When this applies.** After `feature-dev:code-architect` returns a card (Step 2.2) and before you present it to the user (Step 2.4). This is a pre-user *planning-quality* gate. It is NOT code review — `feature-dev:code-reviewer` (Step 2.7) remains the single authoritative code-review gate (Rule 15), and nothing here touches it.

**Mechanism.** Default to inline parallel sub-agents launched in a SINGLE message, run in the FOREGROUND so the result lands before the Step 2.4 user gate. These critics are read-only analysts of the card text + repo — they do not edit the card. Heavy fan-out (Workflow tool / background) is unnecessary here; a card is small. Keep it inline.

**The critics (each attacks ONE dimension; each returns `PASS` or `FAIL — <issues>`).** Map each existing checklist bullet to exactly one critic so no check is dropped:

1. **Scope-language critic.** Fail on scope-reducing language in Goal or In Scope ("initial structure", "stub", "placeholder", "skeleton", "shell", "basic scaffold", "not full flow") for source-required behavior. Cross-check the requirement(s) the card Addresses to confirm full behavior is demanded (see references/coverage-and-must-haves.md → "Depth of Coverage"). Cite the offending line.
2. **Stub-risk / substance critic.** For high-risk Artifacts (API routes, data-fetching components, business-logic modules, persistence), require at least one real substance constraint (`min_lines`, `contains: <pattern>`, `exports: [...]`) that matches the work, not just file existence. Use references/anti-stub-patterns.md to name the concrete stub indicator the constraint guards against. Also confirm security requirements (auth/authz/validation) appear as Truths, not buried in Constraints/Notes (coverage-and-must-haves.md → "Security Requirements as Truths").
3. **Must-Haves shape critic.** Confirm Truths, Artifacts, and Key Links are all present; Truths describe observable behavior/invariants, NOT implementation steps; and any Truth that an empty or trivially-wrong file could satisfy is reframed (coverage-and-must-haves.md → "Truths"). Flag missing Key Links on integration-heavy cards.
4. **Verification-executability critic.** Confirm Verification Commands are concrete and executable, prefer stack-native commands, and include a behavioral check (not just build/file-exists). If Artifacts carry substance constraints, require at least one content check validating one. Flag commands that depend on unlisted prerequisites.
5. **Locked-decision & leakage critic.** Confirm every applicable locked decision from the ledger is honored, and that the card does NOT implement deferred or out-of-scope items (check Non-Goals against In Scope/Must-Haves). Cite the ledger entry or the leaking line.
6. **Sizing critic.** Confirm the card fits XS/S/M, not L/XL. Apply the splitting heuristics (~12 new / ~8 modified files; >~5 independent use cases/handlers; mixed work types). If L/XL, FAIL with a proposed split axis.

**Every critic must cite tool-verifiable evidence** (the exact card line, the requirement text, the ledger entry, or a repo path/grep result) — never an unverified assertion (Rule 1).

**Aggregation rule (conservative, mirrors the reviewer gate).** Aggregate PASS only if EVERY critic returns PASS. ANY FAIL = aggregate FAIL. On FAIL, take the UNION of all critics' issues (do not dismiss, downgrade, or de-duplicate away any finding) and re-invoke `feature-dev:code-architect` (loop back to Step 2.2) with that consolidated, specific feedback — preserving the existing "re-invoke code-architect with specific feedback" behavior. Then re-run the critic fan-out on the revised card. There is no separate verifier that filters findings.

**Feeds, never replaces, the user gate.** Aggregate PASS only means the card is ready to PRESENT at Step 2.4; the user still approves, requests changes, or defers. An aggregate FAIL never reaches the user as-is — it loops back to the architect first.

**Scale to the work (Rule 11).** For an XS/low-risk card (e.g., a docs tweak or a one-line config change with no high-risk Artifacts), do not spin up six agents: collapse the dimensions into a single critic pass, or skip the stub-risk and sizing critics that plainly do not apply. Match the breadth of fan-out to the card's risk and surface area; never add ceremony.

**Escalate to the Workflow tool only if** the card is genuinely large/multi-subsystem and the critics each need substantial independent repo digging to judge sizing or stub risk — then fan out via the Workflow tool and return the same PASS/FAIL verdicts to this same pre-user gate. This is rare here; an inline single-message fan-out is the norm for Step 2.3.

## Step 2.7 — Review (parallel reviewer fan-out)

**Why:** A single reviewer pass can miss a dimension. Fanning out a reviewer per dimension widens coverage WITHOUT weakening the gate. This is breadth-only: it feeds the same authoritative gate and obeys Rule 15 in full.

### Default mechanism: inline parallel sub-agents (foreground)
Launch all dimension reviewers in a SINGLE message (same pattern feature-dev / `/code-review` use) so they run concurrently in the foreground and the per-task user gates still work. Each agent is `feature-dev:code-reviewer` (the SAME reviewer subagent — only its scope differs), pointed at the modified files in `.worktrees/TASK-NN`.

### Reviewer roles (one agent each)
Scope each agent to exactly one dimension and tell it to review only that lens but flag anything critical it notices:
1. **Correctness / bugs** — logic errors, edge cases, broken callers after a change.
2. **Security** — input validation, secrets, authz, injection, unsafe deserialization.
3. **Project conventions** — adherence to CLAUDE.md and repo conventions (cite the file/line evidence; Rule 1).
4. **Simplicity / DRY** — duplication, dead code, over-engineering, needless ceremony.
5. **Tests / coverage** — missing or weak tests for the changed behavior; must-haves covered.

Give EVERY agent the existing reviewer instruction verbatim:
> End your review with a verdict line in exactly this format:
> `## Verdict: PASS` if no issues found, or `## Verdict: FAIL — N issue(s)` if issues found.

Each agent must cite tool evidence for its findings (Rule 1) — no invented repo facts.

### Aggregation rule (conservative — read ONLY the verdict lines)
- Read each agent's `## Verdict` line. Do NOT interpret the substance of findings to decide whether they "really" matter (Step 7's existing rule).
- **Aggregate PASS** only if EVERY dimension returns `PASS`.
- **ANY dimension `FAIL`** ⇒ **aggregate FAIL**, and the issue set is the **UNION** of all FAILing agents' findings (de-duplicate identical findings; never drop a unique one).
- **NEVER self-dismiss / downgrade / drop a finding.** The "never self-dismiss" rule (Rule 15) now ALSO means: never drop dimension A's finding because dimension B passed or because another agent disagrees. Disagreement between agents never resolves in favor of the laxer verdict.
- Do NOT add a "verifier"/triage agent that filters or rejects reviewer findings — that would violate Rule 15. There is no authority above the reviewers except a clean re-review or explicit user acceptance.

### Loop, unchanged
On aggregate FAIL: fix EVERY issue in the union (across all dimensions) in the worktree, run build/tests, then re-fan-out the same dimension reviewers to verify. `review_count` still caps at 5; on 5-and-still-FAIL, escalate via Edge Cases → "Code-reviewer unfixable issues" (present the union of unresolved findings). The only two exit conditions are unchanged: aggregate PASS, or explicit user acceptance (quote them).

### Downstream steps, unchanged
The optional Codex second opinion (Rule 16) still runs once per task, only after the FIRST aggregate PASS, only on user opt-in, and never gates the merge. The Review Gate Checklist and the two-exit-conditions rule are unchanged — just **extend the checklist** to record which dimensions were covered, e.g. add:
```
- Dimensions reviewed: [correctness, security, conventions, simplicity, tests]  (note any skipped + why)
```

### Scale to the work (Rule 11 — keep compact)
- Tiny / low-risk diff (e.g., a one-line copy or config change): a single `feature-dev:code-reviewer` (the original behavior) is fine — do not add ceremony.
- Pick only the dimensions the diff actually touches (e.g., docs-only change → conventions + maybe simplicity; skip security/tests and SAY SO in the checklist).
- Medium/large or security-sensitive diff: use the full set above.

### When to escalate to the Workflow tool (true ultracode, background)
For a heavy Step 7 — when the diff spans more than ~10 files or ~400 changed lines, or the reviewers are slow to run inline — fan out and verify via the Workflow tool (background) instead of inline foreground agents. Wait for the background run to complete, then aggregate. Constraints when doing so:
- Results return to THIS SAME review gate; aggregation rule and max-5 loop are identical.
- The per-task user gates and the hard context reset (Step 10 / Rule 14) still apply — the background fan-out is confined to this task's Step 7 and must complete (and be aggregated) before Step 8.
- Still no finding-filtering verifier; the background workflow only collects per-dimension verdicts and returns the union.

## Step 3.1 — Check success criteria (per-criterion verifier fan-out)

**When to fan out:** Always at Step 3.1, applied automatically — do not ask the user. Scale to the work (Rule 11): one verifier per success criterion. For 1–2 simple SCs, a single sequential pass is fine; for more than ~8 SCs or a large/multi-package repo, run the parallel fan-out via the Workflow tool (true ultracode) in the background per the threshold below, returning all results to the SAME gate (Step 3.2). Default mechanism is inline parallel sub-agents launched in ONE message, foreground.

**Inputs every verifier receives (identical except the target SC):**
- The one `SC-NN` it owns (verbatim text from the manifest).
- Read access to the merged integration branch `main` only — at Phase 3 all tasks have merged (Step 8) and the per-task worktrees are gone (Step 8 cleanup). Do NOT point verifiers at `.worktrees/`.
- The manifest's Requirements → done-task mapping and Completed Tasks summaries, so the verifier can find which task(s) claim to address its SC and check the claim against reality.

**Each verifier's job (adversarial, evidence-first):**
1. Identify which completed task(s), if any, claim to satisfy this SC (from `done (TASK-NN)` mappings + Completed Tasks summaries).
2. Adversarially confirm the SC is GENUINELY met, not merely asserted — try to falsify it. Distinguish "a type/handler exists" from "the observable behavior the SC names actually happens" (mirror the shallow-coverage signals in coverage-and-must-haves.md).
3. Gather observable evidence per Rule 9: externally visible behavior, contract guarantee, data invariant, or a testable outcome — cited as `file:line`, a named passing test, or an observed command/endpoint result.
4. Honor Rule 1: if the SC can only be proven by runtime, an external service, or a staging environment that the verifier cannot exercise, return `unverified` with the precise reason rather than guessing `met`.

**Verifier return shape (one per agent):**
- `sc`: the SC id.
- `status`: `met` | `gap` | `unverified`.
- `evidence`: concrete citations (file:line / test name / observed behavior). Required for `met`; for `gap`, cite what is missing or wrong; for `unverified`, state exactly what could not be exercised and why.
- `addressed_by`: task id(s) that contributed, or `none`.

**Aggregation rule (conservative, lossless — the orchestrator does NOT decide):**
- Collect every verifier result verbatim into a single gap list keyed by SC.
- An SC is reported satisfied ONLY when its verifier returns `met` WITH cited observable evidence. No evidence ⇒ treat as a gap.
- Any `gap` becomes a flagged gap. Any `unverified` is also surfaced as a gap/open item (with its reason) — never silently upgraded to met.
- The orchestrator MUST NOT dismiss, downgrade, or merge away a verifier's gap finding. It only assembles the list; the resolution decision belongs to the user at Step 3.2 (create additional tasks → loop back to Phase 2, accept as-is, or defer), and the chosen resolution is recorded per the manifest template's Phase-3 update rules.

**Boundary with the review gate (Rule 15 — do NOT cross it):** SC verifiers answer "is this success criterion genuinely satisfied?", which is a different question from code quality/correctness. They are NOT `feature-dev:code-reviewer` and add no second review gate; they neither re-open nor override any prior reviewer verdict. The single authoritative review gate remains `feature-dev:code-reviewer` from Step 2.7, unchanged.

**Escalate to the Workflow tool when:** there are more than ~8 success criteria, the repo is large/multi-package, or per-SC verification needs builds/test runs that are slow to run serially — fan out and verify in the background, wait for completion, then return the aggregated gap list to Step 3.2's user gate. For a handful of cheap-to-check SCs, keep it to inline foreground sub-agents (or even a single pass) — never add ceremony.
