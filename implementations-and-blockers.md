# Phase 2–5 Implementations & Blockers

> Progress report on Sentry integrations, ESM migrations, and HTTP-to-Binding optimizations.

---

## Identified Technical Blockers & Remediation Plan

During the documentation of the Madagascar edge architecture, several technical blockers and inconsistencies were identified across the 4 microservices. The following outlines the remediation steps required to achieve total system parity.

---

### Blocker 1: Sentry Absence in `cf-chatbot`

**Issue**: The `cf-chatbot` AI orchestration service lacks error tracking and distributed tracing, breaking the observability chain from `cf-admin` proxy requests.
**Risk**: Silent failures in the LLM fallback chain, uncaptured API timeouts, and zero visibility into Vectorize/D1 failure rates.

**Remediation Steps**:
1. Install `@sentry/cloudflare`.
2. Wrap the Hono router execution context with `Sentry.withScope`.
3. Extract `sentry-trace` and `baggage` headers from incoming `cf-admin` proxy requests to stitch the distributed trace.
4. Add Sentry to `wrangler.jsonc`.

```typescript
// cf-chatbot/src/index.ts (proposed)
import * as Sentry from '@sentry/cloudflare';

export default {
  async fetch(request, env, ctx) {
    Sentry.init({ dsn: env.SENTRY_DSN });
    return Sentry.wrapFetch(request, env, ctx, () => {
      return app.fetch(request, env, ctx);
    });
  }
}
```

---

### Blocker 2: CJS Module Format in `cf-email-consumer`

**Issue**: The `cf-email-consumer` is currently configured to use CommonJS (CJS), which is legacy and can cause issues with tree-shaking and Cloudflare Workers' native ES Module execution environment.
**Risk**: Larger bundle sizes, potential dependency conflicts (especially with modern ESM-only packages like Drizzle or Sentry), and suboptimal startup times.

**Remediation Steps**:
1. Update `package.json` to include `"type": "module"`.
2. Ensure `tsconfig.json` targets `ESNext` and uses `moduleResolution: "bundler"`.
3. Update build scripts (e.g., esbuild or Wrangler) to output ESM format.
4. Refactor any `require()` statements to `import`.

---

### Blocker 3: Lack of Dead Letter Queue (DLQ) Monitoring

**Issue**: `cf-email-consumer` processes messages with retries, but if a message exhausts all retries (e.g., persistent Resend API failure), it is silently dropped or sent to an unmonitored DLQ.
**Risk**: Critical transactional emails (e.g., booking confirmations) fail permanently without alerting the admin team.

**Remediation Steps**:
1. Configure a formal DLQ in `wrangler.toml`:
   ```toml
   [[queues.consumers]]
   queue = "madagascar-emails"
   dead_letter_queue = "madagascar-emails-dlq"
   ```
2. Create a new consumer function (or add a routing layer in the existing consumer) dedicated to the DLQ.
3. The DLQ consumer should immediately dispatch a critical alert to Sentry or via a secondary channel (e.g., Slack webhook or raw HTTP call) notifying admins of the permanent failure.

---

### Blocker 4: Network Latency via HTTP Proxies

**Issue**: `cf-admin` communicates with `cf-chatbot` and `cf-astro` (for ISR revalidation) over public HTTP endpoints.
**Risk**: Introduces unnecessary TLS overhead, DNS resolution time, and network latency between services that theoretically exist on the same edge network.

**Remediation Steps**:
1. Transition inter-service communication to **Cloudflare Service Bindings**.
2. Service bindings allow Workers to invoke each other directly, bypassing the public internet, resulting in sub-millisecond latency.

```toml
# cf-admin/wrangler.toml (proposed)
[[services]]
binding = "CHATBOT_SERVICE"
service = "cf-chatbot"

[[services]]
binding = "ASTRO_SERVICE"
service = "cf-astro"
```

```typescript
// cf-admin/src/api/chatbot/proxy.ts (proposed)
// Instead of fetch('https://chatbot...')
const response = await env.CHATBOT_SERVICE.fetch(
  new Request('http://internal/api/admin/kb', { ... })
);
```
