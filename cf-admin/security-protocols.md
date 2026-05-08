# cf-admin — Security Protocols

> Zero Trust, RBAC, PLAC, and defense-in-depth mechanisms.

---

## The Security Paradigm: Defense in Depth

cf-admin operates on a "Defense in Depth" philosophy. An attacker must breach multiple independent security boundaries to gain access.

1. **Network Layer**: Cloudflare Access (Zero Trust)
2. **Transport Layer**: CSRF tokens + secure cookies
3. **Session Layer**: KV-backed short-lived sessions
4. **Identity Layer**: Supabase RBAC verification
5. **Authorization Layer**: PLAC (Page-Level Access Control)
6. **Data Layer**: Supabase RLS (Row-Level Security)
7. **Audit Layer**: D1 audit logging
8. **Alerting Layer**: Immediate email notification on suspicious activity

---

## 1. Cloudflare Access (Zero Trust)

Before a request even reaches the Worker, it is intercepted by Cloudflare Access at the network edge.

- **Authentication**: Users must log in via an approved Identity Provider (Google Workspace, GitHub).
- **Authorization**: Only users explicitly added to the specific CF Access application group are allowed through.
- **Token Delivery**: Successful login injects a `CF-Access-JWT-Assertion` header.
- **Verification**: The middleware cryptographically verifies the JWT signature against the Cloudflare public keys endpoint before establishing an application session.

---

## 2. Session Management

Instead of relying solely on the long-lived CF Access JWT, the application creates a short-lived internal session stored in Cloudflare KV.

- **Storage**: `SESSION_KV` binding.
- **Key**: Randomly generated UUID (`session:<uuid>`).
- **TTL**: 24 hours (hard expiration).
- **Cookie**: `SESSION_ID` (HttpOnly, Secure, SameSite=Lax).

**Session Object Structure:**
```typescript
interface Session {
  userId: string;
  email: string;
  role: Role;
  loginMethod: 'cf_access' | 'dev_bypass';
  createdAt: number;
  lastRoleCheckedAt: number;
  placMap: AccessMap;
}
```

### Role Freshness Checking
To prevent the "stale session" problem (where a revoked user maintains access until their cookie expires):
- The session tracks `lastRoleCheckedAt`.
- Every 30 minutes, the middleware queries Supabase to re-verify the user's role and `is_active` status.
- If inactive, the session is instantly destroyed.

### Immediate Revocation
When an admin revokes another user's access, a flag is written to KV (`revoked:<userId>` with a 25h TTL). The middleware checks this flag; if present, the session is destroyed immediately on the next request.

---

## 3. RBAC (Role-Based Access Control)

Roles dictate *what* a user can do.

| Role | Capabilities | Target Users |
|------|--------------|--------------|
| `dev` | Superadmin. Can modify roles, view diagnostics, access all modules. | System architects |
| `admin` | Manage bookings, view audit logs, manage chatbot KB. Cannot modify other admins. | Hotel managers |
| `editor` | Manage bookings, update CMS/feature flags. | Staff |
| `viewer` | Read-only access to bookings and reports. | Accounting, auditors |

---

## 4. PLAC (Page-Level Access Control)

PLAC dictates *where* a user can go.

- **Configuration**: Stored in D1 (`admin_plac_config`).
- **Format**: JSON map linking roles to allowed URL paths.
- **Caching**: The computed PLAC map is cached inside the KV session object.
- **Enforcement**: Middleware checks the requested `pathname` against the cached PLAC map.

**Example D1 Record:**
```json
{
  "path": "/dashboard/bookings",
  "allowed_roles": ["dev", "admin", "editor", "viewer"]
}
```
If a `viewer` tries to access `/dashboard/users` (allowed_roles: `["dev"]`), the middleware intercepts and returns `403 Forbidden` (API) or redirects to an error page (HTML).

---

## 5. Audit Logging

### Ghost Audits
Every successful page view and API action is logged to D1 via `waitUntil` (fire-and-forget, zero latency impact).

```typescript
context.waitUntil(
  writeAuditLog(env.DB, {
    userId: session.userId,
    email: session.email,
    role: session.role,
    action: request.method === 'GET' ? 'view' : 'update',
    module: determineModule(url.pathname),
    details: { path: url.pathname }
  })
);
```

### Security Alert Emails
High-risk events trigger an immediate email to `SECURITY_ALERT_EMAIL` via Resend (dispatched async via Queue):
- First-time login from a new IP
- Multiple failed JWT verifications
- CF Access bypass attempts (dev login endpoint hit in production)
- Elevation of privilege (role changed to `dev` or `admin`)

---

## 6. CSRF Protection

All state-mutating requests (POST, PUT, PATCH, DELETE) require a CSRF token.
- Token is generated on login and stored in the session.
- Client includes it in the `X-CSRF-Token` header.
- Middleware validates it before processing the request.

---

## 7. Security Headers

The `securityHeaders` middleware applies strict headers to all responses:
- `X-Frame-Options: DENY` (Prevent clickjacking)
- `X-Content-Type-Options: nosniff` (Prevent MIME sniffing)
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Strict-Transport-Security` (HSTS)
- `Content-Security-Policy` (Restricts script sources to self and trusted CDNs)
