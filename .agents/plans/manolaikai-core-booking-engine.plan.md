# Plan: ManoLaikas — Core Booking Engine (Phase 1)

## Summary

Build the foundational booking engine for ManoLaikas: a Next.js 14 full-stack web application where beauty professionals register, get admin-approved, subscribe (15 EUR/month via Stripe), then receive a shareable booking link synced with their Google/iCloud calendar. The slot computation engine serves real-time availability to a mobile-first public booking page that any client can access without registering. Subscription state gates access: lapsed accounts get a 7-day grace period before their booking page is disabled.

## User Stories

**Professional**: As a self-employed beauty professional, I want to register, get approved, and have a shareable booking link so that clients can book themselves without back-and-forth messages.

**Client**: As a client, I want to open a link from my hairdresser and book an available slot so that I don't have to text them. If I want an earlier date, I open the same link again and check — the availability is always live.

## Metadata

| Field | Value |
|-------|-------|
| Type | NEW_CAPABILITY |
| Complexity | HIGH |
| Systems Affected | Database, Auth, Registration, Stripe, Calendar API, Public Web |
| PRD Phase | 1 of 6 |
| PRD Status | pending |

---

## Client Booking Link — How It Works

**There is no client-facing search or marketplace.** The flow is:

1. Professional shares `/book/[slug]` directly — via WhatsApp, SMS, Instagram bio, business card QR code
2. Client opens the link → sees professional's name + available services → picks service, date, slot → books
3. The link **always shows live availability** — if a client wants to check if an earlier slot opened up, they simply open the same link again and pick an earlier date
4. No client account, login, or registration — just name, phone, optional email, GDPR consent

The platform is never browsable. If a professional's subscription has lapsed, their booking page shows "Not currently accepting bookings" until they renew.

---

## Registration & Monetization Flow

```
/register form
    ↓
Creates Professional (status: pending_approval)
Sends email to ADMIN_EMAIL
    ↓
Admin clicks approval link in email
    → GET /api/admin/approve/[id]?secret=ADMIN_SECRET
    → status → "approved"
    → Sends email to professional with Stripe Checkout link
    ↓
Professional pays 15 EUR/month via Stripe Checkout
    → Stripe fires checkout.session.completed webhook
    → status → "active", store stripeCustomerId + stripeSubscriptionId
    ↓
Professional signs in with Google (same email)
    → calendar access granted
    → dashboard unlocked
    ↓
Subscription lapses (invoice.payment_failed)
    → gracePeriodEnd = now + 7 days
    → Dashboard: shows renewal banner
    → Booking page: still works during grace period
    ↓ (after 7 days)
customer.subscription.deleted or grace period expired
    → status → "suspended"
    → Booking page: "Not currently accepting bookings"
    → Dashboard: readable (can see existing appointments), can't receive new bookings
    → Renewal button → Stripe Customer Portal
```

**Subscription billing**: Stripe (15 EUR/month). Paysera reserved for client deposits (Phase 5).

---

## Architecture Decision

**Stack**: Next.js 14 App Router + TypeScript + Prisma + PostgreSQL + NextAuth.js + Stripe + Resend (email) + Tailwind CSS + shadcn/ui + Bun + Biome

**Rationale**:
- Next.js App Router: single deployment for frontend + API
- Prisma: type-safe schema with migrations and transactions
- NextAuth v5 with Google: OAuth2 tokens for Calendar API access
- Stripe: best-in-class recurring subscriptions with webhook lifecycle events
- Resend: simple transactional email API (admin notifications + professional emails)
- Bun runtime + Biome linting (consistent with framework conventions)

---

## Project Structure

```
app/  (at /home/kaylbie/claude-projects/hss/app/)
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx              ← Google sign-in
│   │   │   ├── register/page.tsx           ← Registration form
│   │   │   ├── pending/page.tsx            ← "Awaiting admin approval"
│   │   │   └── subscribe/page.tsx          ← "Subscribe to activate"
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx                  ← Auth guard + subscription guard
│   │   │   ├── page.tsx                    ← Dashboard home
│   │   │   ├── appointments/page.tsx
│   │   │   ├── services/page.tsx
│   │   │   └── settings/page.tsx
│   │   ├── book/[slug]/
│   │   │   ├── page.tsx                    ← PUBLIC booking page
│   │   │   └── confirm/page.tsx
│   │   ├── api/
│   │   │   ├── auth/[...nextauth]/route.ts
│   │   │   ├── register/route.ts           ← POST new professional
│   │   │   ├── admin/approve/[id]/route.ts ← Admin approval endpoint
│   │   │   ├── subscription/
│   │   │   │   ├── checkout/route.ts       ← Create Stripe Checkout session
│   │   │   │   └── portal/route.ts         ← Stripe Customer Portal link
│   │   │   ├── webhooks/stripe/route.ts    ← Stripe webhook handler
│   │   │   ├── appointments/route.ts
│   │   │   ├── appointments/[id]/route.ts
│   │   │   ├── availability/[slug]/route.ts
│   │   │   ├── services/route.ts
│   │   │   ├── services/[id]/route.ts
│   │   │   ├── booking/route.ts
│   │   │   └── calendar/sync/route.ts
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── booking/
│   │   │   ├── SlotPicker.tsx
│   │   │   ├── ServiceSelector.tsx
│   │   │   └── BookingForm.tsx
│   │   └── dashboard/
│   │       ├── AppointmentList.tsx
│   │       ├── WeekView.tsx
│   │       └── SubscriptionBanner.tsx      ← Lapse warning
│   ├── lib/
│   │   ├── auth.ts                         ← NextAuth config
│   │   ├── db.ts                           ← Prisma singleton
│   │   ├── stripe.ts                       ← Stripe client + helpers
│   │   ├── email.ts                        ← Resend email helpers
│   │   ├── calendar/
│   │   │   ├── google.ts
│   │   │   └── ical.ts
│   │   ├── availability.ts
│   │   └── validations.ts
│   └── types/index.ts
├── prisma/schema.prisma
├── .env.example
├── package.json
├── tsconfig.json
├── tailwind.config.ts
├── biome.json
└── next.config.ts
```

---

## Patterns to Follow

### Error Handling
```
// SOURCE: .claude/agents/silent-failure-hunter.md:39-73
// Never catch broadly — be specific about error types
// Always log with context (operation, IDs, relevant state)
// Return actionable error messages to users
// No empty catch blocks, no silent null returns on error
try {
  const result = await googleCalendar.events.list({ calendarId });
} catch (error) {
  if (error instanceof GaxiosError && error.status === 401) {
    throw new CalendarAuthError("Google Calendar token expired", { professionalId });
  }
  logError("calendar.sync.failed", { error, professionalId, calendarId });
  throw new CalendarSyncError("Failed to fetch calendar events");
}
```

### Type Design
```
// SOURCE: .claude/agents/type-design-analyzer.md:1-112
// Use discriminated unions for status fields — never raw strings
type ProfessionalStatus = "pending_approval" | "approved" | "active" | "suspended" | "cancelled";
type AppointmentStatus = "pending" | "confirmed" | "cancelled" | "no_show";
type CalendarProvider = "google" | "ical";
// Brand IDs to prevent mix-ups
type ProfessionalId = string & { readonly brand: "ProfessionalId" };
```

### Naming Conventions
```
// SOURCE: .claude/agents/code-simplifier.md:1-81
// ES modules with function declarations
// Explicit return types on all functions
// No nested ternaries — use if/switch for status branching
// Kebab-case filenames, PascalCase components, camelCase utilities
```

### Validation (per implement.md pattern)
```
// SOURCE: .claude/commands/implement.md
// Validate after every file: bun run build
// Never accumulate broken state across multiple files
```

### Test Patterns
```
// SOURCE: .claude/agents/pr-test-analyzer.md:15-49
// Test behavior and contracts, not implementation details
// DAMP: Descriptive and Meaningful Phrases
// Critical: slot computation, booking conflict, subscription guard
```

---

## Database Schema

```prisma
// prisma/schema.prisma

model Professional {
  id           String             @id @default(cuid())
  name         String
  email        String             @unique
  phone        String
  specialty    Specialty
  city         String
  bio          String?
  slug         String             @unique   // /book/{slug}
  icalUrl      String?                      // iCloud .ics feed URL

  // Account lifecycle
  status              ProfessionalStatus @default(pending_approval)
  stripeCustomerId    String?            @unique
  stripeSubscriptionId String?           @unique
  currentPeriodEnd    DateTime?          // next billing date
  gracePeriodEnd      DateTime?          // suspend after this if not renewed

  workingHours Json      // WorkingHour[]
  slotDuration Int       @default(30)
  createdAt    DateTime  @default(now())
  updatedAt    DateTime  @updatedAt

  services     Service[]
  appointments Appointment[]
  calendarSync CalendarSync[]
}

model Service {
  id              String  @id @default(cuid())
  professionalId  String
  name            String
  durationMinutes Int
  priceCents      Int
  bufferMinutes   Int     @default(0)
  isActive        Boolean @default(true)

  professional Professional @relation(fields: [professionalId], references: [id])
  appointments Appointment[]
}

model Appointment {
  id             String            @id @default(cuid())
  professionalId String
  serviceId      String
  clientName     String
  clientPhone    String
  clientEmail    String?
  smsConsent     Boolean           @default(false)  // GDPR
  startTime      DateTime
  endTime        DateTime
  status         AppointmentStatus @default(confirmed)
  notes          String?
  googleEventId  String?
  createdAt      DateTime          @default(now())

  professional Professional @relation(fields: [professionalId], references: [id])
  service      Service      @relation(fields: [serviceId], references: [id])

  @@index([professionalId, startTime])
}

model CalendarSync {
  id             String           @id @default(cuid())
  professionalId String
  provider       CalendarProvider
  accessToken    String           // encrypted at rest
  refreshToken   String?          // encrypted at rest
  calendarId     String
  expiresAt      DateTime?
  lastSyncedAt   DateTime?

  professional Professional @relation(fields: [professionalId], references: [id])

  @@unique([professionalId, provider])
  @@index([professionalId])
}

enum ProfessionalStatus {
  pending_approval  // registered, waiting for admin review
  approved          // admin approved, must subscribe to activate
  active            // paying subscriber — all features enabled
  suspended         // subscription lapsed past grace period — booking page disabled
  cancelled         // account closed
}

enum AppointmentStatus {
  pending
  confirmed
  cancelled
  no_show
}

enum CalendarProvider {
  google
  ical
}

enum Specialty {
  hairdresser
  manicurist
  cosmetologist
  other
}

// WorkingHour JSON shape:
// { dayOfWeek: 0-6, startTime: "09:00", endTime: "18:00" }
```

---

## Subscription Status Logic

```ts
// Helper: isAcceptingBookings(professional)
function isAcceptingBookings(p: Professional): boolean {
  if (p.status === "active") return true;
  if (p.status === "suspended" && p.gracePeriodEnd && p.gracePeriodEnd > new Date()) {
    return true; // still within grace period
  }
  return false;
}

// Dashboard access: allow if status is "active" | "suspended" (read-only) | "approved" (redirect to subscribe)
// Reject if: "pending_approval" (redirect to /pending) | "cancelled"
```

---

## Slot Computation Algorithm

```
// lib/availability.ts
// Input: professionalId, date, serviceId
// 1. Load professional (workingHours, slotDuration, icalUrl)
// 2. Load service (durationMinutes, bufferMinutes)
// 3. Get working window for that day (dayOfWeek match)
// 4. Fetch existing Appointments for that day (status != cancelled)
// 5. Fetch Google Calendar freebusy blocks
// 6. Fetch iCal blocks if icalUrl is set
// 7. Merge all busy blocks; merge overlapping intervals
// 8. Generate candidate slots at slotDuration intervals within working hours
// 9. Filter out slots where [start, start+duration+buffer] overlaps any busy block
// 10. Filter out past slots (< now + 60 min)
// Output: TimeSlot[] sorted ascending
```

---

## API Contracts

### Public Endpoints (no auth)

**POST** `/api/register`
```json
// Request
{ "name": "Ana K.", "email": "ana@example.com", "phone": "+37060000000",
  "specialty": "hairdresser", "city": "Vilnius", "bio": "..." }
// Response 201
{ "id": "clx...", "status": "pending_approval" }
// Response 409 if email already registered
```

**GET** `/api/admin/approve/[id]?secret=ADMIN_SECRET`
```
// Sets status → "approved", sends email to professional
// Response 200 text/html — simple "Approved" confirmation page
// Response 401 if wrong secret
// Response 404 if professional not found
// Response 409 if already approved
```

**GET** `/api/availability/[slug]?date=2026-03-01&serviceId=xxx`
```json
// Returns 404 if professional not found
// Returns { accepting: false } if !isAcceptingBookings(professional)
{
  "accepting": true,
  "slots": [{ "startTime": "2026-03-01T09:00:00Z", "endTime": "2026-03-01T10:00:00Z" }],
  "professional": { "name": "Ana", "slug": "ana" },
  "service": { "name": "Haircut", "durationMinutes": 60, "priceCents": 3500 }
}
```

**POST** `/api/booking`
```json
// Request
{ "professionalSlug": "ana", "serviceId": "clx...", "startTime": "2026-03-01T09:00:00Z",
  "clientName": "Marta K.", "clientPhone": "+37060000000", "smsConsent": true }
// Response 201
{ "appointmentId": "clx...", "confirmationCode": "MANO-A1B2" }
// Response 409 if slot already taken
// Response 403 if professional not accepting bookings
```

### Protected Endpoints (require active/suspended session)

**POST** `/api/subscription/checkout` → returns `{ url: "https://checkout.stripe.com/..." }`
**POST** `/api/subscription/portal` → returns `{ url: "https://billing.stripe.com/..." }`
**POST** `/api/webhooks/stripe` → Stripe webhook (raw body, signature verified)
**GET** `/api/appointments?from=&to=` → list appointments
**PATCH** `/api/appointments/[id]` → update status
**GET/POST** `/api/services` → list/create services
**PATCH/DELETE** `/api/services/[id]`
**POST** `/api/calendar/sync`

---

## Files to Change

| File | Action | Purpose |
|------|--------|---------|
| `app/package.json` | CREATE | next, prisma, next-auth, stripe, resend, googleapis, node-ical, zod, @tanstack/react-query, tailwind, shadcn, biome |
| `app/tsconfig.json` | CREATE | Strict TypeScript |
| `app/biome.json` | CREATE | Linting |
| `app/next.config.ts` | CREATE | Next.js config (add stripe webhook raw body parsing) |
| `app/prisma/schema.prisma` | CREATE | Full schema incl. ProfessionalStatus, Specialty enums |
| `app/src/types/index.ts` | CREATE | Branded IDs, WorkingHour, TimeSlot, ApiError types |
| `app/src/lib/db.ts` | CREATE | Prisma singleton |
| `app/src/lib/auth.ts` | CREATE | NextAuth config — registration guard in signIn callback |
| `app/src/lib/stripe.ts` | CREATE | Stripe client, createCheckoutSession(), createPortalSession(), isAcceptingBookings() |
| `app/src/lib/email.ts` | CREATE | Resend helpers: sendAdminApprovalNotification(), sendProfessionalApprovedEmail() |
| `app/src/lib/validations.ts` | CREATE | Zod schemas: RegistrationRequest, BookingRequest, CreateService, UpdateAppointment |
| `app/src/lib/availability.ts` | CREATE | Slot computation engine |
| `app/src/lib/calendar/google.ts` | CREATE | Google Calendar API: freebusy, createEvent, deleteEvent, token refresh |
| `app/src/lib/calendar/ical.ts` | CREATE | iCloud .ics fetch + parse (read-only) |
| `app/src/app/api/auth/[...nextauth]/route.ts` | CREATE | NextAuth route handler |
| `app/src/app/api/register/route.ts` | CREATE | POST professional registration |
| `app/src/app/api/admin/approve/[id]/route.ts` | CREATE | Admin approval + professional notification |
| `app/src/app/api/subscription/checkout/route.ts` | CREATE | Create Stripe Checkout session |
| `app/src/app/api/subscription/portal/route.ts` | CREATE | Stripe Customer Portal link |
| `app/src/app/api/webhooks/stripe/route.ts` | CREATE | Handle Stripe lifecycle events |
| `app/src/app/api/availability/[slug]/route.ts` | CREATE | GET available slots (public, subscription-gated) |
| `app/src/app/api/booking/route.ts` | CREATE | POST public booking (subscription-gated) |
| `app/src/app/api/appointments/route.ts` | CREATE | GET appointments |
| `app/src/app/api/appointments/[id]/route.ts` | CREATE | PATCH appointment status |
| `app/src/app/api/services/route.ts` | CREATE | GET/POST services |
| `app/src/app/api/services/[id]/route.ts` | CREATE | PATCH/DELETE service |
| `app/src/app/api/calendar/sync/route.ts` | CREATE | POST trigger sync |
| `app/src/app/(auth)/login/page.tsx` | CREATE | Google sign-in |
| `app/src/app/(auth)/register/page.tsx` | CREATE | Registration form |
| `app/src/app/(auth)/pending/page.tsx` | CREATE | "Awaiting admin approval" state |
| `app/src/app/(auth)/subscribe/page.tsx` | CREATE | "Subscribe to activate" state |
| `app/src/app/(dashboard)/layout.tsx` | CREATE | Dashboard auth + subscription guard |
| `app/src/app/(dashboard)/page.tsx` | CREATE | Dashboard home |
| `app/src/app/(dashboard)/appointments/page.tsx` | CREATE | Appointment list |
| `app/src/app/(dashboard)/services/page.tsx` | CREATE | Service management |
| `app/src/app/(dashboard)/settings/page.tsx` | CREATE | Working hours + calendar sync |
| `app/src/app/book/[slug]/page.tsx` | CREATE | Public booking page |
| `app/src/app/book/[slug]/confirm/page.tsx` | CREATE | Booking confirmation |
| `app/src/app/layout.tsx` | CREATE | Root layout with providers |
| `app/src/app/page.tsx` | CREATE | Root redirect |
| `app/src/components/booking/SlotPicker.tsx` | CREATE | Date + time slot selector |
| `app/src/components/booking/ServiceSelector.tsx` | CREATE | Service card grid |
| `app/src/components/booking/BookingForm.tsx` | CREATE | Client info + GDPR form |
| `app/src/components/dashboard/AppointmentList.tsx` | CREATE | Appointment card list |
| `app/src/components/dashboard/WeekView.tsx` | CREATE | Week grid |
| `app/src/components/dashboard/SubscriptionBanner.tsx` | CREATE | Lapse warning + renew button |
| `app/.env.example` | CREATE | All env vars |
| `app/src/lib/availability.test.ts` | CREATE | Unit tests for slot engine |

---

## Risks

| Risk | Mitigation |
|------|------------|
| Google OAuth token expiry | Auto-refresh before every calendar API call; store expiresAt |
| Double-booking race condition | Prisma `$transaction` — re-check slot within same tx before insert |
| Stripe webhook replay / duplicate events | Idempotent handlers — check current status before updating |
| Suspended professional books during grace period | `isAcceptingBookings()` helper used in both availability + booking APIs |
| Admin secret in email link (low-security) | Acceptable for 2-person pilot; replace with admin panel post-pilot |
| iCloud CalDAV write complexity | MVP: read-only .ics feed only; CalDAV write deferred |
| GDPR — client phone collection | smsConsent field on Appointment; consent checkbox required on booking form |
| Slug collision | Slug generated from name + 4-char random suffix; UNIQUE DB constraint |

---

## Tasks

Execute in order. Each task must leave `bun run build` passing.

---

### Task 1: Project Scaffolding

- **File**: `app/` + config files
- **Action**: CREATE
- **Implement**:
  - `bun create next-app@latest app --typescript --tailwind --app --src-dir --import-alias "@/*" --no-git`
  - `bun add prisma @prisma/client next-auth@beta googleapis node-ical zod stripe resend @tanstack/react-query`
  - `bun add -d @biomejs/biome`
  - `bunx prisma init --datasource-provider postgresql`
  - `bunx shadcn@latest init`
  - Create `biome.json`; create `.env.example` with all vars below
- **Required env vars**:
  ```
  DATABASE_URL=postgresql://...
  NEXTAUTH_URL=http://localhost:3000
  NEXTAUTH_SECRET=
  GOOGLE_CLIENT_ID=
  GOOGLE_CLIENT_SECRET=
  ENCRYPTION_KEY=          # 32-char key for OAuth token encryption
  STRIPE_SECRET_KEY=
  STRIPE_WEBHOOK_SECRET=
  STRIPE_PRICE_ID=         # 15 EUR/month recurring price ID
  RESEND_API_KEY=
  ADMIN_EMAIL=             # Where admin approval notifications go
  ADMIN_SECRET=            # Secret token in admin approval link
  APP_URL=http://localhost:3000
  ```
- **Validate**: `cd app && bun run build`

---

### Task 2: Database Schema + Prisma Setup

- **File**: `app/prisma/schema.prisma`, `app/src/lib/db.ts`
- **Action**: CREATE
- **Implement**:
  - Full schema from Database Schema section above
  - Prisma singleton in `db.ts` (prevents dev hot-reload connection exhaustion)
  - `bunx prisma generate`
- **Validate**: `cd app && bun run build`

---

### Task 3: Types + Validations

- **File**: `app/src/types/index.ts`, `app/src/lib/validations.ts`
- **Action**: CREATE
- **Implement**:
  - `types/index.ts`: ProfessionalId, WorkingHour, TimeSlot, ApiError types; `isAcceptingBookings()` type guard
  - `validations.ts`: Zod schemas for RegistrationRequest, BookingRequest, CreateServiceRequest, UpdateAppointmentRequest
  - Phone: `z.string().regex(/^\+370\d{8}$/)` (Lithuanian mobile)
  - All schemas export both type and validator: `RegistrationRequestSchema` + `RegistrationRequest`
- **Validate**: `cd app && bun run build`

---

### Task 4: Auth — NextAuth with Google OAuth

- **File**: `app/src/lib/auth.ts`, `app/src/app/api/auth/[...nextauth]/route.ts`
- **Action**: CREATE
- **Implement**:
  - NextAuth v5, Google provider, scopes: `email profile https://www.googleapis.com/auth/calendar`
  - `signIn` callback:
    - Look up Professional by email
    - If not found → deny with `?error=not-registered` (must register first)
    - If `pending_approval` → allow sign-in but `session.redirectTo = "/pending"`
    - If `approved` → allow sign-in but `session.redirectTo = "/subscribe"`
    - If `active` → allow, store/refresh tokens in CalendarSync (google), set `session.redirectTo = "/dashboard"`
    - If `suspended` → allow sign-in (read-only dashboard), `session.redirectTo = "/dashboard"`
    - If `cancelled` → deny
  - `session` callback: attach `professionalId`, `professionalStatus`, `redirectTo` to session
  - `jwt` callback: persist Google access/refresh tokens for calendar use
  - Encrypt tokens with `ENCRYPTION_KEY` before storing in CalendarSync
- **Validate**: `cd app && bun run build`

---

### Task 5: Professional Registration Form

- **File**: `app/src/app/(auth)/register/page.tsx`, `app/src/app/api/register/route.ts`
- **Action**: CREATE
- **Implement**:
  - `register/page.tsx`: Form fields: name, email, phone (+370), specialty (select: hairdresser/manicurist/cosmetologist/other), city, bio (optional)
  - On submit → `POST /api/register`
  - On success → redirect to `/pending`
  - `api/register/route.ts`:
    - Validate body with `RegistrationRequestSchema`
    - Check email not already taken (409 if so)
    - Generate slug: `${firstName.toLowerCase()}-${randomAlphanumeric(4)}` (e.g., "ana-x7k2")
    - Create Professional (status: "pending_approval", default empty workingHours: [])
    - Call `sendAdminApprovalNotification({ professional, approvalUrl })`
    - Return 201 `{ status: "pending_approval" }`
  - `(auth)/pending/page.tsx`: Static message — "Your application is under review. We'll email you when approved." Show email that was registered.
- **Validate**: `cd app && bun run build`

---

### Task 6: Admin Approval + Professional Notification

- **File**: `app/src/lib/email.ts`, `app/src/app/api/admin/approve/[id]/route.ts`, `app/src/app/(auth)/subscribe/page.tsx`
- **Action**: CREATE
- **Implement**:
  - `email.ts` (Resend):
    - `sendAdminApprovalNotification(data)` → sends email to `ADMIN_EMAIL` with professional details + approval link: `${APP_URL}/api/admin/approve/${id}?secret=${ADMIN_SECRET}`
    - `sendProfessionalApprovedEmail(data)` → sends email to professional: "Approved! Subscribe to activate your account: [Stripe Checkout link]"
    - Both functions throw `EmailSendError` on failure; never swallow
  - `api/admin/approve/[id]/route.ts` (GET):
    - Verify `?secret` matches `ADMIN_SECRET` → 401 if not
    - Load Professional by id → 404 if not found
    - If status !== "pending_approval" → 409 "Already processed"
    - Set status → "approved"
    - Create Stripe Customer: `stripe.customers.create({ email, name })`
    - Store stripeCustomerId on Professional
    - Generate Stripe Checkout URL (calls `createCheckoutSession`)
    - Call `sendProfessionalApprovedEmail({ professional, checkoutUrl })`
    - Return simple HTML: "✓ Approved. Email sent to [email]."
  - `(auth)/subscribe/page.tsx`: "Your account is approved! Subscribe for 15 EUR/month to activate your booking link." → Button: "Subscribe now" → `POST /api/subscription/checkout` → redirect to Stripe
- **Validate**: `cd app && bun run build`

---

### Task 7: Stripe Subscription Integration

- **File**: `app/src/lib/stripe.ts`, `app/src/app/api/subscription/checkout/route.ts`, `app/src/app/api/subscription/portal/route.ts`
- **Action**: CREATE
- **Implement**:
  - `stripe.ts`:
    - Export Stripe client singleton
    - `createCheckoutSession(professionalId, stripeCustomerId): Promise<string>` → creates Stripe Checkout session (mode: "subscription", price: STRIPE_PRICE_ID, success_url: `/dashboard?subscribed=1`, cancel_url: `/subscribe`), returns session URL
    - `createPortalSession(stripeCustomerId): Promise<string>` → returns Billing Portal URL
    - `isAcceptingBookings(professional): boolean` → status === "active" OR (status === "suspended" && gracePeriodEnd > now)
  - `api/subscription/checkout/route.ts` (POST, requires session):
    - Get professional from session
    - If status !== "approved" → 409 "Already subscribed or not yet approved"
    - Call `createCheckoutSession`
    - Return `{ url }` (201)
  - `api/subscription/portal/route.ts` (POST, requires session):
    - Get professional from session
    - Call `createPortalSession`
    - Return `{ url }`
- **Validate**: `cd app && bun run build`

---

### Task 8: Stripe Webhook Handler

- **File**: `app/src/app/api/webhooks/stripe/route.ts`, `app/next.config.ts`
- **Action**: CREATE
- **Implement**:
  - `next.config.ts`: Add matcher to disable body parsing for `/api/webhooks/stripe` (needed for Stripe signature verification)
  - Webhook route:
    - Read raw body, verify signature with `STRIPE_WEBHOOK_SECRET`
    - Handle events (idempotent — always check current status before updating):
      - `checkout.session.completed`: look up Professional by stripeCustomerId → set status → "active", store stripeSubscriptionId, currentPeriodEnd from subscription
      - `invoice.payment_succeeded`: update currentPeriodEnd, clear gracePeriodEnd if set, ensure status === "active"
      - `invoice.payment_failed`: set gracePeriodEnd = now + 7 days (do NOT change status yet — grace period active)
      - `customer.subscription.deleted`: set status → "suspended", gracePeriodEnd = now (effective immediate suspension OR let natural grace period expire — use whichever is later)
    - Return 200 for all handled events; 400 for signature failure; never throw unhandled errors (log and 200)
  - Throw `StripeWebhookError` for signature failures only (legitimate errors); log all others
- **Validate**: `cd app && bun run build`

---

### Task 9: Google Calendar Integration

- **File**: `app/src/lib/calendar/google.ts`
- **Action**: CREATE
- **Implement**:
  - `getGoogleCalendarClient(professionalId)` — decrypt tokens from CalendarSync, auto-refresh if expired, return `google.calendar({ version: 'v3' })`
  - `getFreeBusyBlocks(professionalId, start, end): Promise<TimeBlock[]>` — freebusy.query, map to `{ start: Date, end: Date }[]`
  - `createCalendarEvent(professionalId, appointment): Promise<string>` — create event, return eventId
  - `deleteCalendarEvent(professionalId, googleEventId): Promise<void>`
  - Token refresh: on 401, use refreshToken → new accessToken → update CalendarSync record → retry once
  - Typed errors: `CalendarAuthError`, `CalendarSyncError`
- **Validate**: `cd app && bun run build`

---

### Task 10: iCloud CalDAV Integration (Read-Only)

- **File**: `app/src/lib/calendar/ical.ts`
- **Action**: CREATE
- **Implement**:
  - `getIcalBusyBlocks(icalUrl, start, end): Promise<TimeBlock[]>`
  - Fetch .ics via HTTP GET, parse with `node-ical`
  - Filter events within date range, exclude CANCELLED status events
  - In-memory cache per URL (15-minute TTL, `Map<url, { data, fetchedAt }>`)
  - Typed errors: `IcalFetchError`, `IcalParseError`
  - Note: CalDAV write (creating events in iCloud) deferred to post-MVP
- **Validate**: `cd app && bun run build`

---

### Task 11: Slot Computation Engine

- **File**: `app/src/lib/availability.ts`
- **Action**: CREATE
- **Implement**:
  - `computeAvailableSlots({ professionalId, date, serviceId }): Promise<TimeSlot[]>`
  - Steps as defined in Slot Computation Algorithm section
  - Pure computation — calendar and DB calls injected via parameters in tests
  - Handle: no working hours for that day → return []; professional has no Google Calendar sync → skip freebusy; no icalUrl → skip iCal
- **Validate**: `cd app && bun run build`

---

### Task 12: Slot Engine Unit Tests

- **File**: `app/src/lib/availability.test.ts`
- **Action**: CREATE
- **Implement**:
  - `"computeAvailableSlots — empty calendar returns all working-hours slots"`
  - `"computeAvailableSlots — existing appointment blocks its slot"`
  - `"computeAvailableSlots — Google Calendar freebusy block removes overlapping slots"`
  - `"computeAvailableSlots — no working hours for that day returns empty array"`
  - `"computeAvailableSlots — slot too short before end of working hours is omitted"`
  - `"computeAvailableSlots — past slots are filtered out"`
  - `"computeAvailableSlots — buffer time respected (service + buffer = full block)"`
  - `"computeAvailableSlots — overlapping busy blocks are merged before filtering"`
  - Mock DB and calendar calls (vi.mock or bun:test mock)
- **Validate**: `cd app && bun test`

---

### Task 13: Availability API Route

- **File**: `app/src/app/api/availability/[slug]/route.ts`
- **Action**: CREATE
- **Implement**:
  - `GET /api/availability/[slug]?date=YYYY-MM-DD&serviceId=xxx` (no auth required)
  - Validate query params with Zod
  - Look up Professional by slug → 404 if not found
  - If `!isAcceptingBookings(professional)` → return 200 `{ accepting: false, reason: "not_accepting" }`
  - Look up Service → 404 if not found or not belongs to professional
  - Call `computeAvailableSlots`
  - Return `{ accepting: true, slots, professional: { name, slug }, service: { name, durationMinutes, priceCents } }`
  - `Cache-Control: max-age=60`
- **Validate**: `cd app && bun run build`

---

### Task 14: Public Booking API Route

- **File**: `app/src/app/api/booking/route.ts`
- **Action**: CREATE
- **Implement**:
  - `POST /api/booking` (no auth required)
  - Validate body with `BookingRequestSchema`
  - Look up Professional → if `!isAcceptingBookings` → 403
  - Look up Service
  - Compute endTime = startTime + service.durationMinutes
  - Prisma `$transaction`:
    1. Re-fetch appointments overlapping [startTime, endTime] for this professional
    2. If overlap found → 409 "This slot is no longer available"
    3. Create Appointment (status: "confirmed", store smsConsent)
    4. (Outside tx) Create Google Calendar event; on failure: log error, continue (don't fail the booking)
  - Return 201 `{ appointmentId, confirmationCode: "MANO-XXXX" }`
- **Validate**: `cd app && bun run build`

---

### Task 15: Protected Appointment & Service APIs

- **File**: Multiple API routes (see Files to Change)
- **Action**: CREATE
- **Implement**:
  - Auth guard: verify session with `auth()` → 401 if no session
  - Status guard: professionals with "suspended" status can READ appointments but cannot create services (booking is blocked at public booking API level)
  - GET/PATCH appointments (with Google Calendar event deletion on cancel)
  - GET/POST/PATCH/DELETE services
  - POST calendar sync trigger
- **Validate**: `cd app && bun run build`

---

### Task 16: Public Booking Page

- **File**: `app/src/app/book/[slug]/page.tsx` + booking components + confirm page
- **Action**: CREATE
- **Implement**:
  - Server component: load professional + services; if not found → 404; if `!isAcceptingBookings` → show "Not currently accepting bookings" page
  - 3-step client flow: ServiceSelector → SlotPicker → BookingForm
  - SlotPicker: shadcn Calendar for date + time slot grid; fetch from `/api/availability/[slug]`; loading skeleton; mobile: full-width buttons
  - BookingForm: name, phone (with +370 prefix hint), email (optional), GDPR consent checkbox (required: "I consent to receive SMS reminders")
  - On booking success → `/book/[slug]/confirm?id=xxx`
  - Confirm page: appointment details, confirmation code, professional contact info
  - Open Graph meta: "Book with [Professional Name]"
- **Validate**: `cd app && bun run build`

---

### Task 17: Dashboard — Auth + Layout + Appointment Views

- **File**: Login, pending, subscribe pages + dashboard layout + appointment pages
- **Action**: CREATE
- **Implement**:
  - `login/page.tsx`: "Sign in with Google" + "Don't have an account? Register" link
  - `(dashboard)/layout.tsx`:
    - If no session → redirect to `/login`
    - If session.redirectTo exists → redirect there (handles pending/subscribe states)
    - If status === "suspended" → show `SubscriptionBanner` with "Renew subscription" button
  - `SubscriptionBanner.tsx`: amber banner at top — "Your subscription has lapsed. Your booking page is disabled. [Manage subscription]" — links to `/api/subscription/portal`
  - Dashboard home: today's + this week's appointments, quick stats
  - Appointments page: full list with date range filter, status chips, expandable client phone
  - `AppointmentList.tsx`: status chips (pending=yellow, confirmed=green, cancelled=gray, no_show=red)
  - `WeekView.tsx`: 7-column grid; mobile: single-day scroll
- **Validate**: `cd app && bun run build`

---

### Task 18: Dashboard — Services + Settings

- **File**: `app/src/app/(dashboard)/services/page.tsx`, `app/src/app/(dashboard)/settings/page.tsx`
- **Action**: CREATE
- **Implement**:
  - `services/page.tsx`: service list with active/inactive toggle, edit modal, "Add Service" button
  - `settings/page.tsx`:
    1. **Booking link**: display `yourdomain.com/book/[slug]` with copy button
    2. **Working hours**: day toggles + time range pickers, save button
    3. **Calendar sync**: Google — connected/disconnect; iCloud — .ics URL input + test button
    4. **Subscription**: current status badge, next billing date, "Manage subscription" (portal link)
- **Validate**: `cd app && bun run build`

---

### Task 19: Root Layout + Root Page

- **File**: `app/src/app/layout.tsx`, `app/src/app/page.tsx`
- **Action**: CREATE
- **Implement**:
  - Root layout: `<html lang="en">`, session provider (NextAuth), react-query provider
  - Root page: redirect based on session — if session → `/dashboard`; else → `/login`
- **Validate**: `cd app && bun run build`

---

## Validation

```bash
# From app/ directory

bun run build   # Type check + compilation
bun run lint    # Biome
bun test        # Slot engine unit tests

# Manual smoke test sequence:
# 1. POST /api/register — create professional account
# 2. Check ADMIN_EMAIL — click approval link
# 3. Check professional email — click Stripe Checkout link
# 4. Complete Stripe test payment (card: 4242 4242 4242 4242)
# 5. Sign in with Google (same email as registered)
# 6. Configure working hours + add service in dashboard
# 7. Connect Google Calendar
# 8. Visit /book/[slug] as a client — complete booking
# 9. Verify appointment in dashboard + Google Calendar
# 10. Test lapse: mark subscription as past_due via Stripe test webhook
# 11. Verify grace period active, then verify suspension after 7 days
```

---

## Acceptance Criteria

- [ ] Professional can register via form — admin receives email notification
- [ ] Admin can approve via email link — professional receives Stripe Checkout email
- [ ] Professional can subscribe via Stripe Checkout (15 EUR/month)
- [ ] Stripe webhook activates account on successful payment
- [ ] Professional signs in with Google (same email) and accesses dashboard
- [ ] Professional can configure working hours and add services
- [ ] Professional can connect Google Calendar (OAuth)
- [ ] Professional can add iCloud .ics URL for busy block import
- [ ] Public booking page `/book/[slug]` accessible without login
- [ ] Available slots correctly exclude: existing appointments, Google Calendar events, iCloud busy blocks, past times
- [ ] Client can complete full booking flow: service → slot → info → confirm
- [ ] Booking prevents double-booking (409 on conflict)
- [ ] Booking creates Google Calendar event (non-blocking on failure)
- [ ] Suspended account: booking page shows "not accepting" after grace period
- [ ] Suspended account: dashboard readable (appointments visible), renewal banner shown
- [ ] Grace period: booking page still works during 7-day grace period
- [ ] Slot engine unit tests pass (8 scenarios)
- [ ] `bun run build` zero type errors
- [ ] `bun run lint` zero violations
- [ ] All API error paths return typed errors — no silent failures
- [ ] Mobile layout usable at 375px (iPhone SE)

---

## Post-Phase-1 Handoff

Update PRD Phase 1 status to `done`, then proceed to Phase 2 (Client Management):
- Client cards with visit history, photos, and notes
- Visit cycle tracking per client (foundation for Phase 3 Smart Recall)
