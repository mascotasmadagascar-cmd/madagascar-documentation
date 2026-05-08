# cf-astro — Database Schema

> Drizzle ORM schema, Supabase tables, and migration strategy.

---

## ORM: Drizzle

cf-astro uses **Drizzle ORM** with the `postgres.js` driver for type-safe database access.

### Connection Configuration

```typescript
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

const client = postgres(env.DATABASE_URL, {
  ssl: 'require',
  max: 1,              // 1 connection per V8 isolate
  idle_timeout: 20,    // Close idle connections after 20s
});

export const db = drizzle(client);
```

---

## Core Tables

### bookings
```sql
CREATE TABLE bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  guest_name TEXT NOT NULL,
  email TEXT NOT NULL,
  phone TEXT NOT NULL,
  check_in DATE NOT NULL,
  check_out DATE NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',  -- pending, confirmed, checked_in, completed, cancelled
  special_requests TEXT,
  booking_ref TEXT UNIQUE NOT NULL,        -- MDG-2026-0042 format
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);
```

### pets
```sql
CREATE TABLE pets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID REFERENCES bookings(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  species TEXT NOT NULL,           -- 'dog', 'cat', 'bird', 'rabbit', etc.
  breed TEXT,
  weight NUMERIC,                  -- kg
  age INTEGER,                     -- years
  special_needs TEXT,
  vaccination_status TEXT,         -- 'up_to_date', 'pending', 'expired'
  created_at TIMESTAMPTZ DEFAULT now()
);
```

### booking_consents
```sql
CREATE TABLE booking_consents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID REFERENCES bookings(id) ON DELETE CASCADE,
  consent_type TEXT NOT NULL,       -- 'terms', 'privacy', 'marketing_email'
  consent_version TEXT NOT NULL,    -- '2024-01'
  ip_hash TEXT NOT NULL,            -- SHA-256(IP + salt)
  user_agent TEXT,
  granted_at TIMESTAMPTZ DEFAULT now(),
  revoked_at TIMESTAMPTZ,
  revocation_method TEXT
);
```

### email_audit_log
```sql
CREATE TABLE email_audit_log (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tracking_id UUID NOT NULL,        -- Links to Queue message
  booking_id UUID REFERENCES bookings(id),
  email_type TEXT NOT NULL,         -- 'booking_admin', 'booking_customer', 'contact', 'arco'
  recipient_email TEXT NOT NULL,
  status TEXT DEFAULT 'queued',     -- queued, sent, delivered, bounced, failed
  attempts INTEGER DEFAULT 0,
  resend_message_id TEXT,           -- Resend API message ID
  sent_at TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);
```

---

## D1 Tables (Edge)

### admin_feature_flags
```sql
CREATE TABLE admin_feature_flags (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  flag_key TEXT UNIQUE NOT NULL,
  is_enabled INTEGER NOT NULL DEFAULT 0,  -- SQLite boolean
  description TEXT,
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);
```

Used by middleware for runtime feature toggles. Cached in 3-layer system (memory → CF Cache API → D1).

---

## Migration Strategy

### Supabase (Drizzle Kit)
```bash
# Generate migration from schema changes
npx drizzle-kit generate

# Push schema to Supabase
npx drizzle-kit push

# Introspect existing database
npx drizzle-kit introspect
```

### D1 (Wrangler)
```bash
# Apply SQL migration file
wrangler d1 execute madagascar-db --remote --file=migrations/001_feature_flags.sql

# Execute inline SQL
wrangler d1 execute madagascar-db --remote --command "INSERT INTO admin_feature_flags ..."
```

---

## Indexes

```sql
-- Bookings
CREATE INDEX idx_bookings_status ON bookings(status);
CREATE INDEX idx_bookings_check_in ON bookings(check_in);
CREATE INDEX idx_bookings_email ON bookings(email);
CREATE INDEX idx_bookings_ref ON bookings(booking_ref);

-- Email audit
CREATE INDEX idx_email_audit_tracking ON email_audit_log(tracking_id);
CREATE INDEX idx_email_audit_booking ON email_audit_log(booking_id);
CREATE INDEX idx_email_audit_status ON email_audit_log(status);

-- Pets
CREATE INDEX idx_pets_booking ON pets(booking_id);
```
