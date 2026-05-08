# cf-email-consumer — API & Payloads

> Queue message schemas and template generation details.

---

## Queue Message Payloads

The `cf-email-consumer` accepts highly structured JSON payloads. Any payload not conforming to these schemas is rejected and discarded.

### 1. `booking_admin`
Notifies hotel staff of a new booking request.

```json
{
  "type": "booking_admin",
  "trackingId": "uuid-v4",
  "payload": {
    "bookingRef": "MDG-2026-0042",
    "guestName": "María García",
    "email": "maria@example.com",
    "phone": "+524491234567",
    "checkIn": "2026-06-01",
    "checkOut": "2026-06-05",
    "specialRequests": "Late check-in, please",
    "pets": [
      {
        "name": "Luna",
        "species": "dog",
        "breed": "Golden Retriever"
      }
    ]
  },
  "sentryTrace": "...",
  "sentryBaggage": "..."
}
```

### 2. `booking_customer`
Sends a confirmation receipt to the customer.

```json
{
  "type": "booking_customer",
  "trackingId": "uuid-v4",
  "payload": {
    "guestName": "María García",
    "recipientEmail": "maria@example.com",
    "bookingRef": "MDG-2026-0042",
    "checkIn": "2026-06-01",
    "checkOut": "2026-06-05"
  }
}
```

### 3. `contact`
Forwards contact form submissions to the admin team.

```json
{
  "type": "contact",
  "trackingId": "uuid-v4",
  "payload": {
    "name": "Carlos López",
    "email": "carlos@example.com",
    "phone": "+524491234567",
    "message": "Do you accept large breed dogs?"
  }
}
```

### 4. `arco`
Alerts the privacy officer of an LFPDPPP data rights request.

```json
{
  "type": "arco",
  "trackingId": "uuid-v4",
  "payload": {
    "name": "Ana Rodríguez",
    "email": "ana@example.com",
    "requestType": "access",
    "description": "I want to know what data you have about me",
    "reference": "ARCO-2026-0003",
    "documentPath": "arco-documents/2026/05/uuid.pdf"
  }
}
```

---

## Email Template System

Templates are implemented as pure functions returning HTML strings, localized primarily in Spanish as the target demographic is local.

### Example Template Structure

```typescript
// templates/booking-admin.ts
export function buildBookingAdminEmail(payload: BookingPayload): string {
  const petList = payload.pets
    .map(p => `<li><strong>${p.name}</strong> (${p.breed || p.species})</li>`)
    .join('');

  return `
    <div style="font-family: sans-serif; color: #333; max-width: 600px; margin: 0 auto;">
      <h2 style="color: #2b6cb0;">Nueva Reserva Recibida</h2>
      <p><strong>Referencia:</strong> ${payload.bookingRef}</p>
      <p><strong>Huésped:</strong> ${payload.guestName}</p>
      <p><strong>Fechas:</strong> ${payload.checkIn} al ${payload.checkOut}</p>
      
      <h3>Mascotas:</h3>
      <ul>${petList}</ul>
      
      ${payload.specialRequests ? `
        <h3>Solicitudes Especiales:</h3>
        <p style="background: #f7fafc; padding: 10px; border-left: 4px solid #ecc94b;">
          ${payload.specialRequests}
        </p>
      ` : ''}
      
      <div style="margin-top: 30px; padding-top: 20px; border-top: 1px solid #e2e8f0;">
        <a href="https://secure.madagascarhotelags.com/dashboard/bookings/${payload.bookingRef}" 
           style="background: #4299e1; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px;">
          Ver en Dashboard
        </a>
      </div>
    </div>
  `;
}
```

### Template Strategy
- **Inline Styles**: To ensure compatibility across email clients (Gmail, Outlook, Apple Mail), CSS must be inline.
- **Responsive**: Max-width containers ensure emails look good on mobile devices.
- **Sanitization**: All user inputs (names, messages) must be properly escaped before insertion into the HTML template to prevent injection attacks.
