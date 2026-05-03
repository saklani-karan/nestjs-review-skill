# Rule 7 — Strategy Design Pattern

> **Penalty: -50** · **Reward: +50**

## The rule

Any well-defined singular operation gets a strategy. Define an **abstract type** of the strategy and then implement it.

- **Globally reusable** strategies → registered in a shared `strategies/` module via `useClass` / `provide`.
- **Domain-specific** strategies → registered in the owning domain module.

The classic example:
- Abstract: `RefreshTokenManagementStrategy`
- Implementation: `CacheBasedRefreshTokenManagementStrategy`

## Why it matters

- **Open/closed**: adding a new variant doesn't require modifying existing code. New strategy class, new module registration, done.
- **Testability**: you can unit-test each strategy in isolation. You can swap the strategy in tests for a deterministic version.
- **Configurability**: a strategy can be selected at runtime (env var, feature flag) without `if/else` chains in business logic.
- **Readability**: business code reads `await this.tokenStrategy.refresh(token)` — what *kind* of refresh is happening lives elsewhere.

A "well-defined singular operation" is one where:
- There's a clear input → output contract.
- Multiple reasonable implementations exist (or could exist).
- The implementation choice is orthogonal to the caller.

Examples that warrant a strategy:
- Token storage (cache vs DB vs Redis vs encrypted JWT).
- Notification delivery (email vs SMS vs push).
- Pricing calculation (regional, tiered, promotional).
- Rate limiting (sliding window, token bucket, fixed window).
- Recommendation algorithm (collaborative, content-based, hybrid).

Examples that do NOT warrant a strategy:
- A method with a single concrete implementation today and no foreseeable variants.
- An operation whose behavior is genuinely fixed by the domain.

---

## The pattern

### Abstract strategy

```ts
// strategies/refresh-token-management.strategy.ts
export abstract class RefreshTokenManagementStrategy {
  abstract issue(userId: string): Promise<RefreshToken>;
  abstract refresh(token: string): Promise<RefreshToken>;
  abstract revoke(token: string): Promise<void>;
  abstract isValid(token: string): Promise<boolean>;
}
```

### Implementation

```ts
// strategies/cache-based.refresh-token-management.strategy.ts
@Injectable()
export class CacheBasedRefreshTokenManagementStrategy
  extends RefreshTokenManagementStrategy
{
  constructor(private readonly cache: CacheService) { super(); }

  async issue(userId: string): Promise<RefreshToken> {
    const token = randomToken();
    await this.cache.set(`rt:${token}`, { userId, issuedAt: Date.now() }, { ttl: 86400 });
    return { token, expiresIn: 86400 };
  }

  async refresh(token: string): Promise<RefreshToken> { /* ... */ }
  async revoke(token: string): Promise<void> { /* ... */ }
  async isValid(token: string): Promise<boolean> { /* ... */ }
}
```

### Registration

**Domain-specific** — register in the domain module:
```ts
@Module({
  providers: [
    {
      provide: RefreshTokenManagementStrategy,
      useClass: CacheBasedRefreshTokenManagementStrategy,
    },
  ],
})
export class AuthModule {}
```

**Globally reusable** — register in a shared `StrategiesModule`:
```ts
@Module({
  providers: [
    {
      provide: RefreshTokenManagementStrategy,
      useClass: CacheBasedRefreshTokenManagementStrategy,
    },
    // other shared strategies...
  ],
  exports: [RefreshTokenManagementStrategy],
})
export class StrategiesModule {}
```

### Consumer

```ts
@Injectable()
export class AuthServiceImpl extends AuthService {
  constructor(private readonly tokenStrategy: RefreshTokenManagementStrategy) {
    super();
  }

  async login(...) {
    // ...
    const refresh = await this.tokenStrategy.issue(user.id);
  }
}
```

Reward: +50.

---

## Naming and file layout

| Element | Convention | Example |
|---------|-----------|---------|
| Abstract class | `<Operation>Strategy` | `RefreshTokenManagementStrategy` |
| File | `<operation>.strategy.ts` | `refresh-token-management.strategy.ts` |
| Implementation class | `<Variant><Operation>Strategy` | `CacheBasedRefreshTokenManagementStrategy` |
| Implementation file | `<variant>.<operation>.strategy.ts` | `cache-based.refresh-token-management.strategy.ts` |

Files live in `<domain>/strategies/` (domain-specific) or `src/strategies/` (global).

---

## Common violations

### ❌ `if/else` chain in service that should be a strategy

```ts
async sendNotification(userId: string, msg: string, channel: 'email' | 'sms' | 'push') {
  if (channel === 'email') {
    return this.emailAdapter.send(...);
  } else if (channel === 'sms') {
    return this.smsAdapter.send(...);
  } else {
    return this.pushAdapter.send(...);
  }
}
```
Penalty: -50. Each branch is a strategy. Define `NotificationDeliveryStrategy` (abstract) with `EmailNotificationDeliveryStrategy`, etc., then inject a Map<channel, strategy> or use a dispatcher.

### ❌ Concrete class injected directly without an abstract

```ts
constructor(private tokenStore: CacheBasedTokenStore) {} // -50
```
The consumer is bound to the concrete implementation. Define `TokenStoreStrategy` and inject that.

### ❌ Strategy not registered correctly

Strategy exists as `@Injectable()` but the module just lists it as `providers: [CacheBasedTokenStore]`, with no `provide` token mapping it to the abstract. Consumers either inject the concrete class (defeats the point) or get DI errors. -50.

### ❌ "Strategy" file that's actually a utility

```ts
// strategies/token-utils.strategy.ts — but contains static helper functions
export class TokenUtilsStrategy {
  static parse(token: string) { ... }
  static format(payload: any) { ... }
}
```
Not a strategy. Move to `utils/` or `types/`. -50 if filed under `strategies/` misleadingly.

### ❌ Abstract that takes too many parameters / is too narrow

```ts
abstract class TokenStrategy {
  abstract issue(userId: string, scope: string, ttl: number, refresh: boolean): Promise<...>;
}
```
A strategy abstract should be cohesive and operation-focused. Sprawling, multi-parameter abstracts often indicate two strategies smushed into one. -50 for design smell.

---

## Detection

```bash
# Find strategy files
find src -name "*.strategy.ts"

# Each abstract should have at least one Impl variant
# Each Impl should be registered with provide/useClass in its module
```

For each `*.strategy.ts`:
- Is there a corresponding implementation file?
- Is the implementation registered in a module via `provide: <Abstract>, useClass: <Impl>`?
- Does the consumer inject the abstract or the concrete?

Look for `if/else` chains on type-discriminator values (`channel`, `kind`, `type`, `variant`) in services — these are often strategies waiting to be extracted.

## Edge cases

- **Single-implementation strategies**: borderline. If you have only one implementation today and no plan for variants, don't force a strategy. The pattern earns its complexity by enabling variants.
- **Strategy that delegates to an adapter (rule #8)**: fine. The strategy decides *which* adapter to call; the adapter handles the third-party interaction.
- **Strategy registry / dispatcher**: when you have N strategies keyed by some discriminator, a `StrategyRegistry` that maps `key → strategy` is acceptable. Inject the registry; let it route.
- **Strategy in a generic abstract module**: e.g. a `CachingModule` exporting a `CacheStrategy`. Fine; same rules.

## How to write the finding

```
<file>:<line> — Rule #7: <operation> implemented as if/else chain (or concrete class). Define <Operation>Strategy abstract + <Variant><Operation>Strategy implementations and register via provide/useClass. -50
```

Reward:
```
<module>.module.ts — Rule #7 reward: <Operation>Strategy abstract + implementations + correct DI registration. +50
```
