# cf-admin — Authentication & Authorization

Zero Trust identity model, session lifecycle, RBAC, PLAC page-level access control, and force-logout cascade.

---

## Two-Layer Security Model

Every request must pass two independent gatekeepers:

| Layer | System | Question | Location |
|-------|--------|----------|----------|
| Identity | Cloudflare Zero Trust | "Are you who you say you are?" | CF Edge (before Worker runs) |
| Authorization | Supabase whitelist + D1 PLAC | "Are you allowed here?" | cf-admin Worker (middleware) |

Both must pass. Bypassing Zero Trust still leaves the Supabase whitelist check. Being in the Supabase whitelist does nothing without a valid CF Access JWT.

---

## CF Zero Trust Integration (`src/lib/auth/cloudflare-access.ts`, 180 lines)

### How It Works

1. User navigates to `secure.madagascarhotelags.com`
2. Cloudflare Access intercepts **before the Worker runs**
3. If no valid CF Access session: redirect to configured identity provider (Google / GitHub / OTP)
4. On successful IdP auth: CF issues a signed RS256 JWT and injects it as `CF-Access-JWT-Assertion` header
5. Worker receives the request with the JWT header

### JWT Verification (`verifyZeroTrustJwt`)

cf-admin re-verifies the JWT independently — it never trusts the `CF-Access-Email` forwarded header alone:

```typescript
// Claims verified:
{
  email: string                    // User email
  aud: string | string[]           // Must include CF_ACCESS_AUD
  iss: string                      // Must be "https://mascotas.cloudflareaccess.com"
  iat: number                      // Issued-at (Unix seconds)
  exp: number                      // Expiry (Unix seconds) — rejected if expired
  identity_nonce: string
  sub: string                      // CF Access user UUID → stored as cfSubId
  idp?: { id: string; type: string } // Identity provider info
}
```

**Verification steps:**
- Algorithm: RS256 (asymmetric — cannot be forged without CF's private key)
- Audience: Must include `CF_ACCESS_AUD` (`680d415033ba49284452da6f51bc8b7ee9aa23345ac95c53cf49761140b13088`)
- Issuer: Must be `https://mascotas.cloudflareaccess.com`
- Expiry: Rejected if expired
- Clock skew tolerance: ±60 seconds

**JWKS cache:**
- Public keys fetched from `https://mascotas.cloudflareaccess.com/cdn-cgi/access/certs`
- Cached in-memory with 1-hour TTL
- On verification failure (key rotation), cache is busted once and JWKS is re-fetched
- If bust also fails, the request is rejected (fail-secure)

### Login Method Mapping

| `idp.type` claim | Stored `loginMethod` |
|-----------------|----------------------|
| `google-oauth2` | `google` |
| `github` | `github` |
| `otp` | `otp` |

---

## Supabase Authorization Whitelist

After JWT verification, the Worker queries Supabase `admin_authorized_users`:

```sql
SELECT id, role, is_active, display_name, cf_sub_id
FROM admin_authorized_users
WHERE email = $1;
```

| Result | Action |
|--------|--------|
| Row not found | `LOGIN_BLOCKED` forensics event → 403 Access Denied |
| `is_active = 0` | `LOGIN_BLOCKED` forensics event → 403 Access Denied |
| `is_active = 1` | `LOGIN_SUCCESS` forensics event → create KV session |

No self-registration. Every admin must be manually added by a `dev` or `owner`.

---

## Session Management (`src/lib/auth/session.ts`, 254 lines)

### KV Storage Keys

| Key | Value | TTL |
|-----|-------|-----|
| `session:{sessionId}` | `AdminSession` JSON | 86400s (24 hours) |
| `user-session:{userId}:{sessionId}` | `'1'` (reverse index) | 86400s |
| `revoked:{userId}` | `'1'` | 86400s (24 hours) |
| `cf-audit-last-synced` | ISO timestamp | no TTL |

### AdminSession Interface

```typescript
interface AdminSession {
  userId: string             // D1 admin_authorized_users UUID
  cfSubId: string            // CF Access user UUID (needed for Layer 3 force-logout)
  email: string              // Lowercase authenticated email
  displayName: string        // From admin_authorized_users.display_name
  role: 'dev' | 'owner' | 'super_admin' | 'admin' | 'staff'
  loginMethod: 'google' | 'github' | 'otp'
  createdAt: number          // Unix ms (hard expiry anchor)
  lastRoleCheckedAt: number  // Unix ms (30-min staleness guard)
  accessMap?: PageAccessMap  // Pre-computed PLAC (avoids extra KV read per request)
  auditSilenced?: boolean    // DEV-only: suppress audit writes during testing
}
```

### Timing Rules

| Rule | Value | Config var |
|------|-------|------------|
| Hard session expiry | 24 hours | `SESSION_MAX_LIFETIME_MS = 86400000` |
| KV TTL on write | 86400s | Matches max lifetime |
| Role re-check interval | 30 minutes | `SESSION_REFRESH_INTERVAL_MS = 1800000` |
| PLAC access map refresh | 60 minutes | Hardcoded in middleware |

### Session Cookie

| Property | Production | Local Dev |
|----------|------------|-----------|
| Name | `__Host-admin_session` | `admin_session` |
| HttpOnly | true | true |
| Secure | true | false |
| SameSite | lax | lax |
| MaxAge | 86400 (24 hours) | 86400 |

### Session Lifecycle

**Creation (first login):**
1. CF Zero Trust JWT verified
2. Supabase whitelist check passes
3. PLAC `accessMap` computed from D1 (single JOIN query)
4. `AdminSession` written to KV: `session:{sessionId}` (TTL 86400s)
5. Reverse index written: `user-session:{userId}:{sessionId}` (TTL 86400s)
6. `cfSubId` persisted to D1 `admin_authorized_users.cf_sub_id`
7. Session cookie set in response

**Per-request (subsequent requests):**
1. Read `sessionId` from `__Host-admin_session` cookie
2. KV GET `session:{sessionId}` — O(1), ~2–5ms
3. Check `createdAt`: if > 24 hours → destroy and redirect to login
4. Check `revoked:{userId}`: if present → force logout immediately
5. Check `lastRoleCheckedAt`: if > 30 min → async Supabase role re-check
6. Check `accessMap` timestamp: if > 60 min → async D1 PLAC recompute

**Logout (`GET /api/auth/logout` or `POST /api/auth/logout`):**
1. Delete `session:{sessionId}` from KV
2. Clear cookie in response
3. Redirect to CF Access logout URL

---

## Role-Based Access Control — RBAC (`src/lib/auth/rbac.ts`, 105 lines)

### 5-Tier Hierarchy

Lower level number = higher privilege. `hasPermission()` returns `true` when `ROLE_LEVEL[userRole] <= ROLE_LEVEL[requiredRole]`.

```typescript
const ROLE_LEVEL = {
  dev:         0,   // System admin — debugging, feature flags, audit silence
  owner:       1,   // Business owner — financial, user management
  super_admin: 2,   // All modules except owner-only finance
  admin:       3,   // Content, bookings, logs
  staff:       4,   // Limited dashboard access
}

function hasPermission(userRole: Role, requiredRole: Role): boolean {
  return ROLE_LEVEL[userRole] <= ROLE_LEVEL[requiredRole]
}
```

### Role Display Metadata

| Role | Label | Color | Icon |
|------|-------|-------|------|
| `dev` | Developer | `#ef4444` (red) | lightning bolt |
| `owner` | Owner | `#10b981` (emerald) | gem |
| `super_admin` | Super Admin | `#f59e0b` (amber) | crown |
| `admin` | Admin | `#8b5cf6` (purple) | shield |
| `staff` | Staff | `#3b82f6` (blue) | person |

### Hidden Accounts

Users with `is_hidden = true` in `admin_authorized_users` are invisible to actors with a lower privilege level. DEV and OWNER accounts are typically hidden to prevent lower-tier admins from accidentally modifying or deactivating them.

Visibility rule: a user can only see accounts where `target.level >= viewer.level`.

### BREAK_GLASS_EMAILS

Hardcoded in `src/lib/auth/rbac.ts`. These email addresses retain `dev` access even if the database is compromised or the `admin_authorized_users` table is corrupted. Used as last-resort recovery — not for normal operations.

---

## Page-Level Access Control — PLAC (`src/lib/auth/plac.ts`, 390 lines)

RBAC controls what operations a user can perform. PLAC controls which pages they can navigate to. Both are enforced independently (defense in depth).

### Resolution Rule (Deny Wins)

For each page path, evaluated in this order:

1. If explicit deny override (`granted = 0`) → **FALSE** — deny always wins, even if the role would normally allow access
2. Else if explicit grant override (`granted = 1`) → **TRUE** — grants access below the role's minimum
3. Else role hierarchy: `ROLE_LEVEL[userRole] <= ROLE_LEVEL[page.required_role]` → **TRUE/FALSE**

This means a `super_admin` can be explicitly denied a specific page, and a `staff` member can be explicitly granted access to a specific page.

### D1 Schema

```sql
-- Page registry
CREATE TABLE admin_pages (
  path TEXT PRIMARY KEY,
  label TEXT,
  icon TEXT,
  description TEXT,
  required_role TEXT CHECK(required_role IN ('dev','owner','super_admin','admin','staff')),
  sort_order INTEGER,
  is_active INTEGER DEFAULT 1
);

-- Per-user access exceptions
CREATE TABLE admin_page_overrides (
  user_id TEXT REFERENCES admin_authorized_users(id),
  page_path TEXT REFERENCES admin_pages(path),
  granted INTEGER,          -- 0=explicit deny, 1=explicit grant, NULL=no override
  granted_by TEXT REFERENCES admin_authorized_users(id),
  granted_by_email TEXT,
  reason TEXT,
  created_at TEXT,
  updated_at TEXT,
  PRIMARY KEY (user_id, page_path)
);
```

### Access Map Computation

Single D1 JOIN query (~2ms):

```sql
SELECT p.path, p.required_role, o.granted
FROM admin_pages p
LEFT JOIN admin_page_overrides o
  ON o.page_path = p.path AND o.user_id = ?
WHERE p.is_active = 1
```

Result is stored in `AdminSession.accessMap` as a `Record<string, boolean>` (path → allowed).

### Path Matching

- Supports exact match (`/dashboard/users`) and prefix match (`/dashboard/content`)
- `/dashboard/content/gallery` matches against `/dashboard/content` prefix
- Normalized paths (trailing slashes stripped, lowercased)
- O(1) hashmap lookup at enforcement time

### Sidebar Filtering

The `Sidebar.tsx` component reads `accessMap` from the session and renders only pages where `accessMap.pages[path] === true`. This is a UI convenience — the middleware enforces access independently.

---

## Force-Logout Cascade (3 Layers)

Triggered by `POST /api/users/force-kick` (min role: `owner`):

### Layer 1 — KV Session Deletion (immediate, <1 second)

```typescript
// Find all sessions for this user via reverse index
// user-session:{userId}:* pattern scan
// Delete each session:{sessionId}
// Write revoked:{userId} = '1' (24h TTL)
```

The revocation flag blocks CF Zero Trust re-bootstrap even if the user's CF Access session is still valid.

### Layer 2 — KV Revocation Flag (persistent, 24-hour block)

The `revoked:{userId}` KV key is checked at Layer 6 (CF Zero Trust Bootstrap). If present, the middleware refuses to create a new session even for a user with a valid CF Access JWT.

### Layer 3 — CF API Session Revocation

```typescript
// DELETE /accounts/{CF_ACCOUNT_ID}/access/users/{cfSubId}/active_sessions
// Authorization: Bearer {CF_API_TOKEN_ZT_WRITE}
```

Revokes the CF Access session itself — the user can no longer even reach the cf-admin Worker. Requires `CF_API_TOKEN_ZT_WRITE` secret with `Access: Revoke Tokens` scope.

### After Force-Logout

The user's `is_active` may also be set to `0` in `admin_authorized_users` (via `DELETE /api/users/manage`), which permanently blocks all future logins until a `dev` or `owner` re-activates the account.

---

## Local Development Bypass

CF Zero Trust is not available locally. The bypass activates when `SITE_URL` contains `localhost`:

```typescript
// src/lib/auth/guard.ts
function isLocalDev(env: Env): boolean {
  return (env.SITE_URL ?? '').includes('localhost')
}
```

**Behavior when `isLocalDev()` is true:**
- `POST /api/auth/dev-login` (GET/HEAD, public allowlist) auto-bootstraps a session
- Session email: `LOCAL_DEV_ADMIN_EMAIL` (`harshil.8136@gmail.com` from `wrangler.toml [vars]`)
- Session role: `dev` (full local access)
- No JWT verification, no Supabase lookup

**Fail-secure invariant:** If `SITE_URL` is missing or empty, `isLocalDev()` returns `false` — defaults to production mode. The system never accidentally activates dev bypass in production.

`LOCAL_DEV_ADMIN_EMAIL` is stored in `[vars]` (not a secret) because it is not sensitive — it is only functional in local dev mode.
