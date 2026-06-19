# Decision Ledger

Build the `Decision Ledger` during bootstrap, before the first task is designed.

## Purpose

The ledger separates what is fixed from what is flexible so task design does not silently change product intent.

## Scale to the Task Size

Match the ledger's length to the work. A large or risky change earns a full four-bucket ledger; a small one does not.

- **XS / S work:** keep it minimal. Keep all four buckets present (the four-bucket structure is required), but make their content terse — a couple of short bullets where there is real content, and `None` for a bucket that is genuinely empty. Brevity comes from terse content, not from dropping buckets.
- **An implementation detail is not an `Open Question`.** Open Questions are only for decisions that genuinely need the user. A technical concern you can decide yourself — e.g. "avoid a flash of unstyled theme on load" — belongs in `Coding Agent's Discretion` as a one-liner, or is simply left to implementation. Never park such notes in `Open Questions`.
- **No speculative questions or scope padding.** Raise an `Open Question` only when a genuine user-judgment decision actually blocks the plan; if a safe default exists for a low-impact ambiguity, record a one-line assumption instead (see Question Strategy). Do not enumerate out-of-scope categories nobody proposed, and do not write multi-clause analysis weighing options the change does not warrant.
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

Use only for:

- missing user judgment that materially changes the plan
- contradictions in the input
- external constraints the repo cannot resolve

Rules:

- ask questions only for user-judgment gaps
- do not ask about repo facts that tools can verify
- keep questions specific and high-leverage
- if a safe default exists and the ambiguity is not high impact, record an assumption instead

Examples:

- whether to support both old and new payload formats during rollout
- whether a behavior change is allowed to be user-visible
- whether a risky cleanup belongs in this delivery or later

## Question Strategy

Ask only when the answer materially changes:

- card boundaries
- compatibility guarantees
- rollout sequencing
- dependency choice
- user-facing behavior

Prefer one focused round of questions over many small rounds.

Good question:

- "Should this rollout preserve both old and new webhook payloads for one release, or can producers and consumers switch together?"

Bad question:

- "Which file holds the webhook handler?" if the repo can answer it.

## Validation Rules

Before finalizing cards, verify:

- every locked decision that matters appears in at least one relevant card
- no required card implements a deferred item
- unresolved open questions are explicit
- the plan does not silently depend on an unanswered high-impact question

If the ledger is weak, the task design is weak. Revise it before proceeding.
