# Rule 9 — Error Handling

> **Penalty: -200** · **Reward: +50**

## The rule

Every error thrown from the service layer is cast to a known `ErrorTypes` variant. Only truly **unhandled** errors should surface as `InternalServerError`.

| Origin                  | Cast to                                        |
|-------------------------|------------------------------------------------|
| Repository / DB error   | `ErrorTypes.DB_ERROR`                          |
| `HttpException` (caught)| `ErrorTypes.HTTP_EXCEPTION` (preserve original)|
| Other handled errors    | Specific `ErrorTypes.*` variant                |
| Truly unexpected        | Allow `InternalServerError` to bubble          |

## Why it matters

- **Consistent error contract**: every consumer (controllers, other services, tests) sees a known shape. No `instanceof` archaeology.
- **No leaked DB internals**: raw TypeORM/Postgres errors carry SQL fragments, table names, and stack traces that are sensitive in production.
- **Observable failure modes**: `ErrorTypes` becomes a closed enum of named failure modes that maps cleanly to monitoring dashboards and alerts.
- **Preserved HTTP semantics**: an upstream service that returned 404 should propagate as 404 from your service, not get flattened into a 500.
- **Easier debugging**: logs that say `DB_ERROR: foreign_key_violation on user.organization_id` are far more actionable than a stringified Postgres error.

---

## The pattern

### `ErrorTypes` enum and base exception class

```ts
// common/exceptions/error-types.ts
export enum ErrorTypes {
  DB_ERROR        = 'DB_ERROR',
  HTTP_EXCEPTION  = 'HTTP_EXCEPTION',
  NOT_FOUND       = 'NOT_FOUND',
  VALIDATION      = 'VALIDATION',
  CONFLICT        = 'CONFLICT',
  UNAUTHORIZED    = 'UNAUTHORIZED',
  FORBIDDEN       = 'FORBIDDEN',
  EXTERNAL_ERROR  = 'EXTERNAL_ERROR',  // adapter/third-party failures
  RATE_LIMIT      = 'RATE_LIMIT',
  // ...
}

export class AppException extends Error {
  constructor(
    public readonly type: ErrorTypes,
    message: string,
    public readonly cause?: unknown,
    public readonly meta?: Record<string, unknown>,
  ) {
    super(message);
    this.name = 'AppException';
  }
}

export const cast = (
  type: ErrorTypes,
  message: string,
  cause?: unknown,
  meta?: Record<string, unknown>,
) => new AppException(type, message, cause, meta);
```

### Service-layer pattern

```ts
async findById(id: string): Promise<User> {
  this.logger.log(`findById start, id=${id}`);
  try {
    const user = await this.userRepo.findById(id);
    if (!user) {
      throw cast(ErrorTypes.NOT_FOUND, `User ${id} not found`, undefined, { id });
    }
    this.logger.log(`findById end, id=${id}`);
    return user;
  } catch (e) {
    if (e instanceof AppException) throw e;          // already cast — bubble
    if (e instanceof HttpException) {
      throw cast(ErrorTypes.HTTP_EXCEPTION, e.message, e, { status: e.getStatus() });
    }
    if (e instanceof QueryFailedError) {
      throw cast(ErrorTypes.DB_ERROR, 'Database query failed', e);
    }
    throw e; // truly unexpected — InternalServerError will bubble up
  }
}
```

### A higher-level exception filter (controller layer)

```ts
@Catch(AppException)
export class AppExceptionFilter implements ExceptionFilter {
  catch(exception: AppException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = this.statusFromType(exception.type);
    response.status(status).json({
      type: exception.type,
      message: exception.message,
      meta: exception.meta,
    });
  }

  private statusFromType(type: ErrorTypes): number {
    switch (type) {
      case ErrorTypes.NOT_FOUND:    return 404;
      case ErrorTypes.VALIDATION:   return 422;
      case ErrorTypes.UNAUTHORIZED: return 401;
      case ErrorTypes.FORBIDDEN:    return 403;
      case ErrorTypes.CONFLICT:     return 409;
      case ErrorTypes.RATE_LIMIT:   return 429;
      case ErrorTypes.HTTP_EXCEPTION: /* preserve from cause */ return 500;
      case ErrorTypes.DB_ERROR:     return 500;
      case ErrorTypes.EXTERNAL_ERROR: return 502;
      default:                      return 500;
    }
  }
}
```

Reward: +50 when the service consistently uses this pattern.

---

## Common violations

### ❌ Raw `Error` thrown
```ts
if (!user) throw new Error('not found'); // -200
```
No `ErrorTypes`. The caller can't distinguish this from any other error.

### ❌ Raw DB error allowed to bubble
```ts
async findById(id: string) {
  return this.userRepo.findById(id); // throws QueryFailedError unwrapped — -200
}
```
The controller layer (or test) sees a TypeORM-specific error. -200.

### ❌ HttpException flattened
```ts
try {
  return await this.upstream.fetch(id);
} catch (e) {
  throw new InternalServerErrorException(e.message); // -200
}
```
A 404 from upstream becomes a 500. Cast to `ErrorTypes.HTTP_EXCEPTION` and preserve the status.

### ❌ Catch-all that swallows the type
```ts
try {
  // ...
} catch (e) {
  throw new BadRequestException('something went wrong'); // -200
}
```
Lossy — the original error type and message are gone. Real cause is lost.

### ❌ Empty catch
```ts
try {
  await this.eventBus.emit('user.create', user);
} catch (e) {
  // ignored
}
```
Borderline. If the operation is fire-and-forget and a comment explains why a failure is acceptable: ok. Otherwise: -200.

### ❌ Inconsistent ErrorTypes
Some methods cast, others don't. Apply -200 once for the file showing the inconsistency.

---

## Detection

```bash
# Raw Error throws in services
grep -rn "throw new Error(" src/ | grep -E "/services/|\.service\."

# Internal server errors thrown explicitly (often a smell)
grep -rn "InternalServerErrorException\|new InternalServerError" src/

# Try blocks without re-cast
grep -rn -B0 -A3 "} catch" src/ | grep -E "/services/|\.service\."

# Bare repo calls with no try
# (harder to detect mechanically — review service methods that call repo methods
#  to ensure errors are caught and cast)
```

Walk each service method and ask:
1. Does it call a repository? If yes, is `QueryFailedError`/equivalent caught and cast to `DB_ERROR`?
2. Does it call an adapter? If yes, is the adapter error caught and cast (typically to `EXTERNAL_ERROR` or `HTTP_EXCEPTION`)?
3. Does it throw any error? If yes, is the throw a `cast(ErrorTypes.X, ...)` call?

## Edge cases

- **Validation via `class-validator` and Nest's pipes**: handled at the framework boundary (controller) and gives `BadRequestException`. Service-layer validation logic must still cast to `ErrorTypes.VALIDATION`.
- **Transactional rollback**: when a transaction rolls back due to an exception, the exception still needs to be cast. The rollback is independent of the error contract.
- **Repository methods that already throw `AppException`**: fine. The service can re-throw or pass through.
- **Background jobs / queue handlers**: same rules. Errors should be cast so the job orchestrator can decide retry policy by type.
- **Adapter (rule #8) errors**: each provider should map vendor-specific errors to `AppException` with `EXTERNAL_ERROR` or a more specific type. Then services don't have to know vendor error shapes.

## How to write the finding

```
<file>:<line> — Rule #9: <method> throws raw Error / lets QueryFailedError bubble / flattens HttpException. Cast to ErrorTypes.<X> via cast(). -200
```

Reward (per service that consistently casts):
```
<file> — Rule #9 reward: all error paths cast to ErrorTypes; HttpException preserved; DB errors mapped to DB_ERROR. +50
```
