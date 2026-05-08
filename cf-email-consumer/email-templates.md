# cf-email-consumer — Email Templates

> Template architecture, all 4 email types, sanitization requirements, and modification workflow.

---

## Template Architecture

Email templates in cf-email-consumer are **pure TypeScript functions** that return HTML strings. No template engine, no React Email, no external renderer — just functions.

### Why No Template Engine?

- Cloudflare Workers bundle size limit: 3 MB. React Email + its dependencies is ~500 KB+.
- Template engines add cold-start overhead. Pure functions are instantaneous.
- The templates are simple enough that a template engine adds complexity without value.
- **Eta** (`eta` package, ~2.5 KB) is on the approved whitelist if template complexity grows, but is not currently needed.

### Template Function Signature

```typescript
// src/email/templates/booking-admin.ts
export function buildBookingAdminEmail(payload: BookingAdminPayload): {
  subject: string;
  html: string;
  text: string;  // Plain text fallback for accessibility
}
```

### Critical: Input Sanitization

> ⚠️ **All user-supplied data MUST be HTML-escaped before insertion into templates.** Failure to do this is a Cross-Site Scripting (XSS) vulnerability — malicious HTML in a booking field would execute in the email client.

**Required helper** (must be applied to every user-supplied field):

```typescript
// src/email/sanitize.ts
function escapeHtml(str: string): string {
  return str
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;');
}

// Usage in templates:
const safeName = escapeHtml(payload.guestName);  // Always escape
const safeMessage = escapeHtml(payload.message);  // Always escape
// Numbers and dates don't need escaping, but escaping them is harmless
```

**Fields that MUST be escaped** (all come from user input):
- Guest names (`guestName`, `name`)
- Email addresses (`email`, `recipientEmail`)
- Phone numbers (`phone`)
- Messages and special requests (`message`, `specialRequests`, `description`)
- Pet names and breeds (`pets[].name`, `pets[].breed`)
- ARCO descriptions and references

### CSS: Inline Only

All CSS in email templates must be **inline** (`style=""` attributes). External stylesheets and `<style>` blocks are stripped by most email clients:

```html
<!-- ❌ WRONG — stripped by Gmail -->
<style>.header { color: #2b6cb0; }</style>
<h2 class="header">Nueva Reserva</h2>

<!-- ✅ CORRECT — survives all email clients -->
<h2 style="color: #2b6cb0; font-family: sans-serif; margin: 0 0 16px;">Nueva Reserva</h2>
```

**Tested clients**: Gmail (web + Android), Outlook 2019, Apple Mail (macOS + iOS).

---

## Email Type 1: `booking_admin`

**Trigger**: New booking submitted by a customer
**Recipient**: Hotel staff (admin email address)
**Language**: Spanish (staff-facing)

**Subject**: `Nueva Reserva — {{bookingRef}} | {{guestName}}`

**Payload**:
```typescript
interface BookingAdminPayload {
  bookingRef: string;          // 'MDG-2026-0042'
  guestName: string;           // 'María García'
  email: string;               // 'maria@example.com'
  phone: string;               // '+524491234567'
  checkIn: string;             // '2026-06-01'
  checkOut: string;            // '2026-06-05'
  specialRequests?: string;    // Optional (may be empty)
  pets: Array<{
    name: string;
    species: string;
    breed?: string;
  }>;
}
```

**Content sections**:
1. Header with booking reference and "Nueva Reserva Recibida" heading
2. Guest details table (name, email, phone)
3. Stay dates (check-in, check-out, number of nights)
4. Pet list with species and breed
5. Special requests block (if present, yellow-highlighted)
6. CTA button: "Ver en Dashboard" → links to cf-admin booking detail page
7. Footer with hotel branding

---

## Email Type 2: `booking_customer`

**Trigger**: New booking submitted — sent to the guest
**Recipient**: Guest who made the booking
**Language**: Spanish (customer-facing)

**Subject**: `Confirmación de Reserva — {{bookingRef}} | Hotel Madagascar`

**Payload**:
```typescript
interface BookingCustomerPayload {
  guestName: string;
  recipientEmail: string;
  bookingRef: string;
  checkIn: string;
  checkOut: string;
}
```

**Content sections**:
1. Confirmation header with hotel logo area
2. Personal greeting ("Hola, {{guestName}}!")
3. Confirmation message and booking reference (prominent)
4. Stay summary (dates, nights)
5. Next steps (what to bring, check-in instructions)
6. Contact information (phone, WhatsApp, email)
7. Hotel address
8. Footer with privacy policy link

---

## Email Type 3: `contact`

**Trigger**: Customer submits the contact form on the website
**Recipient**: Hotel staff (admin email)
**Language**: Spanish (staff-facing)

**Subject**: `Nuevo Mensaje de Contacto — {{name}}`

**Payload**:
```typescript
interface ContactPayload {
  name: string;
  email: string;
  phone?: string;
  message: string;            // User-supplied — MUST be escaped
}
```

**Content sections**:
1. "Nuevo Mensaje de Contacto" heading
2. Sender details (name, email, phone)
3. Message content (in a bordered box, fully escaped)
4. Reply button: "Responder a {{email}}" (mailto link)

---

## Email Type 4: `arco`

**Trigger**: Customer submits an ARCO privacy rights request
**Recipient**: Privacy officer / hotel management (admin email)
**Language**: Spanish (staff-facing, legal context)

**Subject**: `Solicitud ARCO — {{requestType}} | {{reference}}`

**Payload**:
```typescript
interface ArcoPayload {
  name: string;
  email: string;
  requestType: 'access' | 'rectification' | 'cancellation' | 'objection';
  description: string;        // User-supplied — MUST be escaped
  reference: string;          // 'ARCO-2026-0003'
  documentPath?: string;      // R2 path to uploaded identity document
}
```

**Content sections**:
1. "Solicitud de Derecho ARCO" header (formal tone)
2. Request type badge (color-coded by type)
3. Requester details
4. Description of the request (in a formal bordered block, fully escaped)
5. Legal reference number (prominent)
6. Document link (if identity doc was uploaded, link to cf-astro's document retrieval endpoint)
7. Legal reminder: 20-business-day response deadline per LFPDPPP Article 32
8. Instructions for the privacy officer

---

## Template Modification Workflow

> ⚠️ cf-email-consumer is a **separate deployed Worker**. Modifying templates requires redeploying the consumer independently of cf-astro or cf-admin.

### Steps to Modify a Template

1. Edit the relevant template file in `cf-email-consumer/src/email/templates/`
2. Test locally:
   ```bash
   cd cf-email-consumer
   npm run dev  # wrangler dev
   # Send a test queue message via the local wrangler dev UI
   ```
3. Verify: Check that all user-supplied fields are escaped, inline CSS is applied, and the email renders correctly in a browser
4. Deploy:
   ```bash
   npm run deploy  # wrangler deploy
   ```

### Adding a New Email Type

1. Add the new type to `cf-email-consumer/src/types/` (TypeScript interface)
2. Create a new template file in `cf-email-consumer/src/email/templates/`
3. Add a new `case` in `cf-email-consumer/src/email/dispatcher.ts`
4. Update `cf-astro/src/lib/email/queue-types.ts` (the message contract)
5. Add the new case in whatever cf-astro/cf-admin API route needs to send this email type
6. Redeploy both cf-email-consumer AND cf-astro (the producer must know the new message type)

---

## Resend Configuration

```typescript
// src/email/dispatcher.ts
const resend = new Resend(env.RESEND_API_KEY);

await resend.emails.send({
  from: 'Madagascar Pet Hotel <no-reply@madagascarhotelags.com>',
  to: recipientEmail,
  subject: emailSubject,
  html: emailHtml,
  text: emailText,            // Plain text fallback
  reply_to: 'info@madagascarhotelags.com'
});
```

**Sender domain**: `madagascarhotelags.com` — must be verified in Resend dashboard with SPF, DKIM, and DMARC records in Cloudflare DNS. Without these, emails land in spam.
