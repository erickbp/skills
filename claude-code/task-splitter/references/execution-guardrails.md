# Execution Guardrails

These three default guardrails apply to any agent executing a task card. Most task cards use these as-is (marked `Execution Guardrails: Standard` in the card). Override with task-specific guardrails only for high-risk tasks.

## 1. Analysis Paralysis Guard

If an executing agent makes 5+ consecutive read/search operations without writing code, it should stop and either write code (it has enough context) or report that it is blocked with the specific missing information. Continued reading without action is a stuck signal.

## 2. Deviation Classification

While executing, the agent will discover work not in the card. Apply automatically: auto-fix bugs, missing critical functionality, and blocking dependencies without asking. Stop and ask for architectural changes (new database tables, switching libraries, breaking API contracts). After 3 failed auto-fix attempts on a single issue, stop and report — do not keep retrying.

## 3. Self-Check Before Completion

Before marking a task done, the executing agent must verify its own claims: do created files exist? do they contain expected content? do verification commands pass? Do not declare completion without running the verification commands listed in the card.
