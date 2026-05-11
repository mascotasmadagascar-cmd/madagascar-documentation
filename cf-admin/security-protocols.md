# cf-admin — Security Protocols

All 10 security hardening layers: implementation details, configurations, and threat model context.

---

## Defense in Depth Philosophy

cf-admin implements 10 independent security layers. Each layer assumes the previous layer may be bypassed. An attacker must defeat multiple independent controls simultaneously to gain unauthorized access or perform unauthorized actions.

---

## Layer 1: CSRF Protection

**Threat:** Cross-Site Request Forgery — an attacker tricks a logged-in admin into triggering a state-changing request from another site.

**Implementation:** `src/lib/csrf.ts` — Origin and Referer header validation on all mutations.

All `POST`, `PUT`, `PATCH`, and `DELETE` requests are validated in Layer 3 of the middleware pipeline:

```typescript
// Middleware Layer 3 — runs before session retrieval
const origin = request.headers.get('Origin');
const referer = request.headers.get('Referer');
const expectedOrigin = new URL(env.SITE_URL).origin;  // https://secure.madagascarhotelags.com

if (origin && origin !== expectedOrigin) {
  return new Response('CSRF: invalid origin', { status: 403 });
}
if (referer && !referer.startsWith(expectedOrigin)) {
  return new Response('CSRF: invalid referer', { status: 403 });
}
```

**Coverage:** All state-mutating endpoints. Read-only GET/HEAD requests are exempt.

---

## Layer 2: Rate Limiting

**Threat:** Brute-force attacks, scraping, or abuse of sensitive management endpoints.

**Implementation:** `src/lib/ratelimit.ts` (62 lines) — Upstash Redis sliding window via `@upstash/ratelimit`.

**Limits by endpoint:**

| Endpoint Pattern | Limit | Window |
|-----------------|-------|--------|
| `POST/PATCH/DELETE /api/users/*` | 10 requests | 1 hour |
| `POST /api/users/access` (PLAC provisioning) | 5 requests | 1 minute |

**Rate limit response:**
```json
{ "success": false, "error": "Rate limit exceeded", "code": "RATE_LIMIT" }
```
HTTP status: `429 Too Many Requests`.

**Redis key prefix:** cf-admin uses its own key prefix in the shared Upstash instance to avoid collisions with cf-astro and cf-chatbot rate limiters.

---

## Layer 3: IP Hashing (Privacy-Safe Audit Trail)

**Threat:** Storing raw IP addresses creates GDPR/privacy liability and creates a high-value target if the audit log is exfiltrated.

**Implementation:** All IP addresses in audit logs are stored as HMAC-SHA256 hashes:

```
ip_hash = HMAC-SHA256(rawIpAddress, IP_HASH_SECRET)
```

**Properties:**
- `IP_HASH_SECRET` is stored as a secret (not in `[vars]`) — without it, hashes cannot be reversed
- The same IP address always produces the same hash (correlation still possible across events)
- Raw IP is never written to D1 or logged anywhere

**Rotation:** `IP_HASH_SECRET` should never be rotated — rotation would break the ability to correlate historical audit events from the same IP address.

---

## Layer 4: 3-Layer Force-Logout

**Threat:** A compromised or malicious admin account continues to have access after revocation is initiated.

**Implementation:** `POST /api/users/force-kick` (min role: `owner`) executes all 3 layers in sequence:

### Layer 4a — KV Session Deletion (immediate, <1 second)

```typescript
// Enumerate all sessions for the target user via reverse index
// KV keys: user-session:{userId}:{sessionId}
// Delete each: session:{sessionId}
// Write: revoked:{userId} = '1' (TTL 24 hours)
```

Effect: The user's next request finds no valid session cookie and is redirected to login.

### Layer 4b — KV Revocation Flag (blocks re-bootstrap for 24 hours)

```typescript
// KV key: revoked:{userId} = '1' (TTL 86400s)
// Checked in middleware Layer 6 (CF Zero Trust Bootstrap)
```

Effect: Even if the user has a valid CF Access JWT, the middleware refuses to create a new session while the revocation flag exists. The user cannot re-authenticate for 24 hours without a `dev` or `owner` manually removing the flag.

### Layer 4c — CF API Session Revocation (terminates CF Access session)

```typescript
// DELETE https://api.cloudflare.com/client/v4/accounts/{CF_ACCOUNT_ID}/access/users/{cfSubId}/active_sessions
// Authorization: Bearer {CF_API_TOKEN_ZT_WRITE}
// Required scope: Access: Revoke Tokens
```

Effect: Revokes the Cloudflare Access session itself. The user is blocked at the Cloudflare edge — the Worker never even receives their requests.

**`cfSubId` source:** Stored in `admin_authorized_users.cf_sub_id` and in `AdminSession.cfSubId`. Populated from the JWT `sub` claim at session creation time.

---

## Layer 5: Ghost Audit Trail

**Threat:** Unauthorized or malicious actions go undetected; no forensic trail for incident response.

**Implementation:** `src/lib/audit.ts` — INSERT-only D1 table, written async via `ctx.waitUntil()`.

**Properties:**
- Zero latency impact — writes happen after the HTTP response is sent
- INSERT-only at application layer — no UPDATE or DELETE on `admin_audit_log`
- Every page view, API mutation, and access decision is logged
- `ip_hash` field stores HMAC hash (never raw IP)
- All 11 modules and 13 action types are covered (see [audit-and-diagnostics.md](./audit-and-diagnostics.md))

**Audit silence (DEV-only):** `POST /api/audit/silence` can toggle `auditSilenced = true` in the current session. This suppresses audit writes for testing purposes only. The toggle itself is logged when it is activated and deactivated.

---

## Layer 6: Hidden Accounts

**Threat:** A lower-privilege admin accidentally (or maliciously) modifies, deactivates, or deletes a DEV or OWNER account.

**Implementation:** `is_hidden = 1` in `admin_authorized_users`. Enforced in `GET /api/users/index` and all user management endpoints.

**Visibility rule:** A requesting user can only see accounts with a role level numerically greater than or equal to their own level. DEV (level 0) can see all accounts. STAFF (level 4) cannot see OWNER (level 1) or DEV (level 0) accounts.

```typescript
// In ROLE_LEVEL terms: only show target if ROLE_LEVEL[target.role] >= ROLE_LEVEL[requester.role]
// i.e., target's privilege is lower than or equal to requester's privilege
```

**BREAK_GLASS_EMAILS:** Hardcoded in `src/lib/auth/rbac.ts`. These email addresses retain `dev` access regardless of database state — last-resort recovery if the `admin_authorized_users` table is corrupted or all DEV accounts are accidentally deactivated.

---

## Layer 7: Role Hierarchy Enforcement

**Threat:** Privilege escalation — a lower-privilege admin grants themselves or another user a higher role.

**Implementation:** `hasPermission()` in `src/lib/auth/rbac.ts`:

```typescript
const ROLE_LEVEL = { dev: 0, owner: 1, super_admin: 2, admin: 3, staff: 4 }

function hasPermission(userRole: Role, requiredRole: Role): boolean {
  return ROLE_LEVEL[userRole] <= ROLE_LEVEL[requiredRole]
}
```

**Enforcement rules in user management:**
- You cannot assign a role higher than your own (e.g., an `admin` cannot create an `owner`)
- You cannot modify a user with a higher role than yours (e.g., an `admin` cannot modify a `super_admin`)
- PLAC grants cannot exceed the granting user's own clearance level

These checks are enforced server-side in the API route handlers, independent of PLAC.

---

## Layer 8: Hard Session Expiry

**Threat:** Long-lived sessions that persist indefinitely, allowing continued access after a user leaves the organization or credentials are compromised.

**Implementation:** Every `AdminSession` has a `createdAt` timestamp (Unix ms). The middleware checks this on every request:

```typescript
const maxLifetime = parseInt(env.SESSION_MAX_LIFETIME_MS);  // 86400000ms = 24 hours
if (Date.now() - session.createdAt > maxLifetime) {
  await destroySession(env.SESSION, sessionId);
  return redirect('/');
}
```

**The 24-hour expiry is a hard limit — non-renewable.** When a session expires, the user must re-authenticate via CF Zero Trust. This provides defense-in-depth beyond CF Access session TTL.

**30-minute role re-check:** Even within the 24-hour window, the user's role and `is_active` status are re-verified against Supabase every 30 minutes (async via `waitUntil`). If `is_active` is set to `0`, the session is destroyed immediately on the next request.

---

## Layer 9: No-Cache Headers on Sensitive Responses

**Threat:** Shared or public browsers (hotel front desk computers, etc.) cache sensitive admin pages or API responses, exposing them to subsequent users.

**Implementation:** All API endpoints via `jsonOk()` and `jsonError()` response helpers, and all dashboard page responses:

```
Cache-Control: no-store, no-cache, must-revalidate
Pragma: no-cache
```

These headers prevent browsers, CDN, and proxy caches from storing any admin response content.

---

## Layer 10: Security Response Headers

**Threat:** Clickjacking, MIME type confusion, information leakage via Referer, downgrade attacks.

**Implementation:** Applied by the middleware to every response (including error responses):

| Header | Value | Mitigates |
|--------|-------|-----------|
| `X-Frame-Options` | `DENY` | Clickjacking via iframe embedding |
| `X-Content-Type-Options` | `nosniff` | MIME type confusion attacks |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Referrer leakage to third parties |
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` | SSL stripping / downgrade attacks |
| `Content-Security-Policy` | Restrictive `default-src`, allows Sentry CDN | XSS, data injection |

**CSP policy scope:**
- `default-src 'self'`
- Sentry CDN allowed for error reporting
- No inline scripts (all JavaScript is bundled)
- No `eval()`

---

## Security Header Interaction with Middleware

Security headers are applied at the end of the middleware pipeline — after all auth checks — so they are present on all responses including 401, 403, and 429 error responses.

---

## Login Forensics as a Security Layer

The `admin_login_logs` table and the cron-based CF Access audit log ingestion provide a complete forensic record of all authentication attempts, including those blocked at the Cloudflare edge before reaching the Worker.

Security-relevant data captured:
- `cf_bot_score` — CF Bot Management score; low scores indicate automated attacks
- `cf_ray_id` — cross-references to Cloudflare dashboard for full request trace
- `asn` / `asn_org` — identifies hosting providers commonly used for attacks
- `colo` — Cloudflare colocation, useful for geographically anomalous logins
- `tls_version` — outdated TLS versions may indicate legacy clients or MITM attempts

See [audit-and-diagnostics.md](./audit-and-diagnostics.md) for the full schema.
