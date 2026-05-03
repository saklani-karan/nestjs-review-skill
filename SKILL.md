---
name: nestjs-pr-review
description: Backend PR and system review rubric for NestJS/TypeScript codebases. Scores code against 12 weighted rules covering transactional management, repository/service architecture, abstract service patterns with Impl naming, event emission naming, decorator/strategy/adapter design patterns, domain folder structure, error handling with ErrorTypes casting, request-scoped logging, circular dependencies, and dependency vs composition decisions. Use this skill whenever the user asks to review a pull request, perform a PR review, do a code review, conduct a system review, audit a codebase, check architectural compliance, score code quality, or evaluate backend code. Trigger on phrasings like "review this PR", "audit my code", "is this structured right", "check this against my standards", "score this", or whenever a NestJS/TypeScript diff, file, or module is shared for evaluation. Always consult this skill before giving a verdict on backend code quality — even if the user phrases the request casually.
---

# Backend PR & System Review

This skill defines a 12-rule rubric for reviewing NestJS/TypeScript backend code. Each rule has a penalty for violations and a reward for correct implementation. The net score plus any critical-rule violations determine the merge decision.

## How to use this skill

1. **Walk every rule** against the diff/files under review. The Rule Index below shows where each rule's detailed reference lives.
2. **Read the matching reference file** before scoring a rule — the references contain detection patterns, exemplars, edge cases, and fix templates. Don't score from memory.
3. **Apply penalty per occurrence** where the rule says "per occurrence" (#1, #2). Apply once per module/file otherwise.
4. **Apply reward once per rule** when the rule is correctly implemented across the diff.
5. **Produce the structured output** in the format below.

## Scoring rubric (quick reference)

| #   | Rule                      |      Penalty | Reward | Tier     |
| --- | ------------------------- | -----------: | -----: | -------- |
| 1   | Transactional management  | -1000 / occ. |    +10 | Critical |
| 2   | Repository definition     |  -100 / occ. |    +50 | High     |
| 3   | Service abstraction       |         -100 |   +100 | High     |
| 4   | Event emission pattern    |          -50 |   +100 | Medium   |
| 5   | Decorator design pattern  |          -50 |   +100 | Medium   |
| 6   | Folder structure          |         -200 |    +50 | High     |
| 7   | Strategy design pattern   |          -50 |    +50 | Medium   |
| 8   | Adapter design pattern    |          -50 |    +50 | Medium   |
| 9   | Error handling            |         -200 |    +50 | High     |
| 10  | Logging                   |         -200 |    +50 | High     |
| 11  | Circular dependencies     |         -500 |    +50 | Critical |
| 12  | Dependency vs Composition |          -50 |    +50 | Medium   |

**Critical rules block the merge regardless of net score:** #1, #11.

## Rule index

Open the matching reference file when scoring a rule. Each reference is self-contained — read it cold and apply.

| #   | Rule                                 | Reference file                               |
| --- | ------------------------------------ | -------------------------------------------- |
| 1   | Transactional management             | `references/01-transactional-management.md`  |
| 2   | Repository definition                | `references/02-repository-definition.md`     |
| 3   | Service definition (Abstract + Impl) | `references/03-service-definition.md`        |
| 4   | Event emission                       | `references/04-event-emission.md`            |
| 5   | Decorator design pattern             | `references/05-decorator-pattern.md`         |
| 6   | Folder structure                     | `references/06-folder-structure.md`          |
| 7   | Strategy design pattern              | `references/07-strategy-pattern.md`          |
| 8   | Adapter design pattern               | `references/08-adapter-pattern.md`           |
| 9   | Error handling                       | `references/09-error-handling.md`            |
| 10  | Logging                              | `references/10-logging.md`                   |
| 11  | Circular dependencies                | `references/11-circular-dependencies.md`     |
| 12  | Dependency vs Composition            | `references/12-dependency-vs-composition.md` |

## Review workflow

### Step 1 — Inventory the diff

Identify the modules, services, repositories, controllers, and module files touched. Note any newly added files; folder structure (rule #6) applies most strictly to new domains. If the diff is partial (e.g. service file but no module file), note what context is missing — you'll skip rules that need that context rather than guessing.

### Step 2 — Walk rules in order

Run through rules 1 → 12. For each rule:

- Open the matching reference file.
- Apply detection patterns to the diff.
- Record findings with `file:line`, rule number, and penalty/reward delta.

Stop and call out **critical violations** (rules #1, #11) immediately — they block the merge no matter what else looks good.

### Step 3 — Score and decide

- Sum penalties and rewards.
- Determine recommendation:
    - Net ≥ 0 **and** zero critical violations → **approve**.
    - Otherwise → **request changes**.
- Critical violations always force "request changes" even if the net score is positive.

### Step 4 — Produce output

Use the format below.

## Output format

```
## Findings
<file>:<line> — Rule #<n>: <short violation summary>. <delta>
<file>:<line> — Rule #<n>: <short reward reason>. +<reward>
...

## Score
Penalties: -<sum>
Rewards:   +<sum>
Net:       <net>
Critical violations: <count> (rules #<list>)

## Recommendation
<approve | request changes | request changes — critical violation>

<one-paragraph summary of the most important things to fix, ordered by severity>
```

### Example

```
## Findings
- src/user/services/user.service.impl.ts:42 — Rule #2: createQueryBuilder used in service. -100
- src/user/services/user.service.impl.ts:88 — Rule #9: raw Error('not found') thrown without ErrorTypes cast. -200
- src/user/user.module.ts — Rule #3 reward: abstract UserService + UserServiceImpl + correct DI registration. +100
- src/order/services/order.service.impl.ts:12 — Rule #1: writeOrder called inside @Transactional createOrder is not itself @Transactional. -1000 (CRITICAL)

## Score
Penalties: -1300
Rewards:   +100
Net:       -1200
Critical violations: 1 (rule #1)

## Recommendation
Request changes — critical transactional violation must be fixed before merge. Order.writeOrder needs @Transactional or must be called via the same transaction context as its caller. Also fix the createQueryBuilder leakage in UserServiceImpl and the raw Error throw before re-review.
```

## When you don't have enough context

If the diff is missing the surrounding module file, the abstract service class, or the repository implementation, **say so explicitly** in the output and skip the affected rule rather than guessing. A "cannot evaluate" finding is more useful than a wrong score.

## Notes on the reward model

- Rewards are awarded **once per rule per review**, not per file. A module that does rule #3 correctly across 5 services gets +100 once.
- Rule #1's reward is intentionally tiny (+10) because correct transactional management is table stakes — you don't get points for not breaking the database.
- Rules #3, #4, #5 have rewards equal to or greater than their penalties, reflecting that the patterns are the "right way" and Karan wants to encourage them.
