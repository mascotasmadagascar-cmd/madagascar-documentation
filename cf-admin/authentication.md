# cf-admin — Authentication & Authorization

> Zero Trust identity model, session lifecycle, RBAC, and PLAC page-level access control.

---

## Overview: Two-Layer Security Model

cf-admin uses **two independent gatekeepers**. Both must pass for a user to access any dashboard page:

| Layer | System | Question Answered | Where |
|-------|--------|------------------|-------|
| **Identity** | Cloudflare Zero Trust | "Are you who you say you are?" | CF Edge (before the Worker ever runs) |
| **Authorization** | Supabase whitelist | "Are you allowed here?" | Inside the cf-admin Worker |

This separation means: even if an attacker somehow bypasses Zero Trust, they still cannot access any data unless their email is in Supabase's `admin_authorized_users` table. Conversely, being in the Supabase table does nothing unless they've authenticated via an approved identity provider.

---

## Layer 1: Cloudflare Zero Trust (Identity)

### How It Works

1. Admin navigates to `secure.madagascarhotelags.com`
2. Cloudflare Access intercepts the request **before it reaches any Worker code**
3. If no valid CF-Access-JWT exists:
   - User is redirected to the configured identity provider (Google or GitHub)
   - User authenticates with their account
   - Cloudflare issues a signed JWT (RS256) and sets it as a cookie
4. Cloudflare forwards the request to cf-admin with two injected headers:
   - `CF-Access-JWT-Assertion`: The signed JWT
   - `CF-Access-Email`: The authenticated email address

### JWT Verification (`verifyZeroTrustJwt`)

cf-admin re-verifies the JWT independently — it does not trust the forwarded email header alone:

```typescript
// src/lib/auth/cloudflare-access.ts
const verified = await verifyZeroTrustJwt(jwt, {
  audience: env.CF_ACCESS_AUD,         // Application audience tag from CF dashboard
  issuer: `https://${env.CF_TEAM_NAME}.cloudflareaccess.com`,
  // JWK fetched from: {issuer}/cdn-cgi/access/certs
});
```

Checks performed:
- **Algorithm**: RS256 (asymmetric — cannot be forged without CF's private key)
- **Audience**: Must match `CF_ACCESS_AUD` env var exactly
- **Issuer**: Must be the hotel's CF team subdomain
- **Expiry**: Rejected if expired (CF tokens are short-lived)
- **JWK source**: Public keys fetched from CF's JWKS endpoint (cached)

---

## Layer 2: Supabase Authorization Whitelist

After JWT verification, the Worker queries Supabase:

```sql
SELECT role, is_active, is_hidden
FROM admin_authorized_users
WHERE email = $1;
```

| Result | Action |
|--------|--------|
| Row not found | `LOGIN_FAILED` logged → 403 Access Denied |
| `is_active = false` | `LOGIN_FAILED` logged → 403 Access Denied |
| `is_active = true` | Proceed to session creation |

This whitelist means **no self-registration is possible**. Every admin user must be manually added to Supabase by a DEV or Owner before they can log in.

---

## Session Lifecycle

### Session Creation

On first successful authentication, a KV session is created:

```typescript
interface KVSession {
  userId: string;           // Supabase user UUID
  cfSubId: string;          // CF Access subject ID (permanent user identifier)
  email: string;
  displayName: string;
  role: Role;               // 'dev' | 'owner' | 'superadmin' | 'admin' | 'staff'
  loginMethod: string;      // 'google' | 'github' | 'otp'
  accessMap: PLACMap;       // Computed page-level permissions
  createdAt: number;        // Session start (Unix ms)
  lastRoleCheckedAt: number; // Last Supabase role freshness check
  auditSilenced?: boolean;  // DEV-only: suppress audit noise during testing
}
```

- **KV key**: `session:{sessionId}` (sessionId is a crypto.randomUUID())
- **TTL**: 24 hours — hard expiry, non-renewable (user must re-authenticate)
- **Cookie**: `HttpOnly; Secure; SameSite=Strict; Path=/`

### Session Read (Subsequent Requests)

Every request after login:
1. Read `session_id` from cookie
2. KV GET `session:{sessionId}` → parse session
3. Check `createdAt` — if >24 hours, destroy and redirect to login
4. Check `lastRoleCheckedAt` — if >30 minutes, re-fetch role from Supabase
5. Check `accessMap.computedAt` — if >60 minutes, recompute PLAC map from D1
6. Check `revoked:{userId}` KV key — if present, force logout immediately

**Per-request overhead**: ~1 KV read (~1ms) + conditional Supabase query (every 30 min only).

### Session Destruction (Logout)

```typescript
// src/pages/api/auth/logout.ts
await env.SESSION.delete(`session:${sessionId}`);
// Cookie cleared in response
```

---

## Role-Based Access Control (RBAC)

### 5-Tier Hierarchy

Roles are hierarchical — each role inherits all permissions of roles below it:

| Role | Level | Who | Key Permissions |
|------|-------|-----|----------------|
| `dev` | 5 (highest) | Developer | Everything — all pages, all operations, debug tools, feature flags |
| `owner` | 4 | Hotel owner | All operations except system-level developer tools |
| `superadmin` | 3 | Senior manager | User management, full CMS, all bookings, chatbot config |
| `admin` | 2 | Staff manager | Booking management, basic CMS, limited user view |
| `staff` | 1 (lowest) | Front desk | Booking view only, no CMS or user management |

### Role Check Function

```typescript
// src/lib/auth/rbac.ts
export function hasRole(userRole: Role, requiredRole: Role): boolean {
  const hierarchy: Role[] = ['staff', 'admin', 'superadmin', 'owner', 'dev'];
  return hierarchy.indexOf(userRole) >= hierarchy.indexOf(requiredRole);
}
```

Usage in API routes:
```typescript
if (!hasRole(session.role, 'admin')) {
  return jsonError(403, 'Insufficient permissions');
}
```

### Hidden Accounts (Ghost Protection)

DEV and Owner accounts can be marked `is_hidden = true` in Supabase. Hidden accounts:
- Do not appear in the User Registry dashboard visible to lower-tier admins
- Cannot be modified by non-hidden-account users
- Protect the break-glass accounts from accidental deactivation

`BREAK_GLASS_EMAILS` are hardcoded in `src/lib/auth/rbac.ts` as a last-resort recovery mechanism.

---

## Page-Level Access Control (PLAC)

RBAC controls what operations a user can perform. PLAC controls **which pages they can see in the sidebar and navigate to**.

### How PLAC Works

1. D1 `admin_pages` table defines all dashboard pages with required minimum role
2. On login (or every 60 minutes), the Worker computes a `PLACMap` for the user:
   ```typescript
   interface PLACMap {
     role: Role;
     pages: Record<string, boolean>;  // '/dashboard/users' → true/false
     computedAt: number;
   }
   ```
3. The PLACMap is stored in the KV session (single read per request)
4. Sidebar dynamically renders only pages where `pages[path] === true`
5. API routes additionally check RBAC (defense in depth — PLAC alone is UI-only)

### Fail-Secure Behavior

If D1 is unreachable when computing PLAC:
- The stale PLACMap from the session is extended (computedAt reset) rather than destroyed
- This means: if D1 goes down, existing sessions keep their last-known access map
- A fresh login during D1 outage would fail (cannot compute initial PLAC)

This is a **deliberate trade-off**: operational continuity for existing sessions vs. strict freshness. For security-critical operations, RBAC checks are always enforced regardless of PLAC.

---

## Force-Kick (Session Revocation)

Three-layer revocation for when access must be terminated immediately:

```
Layer 1 (Fastest — <1 second):
  DELETE KV session:{userId}
  → User's current session cookie is now invalid
  → Next request results in re-auth redirect

Layer 2 (Persistent — survives new login attempt):
  PUT KV revoked:{userId} = true (TTL: 48 hours)
  → Even if user re-authenticates via CF Zero Trust
  → cf-admin middleware blocks session creation

Layer 3 (Permanent — database level):
  UPDATE Supabase admin_authorized_users SET is_active = false
  → Blocks all future logins indefinitely
  → Only a DEV can reverse this
```

Additionally, the Worker can call the **Cloudflare API** to revoke the CF Access session itself, preventing the user from even reaching the cf-admin Worker. This requires `CF_API_TOKEN_ZT_WRITE` secret.

---

## Local Development Mode

In local development, Cloudflare Zero Trust is not available. A bypass is configured:

```typescript
// src/lib/auth/guard.ts
const isLocalDev = env.SITE_URL?.includes('localhost') ?? false;

if (isLocalDev) {
  // Bootstrap a fake session for LOCAL_DEV_ADMIN_EMAIL
  // No JWT verification, no Supabase lookup
  // Role defaults to 'dev' for full local access
}
```

**Security invariant**: `isLocalDev()` returns `false` unless `SITE_URL` explicitly contains a local dev domain. Missing `SITE_URL` defaults to production mode — **fail-secure, never fail-open**.

`LOCAL_DEV_ADMIN_EMAIL` is a `[vars]` entry in `wrangler.toml` (not a secret — it's `harshil.8136@gmail.com`).
