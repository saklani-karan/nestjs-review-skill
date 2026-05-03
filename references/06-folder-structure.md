# Rule 6 — Folder Structure

> **Penalty: -200** · **Reward: +50**

## The rule

Every domain is its own NestJS module with the following layout. Every file ends in `<name>.<type>.ts`.

```
<domain>/
├── controllers/
│   └── <domain>.controller.ts
├── services/
│   ├── <domain>.service.ts            # abstract (rule #3)
│   └── <domain>.service.impl.ts       # concrete (rule #3)
├── repositories/
│   └── <domain>.repository.ts
├── entities/
│   └── <domain>.entity.ts
├── types/
│   └── <domain>.types.ts
├── strategies/                         # if applicable (rule #7)
│   ├── <name>.strategy.ts              # abstract
│   └── <name>.<variant>.strategy.ts    # implementation
├── decorators/                         # if applicable (rule #5)
│   ├── base.decorator.ts
│   └── <concern>.decorator.ts
└── <domain>.module.ts
```

## Why it matters

- **Predictability**: anyone landing in a new domain folder knows where to look. Controllers are in `controllers/`, repositories are in `repositories/`. No spelunking.
- **Module-level encapsulation**: each domain folder is a complete, self-contained unit. Moving or extracting a domain is a `git mv` away.
- **Tooling**: lint rules, codegen, and import-path conventions all benefit from consistent structure.
- **Onboarding**: new engineers don't have to learn a bespoke layout per project.

The `<name>.<type>.ts` filename convention also helps editors with fuzzy-finding (`@user.svc` → resolves quickly) and makes grep queries cleaner.

---

## Required folders

| Folder | Required? | Notes |
|--------|-----------|-------|
| `controllers/` | If the domain has HTTP endpoints | One file per controller |
| `services/` | Always | Both abstract and `Impl` files |
| `repositories/` | If the domain persists data | Custom repo class per rule #2 |
| `entities/` | If the domain has DB entities | TypeORM/neomodel/etc. |
| `types/` | Always | DTOs, interfaces, enums, type aliases |
| `<domain>.module.ts` | Always | NestJS module file at domain root |

## Optional folders

| Folder | When to add |
|--------|-------------|
| `strategies/` | When the domain implements rule #7 (strategy pattern) |
| `decorators/` | When the domain implements rule #5 (decorator pattern) |
| `events/` | When the domain has many event handlers; otherwise put listeners alongside services |
| `validators/` | Rare — only if the domain has many class-validator constraints |
| `mappers/` | If you have explicit entity↔DTO mappers; otherwise keep mapping inline |

## File naming

| Type | Filename | Class |
|------|----------|-------|
| Controller | `user.controller.ts` | `UserController` |
| Abstract service | `user.service.ts` | `UserService` |
| Service impl | `user.service.impl.ts` | `UserServiceImpl` |
| Repository | `user.repository.ts` | `UserRepository` |
| Entity | `user.entity.ts` | `User` |
| DTO | `create-user.dto.ts` | `CreateUserDto` |
| Type | `user.types.ts` | (multiple) |
| Strategy abstract | `token.strategy.ts` | `TokenStrategy` |
| Strategy impl | `cache-based.token.strategy.ts` | `CacheBasedTokenStrategy` |
| Decorator base | `base.decorator.ts` | `BaseUserServiceDecorator` |
| Decorator impl | `cache.decorator.ts` | `CacheUserServiceDecorator` |
| Module | `user.module.ts` | `UserModule` |

---

## Examples

### ✅ Correct layout

```
src/user/
├── controllers/
│   └── user.controller.ts
├── services/
│   ├── user.service.ts
│   └── user.service.impl.ts
├── repositories/
│   └── user.repository.ts
├── entities/
│   └── user.entity.ts
├── types/
│   ├── create-user.dto.ts
│   ├── update-user.dto.ts
│   └── user.types.ts
├── decorators/
│   ├── base.decorator.ts
│   └── cache.decorator.ts
└── user.module.ts
```
Reward: +50.

### ❌ Common violations

**Flat structure**
```
src/user/
├── user.controller.ts
├── user.service.ts
├── user.repository.ts
├── user.entity.ts
└── user.module.ts
```
No folders — each file lives at the root. -200.

**Mixed concerns**
```
src/user/
├── services/
│   ├── user.service.ts
│   ├── user.service.impl.ts
│   └── user.repository.ts        ← repository in services/, -200
└── ...
```

**Wrong filename suffix**
```
src/user/services/userService.ts                ← camelCase, no .service.ts
src/user/services/UserServiceImpl.ts            ← PascalCase filename
src/user/repositories/userRepo.ts               ← truncated name
```
-200 for the module.

**No module file**
```
src/user/
├── services/...
└── (no user.module.ts)
```
-200. The domain isn't a NestJS module at all.

**Multiple domains in one folder**
```
src/account/
├── services/
│   ├── user.service.ts
│   ├── user.service.impl.ts
│   ├── organization.service.ts
│   └── organization.service.impl.ts
```
Two domains crammed into one folder. -200. Split into `src/user/` and `src/organization/`.

---

## Detection

For each touched domain folder:

```bash
# Are required folders present?
test -d src/<domain>/services || echo "missing services/"
test -d src/<domain>/types    || echo "missing types/"
test -f src/<domain>/<domain>.module.ts || echo "missing module file"

# Files following the naming convention?
find src/<domain> -name "*.ts" | grep -vE "\.(controller|service|service\.impl|repository|entity|module|types|dto|strategy|decorator)\.ts$"
```

Each match in the last command is a naming violation. Penalize -200 once per domain that has structural problems (don't stack -200 per file).

## Edge cases

- **Tiny domains** (e.g. a domain with only an enum and a single utility): still follow the structure. The `services/` and `module.ts` files are non-negotiable. `repositories/` and `entities/` can be omitted if there's no persistence.
- **Shared / common modules**: `src/common/` and `src/shared/` may use a slightly looser structure since they're not domain-specific, but still require `<name>.<type>.ts` naming.
- **NestJS sub-modules**: a feature module that lives under another module (e.g. `src/billing/invoicing/`) follows the same structure.
- **Test files**: `*.spec.ts` files live next to the file they test; this rule doesn't constrain them.
- **Migrations / config**: outside the rule.

## How to write the finding

Violation:
```
src/<domain>/ — Rule #6: <specific issue, e.g. "no controllers/ folder; controller at module root"> — restructure to standard domain layout (see references/06-folder-structure.md). -200
```

Reward (per domain that follows the layout cleanly):
```
src/<domain>/ — Rule #6 reward: standard domain layout with all required folders and naming conventions. +50
```
