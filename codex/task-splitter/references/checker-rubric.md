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
- every data source, queue, cache, table, or service a task reads from either exists in the repo, is created by a predecessor task, or is created by the task itself

Fail when:

- the graph is cyclic
- a card depends on work that lands later
- "parallel" cards clearly share a risky mutation boundary without acknowledgment
- a task creates a component that reads from or writes to infrastructure that no predecessor task creates and is not created by the task itself (likely underspecified)

### 4. Scope Sanity

Check:

- every card fits one focused PR
- work is split by risk and deployability, not arbitrary architecture buckets
- the plan is not over-fragmented for a tiny change

Fail when:

- a card is clearly too broad for one reviewable implementation pass
- the decomposition explodes a simple change into ceremony

Also check:

- no task exceeds the sizing guidelines (~12 new files, ~5 independent use cases/handlers, ~8 modified files) without an explicit justification in Notes / Risks
- formulaic work (CRUD for N resource types) is counted by total operations, not just by "it's all CRUD so it's one task"

Fail when:

- a task exceeds sizing thresholds and the Notes / Risks section does not justify the exception
- a task's Notes/Risks self-identifies exceeding a sizing guideline (e.g., acknowledges '~15–18 new files') but the checker neither splits the task nor records an explicit exemption with rationale in the Checker Summary

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

### 8. Scope Fidelity

Check:

- each task that maps to a source requirement delivers its full contribution to that requirement, not a reduced shell
- no task Goal or In Scope contains scope-reducing language ("initial structure", "not full flow", "stub", "shell", "placeholder", "skeleton") for source-required behavior
- Follow-up Tasks contain only items the source explicitly marks as optional, post-launch, or cleanup
- no required behavior was silently moved to Follow-up Tasks

Fail when:

- a task claims to cover a requirement but its Goal or In Scope describes only partial delivery of what the source requires
- Follow-up Tasks contain behavior the source treats as required scope

### 9. Parallel Wave Merge Safety

Check:

- parallel tasks in the same wave touch different files or clearly disjoint directories
- when overlap is unavoidable, the merge risk is acknowledged in the wave description
- Patterns to Follow and Predecessor Outputs reference only files that exist before the current wave starts — never same-wave task outputs

Fail when:

- 3+ parallel tasks create or modify files in the same directory without acknowledgment
- the Wave Merge Protocol doesn't account for likely conflicts in overlapping directories
- a task's Patterns to Follow or Predecessor Outputs reference files created by a same-wave parallel task
- a same-wave reference uses a conditional fallback ("if not available, use X") — conditional fallbacks do not legitimize same-wave references because the card's quality still depends on execution order within a parallel wave. If a parallel task would benefit from following another parallel task's pattern, describe the pattern inline in both cards or reference a shared existing file instead
- two or more parallel tasks would both create the same file — the decomposition must eliminate the collision by assigning the file to one task, extracting it into a predecessor task, or merging the overlapping tasks. Acknowledging a known collision in wave notes is not sufficient

### 10. Test Placement

Check:

- unit tests for a layer appear in the same wave or the immediately following wave
- integration and e2e tests may appear later but not more than 2 waves after their subject

Fail when:

- all test cards are in the final wave and earlier waves produced testable code
- a layer is implemented in Wave N but its unit tests don't appear until Wave N+3 or later

### 11. Cross-Task Artifact Traceability

Check:

- for every type, interface, file, or artifact referenced in a task's Must-Haves (Truths, Artifacts, Key Links), In Scope, or Constraints, verify that it is produced by the task itself, an explicit artifact of a completed predecessor task, or an existing repo file
- predecessor tasks that are cited as the source of a type or interface actually define that type or interface in their Artifacts or In Scope
- for each Predecessor Outputs citation, confirm the cited artifact literally appears in the cited task's Artifacts or In Scope — not just that the task exists in the dependency chain

Fail when:

- a task references a type, interface, value object, or DTO that no predecessor task produces and that does not exist in the repo
- the dependency graph is structurally correct (TASK-07 depends on TASK-02) but the content contract is broken (TASK-02 never defines the artifact TASK-07 needs)

### 12. Architectural Consistency

Check:

- when multiple tasks implement the same kind of artifact (e.g., controllers, services, repositories), they follow the same architectural pattern
- if one controller delegates to use cases and another bypasses them for direct repository access, the divergence is intentional and justified

Fail when:

- similar tasks in the same plan follow divergent architectural patterns without explicit justification in the decision ledger
- the divergence would create maintenance confusion (e.g., half the API uses a use-case layer, half injects repositories directly into controllers)

### 13. Test Coverage Completeness

Check:

- every implementation task's primary artifacts map to at least one test task somewhere in the plan
- adapter and integration boundary classes may be covered by integration tests rather than unit tests, but must be covered somewhere

Fail when:

- an implementation task produces production artifacts that no test task in the plan covers
- a Non-Goals entry in a test task explicitly excludes testing artifacts from a predecessor and no other test task picks them up

## Revision Loop

Run at least one checker pass mentally before returning output.

If problems remain:

1. revise the ledger if the issue is intent-related
2. revise card boundaries if the issue is coverage, risk, or scope
3. revise must-haves if the issue is completeness or wiring
4. revise verification and rollout notes if the issue is safety or realism
5. revise task Goal/In Scope or move Follow-up items back to required cards if the issue is scope fidelity
6. revise predecessor task artifacts if the issue is cross-task traceability (add missing types/interfaces to the producing task)
7. revise architectural patterns if the issue is consistency (align divergent tasks or justify divergence in the ledger)
8. add or expand test tasks if the issue is test coverage completeness

Do not return first-draft cards if the rubric still exposes structural weaknesses.

### Large-Plan Skepticism

A first-pass-clean result on a plan with 15+ tasks is uncommon. If the checker reports no revisions on a large plan, re-examine dimensions 4 (Scope Sanity), 5 (Must-Have Completeness), and 8 (Scope Fidelity) with extra scrutiny — these are where large plans most commonly have issues that pass a cursory check.

For plans with 20+ tasks, the checker must perform a documented consolidation pass: identify any two same-wave tasks that touch overlapping subsystems and evaluate whether merging them (while staying within sizing limits) would reduce wave count or dependency complexity. Record the consolidation analysis in the Checker Summary even if no consolidation is made.

## Final Pass Standard

The result is ready only when:

- another agent could execute any single card in a fresh thread
- the coverage map explains why every card exists
- no critical requirement or failed truth is dangling
- the riskiest changes land in the safest sequence
- the result is compact enough to be useful
