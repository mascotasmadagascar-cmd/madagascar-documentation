# 10 — Compliance & Privacy

> LFPDPPP compliance, ARCO rights, consent management, and data minimization.

---

## Regulatory Framework

### LFPDPPP (Ley Federal de Protección de Datos Personales en Posesión de los Particulares)

Mexico's federal data protection law, equivalent to GDPR for Mexican businesses. Key requirements:

| Principle | Requirement | Madagascar Implementation |
|-----------|-------------|--------------------------|
| **Lawfulness** | Collect data with legal basis | Explicit consent forms at booking |
| **Consent** | Obtain informed, explicit consent | Checkbox consent with version tracking |
| **Information** | Privacy notice at collection point | `/privacy` page (es/en), inline notices |
| **Quality** | Data must be accurate and up-to-date | User-editable profiles, ARCO rectification |
| **Purpose** | Use data only for stated purpose | Specific consent types per purpose |
| **Loyalty** | Act in good faith with data | Transparent privacy notice |
| **Proportionality** | Collect minimum necessary | Data minimization enforced |
| **Accountability** | Demonstrate compliance | Audit logs, consent records |

---

## ARCO Rights Implementation

ARCO = **A**cceso, **R**ectificación, **C**ancelación, **O**posición

### Access (Acceso)
- **Endpoint**: `/arco` page + ARCO request form
- **Flow**: User submits request → email to admin → admin processes via cf-admin dashboard
- **Timeline**: 20 business days (per LFPDPPP)
- **Verification**: Identity verification via R2 document upload

### Rectification (Rectificación)
- **Scope**: Correct inaccurate personal data
- **Flow**: User requests via ARCO form → admin updates in Supabase
- **Audit**: All changes logged in `email_audit_log`

### Cancellation (Cancelación)
- **Scope**: Delete personal data (with retention exceptions)
- **Exceptions**: Legal obligations (tax records: 5 years, contracts: 10 years)
- **Implementation**: Soft delete with retention period before hard delete

### Opposition (Oposición)
- **Scope**: Object to data processing for specific purposes
- **Implementation**: Granular consent withdrawal per purpose
- **Effect**: Processing stops for opposed purposes; other consents remain

---

## Consent Architecture

### Collection Points

| Form | Consent Types | Storage |
|------|--------------|---------|
| Booking form | `terms`, `privacy`, `marketing_email` | `booking_consents` table |
| Contact form | `privacy`, `contact_processing` | `contact_consents` table |
| ARCO request | `arco_processing`, `identity_verification` | `arco_consents` table |
| Chatbot (WhatsApp) | Implicit (platform ToS) | `conversations.metadata` |

### Consent Record Schema

```sql
CREATE TABLE booking_consents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID REFERENCES bookings(id),
  consent_type TEXT NOT NULL,         -- 'terms', 'privacy', 'marketing_email'
  consent_version TEXT NOT NULL,       -- '2024-01' (privacy policy version)
  ip_hash TEXT NOT NULL,               -- SHA-256(IP + salt), never raw IP
  user_agent TEXT,                     -- Browser/device fingerprint
  granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  revoked_at TIMESTAMPTZ,             -- Null unless revoked
  revocation_method TEXT               -- 'arco_request', 'user_action', 'admin'
);
```

### Forensic Properties
Each consent record captures:
1. **What**: Specific consent type + version
2. **When**: `granted_at` timestamp (UTC)
3. **Who**: `ip_hash` (anonymized) + `user_agent`
4. **Where**: Booking/form context via foreign key
5. **How**: Form submission (explicit action)

This creates an immutable, auditable consent chain.

---

## Data Minimization

### IP Address Handling
Raw IP addresses are **never stored**. All systems use hashed IPs:

```typescript
async function hashIP(ip: string, salt: string): Promise<string> {
  const data = new TextEncoder().encode(ip + salt);
  const hash = await crypto.subtle.digest('SHA-256', data);
  return Array.from(new Uint8Array(hash))
    .map(b => b.toString(16).padStart(2, '0'))
    .join('');
}
```

### Data Retention Policy

| Data Type | Retention | Justification |
|-----------|-----------|---------------|
| Booking records | 5 years | Mexican tax/commercial law |
| Consent records | 10 years | Legal proof of consent |
| Chat conversations | 1 year | Customer service history |
| Chat messages | 1 year | Linked to conversations |
| Admin audit logs | 2 years | Security compliance |
| Login events | 1 year | Security analysis |
| Email audit logs | 5 years | Business communications record |
| ARCO documents (R2) | Per ARCO request + 1 year | Identity verification |
| Analytics events | 90 days | Business analytics |
| Rate limit keys (Upstash) | <1 hour | Sliding window auto-expiry |
| Session data (KV) | 24 hours | Auto-expiry TTL |
| ISR cache (KV) | 24 hours | Auto-expiry TTL |

### Automated Cleanup
- **KV sessions**: Auto-expire at 24h TTL
- **ISR cache**: Auto-expire at 24h TTL, deploy-scoped keys
- **Upstash keys**: Sliding window auto-expire
- **Chat conversations**: Cron job closes inactive conversations

---

## Privacy Notice

### Required Disclosures (LFPDPPP Art. 15-16)

The `/privacy` page must disclose:
1. Identity of the data controller (Madagascar Pet Hotel)
2. Purpose of data processing
3. Data transfer recipients (if any)
4. How to exercise ARCO rights
5. Consent revocation mechanism
6. Use of tracking technologies (cookies, analytics)
7. Changes to the privacy notice

### Implementation
- Bilingual: Spanish (primary) and English
- Versioned: Each update increments the version
- Accessible: Linked from every form and page footer
- Consent-linked: Version recorded at consent time

---

## Third-Party Data Sharing

| Third Party | Data Shared | Purpose | Legal Basis |
|-------------|-------------|---------|-------------|
| Cloudflare | Request metadata | Infrastructure | Processing agreement |
| Supabase | All application data | Database hosting | Processing agreement |
| Resend | Email addresses + names | Email delivery | Legitimate interest |
| Sentry | Error traces (no PII) | Error monitoring | Legitimate interest |
| Upstash | Rate limit keys (hashed IPs) | Security | Legitimate interest |
| WhatsApp (Meta) | Phone numbers + messages | Communication channel | User-initiated |
| Google (Gemini) | Chat content (anonymized) | AI responses | Legitimate interest |
| Anthropic (Claude) | Chat content (anonymized) | AI responses | Legitimate interest |

### Data Processing Agreements (DPAs)
All major providers (Cloudflare, Supabase, Resend) offer GDPR-compliant DPAs that also satisfy LFPDPPP requirements.

---

## Security Controls for Privacy

| Control | Implementation | Privacy Benefit |
|---------|---------------|----------------|
| RLS (Supabase) | Row-level policies | Data access minimization |
| IP hashing | SHA-256 + salt | Anonymization |
| Consent versioning | Version string per consent | Audit trail |
| R2 private buckets | No public access | Document confidentiality |
| Zero Trust | IdP authentication | Admin access control |
| Audit logging | D1 + Supabase | Accountability |
| Session TTL | 24h hard expiry | Data lifecycle management |
| ARCO form | Dedicated endpoint | Right exercise facilitation |
