# Rule 5 — Decorator Design Pattern

> **Penalty: -50** · **Reward: +100**

## The rule

Service-level cross-cutting concerns (caching, logging, metrics, retries, circuit breakers, audit logging) are added via the **decorator design pattern**:

1. Define a `BaseDecorator` inside the `decorators/` folder of the domain module that extends/implements the abstract service class (rule #3).
2. Concrete decorators (e.g. `CacheDecorator`) extend `BaseDecorator`.
3. Inject in the module via `useFactory` with a `Symbol` token.

## Why it matters

- **Single responsibility**: the service implementation focuses on business logic. Caching, logging, etc. don't pollute it.
- **Composability**: decorators can be stacked. `Cache(Logging(UserServiceImpl))` is a valid configuration.
- **Toggleable**: a decorator can be removed in one place (the module) without touching the service implementation.
- **Testability**: the underlying service can be tested without the decorator's complexity. The decorator can be tested against a mocked underlying service.
- **Open/closed**: adding a new cross-cutting concern doesn't require modifying existing service code.

This rule depends on rule #3. Without an abstract service class, the decorator has nothing clean to wrap.

---

## The pattern

### `decorators/base.decorator.ts`

```ts
export abstract class BaseUserServiceDecorator extends UserService {
  constructor(protected readonly inner: UserService) {
    super();
  }

  // Default implementations forward to the inner service.
  // Concrete decorators override only the methods they care about.
  findById(id: string): Promise<User> {
    return this.inner.findById(id);
  }

  create(dto: CreateUserDto): Promise<User> {
    return this.inner.create(dto);
  }

  update(id: string, dto: UpdateUserDto): Promise<User> {
    return this.inner.update(id, dto);
  }
}
```

### `decorators/cache.decorator.ts`

```ts
export class CacheUserServiceDecorator extends BaseUserServiceDecorator {
  constructor(
    inner: UserService,
    private readonly cache: CacheService,
  ) {
    super(inner);
  }

  async findById(id: string): Promise<User> {
    const cacheKey = `user:${id}`;
    const cached = await this.cache.get<User>(cacheKey);
    if (cached) return cached;

    const user = await this.inner.findById(id);
    await this.cache.set(cacheKey, user, { ttl: 300 });
    return user;
  }

  // Other methods inherit forwarding from BaseUserServiceDecorator.
  // Override write methods to invalidate the cache.
  async update(id: string, dto: UpdateUserDto): Promise<User> {
    const updated = await this.inner.update(id, dto);
    await this.cache.delete(`user:${id}`);
    return updated;
  }
}
```

### Module registration with `useFactory` and a `Symbol` token

```ts
export const CACHED_USER_SERVICE = Symbol('CachedUserService');

@Module({
  providers: [
    UserRepository,

    // The "raw" implementation, kept under a private token if you don't want it injected directly.
    {
      provide: UserServiceImpl,
      useClass: UserServiceImpl,
    },

    // The decorator, registered under the abstract class as the public token.
    {
      provide: UserService,
      useFactory: (inner: UserServiceImpl, cache: CacheService) =>
        new CacheUserServiceDecorator(inner, cache),
      inject: [UserServiceImpl, CacheService],
    },
  ],
  exports: [UserService],
})
export class UserModule {}
```

Consumers continue to inject `UserService` and never know they got a cached version. Reward: +100.

---

## Stacked decorators

```ts
{
  provide: UserService,
  useFactory: (impl: UserServiceImpl, cache: CacheService, logger: Logger) => {
    const logged = new LoggingUserServiceDecorator(impl, logger);
    const cached = new CacheUserServiceDecorator(logged, cache);
    return cached; // Cache → Logging → Impl
  },
  inject: [UserServiceImpl, CacheService, Logger],
}
```

The order matters — the outermost wraps the innermost. Cache-then-log means a cache hit doesn't log the underlying call.

---

## Common violations

### ❌ Caching baked into the service implementation

```ts
@Injectable()
export class UserServiceImpl extends UserService {
  async findById(id: string): Promise<User> {
    const cached = await this.cache.get(`user:${id}`); // -50
    if (cached) return cached;
    const user = await this.userRepo.findById(id);
    await this.cache.set(`user:${id}`, user);
    return user;
  }
}
```
The service knows about the cache. Cross-cutting concern leaked into business logic.

### ❌ Decorator registered with `useClass` instead of `useFactory`

```ts
{ provide: UserService, useClass: CacheUserServiceDecorator } // -50
```
`useClass` can't construct the decorator with its inner service. You'd need `useFactory` or property injection — `useFactory` is the rule's required style.

### ❌ Decorator placed outside the `decorators/` folder

The base + concrete decorators must live in the domain's `decorators/` folder per rule #6. A `cache.ts` file in the root of the module is a structural violation: -50 here and an additional -200 under rule #6.

### ❌ Decorator that doesn't extend the abstract service

```ts
export class CacheUserServiceDecorator { // missing extends UserService — -50
  constructor(private inner: UserService, private cache: CacheService) {}
  findById(id) { /* ... */ }
}
```
This isn't a decorator in the design-pattern sense — it's just a wrapper class. Without inheriting the abstract type, you can't use it as a drop-in replacement for `UserService` in DI.

### ❌ No `Symbol` token when one is warranted

If the module exports both a "raw" `UserService` and a "cached" version for two different consumers, both need distinct tokens. Using a string token (`'CachedUserService'`) instead of `Symbol('CachedUserService')` risks collisions with other modules: -50.

(If only the cached version is exported under `UserService` — the most common case — no separate Symbol is needed.)

---

## Detection

For each domain module:
1. Look in `<domain>/decorators/`. If absent and the service has any cross-cutting concerns inline, that's a violation.
2. Open `<domain>.module.ts`. Look for `useFactory` registrations. If a decorator exists but is registered with `useClass`, -50.
3. Search the service `*.service.impl.ts` for inline cache/log/metric calls that should be in a decorator.

```bash
# Caching inline in services
grep -rn "cache\.\(get\|set\|delete\)" src/ \
  | grep -E "/services/|\.service\." \
  | grep -v "\.decorator\."

# useClass on something named *Decorator (likely wrong)
grep -rn "useClass.*Decorator" src/
```

## Edge cases

- **Logging via NestJS interceptors**: interceptors are an alternative to decorators for HTTP-level logging. They're acceptable for request-scoped logging (rule #10), but service-level logging (e.g. inside a service method) should still be inside the service or in a `LoggingDecorator`.
- **Caching via `@CacheKey` / `@CacheTTL` decorators (NestJS built-in)**: framework-level method decorators on controllers are fine. Service-level caching belongs in a `CacheDecorator` per this rule.
- **Tiny one-off concerns**: even a single cache lookup in a service method should be moved to a decorator. The friction of creating one is the point — it forces clean separation.
- **Mock decorators in tests**: tests can construct decorators directly with mocked inner services; no module changes needed.

## How to write the finding

Violation:
```
<file>:<line> — Rule #5: <concern> implemented inline in service. Move to a <Concern>Decorator extending Base<Domain>ServiceDecorator and register via useFactory in <module>.module.ts. -50
```

Reward:
```
<module>.module.ts — Rule #5 reward: <Concern>Decorator extends BaseDecorator, registered via useFactory with Symbol token. +100
```
