# Rule 8 — Adapter Design Pattern

> **Penalty: -50** · **Reward: +50**

## The rule

Any third-party interaction must go through an **adapter**.

- Define an **interface or abstract class** of the adapter where every type is well-defined (params, return values).
- Implement these as specific adapter implementations called **providers**.
- Each provider that uses HTTP must document its own request/response types.

## Why it matters

- **Vendor lock-in prevention**: swapping SendGrid for Postmark is a new provider class, not a refactor of every service that sends email.
- **Mockability**: the abstract adapter mocks cleanly in tests. Without it, you're stubbing HTTP libraries.
- **Type safety at the boundary**: third-party APIs return loose JSON. The adapter is the place where loose external types are converted into well-typed internal models.
- **Auditable surface**: every external call site is one of N provider classes. You can grep for `extends EmailAdapter` and find every email-sending point.
- **Failure isolation**: retries, circuit breakers, and timeout handling live in the adapter. Services don't have to know the third party can flake.

The rule applies to:
- HTTP APIs (REST, GraphQL, gRPC)
- SDKs (Stripe, Twilio, AWS SDK, Sendbird, SendGrid)
- Message queues (RabbitMQ, SQS, Kafka)
- File storage (S3, GCS)
- Auth providers (Auth0, Cognito)
- LLM APIs (OpenAI, Anthropic, Bedrock)
- Anything that crosses a process boundary you don't own

The rule does NOT apply to:
- Internal databases (those go through repositories — rule #2)
- Internal services in the same process (those are services or strategies)
- Filesystem I/O within your own VM (unless it crosses to a managed service like S3)

---

## The pattern

### Abstract adapter

```ts
// adapters/email.adapter.ts
export interface EmailMessage {
  to: string;
  from: string;
  subject: string;
  body: string;
  attachments?: EmailAttachment[];
}

export interface EmailSendResult {
  messageId: string;
  acceptedAt: Date;
}

export abstract class EmailAdapter {
  abstract send(message: EmailMessage): Promise<EmailSendResult>;
  abstract sendBatch(messages: EmailMessage[]): Promise<EmailSendResult[]>;
}
```

### Provider implementation

```ts
// adapters/sendgrid/sendgrid.email.provider.ts
@Injectable()
export class SendgridEmailProvider extends EmailAdapter {
  constructor(
    private readonly http: HttpService,
    @Inject(SENDGRID_CONFIG) private readonly config: SendgridConfig,
  ) {
    super();
  }

  async send(message: EmailMessage): Promise<EmailSendResult> {
    const request = this.toSendgridRequest(message);
    const response = await this.http.post<SendgridSendResponse>(
      'https://api.sendgrid.com/v3/mail/send',
      request,
      { headers: { Authorization: `Bearer ${this.config.apiKey}` } },
    );
    return this.toSendResult(response.data);
  }

  async sendBatch(messages: EmailMessage[]): Promise<EmailSendResult[]> {
    return Promise.all(messages.map((m) => this.send(m)));
  }

  private toSendgridRequest(m: EmailMessage): SendgridSendRequest { /* ... */ }
  private toSendResult(r: SendgridSendResponse): EmailSendResult { /* ... */ }
}
```

### Provider-specific request/response types

These are **required** for HTTP providers and must live alongside the provider:

```ts
// adapters/sendgrid/sendgrid.types.ts
export interface SendgridSendRequest {
  personalizations: Array<{
    to: Array<{ email: string; name?: string }>;
    subject?: string;
  }>;
  from: { email: string; name?: string };
  content: Array<{ type: 'text/plain' | 'text/html'; value: string }>;
  attachments?: Array<{
    content: string; // base64
    type: string;
    filename: string;
  }>;
}

export interface SendgridSendResponse {
  message_id: string; // from response header X-Message-Id
}
```

The internal `EmailMessage` type stays clean; the SendGrid-specific shape is contained.

### Module registration

```ts
@Module({
  providers: [
    {
      provide: EmailAdapter,
      useClass: SendgridEmailProvider,
    },
  ],
  exports: [EmailAdapter],
})
export class EmailModule {}
```

Consumer injects `EmailAdapter`, never `SendgridEmailProvider`. Reward: +50.

---

## Naming and file layout

| Element | Convention | Example |
|---------|-----------|---------|
| Abstract | `<Domain>Adapter` | `EmailAdapter` |
| File | `<domain>.adapter.ts` | `email.adapter.ts` |
| Provider class | `<Vendor><Domain>Provider` | `SendgridEmailProvider` |
| Provider folder | `adapters/<vendor>/` | `adapters/sendgrid/` |
| Provider file | `<vendor>.<domain>.provider.ts` | `sendgrid.email.provider.ts` |
| Vendor types | `<vendor>.types.ts` | `sendgrid.types.ts` |

---

## Common violations

### ❌ Direct SDK call inside a service

```ts
@Injectable()
export class UserServiceImpl extends UserService {
  async sendWelcomeEmail(user: User) {
    const sgMail = require('@sendgrid/mail');                    // -50
    sgMail.setApiKey(process.env.SENDGRID_API_KEY);
    await sgMail.send({ to: user.email, from: ..., subject: ..., text: ... });
  }
}
```
The service is now bound to SendGrid and to the SendGrid SDK's specific shape.

### ❌ Provider injected without an abstract

```ts
constructor(private sendgrid: SendgridEmailProvider) {} // -50
```
You have an adapter, but the consumer is still bound to the concrete provider. Inject `EmailAdapter`.

### ❌ HTTP provider without typed request/response

```ts
async send(message: EmailMessage): Promise<EmailSendResult> {
  const response: any = await this.http.post(...); // -50: response untyped
  return { messageId: response.id, acceptedAt: new Date() };
}
```
The provider must declare types for both directions.

### ❌ Adapter that leaks vendor-specific fields

```ts
export interface EmailMessage {
  to: string;
  sendgridTemplateId?: string;  // -50: vendor concept in abstract
  twilioAccountSid?: string;
}
```
The abstract should expose internal concepts only. If you need template support generically, make `template: { id: string; data: Record<string, unknown> }` and let each provider map it.

### ❌ Multiple services calling the SDK directly

If three services all `import * as Stripe from 'stripe'` and call methods, that's three -50s and a strong signal you need a `PaymentAdapter` immediately.

### ❌ "Adapter" that's just a thin wrapper with no abstract

```ts
@Injectable()
export class EmailService {
  async send(...) { return sgMail.send(...); }
}
```
This is a wrapper, not an adapter. No abstract, no contract, no swap-ability. -50.

---

## Detection

```bash
# Direct SDK imports outside adapter folders
grep -rn "require.*['\"]\(@sendgrid\|@stripe\|@aws-sdk\|twilio\|axios\)" src/ \
  | grep -v "/adapters/"

grep -rn "from ['\"]@\(sendgrid\|stripe\|aws-sdk\|twilio\)" src/ \
  | grep -v "/adapters/"

# HTTP calls outside adapter folders
grep -rn "this\.http\.\(get\|post\|put\|delete\)" src/ \
  | grep -v "/adapters/"
```

Each match is a -50 candidate. Apply once per service that violates (don't stack per call site within the same service).

## Edge cases

- **Internal microservices**: calling another microservice you own is borderline. If it's via HTTP and the contract may change, treat it like a third party — adapter required. If it's via a typed shared SDK package you also own, the SDK is the adapter.
- **AWS SDK**: a thin `S3Adapter`, `SqsAdapter`, etc. is still required. The abstraction lets you mock without `aws-sdk-mock` and lets you swap clouds.
- **gRPC clients**: the generated client counts as an SDK. Wrap it in an adapter.
- **Multi-vendor support**: `EmailAdapter` with both `SendgridEmailProvider` and `PostmarkEmailProvider` registered behind a strategy (rule #7) selects which to use. Acceptable and even encouraged.
- **Webhooks coming in (inbound)**: not subject to this rule. Inbound HTTP is a controller. The adapter rule is for *outbound* third-party interaction.

## How to write the finding

Violation:
```
<file>:<line> — Rule #8: <vendor> SDK/HTTP called directly. Wrap in <Domain>Adapter abstract + <Vendor><Domain>Provider implementation under adapters/. -50
```

Reward:
```
<module>.module.ts — Rule #8 reward: <Domain>Adapter with typed <Vendor><Domain>Provider and request/response types documented. +50
```
