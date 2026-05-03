# Rule 10 — Logging

> **Penalty: -200** · **Reward: +50**

## The rule

**Request-scoped logging** with logs at:
- The **start** of every service method.
- The **end** of every service method (success/failure).
- **Around every repository call** (or other I/O boundary).
- **Every important step or decision point** in the function.

Use the correct log level: `log` (info) for normal flow, `warn` for recoverable issues, `error` for failures, `debug` for verbose diagnostics.

## Why it matters

- **Production debugging**: when a request fails at 3 AM, the logs are all you have. Sparse logging means hours of guessing.
- **Latency attribution**: logs around repo calls let you tell whether a slow endpoint is slow because of the DB, an adapter, or your own logic.
- **Audit trail**: in CSR/financial/sensitive domains, "who did what when" must be reconstructible from logs.
- **Tracing context**: request-scoped loggers carry a correlation/trace ID through the call. One ID lets you follow a request across services and repos.
- **Level discipline**: `error` should mean something is broken. If everything is `error`, alerting becomes useless.

---

## What "request-scoped" means

The logger should carry per-request context: a correlation ID, the route or job name, the user ID (if authenticated), and any other invariant identifiers.

In NestJS, this typically means a request-scoped logger (`@Inject(REQUEST)` or `Scope.REQUEST` providers) or a context-propagating logger like `nestjs-pino` with `genReqId`. The exact mechanism varies; the rule is that *every log line* in a request is taggable back to that one request.

```ts
@Injectable({ scope: Scope.REQUEST })
export class RequestLogger {
  constructor(@Inject(REQUEST) private readonly req: Request) {}
  private get ctx() {
    return { correlationId: this.req.headers['x-correlation-id'], userId: this.req.user?.id };
  }
  log(msg: string, extra?: object)   { logger.info({ ...this.ctx, ...extra }, msg); }
  warn(msg: string, extra?: object)  { logger.warn({ ...this.ctx, ...extra }, msg); }
  error(msg: string, extra?: object) { logger.error({ ...this.ctx, ...extra }, msg); }
  debug(msg: string, extra?: object) { logger.debug({ ...this.ctx, ...extra }, msg); }
}
```

---

## Required log points

For a typical service method:

| Position | Level | What to log |
|----------|-------|-------------|
| Start | `log` | Method name + key inputs (IDs, not payloads) |
| Before each repo call | `log` or `debug` | Operation + key params |
| After each repo call | `log` or `debug` | Result summary (count, ID, found/not-found) |
| Decision branches | `log` or `warn` | Which branch was taken + why |
| Recoverable error / fallback | `warn` | What failed + what fallback ran |
| Caught & re-thrown | `error` | The error + the cast type |
| End | `log` | Method name + outcome summary |

### Example: a well-logged method

```ts
async createUserWithProfile(dto: CreateUserDto): Promise<User> {
  this.log.log('createUserWithProfile start', { email: dto.email });

  try {
    this.log.log('createUserWithProfile: checking duplicates');
    const existing = await this.userRepo.findByEmail(dto.email);
    if (existing) {
      this.log.warn('createUserWithProfile: duplicate email', { email: dto.email });
      throw cast(ErrorTypes.CONFLICT, `User with email ${dto.email} already exists`);
    }

    this.log.log('createUserWithProfile: persisting user');
    const user = await this.userRepo.save(dto);
    this.log.log('createUserWithProfile: user persisted', { userId: user.id });

    this.log.log('createUserWithProfile: creating profile');
    await this.profileService.createForUser(user);
    this.log.log('createUserWithProfile: profile created', { userId: user.id });

    this.eventBus.emit('user.create', user);
    this.log.log('createUserWithProfile end', { userId: user.id, outcome: 'success' });
    return user;
  } catch (e) {
    this.log.error('createUserWithProfile failed', { email: dto.email, error: String(e) });
    throw e;
  }
}
```

Reward: +50 when this discipline is consistent across the module.

---

## Common violations

### ❌ Silent service method
```ts
async createUserWithProfile(dto: CreateUserDto): Promise<User> {
  const user = await this.userRepo.save(dto);
  await this.profileService.createForUser(user);
  return user;
}
```
Zero logs. -200.

### ❌ Only error path logged
```ts
async findById(id: string) {
  try {
    return await this.userRepo.findById(id);
  } catch (e) {
    this.log.error('failed', e); // only logs on failure
    throw e;
  }
}
```
Success paths are invisible. -200 for incomplete coverage.

### ❌ `console.log` instead of the request logger
```ts
console.log('user created', user); // -200
```
No correlation context. Doesn't ship to the log aggregator with structured fields.

### ❌ Logging full payloads (PII risk)
```ts
this.log.log('login attempt', { dto }); // dto contains password — -200 + serious privacy issue
```
Log IDs and outcomes, never raw credentials, tokens, or full PII payloads.

### ❌ Wrong level
```ts
this.log.error('user not found'); // -200: this is a normal NOT_FOUND, not an error
this.log.log('database connection lost'); // -200: this is at least warn, often error
```
Misleveled logs poison alerting.

### ❌ Logger imported as a global
```ts
import { Logger } from '@nestjs/common';
const log = new Logger('UserService'); // module-scoped, no request context — -200
```
Without request scope, you can't correlate logs to a request.

### ❌ Logging inside a tight loop without sampling
```ts
for (const item of items) {
  this.log.log('processing item', { item }); // -200 if items can be large
}
```
For batches > a few hundred, log start/end with a count and sample failures, not every iteration.

---

## Detection

For each service method in the diff:
- Count log statements. Zero ⇒ -200.
- Check if `start` and `end` are both logged.
- Check that repository calls are bracketed by logs (or at minimum, post-log).
- Check log levels: `error` for actual errors, not for expected control flow.
- Check that PII isn't being logged (passwords, tokens, full bodies).

```bash
# console.log usage in services (almost always wrong)
grep -rn "console\.\(log\|info\|warn\|error\)" src/ | grep -E "/services/|\.service\."

# Methods with no logging at all (rough heuristic — count logs per file)
for f in $(find src/services -name "*.service.impl.ts"); do
  count=$(grep -c "this\.\(log\|logger\|log\.\)" "$f")
  echo "$count $f"
done | sort -n
```

## Edge cases

- **Pure utility methods (no I/O)**: less strict. A `formatPhoneNumber(s)` helper doesn't need start/end logs.
- **Hot paths**: extremely high-throughput methods may need log-level tuning (move `start`/`end` to `debug`). Document the rationale.
- **Background jobs**: same rules apply, but the "request" context becomes the job context (job ID, attempt count).
- **Tests**: tests don't need to be logged at this rigor; the rule applies to production code.
- **Sensitive contexts**: in finance/health/CSR-sensitive operations, *more* logging is required, not less, but with strict PII redaction.

## How to write the finding

```
<file>:<line> — Rule #10: <method> has no start/end logging / no logs around repo calls / wrong log level. Add request-scoped logging at start, end, and around I/O boundaries. -200
```

Reward (per service):
```
<file> — Rule #10 reward: consistent request-scoped logging at start, end, around repo calls, with correct levels. +50
```
