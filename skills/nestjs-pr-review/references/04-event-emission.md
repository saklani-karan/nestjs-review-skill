# Rule 4 — Event Emission

> **Penalty: -50** · **Reward: +100**

## The rule

Event patterns describe **the transaction that has been committed**, not the work to be done. Emitters never know about subscribers.

- ✅ `user.create` — emitted when a user has been committed.
- ❌ `send.welcome.email` — describes the subscriber's job.

## Why it matters

- **Loose coupling**: new subscribers can be added without touching the emitter. If the pattern is named after a subscriber's job, every new subscriber requires emitting a new event.
- **Single source of truth**: `user.create` is fired once. Five subscribers can attach to it (welcome email, audit log, analytics, search index, CRM sync). With a job-named pattern, you'd need five separate emits.
- **Reasoning**: when reading `eventBus.emit('user.create', user)`, a reader immediately knows what *happened*. With `eventBus.emit('send.welcome.email', user)`, they have no idea what state actually changed.
- **Testability**: assertions like "after `createUser`, `user.create` was emitted" are stable. Assertions like "after `createUser`, `send.welcome.email` was emitted" are brittle and conflate the emitter with the subscriber.

## The naming pattern

`<entity>.<action>` where:
- `<entity>` is the domain entity (singular, lowercase).
- `<action>` is the past-tense verb describing what happened to it.

| ✅ Correct | ❌ Wrong | Why wrong |
|------------|----------|-----------|
| `user.create` | `send.welcome.email` | Names the subscriber's job |
| `user.create` | `user.created.send.email` | Mixes event + job |
| `order.paid` | `notify.warehouse` | Names the subscriber |
| `subscription.cancel` | `cleanup.user.data` | Names the cleanup task |
| `payment.refund` | `update.accounting.ledger` | Names a downstream system |

Some teams prefer `user.created` (past participle); either tense is acceptable as long as it describes the *event*, not the *handler*. Pick one and apply consistently.

---

## Examples

### ✅ Correct

```ts
@Injectable()
export class UserServiceImpl extends UserService {
  constructor(private readonly eventBus: EventBus) { super(); }

  @Transactional()
  async create(dto: CreateUserDto): Promise<User> {
    const user = await this.userRepo.save(dto);
    this.eventBus.emit('user.create', user); // describes what committed
    return user;
  }
}

@EventListener('user.create')
class SendWelcomeEmailHandler { /* ... */ }

@EventListener('user.create')
class CreateAuditLogHandler { /* ... */ }

@EventListener('user.create')
class IndexInSearchHandler { /* ... */ }
```

Reward: +100.

### ❌ Violation: subscriber-named events

```ts
async create(dto: CreateUserDto): Promise<User> {
  const user = await this.userRepo.save(dto);
  this.eventBus.emit('send.welcome.email', user);   // -50
  this.eventBus.emit('create.audit.log', user);     // count once per service that does this; the rule is about the pattern not each emit
  this.eventBus.emit('index.user.in.search', user);
  return user;
}
```
Penalty: -50 (applied once per service exhibiting the anti-pattern).

If the same service mixes correct and incorrect patterns, still -50 — the codebase needs a single consistent style.

---

## Detection

```bash
# Find all emit calls
grep -rn "\.emit(\|emit(" src/ --include="*.ts"
```

For each match, evaluate the pattern string:
- Does it match `<entity>.<action>`?  → ✅
- Does it contain a verb implying the *subscriber's* job (`send`, `notify`, `update`, `sync`, `index`, `cleanup`, `trigger`)?  → ❌
- Is it ambiguous? Read the surrounding code: what was just committed? If the name doesn't match, it's a violation.

### Quick heuristic
If you can answer "what was the subscriber going to do?" from the event name alone, it's likely wrong.
If you can answer "what changed in the database?" from the event name alone, it's likely right.

## Edge cases

- **Aggregate events** (e.g. `order.lifecycle`): rare and usually a smell. Prefer specific events (`order.create`, `order.paid`, `order.shipped`, `order.deliver`).
- **Domain events that aren't tied to a single entity** (e.g. `import.complete`): the entity here is the import job itself. Acceptable.
- **External webhook adapters** that re-emit third-party events: you may have to keep the third-party naming for the inbound event, but as soon as it crosses into your domain, re-emit with the proper convention. (`stripe.payment_intent.succeeded` → `payment.succeed` internally.)
- **Wildcard subscribers** (`user.*`): fine and even encouraged for cross-cutting handlers like audit logs.
- **Past tense vs imperative**: `user.create` and `user.created` are both common. Within one codebase, pick one. Penalize mixing.

## How to write the finding

```
<file>:<line> — Rule #4: event pattern '<pattern>' names the subscriber's job rather than the committed transaction. Rename to '<entity>.<action>'. -50
```

Reward (once per module that consistently follows the pattern):
```
<file> — Rule #4 reward: event emission consistently uses entity.action pattern. +100
```
