# Rule 3 — Service Definition (Abstract + Impl)

> **Penalty: -100** · **Reward: +100**

## The rule

Every service is defined as an **abstract class named after the domain** (e.g. `UserService`, **not** `AbstractUserService`). The concrete class is `<Name>Impl` (e.g. `UserServiceImpl`).

In the NestJS module, register with:
```ts
{
  provide: UserService,        // abstract class is the DI token
  useClass: UserServiceImpl,   // concrete implementation
}
```

Consumers inject `UserService` (the abstract type), never `UserServiceImpl`.

## Why it matters

- **Decorator pattern feasibility (rule #5)**: decorators wrap the abstract type. If consumers depend on `UserServiceImpl` directly, you cannot transparently swap in a `CacheDecorator`.
- **Testability**: tests can provide a mock against the abstract class without faking out an entire concrete service.
- **Strategy / multiple implementations**: when a service has multiple variants (e.g. `LocalUserServiceImpl` vs `RemoteUserServiceImpl`), the abstract token lets you swap by environment.
- **Naming clarity**: the domain-named abstract class is what the rest of the system "talks about." The `Impl` suffix marks the implementation detail.

## Naming conventions

| Element | Convention | Example |
|--------|-----------|---------|
| Abstract class | `<Domain>Service` | `UserService` |
| Concrete class | `<Domain>ServiceImpl` | `UserServiceImpl` |
| File for abstract | `<domain>.service.ts` | `user.service.ts` |
| File for concrete | `<domain>.service.impl.ts` | `user.service.impl.ts` |

**Never** use:
- `AbstractUserService` (the `Abstract` prefix is wrong)
- `IUserService` (interface-style Hungarian prefix)
- `UserServiceBase`
- `UserServiceContract`
- `BaseUserService` (this name is reserved for decorator base — see rule #5)

---

## Examples

### ✅ Correct

**`user.service.ts`** (abstract)
```ts
export abstract class UserService {
  abstract findById(id: string): Promise<User>;
  abstract create(dto: CreateUserDto): Promise<User>;
  abstract update(id: string, dto: UpdateUserDto): Promise<User>;
}
```

**`user.service.impl.ts`** (concrete)
```ts
@Injectable()
export class UserServiceImpl extends UserService {
  constructor(
    private readonly userRepo: UserRepository,
    private readonly logger: Logger,
  ) {
    super();
  }

  async findById(id: string): Promise<User> {
    this.logger.log(`findById start, id=${id}`);
    const user = await this.userRepo.findById(id);
    if (!user) throw cast(ErrorTypes.NOT_FOUND, `User ${id} not found`);
    this.logger.log(`findById end, id=${id}`);
    return user;
  }

  // ...
}
```

**`user.module.ts`**
```ts
@Module({
  providers: [
    {
      provide: UserService,
      useClass: UserServiceImpl,
    },
    UserRepository,
  ],
  exports: [UserService],
})
export class UserModule {}
```

**Consumer**
```ts
@Injectable()
export class OrderServiceImpl extends OrderService {
  constructor(private readonly userService: UserService) { // ← abstract type
    super();
  }
}
```

Reward: +100.

---

## Common violations

### ❌ Concrete class only, no abstract
```ts
@Injectable()
export class UserService { /* ... */ } // -100
```
The class is concrete and named `UserService`. There's no abstract layer, so consumers depend on the implementation directly.

### ❌ Abstract prefix
```ts
export abstract class AbstractUserService { ... }  // -100
@Injectable()
export class UserService extends AbstractUserService { ... }
```
Wrong direction: abstract class should own the domain name.

### ❌ Interface-prefix Hungarian style
```ts
export interface IUserService { ... }
@Injectable()
export class UserService implements IUserService { ... }  // -100
```
Use abstract classes, not interfaces. Abstract classes work as DI tokens; interfaces don't (they vanish at runtime).

### ❌ Wrong DI registration
```ts
@Module({
  providers: [UserServiceImpl], // -100, registered concrete as token
})
```
Consumers can no longer inject `UserService` cleanly. The decorator pattern (rule #5) breaks.

### ❌ `useExisting` chain that bypasses the abstract
```ts
{ provide: UserService, useExisting: UserServiceImpl }
```
Borderline — accepted only if `UserServiceImpl` is also a separate provider for some reason. Otherwise prefer `useClass`.

---

## Detection

For each `*.service.ts` or `*.service.impl.ts`:
1. Find the class declaration.
2. Check naming: abstract class is `<Domain>Service`, concrete is `<Domain>ServiceImpl`.
3. Check the corresponding `*.module.ts` for `provide: <Domain>Service, useClass: <Domain>ServiceImpl`.
4. Spot-check consumers: do they inject `<Domain>Service` (abstract) or `<Domain>ServiceImpl` (concrete)?

```bash
# Find concrete-only services (no Impl suffix → likely a violation)
find src -name "*.service.ts" -not -name "*.service.impl.ts" \
  -exec grep -l "@Injectable" {} \;

# Find consumers that inject the Impl directly
grep -rn "ServiceImpl" src/ | grep -E "constructor|inject"
```

## Edge cases

- **Stateless utility services** (e.g. `HashService`, `ClockService`): still follow the rule. The abstract class buys you mockability in tests.
- **Service with one method that's almost trivial**: still follow the rule. Consistency across the codebase outweighs the "but it's tiny" objection.
- **Generic/abstract service for a base CRUD layer**: fine to have a generic abstract `CrudService<T>` that domain abstracts extend, but the domain abstract still owns the public surface.
- **Helper classes that aren't services**: utility classes that aren't `@Injectable` aren't bound by this rule.

## How to write the finding

Violation:
```
<file>:<line> — Rule #3: <Domain>Service is concrete (no abstract). Split into <Domain>Service (abstract) and <Domain>ServiceImpl, register with provide/useClass. -100
```

Reward (apply once per module that does this correctly):
```
<module>.module.ts — Rule #3 reward: abstract <Domain>Service + <Domain>ServiceImpl + correct DI. +100
```
