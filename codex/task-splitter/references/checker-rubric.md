# Checker Rubric

Run this review before returning final cards. If the first pass fails, revise and re-check.

## Review Dimensions

### 1. Decision Compliance

Check:

- do cards honor locked decisions
- do cards avoid deferred items
- are discretion areas used only where appropriate

Fail when:

- a required card contradicts an explicit user choice
- deferred scope appears in required cards

### 2. Requirement Coverage

Check:

- every committed requirement, behavior, or failed truth maps to task IDs
- no critical source item is unmapped
- overloaded tasks are surfaced

Fail when:

- required source items are missing from the coverage map
- one vague card is pretending to cover several distinct requirements

### 3. Dependency Correctness

Check:

- dependencies are forward-only
- execution waves make sense
- parallel work is only parallel when it is genuinely safe

Fail when:

- the graph is cyclic
- a card depends on work that lands later
- "parallel" cards clearly share a risky mutation boundary without acknowledgment

### 4. Scope Sanity

Check:

- every card fits one focused PR
- work is split by risk and deployability, not arbitrary architecture buckets
- the plan is not over-fragmented for a tiny change

Fail when:

- a card is clearly too broad for one reviewable implementation pass
- the decomposition explodes a simple change into ceremony

### 5. Must-Have Completeness

Check:

- implementation cards include truths, artifacts, and key links
- truths are behavior- or invariant-oriented
- key links are meaningful rather than generic

Fail when:

- cards list artifacts without meaningful truths
- wiring requirements are missing from integration-heavy work

### 6. Rollout Safety

Check:

- risky cards include landing shape and rollback thinking
- migration-heavy work is sequenced safely
- contract changes call out compatibility clearly

Fail when:

- destructive cleanup lands too early
- breaking or coordinated rollout work is hidden inside a generic task

### 7. Verification Realism

Check:

- commands are concrete when possible
- prerequisites are listed when needed
- manual verification is called out only when automation is insufficient

Fail when:

- verification is vague
- commands are invented without evidence and not labeled
- prerequisites are missing for known services or environments

## Revision Loop

Run at least one checker pass mentally before returning output.

If problems remain:

1. revise the ledger if the issue is intent-related
2. revise card boundaries if the issue is coverage, risk, or scope
3. revise must-haves if the issue is completeness or wiring
4. revise verification and rollout notes if the issue is safety or realism

Do not return first-draft cards if the rubric still exposes structural weaknesses.

## Final Pass Standard

The result is ready only when:

- another agent could execute any single card in a fresh thread
- the coverage map explains why every card exists
- no critical requirement or failed truth is dangling
- the riskiest changes land in the safest sequence
- the result is compact enough to be useful
