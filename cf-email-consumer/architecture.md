# cf-email-consumer — Architecture

> Queue processing logic, Resend integration, and Supabase audit logging.

---

## Queue Processing Architecture

The `cf-email-consumer` worker does not expose HTTP endpoints. It relies exclusively on the `queue` handler triggered by Cloudflare.

```mermaid
graph TD
    A[Producer: cf-astro] -->|env.EMAIL_QUEUE.send()| Q[(Cloudflare Queue:\nmadagascar-emails)]
    B[Producer: cf-admin] -->|env.EMAIL_QUEUE.send()| Q
    
    Q -->|Triggers| C[cf-email-consumer]
    
    subgraph "Consumer Worker"
        C --> D[Zod Validation]
        D -->|Invalid| E[Log Error & Discard]
        D -->|Valid| F[Process Batch]
        
        F --> G1{Message 1}
        F --> G2{Message 2}
        
        G1 --> H1[Generate HTML Template]
        H1 --> I1[Call Resend API]
        I1 --> J1[Update Supabase Audit Log]
    end

    I1 -->|Success| K[Mark Msg Ack'd]
    I1 -->|Failure| L[Mark Msg Retry]
```

---

## Queue Configuration

Defined in `wrangler.toml`:

```toml
[[queues.consumers]]
queue = "madagascar-emails"
max_batch_size = 10     # Process up to 10 emails simultaneously
max_batch_timeout = 5   # Or wait 5 seconds to gather messages
max_retries = 3         # Retry failed messages up to 3 times
```

- **Batching**: Reduces function invocations and allows efficient connection pooling to Supabase.
- **Retries**: Transient errors (e.g., Resend 5xx, network timeouts) trigger automatic retries.

---

## Message Schema Validation

Every incoming message is strictly validated using Zod to prevent malformed data from crashing the consumer.

```typescript
// schema.ts
export const EmailMessageSchema = z.object({
  type: z.enum(['booking_admin', 'booking_customer', 'contact', 'arco']),
  trackingId: z.string().uuid(),
  payload: z.any(), // Structure varies by type
  sentryTrace: z.string().optional(),
  sentryBaggage: z.string().optional(),
});
```

---

## Resend Integration

Emails are dispatched using the official Resend SDK.

```typescript
// dispatcher.ts
import { Resend } from 'resend';

const resend = new Resend(env.RESEND_API_KEY);

const { data, error } = await resend.emails.send({
  from: 'Madagascar Pet Hotel <no-reply@madagascarhotelags.com>',
  to: recipientEmail,
  subject: emailSubject,
  html: emailHtml,
  reply_to: 'info@madagascarhotelags.com'
});
```

### Domain Configuration
- Sender domain must be verified in Resend.
- SPF, DKIM, and DMARC records must be configured in Cloudflare DNS for `madagascarhotelags.com` to ensure high deliverability and avoid spam filters.

---

## Database Audit Logging

To maintain a durable record of communications, the consumer updates the `email_audit_log` table in Supabase via Drizzle ORM.

### Initial State
When a producer (like `cf-astro`) sends a queue message, it *first* inserts a row into `email_audit_log` with status `queued` and generates a `tracking_id`.

### Consumer Updates
Upon processing, the consumer updates that row based on the Resend API result.

```typescript
// audit.ts
export async function updateAuditLog(
  db: PostgresJsDatabase, 
  trackingId: string, 
  status: 'sent' | 'failed', 
  details?: { messageId?: string, error?: string }
) {
  await db.update(email_audit_log)
    .set({
      status,
      resend_message_id: details?.messageId,
      error_message: details?.error,
      attempts: sql`${email_audit_log.attempts} + 1`,
      sent_at: status === 'sent' ? sql`now()` : null
    })
    .where(eq(email_audit_log.tracking_id, trackingId));
}
```

### Resilient Logging Pattern (`safeDbUpdate`)
Updating the audit log must never cause an email to fail. If the Resend API succeeds but the Supabase update fails, the message should *not* be retried (which would result in duplicate emails).

```typescript
// The safe wrapper
try {
  await updateAuditLog(db, trackingId, 'sent', { messageId: data.id });
} catch (dbError) {
  // Log the DB error to Sentry, but DO NOT throw.
  // We must acknowledge the queue message since the email was sent.
  console.error('Audit update failed, but email was sent:', dbError);
  Sentry.captureException(dbError);
}
```

---

## Distributed Tracing (Sentry)

To trace a request from the initial user action (e.g., clicking "Book") all the way through the queue to the email delivery, `cf-email-consumer` implements distributed tracing.

Producers attach `sentryTrace` and `sentryBaggage` to the queue payload. The consumer resumes this trace.

```typescript
// index.ts
import * as Sentry from '@sentry/cloudflare';

async queue(batch: MessageBatch<any>, env: Env, ctx: ExecutionContext) {
  for (const message of batch.messages) {
    const { sentryTrace, sentryBaggage, type } = message.body;

    await Sentry.continueTrace({ sentryTrace, baggage: sentryBaggage }, async () => {
      await Sentry.startSpan({ name: `email.${type}`, op: 'queue.process' }, async () => {
        
        // Process message...
        
      });
    });
  }
}
```
This produces a unified flamegraph in Sentry across service boundaries.
