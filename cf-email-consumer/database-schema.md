# cf-email-consumer — Database Schema

> Drizzle ORM schema for the email audit log.

---

## ORM: Drizzle

`cf-email-consumer` uses Drizzle ORM to update the `email_audit_log` table in Supabase. It uses the `postgres.js` driver.

### Connection Configuration

```typescript
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

export function getDb(connectionString: string) {
  // Max 1 connection per isolate, idle timeout 20s
  const client = postgres(connectionString, { max: 1, idle_timeout: 20 });
  return drizzle(client);
}
```

---

## The Audit Table

The `email_audit_log` table is initially populated by the message producers (`cf-astro` or `cf-admin`) right before the message is pushed to the Queue.

### Schema Definition

```sql
CREATE TABLE email_audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tracking_id UUID UNIQUE NOT NULL,      -- Corresponds to the Queue message trackingId
  booking_id UUID REFERENCES bookings(id), -- Optional: Link to a specific booking
  email_type TEXT NOT NULL,              -- 'booking_admin', 'contact', etc.
  recipient_email TEXT NOT NULL,         
  status TEXT DEFAULT 'queued',          -- 'queued', 'sent', 'failed'
  attempts INTEGER DEFAULT 0,            -- Number of times consumer tried to send
  resend_message_id TEXT,                -- The ID returned by the Resend API
  sent_at TIMESTAMPTZ,                   -- Timestamp of successful dispatch
  error_message TEXT,                    -- Stack trace or API error message if failed
  created_at TIMESTAMPTZ DEFAULT now()   -- When the producer enqueued it
);
```

### Indexes

```sql
CREATE INDEX idx_email_audit_tracking ON email_audit_log(tracking_id);
CREATE INDEX idx_email_audit_status ON email_audit_log(status);
```

---

## The Logging Lifecycle

1. **Producer Side (cf-astro)**
   ```typescript
   // 1. Insert row as 'queued'
   const trackingId = crypto.randomUUID();
   await db.insert(email_audit_log).values({
     tracking_id: trackingId,
     email_type: 'contact',
     recipient_email: 'admin@madagascar.com'
   });

   // 2. Send message to queue
   await env.EMAIL_QUEUE.send({
     type: 'contact',
     trackingId: trackingId,
     payload: { ... }
   });
   ```

2. **Consumer Side (Success)**
   ```typescript
   // Update row to 'sent'
   await db.update(email_audit_log)
     .set({ 
       status: 'sent', 
       resend_message_id: resendResponse.id,
       attempts: sql`attempts + 1`,
       sent_at: sql`now()`
     })
     .where(eq(email_audit_log.tracking_id, message.trackingId));
   ```

3. **Consumer Side (Failure)**
   ```typescript
   // Update row to 'failed'
   await db.update(email_audit_log)
     .set({ 
       status: 'failed', 
       error_message: error.message,
       attempts: sql`attempts + 1`
     })
     .where(eq(email_audit_log.tracking_id, message.trackingId));
   ```

---

## Migration Strategy

Because this table is part of the central Supabase database, schema changes are managed centrally via the `cf-astro` Drizzle setup. The consumer simply uses the shared table structure.

If the schema is updated in `cf-astro`:
1. Copy the updated schema definition to `cf-email-consumer/src/db/schema.ts`.
2. Redeploy the consumer so its ORM types match the new database schema.
