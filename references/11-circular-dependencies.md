# Rule 11 — Circular Dependencies

> **Penalty: -500** · **Reward: +50**
> **Critical** — violations block merge regardless of net score.

## The rule

**Zero** circular dependencies in the codebase. `forwardRef` workarounds are not acceptable — restructure instead.

## Why it matters

- **Design smell**: a cycle says two modules can't be reasoned about independently. Either they should be merged, or one of them is doing too much.
- **Build/runtime fragility**: NestJS DI can usually resolve cycles via `forwardRef`, but the resolution depends on instantiation order — a refactor elsewhere can suddenly produce `undefined` injections that work in dev and crash in prod.
- **Test difficulty**: cyclic modules are nearly impossible to unit test in isolation. You need to wire up both sides every time.
- **Reasoning ceiling**: cycles obscure the dependency graph. New engineers can't follow the call chain because there isn't a single direction.
- **Refactor blast radius**: in a clean DAG, changing module A only affects modules that depend on A. In a cycle, changes ripple in both directions.

`forwardRef` is the canonical "temporary fix" that becomes permanent. Once accepted, more cycles tend to follow. The rule treats `forwardRef` as a violation, not a tool.

---

## Detection

Run `madge` on the changed module graph:

```bash
# Module-level cycles
npx madge --circular --extensions ts src/

# Specific format
npx madge --circular --extensions ts --format json src/ > cycles.json
```

NestJS itself emits a warning in startup logs for circular module dependencies (`A circular dependency has been detected ...`). This warning is sufficient evidence on its own.

Other detection signals:
- `forwardRef(() => SomeModule)` or `forwardRef(() => SomeService)` anywhere in the diff. Each occurrence is a -500 candidate.
- `@Inject(forwardRef(...))` in a constructor.

```bash
grep -rn "forwardRef" src/
```

Each match is a critical finding.

---

## Resolution patterns

When you spot a cycle, choose one of these, in order of preference:

### Option 1 — Composition over dependency (rule #12)

The most common fix. If `RuleService` and `ConditionService` reference each other:

**Before** (cycle):
```
RuleService → ConditionService → RuleService (to validate ruleId)
```

**After**:
```
RuleService → ConditionService
                      ↑
              (passes Rule entity in, no callback to RuleService)
```

`ConditionService` receives the validated `Rule` instance from its caller and trusts it. No circular call. This is rule #12 in action; see `references/12-dependency-vs-composition.md`.

### Option 2 — Extract a shared interface or domain model

If two services both need to know a small piece of each other's domain, factor that piece into a third module both depend on.

```
A ↔ B   →   A → C ← B
```

`C` typically holds:
- A shared abstract type / interface (rule #3 / rule #7).
- A simple value object or DTO.
- A small read-only repository.

### Option 3 — Event-based decoupling

Replace a direct call with an event emission.

**Before**:
```ts
// in OrderService
await this.notificationService.notifyAdmins(order); // imports NotificationModule
```
**After**:
```ts
// in OrderService
this.eventBus.emit('order.create', order);
// elsewhere, NotificationListener listens for 'order.create'
```

`OrderModule` no longer imports `NotificationModule`; the listener lives in `NotificationModule` (or a dedicated subscribers module).

### Option 4 — Merge the modules

If A and B are tightly coupled enough to need a cycle, they may genuinely be one domain. Combine them. This is uncommon but legitimate — a cycle that resists every other resolution is often a misdrawn module boundary.

---

## What is NOT acceptable

### ❌ `forwardRef`
```ts
@Module({
  imports: [forwardRef(() => UserModule)], // -500
})
export class OrderModule {}

constructor(@Inject(forwardRef(() => UserService)) private user: UserService) {} // -500
```
Each `forwardRef` is a -500. The rule treats `forwardRef` as the marker of an unresolved cycle, not a fix.

### ❌ Lazy require
```ts
async someMethod() {
  const { OtherService } = require('../other/services/other.service'); // -500
  // ...
}
```
A workaround that hides the cycle from the dependency graph but doesn't remove it.

### ❌ "Just for now" cycle
A cycle introduced with a comment promising to fix it later is still -500. The rule is critical — it doesn't bend for promises.

---

## Edge cases

- **Module imports vs service imports**: NestJS distinguishes module-level cycles (`@Module({ imports: [...] })`) from service-level cycles (constructor injection across modules). Both are violations under this rule. `madge` catches both at the file level.
- **Type-only imports**: `import type { X } from '...'` does not create a runtime cycle and does not contribute to NestJS DI cycles. Type-only cycles are tolerated **only** if the imports are truly type-only (compiled away).
- **Test-only cycles**: a cycle that exists only between test files is annoying but not a production violation. Note it as a low-severity finding without the -500.
- **Generated code**: ORM generators or codegen tools occasionally produce cycles. If unavoidable, isolate the generated code in a single module and don't let domain code participate.

## How to write the finding

```
<file>:<line> — Rule #11: forwardRef on <Module/Service> indicates circular dependency between <A> and <B>. Resolve via composition (rule #12), shared interface, or event-based decoupling. -500 (CRITICAL)
```

If `madge` reports a cycle:
```
src/<a>/, src/<b>/ — Rule #11: circular module dependency detected by madge: <a> → <b> → <a>. -500 (CRITICAL)
```

Reward (per module that's part of a clean DAG):
```
src/<domain>/ — Rule #11 reward: no circular dependencies; clean DAG. +50
```
