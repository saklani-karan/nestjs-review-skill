# Rule 1 — Transactional Management

> **Penalty: -1000 per violation** (must-have) · **Reward: +10**
> **Critical** — violations block merge regardless of net score.

## The rule

No competing transactions in a single request. Pick **one** strategy per project; never mix.

- If the project uses **`nestjs-transactions`** (e.g. `typeorm-transactional`) → every function that invokes a DB transaction must be marked `@Transactional()`.
- If the project uses **`withTransaction`** / manual transaction context → every DB call must run inside `runTransaction(...)`.

## Why it matters

Competing transactions cause:
- **Phantom commits** — work the caller thinks rolled back actually committed in a sibling transaction.
- **Deadlocks** — two transactions in the same request hold incompatible locks.
- **Lost writes** — a child transaction commits before the parent decides to roll back.
- **Event-timing bugs** — async event handlers reading data the parent hasn't committed yet.

The rule is about *intent*: every code path that touches the DB must declare which transaction it belongs to, and the declaration must be consistent across the codebase.

## Identifying the strategy in use

Check in order:
1. Imports of `Transactional` from `typeorm-transactional` or similar → **Strategy A (nestjs-transactions)**.
2. A `TransactionManager` / `runTransaction` helper that wraps a callback → **Strategy B (withTransaction)**.
3. Raw `EntityManager.transaction(...)` or `dataSource.transaction(...)` → treat as Strategy B.

If both styles appear in the same codebase, that itself is a -1000 violation. File a critical finding and stop.

---

## Strategy A — `nestjs-transactions`

### Correct usage

```ts
import { Transactional } from 'typeorm-transactional';

@Injectable()
export class UserServiceImpl extends UserService {
  @Transactional()
  async createUserWithProfile(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.save(dto);
    await this.profileService.createForUser(user); // also @Transactional → joins same tx
    return user;
  }
}
```

`profileService.createForUser` must also be `@Transactional()` so it joins the existing transaction context instead of opening its own.

### Common violations

**❌ Missing `@Transactional` on a method that writes to the DB**
```ts
async createUserWithProfile(dto: CreateUserDto) { // no decorator
  const user = await this.userRepo.save(dto);
  await this.profileService.createForUser(user);
}
```
Penalty: -1000.

**❌ Caller is `@Transactional`, callee is not**
```ts
@Transactional()
async createOrder(...) {
  await this.lineItemService.createBatch(...); // not @Transactional → opens its own tx
}
```
Penalty: -1000. The line items commit independently of the order — if the order rolls back, the line items remain.

**❌ `propagation: 'REQUIRES_NEW'` without justification**
This *intentionally* opens a sibling transaction. Allowed only when a comment above the decorator explains why (e.g. audit logging that must persist even if the parent rolls back). Without justification: -1000.

---

## Strategy B — `withTransaction` / `runTransaction`

### Correct usage

```ts
async createUserWithProfile(dto: CreateUserDto): Promise<User> {
  return this.txManager.runTransaction(async (tx) => {
    const user = await this.userRepo.save(tx, dto);
    await this.profileService.createForUser(tx, user);
    return user;
  });
}
```

Every repository call receives the `tx` context. `profileService.createForUser` accepts and forwards `tx`.

### Common violations

**❌ DB call outside `runTransaction`**
```ts
async createUserWithProfile(dto: CreateUserDto) {
  const user = await this.userRepo.save(dto); // no tx
  return this.txManager.runTransaction(async (tx) => {
    await this.profileService.createForUser(tx, user);
  });
}
```
Penalty: -1000. The user save commits independently.

**❌ Forgetting to forward `tx`**
```ts
this.txManager.runTransaction(async (tx) => {
  await this.userRepo.save(dto); // tx dropped → uses default connection
});
```
Penalty: -1000.

**❌ Nested `runTransaction` calls**
```ts
this.txManager.runTransaction(async (tx) => {
  await this.profileService.createForUser(user); // opens its own runTransaction inside
});
```
Penalty: -1000 unless justified (REQUIRES_NEW equivalent).

---

## Detection patterns

Run on the diff:

```bash
# Strategy A: every DB-writing service method should have @Transactional
grep -rn "@Transactional" src/

# Strategy B: every write should be near a runTransaction or accept a tx parameter
grep -rn "runTransaction\|\.save(\|\.update(\|\.delete(\|\.insert(" src/services/

# Mixed strategies smell
grep -rn "REQUIRES_NEW\|propagation" src/   # flag every occurrence for justification check
```

For each write site, trace upward: is there a transaction wrapper somewhere in the call chain? If not → -1000.

## Edge cases

- **Read-only handlers**: pure reads (e.g. controller GETs that only call `.find`) don't need transactions. No penalty for missing `@Transactional` on read-only paths.
- **Async event handlers triggered inside a transaction**: handlers must NOT do work that depends on the parent's commit visibility. Either use `AFTER_COMMIT` events or document the read-your-own-writes implication.
- **Queue producers inside a transaction**: enqueueing to a job queue inside a transaction is a smell — if the tx rolls back, the job already shipped. Flag for human review (no automatic penalty, but mention it in findings).
- **Soft deletes via `update()` calls**: still writes — same rules apply.
- **Bulk seed/migration scripts**: outside the request lifecycle; this rule doesn't apply.

## How to write the finding

```
<file>:<line> — Rule #1: <method> writes to the DB but is not @Transactional / not inside runTransaction. -1000 (CRITICAL)
```

Include the call chain when relevant: "called by `createOrder` (which IS @Transactional)" — this clarifies that the caller has a tx context the callee silently bypassed.
