# Rule 12 — Dependency vs Composition

> **Penalty: -50** · **Reward: +50**

## The rule

Decide explicitly whether a related service is a **dependency** or part of a **composition** — minute details make this difference.

- **Dependency**: the related service can function independently → inject normally.
- **Composition**: the related service cannot exist without its parent → pass the parent **entity/class instance** (not an ID), and have the child service skip re-validation since the caller has already done it.

The canonical example: `Rule` and `Condition`. Conditions cannot exist without rules. So `ConditionService.create(rule, dto)` accepts the validated `Rule` directly; it does not re-validate by calling `RuleService.findById(ruleId)` (which would risk a cycle and duplicate work).

## Why it matters

- **Eliminates accidental cycles** (rule #11): the most common cause of cycles is a child service calling back into its parent for "validation" or "lookup." Pass the entity in and the cycle disappears.
- **Performance**: re-fetching the parent for every child operation hits the DB unnecessarily.
- **Trust boundaries**: the caller knows it has a valid parent. The child trusting the caller is correct, not unsafe.
- **Clarity of ownership**: when the relationship is composition, the parent service is "in charge." The child is implementation detail. Reflecting that in the API ("pass the parent in") makes the ownership obvious.

---

## The decision

Ask: **can the related service be invoked usefully outside the parent's context?**

- **Yes** → it's a dependency. The parent injects it normally and calls it. The child validates its own inputs.
- **No** → it's composition. The parent passes its own entity instance to the child. The child trusts the caller and skips validation.

| Relationship | Type | Pattern |
|--------------|------|---------|
| `User` ↔ `Order` | Dependency | Order references userId; User exists without orders |
| `Rule` ↔ `Condition` | Composition | Conditions only exist for a Rule |
| `Order` ↔ `OrderItem` | Composition | Order items only exist within an order |
| `User` ↔ `Profile` | Composition (usually) | A profile is part of the user, not a separate entity |
| `Organization` ↔ `User` | Dependency | Users can move between orgs; both are independent |
| `Cart` ↔ `CartItem` | Composition | Cart items don't exist without a cart |
| `Article` ↔ `Comment` | Dependency | Articles exist without comments; comments reference articles |
| `Form` ↔ `FormField` | Composition | Fields are part of the form definition |

---

## Pattern A — Dependency

```ts
// user.service.impl.ts
@Injectable()
export class UserServiceImpl extends UserService {
  // standard injection
}

// order.service.impl.ts
@Injectable()
export class OrderServiceImpl extends OrderService {
  constructor(private readonly userService: UserService) { super(); }

  async create(dto: CreateOrderDto): Promise<Order> {
    // validate the dependency exists — it's an independent entity
    const user = await this.userService.findById(dto.userId);
    return this.orderRepo.save({ ...dto, user });
  }
}
```

Notice `OrderService` calls `UserService.findById` to validate. That's correct because `User` is a dependency, not a composed part.

---

## Pattern B — Composition

```ts
// rule.service.ts (abstract) and rule.service.impl.ts
@Injectable()
export class RuleServiceImpl extends RuleService {
  constructor(
    private readonly ruleRepo: RuleRepository,
    private readonly conditionService: ConditionService,
  ) {
    super();
  }

  @Transactional()
  async createWithConditions(dto: CreateRuleDto): Promise<Rule> {
    this.log.log('createWithConditions start');

    // The parent (Rule) is created first.
    const rule = await this.ruleRepo.save({ name: dto.name, /* ... */ });

    // Pass the entity, not the ID. The child service will trust this Rule
    // because the parent has already created and validated it.
    for (const condDto of dto.conditions) {
      await this.conditionService.createForRule(rule, condDto);
    }

    this.log.log('createWithConditions end', { ruleId: rule.id });
    return rule;
  }
}
```

```ts
// condition.service.impl.ts
@Injectable()
export class ConditionServiceImpl extends ConditionService {
  constructor(private readonly conditionRepo: ConditionRepository) { super(); }

  /**
   * Note: accepts the Rule entity, not a ruleId.
   * The caller (RuleService) has already validated the rule's existence.
   * We do NOT call RuleService.findById here — that would create a cycle.
   */
  async createForRule(rule: Rule, dto: CreateConditionDto): Promise<Condition> {
    this.log.log('createForRule start', { ruleId: rule.id });
    const condition = await this.conditionRepo.save({
      ruleId: rule.id,
      ...dto,
    });
    this.log.log('createForRule end', { ruleId: rule.id, conditionId: condition.id });
    return condition;
  }
}
```

Reward: +50.

---

## Common violations

### ❌ Composition treated as dependency (causes cycles or duplicate work)

```ts
// ConditionServiceImpl
async createForRule(ruleId: string, dto: CreateConditionDto) {
  const rule = await this.ruleService.findById(ruleId); // -50: re-validation + cycle risk
  // ...
}
```
This is the typical setup that produces a cycle:
- `RuleModule` imports `ConditionModule` (because `RuleService` calls `ConditionService`).
- `ConditionModule` imports `RuleModule` (because `ConditionService` calls `ConditionService.findById`).
- → -500 under rule #11 plus -50 here.

Fix: accept the `Rule` entity, drop the `RuleService` injection.

### ❌ Dependency treated as composition (loses validation)

```ts
async createOrder(user: User, dto: CreateOrderDto) {
  // accepts the user entity, but User and Order are independent —
  // the user might have been mutated, soft-deleted, etc. since the caller fetched it
}
```
For independent entities, the child should re-validate (or at least re-fetch authoritative state). Accepting a stale entity in a dependency relationship can let stale or invalid state slip through.

### ❌ Passing both the ID and the entity

```ts
async createForRule(ruleId: string, rule: Rule, dto: CreateConditionDto) {
  // -50: redundant. Pass the Rule; rule.id is enough.
}
```

### ❌ Composition child has its own controller / public API surface

If `ConditionController` exposes endpoints that take `ruleId` and let users mutate conditions independently of their rule, the composition relationship has been broken at the API boundary. Conditions should be addressable only through their rule (e.g. `POST /rules/:ruleId/conditions`).

---

## How to apply this rule when reviewing

1. **Identify related services within a domain (or close domains).**
2. **Ask: "Does B make sense without A?"**
   - Yes → expect to see A's ID passed and B re-validating. That's normal.
   - No → expect to see A's *entity* passed and B *not* re-validating. If B is doing validation by ID, that's the violation.
3. **Look for cycles between A and B.** A composition relationship implemented as a dependency is the #1 cause of `forwardRef` showing up.

## Detection

```bash
# Method signatures that take an Id of a parent + a DTO often indicate composition
# being implemented as dependency. Worth a look.
grep -rn "Id.*Dto\|Id.*: string.*Dto" src/services/

# Calls from a "child" service back into a "parent" service (heuristic)
# Worth reading carefully whenever you see X.Service injecting Y.Service AND Y.Service injecting X.Service
grep -rn "private readonly.*Service" src/services/ | sort | uniq -c
```

The strongest mechanical signal is rule #11 — any cycle is suspect for a misclassified composition relationship.

## Edge cases

- **Cross-module composition**: composition usually lives within a single module. If A and B are in different modules and their relationship is composition, consider whether they really belong in the same module.
- **N+1 lookups**: if the parent passes itself into the child for many siblings (e.g. 100 conditions per rule), passing the entity once is much faster than 100 `findById` calls.
- **Ownership transfer**: if a composition can later be "reparented" (a comment moved from one article to another), it's actually a dependency. Reclassify.
- **Soft validation**: even for composition, the child can still validate constraints local to itself (e.g. condition values must be in range). What it skips is re-validating that the parent exists.

## How to write the finding

Violation:
```
<file>:<line> — Rule #12: <ChildService>.<method>(<ParentId>) treats composition as dependency. Accept <Parent> entity instead, drop <ParentService> injection. -50
```

Reward:
```
<file>:<line> — Rule #12 reward: composition relationship modeled correctly — child accepts parent entity, no callback to parent service. +50
```
