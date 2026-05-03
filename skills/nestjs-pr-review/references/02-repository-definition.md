# Rule 2 — Repository Definition

> **Penalty: -100 per occurrence** · **Reward: +50**

## The rule

All native queries and query-builder logic live in repositories. Services must contain zero `createQueryBuilder`, raw SQL, or direct DataSource queries.

## Why it matters

- **Testability**: services with raw queries are hard to unit test — you'd need to spin up a DB or stub the entire query builder. Repository methods can be cleanly mocked.
- **Reuse**: queries hidden inside service methods can't be reused. Repository methods are reusable units.
- **Type safety**: repository methods declare their return types. Inline query builders often degrade to `any`.
- **Service readability**: a service should read like a business workflow, not a SQL builder. Mixing the two makes both unreadable.

## What goes where

**Repository:**
- `createQueryBuilder` calls
- Raw SQL via `.query()` or `EntityManager.query`
- Complex `find` calls with deep relations or custom WHERE
- Aggregations, joins, window functions
- Native upserts (`ON CONFLICT`, `MERGE`)
- Any query the service would otherwise have to construct inline

**Service:**
- Calls to repository methods
- Validation, business rules
- Orchestration across multiple repositories
- Event emission, logging, error casting
- Caching/decorator wiring (delegated to decorators per rule #5)

---

## Examples

### ❌ Violation: query builder in service

```ts
@Injectable()
export class OrderServiceImpl extends OrderService {
  async findActiveOrdersForUser(userId: string) {
    return this.orderRepo
      .createQueryBuilder('o')
      .leftJoinAndSelect('o.items', 'i')
      .where('o.userId = :userId', { userId })
      .andWhere('o.status IN (:...statuses)', { statuses: ['PENDING', 'PAID'] })
      .orderBy('o.createdAt', 'DESC')
      .getMany();
  }
}
```
Penalty: -100.

### ✅ Refactored

**`order.repository.ts`**
```ts
@Injectable()
export class OrderRepository {
  constructor(
    @InjectRepository(Order) private readonly repo: Repository<Order>,
  ) {}

  findActiveByUser(userId: string): Promise<Order[]> {
    return this.repo
      .createQueryBuilder('o')
      .leftJoinAndSelect('o.items', 'i')
      .where('o.userId = :userId', { userId })
      .andWhere('o.status IN (:...statuses)', { statuses: ['PENDING', 'PAID'] })
      .orderBy('o.createdAt', 'DESC')
      .getMany();
  }
}
```

**`order.service.impl.ts`**
```ts
@Injectable()
export class OrderServiceImpl extends OrderService {
  constructor(private readonly orderRepo: OrderRepository) {}

  async findActiveOrdersForUser(userId: string): Promise<Order[]> {
    return this.orderRepo.findActiveByUser(userId);
  }
}
```
Reward: +50 when applied consistently across the module.

---

## More violation patterns

### ❌ Raw query in service
```ts
async getStats() {
  return this.dataSource.query('SELECT ...'); // -100
}
```

### ❌ Inline `Repository<>` from `@InjectRepository` used for query builder
```ts
constructor(@InjectRepository(User) private userRepo: Repository<User>) {}

async findX() {
  return this.userRepo.createQueryBuilder('u')...; // -100
}
```
Even though `Repository<User>` is technically a repository class, using its query builder inside a service still violates the rule. Wrap the query in a custom `UserRepository` class.

### ❌ Spreading WHERE filters into a `.find` from inside the service
```ts
const where: any = { status: 'ACTIVE' };
if (filter.tier) where.tier = filter.tier;
if (filter.region) where.region = filter.region;
return this.userRepo.find({ where }); // -100
```
Borderline cases:
- Simple `find({ where: { id } })` in a service is **fine** (no penalty).
- Multi-condition dynamic WHERE construction belongs in a repository method like `findByFilter(filter)`.

### ❌ DTO-to-query mapping in service
```ts
async search(dto: SearchDto) {
  const qb = this.repo.createQueryBuilder('x');
  if (dto.q) qb.where('x.name ILIKE :q', { q: `%${dto.q}%` });
  if (dto.tags?.length) qb.andWhere('x.tags && :tags', { tags: dto.tags });
  // ... -100
}
```
Move to `XRepository.search(dto)`.

---

## Detection

```bash
grep -rn "createQueryBuilder\|\.query(\|EntityManager\|getRepository(" src/ \
  | grep -E "/services/|\.service\." \
  | grep -v "\.repository\."
```

Each match is a -100 candidate. Review for false positives:
- `@InjectRepository` in a constructor is not itself a violation — it's only a violation if the service then calls `createQueryBuilder` on it. (Ideally services don't inject `Repository<>` at all and inject the custom repository class instead.)
- `.find({ where: { id } })` — simple finds in services are fine.
- `.findOne(id)` — fine.
- `.save(entity)` — fine in services (this is the "command" side, not query construction).

## Edge cases

- **Tiny services with one method**: even a 20-line service should not contain `createQueryBuilder`. The repository file is cheap.
- **Aggregations in controllers**: very rare and almost always wrong. Same -100 penalty applies.
- **Ad-hoc admin scripts**: outside the rule. Don't penalize one-off scripts.
- **Repository methods that internally call other repository methods**: fine. The boundary is service ↔ repository, not repository ↔ repository.

## How to write the finding

```
<file>:<line> — Rule #2: <method> uses createQueryBuilder/raw query inside service. Move to <Domain>Repository. -100
```

If the same file has multiple violations, list each with its own line number and apply -100 each.
