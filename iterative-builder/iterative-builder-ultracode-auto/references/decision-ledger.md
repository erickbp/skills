# Decision Ledger

Build the `Decision Ledger` during bootstrap, before the first task is designed.

## Purpose

The ledger separates what is fixed from what is flexible so task design does not silently change product intent.

## Scale to the Task Size

Match the ledger's length to the work. A large or risky change earns a full four-bucket ledger; a small one does not.

- **XS / S work:** keep it minimal. Keep all four buckets present (the four-bucket structure is required), but make their content terse — a couple of short bullets where there is real content, and `None` for a bucket that is genuinely empty. Brevity comes from terse content, not from dropping buckets.
- **An implementation detail is not an `Open Question`.** Open Questions are only for genuine user-judgment decisions (which autonomous mode resolves as logged assumptions, not questions). A technical concern you can decide yourself — e.g. "avoid a flash of unstyled theme on load" — belongs in `Coding Agent's Discretion` as a one-liner, or is simply left to implementation. Never park such notes in `Open Questions`.
- **No speculative open questions or scope padding.** Record an `Open Question` only when a genuine user-judgment decision is at stake (resolved with a logged assumption); for a low-impact ambiguity, just record a one-line assumption (see Assumption Strategy). Do not enumerate out-of-scope categories nobody proposed, and do not write multi-clause analysis weighing options the change does not warrant.
- The four buckets are a checklist of what to consider, not a quota to fill with invented content. On a small task, expect most buckets to be `None` or a single bullet — usually just a few Locked Decisions and maybe one Discretion note, with Deferred and Open Questions often `None`.

This mirrors Rule 11 ("keep the result compact"): for small work the ledger stays terse but still present.

## Required Sections

### `Locked Decisions`

Use for:

- explicit user choices
- hard constraints from the input
- dependency choices the user or repo already committed to
- rollout, compatibility, security, or compliance requirements that must be preserved

Rules:

- treat as binding
- propagate only the decisions that actually constrain a card
- if a locked decision changes task boundaries, split accordingly

Examples:

- "Keep the public API backward-compatible."
- "Use the existing Stripe integration; do not switch providers."
- "No dual-write period; rollout must be gated behind a feature flag."

### `Coding Agent's Discretion`

Use for:

- implementation choices the user has not fixed
- repo-grounded choices that can safely be made during decomposition
- details that affect structure but do not affect product intent

Rules:

- only include real freedom areas
- do not hide unresolved user-judgment issues here
- do not use this section to override locked decisions

Examples:

- exact task grouping when multiple safe splits are possible
- whether test hardening is folded into one card or separated
- whether UI wiring and analytics wiring belong in the same focused card

### `Deferred / Out of Scope`

Use for:

- explicitly deferred features
- ideas mentioned but not committed
- cleanup or follow-up items that should not distort required task boundaries

Rules:

- never plan these into required cards
- move nice-to-have work into `Follow-up Items`
- if a deferred item is required for safety, call out the conflict as an open question

Examples:

- "Admin dashboard later."
- "Cleanup old endpoint after rollout stabilizes."
- "Nice-to-have observability dashboard improvements."

### `Open Questions`

This bucket records genuine user-judgment gaps. In autonomous mode the skill does not ask — it resolves each gap with an explicit assumption and logs it here (alongside the question) so a human can review or override it later.

Use only for:

- missing user judgment that materially changes the plan
- contradictions in the input
- external constraints the repo cannot resolve

Rules:

- record gaps only for user-judgment matters, never for repo facts that tools can verify
- for each gap, write the chosen assumption and a one-line rationale: `<gap> — assumed: <choice> (<why>)`
- flag high-impact assumptions clearly (e.g. prefix `[high-impact]`) so they stand out for human audit
- never leave a gap unresolved waiting for a user, and never let the plan silently depend on an unrecorded judgment call
- a product-judgment gap is never a reason to halt; halt only for credentials, an external/cloud check, or a risky/destructive action (SKILL.md → Autonomous Operation)

Examples:

- support both old and new payload formats during rollout — assumed: support both for one release (safest for consumers)
- whether a behavior change may be user-visible — assumed: keep it invisible (no stated UX change)
- `[high-impact]` whether a risky cleanup belongs in this delivery — assumed: defer to a follow-up (avoids coupling cleanup to the feature)

## Assumption Strategy

Record an explicit assumption (never a question) when a judgment call materially changes:

- card boundaries
- compatibility guarantees
- rollout sequencing
- dependency choice
- user-facing behavior

Choose the most reasonable, lowest-risk interpretation, write it with a one-line rationale, and proceed. Resolve repo facts with tools (Rule 1) — they are never assumptions.

Good assumption:

- "Rollout preserves both old and new webhook payloads for one release — assumed because switching producers and consumers together is riskier and the input did not require it."

Not an assumption (verify instead):

- "Which file holds the webhook handler?" — the repo answers this; find it with tools rather than assuming.

## Validation Rules

Before finalizing cards, verify:

- every locked decision that matters appears in at least one relevant card
- no required card implements a deferred item
- every judgment-gap is resolved by an explicit, logged assumption (none left waiting for a user)
- the plan does not silently depend on an unrecorded judgment call; high-impact assumptions are flagged for audit

If the ledger is weak, the task design is weak. Revise it before proceeding.
